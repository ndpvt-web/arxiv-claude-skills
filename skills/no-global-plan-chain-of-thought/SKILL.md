---
name: "no-global-plan-chain-of-thought"
description: "Optimize Chain-of-Thought reasoning by detecting when CoT can be bypassed and identifying pivot positions that capture path uncertainty. Use when: 'skip unnecessary reasoning steps', 'optimize CoT length', 'detect if this problem needs chain-of-thought', 'estimate confidence of reasoning', 'reduce thinking tokens', 'bypass CoT for easy questions'."
---

# No Global Plan in Chain-of-Thought: Myopic Reasoning Optimization

This skill applies findings from the Tele-Lens research to optimize how Claude uses Chain-of-Thought reasoning. The core discovery is that LLMs have a **myopic planning horizon** — they reason incrementally one step at a time rather than executing a precomputed global plan. This means explicit CoT is often redundant for tasks where the answer is already latently available, and for harder tasks, only a small subset of reasoning positions (pivot points) carry meaningful uncertainty signal. Claude can use these insights to: (1) detect when CoT can be safely bypassed, (2) compress reasoning by focusing on critical pivot positions, and (3) produce better-calibrated confidence estimates by sampling uncertainty from key positions rather than aggregating over the full reasoning path.

## When to Use

- When a user asks Claude to solve a batch of problems and wants to minimize token usage or latency by skipping reasoning on easy items
- When building an agent pipeline that needs to decide whether to invoke expensive chain-of-thought or answer directly
- When the user asks for confidence estimates on multi-step reasoning outputs
- When designing a routing system that classifies questions as "needs deep reasoning" vs. "answer directly"
- When implementing a CoT-compression strategy that keeps only the critical reasoning steps
- When auditing whether an LLM's verbose reasoning is actually contributing to answer quality
- When building evaluation harnesses that need uncertainty-aware scoring of reasoning traces

## Key Technique

**Myopic Horizon.** LLMs do not form a global plan before generating CoT. On compositional tasks (parity checks, cycle detection, multi-step arithmetic), the model's internal confidence in the final answer stays near chance until the very last reasoning step, where it spikes to 90%+ accuracy. This means each CoT token primarily enables the *next* token — not a distant conclusion. The practical implication: if the model's early hidden states already show high confidence in the answer, the remaining CoT tokens are ceremonial and can be skipped.

**Pivot-Position Uncertainty.** Not all CoT positions contribute equally to uncertainty estimation. The "Wooden Barrel Principle" from this research shows that a path's true uncertainty concentrates at a few critical pivot positions — the weakest links in the reasoning chain. Selecting the top-5 positions with lowest final-answer entropy (highest confidence variance) improves AUROC by up to 9% absolute over naive full-path aggregation. For a practical proxy without internal probes, selecting the top-100 positions with highest token-level entropy yields 3-6% AUROC improvement.

**CoT Bypass Detection.** By measuring the normalized entropy of the model's answer distribution after only the first 5 reasoning tokens, you can detect when CoT is unnecessary. If normalized entropy H < 0.1, the model already "knows" the answer and CoT can be skipped. In experiments, this achieved 12-16% thinking reduction with only 0.03% accuracy degradation across commonsense and knowledge benchmarks.

## Step-by-Step Workflow

1. **Classify the task difficulty tier.** Before generating reasoning, assess whether the problem falls into: (a) knowledge/recall (factual lookup, commonsense), (b) implicit compositional (math word problems, logical puzzles), or (c) explicit compositional (algorithmic multi-step like parity, sorting). Tier (a) is most likely to benefit from CoT bypass; tier (c) almost always requires full CoT.

2. **Apply the early-exit entropy check.** Generate the first 5 reasoning tokens. Compute the normalized entropy of the answer distribution: `H_norm = (-sum(p_i * log(p_i))) / log(C)` where C is the number of candidate answers. If `H_norm < 0.1`, the model is already confident — skip the remaining CoT and output the answer directly.

3. **If CoT proceeds, identify pivot positions during generation.** As reasoning tokens are produced, track positions where: (a) token-level log-likelihood drops sharply (the model is uncertain about the next step), (b) the topic or reasoning strategy shifts (e.g., moving from setup to calculation), or (c) an intermediate conclusion is stated. These are your pivot candidates.

4. **Estimate path uncertainty from pivots, not the full path.** Instead of averaging confidence across all N reasoning tokens, select the top-k positions (k=5 for probed confidence, k=100 for token-entropy proxy on long traces) with the highest local uncertainty. The maximum uncertainty among these pivots better represents the true failure risk of the reasoning chain.

