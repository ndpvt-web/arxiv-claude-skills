---
name: "omnicode-benchmark-evaluating-software"
description: "Evaluate and improve code across four software engineering dimensions: bug fixing, test generation, code review fixing, and style fixing — using the OmniCode multi-task framework for Python, Java, and C++. Use when user says 'evaluate my code quality', 'run a full SE audit', 'generate tests for this repo', 'fix style violations', 'review and fix this code', or 'benchmark my agent on SE tasks'."
---

# OmniCode: Multi-Dimensional Software Engineering Evaluation

This skill enables Claude to systematically evaluate and improve codebases across the four core dimensions identified by the OmniCode benchmark: **bug fixing**, **test generation**, **code review fixing**, and **style fixing**. Rather than treating code quality as a single axis (does it compile? does the patch apply?), this approach mirrors how real software engineers work — simultaneously reasoning about correctness, test coverage, reviewer feedback, and style conformance across Python, Java, and C++. The technique exposes blind spots that single-task evaluations miss, such as an agent that can fix bugs but cannot write the tests that would have caught them.

## When to Use This Skill

- When the user asks to **audit a codebase** for quality across multiple dimensions (not just "find bugs")
- When the user wants to **generate comprehensive tests** that catch real defects, not just achieve line coverage
- When the user needs to **fix linter/style violations** without breaking functionality (Pylint, Checkstyle, PMD, Clang-Tidy)
- When the user asks to **respond to code review feedback** by producing an improved patch that addresses reviewer comments
- When the user wants to **benchmark or evaluate a coding agent** on diverse SE tasks beyond patch generation
- When the user needs to **synthetically generate SE tasks** (bug-seeding, style-breaking, test-gap creation) from an existing codebase for training or evaluation purposes
- When the user asks to improve code that spans **multiple languages** (Python, Java, C++) in a single project

## Key Technique

OmniCode's core insight is that software engineering ability is not one-dimensional. Prior benchmarks like SWE-Bench measure only patch generation — whether an agent can produce a diff that makes failing tests pass. OmniCode decomposes SE competence into four orthogonal categories, each with distinct evaluation criteria:

1. **Bug Fixing**: Given a buggy repository and issue description, produce a minimal patch. Evaluated by running the repository's existing test suite — the patch must make failing tests pass without breaking passing tests (`FAIL_TO_PASS` and `PASS_TO_PASS` test sets).
2. **Test Generation**: Given a repository and a bug description, write new tests using the project's existing test framework (pytest, Maven/Gradle, CMake). Tests are evaluated with a dual criterion: they must *fail* on the buggy code and *pass* on the fixed code, ensuring they actually detect the defect rather than being vacuous.
3. **Code Review Fixing**: Given a failed patch plus reviewer comments identifying specific problems, produce an improved patch that addresses the reviewer's feedback while still fixing the original issue. This evaluates the ability to incorporate human feedback into code changes.
4. **Style Fixing**: Given code with linter violations, fix all violations without altering program behavior. Evaluated by (a) re-running the linter to confirm zero violations and (b) re-running the test suite to confirm no regressions. Language-specific tools: Pylint (Python), Checkstyle/PMD (Java), Clang-Tidy (C++).

The second key contribution is a **synthetic task generation framework** that creates realistic tasks from limited real-world data. This works by mining real repositories for authentic patterns (bugs from actual commits, style from actual linter runs), then instrumenting the code to produce controlled variants — for instance, generating ~25 "bad patches" per bug instance using LLMs, then validating each against the test suite. This avoids data leakage (tasks are not in training sets) while maintaining real-world difficulty distributions.

## Step-by-Step Workflow

### For Evaluating/Improving a Codebase

1. **Identify the language and build system.** Determine if the project uses Python (pytest), Java (Maven/Gradle), or C++ (CMake). This dictates which linters and test frameworks apply.

2. **Run the existing test suite to establish a baseline.** Record which tests pass and which fail. This creates the `PASS_TO_PASS` baseline — any changes must not regress these.

3. **Perform bug-fixing analysis.** For each known issue or failing test, isolate the minimal code change needed. Produce a unified diff (`model_patch`) that targets only the affected files. Verify the patch by running the full test suite.

4. **Generate defect-detecting tests.** For each bug fix, write at least one test that (a) fails on the buggy version and (b) passes on the fixed version. Use the project's existing test framework and conventions — match the test file naming, assertion style, and fixture patterns already in use.

5. **Run language-specific linters to surface style violations.**
   - Python: `pylint --output-format=json <files>`
   - Java: `checkstyle -c /google_checks.xml <files>` and/or `pmd check -R rulesets/java/quickstart.xml -d <files>`
   - C++: `clang-tidy <files> -- <compile_flags>`

