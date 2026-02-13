---
name: "rubberduckbench-benchmark-ai-coding"
description: "Evaluate and improve AI coding assistant responses using RubberDuckBench's rubric-based methodology. Detects hallucinations, scores partial credit, and enforces truthful code reasoning. Use when: 'evaluate this code explanation', 'check my AI answer for hallucinations', 'score this coding response', 'audit code assistant accuracy', 'benchmark code Q&A quality', 'review this technical answer for correctness'."
---

# RubberDuckBench: Rubric-Based Evaluation of AI Code Answers

This skill enables Claude to evaluate answers to contextualized coding questions using the rubric-based methodology from RubberDuckBench. The core technique treats every code explanation as a claim that can be decomposed into verifiable sub-criteria, scored with negative deductions (starting from perfect), and audited for hallucinations versus omissions. This produces far more diagnostic feedback than binary correct/wrong judgments, and directly addresses the finding that even top models hallucinate in 58.3% of coding responses.

## When to Use

- When a user asks Claude to verify or critique a code explanation (e.g., "Is this answer about `std::map::at` correct?")
- When evaluating whether an AI-generated code answer contains fabricated claims about APIs, libraries, or project behavior
- When a user wants to build a quality rubric for assessing coding Q&A in their own project
- When reviewing pull request comments or code review answers for technical accuracy
- When a user asks Claude to answer a contextualized question about their codebase and wants a self-audited, hallucination-aware response
- When benchmarking or comparing multiple LLM responses to the same coding question

## Key Technique

RubberDuckBench's methodology starts from a critical observation: coding questions grounded in a specific project, file, and line number are fundamentally harder than generic language questions. The benchmark curates questions from real GitHub PR comments, rephrases them as standalone queries, and evaluates answers against hand-crafted rubrics. Each rubric uses **negative scoring** -- responses start at full marks and lose points for errors. This matters because it distinguishes two failure modes: **hallucinations** (fabricated or incorrect claims, penalized heavily) versus **omissions** (missing information, penalized lightly). The paper found that models lose far more points to hallucinations than omissions, meaning the primary failure mode is confidently stating falsehoods, not failing to mention relevant details.

Questions fall into four categories that demand different reasoning depths: **Project Behavior** (how does this specific code work?), **Library Behavior** (what does this API actually do in this context?), **Value** (what values flow through these variables?), and **Performance** (what are the efficiency implications?). Project Behavior questions are hardest (55% average score) because they require reasoning over project-specific code rather than general knowledge. The benchmark covers Java, Python, and C++ across 13 high-profile open-source repositories.

The evaluation protocol uses three independent trials at near-zero temperature (0.01) with chain-of-thought prompting. A response is only "fully correct" if it scores perfectly across all three trials. The best models achieved full correctness on at most 2 out of 15 questions, revealing that even strong models are inconsistent. This means a single correct-looking answer is insufficient evidence of reliability.

## Step-by-Step Workflow

1. **Identify the question type.** Classify the coding question into one of four categories: Project Behavior (code-specific logic), Library Behavior (API/library semantics), Value (variable propagation and state), or Performance (efficiency concerns). This determines what evidence is needed.

2. **Gather the full code context.** Read the specific file, function, and surrounding code referenced by the question. For project behavior questions, trace call chains and data flow. For library questions, identify the exact API version and documented behavior. Never answer from general knowledge alone.

3. **Construct a verification rubric.** Break the ideal answer into 3-7 independently verifiable sub-criteria. Assign point values proportional to importance. Example for "What does `map.at()` vs `map[]` do?":
   - Criterion 1 (2 pts): Correctly states the operational difference
   - Criterion 2 (3 pts): Correctly describes non-existent key behavior for each
   - Criterion 3 (2 pts): Correctly identifies const-correctness implications

4. **Weight hallucinations heavier than omissions.** For each criterion, define separate deduction amounts: full deduction for stating something false (hallucination), partial deduction for missing a relevant point (omission). A response claiming `map[]` throws on missing keys is worse than one that simply doesn't mention the difference.

