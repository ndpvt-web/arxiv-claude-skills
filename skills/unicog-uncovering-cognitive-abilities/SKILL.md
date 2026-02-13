---
name: "unicog-uncovering-cognitive-abilities"
description: "Analyze and diagnose LLM reasoning through latent cognitive ability decomposition inspired by the UniCog framework. Decomposes reasoning traces into sparse cognitive dimensions to identify failure modes, rank candidate solutions, and improve multi-step reasoning accuracy. Use when: 'diagnose why the model reasoning failed', 'analyze cognitive load of these solutions', 'rank candidate answers by reasoning quality', 'find the reasoning failure in this chain of thought', 'decompose this reasoning into cognitive abilities', 'prioritize solutions using activation analysis'."
---

# UniCog: Cognitive Ability Analysis for LLM Reasoning

This skill enables Claude to apply the UniCog framework's core insight -- that LLM reasoning decomposes into sparse, disentangled cognitive dimensions with a shared core and ability-specific signatures -- to diagnose reasoning failures, rank competing solutions, and improve multi-step problem solving. Rather than treating LLM outputs as opaque, this approach maps reasoning traces onto identifiable cognitive abilities (paraphrasing, analogical reasoning, numerical transformation, backward reasoning, etc.) and uses activation intensity patterns to detect errors before they propagate.

## When to Use

- When the user asks to **diagnose why a chain-of-thought reasoning trace went wrong** and identify which cognitive step failed
- When the user has **multiple candidate solutions** to a math or logic problem and needs to rank them by reasoning quality
- When the user wants to **decompose a multi-step reasoning task** into distinct cognitive abilities to find weak links
- When building a **candidate prioritization pipeline** that selects the best answer from multiple LLM generations
- When the user needs to **detect reasoning anomalies** such as unnecessary complexity, circular logic, or noise injection in LLM outputs
- When designing **evaluation rubrics** for LLM reasoning that go beyond surface-level correctness

## Key Technique

UniCog models LLM cognition as a latent variable problem. The central observation is that dense neural activations during reasoning can be projected into a sparse latent "mind space" where only a small subset of dimensions activate for any given cognitive task. Analysis across nine cognitive variants reveals a **Pareto principle**: 82-97% of activated dimensions are shared across all reasoning types (the "shared reasoning core"), while only 3-18% are ability-specific signatures. This means most reasoning failures stem from disruption of the shared core, not from missing specialized abilities.

The second key insight is the **activation amplification effect**: when reasoning goes wrong, specific latent dimensions show anomalously high activation (1.1x to 2.0x stronger than correct reasoning) despite activating less frequently. These act as "dominant but sporadic signals during failure modes." In practice, this means that overly intense or complex-looking reasoning steps are more likely to be erroneous than calm, measured ones -- a counterintuitive finding that inverts the naive assumption that "more thinking = better."

For practical application, UniCog introduces **latent-informed candidate prioritization**: given N candidate solutions, compute the average activation strength across the reasoning trace. Rank solutions in ascending order of activation magnitude (lower = more likely correct). Apply weighted voting with `w_i = 1 + 0.5 * (r_i - 1)` where `r_i` is the rank position. This simple heuristic achieves up to 7.5% accuracy gains over standard inference on benchmarks from GSM8K to AIME competition problems.

## Step-by-Step Workflow

1. **Identify the reasoning task type.** Classify the problem into cognitive categories: numerical transformation, analogical reasoning, backward reasoning, paraphrasing/reformulation, condition analysis (missing/redundant), or intermediate step questioning. This determines which ability-specific dimensions to monitor.

2. **Generate multiple candidate reasoning traces.** Produce 3-8 independent chain-of-thought solutions for the same problem. Use temperature > 0 or varied prompts to ensure diversity. Each trace becomes a data point for comparative analysis.

3. **Decompose each trace into cognitive steps.** Break every candidate into atomic reasoning operations: equation setup, variable substitution, logical inference, condition checking, numerical computation, result verification. Label each step with its cognitive ability type.

4. **Score each step for activation intensity.** Assign a complexity/intensity score (1-5) to each cognitive step based on: verbosity relative to the operation's inherent difficulty, number of intermediate sub-steps introduced, degree of hedging or self-correction, and whether the step introduces new assumptions not present in the problem.

5. **Compute the average activation strength per candidate.** Sum the intensity scores across all steps and divide by the number of steps: `A_i = (1/N_steps) * sum(intensity_scores)`. Higher values indicate more "cognitive load" -- and per UniCog's finding, higher error probability.

