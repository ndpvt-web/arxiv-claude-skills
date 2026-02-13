---
name: "martingale-foresight-sampling-principled"
description: "Implement Martingale Foresight Sampling (MFS) for principled LLM decoding with lookahead search. Replaces heuristic beam search and tree-of-thought pruning with mathematically grounded path valuation, pruning, and early stopping. Trigger phrases: 'implement foresight sampling', 'martingale decoding', 'principled beam search for LLM', 'optimal decoding strategy', 'lookahead search with convergence guarantees', 'inference-time reasoning optimization'"
---

# Martingale Foresight Sampling: Principled Inference-Time LLM Decoding

This skill enables Claude to implement and apply **Martingale Foresight Sampling (MFS)**, a decoding framework that treats LLM reasoning-path search as a stochastic optimization problem grounded in martingale theory. Instead of relying on ad-hoc heuristics for scoring, pruning, and stopping during multi-path generation, MFS uses three mathematical principles — the Doob Decomposition Theorem for step valuation, Optional Stopping Theory for path pruning, and the Martingale Convergence Theorem for adaptive early stopping — to achieve higher accuracy with fewer compute cycles than methods like Tree-of-Thoughts, MCTS, or heuristic foresight decoding.

## When to Use

- When the user asks to build a multi-path decoding or search system for LLM inference (e.g., best-of-N sampling, beam search, tree search) and wants principled scoring and pruning rather than hand-tuned heuristics
- When implementing inference-time compute scaling where an LLM generates and evaluates multiple reasoning trajectories before committing to an answer
- When the user needs to decide *when to stop exploring* during lookahead search — the classic early-stopping problem in multi-step generation
- When building a reasoning pipeline (chain-of-thought, multi-step math, logic problems) that must balance accuracy against compute budget
- When refactoring an existing Tree-of-Thoughts or MCTS-based decoding system to reduce FLOPS while maintaining or improving accuracy
- When the user asks about optimal stopping, submartingale processes, or Doob decomposition in the context of sequential decision-making over LLM outputs

## Key Technique

**The core insight:** Model the quality of a partial reasoning path as a *submartingale* — a stochastic process whose expected future value is at least as large as its current value. Under this framing, each generation step either improves the path or adds noise. The Doob Decomposition Theorem separates any such process into a *predictable drift* component (the genuine improvement a token contributes) and a *martingale noise* component (random fluctuation). MFS uses only the predictable drift as the step-value signal, filtering out noise that misleads heuristic scorers.

**Path pruning without magic thresholds:** For each candidate path, MFS tracks a *deficit process* — how far behind the current best path it falls. Optional Stopping Theory guarantees that once a path's deficit exceeds an adaptive threshold (calibrated from the running mean and standard deviation of all candidate scores), the probability of that path catching up is provably bounded. This replaces arbitrary top-k or score-cutoff pruning with a statistically justified elimination rule.

**Knowing when to stop looking ahead:** The Martingale Convergence Theorem states that a bounded submartingale must converge to a finite limit. MFS monitors the maximum predictable advantage across all surviving candidates; when it drops below a small epsilon (typically 1e-6), the quality process has plateaued and further lookahead is wasteful. At that point, standard autoregressive decoding finishes the generation. Ablation studies show this early stopping *improves* accuracy (by ~1-2%) while cutting FLOPS by ~37%, because continued unguided exploration introduces noise.

## Step-by-Step Workflow

1. **Define the quality function F_t.** For a partial path at step t, compute `F_t = p_theta(successful_completion | prompt, path_so_far)` — the model's estimated probability of reaching a correct answer from the current state. Estimate this via N short rollouts (typically N=8) from the current position, scoring each rollout's final answer.

2. **Initialize the candidate beam.** Sample M candidate next-tokens (typically M=8) from the LLM's distribution at temperature tau (typically 0.7). Each candidate defines a branch in the search tree.

3. **Simulate rollouts for each candidate.** For each of the M candidates, generate N continuation rollouts of moderate depth. Record the quality score F_t for each rollout. This yields an M x N matrix of future quality estimates.

