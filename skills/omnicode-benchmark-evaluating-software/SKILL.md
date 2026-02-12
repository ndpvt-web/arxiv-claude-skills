---
name: "omnicode-benchmark-evaluating-software"
description: >
  Evaluate and improve coding agent performance across four real-world software engineering task
  categories: bug fixing, test generation, code review response, and style fixing -- in Python,
  Java, and C++. Uses the OmniCode multi-dimensional evaluation framework to expose blind spots
  in agent capabilities beyond simple patch generation.
  Trigger phrases: "evaluate my coding agent", "benchmark software engineering tasks",
  "test generation quality", "code review response", "style fixing evaluation",
  "multi-language agent benchmark"
---

# OmniCode: Multi-Dimensional Software Engineering Agent Evaluation

This skill enables Claude to apply the OmniCode evaluation framework to assess and improve coding
agent performance across four distinct software engineering dimensions: bug fixing, test generation,
code review response, and style fixing. Unlike narrow benchmarks (HumanEval, SWE-Bench) that only
measure patch generation, OmniCode reveals that agents strong at bug fixing often fail catastrophically
at test generation (as low as 4.5% when vacuous tests are filtered) and struggle with non-Python
languages. This skill teaches Claude to systematically evaluate agent output using OmniCode's
methodology and to structure its own responses to match the rigor these tasks demand.

## When to Use

- When the user asks to evaluate a coding agent's capabilities beyond simple code generation
- When generating tests that must discriminate correct implementations from incorrect ones (not just pass on the correct code)
- When responding to code review feedback to produce an improved patch
- When fixing style/linter violations without breaking functionality
- When the user needs to assess agent performance across Python, Java, and C++ simultaneously
- When building a synthetic evaluation pipeline to generate diverse SE tasks from a small set of real-world instances
- When the user asks to identify blind spots in an LLM coding workflow (e.g., "why do my AI-generated tests miss bugs?")

## Key Technique

OmniCode's core insight is that software engineering competence is multi-dimensional. An agent that
excels at bug fixing (56.4% on Python) may perform poorly at test generation (18.7%) or style
fixing in C++ (18.8%). The benchmark operationalizes this through four task categories with
category-specific evaluation metrics. Bug fixing and code review response use test-suite pass
rates (fail-to-pass + regression). Test generation uses a discriminative metric: generated tests
must pass on the ground-truth patch AND fail on all known-bad patches, filtering out vacuous tests
that pass everything. Style fixing uses a net-resolution score: max((resolved - new_violations) /
original_violations, 0), which is zeroed if any functional test breaks.

The synthetic generation framework is equally important. Starting from only 494 real GitHub instances
across 28 repositories, OmniCode produces 1,794 tasks by: (1) collecting failed LLM attempts as
bad patches for test generation validation, (2) systematically perturbing correct patches to create
adversarial variants, (3) generating realistic code review feedback using LLMs given the problem,
a bad patch, and the correct solution, and (4) extracting style violations via language-specific
linters (pylint, clang-tidy, PMD/Checkstyle). This pipeline lets teams create rigorous evaluations
from limited real-world data without manual annotation at scale.

A critical finding: patch complexity -- measured as (delta_files + delta_hunks + delta_lines/10) --
strongly predicts agent success. Resolved instances cluster at low complexity while failures show
heavy-tailed distributions. Bug-fixing and code-review-response performance correlate strongly
(Pearson 0.921), but style-fixing is weakly correlated (0.512), meaning these are genuinely distinct
capabilities requiring separate evaluation.

## Step-by-Step Workflow

1. **Classify the task category.** Determine which of the four OmniCode categories the current
   request maps to: bug fixing (issue + failing tests), test generation (write discriminative tests),
   code review response (patch + reviewer feedback), or style fixing (linter violations to resolve).

2. **Identify the language and gather context.** Determine if the target is Python, Java, or C++.
   Read the repository structure, existing test framework (pytest, JUnit, Google Test), and any
   linter configurations (.pylintrc, .clang-tidy, pmd-ruleset.xml).

