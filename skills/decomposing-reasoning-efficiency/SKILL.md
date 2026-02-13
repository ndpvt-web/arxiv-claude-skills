---
name: "decomposing-reasoning-efficiency"
description: >
  Analyze and optimize LLM reasoning token efficiency using a multiplicative decomposition framework.
  Breaks down reasoning performance into completion rate, conditional correctness, verbosity,
  verbalization overhead, and coupling coefficient. Identifies bottleneck profiles and suggests
  targeted interventions. Includes trace-quality diagnostics (grounding, repetition, prompt copying).
  Trigger phrases: "analyze reasoning efficiency", "decompose token usage", "why is this model
  so verbose", "token budget analysis", "reasoning trace quality", "optimize reasoning tokens"
---

# Decomposing Reasoning Efficiency

This skill enables Claude to analyze and optimize how reasoning tokens are spent (or wasted) in LLM outputs. Rather than treating accuracy as the sole metric, it applies the multiplicative decomposition framework from Kaiser et al. (2026) to separate token efficiency into interpretable factors: whether the model completes within budget, whether completed responses are correct, and how verbosely the model reasons. When task metadata or reasoning traces are available, it goes further -- measuring verbalization overhead per work unit, how overhead scales with difficulty, and whether verbose output reflects genuine reasoning or degenerate looping.

## When to Use

- When the user asks to evaluate or compare reasoning model outputs for efficiency, not just accuracy
- When analyzing why a chain-of-thought prompt produces excessively long reasoning traces
- When debugging token budget overruns in production LLM pipelines (e.g., "my o3 calls keep hitting the max_tokens limit")
- When the user wants to profile a set of model responses to identify whether the bottleneck is truncation, incorrect reasoning, or pure verbosity
- When building evaluation harnesses for reasoning models and wanting metrics beyond pass@1
- When optimizing prompt engineering to reduce token waste without sacrificing correctness
- When comparing multiple models or prompting strategies on the same task set and needing a structured efficiency breakdown

## Key Technique

Standard LLM evaluations report a single accuracy number, hiding critical information about *where* tokens go. A model scoring 80% accuracy might be truncating 15% of its responses (never getting a chance to answer) and getting 94% of completed responses correct -- or it might complete everything but only get 80% right. These two failure modes demand completely different interventions (raise token budget vs. improve reasoning quality), yet accuracy alone cannot distinguish them.

The decomposition framework factors token-level efficiency multiplicatively:

**Efficiency = Completion Rate x Conditional Correctness x Verbosity Score**

- **Completion Rate**: fraction of responses that finish within the token budget (not truncated). Low completion signals the model needs a higher budget or more concise reasoning.
- **Conditional Correctness**: accuracy computed only over completed responses. Low conditional correctness means the model reasons poorly even when given enough tokens.
- **Verbosity Score**: an inverse measure of token consumption normalized by task difficulty. High verbosity means the model uses far more tokens than necessary for the work done.

When per-instance workload metadata is available (e.g., number of digits in an arithmetic problem, rows in a table), verbosity further decomposes into **verbalization overhead** (baseline tokens per work unit) and a **coupling coefficient** (how overhead scales with workload). A model with high overhead but low coupling wastes tokens on boilerplate regardless of difficulty. A model with low overhead but high coupling becomes disproportionately verbose on hard problems.

When reasoning traces are available, three deterministic trace-quality measures distinguish productive verbosity from degenerate output: **grounding ratio** (does the trace reference problem elements?), **repetition ratio** (does it loop over the same content?), and **prompt-copying ratio** (does it parrot the input?). These require no human judges or LLM-as-judge calls -- they are computed directly from text overlap and n-gram statistics.

## Step-by-Step Workflow

1. **Collect structured response data.** For each model response, record: (a) the problem/prompt, (b) the full reasoning trace if available, (c) the final answer, (d) the token count, (e) the token budget/limit, and (f) the ground-truth answer. Store as a list of structured records (JSON or dataframe rows).

2. **Compute Completion Rate.** For each response, check whether the output was truncated (hit the token limit without producing a final answer). Completion Rate = (number of non-truncated responses) / (total responses). Flag truncated instances for separate analysis.

