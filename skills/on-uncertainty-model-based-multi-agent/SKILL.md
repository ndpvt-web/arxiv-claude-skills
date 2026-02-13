---
name: "on-uncertainty-model-based-multi-agent"
description: "Apply entropy-based uncertainty analysis to multi-agent LLM systems. Diagnose when multi-agent setups hurt performance, select the best solution from multiple agent runs using the Entropy Judger technique, and design uncertainty-aware agent topologies. Trigger phrases: 'analyze multi-agent uncertainty', 'entropy judger for agent selection', 'should I use multiple agents or one', 'select best answer from agent runs', 'diagnose multi-agent failure', 'optimize agent topology with entropy'"
---

# Entropy-Based Uncertainty Analysis for Multi-Agent LLM Systems

This skill enables Claude to apply the Entropy Judger framework from Zhao et al. (2026) to diagnose, evaluate, and improve multi-agent LLM systems. The core technique uses token-level, trajectory-level, and round-level entropy measurements to predict whether a multi-agent configuration will outperform a single agent, select the best solution from multiple runs (pass@k), and identify failure modes early in agent interactions. A critical finding: a single agent outperforms multi-agent systems in ~43% of cases, and first-round entropy dynamics are the strongest predictor of final correctness.

## When to Use

- When the user is building a multi-agent system and wants to know whether multiple agents will actually help versus a single agent
- When the user has multiple candidate outputs from agent runs and needs to select the best one without ground truth
- When debugging why a multi-agent pipeline (debate, sequential, centralized) produces worse answers than expected
- When designing agent topologies (sequential, centralized, decentralized, debate, hybrid) and choosing which fits a task
- When the user wants to add an entropy-based quality gate to filter or rank LLM outputs before returning them
- When evaluating whether to add more agents or communication rounds to an existing system

## Key Technique

**The Entropy Judger** reframes multi-agent solution selection as a supervised classification problem: given entropy traces from an agent run, predict whether the output is correct. For each candidate solution in a pass@k set, extract a feature vector of entropy statistics across tokens, reasoning trajectories, and communication rounds. An ensemble of XGBoost and LightGBM classifiers scores each candidate f(x) in [0,1], and the candidate with the highest predicted correctness probability is selected: `best = argmax over k candidates of f(x_k)`.

The method rests on three empirically validated observations. **Certainty Preference**: reducing entropy at any stage for any agent correlates with correct solutions -- high peak entropy is universally harmful. **Base Uncertainty**: the foundation model's intrinsic entropy during problem-solving directly bounds multi-agent performance -- if the base model is highly uncertain (entropy > 100 nats across the sequence), adding agents rarely helps. **Task Awareness**: entropy dynamics play different roles across task types -- math tasks show strong entropy-correctness correlation (rho up to -0.92), while code generation tasks show weaker but still useful signal.

The practical implication is that first-round entropy dominates prediction. Round-1 maximum agent entropy is consistently among the top predictive features. Extended deliberation (5+ rounds) rarely overcomes poor initial uncertainty. This means you can make go/no-go decisions about an agent run very early.

## Step-by-Step Workflow

1. **Define the agent topology.** Choose from: single agent, sequential (planner -> solver -> critic -> judger), centralized (domain agents + coordinator), decentralized (agents with feedback loops), full-decentralized (complete graph), debate (multi-agent voting), or hybrid (two-layer with context propagation). Default to single agent unless the task demonstrably benefits from decomposition.

2. **Instrument entropy collection.** For each LLM call in the pipeline, capture the raw logits or log-probabilities for every generated token. Compute per-token entropy as `H(t) = -sum(p(v) * log(p(v)))` over the vocabulary dimension. Store these as arrays alongside the generated text.

3. **Compute trajectory-level features.** For each agent's response, aggregate token entropies into: total_entropy, mean_entropy, max_entropy, min_entropy, median_entropy, std_entropy, q1/q3 entropy, coefficient of variation, IQR, skewness, and tail weight. This yields ~13 core features per agent response.

4. **Compute round-level transition features.** Across communication rounds, track: first-to-last round entropy difference, entropy slope (linear trend), volatility (std of per-round means), and per-round agent spread (max - min entropy across agents in a round). Focus especially on round-1 metrics -- they carry the most predictive signal.

5. **Extract base-model reference entropy.** Run the same prompt through a single agent with the same base model. Compute its entropy features as a baseline. Calculate deltas and ratios: `delta_mean = mas_mean_entropy - base_mean_entropy`, `ratio_total = mas_total_entropy / base_total_entropy`. These ~17 features capture whether the multi-agent setup is increasing or decreasing uncertainty relative to the base model.