4. **Compute the predictable advantage via Doob Decomposition.** For each candidate token a_t, calculate `V(a_t) = E[F_t - F_{t-1} | history, a_t]` by averaging the quality deltas across the N rollouts. This isolates the genuine expected improvement from noise. Rank candidates by V(a_t).

5. **Apply deficit-based pruning via Optional Stopping Theory.** For each surviving path i, compute the deficit `D_t^i = F_t^best - F_t^i`. Calculate the adaptive pruning threshold `c_prune(t) = mean(F_t) + lambda_1 * std(F_t)` where lambda_1 is a sensitivity parameter (0.6-1.0, task-dependent). Eliminate any path whose deficit exceeds c_prune(t).

6. **Check the convergence stopping criterion.** If `max(V(a_t)) <= epsilon_stop` (typically 1e-6) across all surviving candidates, halt foresight sampling — the quality process has converged and further lookahead adds noise, not signal.

7. **Advance the beam.** Select the top-scoring surviving candidates as the new beam. Repeat from step 3 for the next reasoning step.

8. **Complete via standard decoding.** Once foresight sampling stops (either by convergence or reaching a step budget), finish each surviving path using standard autoregressive generation.

9. **Aggregate final answers.** Apply majority voting across the completed paths to select the final answer. If answers are non-discrete (e.g., code), select the path with highest cumulative quality score.

10. **Tune hyperparameters per task family.** Use lambda_1=0.8 for arithmetic reasoning (GSM8K-style), lambda_1=0.6-0.7 for harder math and logic tasks, M=8 and N=8 as robust defaults, and epsilon_stop=1e-6 universally.

## Concrete Examples

**Example 1: Building an MFS decoding loop for a math reasoning API**

User: "I have a vLLM-served LLaMA-3.1-8B model. Build me a decoding wrapper that uses martingale foresight sampling to solve math word problems more accurately than standard sampling."

Approach:
1. Create a `MFSDecoder` class that wraps the vLLM client, parameterized by M (beam width), N (rollouts per candidate), lambda_1 (pruning sensitivity), epsilon_stop (convergence threshold), and tau (temperature).
2. Implement `estimate_quality(prompt, partial_path)` — generate N completions from the current state, extract final numerical answers, compute the fraction that match the most common answer as the quality score F_t.
3. Implement `compute_advantage(candidates, quality_scores)` — for each candidate token, average the quality delta (F_t - F_{t-1}) across its N rollouts to get V(a_t).
4. Implement `prune_paths(paths, scores)` — compute per-path deficit against the best, compute the adaptive threshold from running score statistics, eliminate paths exceeding the threshold.
5. Implement the main loop: generate candidates, rollout, value, prune, check convergence, advance or stop.
6. Return the majority-vote answer from surviving completed paths.