6. **Flag anomalous activation patterns.** Identify steps where intensity is 2x or more above the trace average. These are candidate failure points. Check whether they correspond to: circular reasoning, unnecessary case splits, incorrect intermediate results that get "patched" later, or hallucinated constraints.

7. **Rank candidates by ascending activation strength.** The candidate with the lowest average activation strength (calmest, most direct reasoning) ranks first. Apply weighted voting: `w_i = 1 + 0.5 * (rank_i - 1)` to aggregate across candidates if answers differ.

8. **Validate the top-ranked candidate.** Verify the lowest-activation candidate against the problem constraints. If it fails validation, examine whether the failure is in the shared reasoning core (basic arithmetic, logic) or an ability-specific component (e.g., backward reasoning step).

9. **Report the cognitive diagnosis.** Present findings as: (a) which candidate is recommended, (b) what cognitive abilities each candidate exercised, (c) where anomalous activation was detected, and (d) the specific failure mode if errors were found.

10. **Apply corrective re-generation if needed.** If all candidates show high activation and errors, re-prompt with explicit constraints targeting the identified failure mode -- e.g., "Solve this without introducing auxiliary variables" or "Work forward from the given conditions only."

## Concrete Examples

**Example 1: Ranking math problem solutions**

User: "I have three different solutions to this problem: 'A store sells apples at $2 each and oranges at $3 each. John buys 5 fruits for $12. How many of each did he buy?' Which solution is most likely correct?"

Approach:
1. Decompose each solution into cognitive steps
2. Score activation intensity per step
3. Rank by average activation strength

```
Solution A (3 steps, avg intensity 1.7):
  Step 1: Set up equations: a + o = 5, 2a + 3o = 12 [intensity: 2, standard setup]
  Step 2: Substitute: 2(5-o) + 3o = 12 → o = 2 [intensity: 1, direct algebra]
  Step 3: Therefore a = 3, o = 2. Check: 6 + 6 = 12 ✓ [intensity: 2, verification]

Solution B (6 steps, avg intensity 3.2):
  Step 1: Consider cases for oranges: 0,1,2,3,4,5 [intensity: 3, unnecessary enumeration]
  Step 2: If o=0: cost = 10, too low [intensity: 2]
  Step 3: If o=1: cost = 11, too low [intensity: 2]
  Step 4: If o=2: cost = 12 ✓ [intensity: 2]
  Step 5: But wait, let me verify with algebra... [intensity: 5, ANOMALOUS - redundant re-check]
  Step 6: Yes, a=3, o=2 [intensity: 2]

Solution C (5 steps, avg intensity 3.8):
  Step 1: Let me try o=3: 3×3 + 2×2 = 13, too high [intensity: 3]
  Step 2: Hmm, let me reconsider... maybe a=4, o=1: 8+3=11 [intensity: 4, backtracking]
  Step 3: Actually I need to be more systematic [intensity: 5, ANOMALOUS - meta-hedging]
  Step 4: 2a + 3(5-a) = 12 → a = 3 [intensity: 3]
  Step 5: So a=3, o=2 [intensity: 2]

Ranking (ascending activation): A (1.7) > B (3.2) > C (3.8)
Recommendation: Solution A — direct, minimal cognitive load, correct.
Note: Solutions B and C both arrive at the correct answer but exhibit
activation amplification (unnecessary enumeration, backtracking, meta-hedging)
that correlates with higher error risk on harder problems.
```

**Example 2: Diagnosing a reasoning failure**

User: "This chain-of-thought got the wrong answer for 'If 3x + 7 = 22, what is x?'. The model said x = 6. Where did it fail?"

Approach:
1. Decompose the trace into cognitive steps with ability labels
2. Score intensity and flag anomalies
3. Identify the failure point by cognitive ability type

```
Reasoning trace analysis:
  Step 1 [Equation Setup]: "We need to solve 3x + 7 = 22" [intensity: 1, normal]
  Step 2 [Numerical Transform]: "Subtract 7 from both sides: 3x = 15" [intensity: 1, normal]
  Step 3 [Numerical Transform]: "Divide by 3: x = 6" [intensity: 2, ANOMALOUS]
    └── Diagnosis: 15 ÷ 3 = 5, not 6. Pure arithmetic error.

Cognitive diagnosis:
  - Failure type: Shared reasoning core (basic numerical computation)
  - Failure location: Step 3
  - Cognitive ability: Numerical transformation
  - Pattern: The setup and algebraic manipulation (ability-specific) were correct.
    The error is in the shared arithmetic core, consistent with UniCog's finding
    that most failures originate in the shared reasoning substrate.
  - Fix: Re-prompt with "Carefully compute 15 divided by 3" or ask for verification step.
```

