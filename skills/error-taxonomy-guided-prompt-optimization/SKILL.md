---
name: "error-taxonomy-guided-prompt-optimization"
description: |
  Optimize LLM prompts by systematically collecting errors, building a taxonomy of failure modes, and augmenting prompts with targeted guidance for the most frequent error categories. Based on ETGPO (Singh et al., 2026).
  Trigger phrases: "optimize this prompt", "my prompt keeps failing", "improve prompt accuracy", "debug prompt errors", "fix LLM failures", "prompt isn't working well"
---

# Error Taxonomy-Guided Prompt Optimization (ETGPO)

This skill enables Claude to optimize LLM prompts using a top-down, error-taxonomy-driven approach. Instead of trial-and-error prompt tweaking, ETGPO collects failures from running a prompt against test cases, categorizes those failures into a structured taxonomy ranked by frequency, and generates targeted guidance text that addresses the most prevalent error patterns. The result is a prompt that systematically closes the most impactful failure modes in a single pass, using roughly one third the token budget of iterative methods.

## When to Use

- When a user has a prompt that produces incorrect outputs on a known set of test cases and wants to systematically improve it
- When the user says "my prompt gets about 70% accuracy, how do I push it higher"
- When debugging why an LLM pipeline fails on certain input categories (math errors, reasoning gaps, formatting mistakes)
- When the user wants to optimize a prompt for a benchmark or evaluation suite with ground-truth answers
- When building a prompt for a production pipeline and needing to harden it against common failure modes before deployment
- When the user has collected failing examples and wants a structured method to analyze and fix them

## Key Technique

**Top-down vs. bottom-up.** Most prompt optimization methods work bottom-up: they look at individual failures, tweak the prompt, re-evaluate, and repeat. This risks overfitting to specific examples and losing sight of the global error landscape. ETGPO inverts this. It first collects *all* failures across multiple stochastic runs, then builds a complete error taxonomy before making any prompt changes. This single-pass, top-down strategy produces prompts that generalize better while consuming far fewer tokens.

**The error taxonomy.** The taxonomy is a structured categorization of every failure the model makes on the validation set. Each category has: a name and summary, a detailed description of what goes wrong, self-contained examples of the error, an explanation of *why* it leads to wrong answers, and prevalence statistics (failure count and unique problem count). Categories with fewer than 2 unique problems are discarded to avoid overfitting. The remaining categories are ranked by failure count, and the top G (typically 10) are selected for guidance generation.

**Guidance augmentation.** For each selected error category, the optimizer LLM generates actionable guidance text containing: a description of the error pattern, concrete advice for avoiding it, a WRONG example demonstrating the error, and a CORRECT example showing proper reasoning. This guidance is appended to the original prompt with a preamble, producing the final optimized prompt in one shot.

## Step-by-Step Workflow

1. **Collect the validation set.** Gather 50-200 input-output pairs where the ground truth is known. These are the cases you will use to diagnose failures. Separate a held-out test set for final evaluation.

2. **Run K stochastic passes.** Execute the current prompt against the validation set K times (K=5 is a good default). Use temperature > 0 so different runs can surface different failure modes. Record every failed trace: the input, the expected output, the model's reasoning chain, and the model's incorrect answer.

3. **Batch failures for taxonomy creation.** Split the collected failed traces into batches of B (B=6 works well). For the first batch, ask the LLM to identify where reasoning first goes wrong in each trace, determine the nature of the error, and group traces into named categories with descriptions and examples. Output as structured JSON.

4. **Incrementally build the taxonomy.** For each subsequent batch, provide the current taxonomy (category names, descriptions, and prevalence counts) and instruct the LLM to assign new traces to existing categories when possible, creating new categories only when a failure is fundamentally different from all existing ones.

5. **Filter and rank categories.** Discard any category backed by fewer than 2 unique problems (these are likely problem-specific, not systematic). Rank the remaining categories by total failure count in descending order.