3. **For bug fixing:** Read the issue description and failing test cases. Identify the minimal
   patch location using the fail-to-pass test as a guide. Generate the smallest diff that makes
   failing tests pass while keeping all existing tests green. Verify the patch touches the fewest
   files and hunks possible (lower complexity = higher success probability).

4. **For test generation:** Write tests that are *discriminative*, not just correct-path. Each test
   must: (a) pass on the correct implementation, (b) fail on at least one plausible incorrect
   implementation. Generate candidate bad patches by introducing common mutation patterns (off-by-one,
   wrong operator, missing edge case, swapped arguments). Validate each test against these bad
   patches mentally or via execution.

5. **For code review response:** Parse the reviewer's feedback into a checklist of specific
   requested changes. Apply each fix to the existing patch incrementally. After each change, verify
   the full test suite still passes. Flag any reviewer suggestions that conflict with test behavior.

6. **For style fixing:** Run or simulate the appropriate linter for the language. Group violations
   by file and rule. Fix violations in order of confidence (formatting > naming > structural).
   After each batch of fixes, verify no functional tests break. Track the net score:
   (resolved - newly_introduced) / original_count.

7. **Apply the complexity heuristic.** Before submitting, compute the patch complexity:
   delta_files + delta_hunks + delta_lines/10. If this exceeds ~15, look for ways to simplify.
   Split large patches into smaller, independently testable changes when possible.

8. **Validate with the discriminative test criterion.** For any generated tests, mentally simulate
   or actually run them against 2-3 plausible incorrect implementations. If a test passes on all
   variants, it is vacuous and must be strengthened with tighter assertions.

9. **Cross-language verification.** If the task involves Java or C++, double-check build system
   integration (Maven/Gradle for Java, CMake/Make for C++), compilation, and test runner
   configuration. Agents fail on non-Python tasks primarily due to build/environment issues rather
   than logic errors.

10. **Report results using OmniCode metrics.** Present pass/fail using the category-appropriate
    metric: test pass rate for bug fixing and review response, discriminative pass rate for test
    generation, and net-resolution score for style fixing.

## Concrete Examples

**Example 1: Discriminative Test Generation**

User: "Write tests for this Python function that calculates shipping cost. The function has a bug
where orders over $100 should get free shipping but the threshold check uses `>` instead of `>=`."

Approach:
1. Identify the ground-truth fix: change `> 100` to `>= 100`
2. Write tests that pass on the correct implementation
3. Generate bad-patch variants: (a) original bug (`> 100`), (b) wrong threshold (`> 99`),
   (c) missing discount entirely
4. Ensure at least one test fails on each bad variant

Output:
```python
def test_free_shipping_at_exact_threshold():
    """Discriminative: fails on > 100 bug, passes on >= 100 fix."""
    assert calculate_shipping(order_total=100.00) == 0.00

def test_shipping_charged_below_threshold():
    """Passes on both correct and buggy -- not discriminative alone."""
    assert calculate_shipping(order_total=50.00) == 5.99

def test_free_shipping_above_threshold():
    """Passes on both correct and buggy -- not discriminative alone."""
    assert calculate_shipping(order_total=150.00) == 0.00

def test_shipping_just_below_threshold():
    """Discriminative against wrong-threshold variant (> 99)."""
    assert calculate_shipping(order_total=99.99) == 5.99
```

Key: `test_free_shipping_at_exact_threshold` is the discriminative test -- it fails on the buggy
`> 100` implementation but passes on the correct `>= 100`. Without it, a test suite could score
100% on the correct code yet miss the bug entirely.

---

**Example 2: Code Review Response**

User: "The reviewer said my patch for the database connection retry logic has three issues:
(1) the retry count should be configurable, not hardcoded to 3, (2) exponential backoff is
missing, and (3) the exception type is too broad."

Approach:
1. Parse feedback into three discrete changes
2. Apply each fix sequentially, verifying tests pass after each
3. Keep the diff minimal -- don't refactor unrelated code

