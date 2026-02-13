---
name: "synthesizing-file-level-data-unit"
description: "Generate high-quality unit tests with self-debugging repair loops and chain-of-thought reasoning. Produces tests with meaningful assertions, high branch coverage, and strong mutation scores by iteratively diagnosing and fixing errors, assertion failures, and coverage gaps. Use when: 'generate unit tests for this file', 'write tests with good coverage', 'create tests that catch real bugs', 'improve my test assertions', 'debug and fix failing tests', 'generate tests with reasoning'."
---

# Self-Debugging Unit Test Generation with Chain-of-Thought Reasoning

This skill enables Claude to generate high-quality unit tests by applying a self-debugging repair loop inspired by the Repo-Smith data distillation pipeline. Instead of producing tests in a single shot, Claude iteratively generates a test, executes it, classifies any defects (errors, assertion failures, coverage gaps), repairs the test targeting the weakest metric, and compresses the debugging reasoning into a concise chain-of-thought that justifies the final test. This approach produces tests with correct, meaningful assertions rather than trivial `assertIsNotNone` checks.

## When to Use

- When the user asks to generate unit tests for a Python file, module, or focal method and wants tests that actually catch bugs
- When existing tests have low branch coverage and the user wants targeted tests for uncovered code paths
- When generated tests are failing and the user wants systematic diagnosis and repair rather than ad-hoc fixes
- When the user wants tests with strong mutation-killing assertions (tests that detect real code changes)
- When writing tests for complex functions with multiple branches, edge cases, or helper dependencies
- When the user asks to "write tests like a human developer would" with reasoning about why each assertion exists

## Key Technique

**Self-Debugging Guided Repair** replaces single-shot test generation with an iterative loop that classifies defects by type and repairs the weakest dimension first. After generating an initial test file, execute it and compute three relative scores against the desired quality: `S_pass` (fraction of assertions passing), `S_cov` (branch coverage achieved), and `S_mut` (mutation score). The metric with the lowest relative score determines the repair type: error-focused repair for syntax/runtime failures, failure-focused repair for wrong assertions, or coverage-focused repair for missed branches. This prioritization matters because failure debugging has the highest acceptance rate (20%+ per round) while other types stay below 10%—so the loop naturally spends effort where it yields the most improvement.

**CoT Compression** solves the problem of verbose, unfaithful reasoning. After each repair round, the original generation reasoning (R0) and the debugging reasoning (R1) are merged into a single compressed CoT (R2) that reads as if the correct test was generated in one pass. The compression prompt explicitly forbids referencing intermediate debugging steps, producing explanations that directly justify each assertion. This compressed reasoning is what makes the tests explainable—every assertion has a traceable rationale tied to the focal code's behavior.

The combination achieves substantially higher mutation scores (88.66%) than single-shot generation, meaning the tests detect real code changes rather than passing vacuously.

## Step-by-Step Workflow

1. **Identify the focal file and focal method(s).** Read the source file the user wants tested. Identify public methods, their signatures, return types, side effects, and branch points (if/else, loops, try/except). Note imports and dependencies that tests will need to mock or set up.

2. **Generate the initial test file with reasoning.** Write a complete test file including imports, any necessary helper functions or fixtures, and test methods targeting each branch of the focal code. For each test method, internally reason about: what behavior is being tested, what input triggers that branch, and what the expected output should be. Write this reasoning as inline comments above each test.

3. **Execute the test file and collect diagnostics.** Run the test with `pytest --tb=short` to detect errors. If available, run `pytest --cov=<module> --cov-branch` to measure branch coverage. Classify the result into one of four categories:
   - **Error**: Import failures, syntax errors, runtime exceptions (test cannot run)
   - **Failure**: Assertions fail (test runs but produces wrong expected values)
   - **Coverage gap**: Test passes but misses branches in the focal code
   - **Pass**: All assertions pass with adequate coverage

4. **Prioritize the weakest metric for repair.** If the test cannot execute at all, fix errors first (missing imports, incorrect setup). If assertions fail, diagnose each failure by examining the actual vs. expected values and fix the assertion logic. If coverage is low, identify the uncovered line ranges and write additional test cases targeting those specific paths.

5. **Apply targeted repair with diagnostic context.** For each repair round, provide the current test file plus the specific diagnostic:
   - For errors: the full traceback
   - For failures: the assertion diff (expected vs. actual)
   - For coverage gaps: the exact uncovered lines with their source code
   Rewrite only the affected test methods, preserving passing tests.

6. **Repeat the diagnose-repair cycle (up to 5 rounds).** After each repair, re-execute and re-classify. Stop early if all tests pass and coverage targets are met. Five rounds is the practical limit—diminishing returns set in after that.