6. **Select top G categories.** Pick the top G error categories (G=10 is a reasonable default; use G=5 for smaller or simpler domains). These represent the highest-impact failure modes.

7. **Generate targeted guidance.** For each selected category, prompt the LLM to produce guidance text structured as: (a) description of the error pattern, (b) actionable advice to avoid it, (c) a WRONG example showing the error in action, (d) a CORRECT example showing proper handling. Also generate a preamble that introduces the guidance block.

8. **Assemble the optimized prompt.** Append the preamble and all guidance items to the original prompt. The guidance section should come after the task instructions but before any few-shot examples.

9. **Evaluate on the held-out test set.** Run the optimized prompt against the test set to measure improvement. Compare against the baseline prompt accuracy.

10. **Iterate only if needed.** If a specific error category still dominates failures on the test set, generate additional guidance for that category. ETGPO is designed to work in a single pass, so further iteration should be rare.

## Concrete Examples

**Example 1: Optimizing a math reasoning prompt**

User: "My prompt for solving competition math problems gets 60% on my test set. Help me improve it."

Approach:
1. Run the user's prompt against their validation problems 5 times, collecting all failures
2. Build taxonomy from failed traces. Example categories discovered:
   - "Algebraic Calculation Error" (23 failures, 12 problems) — model drops terms during expansion
   - "Incorrect Pattern Generalization" (18 failures, 9 problems) — model assumes patterns hold without proof
   - "Unit/Constraint Ignoring" (11 failures, 7 problems) — model forgets domain constraints
3. Select top categories, generate guidance for each
4. Append to original prompt

Output (guidance appended to prompt):
```
## Common Pitfalls to Avoid

The following guidance addresses systematic errors observed in mathematical reasoning.

### 1. Algebraic Calculation Errors
When expanding, factoring, or simplifying expressions, verify each step explicitly.
Do not skip intermediate steps in polynomial expansion or fraction simplification.

WRONG: "Expanding (x+2)^3 = x^3 + 6x + 8" (missing terms)
CORRECT: "Expanding (x+2)^3 = x^3 + 3(x^2)(2) + 3(x)(4) + 8 = x^3 + 6x^2 + 12x + 8"

### 2. Incorrect Generalization from Patterns
Do not assume a pattern observed in small cases holds generally. Always prove or
verify the pattern for the general case before applying it.

WRONG: "f(1)=2, f(2)=4, f(3)=8, so f(n)=2^n" (without proving the recurrence)
CORRECT: "f(1)=2, f(2)=4, f(3)=8. Hypothesis: f(n)=2^n. Verify: the recurrence
f(n)=2*f(n-1) with f(1)=2 gives f(n)=2^n by induction."

### 3. Ignoring Domain Constraints
Always re-read the problem constraints before finalizing an answer. Check that your
solution satisfies all stated bounds, integrality requirements, and domain restrictions.

WRONG: Solving for x and returning x=-3 when the problem states x > 0
CORRECT: Solving for x, finding x=-3 and x=5, then selecting x=5 since x > 0
```

**Example 2: Improving a multi-hop QA prompt**

User: "My RAG pipeline's answer-generation prompt mishandles questions requiring reasoning over multiple documents. Can you optimize it?"

Approach:
1. Collect 100 multi-hop QA failures across 5 runs
2. Build taxonomy. Categories discovered:
   - "Bridge Entity Missed" (31 failures) — model fails to connect entity mentioned in doc A to the same entity in doc B
   - "Partial Evidence Synthesis" (22 failures) — model answers from one source, ignoring contradicting/complementary info in another
   - "Temporal Confusion" (14 failures) — model conflates events from different time periods across documents
3. Generate guidance targeting these three categories
4. Append to the answer-generation prompt