6. **Build the feature vector.** Concatenate sample-level (~13), shape (~7), round-agent (~100+), cross-round dynamics (~20), answer-token (~12), agent-level (~10), base-model comparison (~17), and computational metrics (~15 for time/tokens) into a single vector per candidate solution. The full set is 245 features.

7. **Train or apply the Entropy Judger classifier.** If you have labeled data (correct/incorrect for past runs), train an XGBoost + LightGBM ensemble on the feature vectors. Use 5-fold cross-validation. If no labeled data is available, use a heuristic: select the candidate with the lowest mean entropy and lowest round-1 max entropy -- the Certainty Preference principle.

8. **Select from pass@k candidates.** Run the multi-agent system k times on the same input. Score each candidate with the trained classifier (or heuristic). Return the candidate with highest predicted correctness: `best = argmax_k f(x_k)`.

9. **Apply early-exit logic.** After round 1, check if max agent entropy exceeds your task-specific threshold (e.g., >100 nats for math tasks). If so, consider falling back to a single-agent solution rather than continuing multi-round deliberation, which is unlikely to recover.

10. **Iterate on topology.** If the Entropy Judger consistently scores single-agent runs higher than multi-agent runs for your task, simplify the topology. Multi-agent systems add value primarily when base model uncertainty is moderate and the task benefits from decomposition (e.g., complex multi-step reasoning).

## Concrete Examples

**Example 1: Selecting the best answer from multiple agent debate runs**

```
User: I'm running a 3-agent debate system on math problems and generating
5 candidate answers per problem. How do I pick the best one?

Approach:
1. For each of the 5 runs, collect per-token entropy from every agent
   at every debate round.
2. Compute per-run feature vectors:
   - mean_entropy across all agents and rounds
   - max_entropy in round 1 (strongest single predictor)
   - entropy_slope across rounds (is uncertainty decreasing?)
   - std_entropy across agents in final round (consensus measure)
3. Rank candidates by: lowest round-1 max entropy first, then by
   most negative entropy slope (uncertainty decreased most).
4. Return the top-ranked candidate.

Output (selection logic in Python):
```python
import numpy as np

def select_best_candidate(candidates):
    """Select best from pass@k using entropy heuristic.

    Each candidate is a dict with:
      - 'answer': str
      - 'round_entropies': list of list of float
        (rounds x agents, each entry is mean token entropy)
    """
    scores = []
    for c in candidates:
        r1_max = max(c['round_entropies'][0])  # round-1 max agent entropy
        all_means = [np.mean(r) for r in c['round_entropies']]
        slope = all_means[-1] - all_means[0]  # negative = uncertainty decreased
        final_std = np.std(c['round_entropies'][-1])  # agent consensus
        # Lower is better for all three components
        score = r1_max + 0.5 * slope + 0.3 * final_std
        scores.append(score)
    return candidates[np.argmin(scores)]['answer']
```

**Example 2: Deciding whether to use multi-agent or single-agent**

```
User: I have a code generation task. Should I use a multi-agent system
or just a single agent?

Approach:
1. Run 20 sample problems through a single agent, collecting entropy.
2. Compute the base model's average mean_entropy across samples.
3. If mean_entropy > 80 nats: the base model is highly uncertain.
   Multi-agent is unlikely to help (Base Uncertainty principle).
   Invest in a better base model or better prompting first.
4. If mean_entropy is moderate (30-80): multi-agent may help.
   Try a sequential (planner -> coder -> reviewer) topology.
5. If mean_entropy < 30: the model is already confident.
   Single agent is likely sufficient. Multi-agent adds latency
   without meaningful accuracy gain.

Decision framework:
```python
def should_use_multi_agent(base_entropies: list[float]) -> str:
    avg = np.mean(base_entropies)
    if avg > 80:
        return "single-agent (base model too uncertain for MAS benefit)"
    elif avg > 30:
        return "multi-agent (moderate uncertainty, MAS can help)"
    else:
        return "single-agent (model already confident)"
```

**Example 3: Early-exit from a failing multi-agent pipeline**

