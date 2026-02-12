---
name: "tam-eval-evaluating-llms-for"
description: |
  Systematic test suite maintenance using the TAM-Eval framework's three-scenario approach:
  creating new tests, repairing broken tests, and updating tests after code changes.
  Evaluates test quality via pass rate, code coverage delta, and mutation score delta
  rather than relying on reference test files. Operates at the test-file level with
  full repository context.

  Trigger phrases:
  - "maintain my test suite"
  - "fix these broken tests"
  - "update tests after refactoring"
  - "generate tests with mutation coverage"
  - "evaluate my test quality"
  - "improve test effectiveness"
---

# TAM-Eval: Systematic Test Suite Maintenance

This skill enables Claude to maintain unit test suites using the structured methodology from TAM-Eval (Test Automated Maintenance Evaluation). Instead of generating tests in isolation or fixing them ad hoc, Claude follows a three-scenario framework — creation, repair, and updating — that mirrors real-world test maintenance workflows. Each scenario operates at the test-file level (not individual functions), uses full repository context to resolve dependencies, and measures quality through a reference-free protocol: pass rate, coverage delta, and mutation score delta.

## When to Use

- When the user asks to write tests for code that has no tests or incomplete coverage
- When existing tests are failing after dependency upgrades, API changes, or environment shifts
- When production code has been refactored and existing tests need to be adapted to match
- When the user wants to improve test effectiveness beyond simple line coverage (mutation testing)
- When evaluating whether a test suite is actually catching bugs, not just executing code paths
- When the user needs to recover deleted or corrupted test files from repository history
- When adding test cases to an existing test file to cover edge cases or uncovered branches

## Key Technique

TAM-Eval identifies three distinct maintenance scenarios that cover the full test lifecycle. **Creation** generates test files from scratch, extends partial test suites, or recovers lost tests. **Repair** fixes broken tests caused by syntax errors, execution failures, coverage regressions, or weakened mutation detection. **Updating** adapts existing tests to reflect changes in the production code, typically after refactoring or feature modification. Each scenario has specific inputs (source code, existing tests, error output, diffs) and a single output: the complete, corrected test file.

The critical insight is the **reference-free evaluation protocol**. Rather than comparing generated tests against a "gold standard" test file, TAM-Eval measures three orthogonal quality dimensions: (1) **Pass rate** — do the tests actually run without errors against the current implementation? (2) **Coverage delta (ΔTestCov)** — how much additional line coverage do the tests provide compared to the baseline? (3) **Mutation score delta (ΔMutCov)** — do the tests detect intentionally injected code mutations (operator swaps, condition flips, literal changes)? A test suite with high coverage but low mutation score is exercising code without truly verifying behavior.

The framework also uses a **multi-attempt protocol with failure feedback**. When tests fail on the first attempt, the exact error output (syntax errors, stack traces, compiler messages) is fed back to generate a corrected version, up to 3 attempts. This iterative loop is where most quality gains occur — research shows that execution errors account for over 60% of initial failures, and structured feedback resolves many of them.

## Step-by-Step Workflow

1. **Classify the maintenance scenario.** Determine whether this is creation (no tests or insufficient tests exist), repair (tests exist but fail), or updating (tests exist but production code has changed). This determines what context to gather and what the prompt structure should be.

2. **Assemble repository context.** Read the focal source file (the production code under test) and the existing test file (if any). Identify imports, class hierarchies, and external dependencies by tracing the module graph. For repair tasks, capture the full error output. For update tasks, generate or obtain the diff between old and new production code.

3. **Establish the quality baseline.** Before making changes, run the existing test suite and record: number of passing/failing tests, line coverage percentage, and if possible, mutation score. These baselines are essential for measuring improvement via deltas rather than absolute numbers.

4. **Generate the test file using scenario-specific structure.** For creation: provide the source file and instruct generation of a complete test file covering public interfaces, edge cases, and error paths. For repair: provide the source file, the broken test file, and the exact error output. For updating: provide the source file, the outdated test file, and the code diff. Always request a complete test file, not fragments.

5. **Validate pass rate immediately.** Run the generated tests against the production code. If any tests fail, extract the exact error messages (assertion failures, import errors, type mismatches) and feed them back as structured failure context for a second attempt. Repeat up to 3 iterations.

6. **Measure coverage delta.** Run the passing tests with coverage instrumentation (coverage.py for Python, JaCoCo for Java, `go test -cover` for Go). Compute ΔTestCov = new coverage - baseline coverage. If the delta is negative or zero, the generated tests are not adding value — identify uncovered branches and add targeted test cases.