5. **Generate or evaluate the answer against each criterion.** For each sub-criterion, check: (a) Is the claim present? (b) Is the claim factually correct against the actual code/documentation? (c) Is there any fabricated detail? Mark each criterion as: full credit, partial (omission), or deducted (hallucination).

6. **Flag all hallucinations explicitly.** List every fabricated or incorrect claim separately with a brief explanation of what is actually true. This is the most actionable output -- users need to know which specific statements to distrust.

7. **Compute the score and provide a breakdown.** Report the total score as a percentage, list each criterion with its status, and provide an overall assessment. Distinguish between "mostly correct with gaps" (omission-heavy) and "confidently wrong" (hallucination-heavy).

8. **State uncertainty honestly.** If the code context is insufficient to verify a criterion, say so explicitly rather than guessing. The benchmark found that models hallucinate most when they lack context but attempt to answer anyway.

9. **For self-auditing: re-examine your own answer.** After drafting a response to a coding question, apply the rubric to your own answer before presenting it. Remove or qualify any claim you cannot verify against the actual code.

## Concrete Examples

**Example 1: Evaluating an AI answer about API behavior**

User: "I asked an AI why `pandas.DataFrame.merge()` drops rows when I use `how='inner'` and it said: 'Inner merge keeps all rows from both DataFrames and fills missing values with NaN.' Is this correct?"

Approach:
1. Classify: Library Behavior question about pandas merge semantics
2. Build rubric:
   - C1 (3 pts): Correctly describes inner join semantics (keeps only matching rows)
   - C2 (2 pts): Correctly distinguishes from outer join (which fills NaN)
   - C3 (2 pts): Does not fabricate behavior
3. Evaluate the AI answer against rubric

Output:
```
## Rubric Evaluation: pandas inner merge explanation

| Criterion | Points | Status | Detail |
|-----------|--------|--------|--------|
| Inner join semantics | 3 | HALLUCINATION (-3) | Claims inner merge "keeps all rows" -- this is false. Inner merge keeps only rows with matching keys in both DataFrames. |
| Distinction from outer | 2 | HALLUCINATION (-2) | Attributes NaN-filling to inner join -- this is outer join behavior. |
| No fabricated claims | 2 | DEDUCTED (-2) | The entire explanation describes outer join behavior, not inner. |

Score: 0/7 (0%)
Verdict: Confidently incorrect. The response describes `how='outer'` behavior
while labeling it as `how='inner'`. The actual behavior of inner merge is to
DISCARD non-matching rows, not preserve them.

Correct answer: `how='inner'` retains only rows where the join key exists in
both DataFrames. Rows without a match in the other DataFrame are dropped
entirely -- no NaN filling occurs.
```

**Example 2: Self-audited response to a project-specific question**

User: "In the Flask codebase (commit abc123), why does `app.py:142` call `ensure_sync()` before invoking the view function?"

Approach:
1. Classify: Project Behavior -- requires reading the specific Flask code
2. Read `app.py` around line 142 and trace `ensure_sync` definition
3. Build rubric from what the code actually does
4. Draft answer, then self-audit against rubric

Output:
```
## Answer (self-audited)

`ensure_sync()` at app.py:142 wraps the view function to guarantee it runs
synchronously in the WSGI worker thread. Flask supports both sync and async
view functions, but the WSGI interface requires synchronous callables.
`ensure_sync` checks if the view is a coroutine function and, if so, wraps it
with `asyncio.run()` to execute it in a new event loop.

## Self-Audit Rubric

| Criterion | Points | Status |
|-----------|--------|--------|
| Purpose: WSGI sync compatibility | 3 | PASS |
| Mechanism: coroutine detection + wrapping | 2 | PASS |
| Scope: applies to view functions | 2 | PASS |
| No fabricated implementation details | 3 | UNCERTAIN -- I stated it uses asyncio.run(); verify against the actual ensure_sync implementation |

Score: 7/10 (pending verification of C4)
Note: Criterion 4 flagged for manual verification. The wrapping mechanism
should be confirmed against the actual `ensure_sync` source code.
```

**Example 3: Building a rubric for a code review question**

User: "I need a rubric to evaluate answers to: 'Why does this Python function use `__slots__`?'"

