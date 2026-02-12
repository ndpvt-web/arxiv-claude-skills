---
name: "swe-world-building-software-engineering"
description: >
  Build and train software engineering agents without Docker by replacing containerized execution
  with learned surrogate models that predict execution outcomes and test feedback. Apply the
  SWE-World framework to solve code modification tasks, evaluate patches without running tests,
  and scale inference by scoring multiple candidate solutions.
  Trigger phrases: "fix this bug without running tests", "evaluate my patch", "predict test
  outcomes for this change", "score candidate patches", "Docker-free agent training",
  "test-time scaling for code fixes"
---

# SWE-World: Docker-Free Software Engineering Agents

This skill enables Claude to apply the SWE-World framework for solving software engineering tasks
without relying on Docker containers or physical test execution. The core technique replaces
containerized environments with two learned surrogate models -- a Transition Model (SWT) that
predicts step-level execution feedback (stdout/stderr/exit code) and a Reward Model (SWR) that
simulates unit test results and assigns binary pass/fail rewards. This allows evaluating code
patches, selecting the best fix among candidates, and reasoning about execution outcomes purely
through learned prediction, without ever instantiating a test environment.

## When to Use

- When the user asks to fix a bug or modify code in a repository where setting up the full test environment is impractical or unavailable
- When evaluating whether a proposed patch will pass tests without actually running pytest or the project's test suite
- When the user wants to generate multiple candidate fixes and select the best one (test-time scaling)
- When building or training an SWE agent pipeline and needs to replace Docker-based evaluation with a surrogate
- When reasoning about what stdout/stderr a code change would produce without executing it
- When the user needs to triage multiple patches for a GitHub issue and rank them by likely correctness

## Key Technique

**Surrogate Environment Architecture.** SWE-World decomposes the execution environment into three tiers. (1) A **Sandbox** handles deterministic file-system operations -- `ls`, `cat`, `grep`, `str_replace`, `view`, `create` -- by maintaining a mutable workspace that tracks edits faithfully with zero hallucination risk. (2) A **Transition Model (SWT)** handles repository-specific commands like `python` or `pytest` that normally require a dependency-complete environment. SWT takes the instance metadata, the agent's current accumulated patch, and the command as context, then predicts `{"stdout": ..., "stderr": ..., "exit_code": ...}`. (3) A **Reward Model (SWR)** acts as a virtual test runner at episode end: it receives the final patch and simulates execution of the project's unit tests, producing a structured test report covering Fail-to-Pass and Pass-to-Pass test categories, plus a binary reward signal (0 or 1).

**Training and Scaling.** Both SWT and SWR are trained on real agent-Docker interaction traces collected from open-source datasets (R2E-Gym, SWE-Gym, SWE-rebench). SWR uses Chain-of-Thought (CoT) reasoning -- reverse-reasoning distillation generates intermediate logical steps before the structured output, boosting reward prediction accuracy from 0.578 to 0.712. For test-time scaling (TTS), the agent generates N candidate trajectories (e.g., N=8), each is scored M times by SWR (e.g., M=3), and the trajectory with the highest average reward is selected. This raised resolve rate from 55.0% to 68.2% on SWE-bench Verified without any Docker execution.

**Why This Matters Practically.** The insight is that execution feedback is often predictable from context -- the issue description, the codebase structure, the patch diff, and the test file contents carry enough signal for a strong model to predict what would happen. This means you can reason about "will this fix work?" by analyzing the patch against the test expectations, rather than running the tests.

## Step-by-Step Workflow

1. **Analyze the issue and repository structure.** Read the bug report or feature request. Use `find_file`, `grep`, and `cat` equivalents to locate the relevant source files, test files, and any stack traces or error messages referenced in the issue.

2. **Identify the Fail-to-Pass tests.** Find the specific test(s) that currently fail due to the bug. Read their assertions, expected values, and the code paths they exercise. These are the primary success criteria.

3. **Identify Pass-to-Pass tests.** Locate other tests in the same module or package that currently pass. Any fix must not break these. Catalog them as regression constraints.

4. **Formulate a hypothesis about the root cause.** Based on the issue description and failing test expectations, trace the code path to identify where the behavior diverges from what the tests expect. Document the specific function, line, and logic error.