Output structure:
```python
class MFSDecoder:
    def __init__(self, client, M=8, N=8, lambda_1=0.8, epsilon_stop=1e-6, tau=0.7):
        self.client = client
        self.M = M  # beam width
        self.N = N  # rollouts per candidate
        self.lambda_1 = lambda_1  # pruning sensitivity
        self.epsilon_stop = epsilon_stop
        self.tau = tau

    def estimate_quality(self, prompt, partial_path, n_rollouts):
        """Generate n_rollouts completions; return fraction agreeing with mode answer."""
        completions = self.client.generate(
            prompt + partial_path, n=n_rollouts, temperature=self.tau
        )
        answers = [extract_answer(c) for c in completions]
        if not answers:
            return 0.0
        mode_answer = most_common(answers)
        return sum(1 for a in answers if a == mode_answer) / len(answers)

    def compute_advantage(self, prev_quality, candidate_qualities):
        """Doob decomposition: predictable drift = E[F_t - F_{t-1}]."""
        return {tok: np.mean(scores) - prev_quality
                for tok, scores in candidate_qualities.items()}

    def should_prune(self, path_score, best_score, mean_score, std_score):
        """Optional stopping: prune if deficit exceeds adaptive threshold."""
        deficit = best_score - path_score
        threshold = mean_score + self.lambda_1 * std_score
        return deficit >= threshold

    def has_converged(self, advantages):
        """Martingale convergence: stop if max advantage below epsilon."""
        return max(advantages.values()) <= self.epsilon_stop

    def decode(self, prompt):
        beam = [("", 0.0)]  # (partial_path, quality)
        while True:
            all_candidates = {}
            for path, prev_q in beam:
                tokens = self.client.sample_next_tokens(prompt + path, n=self.M, temperature=self.tau)
                for tok in tokens:
                    qualities = [self.estimate_quality(prompt, path + tok, 1) for _ in range(self.N)]
                    all_candidates[(path, tok)] = qualities

            advantages = {}
            for (path, tok), quals in all_candidates.items():
                prev_q = dict(beam)[path]
                advantages[(path, tok)] = np.mean(quals) - prev_q

            if self.has_converged(advantages):
                break  # quality plateau reached

            # Score, prune, and select top-M surviving paths
            scored = [(p + t, np.mean(q)) for (p, t), q in all_candidates.items()]
            best = max(s for _, s in scored)
            mu, sigma = np.mean([s for _, s in scored]), np.std([s for _, s in scored])
            beam = [(p, s) for p, s in scored if not self.should_prune(s, best, mu, sigma)]
            beam = sorted(beam, key=lambda x: -x[1])[:self.M]

        # Complete remaining paths and majority-vote
        completions = [self.client.generate(prompt + p, n=1)[0] for p, _ in beam]
        return majority_vote([extract_answer(c) for c in completions])
```

**Example 2: Adding principled early stopping to an existing best-of-N pipeline**

User: "I have a best-of-N sampling pipeline that generates 64 candidates and picks the best by reward model score. It's too expensive. Can you add martingale-based early stopping?"

Approach:
1. Treat the reward model score of each partial generation as the quality process F_t.
2. After each batch of candidates is scored, compute the maximum predictable advantage: `max_advantage = max(batch_mean_scores) - prev_round_best`.
3. Track a running window of max_advantage values. When max_advantage falls below epsilon_stop for 2 consecutive rounds, halt generation — further candidates won't meaningfully improve the best score.
4. Additionally, apply deficit-based pruning: after each round, drop candidates whose reward-model deficit from the leader exceeds `mean + lambda_1 * std` of current scores, reducing the pool progressively.

Output:
```python
def early_stopping_best_of_n(generate_fn, score_fn, prompt, max_n=64, batch=8,
                              lambda_1=0.8, epsilon_stop=1e-6):
    candidates = []
    prev_best = 0.0
    converged_rounds = 0

    for round_idx in range(max_n // batch):
        new = generate_fn(prompt, n=batch)
        scores = [score_fn(c) for c in new]
        candidates.extend(zip(new, scores))

        current_best = max(s for _, s in candidates)
        advantage = current_best - prev_best

        if advantage <= epsilon_stop:
            converged_rounds += 1
            if converged_rounds >= 2:
                break  # Martingale convergence: quality has plateaued
        else:
            converged_rounds = 0

        # Deficit-based pruning of accumulated pool
        mu = np.mean([s for _, s in candidates])
        sigma = np.std([s for _, s in candidates])
        threshold = mu + lambda_1 * sigma
        candidates = [(c, s) for c, s in candidates
                       if (current_best - s) < threshold]

        prev_best = current_best

    return max(candidates, key=lambda x: x[1])[0]
```

This typically terminates after 16-32 candidates instead of 64, cutting compute by 50-75% with negligible accuracy loss.

**Example 3: Principled pruning in a code-generation tree search**

User: "I'm building a tree search for code generation where I expand partial programs and test them. How should I prune bad branches?"

Approach:
1. Define F_t as the fraction of unit tests passed by the best completion from each partial program at depth t.
2. At each expansion step, generate M candidate continuations per surviving branch and run N quick completions of each to estimate F_t.
3. Compute the deficit of each branch against the current leader. Prune branches where `deficit >= mean(F_t) + 0.7 * std(F_t)`.
4. Stop expanding when the maximum Doob advantage (expected test-pass improvement from the next expansion) drops below 1e-6.
5. Return the branch whose completions pass the most tests.

