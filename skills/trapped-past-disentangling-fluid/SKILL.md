---
name: "trapped-past-disentangling-fluid"
description: "Diagnose whether an LLM is memorizing or reasoning by constructing distributional proximity tests. Classifies task inputs as within-distribution (WD), near-distribution (ND), or out-of-distribution (OOD) and measures the generalization gap. Use when: 'audit LLM reasoning vs memorization', 'test if my model generalizes', 'measure crystallized vs fluid intelligence', 'evaluate model robustness on novel inputs', 'benchmark OOD performance degradation', 'diagnose training data leakage in evaluations'."
---

# Disentangling Fluid and Crystallized Intelligence in LLMs

This skill enables Claude to apply the WD/ND/OOD distributional proximity framework from Pleiss, Schiffer & von Weizsäcker (2026) to diagnose whether an LLM is relying on memorized patterns (crystallized intelligence) or genuine reasoning (fluid intelligence) on a given task. The core technique constructs a three-tier taxonomy of test inputs ordered by proximity to likely training data, then measures performance degradation across tiers to quantify the generalization gap. This is applicable far beyond chess -- any domain with a structured task space and a verifiable ground truth can use this framework.

## When to Use

- When the user wants to evaluate whether an LLM is actually reasoning about a problem or just pattern-matching from training data
- When designing a benchmark and needs to control for data contamination or memorization
- When the user asks to audit an LLM's robustness on novel, unseen input configurations
- When comparing model generations (e.g., GPT-3.5 vs GPT-4 vs GPT-5) on genuine reasoning ability
- When the user wants to measure diminishing returns of chain-of-thought or extended reasoning on hard problems
- When building an evaluation suite that separates recall-based performance from generalization
- When investigating whether scaling or reasoning tokens improve OOD capability or only amplify WD performance

## Key Technique

The paper's core insight is a **distributional proximity taxonomy** that partitions task inputs into three tiers based on how likely they appeared in the model's training corpus:

1. **Within-Distribution (WD)**: Inputs that almost certainly appear in training data (e.g., common chess openings seen 1,000+ times in master-level game databases). Performance here reflects **crystallized intelligence** -- sophisticated recall of memorized patterns.
2. **Near-Distribution (ND)**: Inputs structurally similar to training data but not directly present (e.g., positions reachable by a short random walk from known states). These require interpolation between memorized patterns.
3. **Out-of-Distribution (OOD)**: Inputs with minimal resemblance to training data (e.g., randomly assembled valid configurations). Performance here isolates **fluid intelligence** -- the ability to reason from first principles.

The **generalization gap** is the core diagnostic metric: `Delta_gen = E[Loss | OOD] - E[Loss | WD]`. A large gap means the model is heavily reliant on memorization. The paper found ACPL (average centipawn loss) increased 7.79x from WD to OOD, and illegal move rates increased 4.72x. Critically, reasoning tokens (chain-of-thought) showed 88.56% reduced marginal benefit per token on OOD vs WD tasks -- meaning extended reasoning mostly amplifies recall rather than enabling novel problem-solving. Progress across model generations also decelerates on OOD tasks: GPT-3.5 to GPT-4 yielded 9.08% OOD improvement, but GPT-4 to GPT-5 only 5.73%.

To apply this beyond chess, identify a **verifiable ground truth** (chess engine, SQL results, math proofs, compiler output), construct your three tiers by controlling proximity to plausible training data, and measure the degradation gradient.

## Step-by-Step Workflow

1. **Define the task domain and ground truth oracle.** Choose a domain where outputs can be objectively verified: chess (engine evaluation), code (test suites), math (symbolic solvers), SQL (execution results), logic puzzles (formal verification). The oracle provides the loss function.

2. **Identify a proxy for training data distribution.** Find a large public corpus that approximates what the LLM likely trained on. For chess: Lichess Masters database. For code: GitHub public repos. For math: common textbook problems. For SQL: public benchmark datasets (Spider, WikiSQL). Frequency in this corpus approximates training probability.

3. **Construct the Within-Distribution (WD) tier.** Select inputs with high frequency in the proxy corpus (threshold: appears 1,000+ times or is a canonical/textbook instance). These should be solvable by pure memorization. Generate 50-200 WD test cases.

4. **Construct the Near-Distribution (ND) tier.** Generate inputs structurally similar to WD but absent from the proxy corpus. Techniques: apply small random perturbations to WD instances (swap variable names, change constants, reorder clauses, add one unusual step). Verify zero exact matches in the proxy corpus. Generate 50-200 ND test cases.