Approach:
1. Classify: Library Behavior (Python language feature)
2. Identify all verifiable claims about `__slots__`
3. Weight by importance and hallucination risk

Output:
```
## Evaluation Rubric: __slots__ usage (10 points, negative scoring)

C1 - Memory optimization (3 pts):
  - PASS: States __slots__ reduces per-instance memory by avoiding __dict__
  - HALLUCINATION (-3): Claims __slots__ speeds up method calls or CPU
  - OMISSION (-1): Doesn't mention memory benefit

C2 - Attribute restriction (3 pts):
  - PASS: States __slots__ prevents creation of arbitrary instance attributes
  - HALLUCINATION (-3): Claims __slots__ makes attributes immutable
  - OMISSION (-1): Doesn't mention attribute restriction

C3 - Inheritance interaction (2 pts):
  - PASS: Notes that __slots__ must be declared in each class in the hierarchy
  - HALLUCINATION (-2): Claims __slots__ automatically inherits to subclasses
  - OMISSION (-0.5): Doesn't mention inheritance

C4 - No fabricated claims (2 pts):
  - Deduct 2 points for any false statement about __slots__ behavior

Scoring guide:
  8-10: Strong answer, suitable for code review
  5-7:  Partial understanding, has gaps
  0-4:  Unreliable, contains fabrications
```

## Best Practices

- **Do:** Always read the actual code before answering project-specific questions. The benchmark showed Project Behavior questions (requiring codebase reading) have the lowest scores (55%) because models rely on general knowledge instead of specific code.
- **Do:** Explicitly separate hallucinations from omissions in your evaluation. A response that says nothing wrong but misses a point is fundamentally different from one that confidently fabricates API behavior.
- **Do:** Use negative scoring (start from perfect, deduct for errors) rather than positive scoring (start from zero, add for correct claims). Negative scoring naturally penalizes hallucinations more and better reflects how wrong answers damage trust.
- **Do:** Run the evaluation mentally at least twice if consistency matters. The benchmark found that models scored perfectly on at most 2 out of 15 questions across three trials, revealing high variance in correctness.
- **Avoid:** Treating "partially correct" as "good enough." The benchmark found most models survive on partial credit alone. An answer scoring 60% still contains substantial errors or gaps.
- **Avoid:** Assuming expensive models or large parameter counts produce better answers. The benchmark found zero correlation between API cost and accuracy. A $0.05 response can outperform a $0.60 one.

## Error Handling

- **Insufficient code context:** If the referenced file, commit, or line number is unavailable, state this clearly and refuse to speculate. Models hallucinate most when they lack context but answer anyway.
- **Ambiguous question scope:** If a question could be about general language semantics or project-specific behavior, ask the user to clarify. The benchmark distinguishes Library Behavior from Project Behavior because they require different evidence.
- **Unfamiliar library versions:** If the question references a library version whose behavior you cannot verify, flag this uncertainty. Do not extrapolate from a different version -- APIs change.
- **Rubric disagreement:** If two criteria seem to conflict (e.g., the code does something that contradicts documentation), note the discrepancy rather than forcing a judgment. Real codebases contain bugs and undocumented behavior.

## Limitations

- This methodology works best for questions with objectively verifiable answers. Subjective questions ("Is this code clean?") cannot be rubric-scored reliably.
- Building high-quality rubrics is labor-intensive -- the paper reports an average of 12 person-hours per rubric. For ad-hoc evaluation, use simplified 3-4 criterion rubrics.
- The benchmark covers Java, Python, and C++ only. The rubric approach generalizes to other languages, but hallucination patterns may differ.
- Negative scoring assumes you know what the complete correct answer looks like. For open-ended questions where the full answer space is unknown, positive scoring may be more appropriate.
- The 15-question benchmark is deliberately small and deep rather than broad. It cannot measure general coding ability -- it measures truthfulness and precision on specific contextualized questions.

## Reference

**RubberDuckBench: A Benchmark for AI Coding Assistants** (Mohammad et al., 2026) -- arXiv:2601.16456v1
Core insight: Rubric-based negative scoring that distinguishes hallucinations from omissions reveals that even top models fabricate claims in 58.3% of coding responses, and no model reliably answers project-specific questions correctly across repeated trials.