7. **Evaluate mutation score when thoroughness matters.** Run mutation testing (mutmut or mutpy for Python, PIT for Java, go-mutesting for Go) to check whether tests detect injected faults. Compute ΔMutCov = new mutation score - baseline. A test that covers a line but doesn't detect mutations on that line is a weak test — strengthen assertions or add boundary-value cases.

8. **Refine weak tests.** For any surviving mutants (mutations not caught by tests), analyze what the mutant changed (e.g., `>` became `>=`, a return value changed from `True` to `False`) and add an assertion that specifically distinguishes correct from mutated behavior.

9. **Verify dependency isolation.** Ensure tests don't depend on external services, file system state, or execution order. Mock or stub external dependencies. Run tests in a clean environment to confirm they pass independently.

10. **Deliver the complete test file with quality metrics.** Provide the final test file alongside a summary: pass rate (should be 100%), ΔTestCov, and ΔMutCov. Explain any tests that were intentionally excluded and why.

## Concrete Examples

**Example 1: Creating tests from scratch (Python)**

User: "Write tests for `src/utils/tokenizer.py` — it has no tests."

Approach:
1. Read `src/utils/tokenizer.py` and its imports to understand the public API
2. Identify the scenario: creation from scratch
3. Examine existing test patterns in the repo (pytest vs unittest, fixture style, naming conventions)
4. Generate `tests/test_tokenizer.py` covering: normal inputs, empty strings, unicode, malformed input, boundary lengths
5. Run tests — fix any import path or fixture issues on failure feedback
6. Measure coverage with `pytest --cov=src.utils.tokenizer`

Output:
```python
# tests/test_tokenizer.py
import pytest
from src.utils.tokenizer import Tokenizer

class TestTokenizer:
    def test_tokenize_simple_sentence(self):
        t = Tokenizer()
        assert t.tokenize("hello world") == ["hello", "world"]

    def test_tokenize_empty_string(self):
        t = Tokenizer()
        assert t.tokenize("") == []

    def test_tokenize_preserves_punctuation(self):
        t = Tokenizer()
        tokens = t.tokenize("hello, world!")
        assert "hello," in tokens or "hello" in tokens  # depends on spec

    def test_tokenize_unicode(self):
        t = Tokenizer()
        result = t.tokenize("café résumé")
        assert len(result) == 2

    def test_tokenize_max_length_boundary(self):
        t = Tokenizer(max_length=3)
        result = t.tokenize("a b c d e")
        assert len(result) == 3

# Quality metrics:
# Pass rate: 5/5 (100%)
# ΔTestCov: +72.3% (from 0% baseline)
# ΔMutCov: +58.1% (15 of 22 mutants killed)
```

**Example 2: Repairing broken tests after a dependency upgrade (Java)**

User: "Our tests in `UserServiceTest.java` are failing after upgrading to Spring Boot 3.2. Fix them."

Approach:
1. Read `UserServiceTest.java` and `UserService.java`
2. Identify the scenario: repair (tests exist but fail)
3. Run the tests and capture exact error output — likely `jakarta.servlet` vs `javax.servlet` namespace changes
4. Feed the source file, broken test file, and error messages into repair context
5. Fix namespace imports, deprecated API calls, and changed constructor signatures
6. Re-run and iterate if additional failures surface

Output:
```java
// Key repairs applied to UserServiceTest.java:

// 1. Namespace migration (javax -> jakarta)
// BEFORE: import javax.servlet.http.HttpServletRequest;
// AFTER:  import jakarta.servlet.http.HttpServletRequest;

// 2. Deprecated MockMvc setup replaced
// BEFORE: MockMvcBuilders.standaloneSetup(controller).build();
// AFTER:  MockMvcBuilders.standaloneSetup(controller)
//             .setCustomArgumentResolvers(new PageableHandlerMethodArgumentResolver())
//             .build();

// 3. Changed assertion API
// BEFORE: assertEquals(expected, actual);  // argument order warning
// AFTER:  assertThat(actual).isEqualTo(expected);  // AssertJ style

// Quality metrics:
// Pass rate: 12/12 (100%, was 0/12)
// ΔTestCov: +0.0% (coverage unchanged — repair, not enhancement)
// ΔMutCov: +2.1% (stronger assertions catch more mutants)
```

**Example 3: Updating tests after a refactor (Go)**

User: "I refactored `pkg/cache/store.go` to use generics instead of `interface{}`. Update the tests."

Approach:
1. Read `pkg/cache/store.go` (new) and `pkg/cache/store_test.go` (outdated)
2. Identify the scenario: updating (production code changed, tests must adapt)
3. Get the diff: `git diff HEAD~1 -- pkg/cache/store.go`
4. Key changes: `Get(key string) interface{}` became `Get[T any](key string) T`, `Put(key string, value interface{})` became `Put[T any](key string, value T)`
5. Update all test call sites to use type parameters, remove type assertions from test expectations
6. Run, fix compilation errors from the type parameter syntax, measure coverage