5. **Generate the patch using `str_replace` edits.** Make minimal, targeted changes to fix the root cause. Accumulate edits as a diff. Prefer the smallest change that addresses the failing tests without touching unrelated code.

6. **Predict execution outcomes without running tests.** For each edit, reason about what stdout/stderr the test runner would produce. Ask: "Given this patch, would the Fail-to-Pass tests now pass? Would any Pass-to-Pass tests break?" Trace the logic manually through the changed code path.

7. **Generate multiple candidate patches if uncertain.** When the fix is ambiguous, produce 2-4 alternative patches with different approaches (e.g., different boundary conditions, different error handling strategies).

8. **Score each candidate against test expectations.** For each candidate patch, simulate the test report: walk through each Fail-to-Pass test with the patched code and predict PASSED/FAILED. Walk through Pass-to-Pass tests and verify no regressions. Assign a confidence score.

9. **Select the highest-scoring candidate.** Choose the patch with the highest predicted pass rate. If scores tie, prefer the minimal patch (fewer lines changed, fewer side effects).

10. **Present the solution with predicted test outcomes.** Show the final patch as a diff, include the predicted test report (which tests pass, which might still fail), and flag any uncertainty or edge cases that warrant actual execution to confirm.

## Concrete Examples

**Example 1: Fixing a string parsing bug without running tests**

```
User: This function `parse_duration("3h30m")` returns 330 instead of 12600.
The test `test_parse_duration_hours_minutes` is failing. Fix it without
running the test suite -- we can't set up the environment right now.

Approach:
1. Read parse_duration() in utils/time.py -- find it multiplies hours by 100
   instead of 3600
2. Read test_parse_duration_hours_minutes -- confirms expected output is
   3*3600 + 30*60 = 12600
3. Check other tests (test_parse_duration_minutes_only, test_parse_duration_seconds)
   to identify regression risk
4. Generate patch: change `hours * 100` to `hours * 3600`
5. Predict outcomes:
   - test_parse_duration_hours_minutes: PASSED (3*3600 + 30*60 = 12600)
   - test_parse_duration_minutes_only: PASSED (unchanged code path)
   - test_parse_duration_seconds: PASSED (unchanged code path)

Output:
--- a/utils/time.py
+++ b/utils/time.py
@@ -15,7 +15,7 @@
 def parse_duration(s: str) -> int:
     total = 0
     if hours:
-        total += int(hours) * 100
+        total += int(hours) * 3600
     if minutes:
         total += int(minutes) * 60
     return total

Predicted test report:
  PASSED: test_parse_duration_hours_minutes (F2P)
  PASSED: test_parse_duration_minutes_only (P2P)
  PASSED: test_parse_duration_seconds (P2P)
  Confidence: HIGH -- single arithmetic constant fix, no side effects
```

**Example 2: Selecting best patch among candidates via test-time scoring**

```
User: The CSV export crashes on empty datasets. I have three possible fixes
from my team. Which one is most likely correct?

Patch A: Adds `if not data: return ""` at the top of export_csv()
Patch B: Wraps the header generation in `if data else []`
Patch C: Changes the for-loop to handle empty iterables with a default

Approach:
1. Read the failing test: test_export_csv_empty expects an empty string output
2. Read test_export_csv_with_data: expects "name,age\nAlice,30\n"
3. Read test_export_csv_headers_only: expects "name,age\n" (headers, no rows)
4. Score each patch:
   - Patch A: F2P passes (returns "" for empty). But test_export_csv_headers_only
     would FAIL -- headers-only dataset is truthy but has no rows, and this
     patch bypasses header generation entirely. Score: 1/3 tests impacted.
   - Patch B: F2P passes (empty data -> no headers). But test_export_csv_headers_only
     FAILS -- headers should still appear for headers-only case. Score: 1/3.
   - Patch C: F2P passes (empty iterable handled). test_export_csv_with_data
     PASSES (loop runs normally). test_export_csv_headers_only PASSES (headers
     generated, loop produces nothing). Score: 3/3.
5. Select Patch C as the best candidate.

Output:
Recommended: Patch C
  Predicted: 3/3 tests pass (F2P + all P2P)
  Reason: Handles empty case without breaking header generation for
  non-empty datasets. Patches A and B both break test_export_csv_headers_only
  because they conflate "empty data" with "no headers needed."
```

