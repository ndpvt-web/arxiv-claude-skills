---
name: "markovscale-optimal-sequential-scaling"
description: "Implement MarkovScale's principled sequential scaling for LLM inference pipelines. Models retry/refinement loops as a two-state Markov chain to compute optimal stopping points, accuracy bounds, and cost-efficient sampling budgets. Use when: 'optimize my LLM retry loop', 'how many retries should I use', 'sequential scaling for inference', 'Markov chain for LLM sampling', 'optimal stopping for LLM calls', 'reduce inference cost while maintaining accuracy'."
---

# MarkovScale: Optimal Sequential Scaling at Inference Time

This skill enables Claude to design and implement principled sequential scaling systems for LLM inference pipelines. Instead of using ad-hoc retry counts or heuristic refinement loops, it applies MarkovScale's framework: model the retry/refinement process as a two-state Markov chain (Correct vs. Wrong), estimate transition probabilities from a small calibration set, then compute closed-form accuracy bounds and optimal stopping criteria. This replaces guesswork with mathematically grounded decisions about when additional LLM calls help, when they're neutral, and when they actively degrade quality.

## When to Use

- When the user is building an LLM pipeline with retry or self-refinement loops and asks how many iterations to run
- When implementing Best-of-N sampling, majority voting, or sequential refinement and wanting to optimize the compute budget
- When the user asks to reduce inference costs on a pipeline that makes repeated LLM calls for the same query
- When designing an evaluation harness that needs to determine if sequential attempts are actually improving accuracy
- When building an agentic system where a model retries failed tool calls or code generation and the user wants principled stopping criteria
- When comparing parallel scaling (independent samples) vs. sequential scaling (iterative refinement) strategies

## Key Technique

**The Two-State Markov Model.** MarkovScale treats each round of sequential LLM inference as a transition in a two-state Markov chain. The states are **Correct (C)** and **Wrong (W)**. Four transition probabilities govern the system: `p_cc` (correct stays correct), `p_cw` (correct flips to wrong), `p_wc` (wrong flips to correct), and `p_ww` (wrong stays wrong). Because each row sums to 1, only two free parameters matter: `p_cw` (the "corruption rate") and `p_wc` (the "repair rate"). This compact model captures the essential dynamics of sequential refinement.

**Closed-Form Bounds.** Given an initial accuracy `a_0` and the transition probabilities, accuracy after `n` rounds of sequential scaling can be expressed in closed form using the Markov chain's stationary distribution and convergence rate. The critical insight is the relationship between `p_wc` and `p_cw`: when `p_wc > p_cw` (the model is more likely to fix errors than introduce them), sequential scaling provably improves accuracy and converges to an upper bound of `p_wc / (p_wc + p_cw)`. When `p_wc = p_cw`, additional rounds have zero net effect. When `p_wc < p_cw`, further iterations degrade accuracy. This gives an analytical stopping criterion: continue only while the expected marginal accuracy gain exceeds a cost threshold.

**Practical Calibration.** MarkovScale estimates the transition probabilities from a small calibration set (typically 20-50 labeled examples). Run two sequential rounds on calibration data, classify each round's output as correct or wrong, then count transitions to estimate `p_wc` and `p_cw`. From these estimates, compute the optimal number of rounds for any target accuracy-cost tradeoff. This makes the system self-tuning: it adapts to different models, prompts, and task difficulties without manual configuration.

## Step-by-Step Workflow

1. **Define the sequential scaling loop.** Identify the component in the pipeline that calls the LLM multiple times for the same input -- whether it's a retry-on-failure loop, a self-refinement chain, or a generate-then-verify cycle. Isolate the function so each "round" produces a single candidate answer.

2. **Prepare a calibration dataset.** Assemble 20-50 representative input-output pairs with known correct answers. These should span the difficulty distribution of real inputs. If ground truth is unavailable, use a stronger model or human labels on a sample.

3. **Run two sequential rounds on calibration data.** For each calibration input, execute the LLM pipeline twice sequentially (round 1 produces an answer, round 2 refines or retries). Record both outputs and classify each as correct or wrong against ground truth.

4. **Estimate transition probabilities.** Count the four transition types across all calibration examples:
   - `n_cc`: correct in round 1 AND correct in round 2
   - `n_cw`: correct in round 1 AND wrong in round 2
   - `n_wc`: wrong in round 1 AND correct in round 2
   - `n_ww`: wrong in round 1 AND wrong in round 2

   Compute: `p_cw = n_cw / (n_cc + n_cw)` and `p_wc = n_wc / (n_wc + n_ww)`.

