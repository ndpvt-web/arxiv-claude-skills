---
name: "cvedrl-code-verifier-difficulty-aware"
description: "Generate difficulty-aware unit tests that verify LLM-generated code using branch coverage analysis, complexity-weighted rewards, and majority voting selection. Use when asked to 'verify generated code', 'write tests for hard branches', 'difficulty-aware testing', 'rank code solutions with tests', 'generate verification tests', or 'test code with branch coverage'."
---

# CVeDRL: Difficulty-Aware Code Verification via Unit Test Generation

This skill enables Claude to verify LLM-generated code solutions by generating high-quality unit tests that are weighted toward difficult branches and complex samples. Based on CVeDRL (Code Verifier via Difficulty-aware Reinforcement Learning), the technique prioritizes testing hard-to-reach branches using exponential reward shaping and scores code complexity via Halstead difficulty and Maintainability Index before test generation. The result is a verification pipeline where multiple candidate solutions are ranked by how many independently-generated test suites they pass (majority voting), producing reliable code selection even for hard problems.

## When to Use

- When the user has multiple candidate code solutions and needs to pick the most correct one
- When generating unit tests for code with complex branching logic (nested conditionals, error paths, edge cases)
- When the user asks to verify whether generated code is correct without labeled test cases
- When writing tests that specifically target untested or hard-to-reach branches
- When the user wants to rank or score candidate implementations by test-passing behavior
- When building a post-verification pipeline for any code generation system
- When the user asks for high branch coverage tests prioritized by difficulty

## Key Technique

**Difficulty-aware reward decomposition.** CVeDRL decomposes test quality into four jointly-optimized signals: (1) syntactic correctness -- does the test parse and compile, (2) functional correctness -- does it execute without errors and fail on wrong code, (3) branch coverage -- how much of the target code's control flow does it exercise, and (4) sample difficulty -- how inherently complex is the code under test. The insight is that naive test generation wastes effort on trivial paths while ignoring the hard branches that actually distinguish correct from incorrect solutions.

**Exponential reward shaping for branch coverage.** Rather than treating 50% branch coverage as half as good as 100%, CVeDRL applies an exponential transformation: `r_cov = (e^alpha - 1)^(-1) * [exp(alpha * cov) - 1]`. With a tunable `alpha`, this stays nearly flat at low coverage but rises sharply as coverage approaches 100%, strongly incentivizing tests that reach the last uncovered branches. This is the key mechanism: it forces test generation to focus on the marginal hard branches rather than accumulating easy ones.

**Complexity-weighted feedback.** Before generating tests, CVeDRL computes two static metrics on the target code: Halstead Difficulty `D_H = (eta_1/2) * (N_2/eta_2)` (measuring operator/operand complexity) and inverted Maintainability Index `D_M = max(0, 1 - MI/100)`. These combine as `D = sqrt(D_H * D_M)` to produce a difficulty score in [0,1]. For hard code (high D), passing tests get amplified rewards `r_cov * (1 + D)` while failures get softened penalties `-1.0 - (1 - D)`, preventing the model from giving up on hard problems.

## Step-by-Step Workflow

1. **Analyze the target code's branch structure.** Identify all conditional branches (`if/elif/else`, `try/except`, `match/case`, loop guards, early returns). List each branch and classify it as easy (reachable with typical inputs) or hard (requires specific edge cases, error conditions, or boundary values).

2. **Compute code complexity metrics.** Calculate Halstead Difficulty from the code's distinct operators (`eta_1`), distinct operands (`eta_2`), and total operand occurrences (`N_2`). Estimate Maintainability Index from volume, cyclomatic complexity, and lines of code. Combine into a difficulty score `D = sqrt(D_H * D_M)` normalized to [0,1]. Use this to calibrate how aggressively to target edge cases.

3. **Generate a base test suite targeting easy branches first.** Write standard unit tests covering the happy path and obvious input categories. Use `unittest.TestCase` or `pytest` style. Each test should have explicit assertions checking return values, not just "does it run."

4. **Generate difficulty-targeted tests for hard branches.** For each hard branch identified in step 1, craft specific inputs designed to exercise that path. This includes: boundary values (0, -1, empty list, None), type edge cases, large inputs triggering overflow or timeout paths, and inputs hitting specific `except` clauses. Prioritize branches with the lowest current coverage.