5. **Construct the Out-of-Distribution (OOD) tier.** Generate valid inputs with minimal resemblance to known patterns. Techniques: random valid configurations, adversarial construction, novel combinations of primitives not seen together in training. Verify structural dissimilarity to the proxy corpus. Generate 50-200 OOD test cases.

6. **Define the loss metric.** Choose a continuous metric aligned with your oracle: centipawn loss (chess), test failure rate (code), symbolic distance from correct answer (math), execution accuracy (SQL). Also track binary validity (illegal moves / syntax errors / runtime crashes) as a secondary metric.

7. **Evaluate the target LLM(s) across all three tiers.** Run each model on WD, ND, and OOD inputs under identical conditions. Record both the primary loss metric and the validity rate.

8. **Compute the generalization gap and degradation gradient.** Calculate: `Delta_gen = mean(Loss_OOD) - mean(Loss_WD)`. Also compute the normalized ratio: `Loss_OOD / Loss_WD`. Report the gradient as a table: WD -> ND -> OOD with loss and validity at each tier.

9. **Measure reasoning efficiency (optional).** If comparing reasoning intensities (e.g., zero-shot vs chain-of-thought vs extended thinking), compute marginal improvement per token at each tier: `efficiency = (Loss_reduction) / (reasoning_tokens_used)`. A collapse in efficiency from WD to OOD indicates reasoning amplifies recall rather than enabling generalization.

10. **Report findings with the Progress Acceleration Ratio.** When comparing model generations, compute `R = improvement_rate(gen_n+1) / improvement_rate(gen_n)` for each tier. R < 1 on OOD indicates decelerating progress on genuine reasoning despite overall model improvements.

## Concrete Examples

**Example 1: Evaluating Code Generation Generalization**

```
User: "I want to test whether GPT-4o is actually reasoning about code
or just memorizing LeetCode solutions."

Approach:
1. Oracle: Python test suites (pytest) for each problem.
2. Proxy corpus: LeetCode public problems + solutions on GitHub.
3. WD tier: 100 classic LeetCode problems (Two Sum, Valid Parentheses,
   Merge Sorted Lists) -- appear in thousands of GitHub repos.
4. ND tier: 100 modified LeetCode problems with changed constraints,
   renamed variables, and combined subproblems not seen as a unit
   (e.g., "Two Sum but on a circular buffer with wraparound indexing").
5. OOD tier: 100 novel algorithmic problems from recent programming
   contests (post-training cutoff) or synthetically generated problems
   with unusual constraint combinations.
6. Metric: test pass rate (primary), syntax error rate (secondary).

Output:
| Tier | Pass Rate | Syntax Errors | Ratio to WD |
|------|-----------|---------------|-------------|
| WD   | 92%       | 1%            | 1.00x       |
| ND   | 61%       | 4%            | 0.66x       |
| OOD  | 23%       | 12%           | 0.25x       |

Generalization gap: 69 percentage points.
Interpretation: Strong memorization signal. The 4x degradation from
WD to OOD suggests most "coding ability" is pattern recall.
```

**Example 2: Auditing SQL Generation for Training Leakage**

```
User: "Our SQL benchmark shows 95% accuracy but I suspect data
contamination. How do I check?"

Approach:
1. Oracle: Execute queries against the actual database, compare results.
2. Proxy corpus: Spider, WikiSQL, public SQL tutorials.
3. WD tier: 50 queries that appear verbatim or near-verbatim in Spider
   (e.g., "SELECT name FROM students WHERE age > 20").
4. ND tier: 50 queries on the same schemas but with unusual joins,
   subqueries, or window functions not present in Spider.
5. OOD tier: 50 queries on entirely novel schemas (invented domains
   like "interstellar shipping manifests") with complex nested logic.
6. Metric: execution accuracy (primary), SQL syntax validity (secondary).

Output:
| Tier | Exec Accuracy | Syntax Valid | Reasoning Tokens (avg) |
|------|---------------|--------------|------------------------|
| WD   | 94%           | 99%          | 120                    |
| ND   | 68%           | 91%          | 340                    |
| OOD  | 31%           | 74%          | 510                    |

Reasoning efficiency: WD = 0.78%/token, OOD = 0.06%/token (92% drop).
Interpretation: Confirmed contamination concern. The model spends 4x
more reasoning tokens on OOD but gets 13x less benefit per token.
The 95% headline number is inflated by memorization.
```

**Example 3: Comparing Model Generations on Math Reasoning**