5. **Evaluate the scaling regime.** Check the critical condition:
   - If `p_wc > p_cw`: sequential scaling is beneficial. Proceed to compute optimal rounds.
   - If `p_wc ≈ p_cw`: sequential scaling is neutral. Consider parallel scaling (Best-of-N) instead.
   - If `p_wc < p_cw`: sequential scaling is harmful. Stop at round 1 or switch strategies.

6. **Compute the accuracy upper bound.** The stationary accuracy (theoretical maximum with unlimited rounds) is `a_max = p_wc / (p_wc + p_cw)`. If the initial accuracy `a_0` is already close to `a_max`, few additional rounds are needed.

7. **Determine the optimal number of rounds.** Using the convergence rate `r = 1 - p_wc - p_cw`, accuracy after `n` rounds is approximately `a_n = a_max - (a_max - a_0) * r^n`. Solve for the `n` where marginal gain `a_{n+1} - a_n` drops below the cost-per-round expressed in accuracy units. Implement this as the stopping criterion.

8. **Implement the MarkovScale controller.** Wrap the LLM call loop with a controller that:
   - Runs up to `n_optimal` rounds (from step 7)
   - Optionally implements early stopping if the answer stabilizes (same answer in consecutive rounds, suggesting the absorbing state)
   - Logs transition counts for ongoing recalibration

9. **Add runtime monitoring.** Track actual transition frequencies in production. If `p_wc` or `p_cw` drift significantly from calibration estimates (e.g., due to prompt changes or distribution shift), trigger recalibration. Use a sliding window of the last 100-200 queries.

10. **Benchmark against baselines.** Compare MarkovScale against fixed-retry (e.g., always 3 rounds), Best-of-N with majority voting, and unlimited sequential refinement. Measure both accuracy and total LLM calls to validate the cost-accuracy tradeoff.

## Concrete Examples

**Example 1: Optimizing a code generation retry loop**

User: "My pipeline retries code generation up to 5 times if tests fail. It's expensive. How do I know the right number of retries?"

Approach:
1. Collect 30 representative coding problems with test suites
2. Run the pipeline for 2 rounds on each, recording pass/fail per round
3. Suppose results: `n_cc=12, n_cw=3, n_wc=6, n_ww=9`
4. Compute: `p_cw = 3/15 = 0.20`, `p_wc = 6/15 = 0.40`
5. Since `p_wc > p_cw`, sequential retries help
6. Upper bound: `a_max = 0.40 / (0.40 + 0.20) = 0.667`
7. Convergence rate: `r = 1 - 0.40 - 0.20 = 0.40`
8. Starting from `a_0 = 0.50`, accuracy after n rounds: `a_n = 0.667 - 0.167 * 0.40^n`

Output:
```
Round 1: 50.0% accuracy (baseline)
Round 2: 60.0% accuracy (+10.0pp)
Round 3: 64.3% accuracy (+4.3pp)
Round 4: 66.0% accuracy (+1.7pp)
Round 5: 66.5% accuracy (+0.5pp)
Upper bound: 66.7%

Recommendation: Use 3 retries (captures 94% of maximum gain).
Savings: 40% fewer LLM calls vs. your current 5-retry loop.
```

**Example 2: Deciding between sequential and parallel scaling**

User: "Should I use Best-of-N or sequential refinement for my math problem solver?"

Approach:
1. Run calibration with 40 math problems, 2 sequential rounds each
2. Suppose results: `n_cc=8, n_cw=7, n_wc=5, n_ww=20`
3. Compute: `p_cw = 7/15 = 0.467`, `p_wc = 5/25 = 0.200`
4. Since `p_wc < p_cw`, sequential refinement DEGRADES accuracy
5. The model is more likely to corrupt correct answers than fix wrong ones

Output:
```
WARNING: Sequential scaling is counterproductive for this task.
- Corruption rate (p_cw): 0.467
- Repair rate (p_wc): 0.200
- Each additional round REDUCES expected accuracy.

Recommendation: Use parallel scaling (Best-of-N with majority voting)
instead. Generate N independent samples and select by consensus.
The sequential refinement prompt may be introducing errors -- consider
auditing the refinement instruction.
```

**Example 3: Implementing a MarkovScale controller in Python**

User: "Give me a MarkovScale wrapper for my LLM pipeline."