5. **Apply exponential coverage prioritization.** When reviewing your test suite, weight uncovered branches exponentially: a test that covers the last 10% of branches is far more valuable than one covering already-tested paths. Iteratively add tests targeting the lowest-coverage regions until branch coverage plateaus.

6. **Execute tests against all candidate solutions.** Run the full test suite against each candidate code solution. Classify each execution as: pass (all assertions succeed), failure (assertions fail but test runs), or error (test crashes or times out).

7. **Score with majority voting.** For each candidate solution, count how many independently-generated test suites it passes completely. Select the solution with the highest pass count: `s_opt = argmax_i sum_j(p_i,j)` where `p_i,j = 1` if solution `i` passes all cases in test suite `j`.

8. **Validate selection confidence.** If the top candidate passes significantly more test suites than the runner-up (e.g., 8/10 vs 3/10), confidence is high. If scores are close (e.g., 6/10 vs 5/10), flag the result as uncertain and recommend manual review or additional targeted tests.

9. **Report branch coverage gaps.** List any branches that remain uncovered across all test suites. These represent potential hidden bugs that no test could reach -- flag them explicitly for the user.

## Concrete Examples

**Example 1: Verifying a merge-sort implementation**

User: "I have 3 candidate merge sort implementations. Write tests to figure out which one is correct."

Approach:
1. Analyze branch structure: recursive base case (len <= 1), split logic, merge comparison (left < right, left >= right), and leftover-element handling.
2. Compute complexity: moderate D (~0.4) due to recursion and multiple branches.
3. Generate base tests:
```python
import unittest

class TestMergeSort(unittest.TestCase):
    def test_empty_list(self):
        self.assertEqual(merge_sort([]), [])

    def test_single_element(self):
        self.assertEqual(merge_sort([1]), [1])

    def test_sorted_input(self):
        self.assertEqual(merge_sort([1, 2, 3]), [1, 2, 3])

    def test_reverse_sorted(self):
        self.assertEqual(merge_sort([3, 2, 1]), [1, 2, 3])

    def test_duplicates(self):
        self.assertEqual(merge_sort([3, 1, 3, 1]), [1, 1, 3, 3])
```
4. Generate hard-branch tests (targeting merge edge cases):
```python
    def test_all_equal(self):
        self.assertEqual(merge_sort([5, 5, 5, 5]), [5, 5, 5, 5])

    def test_odd_length(self):
        self.assertEqual(merge_sort([4, 2, 7, 1, 3]), [1, 2, 3, 4, 7])

    def test_negative_numbers(self):
        self.assertEqual(merge_sort([-3, 0, -1, 2]), [-3, -1, 0, 2])

    def test_large_input(self):
        import random
        data = random.sample(range(10000), 1000)
        self.assertEqual(merge_sort(data), sorted(data))

    def test_two_elements_swapped(self):
        # Targets the exact merge comparison branch
        self.assertEqual(merge_sort([2, 1]), [1, 2])
```
5. Run all tests against candidates. Candidate A passes 10/10, Candidate B fails `test_odd_length`, Candidate C errors on `test_large_input`. Select Candidate A.

**Example 2: Testing a calculator with error handling**

User: "Verify this `evaluate(expr: str) -> float` function that parses and evaluates math expressions."

Approach:
1. Branch analysis: valid expression path, division-by-zero guard, parenthesis mismatch, unknown operator, empty input, overflow. Six distinct branches.
2. Complexity: high D (~0.7) due to parsing logic and multiple exception paths.
3. Generate tests with difficulty weighting (hard branches get more tests):
```python
class TestEvaluate(unittest.TestCase):
    # Easy branches
    def test_simple_addition(self):
        self.assertAlmostEqual(evaluate("2 + 3"), 5.0)

    def test_operator_precedence(self):
        self.assertAlmostEqual(evaluate("2 + 3 * 4"), 14.0)

    # Hard branches -- weighted higher
    def test_division_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            evaluate("5 / 0")

    def test_nested_parentheses(self):
        self.assertAlmostEqual(evaluate("((2 + 3) * (4 - 1))"), 15.0)

    def test_mismatched_parens(self):
        with self.assertRaises(ValueError):
            evaluate("(2 + 3")

    def test_empty_input(self):
        with self.assertRaises(ValueError):
            evaluate("")

    def test_unknown_operator(self):
        with self.assertRaises(ValueError):
            evaluate("2 @ 3")

    def test_large_exponent(self):
        # Targets potential overflow path
        with self.assertRaises((OverflowError, ValueError)):
            evaluate("10 ** 10000")
```
4. Report: 6/6 branches covered. The division-by-zero and mismatched-parens branches (hardest) each have dedicated tests.