3. **Compute Conditional Correctness.** Among only the completed (non-truncated) responses, compute accuracy: Conditional Correctness = (correct completed responses) / (total completed responses). This isolates reasoning quality from budget constraints.

4. **Compute raw Verbosity.** For each correct, completed response, record the token count. Compute the mean and distribution. Verbosity Score = (reference token count) / (actual mean token count), where the reference can be the minimum observed, a theoretical lower bound, or a cross-model baseline.

5. **Decompose Verbosity (if workload metadata available).** If each problem has a workload proxy (e.g., number of reasoning steps, input size, digit count), fit a linear model: `tokens_i = alpha + beta * workload_i`. The intercept `alpha` is the verbalization overhead (fixed cost per response). The slope `beta` is the coupling coefficient (marginal token cost per work unit). Compare these across models or prompt strategies.

6. **Compute Trace-Quality Measures (if reasoning traces available).** For each trace:
   - **Grounding ratio**: Count unique problem-element references (entities, numbers, constraints from the prompt) appearing in the trace, divided by total problem elements. Higher = more engaged reasoning.
   - **Repetition ratio**: Compute the fraction of n-grams (use 4-grams) in the trace that appear more than once. Higher = more looping.
   - **Prompt-copying ratio**: Compute the fraction of the trace's n-grams that appear verbatim in the input prompt. Higher = more parroting.

7. **Classify the bottleneck profile.** Based on the decomposition, assign each model/prompt to a profile:
   - **Truncation-bound**: Low completion rate is the primary drag. Intervention: increase budget or add conciseness instructions.
   - **Correctness-bound**: Completion is fine but conditional correctness is low. Intervention: improve reasoning (better prompts, few-shot examples, stronger model).
   - **Verbosity-bound**: Completes and answers correctly but uses excessive tokens. Intervention: conciseness prompting, shorter chain-of-thought templates.
   - **Degenerate**: High repetition or prompt-copying ratios indicate the model is not reasoning productively. Intervention: restructure prompt, add explicit anti-repetition instructions, or switch models.

8. **Generate a structured efficiency report.** Present findings as a table with columns: Model/Prompt, Completion Rate, Conditional Correctness, Mean Tokens, Verbosity Score, Bottleneck Profile. If trace data is available, add Grounding, Repetition, and Prompt-Copying columns.

9. **Recommend targeted interventions.** Based on the bottleneck profile, provide specific, actionable recommendations (not generic "make it better"). Each recommendation should address the identified limiting factor.

10. **Iterate and re-measure.** After applying interventions, re-run the decomposition to verify the bottleneck has shifted or resolved. Efficiency improvements should show up as changes in the specific decomposed factor that was targeted.

## Concrete Examples

**Example 1: Comparing two prompting strategies on math problems**

User: "I'm running GPT-4o on 200 competition math problems with two prompts -- a standard CoT prompt and a concise CoT prompt. Both get ~72% accuracy. Which is better?"

Approach:
1. Collect token counts, truncation flags, and correctness for all 400 responses
2. Compute the decomposition for each strategy

Output:
```
| Strategy     | Completion | Cond. Correctness | Mean Tokens | Verbosity Score | Bottleneck   |
|--------------|------------|-------------------|-------------|-----------------|--------------|
| Standard CoT | 0.88       | 0.82              | 2,847       | 0.31            | Truncation   |
| Concise CoT  | 0.97       | 0.74              | 1,203       | 0.73            | Correctness  |

Analysis: Standard CoT and Concise CoT both achieve ~72% accuracy, but for
different reasons. Standard CoT truncates 12% of responses (losing easy points)
but gets 82% of completed answers right. Concise CoT rarely truncates but its
compressed reasoning hurts quality (74% conditional correctness).

Recommendation:
- Standard CoT: Increase token budget from 4096 to 6144. Predicted accuracy
  gain: ~4-6% from recovered truncated responses.
- Concise CoT: Add 2-3 worked examples to the prompt to improve reasoning
  quality. Token savings are real but correctness needs work.
```

**Example 2: Diagnosing a verbose reasoning model**

User: "Our fine-tuned model gets 91% on our internal QA benchmark but costs 3x more tokens than the base model at 89%. Is the extra reasoning worth it?"