Output:
```python
class MarkovScaleController:
    def __init__(self, p_wc: float, p_cw: float, cost_threshold: float = 0.01):
        self.p_wc = p_wc
        self.p_cw = p_cw
        self.cost_threshold = cost_threshold
        self.a_max = p_wc / (p_wc + p_cw) if (p_wc + p_cw) > 0 else 0.5
        self.r = 1.0 - p_wc - p_cw

    @property
    def is_beneficial(self) -> bool:
        return self.p_wc > self.p_cw

    def optimal_rounds(self, a_0: float) -> int:
        if not self.is_beneficial or self.r <= 0:
            return 1
        n = 1
        a_prev = a_0
        while n < 20:  # safety cap
            a_n = self.a_max - (self.a_max - a_0) * (self.r ** n)
            marginal_gain = a_n - a_prev
            if marginal_gain < self.cost_threshold:
                break
            a_prev = a_n
            n += 1
        return n

    def run(self, llm_fn, input_data, a_0: float):
        max_rounds = self.optimal_rounds(a_0)
        result = None
        for round_num in range(max_rounds):
            result = llm_fn(input_data, previous_result=result)
            if round_num > 0 and result == prev_result:
                break  # answer stabilized, early stop
            prev_result = result
        return result

    @classmethod
    def from_calibration(cls, transitions: list[tuple[bool, bool]], **kwargs):
        n_cc = sum(1 for a, b in transitions if a and b)
        n_cw = sum(1 for a, b in transitions if a and not b)
        n_wc = sum(1 for a, b in transitions if not a and b)
        n_ww = sum(1 for a, b in transitions if not a and not b)
        p_cw = n_cw / max(n_cc + n_cw, 1)
        p_wc = n_wc / max(n_wc + n_ww, 1)
        return cls(p_wc=p_wc, p_cw=p_cw, **kwargs)
```

## Best Practices

- **Do:** Always run calibration before deploying. Even 20 examples give useful transition probability estimates. Without calibration, you're guessing.
- **Do:** Check `p_wc > p_cw` before enabling sequential scaling. If this condition fails, sequential retries actively hurt -- switch to parallel sampling.
- **Do:** Implement early stopping on answer stabilization. If two consecutive rounds produce identical outputs, further rounds almost certainly will too.
- **Do:** Recalibrate when changing prompts, models, or task distributions. Transition probabilities are specific to a given configuration.
- **Avoid:** Using a fixed retry count across all tasks. Different difficulty levels have different transition dynamics; hard problems may have lower `p_wc`.
- **Avoid:** Conflating sequential scaling with parallel scaling. Best-of-N generates independent samples; sequential refinement conditions each round on the previous output. They have fundamentally different Markov dynamics.

## Error Handling

- **Insufficient calibration data:** With fewer than 15 examples, transition probability estimates will have high variance. Warn the user and recommend collecting more data. As a fallback, use conservative defaults (assume `p_wc ≈ p_cw`, meaning neutral regime -- default to 1 round).
- **Zero-count transitions:** If no `cw` or `wc` transitions are observed in calibration, use Laplace smoothing: add 1 to each count before computing probabilities. This prevents division-by-zero and overly confident estimates.
- **Non-stationary dynamics:** If the LLM's behavior changes over a conversation (e.g., due to growing context), the Markov assumption weakens. Monitor prediction residuals and recalibrate if accuracy after `n` rounds deviates from the predicted `a_n` by more than 5 percentage points.
- **Cost accounting mismatch:** The `cost_threshold` parameter must be in the same units as accuracy gain. If using token cost instead of call count, normalize the marginal gain by cost-per-token to compare properly.

## Limitations

- The two-state model (Correct/Wrong) loses information when partial correctness matters (e.g., multi-part answers, code with some tests passing). For graded outputs, consider extending to a multi-state chain, though closed-form solutions become harder.
- The Markov assumption (next state depends only on current state, not history) may not hold if the model accumulates context across rounds. This is most accurate for stateless retry loops and weakest for long chain-of-thought refinement with full history.
- Calibration requires labeled data. For tasks without ground truth (open-ended generation, creative writing), the framework cannot be directly applied.
- The framework assumes homogeneous transition probabilities across inputs. In practice, easy and hard problems have very different `p_wc` values. Stratifying calibration by estimated difficulty improves predictions.
- MarkovScale optimizes expected accuracy across a distribution of inputs. For individual high-stakes queries, the variance around the expected value may matter more than the mean.

## Reference

**Paper:** [MarkovScale: Towards Optimal Sequential Scaling at Inference Time](https://arxiv.org/abs/2602.01120v1) (Wang et al., 2026). Focus on Section 3 (Markov formulation and closed-form bounds) and Section 4 (the MarkovScale algorithm and stopping criteria) for implementation details.