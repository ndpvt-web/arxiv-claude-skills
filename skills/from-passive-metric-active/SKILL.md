---
name: "from-passive-metric-active"
description: "Build systems that use LLM uncertainty as an active control signal -- routing computation, triggering tool calls, enabling self-correction, and governing agent decisions. Use when asked to: 'add uncertainty-aware routing to my LLM pipeline', 'make my agent decide when to use tools based on confidence', 'implement adaptive compute for reasoning chains', 'add self-correction when the model is unsure', 'build a confidence-gated retrieval system', 'prevent reward hacking in my RLHF loop'."
---

This skill enables Claude to design and implement systems where LLM uncertainty is not merely logged or displayed, but actively drives runtime behavior: routing requests to different processing depths, triggering tool use, initiating self-correction loops, gating outputs, and shaping reinforcement learning. The approach is grounded in the survey by Zhang et al. (2026), which maps the evolution of uncertainty from passive diagnostic to active control signal across reasoning, agentic, and RL settings.

## When to Use

- When the user wants to build an LLM pipeline that allocates more compute (retries, deeper chains) to hard queries and less to easy ones
- When designing an autonomous agent that should call external tools only when its internal confidence is low
- When implementing self-correction or self-verification that triggers only when the model is uncertain
- When building a retrieval-augmented generation system that fetches documents only when confidence is below a threshold
- When the user needs conformal prediction sets to provide coverage guarantees on LLM outputs
- When designing an RLHF training loop that discounts reward signals in high-uncertainty regions to prevent reward hacking
- When adding uncertainty monitoring to a production LLM service for drift detection and quality gating

## Key Technique

**The core insight**: Traditional LLM deployments compute uncertainty (token entropy, sampling variance) and report it as a metric -- a number attached to an output for humans to inspect. The shift described in this paper is to feed that number back into the system as a control signal that changes what the system does next. This converts uncertainty from a passive label into an active decision variable.

**Three estimation families** underpin the approach. (1) Token-level entropy measures spread across the next-token distribution -- cheap but noisy. (2) Semantic entropy generates multiple completions, clusters them by meaning, and measures entropy over the clusters -- robust to paraphrase variation but requires multiple forward passes. (3) Conformal prediction constructs prediction sets with formal coverage guarantees (e.g., "the true answer is in this set with 90% probability") without assuming the model is calibrated. These can be combined: token entropy for cheap fast-path gating, semantic entropy for expensive verification, conformal sets for safety-critical outputs.

**Three application frontiers** use these signals. In *reasoning*, uncertainty governs adaptive compute depth -- high-confidence problems get short chains, low-confidence ones get deeper search or self-correction retries. In *agentic systems*, uncertainty triggers metacognitive decisions: whether to call a tool, seek clarification, or reflect on the current plan. In *reinforcement learning*, uncertainty on reward model predictions prevents reward hacking by discounting unreliable reward signals and provides intrinsic exploration bonuses for underexplored behavior regions.

## Step-by-Step Workflow

1. **Identify the decision boundary.** Determine what runtime behavior should change based on uncertainty. Map each boundary to one of: route selection, compute depth, tool invocation, output gating, self-correction trigger, or reward discounting.

2. **Choose the estimation method matching your latency and accuracy budget.**
   - Token-level entropy (`-sum(p * log(p))` over logprobs): <1ms overhead, works with a single forward pass. Good for fast-path gating.
   - Sampling consistency: generate N completions (N=5-10), measure agreement. Moderate cost, catches semantic uncertainty.
   - Semantic entropy: cluster the N completions by meaning (e.g., via embedding similarity or NLI), compute entropy over clusters. Higher cost, most robust.
   - Conformal prediction: calibrate on a held-out set to find a threshold that guarantees coverage. Requires a calibration dataset but provides formal guarantees.

3. **Collect a calibration dataset.** Sample 200-1000 examples representative of production traffic. For each, record the uncertainty estimate and whether the model's answer was correct. This dataset is used to set thresholds and evaluate the uncertainty signal's discriminative power.

4. **Set decision thresholds.** Plot uncertainty vs. accuracy on the calibration set. Choose thresholds that achieve the desired operating point (e.g., "route to expensive pipeline when entropy > 2.1" or "abstain when semantic entropy > 0.7"). Use percentile-based thresholds (e.g., top-25% uncertainty) for robustness across distributions.

5. **Implement the branching logic.** Wire the uncertainty estimate into your pipeline as a conditional:
   - Below threshold: fast path (direct answer, no tool call, short chain).
   - Above threshold: slow path (deeper reasoning, tool invocation, retrieval, self-correction loop).
   - Optionally add a third "abstain" tier for extreme uncertainty.