Approach:
1. Decompose both models on the same benchmark
2. Fit verbosity against workload (question complexity score) for each model
3. Check trace quality on a sample of responses

Output:
```
| Model      | Completion | Cond. Corr. | Overhead (α) | Coupling (β) | Grounding | Repetition |
|------------|------------|-------------|--------------|---------------|-----------|------------|
| Base       | 0.99       | 0.90        | 180 tokens   | 12 tok/unit   | 0.72      | 0.04       |
| Fine-tuned | 0.98       | 0.93        | 620 tokens   | 28 tok/unit   | 0.68      | 0.19       |

Diagnosis: The fine-tuned model's 3x token cost comes from two sources:
1. Verbalization overhead (α) is 3.4x higher -- it produces ~440 extra tokens
   of boilerplate per response regardless of difficulty.
2. Coupling coefficient (β) is 2.3x higher -- it scales worse on hard questions.
3. Repetition ratio of 0.19 suggests ~19% of reasoning content is redundant
   looping, while grounding actually decreased vs. base.

The 2% accuracy gain (91% vs 89%) does not justify the 3x cost. The fine-tuning
introduced verbose habits without proportional reasoning improvement.

Recommendation: Apply repetition penalty during decoding (rep_penalty=1.15).
Add a system instruction: "State each reasoning step once. Do not re-derive
previous conclusions." Expected token reduction: 30-40% with minimal accuracy loss.
```

**Example 3: Building an efficiency evaluation harness**

User: "I'm building an eval pipeline for our reasoning models. How do I add efficiency metrics alongside accuracy?"

Approach:
1. Define the data schema for capturing decomposition inputs
2. Implement the metric computations
3. Generate the report format

Output:
```python
import numpy as np
from collections import Counter

def compute_efficiency_decomposition(results: list[dict]) -> dict:
    """
    Each result dict must have:
      - 'completed': bool (not truncated)
      - 'correct': bool (final answer matches ground truth)
      - 'token_count': int (total reasoning + answer tokens)
      - 'workload': float (optional, task difficulty proxy)
      - 'trace': str (optional, full reasoning text)
      - 'prompt': str (optional, original prompt text)
    """
    n = len(results)
    completed = [r for r in results if r['completed']]
    correct_completed = [r for r in completed if r['correct']]

    completion_rate = len(completed) / n
    cond_correctness = len(correct_completed) / len(completed) if completed else 0.0
    mean_tokens = np.mean([r['token_count'] for r in correct_completed]) if correct_completed else float('inf')

    report = {
        'completion_rate': round(completion_rate, 3),
        'conditional_correctness': round(cond_correctness, 3),
        'mean_tokens_correct': round(mean_tokens, 1),
        'n_total': n,
        'n_completed': len(completed),
        'n_correct_completed': len(correct_completed),
    }

    # Verbosity decomposition (if workload metadata present)
    if all('workload' in r for r in correct_completed) and len(correct_completed) > 5:
        workloads = np.array([r['workload'] for r in correct_completed])
        tokens = np.array([r['token_count'] for r in correct_completed])
        # Linear fit: tokens = alpha + beta * workload
        A = np.vstack([np.ones_like(workloads), workloads]).T
        alpha, beta = np.linalg.lstsq(A, tokens, rcond=None)[0]
        report['verbalization_overhead_alpha'] = round(float(alpha), 1)
        report['coupling_coefficient_beta'] = round(float(beta), 2)

    # Trace quality (if traces present)
    if all('trace' in r and 'prompt' in r for r in completed[:50]):
        sample = completed[:50]
        gnd, rep, cpy = [], [], []
        for r in sample:
            trace_4grams = _ngrams(r['trace'], 4)
            prompt_4grams = set(_ngrams(r['prompt'], 4))
            trace_set = set(trace_4grams)
            counts = Counter(trace_4grams)
            rep.append(sum(1 for c in counts.values() if c > 1) / max(len(trace_set), 1))
            cpy.append(len(trace_set & prompt_4grams) / max(len(trace_set), 1))
        report['repetition_ratio'] = round(float(np.mean(rep)), 3)
        report['prompt_copying_ratio'] = round(float(np.mean(cpy)), 3)

    # Bottleneck classification
    report['bottleneck'] = _classify_bottleneck(report)
    return report

def _ngrams(text: str, n: int) -> list[tuple]:
    tokens = text.split()
    return [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]

def _classify_bottleneck(report: dict) -> str:
    if report['completion_rate'] < 0.90:
        return 'truncation-bound'
    if report['conditional_correctness'] < 0.80:
        return 'correctness-bound'
    if report.get('repetition_ratio', 0) > 0.15:
        return 'degenerate-looping'
    if report.get('verbalization_overhead_alpha', 0) > 500:
        return 'verbosity-bound (high overhead)'
    if report.get('coupling_coefficient_beta', 0) > 25:
        return 'verbosity-bound (high coupling)'
    return 'balanced'
```