6. **Fix style violations one category at a time.** Group violations by rule (e.g., all `missing-docstring`, all `line-too-long`). Apply fixes, then re-run both the linter (confirm violation count drops to zero) and the test suite (confirm no regressions).

7. **Simulate code review feedback.** If a proposed patch exists, review it for common problems: incomplete fixes, introduced side effects, violated conventions. Produce specific, actionable reviewer comments, then generate an improved patch addressing each comment.

8. **Produce a structured evaluation report.** For each dimension, report:
   - Bug Fixing: number of issues resolved / total, with `FAIL_TO_PASS` and `PASS_TO_PASS` results
   - Test Generation: coverage delta and mutation kill rate of new tests
   - Style Fixing: violation count before/after, zero-regression confirmation
   - Review Fixing: number of reviewer comments addressed in the revised patch

9. **Validate all changes in isolation.** Apply each category's changes independently to the base commit. Run the full test suite for each to ensure no cross-contamination between fix types.

10. **Prioritize by impact.** Rank findings by severity: correctness bugs first, then missing test coverage for critical paths, then review-flagged issues, then style violations.

### For Benchmarking a Coding Agent

1. **Prepare instance files.** Structure tasks as JSON with fields: `instance_id`, `repo`, `base_commit`, `patch` (ground truth), `test_patch`, `FAIL_TO_PASS`, `PASS_TO_PASS`.
2. **Run the agent on each task category separately** (`--BugFixing`, `--TestGeneration`, `--StyleReview`, `--ReviewFix`).
3. **Collect predictions** in the format: `{"instance_id": "...", "model_patch": "...", "model_name_or_path": "..."}`.
4. **Evaluate using containerized execution** (Docker or Apptainer) to ensure reproducibility.
5. **Report per-category, per-language pass rates** to identify specific weaknesses.

## Concrete Examples

**Example 1: Multi-dimensional audit of a Python module**

User: "Audit this Python module for bugs, missing tests, and style issues"

Approach:
1. Read the module and its existing test file
2. Run `pytest --tb=short` to establish the passing test baseline
3. Identify logic bugs by analyzing control flow, edge cases, and boundary conditions
4. Write new pytest tests targeting discovered bugs (must fail on current code)
5. Produce minimal patches for each bug (verify tests now pass)
6. Run `pylint` on the module, collect violations grouped by rule
7. Fix violations (e.g., add type hints, rename variables to snake_case) and re-run pytest

Output:
```
## Audit Report: utils/parser.py

### Bug Fixing (2 issues found)
- **Off-by-one in `parse_range()`** (line 47): `range(start, end)` should be
  `range(start, end + 1)`. Patch: single-line change. Tests: 3 existing pass,
  1 new test added.
- **Unhandled None in `normalize()`** (line 82): crashes on `None` input.
  Patch: early return guard. Tests: 1 new test added.

### Test Generation (4 new tests)
- `test_parse_range_inclusive_end` — fails on buggy code, passes on fix
- `test_normalize_none_input` — fails on buggy code, passes on fix
- `test_parse_range_empty` — covers edge case, currently passes
- `test_normalize_unicode` — covers edge case, currently passes

### Style Fixing (11 violations → 0)
- C0114: Missing module docstring (1) — added
- C0103: Variable name "x" doesn't conform to snake_case (3) — renamed
- W0612: Unused variable "tmp" (1) — removed
- C0301: Line too long >120 chars (6) — reformatted
- All 47 existing tests still pass after style fixes.
```

**Example 2: Java test generation for a known bug**

User: "Write tests that would catch this null pointer exception in the UserService class"

Approach:
1. Read `UserService.java` and the stack trace
2. Identify the null dereference path: `getUser()` returns null when user not found, caller does not check
3. Examine existing test conventions (JUnit 5, Mockito, test file naming)
4. Write tests that trigger the null path
5. Verify: tests fail on current code, pass after adding the null check

Output:
```java
// src/test/java/com/app/service/UserServiceTest.java
@Test
void getUserProfile_whenUserNotFound_shouldReturnEmpty() {
    when(userRepository.findById(999L)).thenReturn(Optional.empty());

    // This test FAILS on buggy code (NullPointerException)
    // and PASSES on fixed code (returns Optional.empty())
    Optional<UserProfile> result = userService.getUserProfile(999L);

    assertThat(result).isEmpty();
}

@Test
void getUserProfile_whenUserExists_shouldReturnProfile() {
    User user = new User(1L, "alice");
    when(userRepository.findById(1L)).thenReturn(Optional.of(user));

    Optional<UserProfile> result = userService.getUserProfile(1L);

    assertThat(result).isPresent();
    assertThat(result.get().getName()).isEqualTo("alice");
}
```

