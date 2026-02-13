---
name: "consistency-meets-verification-enhancing"
description: "Generate high-reliability test suites without ground-truth implementations using the ConVerTest pipeline: Self-Consistency voting, Chain-of-Verification refinement, and Dual Execution Agreement. Use when asked to 'generate tests for this spec', 'write tests before implementation', 'create a test suite without reference code', 'test-driven development for this feature', 'generate reliable unit tests', or 'validate tests without a working implementation'."
---

# ConVerTest: Consistency-Driven Test Generation Without Ground Truth

This skill enables Claude to generate reliable, high-coverage test suites for functions or modules **before** a reference implementation exists. It applies the ConVerTest pipeline from Taherkhani et al. (2026), which combines three strategies — Self-Consistency majority voting, Chain-of-Verification iterative refinement, and Dual Execution Agreement — to produce tests that are valid, comprehensive, and free of hallucinated assertions. The technique improves test validity by up to 39%, line coverage by 28%, and mutation detection by 18% over naive LLM test generation.

## When to Use

- When the user asks to write tests from a specification, docstring, or requirements doc — with no working code yet
- When doing test-driven development (TDD) and tests must be written before the implementation
- When the user wants to validate that generated tests are internally consistent and not hallucinated
- When generating a test suite for a function signature + description and the user wants high confidence in assertion correctness
- When the user needs to generate both candidate implementations and tests simultaneously and cross-validate them
- When reviewing or hardening an existing test suite by re-generating and voting on expected outputs

## Key Technique

**The core problem:** When an LLM generates tests without a reference implementation, it frequently hallucinates expected values in assertions. A single-shot generation has no way to verify whether `assertEqual(result, 42)` is correct. ConVerTest solves this by generating many candidates and using statistical agreement as a proxy for correctness.

**The pipeline has two stages.** Stage 1 runs two parallel tracks: (a) *Self-Consistency (SC)* generates M diverse test stubs (setup + inputs + function calls, no assertions), then for each stub prompts the LLM N times to complete the assertions. The completions are parsed into ASTs, grouped by functionally identical logic, and the most frequent assertion set wins via majority vote. (b) *Chain-of-Verification (CoVe)* generates Z candidate code implementations, each refined through an iterative loop: generate a baseline solution, formulate verification questions (edge cases, constraint adherence, error handling), answer those questions against the code, and regenerate if issues are found.

**Stage 2 performs Dual Execution Agreement.** Every (code candidate, test case) pair is executed, producing a pass/fail matrix. Code candidates are clustered into agreement sets — groups that produce identical pass/fail patterns across all tests. Each set is scored by `(tests_passed) * sqrt(set_size)`, rewarding both robustness (many tests passed) and consensus (many implementations agree). The highest-scoring set's representative solution becomes the reference, and tests are labeled valid if they pass against it. This eliminates tests with hallucinated expectations and implementations with subtle bugs in one step.

## Step-by-Step Workflow

1. **Extract the specification.** Parse the user's input to isolate the function signature, docstring, parameter types, return type, and any behavioral constraints. If only a natural-language description exists, formalize it into a clear spec with input/output contracts.

2. **Generate M diverse test stubs (SC Stage 1).** Produce M (aim for 5-8) test skeletons that each set up different input scenarios: typical cases, boundary values, error conditions, empty inputs, large inputs. Each stub includes `setUp`, input variables, and the function call — but **no assertions yet**. Vary the scenarios deliberately to maximize coverage diversity.

3. **Complete each stub N times via majority voting (SC Stage 2).** For each of the M stubs, generate N (aim for 5-7) independent completions that add assertions. Parse each completion's assertions, group functionally identical ones (same expected values and assertion types), and select the assertion set with the highest vote count. Discard stubs where no assertion receives a clear majority (>50% agreement).

4. **Generate Z candidate implementations via CoVe.** Produce Z (aim for 3-5) independent code implementations of the spec. For each candidate, run a verification loop: (a) generate the baseline code, (b) formulate 4-6 verification questions covering correctness, edge cases, constraint adherence, and error handling, (c) answer each question against the code, (d) if any answer reveals an issue, regenerate with the verification context appended to the prompt.

5. **Build the execution matrix.** Execute every (implementation, test) pair. Record pass/fail for each cell. This produces a Z x M matrix.

6. **Cluster into agreement sets.** Group implementations that have identical pass/fail vectors across all tests. Two implementations are in the same set only if they agree on every single test outcome.

7. **Score and select the consensus set.** Score each agreement set as `tests_passed * sqrt(set_size)`. Select the highest-scoring set. Pick any member of that set as the representative implementation.