7. **Compress the reasoning into final CoT comments.** After reaching a stable test file, rewrite the inline reasoning comments so they read as direct justifications for each assertion. Remove any references to "I fixed this because..." or debugging history. Each comment should explain the *why*: what behavior the assertion verifies and what code path it exercises.

8. **Validate assertion quality.** Review each assertion for meaningfulness: replace any `assertTrue(result is not None)` with specific value checks like `assertEqual(result.shape, (3, 3))`. Every assertion should fail if the focal method's logic changes—this is the mutation-killing criterion.

9. **Present the final test file with coverage summary.** Output the clean test file and a brief summary: number of test methods, assertions, measured branch coverage percentage, and which code paths are covered.

## Concrete Examples

**Example 1: Testing a matrix transformation function**

User: "Generate unit tests for this function in `transforms.py`"
```python
def reorder_axes(data, affine):
    """Reorder data array axes to match a canonical orientation."""
    axis_order = _get_axis_order(affine)
    if axis_order == [0, 1, 2]:
        return data, affine
    reordered = np.transpose(data, axis_order)
    new_affine = affine[:, axis_order + [3]]
    return reordered, new_affine
```

Approach:
1. Identify branches: early return when axes are already canonical, transpose path when they are not
2. Generate initial test with both paths covered
3. Execute—discover `_get_axis_order` is not imported, fix the import error
4. Re-execute—assertion on `new_affine` shape fails because column indexing needs a list, fix expected value
5. Check coverage—early return branch is covered but the swap path lacks edge cases for all 6 permutations, add parameterized tests
6. Compress reasoning into comments explaining each permutation

Output:
```python
import numpy as np
import pytest
from transforms import reorder_axes

class TestReorderAxes:
    def test_canonical_order_returns_unchanged(self):
        # When affine already maps axes in order [0,1,2], data passes through untouched
        data = np.arange(24).reshape(2, 3, 4)
        affine = np.eye(4)
        result_data, result_affine = reorder_axes(data, affine)
        np.testing.assert_array_equal(result_data, data)
        np.testing.assert_array_equal(result_affine, affine)

    @pytest.mark.parametrize("perm", [
        [1, 0, 2], [2, 0, 1], [0, 2, 1], [2, 1, 0], [1, 2, 0]
    ])
    def test_noncanonical_order_transposes_data(self, perm):
        # Each axis permutation must produce transposed data and reordered affine columns
        data = np.arange(24).reshape(2, 3, 4)
        affine = np.eye(4)[np.array([*perm, 3]), :]
        result_data, result_affine = reorder_axes(data, affine)
        assert result_data.shape == tuple(data.shape[i] for i in perm)
        # Reordered affine should restore diagonal structure
        assert np.allclose(result_affine[:3, :3], np.eye(3), atol=1e-10)
```

**Example 2: Repairing a failing test for a string parser**

User: "My test for `parse_config` is failing, can you debug and fix it?"
```python
# Failing test
def test_parse_config_with_comments():
    raw = "key=value\n# comment\nfoo=bar"
    result = parse_config(raw)
    assert result == {"key": "value", "# comment": "", "foo": "bar"}
```

Approach:
1. Classify as failure-focused: the assertion is wrong, not the code
2. Read the focal method—`parse_config` skips lines starting with `#`
3. Fix the expected value to exclude the comment line
4. Check coverage—the empty-line branch is untested, add a case
5. Compress reasoning: comment explains that `#` lines are filtered, not parsed

Output:
```python
def test_parse_config_with_comments():
    # Lines starting with '#' are treated as comments and excluded from output
    raw = "key=value\n# comment\nfoo=bar"
    result = parse_config(raw)
    assert result == {"key": "value", "foo": "bar"}
    assert "# comment" not in result

def test_parse_config_with_blank_lines():
    # Blank lines between entries are silently skipped
    raw = "key=value\n\n\nfoo=bar"
    result = parse_config(raw)
    assert result == {"key": "value", "foo": "bar"}
```

**Example 3: Coverage-focused repair for an HTTP client**

User: "Write tests for `fetch_json` that cover the error handling branches"
```python
def fetch_json(url, timeout=30):
    try:
        resp = requests.get(url, timeout=timeout)
        resp.raise_for_status()
        return resp.json()
    except requests.Timeout:
        raise TimeoutError(f"Request to {url} timed out after {timeout}s")
    except requests.HTTPError as e:
        raise ValueError(f"HTTP {e.response.status_code} from {url}")
```