6. **Build self-correction loops for the slow path.** When uncertainty is high, generate an initial answer, then prompt the model to critique and revise it. Re-estimate uncertainty on the revision. Cap iterations (typically 2-3) to prevent infinite loops.

7. **Implement tool-use gating for agents.** Before each agent action, estimate uncertainty on the planned step. If above threshold, the agent should: (a) invoke a relevant tool (search, calculator, API), (b) ask the user for clarification, or (c) decompose the step into sub-steps and re-evaluate.

8. **Add output-level conformal prediction sets (for safety-critical use).** On the calibration set, compute nonconformity scores (e.g., 1 - P(correct_answer)). At inference, include all answers whose score is below the calibrated quantile. Return the prediction set alongside the point estimate.

9. **Monitor uncertainty distributions in production.** Track rolling statistics (mean, p95) of uncertainty scores. Alert when the distribution shifts significantly from calibration -- this signals data drift or model degradation, and thresholds may need recalibration.

10. **For RL applications, integrate uncertainty into the reward signal.** Multiply reward model outputs by a confidence weight: `adjusted_reward = reward * (1 - uncertainty)`. This discounts rewards in regions where the reward model is unreliable, reducing reward hacking. Optionally add an intrinsic exploration bonus proportional to uncertainty to encourage diverse behavior.

## Concrete Examples

**Example 1: Uncertainty-Routed Question Answering Pipeline**

User: "I have a FastAPI service that answers questions using GPT-4. Some questions are easy, some are hard. I want to save cost by using a small model for easy questions and only calling GPT-4 for hard ones."

Approach:
1. Add logprob extraction to the small model's API call.
2. Compute token-level entropy from logprobs on the candidate answer.
3. Route based on entropy threshold calibrated on a sample of production queries.

Output:
```python
import math
from openai import OpenAI

client = OpenAI()

def compute_token_entropy(logprobs: list[dict]) -> float:
    """Average entropy across tokens from OpenAI logprobs."""
    entropies = []
    for token_lp in logprobs:
        top = token_lp.get("top_logprobs", [])
        if not top:
            continue
        probs = [math.exp(lp["logprob"]) for lp in top]
        total = sum(probs)
        probs = [p / total for p in probs]
        entropy = -sum(p * math.log(p + 1e-12) for p in probs)
        entropies.append(entropy)
    return sum(entropies) / len(entropies) if entropies else 0.0

ENTROPY_THRESHOLD = 1.8  # Calibrate on your data

async def answer_question(question: str) -> dict:
    # Fast path: small model
    fast_resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": question}],
        logprobs=True,
        top_logprobs=5,
    )
    entropy = compute_token_entropy(
        fast_resp.choices[0].logprobs.content
    )

    if entropy < ENTROPY_THRESHOLD:
        return {
            "answer": fast_resp.choices[0].message.content,
            "route": "fast",
            "entropy": entropy,
        }

    # Slow path: large model with chain-of-thought
    slow_resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "Think step by step."},
            {"role": "user", "content": question},
        ],
    )
    return {
        "answer": slow_resp.choices[0].message.content,
        "route": "slow",
        "entropy": entropy,
    }
```

**Example 2: Confidence-Gated Tool Use in an Agent**

User: "My LangChain agent calls a search tool on every question, even when it already knows the answer. Make it only search when it's actually unsure."

Approach:
1. Before the tool call, generate a draft answer with sampling (N=5).
2. Measure agreement across samples.
3. Only invoke the search tool when agreement is low.

Output:
```python
from collections import Counter

def measure_sampling_consistency(
    llm, prompt: str, n_samples: int = 5
) -> tuple[str, float]:
    """Generate n samples and measure agreement."""
    responses = []
    for _ in range(n_samples):
        resp = llm.invoke(prompt, temperature=0.7)
        responses.append(resp.strip().lower())

    # Cluster by exact match (use embeddings for open-ended)
    counts = Counter(responses)
    most_common_answer, top_count = counts.most_common(1)[0]
    consistency = top_count / n_samples  # 1.0 = perfect agreement
    return most_common_answer, consistency

CONSISTENCY_THRESHOLD = 0.6  # Below this, use tool

def confidence_gated_agent_step(llm, question: str, search_tool):
    draft_answer, consistency = measure_sampling_consistency(
        llm, question
    )

    if consistency >= CONSISTENCY_THRESHOLD:
        return {
            "answer": draft_answer,
            "used_tool": False,
            "consistency": consistency,
        }

    # Low confidence -- invoke search
    search_results = search_tool.run(question)
    grounded_prompt = (
        f"Using this context:\n{search_results}\n\n"
        f"Answer: {question}"
    )
    answer = llm.invoke(grounded_prompt, temperature=0.0)
    return {
        "answer": answer,
        "used_tool": True,
        "consistency": consistency,
    }
```