8. **Filter tests by validity.** Label each test as valid (passes against the representative) or invalid (fails). Discard invalid tests — these likely contain hallucinated assertions.

9. **Output the final test suite.** Return the validated tests as a clean, runnable test file. Include a brief comment on each test noting the input scenario it covers. Optionally return the representative implementation as a reference.

10. **Report coverage and confidence.** Summarize how many stubs were generated, how many survived voting, the agreement set size, and the consensus score. Flag any tests that had weak majority votes (<60% agreement) as lower-confidence.

## Concrete Examples

**Example 1: TDD for a string utility**

User: "Write tests for a function `anagram_combos(sub_words: list[str], whole_word: str) -> list[str]` that returns all permutations of sub_words whose concatenation equals whole_word."

Approach:
1. Extract spec: takes a list of substrings and a target string, returns permutations that concatenate to the target.
2. Generate 6 test stubs covering: basic match, no valid permutations, single-element list, duplicate sub_words, empty list, all permutations valid.
3. For each stub, generate 5 completions with assertions. For the basic case, 4/5 completions agree that `anagram_combos(["ab", "cd"], "abcd")` returns `["abcd"]` — select this.
4. Generate 3 candidate implementations. CoVe catches that one candidate doesn't handle duplicate sub_words; it's regenerated.
5. Run all (3 implementations, 6 tests). Two implementations agree on all 6 tests; one disagrees on the duplicate case. Agreement set of size 2 scores highest.
6. Filter: all 6 tests pass against the consensus set. Final suite has 6 valid tests.

Output:
```python
import unittest
from solution import anagram_combos

class TestAnagramCombosBasic(unittest.TestCase):
    def test_single_valid_permutation(self):
        result = anagram_combos(["ab", "cd"], "abcd")
        self.assertEqual(sorted(result), ["abcd"])

    def test_multiple_valid_permutations(self):
        result = anagram_combos(["ab", "cd", "ef"], "abcdef")
        self.assertIn("abcdef", result)
        self.assertTrue(all(len(p) == 6 for p in result))

    def test_no_valid_permutation(self):
        result = anagram_combos(["ab", "cd"], "abdc")
        self.assertEqual(result, [])

    def test_single_element_list(self):
        result = anagram_combos(["hello"], "hello")
        self.assertEqual(result, ["hello"])

    def test_empty_list(self):
        result = anagram_combos([], "abc")
        self.assertEqual(result, [])

    def test_duplicate_sub_words(self):
        result = anagram_combos(["ab", "ab"], "abab")
        self.assertEqual(len(result), 1)  # only one unique concatenation
```

**Example 2: Generating tests for a data processing function from a docstring**

User: "Generate a test suite for this spec: `def merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]` — merges overlapping intervals, returns sorted non-overlapping intervals."