## Best Practices

**Do:**
- Use the quality function that most directly measures task success (answer agreement for math, test-pass rate for code, reward-model score for open-ended generation). The theory is agnostic to the specific quality metric.
- Start with M=8, N=8, lambda_1=0.8, epsilon_stop=1e-6 as defaults and only tune if needed. These are robust across diverse reasoning benchmarks.
- Monitor the convergence criterion as a diagnostic — if foresight sampling almost never triggers early stopping, lambda_1 may be too aggressive (pruning too many paths before convergence can be detected).
- Log the deficit and advantage values during development. Monotonically increasing deficits confirm the submartingale assumption holds for your task.

**Avoid:**
- Do not skip the pruning step to "keep more options open." Unpruned exploration introduces noise that degrades accuracy — ablations show ~1-2% accuracy *drop* without pruning.
- Do not set epsilon_stop too large (e.g., 0.1). This causes premature stopping before genuine convergence, losing accuracy. The 1e-6 default is intentionally conservative.
- Do not use greedy (temperature=0) sampling for rollouts. MFS requires stochastic rollouts to estimate the quality distribution; tau=0.7 provides good coverage without excessive randomness.
- Do not apply MFS to single-token classification tasks. The overhead of multi-step foresight only pays off for multi-step reasoning (5+ tokens of non-trivial generation).

## Error Handling

- **All rollouts produce identical outputs:** This collapses the quality estimate variance to zero, making pruning thresholds meaningless. Increase temperature or add top-p sampling to restore diversity. If the model is highly confident, this is actually a signal that foresight sampling is unnecessary for this input — fall back to standard decoding.
- **Quality scores are non-monotonic (not a submartingale):** The theoretical guarantees weaken. This can happen with poorly calibrated models or adversarial inputs. Mitigation: apply a rolling-average smoother to F_t before computing advantages, or switch to a better quality estimator.
- **Pruning eliminates all but one path in the first step:** lambda_1 is too low for this task's score distribution. Increase lambda_1 toward 1.0 or set a minimum beam size floor (e.g., keep at least 2 paths alive).
- **Convergence never triggers within compute budget:** The quality process may not be bounded for this task/model combination. Set a hard step limit as a fallback (e.g., max 10 foresight steps) and switch to standard decoding.
- **Memory pressure from M x N rollouts:** Reduce N first (rollout count), as reducing M (beam width) has a larger impact on final accuracy. Batching rollouts through vLLM or similar serving frameworks amortizes the cost.

## Limitations

- MFS assumes the quality process is a submartingale (expected quality increases with more reasoning steps). This holds for well-trained models on reasoning tasks but may fail for creative or open-ended generation where quality is subjective and non-monotonic.
- The compute overhead of M x N rollouts per step is substantial. MFS is most beneficial when the cost of a wrong answer exceeds the cost of extra inference (e.g., high-stakes reasoning, not casual chat).
- Quality estimation via rollout agreement is noisy for tasks without clear discrete answers. Code generation with test suites works well; open-ended essay writing does not.
- The framework requires access to the model's sampling API (generating multiple completions per call). It cannot be applied to black-box APIs that only return a single completion without repeated calls.
- Hyperparameter lambda_1 is somewhat task-dependent (0.6-1.0 range). While the defaults are robust, optimal performance on a new task family may require a small tuning sweep.

## Reference

**Paper:** "Martingale Foresight Sampling: A Principled Approach to Inference-Time LLM Decoding" — Li, He, Tian, Wen, Li (2026). arXiv: [2601.15482](https://arxiv.org/abs/2601.15482v1). Look for: Section 3 (theoretical framework with Doob decomposition, optional stopping, and convergence theorem), Algorithm 1 (full pseudocode), Table 1 (benchmark results vs. baselines), and the ablation study showing that removing early stopping *hurts* both accuracy and efficiency.