Output:
```python
# Before (original patch):
def connect_with_retry(db_url):
    for attempt in range(3):
        try:
            return create_connection(db_url)
        except Exception:
            time.sleep(1)
    raise ConnectionError("Failed after 3 retries")

# After (review-response patch):
def connect_with_retry(db_url, max_retries=3):          # Fix 1: configurable
    for attempt in range(max_retries):
        try:
            return create_connection(db_url)
        except (ConnectionError, TimeoutError):          # Fix 3: specific exceptions
            time.sleep(2 ** attempt)                     # Fix 2: exponential backoff
    raise ConnectionError(f"Failed after {max_retries} retries")
```

---

**Example 3: Style Fixing with Net-Resolution Scoring**

User: "pylint reports 12 violations in my module. Fix them without breaking the tests."

Approach:
1. Read all 12 violations, group by rule (e.g., 4x line-too-long, 3x missing-docstring,
   2x unused-import, 2x invalid-name, 1x broad-except)
2. Fix highest-confidence first: unused-import (remove), line-too-long (wrap)
3. After each batch, verify functional tests pass
4. Track score: if 10 resolved and 1 new violation introduced, score = (10-1)/12 = 75%

Output:
```
Style Fix Report:
- Resolved: 11 of 12 violations
- New violations introduced: 0
- Net resolution score: 11/12 = 91.7%
- Remaining: 1 x C0116 (missing-function-docstring) -- skipped because adding a docstring
  to `_internal_helper` would be speculative without domain context
- All 47 functional tests: PASSING
```

## Best Practices

- **Do:** Write tests at boundary values where correct and incorrect implementations diverge.
  The exact threshold, the empty input, the off-by-one -- these are where discriminative power lives.
- **Do:** Measure patch complexity before submitting. Simpler patches (fewer files, fewer hunks)
  have dramatically higher success rates across all four task categories.
- **Do:** Treat each review comment as an independent, testable change. Apply and verify
  incrementally rather than making all changes at once.
- **Do:** Use language-specific linters as oracles for style fixing -- don't guess at style rules.
  Run pylint/clang-tidy/PMD to get the ground truth before and after.
- **Avoid:** Writing tests that only exercise the happy path. OmniCode shows that without bad-patch
  validation, apparent test generation success rates inflate by 5-18 percentage points.
- **Avoid:** Assuming Python-level success transfers to Java/C++. Agent performance drops 25-37
  percentage points on non-Python tasks, primarily due to build system and compilation issues.

## Error Handling

- **Test generation produces vacuous tests:** If all generated tests pass on both correct and
  mutated code, systematically strengthen assertions. Add exact value checks, boundary tests,
  and negative cases. If no discriminative test can be written, the specification may be ambiguous.
- **Style fixes break functional tests:** Roll back the most recent batch of style changes and
  reapply one at a time. The culprit is usually a renamed variable that was referenced elsewhere
  or a removed import that was actually used at runtime (not detected by static analysis alone).
- **Review response creates conflicts:** When reviewer feedback contradicts existing tests, flag
  the conflict explicitly rather than silently choosing one interpretation. Present both options.
- **Build failures on Java/C++:** Verify the build system configuration before attempting code
  changes. Check that Maven/Gradle dependencies resolve and that CMake targets compile cleanly.
  Environment setup failures account for a large share of non-Python task failures.

## Limitations

- OmniCode's synthetic generation relies on LLM-produced bad patches and review comments, which
  may not capture the full diversity of real-world errors and feedback styles.
- The benchmark covers 28 repositories. Agents tuned to common open-source project structures
  may still fail on proprietary or unconventional codebases.
- Style fixing evaluation depends on linter rule sets, which vary across organizations. A
  "fixed" violation under one ruleset may be irrelevant or incorrect under another.
- Test generation's discriminative criterion requires known-bad patches. In real workflows,
  you must generate plausible incorrect implementations yourself (via mutation), which adds effort.
- The benchmark does not cover tasks like documentation generation, dependency management, CI/CD
  configuration, or deployment -- aspects of SE that are equally important in practice.

## Reference

**Paper:** [OmniCode: A Benchmark for Evaluating Software Engineering Agents](https://arxiv.org/abs/2602.02262v2)
(Sonwane et al., 2026). Look for: Table 2 (pass rates by category and language), the synthetic
generation pipeline in Section 4, the discriminative test metric in Section 3.2, and the patch
complexity analysis in Section 5.3. Code: https://github.com/seal-research/OmniCode