Output:
```go
// store_test.go — updated for generics

func TestStore_PutAndGet(t *testing.T) {
    // BEFORE: s := NewStore()
    //         s.Put("key", "value")
    //         got := s.Get("key").(string)  // type assertion

    // AFTER:
    s := NewStore[string]()
    s.Put("key", "value")
    got := s.Get("key")  // returns string directly, no assertion needed
    if got != "value" {
        t.Errorf("Get() = %v, want %v", got, "value")
    }
}

func TestStore_GetMissing(t *testing.T) {
    s := NewStore[int]()
    got := s.Get("missing")
    if got != 0 {  // zero value for int, not nil
        t.Errorf("Get() = %v, want zero value", got)
    }
}

// Quality metrics:
// Pass rate: 8/8 (100%, was 0/8 due to compilation errors)
// ΔTestCov: -1.2% (minor — some removed type-assertion branches no longer exist)
// ΔMutCov: +3.4% (generics eliminated some weak interface{} comparisons)
```

## Best Practices

- **Do:** Always run existing tests first to establish a baseline before generating or modifying anything. Without a baseline, deltas are meaningless.
- **Do:** Provide the full error output (stack traces, compiler messages) when repairing tests — partial error context leads to partial fixes that fail on the next run.
- **Do:** Match the repository's existing test conventions (test runner, assertion library, file naming, fixture patterns) rather than imposing a different style.
- **Do:** Target mutation score alongside line coverage. A test that executes a branch but doesn't assert on its output provides false confidence.
- **Avoid:** Generating tests for internal/private implementation details — these break on every refactor and provide little maintenance value.
- **Avoid:** Adding more than the necessary number of test cases per iteration. The TAM-Eval research shows diminishing returns and increased execution errors beyond focused, targeted additions.
- **Avoid:** Skipping the iterative feedback loop. The multi-attempt protocol with failure feedback resolves the majority of initial execution errors — single-shot generation has significantly lower pass rates.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| Tests fail with import errors | Missing dependencies or wrong module paths | Read `pyproject.toml`/`pom.xml`/`go.mod` to verify dependency availability; adjust import paths |
| Tests pass but coverage drops | Generated tests replace existing ones instead of extending | Always append to or modify existing test files rather than overwriting |
| Mutation testing times out | Too many mutants generated for large source files | Scope mutation testing to the specific functions under test using tool-specific filters (e.g., PIT `targetClasses`, mutmut `--paths-to-mutate`) |
| Tests pass locally but fail in CI | Environment differences (OS, Python version, env vars) | Add environment checks at test setup; mock system-dependent operations |
| Flaky tests after generation | Non-deterministic behavior (time, randomness, concurrency) | Freeze time with `freezegun`/`Clock`, seed random generators, avoid shared mutable state |
| Compilation errors in generated code | LLM hallucinates API signatures or uses wrong syntax | Feed exact compiler errors back in the next attempt; include relevant type signatures in context |

## Limitations

- **Mutation testing is slow.** For large codebases, running a full mutation suite can take minutes to hours. Use it selectively on critical paths rather than across every test file.
- **Language-specific tooling required.** Each language needs its own coverage and mutation tools properly configured (coverage.py + mutmut for Python, JaCoCo + PIT for Java, go cover + go-mutesting for Go). If these aren't set up in the project, the evaluation protocol is incomplete.
- **The multi-attempt loop requires execution.** The repair and feedback cycle depends on actually running the tests. In environments where execution isn't possible (e.g., reviewing PRs without CI), you can still apply the structural approach but cannot verify pass rate or measure deltas.
- **Repository context window constraints.** Large source files with deep dependency trees may exceed practical context limits. Prioritize the focal file and its direct imports; include transitive dependencies only when errors indicate they're needed.
- **Test maintenance is inherently project-specific.** Mocking strategies, fixture patterns, and assertion styles vary widely. The TAM-Eval framework provides the evaluation structure, but the generated tests must conform to each project's conventions — there is no universal template.

## Reference

- **Paper:** [TAM-Eval: Evaluating LLMs for Automated Unit Test Maintenance](https://arxiv.org/abs/2601.18241v1) — See Section 3 for the three-scenario task taxonomy, Section 4 for the reference-free evaluation protocol (pass rate, ΔTestCov, ΔMutCov), and Section 5 for empirical results showing that iterative feedback significantly outperforms single-shot generation.
- **Code:** [github.com/trndcenter/TAM-Eval](https://github.com/trndcenter/TAM-Eval) — Benchmark scenarios in `benchmark/v1/` (TOML format), prompt templates in `generate/prompts/`, evaluation harness in `teval/`.