Approach:
1. Spec: merge overlapping `(start, end)` tuples, return sorted merged result.
2. Generate 7 stubs: no overlap, full overlap, partial overlap, single interval, empty list, adjacent intervals (e.g., `(1,3)` and `(3,5)`), nested intervals.
3. Complete each stub 5 times. The "adjacent intervals" stub shows 3/5 agreement that `(1,3)` and `(3,5)` merge into `(1,5)` — marginal majority. Flag as lower-confidence. All other stubs show 4/5+ agreement.
4. Generate 4 CoVe implementations. Verification catches that one handles the adjacent case differently (doesn't merge touching intervals). This creates two distinct agreement sets.
5. Agreement set A (3 implementations, passes 6/7 tests) scores `6 * sqrt(3) = 10.4`. Set B (1 implementation, passes 7/7) scores `7 * sqrt(1) = 7.0`. Set A wins.
6. Filter: 6 tests valid, 1 test (adjacent-merge) is invalid against Set A's consensus. Discard it but report the ambiguity to the user.

Output:
```python
import unittest
from solution import merge_intervals

class TestMergeIntervals(unittest.TestCase):
    def test_no_overlap(self):
        self.assertEqual(merge_intervals([(1, 2), (5, 7)]), [(1, 2), (5, 7)])

    def test_partial_overlap(self):
        self.assertEqual(merge_intervals([(1, 4), (3, 6)]), [(1, 6)])

    def test_full_overlap(self):
        self.assertEqual(merge_intervals([(1, 10), (3, 5)]), [(1, 10)])

    def test_single_interval(self):
        self.assertEqual(merge_intervals([(2, 5)]), [(2, 5)])

    def test_empty_list(self):
        self.assertEqual(merge_intervals([]), [])

    def test_nested_intervals(self):
        self.assertEqual(
            merge_intervals([(1, 10), (2, 3), (4, 8)]), [(1, 10)]
        )

# NOTE: Adjacent interval behavior (e.g., (1,3) + (3,5)) had weak
# consensus (3/5 votes). The spec should clarify whether touching
# endpoints merge. This test was excluded from the validated suite.
```

**Example 3: Hardening an existing test suite**

User: "I have these 4 tests for `calculate_discount(price, tier)` but I'm not sure the expected values are right. Can you verify them?"

Approach:
1. Treat the user's 4 tests as the initial stub set (M=4). Skip stub generation.
2. For each test, generate 5 independent completions of the expected values based on the spec. Compare the user's expected values against the majority vote.
3. Generate 3 CoVe implementations of `calculate_discount`.
4. Run the execution matrix. If the user's assertion disagrees with the majority AND fails against the consensus implementation, flag it as likely incorrect and suggest the majority-voted value.
5. Report: "3 of your 4 tests match consensus. Test #2 expects 15.0 but 4/5 completions and 3/3 implementations agree on 12.0. The discount tier 'silver' likely applies a 20% discount, not 25%."

## Best Practices

- **Do:** Generate test stubs and assertion completions as separate steps. Decoupling exploration (what to test) from assertion (what to expect) produces more diverse, higher-coverage suites.
- **Do:** Parse completions into AST-level comparison rather than string matching. Two assertions like `assertEqual(result, [1, 2])` and `assertEqual(result, [1,2])` are functionally identical.
- **Do:** Report confidence levels. A 5/5 vote is much stronger than a 3/5 vote. Flag marginal tests so the user can review them.
- **Do:** Use the execution matrix to detect spec ambiguities. When implementations split into multiple agreement sets of similar size, the spec likely has an underspecified edge case — surface this to the user.
- **Avoid:** Treating the consensus implementation as "correct." It's the best available proxy, not ground truth. Always note that tests are validated against a generated reference, not a proven one.
- **Avoid:** Generating all completions with identical prompting. Vary temperature or phrasing slightly across the N completions to get genuine diversity, not N copies of the same reasoning path.

## Error Handling

- **No majority emerges for a stub:** If no assertion gets >50% of votes across N completions, discard that test case and note it as "ambiguous specification behavior." Do not guess.
- **All implementations fail a test:** This likely means the test has a hallucinated assertion. Remove it rather than assuming all code is wrong.
- **No agreement set forms (all implementations disagree):** The spec is likely too ambiguous to test reliably. Ask the user to clarify the expected behavior for specific edge cases before proceeding.
- **Execution errors (import failures, syntax errors):** Re-run the CoVe verification loop specifically targeting the error. If a test has a syntax error in >50% of completions, the stub itself may be malformed — regenerate it.
- **Single-member agreement sets only:** Consensus is weak. Increase Z (generate more implementations) or simplify the spec. Report low confidence to the user.

## Limitations

- **Spec ambiguity amplifies errors.** If the specification is genuinely ambiguous (e.g., doesn't define behavior for negative inputs), majority voting may converge on an arbitrary interpretation. ConVerTest cannot invent requirements.
- **Computationally expensive.** Generating M stubs x N completions + Z implementations + full execution matrix requires many LLM calls. For simple functions, single-shot generation may be sufficient — use ConVerTest for functions where correctness matters and the spec is non-trivial.
- **Language and framework scope.** The technique works best for pure functions with deterministic outputs. Non-deterministic behavior (random outputs, time-dependent results, external API calls) breaks majority voting on expected values. Mock external dependencies first.
- **Agreement set scoring favors majority behavior.** If the correct behavior is rare (only 1 of 5 implementations gets it right), ConVerTest will select the wrong consensus. This is inherent to voting-based approaches.
- **Cannot replace domain expertise.** The pipeline validates internal consistency, not correctness against real-world requirements. A domain expert should still review the final test suite for business logic accuracy.

## Reference

Taherkhani, H., DaghighFarsoodeh, A., Chowdhury, M., Pham, H. V., & Hemmati, H. (2026). *Consistency Meets Verification: Enhancing Test Generation Quality in Large Language Models Without Ground-Truth Solutions.* arXiv:2602.10522v1. [https://arxiv.org/abs/2602.10522v1](https://arxiv.org/abs/2602.10522v1)

Key sections: Section 3 for the full pipeline algorithm, Section 4 for the dual execution agreement scoring formula `score = tests_passed * sqrt(set_size)`, and Section 5 ablations showing each component's individual contribution.