```
User: My sequential agent pipeline (planner -> solver -> critic -> judger)
sometimes wastes tokens on problems it gets wrong anyway. Can I detect
failure early?

Approach:
1. After the planner (round 1) generates its output, compute its
   mean token entropy and max token entropy.
2. Compare against historical thresholds from your labeled data:
   - If round-1 max_entropy > threshold_high: abort and return
     single-agent fallback (saves 3 agent calls).
   - If round-1 mean_entropy < threshold_low: skip critic/judger,
     the solver's answer is likely already correct.
3. Only continue the full pipeline for moderate-entropy cases
   where multi-agent deliberation adds value.

Implementation:
```python
def early_exit_check(round1_entropies: list[float],
                     high_threshold: float = 120.0,
                     low_threshold: float = 15.0) -> str:
    max_ent = max(round1_entropies)
    mean_ent = np.mean(round1_entropies)
    if max_ent > high_threshold:
        return "abort"   # Fall back to single agent
    elif mean_ent < low_threshold:
        return "accept"  # Skip remaining agents, answer is confident
    else:
        return "continue"  # Proceed with full pipeline
```

## Best Practices

- **Do** measure entropy from the actual token logits/log-probs, not from proxy signals like response length or verbosity. Entropy computed from the probability distribution over the vocabulary is the true uncertainty signal.
- **Do** weight round-1 features most heavily in any selection or diagnostic scheme. The paper shows first-round entropy is consistently the top predictor across all topologies and tasks.
- **Do** always benchmark against a single-agent baseline before deploying a multi-agent system. In 43% of cases, a single agent wins -- you need evidence that your multi-agent setup beats it.
- **Do** use task-specific entropy thresholds. Math reasoning tasks show tight entropy-correctness correlation (rho ~ -0.92), while code generation is weaker. Calibrate thresholds per task domain.
- **Avoid** adding more communication rounds hoping to "converge" when round-1 entropy is already high. The paper shows extended deliberation rarely recovers from poor initial uncertainty.
- **Avoid** using multi-agent systems with base models that show very high intrinsic entropy (>100 nats average). Fix the base model or prompting first -- multi-agent amplifies base model quality, it does not compensate for it.

## Error Handling

- **No logit access**: If using a closed API (e.g., GPT-4, Claude) that doesn't expose token logits, approximate entropy using: (a) log-probabilities if available via the API, (b) self-consistency across k samples as a proxy for uncertainty, or (c) ask the model to self-rate confidence and treat low-confidence as high entropy. The heuristic degrades gracefully.
- **Feature extraction fails for short responses**: Very short outputs (< 5 tokens) produce unstable entropy statistics. Pad the feature vector with the sample mean or flag these as low-confidence and default to the single-agent answer.
- **Classifier overfits to one task domain**: The Entropy Judger should be trained per-task or per-domain. A classifier trained on math problems will not transfer well to code generation. Use the Task Awareness principle -- always retrain or recalibrate when switching domains.
- **Inconsistent entropy scales across models**: Different LLMs have different vocabulary sizes, which affects raw entropy magnitude. Normalize entropy features by `log(vocab_size)` to make them comparable across models.

## Limitations

- Requires access to token-level logits or log-probabilities. Many commercial APIs do not expose these, limiting the technique to open-weight models or APIs with logprob support.
- The supervised Entropy Judger (XGBoost + LightGBM ensemble) requires labeled training data -- you need a set of (input, agent_run, correct/incorrect) examples to train on. The heuristic fallback (lowest entropy wins) works without labels but is less accurate.
- Entropy is a necessary but not sufficient signal. A model can be confidently wrong (low entropy, incorrect answer). The technique improves selection accuracy, it does not guarantee correctness.
- The 245-feature set was validated on math (GSM8K, AIME, MATH-500), reasoning (MMLU), and code (HumanEval) benchmarks. Performance on creative, open-ended, or subjective tasks is unvalidated.
- The approach optimizes selection from existing candidates. It does not improve the generation quality of any individual agent -- it picks the best from what is already produced.

## Reference

Zhao, Y., Chen, S., & Su, N. (2026). "On the Uncertainty of Large Language Model-Based Multi-Agent Systems." arXiv:2602.04234v2. [https://arxiv.org/abs/2602.04234v2](https://arxiv.org/abs/2602.04234v2)

Look for: Section 4 (Entropy Judger algorithm and feature taxonomy), Table 1 (cross-validation accuracy by feature group), and Section 3 (the three key observations on Certainty Preference, Base Uncertainty, and Task Awareness). Source code: [https://github.com/AgenticFinLab/multiagent-entropy](https://github.com/AgenticFinLab/multiagent-entropy)