**Example 3: Reasoning about execution feedback for a command**

```
User: I added a new regex pattern to the validator. What would happen if I
run `python -m pytest tests/test_validator.py::test_email_validation -v`?

Approach:
1. Read the user's patch to the validator module
2. Read test_email_validation to understand test cases and assertions
3. Trace each test case through the new regex:
   - "user@example.com" -> matches -> assert valid: PASS
   - "user@.com" -> does NOT match with new regex -> assert invalid: PASS
   - "user@exam ple.com" -> new regex rejects spaces -> assert invalid: PASS
   - "user+tag@example.com" -> new regex lacks + in character class ->
     assert valid: FAIL (regex rejects it, but test expects valid)
4. Predict stdout:
   FAILED tests/test_validator.py::test_email_validation[user+tag] -
   AssertionError: Expected True but got False
   1 failed, 3 passed

Output:
Predicted execution:
  stdout: "FAILED test_email_validation[user+tag@example.com]"
  stderr: "AssertionError: assert False == True"
  exit_code: 1

Root cause: Your regex character class `[a-zA-Z0-9._-]` before the @
is missing the `+` character. Change to `[a-zA-Z0-9._+-]` to fix.
```

## Best Practices

- **Do:** Read the actual test files before predicting outcomes. The test assertions are the ground truth for what "correct" means -- never guess at expected values.
- **Do:** Separate Fail-to-Pass tests (the bug's tests) from Pass-to-Pass tests (regression guards). Both matter equally for patch correctness.
- **Do:** Generate multiple candidate patches when the root cause is ambiguous, then score each against the full test suite expectations. This is the TTS principle.
- **Do:** Use Chain-of-Thought reasoning when predicting test outcomes -- trace the code path step by step rather than pattern-matching on the diff.
- **Avoid:** Predicting outcomes for commands that depend on external state (network calls, database contents, filesystem timestamps) -- these are not reliably predictable from code alone.
- **Avoid:** Overconfidence in predictions. Always flag uncertainty levels. A predicted "PASS" on a complex integration test is less reliable than on a unit test with simple assertions.
- **Avoid:** Making large, sweeping patches. The SWE-World framework demonstrates that minimal patches are both easier to evaluate and more likely to be correct.

## Error Handling

- **Ambiguous test expectations:** When tests use dynamic fixtures, mock objects, or parameterized inputs that aren't fully visible, explicitly state which test parameters you can and cannot predict outcomes for.
- **Missing test files:** If the repository lacks tests for the affected code, fall back to manual reasoning about the code's behavior and recommend writing tests as part of the fix.
- **Complex execution dependencies:** When a command would trigger import chains, database migrations, or external service calls, acknowledge that surrogate prediction is unreliable and recommend actual execution for that specific step.
- **Conflicting patches:** When multiple candidates score equally, present all tied candidates with their tradeoffs rather than arbitrarily picking one.

## Limitations

- **Integration and end-to-end tests** that depend on runtime state, database fixtures, or network services cannot be reliably predicted from code analysis alone. This technique works best for unit tests with deterministic inputs and outputs.
- **Dynamic language features** (metaprogramming, monkey-patching, runtime code generation) reduce prediction accuracy because the execution path cannot be statically traced.
- **Large-scale refactors** touching dozens of files produce patches too complex for reliable manual test outcome prediction. The technique is most effective for focused, few-file changes.
- **Flaky tests** that depend on timing, ordering, or randomness are inherently unpredictable by any surrogate model.
- **Environment-specific bugs** (platform-dependent behavior, version-specific APIs) require knowing the exact runtime configuration, which may not be available from code alone.

## Reference

**Paper:** [SWE-World: Building Software Engineering Agents in Docker-Free Environments](https://arxiv.org/abs/2602.03419v1) (Sun et al., 2026)
**Key takeaway:** Execution feedback is learnable -- an LLM trained on real agent-environment traces can predict stdout/stderr/exit codes and test pass/fail with sufficient accuracy (0.77 reward accuracy) to train agents from 6.2% to 55.0% resolve rate without Docker, and to 68.2% with test-time scaling by scoring 8 candidate trajectories.
**Code:** [github.com/RUCAIBox/SWE-World](https://github.com/RUCAIBox/SWE-World)