```
User: "Is GPT-5 actually better at math reasoning than GPT-4,
or just trained on more math?"

Approach:
1. Oracle: Symbolic math solver (SymPy / Wolfram Alpha).
2. WD: 100 textbook calculus problems from common sources.
3. ND: 100 problems with same techniques but novel formulations.
4. OOD: 100 problems requiring novel combinations of techniques
   or from obscure competition math not in standard corpora.
5. Metric: exact match accuracy after symbolic simplification.

Output:
| Tier   | GPT-4 | GPT-5 | Improvement | Acceleration Ratio |
|--------|-------|-------|-------------|--------------------|
| WD     | 82%   | 94%   | +14.6%      | --                 |
| ND     | 54%   | 67%   | +24.1%      | --                 |
| OOD    | 19%   | 24%   | +26.3%      | R = 0.61 (decel)   |

Interpretation: GPT-5 improves across all tiers, but OOD accuracy
remains low (24%). The acceleration ratio R=0.61 < 1 on OOD means
progress on genuine reasoning is decelerating. At this rate,
reaching 50% OOD accuracy would require several more generations.
```

## Best Practices

- **Do:** Use frequency in a public proxy corpus (not the actual training data, which is unavailable) as your distributional proximity measure. The Lichess Masters database / GitHub / public benchmarks are good proxies.
- **Do:** Always include a random baseline (lower bound) and an oracle/expert baseline (upper bound) when reporting results. Raw numbers are meaningless without context.
- **Do:** Track both continuous loss metrics AND binary validity metrics (illegal moves / syntax errors). Validity collapse on OOD is often the clearest memorization signal.
- **Do:** Normalize metrics by task difficulty before comparing across tiers. OOD tasks may be inherently harder -- divide by random-baseline performance to control for this.
- **Avoid:** Using only WD benchmarks to evaluate models. This is the paper's central warning: headline benchmark numbers conflate memorization with reasoning.
- **Avoid:** Assuming more reasoning tokens will fix OOD performance. The paper shows marginal benefit per token drops ~89% from WD to OOD. Scale alone does not solve the generalization gap.
- **Avoid:** Treating ND as equivalent to OOD. The three-tier distinction matters: ND performance reflects interpolation ability, while OOD isolates extrapolation.

## Error Handling

- **Proxy corpus is incomplete or biased.** If your proxy corpus doesn't well-approximate the LLM's training data, tier assignments will be noisy. Mitigate by using multiple proxy sources and checking for consistency across them.
- **WD tier is contaminated with ND inputs.** If "common" inputs are actually rare in training data, you'll underestimate crystallized intelligence. Validate by checking that WD performance is high (>80%) -- if not, your WD tier may not be truly within-distribution.
- **OOD tier contains invalid inputs.** Randomly generated inputs may violate domain constraints (illegal chess positions, syntactically invalid code). Always validate OOD inputs against domain rules before testing.
- **Small sample sizes produce noisy estimates.** Use at least 50 instances per tier. Report confidence intervals. A generalization gap that isn't statistically significant (p > 0.05) is not diagnostic.
- **Model refuses or abstains on OOD inputs.** Some models may decline to answer when uncertain. Track refusal rate separately -- high refusal on OOD vs WD is itself a memorization signal (the model recognizes it hasn't "seen this before").

## Limitations

- The framework requires a **verifiable ground truth oracle**. It cannot be applied to open-ended tasks (creative writing, summarization) where there is no objectively correct answer.
- **Proxy corpus quality is critical.** Without access to actual training data, the three-tier assignment is always an approximation. Results are only as good as the proxy.
- The method diagnoses the *presence* of a generalization gap but does not prescribe how to *fix* it. The paper explicitly notes that "mechanisms beyond scale" are needed but does not specify what those mechanisms are.
- Performance on OOD tasks may be confounded by task difficulty, not just distributional novelty. The normalized ACPL metric helps but doesn't fully eliminate this confound.
- The chess domain has unusually clean structure (deterministic rules, perfect information, engine oracles). Applying the framework to messier domains (NLP, vision) requires more careful tier construction.

## Reference

Pleiss, L. S., Schiffer, M., & von Weizsäcker, R. K. (2026). *Trapped in the past? Disentangling fluid and crystallized intelligence of large language models using chess.* arXiv:2601.16823v1. [https://arxiv.org/abs/2601.16823v1](https://arxiv.org/abs/2601.16823v1)

Key takeaway: Look for Section 3 (taxonomy construction), Definition 3.1 (formal crystallized/fluid distinction), and the marginal reasoning efficiency analysis showing 88.56% drop from WD to OOD.