5. **Compress the reasoning trace for downstream use.** When storing or displaying CoT for audit purposes, retain only the pivot positions and their surrounding context (2-3 sentences each). This produces a compressed trace that preserves the meaningful decision points while discarding incremental filler.

6. **Route based on bypass signal for batch processing.** When processing multiple questions, run step 2 on each item first. Partition into bypass-eligible and full-CoT queues. Process bypass items with direct answering and full-CoT items with standard reasoning. This yields 12-16% throughput improvement with negligible accuracy loss.

7. **Validate bypass decisions with spot-checking.** Periodically run full CoT on a sample of bypassed items. If the full-CoT answer differs from the direct answer at a rate above 1%, tighten the entropy threshold (e.g., from 0.1 to 0.05).

8. **Report calibrated confidence using pivot-based scoring.** When the user asks "how confident are you?", compute confidence from the pivot positions rather than giving a generic hedged answer. State which reasoning step carried the most uncertainty and why.

## Concrete Examples

**Example 1: Batch Question Answering with CoT Bypass**

User: "Answer these 5 questions and tell me which ones you're confident about: (1) What is the capital of France? (2) What is 347 * 29? (3) If all bloops are ramps and all ramps are clinks, are all bloops clinks? (4) Solve: 3x + 7 = 22. (5) What year did the Berlin Wall fall?"

Approach:
1. Classify each question: Q1=knowledge, Q2=arithmetic, Q3=syllogism, Q4=algebra, Q5=knowledge
2. Run early-exit check: Q1, Q3, Q5 show very low answer entropy after first few tokens — bypass CoT
3. Q2 and Q4 require multi-step computation — use full CoT
4. Report pivot-based confidence for Q2 and Q4

Output:
```
1. Paris [bypassed CoT — high confidence, knowledge recall]
2. 10,063 [full CoT: 347 * 29 = 347 * 30 - 347 = 10,410 - 347 = 10,063]
   Confidence: High — single pivot at the subtraction step, verified.
3. Yes, all bloops are clinks [bypassed CoT — transitive inference, immediate]
4. x = 5 [full CoT: 3x = 15, x = 5]
   Confidence: High — straightforward, no uncertain pivots.
5. 1989 [bypassed CoT — knowledge recall]

Bypass rate: 3/5 questions (60%). All bypassed answers verified as high-confidence.
```

**Example 2: Confidence Estimation on a Hard Math Problem**

User: "Solve this and tell me how confident you are: A train leaves station A at 60 mph. Another train leaves station B, 300 miles away, at 40 mph toward A. A bird flies back and forth between the trains at 80 mph. How far does the bird fly before the trains meet?"

Approach:
1. Early-exit check: entropy is moderate — proceed with CoT
2. Generate reasoning, tracking pivot positions:
   - Pivot 1: Recognizing this is a time-based problem, not a geometric series (low uncertainty — correct framing)
   - Pivot 2: Computing meeting time = 300 / (60+40) = 3 hours (low uncertainty — clean arithmetic)
   - Pivot 3: Computing bird distance = 80 * 3 = 240 miles (low uncertainty)
3. All pivots show low uncertainty — high overall confidence

Output:
```
The trains approach each other at a combined 100 mph. They meet after
300 / 100 = 3 hours. The bird flies continuously at 80 mph for 3 hours,
covering 240 miles.

Confidence: Very high. Three reasoning pivots identified:
  1. Problem framing (time-based, not series) — certain
  2. Meeting time calculation — single arithmetic step, certain
  3. Bird distance calculation — single multiplication, certain
No uncertain pivots in the chain.
```

**Example 3: Agent Pipeline with Reasoning Router**

User: "I'm building a QA agent. Help me implement a router that decides whether to use CoT or direct answering."

Approach:
1. Implement the normalized entropy classifier as the routing mechanism
2. Use the 0.1 threshold with configurable override
3. Add monitoring for bypass accuracy