**Example 3: Self-Correction Loop Triggered by Uncertainty**

User: "Add a self-correction mechanism to my code generation pipeline. If the model is uncertain about its code, it should review and fix it before returning."

Approach:
1. Generate code, then ask the model to rate its own confidence.
2. If confidence is below threshold, enter a critique-and-revise loop (max 2 iterations).
3. Return the final version with metadata about iterations used.

Output:
```python
import json

def generate_with_self_correction(
    client, task: str, max_revisions: int = 2
) -> dict:
    code = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": f"Write code for: {task}"}],
    ).choices[0].message.content

    for i in range(max_revisions):
        # Ask model to assess its own uncertainty
        assessment = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "user", "content": f"Write code for: {task}"},
                {"role": "assistant", "content": code},
                {"role": "user", "content": (
                    "Rate your confidence in this code from 0-10. "
                    "List any bugs or issues. Respond as JSON: "
                    '{"confidence": N, "issues": [...]}'
                )},
            ],
        ).choices[0].message.content

        try:
            parsed = json.loads(assessment)
        except json.JSONDecodeError:
            break

        if parsed.get("confidence", 10) >= 7:
            return {"code": code, "revisions": i, "assessment": parsed}

        # Low confidence -- revise
        issues = parsed.get("issues", [])
        code = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "user", "content": f"Write code for: {task}"},
                {"role": "assistant", "content": code},
                {"role": "user", "content": (
                    f"Fix these issues: {issues}. "
                    "Return only the corrected code."
                )},
            ],
        ).choices[0].message.content

    return {"code": code, "revisions": max_revisions, "assessment": parsed}
```

## Best Practices

- **Do** calibrate thresholds on a held-out dataset that matches your production distribution. A threshold tuned on academic benchmarks will not transfer to your domain.
- **Do** combine cheap and expensive uncertainty signals in a cascade: use token entropy for fast rejection, then semantic entropy only for borderline cases.
- **Do** cap self-correction iterations (2-3 max). Unbounded loops waste compute and can degrade quality as the model over-corrects.
- **Do** log uncertainty scores alongside responses in production. This gives you a free monitoring signal for data drift and regression detection.
- **Avoid** using raw softmax probabilities as calibrated confidence. LLMs are notoriously miscalibrated -- always validate on your calibration set.
- **Avoid** setting a single global threshold across heterogeneous tasks. Different question types have different entropy baselines; use per-task or per-category thresholds.
- **Avoid** routing 100% of uncertain queries to expensive paths without a cost budget. Add a rate limiter or sampling mechanism to control slow-path utilization.

## Error Handling

- **Logprobs unavailable**: Some APIs or models don't expose logprobs. Fall back to sampling consistency (generate N outputs, measure agreement) which works with any black-box model.
- **Threshold miscalibration**: If the fast path returns too many wrong answers, lower the entropy threshold. If the slow path is called too often, raise it. Monitor the fast-path accuracy rate continuously.
- **Self-correction loops that diverge**: If a revision is worse than the original, keep the original. Compare uncertainty before and after revision; if uncertainty increased, revert.
- **Conformal sets that are too large**: Prediction sets that include most of the output space are uninformative. This usually means the model is poorly suited to the task or the calibration set is too small. Increase calibration data or use a more capable model.
- **Semantic clustering failures**: Embedding-based clustering can group semantically different answers together. Validate clusters manually on a sample. Consider NLI-based equivalence checking for higher precision.

## Limitations

- Token-level entropy only captures surface-level uncertainty. A model can produce low-entropy tokens that form a confidently wrong answer (hallucination with high confidence). Semantic entropy partially addresses this but at higher cost.
- Self-assessed confidence ("rate your confidence 0-10") is unreliable in isolation because LLMs exhibit systematic overconfidence. Always cross-validate with sampling-based measures.
- Conformal prediction requires exchangeability between calibration and test data. Distribution shift invalidates coverage guarantees. Periodic recalibration is mandatory.
- Uncertainty-gated tool use adds latency to the decision path (multiple samples before acting). This is unsuitable for ultra-low-latency applications without aggressive caching.
- The approach assumes uncertainty is actionable -- that there exists a meaningful intervention (tool call, deeper reasoning, abstention) for uncertain cases. If no good fallback exists, measuring uncertainty is overhead without benefit.

## Reference

Zhang, J., Cui, W., Li, Z., Huang, L., & Malin, B. (2026). *From Passive Metric to Active Signal: The Evolving Role of Uncertainty Quantification in Large Language Models.* arXiv:2601.15690v1. Look for: Table 1 (taxonomy of UQ methods), Figure 2 (the passive-to-active evolution diagram), Section 4 (design patterns for uncertainty as control signal), and Section 5 (RL integration patterns).