Output (excerpt):
```
## Multi-Document Reasoning Guidelines

### Bridge Entity Resolution
When answering requires information from multiple sources, explicitly identify the
connecting entity between documents. State which entity links the sources before
reasoning about the answer.

WRONG: "Document 1 says the CEO founded the company in 2010. The answer is 2010."
(ignores Document 2 which clarifies the CEO role changed in 2015)
CORRECT: "Document 1 mentions John Smith as CEO. Document 2 clarifies Smith became
CEO in 2015, replacing Jane Doe who founded the company in 2010. The founder is Jane Doe."
```

**Example 3: Fixing a logical reasoning prompt**

User: "My prompt for evaluating logical arguments gets confused by negations and quantifiers."

Approach:
1. Collect failures from logical reasoning test cases
2. Taxonomy reveals:
   - "Negation Scope Error" (19 failures) — model misapplies negation to wrong clause
   - "Universal/Existential Confusion" (15 failures) — model treats "some" as "all" or vice versa
3. Generate guidance with WRONG/CORRECT examples for each pattern
4. Append guidance instructing the model to explicitly parse quantifier scope and negation placement before evaluating arguments

## Best Practices

- **Do:** Run multiple stochastic passes (K >= 5) to surface failure modes that appear intermittently. A single pass misses errors that occur on only some runs.
- **Do:** Use structured JSON output when building the taxonomy so categories, counts, and examples are machine-parseable and easy to rank.
- **Do:** Include self-contained WRONG/CORRECT example pairs in every guidance item. The ablation data shows detailed guidance with examples outperforms short guidance by over 2 percentage points.
- **Do:** Discard categories backed by only 1 unique problem. These represent problem-specific quirks, not systematic failure modes.
- **Avoid:** Iteratively tweaking the prompt after each individual failure. This bottom-up approach loses the global perspective and burns tokens on local fixes.
- **Avoid:** Selecting too many guidance categories (G > 15). Prompt length increases reduce the model's ability to attend to each piece of guidance. Start with G=10 and tune down for simpler tasks.

## Error Handling

- **Too few failures to build a taxonomy:** If the prompt already achieves >90% accuracy, there may not be enough failures to form meaningful categories. Lower the category-inclusion threshold from 2 problems to 1, or increase the validation set size.
- **Taxonomy categories are too vague:** If the LLM produces categories like "reasoning error" that are too broad, re-prompt with instructions to be more specific: "Each category should describe a single, concrete failure mechanism, not a general class."
- **Guidance makes accuracy worse:** If appending guidance decreases performance, check whether the guidance is too long (exceeding the model's effective context) or contradicts the original instructions. Reduce G or shorten guidance items.
- **Stochastic runs produce identical failures:** Increase temperature or top-p to introduce more variation. If failures are deterministic, K=1 is sufficient and you can increase the validation set instead.

## Limitations

- Requires a validation set with ground-truth answers. ETGPO cannot be applied when correct outputs are subjective or unavailable.
- The taxonomy quality depends on the optimizer LLM's ability to diagnose errors. If the optimizer model is weaker than the backbone model, taxonomy categories may be superficial.
- Works best when failures cluster into a moderate number of categories (5-20). If every failure is unique, taxonomy-based optimization provides little leverage.
- Single-pass design means ETGPO does not adapt to errors introduced by the guidance itself. In rare cases, guidance for one category can cause regressions in another.
- Primarily validated on tasks with clear right/wrong answers (math, QA, logic). Effectiveness on open-ended generation tasks (creative writing, summarization) is undemonstrated.

## Reference

**Paper:** Singh, Yadav, and Blanco. "Error Taxonomy-Guided Prompt Optimization." arXiv:2602.00997v1, February 2026.
**Key takeaway:** Section 3.2 (taxonomy creation via batched LLM analysis) and Section 3.4 (guidance generation with WRONG/CORRECT examples) contain the core algorithmic details. Table 2 shows ETGPO uses ~1/3 the tokens of competing methods at equal or better accuracy.