**Example 3: Ranking solutions for a LeetCode-style problem**

User: "I generated 5 solutions for 'longest substring without repeating characters'. Which is best?"

Approach:
1. Generate 3 independent test suites (each 8-10 tests) covering: empty string, single char, all unique, all same, unicode, very long strings, and the sliding-window boundary condition.
2. Run each suite against all 5 solutions.
3. Majority vote results:

```
Solution  | Suite 1 | Suite 2 | Suite 3 | Total Passes
----------|---------|---------|---------|-------------
Sol A     |  PASS   |  PASS   |  PASS   |     3
Sol B     |  PASS   |  FAIL   |  PASS   |     2
Sol C     |  PASS   |  PASS   |  PASS   |     3
Sol D     |  FAIL   |  FAIL   |  FAIL   |     0
Sol E     |  PASS   |  PASS   |  FAIL   |     2
```

4. Solutions A and C tie. Generate a 4th targeted suite focusing on the hardest branch (overlapping character at window boundary). Sol A passes, Sol C fails. Select Sol A.

## Best Practices

- **Do:** Always generate multiple independent test suites rather than one large suite. Independence is what makes majority voting reliable -- correlated tests provide no additional signal.
- **Do:** Compute branch structure before writing tests. Listing branches first prevents the common trap of writing 20 tests that all exercise the same happy path.
- **Do:** Weight test effort toward hard branches. A single test covering an uncovered `except` clause is worth more than five more tests on the main path.
- **Do:** Include both positive assertions (expected output) and negative assertions (`assertRaises`) -- functionality-aware testing requires checking that wrong code actually fails.
- **Avoid:** Generating tests without assertions. A test that just calls the function without checking the result provides zero verification signal.
- **Avoid:** Treating all branches equally. Linear coverage counting masks the fact that the last 10% of branches are exponentially harder and more informative.
- **Avoid:** Relying on a single test suite for selection. One suite can have systematic blind spots. Use at least 3 independent suites for majority voting.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| All tests pass for all candidates | Tests are too easy; only covering trivial branches | Reanalyze branch structure, add edge-case and boundary tests targeting hard branches |
| All tests fail for all candidates | Tests have bugs or wrong expected values | Verify test correctness against a known-good reference or manual calculation first |
| Test execution times out | Infinite loop in candidate or test triggers worst-case complexity | Add timeout decorators (`@unittest.timeout`) and treat timeouts as errors |
| Cannot distinguish top candidates | Majority vote is tied | Generate additional targeted test suites focusing specifically on branches where candidates diverge |
| Low branch coverage despite many tests | Tests cluster on same paths | Use a coverage tool (`coverage.py`) to identify uncovered lines, then write tests specifically for those paths |

## Limitations

- **Requires executable code.** This technique only works when candidate solutions can actually be run. It cannot verify pseudocode, partial implementations, or code with unresolvable dependencies.
- **Branch coverage is not correctness.** 100% branch coverage does not guarantee correct code -- a branch can be covered by a test with a wrong assertion. Always combine coverage with meaningful assertions.
- **Static complexity metrics are approximate.** Halstead Difficulty and Maintainability Index are heuristics. Some genuinely hard code (e.g., subtle off-by-one in a simple loop) may score as easy.
- **Majority voting needs enough suites.** With fewer than 3 independent test suites, voting becomes unreliable. For critical verification, use 5-10 suites.
- **Does not handle non-deterministic code.** If the target code involves randomness, concurrency, or external I/O, test results may be flaky and majority voting degrades.
- **Edge cases in the test generator.** The approach assumes test inputs can be constructed statically. For functions requiring complex object graphs or stateful setup, manual test scaffolding may be needed.

## Reference

**Paper:** [CVeDRL: An Efficient Code Verifier via Difficulty-aware Reinforcement Learning](https://arxiv.org/abs/2601.22803v1) (Shi et al., 2026). Key sections: Section 3 for the theoretical reward decomposition, Section 4.2 for the exponential branch-coverage shaping formula, and Section 4.3 for the Halstead/MI difficulty metrics. The core insight is that exponential reward shaping on branch coverage combined with static complexity weighting produces tests that focus on the hardest, most informative branches.