## Best Practices

- **Do** always compute completion rate first. A model truncating 20% of responses has a fundamentally different problem than one completing everything but answering wrong. This is the most commonly overlooked factor.
- **Do** use the multiplicative decomposition (completion x correctness x verbosity) rather than additive metrics, since efficiency factors compound -- a 10% loss in each factor yields a 27% overall loss, not 30%.
- **Do** measure trace quality with deterministic text-overlap metrics (n-gram repetition, prompt copying) rather than subjective assessments or expensive LLM-as-judge calls.
- **Do** fit verbosity against a workload proxy when available -- the slope and intercept reveal structurally different verbosity problems requiring different fixes.
- **Avoid** comparing models on accuracy alone when token budgets differ. A model that "scores lower" may simply be truncating more often.
- **Avoid** treating all verbosity as waste. Check grounding ratio first -- verbose responses with high grounding (referencing problem elements) indicate engaged reasoning, while verbose responses with high repetition indicate degenerate looping.
- **Avoid** applying uniform "be concise" instructions without diagnosing the bottleneck. Forcing conciseness on a correctness-bound model will make things worse.

## Error Handling

- **Missing truncation signals**: If you cannot detect whether a response was truncated (no `finish_reason` field), use a heuristic: responses ending mid-sentence or at exactly the token limit are likely truncated. Flag these as uncertain and report completion rate as a range.
- **No workload metadata**: Skip the verbosity decomposition into overhead and coupling. Report raw mean token counts and distributions instead. The completion/correctness decomposition still works.
- **No reasoning traces**: Skip trace-quality measures entirely. The framework is explicitly designed to be "trace-optional" -- the core decomposition (completion, conditional correctness, verbosity) requires only final answers and token counts.
- **Small sample sizes**: With fewer than 30 instances, confidence intervals on all metrics will be wide. Report bootstrap 95% CIs alongside point estimates. The linear fit for verbosity decomposition is unreliable below ~20 data points.
- **Uneven workload distribution**: If workload values are clustered (e.g., most problems are easy), the coupling coefficient estimate will be unstable. Check the R-squared of the linear fit; below 0.3, report the coupling coefficient as unreliable.

## Limitations

- The framework assumes a fixed token budget. In streaming or interactive settings where budget is dynamic, completion rate must be redefined.
- Verbosity decomposition requires a meaningful per-instance workload proxy. If the task has no natural difficulty axis (e.g., open-ended creative writing), verbosity can only be measured in aggregate.
- Trace-quality measures (grounding, repetition, prompt copying) are n-gram based and cannot assess *semantic* reasoning quality -- a model could produce novel, grounded, non-repetitive text that is still logically wrong.
- The decomposition identifies *what* is inefficient but not *why*. A low conditional correctness score tells you the model reasons poorly but not which reasoning step fails.
- Coupling coefficient assumes a linear relationship between workload and tokens. For tasks with phase transitions in difficulty, a piecewise or nonlinear model may be needed.

## Reference

Kaiser, D., Frigessi, A., Ramezani-Kebrya, A., & Ricaud, B. (2026). Decomposing Reasoning Efficiency in Large Language Models. arXiv:2602.09805v1. https://arxiv.org/abs/2602.09805v1

Key insight to look for: Table 2 and Figure 2 showing how accuracy rankings vs. efficiency rankings diverge (Spearman rho=0.63), and the bottleneck profile taxonomy in Section 5 that maps different models to different efficiency interventions.