**Example 3: Building a candidate prioritization pipeline**

User: "I'm generating 5 answers per question for a math benchmark. How do I pick the best one using cognitive analysis?"

Approach:
1. Design the scoring pipeline
2. Implement weighted voting
3. Provide code skeleton

```python
def score_candidate(reasoning_trace: str) -> float:
    """Score a reasoning trace by cognitive activation intensity."""
    steps = decompose_into_steps(reasoning_trace)
    intensities = []
    for step in steps:
        intensity = 0
        # Verbosity relative to operation complexity
        intensity += len(step.text.split()) / expected_words(step.operation_type)
        # Backtracking signals (hedging, self-correction)
        intensity += count_hedging_phrases(step.text) * 1.5
        # Unnecessary sub-step expansion
        intensity += max(0, step.sub_step_count - expected_sub_steps(step.operation_type))
        intensities.append(intensity)
    return sum(intensities) / len(intensities)

def rank_and_vote(candidates: list[str]) -> str:
    """Rank candidates by ascending activation, return weighted-vote winner."""
    scored = [(score_candidate(c), c, extract_answer(c)) for c in candidates]
    scored.sort(key=lambda x: x[0])  # ascending = lower activation preferred

    votes = {}
    for rank_idx, (score, trace, answer) in enumerate(scored):
        weight = 1 + 0.5 * rank_idx  # lower activation gets higher weight
        votes[answer] = votes.get(answer, 0) + weight

    return max(votes, key=votes.get)
```

## Best Practices

- **Do:** Generate at least 3 diverse candidates before ranking. The activation-based ranking gains power with more comparison points. A single trace gives no baseline for "normal" activation levels.
- **Do:** Weight the shared reasoning core more heavily in diagnosis. UniCog shows 82-97% of reasoning uses shared dimensions -- arithmetic errors, logical missteps, and basic comprehension failures are far more common than ability-specific failures.
- **Do:** Treat high verbosity and self-correction as warning signals, not signs of thoroughness. The activation amplification effect means traces that "try harder" are statistically more likely to contain errors.
- **Do:** Use the technique iteratively -- after identifying a failure mode, re-generate with targeted constraints and re-score.
- **Avoid:** Assuming the longest or most detailed solution is best. UniCog's core finding directly contradicts this intuition.
- **Avoid:** Applying this to creative or open-ended tasks where there is no ground truth. The framework is validated on mathematical reasoning where correctness is verifiable.

## Error Handling

- **All candidates have similarly high activation:** The problem may be genuinely difficult. Lower the sparsity threshold (accept more "normal" activation) or re-generate with explicit step-by-step constraints to reduce noise.
- **The lowest-activation candidate is wrong:** This occurs roughly 5-10% of the time. Fall back to majority voting among the top-3 lowest-activation candidates rather than trusting only the top-1.
- **Traces are too short to decompose:** For single-step problems (e.g., simple factual recall), the cognitive decomposition adds no value. Use this technique only for multi-step reasoning with 3+ identifiable cognitive operations.
- **Cognitive ability classification is ambiguous:** When a step blends multiple abilities (e.g., analogical reasoning + numerical transformation), score it under the dominant ability but flag the blend. Mixed-ability steps tend to have naturally higher activation.

## Limitations

- **Mathematical reasoning focus:** UniCog was validated primarily on mathematical problem-solving (GSM8K, MATH-500, AIME). Applicability to code debugging, natural language inference, or commonsense reasoning is extrapolated, not proven.
- **Proxy-based scoring:** Without access to actual model internals, the intensity scoring here uses surface-level textual proxies (verbosity, hedging, backtracking) rather than true latent activations. This is an approximation of the paper's method.
- **No improvement on trivially easy problems:** The activation amplification effect is most pronounced on challenging tasks. For problems where all candidates get the right answer with low effort, the ranking provides no differentiation.
- **Model-specific calibration needed:** The 1.1x-2.0x amplification range was measured across specific models. Different LLMs may exhibit different intensity baselines, so absolute thresholds should be calibrated per model.

## Reference

- **Paper:** [UniCog: Uncovering Cognitive Abilities of LLMs through Latent Mind Space Analysis](https://arxiv.org/abs/2601.17897v1) (Liu et al., 2026)
- **Key insight to look for:** Section on the Pareto principle in LLM reasoning (82-97% shared core) and the activation amplification effect where erroneous reasoning shows anomalously intense latent activations -- the foundation for ranking candidates by "calmness" of reasoning.