**Example 3: Responding to code review feedback on a C++ patch**

User: "The reviewer said my fix for the memory leak introduces a use-after-free. Help me fix the patch."

Approach:
1. Read the original buggy code, the proposed patch, and the reviewer comment
2. Identify the use-after-free: patch calls `delete ptr` then later accesses `ptr->field`
3. Restructure the fix: extract the needed value before deletion, or use smart pointers
4. Run Clang-Tidy to check for additional memory safety issues
5. Run the CMake test suite to verify both the original leak is fixed and no new failures

Output:
```diff
- delete resource;
- logger.log(resource->name);  // use-after-free!
+ std::string name = resource->name;  // capture before delete
+ delete resource;
+ resource = nullptr;
+ logger.log(name);
```
Clang-Tidy: 0 warnings. Test suite: 38/38 pass.

## Best Practices

- **Do:** Evaluate all four dimensions independently before combining changes. A style fix that accidentally changes semantics is worse than the style violation.
- **Do:** Use the dual-criterion for test generation — a test that passes on both buggy and fixed code is worthless. Always verify the test fails on the defective version.
- **Do:** Match existing project conventions for generated tests (naming, directory structure, assertion library, fixture patterns). Foreign-looking tests get rejected in review.
- **Do:** Run the full test suite after every category of change, not just at the end. Catch regressions immediately.
- **Avoid:** Fixing style violations and bugs in the same commit/patch. Keep concerns separated so reviewers can evaluate each independently.
- **Avoid:** Generating tests that only exercise happy paths. OmniCode's evaluation specifically requires tests that detect defects — target error paths, boundary conditions, and null/empty inputs.
- **Avoid:** Assuming one language's patterns apply to others. Python agents that perform well on bug fixing often fail on Java/C++ due to build system complexity (Maven dependency resolution, CMake configuration).

## Error Handling

- **Test suite won't run:** Check that the build system is properly configured. For Java, verify Maven/Gradle wrapper is present and dependencies resolve. For C++, ensure CMake can find required libraries. Fall back to compiling individual files if the full build is broken.
- **Linter produces false positives:** Cross-reference linter output with the project's existing suppression rules (`.pylintrc`, `checkstyle-suppressions.xml`, `.clang-tidy`). Only fix genuine violations.
- **Generated tests are flaky:** Avoid time-dependent assertions, filesystem-dependent paths, and network calls. Use mocking for external dependencies. Run the test 3 times to confirm determinism.
- **Patch passes tests but breaks unreachable code:** Run the linter in addition to the test suite. Static analysis catches issues (unused imports, dead code) that tests miss.
- **Review fix introduces new issues:** Apply the reviewer's feedback incrementally. After each comment is addressed, re-run the full test suite before proceeding to the next.

## Limitations

- **Language coverage:** The OmniCode framework covers Python, Java, and C++. For other languages (Go, Rust, TypeScript), the four-dimensional evaluation approach still applies conceptually, but specific linter and test framework mappings must be adapted.
- **Test generation quality ceiling:** Automatically generated tests tend toward structural coverage (line/branch) rather than semantic correctness. They may miss logic errors that require domain knowledge to test.
- **Style fixing scope:** Linter rules vary by project. A fix that satisfies Pylint defaults may violate a project's custom `.pylintrc`. Always use the project's own linter configuration.
- **Synthetic task representativeness:** While synthetically generated tasks avoid data leakage, they may not capture the full complexity of emergent bugs in production systems (concurrency issues, distributed system failures).
- **Build environment complexity:** Real repositories often have non-trivial setup requirements (specific JDK versions, system libraries, environment variables). Containerized evaluation (Docker/Apptainer) is strongly recommended but adds infrastructure overhead.

## Reference

**Paper:** [OmniCode: A Benchmark for Evaluating Software Engineering Agents](https://arxiv.org/abs/2602.02262v2) — Sonwane et al., 2026. Look for: the four-category task taxonomy (Section 3), the synthetic task generation pipeline (Section 4), per-category/per-language evaluation results showing agent blind spots (Section 5), and the dual-criterion test evaluation methodology.

**Code & Data:** [github.com/seal-research/OmniCode](https://github.com/seal-research/OmniCode) — Contains instance files, evaluation scripts (`omnicode.py`), SWE-Agent baselines, and the synthetic bad-patch generation pipeline.