Approach:
1. Identify three branches: success, timeout, HTTP error
2. Generate initial tests mocking `requests.get` for each path
3. Execute—all pass but branch coverage shows `timeout` parameter not exercised with non-default value
4. Add a test with `timeout=5` to cover that parameter path
5. Validate assertions: timeout test checks the exact error message includes the timeout value, HTTP error test checks status code is embedded

Output:
```python
from unittest.mock import patch, Mock
import pytest
from client import fetch_json

class TestFetchJson:
    @patch("client.requests.get")
    def test_success_returns_parsed_json(self, mock_get):
        mock_get.return_value = Mock(json=lambda: {"ok": True}, raise_for_status=lambda: None)
        assert fetch_json("https://api.example.com/data") == {"ok": True}

    @patch("client.requests.get", side_effect=__import__('requests').Timeout)
    def test_timeout_raises_with_url_and_duration(self, mock_get):
        # TimeoutError message must include both the URL and the timeout value
        with pytest.raises(TimeoutError, match=r"timed out after 5s"):
            fetch_json("https://api.example.com/data", timeout=5)

    @patch("client.requests.get")
    def test_http_error_raises_with_status_code(self, mock_get):
        resp = Mock()
        resp.status_code = 403
        resp.raise_for_status.side_effect = __import__('requests').HTTPError(response=resp)
        mock_get.return_value = resp
        # ValueError message must embed the actual HTTP status code
        with pytest.raises(ValueError, match="HTTP 403"):
            fetch_json("https://api.example.com/secret")
```

## Best Practices

- **Do:** Write assertions that check specific values, shapes, types, and error messages—never just `assertIsNotNone` or `assertTrue(result)`. Every assertion should fail if someone changes a single line in the focal method.
- **Do:** Run the test after each repair round rather than accumulating fixes. A fix for one failure can introduce new errors.
- **Do:** Include the exact uncovered source lines when prompting for coverage repair. Vague instructions like "improve coverage" produce worse results than "lines 42-47 are uncovered, these lines handle the empty-input case."
- **Do:** Compress debugging history into clean comments. The final test file should read as if written correctly the first time.
- **Avoid:** Generating tests without reading the focal code first. Tests written from function signatures alone produce trivial or incorrect assertions.
- **Avoid:** More than 5 repair rounds on a single test file. If a test still fails after 5 iterations, the problem is likely in the test design (wrong mocking strategy, misunderstood API), not a fixable assertion.
- **Avoid:** Leaving debugging artifacts in the final output—no `# FIXED:` comments, no commented-out old assertions, no `TODO` markers from intermediate rounds.

## Error Handling

| Problem | Diagnosis | Resolution |
|---------|-----------|------------|
| Import errors in test file | Missing dependency or wrong module path | Read the project structure to find correct import paths; check `__init__.py` files |
| All assertions fail | Likely misunderstanding of the focal method's return type or side effects | Re-read the focal method, trace through with a concrete input manually, then rewrite expected values |
| Coverage stuck at same percentage | New tests duplicate existing paths instead of targeting uncovered branches | Examine the uncovered line ranges explicitly; often they involve error handling, default parameters, or early returns |
| Mocking doesn't intercept calls | Mock target path is wrong (must patch where the name is looked up, not where it is defined) | Use `patch("module_under_test.dependency")` not `patch("dependency_module.function")` |
| Tests pass but are meaningless | Assertions are too weak (e.g., `assert result is not None`) | Apply the mutation test: would this assertion still pass if someone changed `>` to `>=` in the focal code? If yes, strengthen it |

## Limitations

- This workflow is most effective for **Python** projects with pytest. The repair loop depends on executing tests and parsing pytest output; other languages need adapted tooling.
- **Heavily mocked tests** can pass all assertions while testing nothing real. If a focal method has complex external dependencies, prefer integration-style tests or clearly document what each mock represents.
- The technique cannot generate tests for **non-deterministic** code (random outputs, time-dependent behavior) without the user providing seeding or freezing strategies.
- **Private/internal methods** with no clear contract are poor candidates—the assertions would encode implementation details rather than behavior.
- Branch coverage as measured by pytest-cov may not reflect **semantic coverage** (e.g., different data distributions through the same branch). High coverage numbers do not guarantee bug-catching power; mutation score is the better indicator.

## Reference

[Synthesizing File-Level Data for Unit Test Generation with Chain-of-Thoughts via Self-Debugging](https://arxiv.org/abs/2602.03181v1) — Hua et al., 2026. Key insight: iteratively classifying test defects into error/failure/coverage categories and repairing the weakest metric first, combined with compressing multi-round debugging reasoning into single-pass CoT, produces tests with 88.66% mutation score—substantially outperforming single-shot generation from frontier models.