Output:
```python
from dataclasses import dataclass
from math import log

@dataclass
class RoutingDecision:
    use_cot: bool
    entropy: float
    reasoning: str

def route_question(
    answer_probs: dict[str, float],
    threshold: float = 0.1,
) -> RoutingDecision:
    """Decide whether a question needs Chain-of-Thought reasoning.

    Based on the Tele-Lens finding: if the model's answer distribution
    has normalized entropy below threshold after minimal reasoning tokens,
    CoT can be safely bypassed.

    Args:
        answer_probs: mapping from candidate answer to probability,
                      obtained after generating ~5 reasoning tokens.
        threshold: normalized entropy cutoff (default 0.1 from paper).
    """
    n_classes = len(answer_probs)
    if n_classes <= 1:
        return RoutingDecision(False, 0.0, "single candidate")

    probs = list(answer_probs.values())
    raw_entropy = -sum(p * log(p) for p in probs if p > 0)
    max_entropy = log(n_classes)
    norm_entropy = raw_entropy / max_entropy if max_entropy > 0 else 0.0

    use_cot = norm_entropy >= threshold
    reason = (
        f"H_norm={norm_entropy:.3f} >= {threshold}: needs reasoning"
        if use_cot
        else f"H_norm={norm_entropy:.3f} < {threshold}: bypass CoT"
    )
    return RoutingDecision(use_cot, norm_entropy, reason)


def estimate_path_confidence(
    token_entropies: list[float],
    k: int = 100,
) -> float:
    """Estimate reasoning path confidence from pivot positions.

    Instead of averaging over all tokens, select the top-k highest
    entropy positions (weakest links) and report the worst-case
    uncertainty. This improves AUROC by 3-6% over full aggregation.
    """
    if not token_entropies:
        return 1.0
    pivots = sorted(token_entropies, reverse=True)[:k]
    max_uncertainty = max(pivots)
    # Convert entropy to confidence (0-1 scale)
    confidence = max(0.0, 1.0 - max_uncertainty)
    return confidence
```

## Best Practices

- **Do:** Apply CoT bypass only to knowledge-recall and commonsense questions where the model's early confidence is genuinely high. These are the task categories where the paper shows bypass works safely.
- **Do:** Use pivot-position uncertainty (not full-path average) when reporting confidence. The weakest link in a reasoning chain is more informative than the average link.
- **Do:** Start with a conservative entropy threshold (0.05) and relax it toward 0.1 after validating on your specific task distribution. The paper's 0.1 threshold was tuned on CSQA/MMLU.
- **Do:** Log bypass decisions and periodically audit them with full CoT to detect threshold drift.
- **Avoid:** Bypassing CoT on explicit compositional tasks (multi-step algorithms, parity checks, chain-of-operations). The paper shows these tasks have near-random latent accuracy until the final step — CoT is essential.
- **Avoid:** Treating early high confidence as a guarantee of correctness. The myopic horizon finding means the model can be confident early for the wrong reasons on deceptive problems. Always validate bypass accuracy on your domain.

## Error Handling

- **False bypass on hard questions:** If the entropy check incorrectly classifies a hard question as easy (low entropy but wrong answer), the spot-check in step 7 will catch this. Response: tighten the threshold and add the question type to a "always use CoT" allowlist.
- **Pivot selection on very short CoT:** When the reasoning trace is under 20 tokens, pivot selection is unreliable since there aren't enough positions to sample. Fall back to full-path uncertainty in this case.
- **Entropy computation on open-ended generation:** The bypass method assumes a fixed answer space (multiple choice, numeric, yes/no). For free-form generation where the answer vocabulary is unbounded, the normalized entropy metric is not directly applicable. Use token-level log-likelihood variance at pivot positions instead.
- **Threshold calibration across domains:** The 0.1 threshold from the paper was validated on commonsense (CSQA) and knowledge (MMLU) benchmarks. For specialized domains (medical, legal, scientific), recalibrate using a held-out validation set of 50-100 items with known answers.

## Limitations

- The CoT bypass technique works best on classification-style tasks with bounded answer spaces. Open-ended generation, creative writing, and code generation don't have a clean entropy threshold to exploit.
- The pivot-position uncertainty method requires access to token-level probabilities or entropy scores, which are not available through all API interfaces (e.g., some hosted APIs don't expose logprobs).
- The paper's experiments primarily used Qwen3 and DeepSeek-R1 models. The specific thresholds (0.1 entropy, k=5 pivots) may need recalibration for other model families.
- Myopic reasoning is a finding about current LLMs, not a fundamental law. Future models with improved planning capabilities may invalidate the bypass heuristics.
- This technique optimizes inference efficiency, not reasoning quality. It cannot make a model solve problems it otherwise couldn't — it only avoids wasting tokens on problems the model can already solve without explicit reasoning.

## Reference

- **Paper:** [No Global Plan in Chain-of-Thought: Uncover the Latent Planning Horizon of LLMs](https://arxiv.org/abs/2602.02103v1) (Xu et al., 2026)
- **Code:** [github.com/lxucs/tele-lens](https://github.com/lxucs/tele-lens)
- **Key takeaway:** LLMs reason myopically — one step at a time — with no global plan. Exploit this by bypassing CoT when early confidence is high (normalized entropy < 0.1) and estimating uncertainty from pivot positions rather than full reasoning paths.