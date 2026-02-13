---
name: "testexplora-benchmarking-proactive-bug"
description: >
  Proactive bug discovery through documentation-driven test generation. Generates tests
  that find latent bugs by comparing code implementations against documentation-derived
  intent, rather than treating existing code as ground truth. Use when:
  "find bugs in this codebase", "write tests that catch real defects",
  "proactive testing against the docs", "generate bug-finding tests",
  "test this repo for hidden bugs", "audit code against its specification"
---

# Proactive Bug Discovery via Documentation-Driven Test Generation

This skill enables Claude to act as a **proactive tester** that discovers latent bugs in codebases by generating tests derived from documentation and specifications rather than from existing code behavior. Based on the TestExplora methodology (Liu et al., 2026), the core insight is that treating current code as ground truth creates a "compliance trap" -- tests that merely verify what code does, not what it should do. Instead, this approach uses documentation as the oracle: reading specs, docstrings, and READMEs to understand intended behavior, then writing tests that expose gaps between intent and implementation.

## When to Use

- When a user asks to "find bugs" or "audit" a codebase without a specific bug report in hand
- When asked to write tests that go beyond regression -- tests meant to catch defects, not just lock in behavior
- When documentation exists (READMEs, docstrings, API docs, type annotations) and the user wants to validate code against it
- When reviewing a pull request and wanting to proactively check for defects in the changed code
- When asked to generate a test suite for a module the user suspects is fragile but has no failing test yet
- When performing pre-release quality assurance across a repository

## Key Technique: Documentation-as-Oracle with Agentic Exploration

Traditional test generation treats the current codebase as the source of truth -- if the code returns 42, the test asserts 42. This catches future regressions but never surfaces existing bugs. TestExplora inverts this: the **documentation** is the oracle. The model reads docstrings, README files, API specifications, type hints, and inline comments to build a model of *intended* behavior. It then writes tests that assert the documented contract. When a test fails, it signals a bug -- the implementation diverges from its specification.

The second critical insight is **cross-module navigation via invocation graphs**. Most bugs hide at module boundaries where assumptions from one module collide with the implementation of another. Effective proactive testing requires tracing call paths from public API surfaces down through internal dependencies, identifying where documented contracts could be violated. This means reading not just the target function, but its callees and callers.

The third insight is that **agentic exploration outperforms static context dumping**. In the TestExplora evaluation, agents that iteratively explored repositories (viewing code, running partial tests, adapting based on findings) achieved 29.7% bug detection at 5 attempts versus 16.06% for single-shot approaches. The practical takeaway: explore the repo incrementally, form hypotheses, and test them -- don't try to absorb all context at once.

## Step-by-Step Workflow

1. **Gather documentation sources.** Scan the repository for README files, docstrings, type annotations, API docs, changelogs, and inline specification comments. Prioritize official docs and docstrings on public functions. These form the "oracle" against which you test.

2. **Identify target modules and their entry points.** Focus on modules the user specifies, or -- if none specified -- prioritize modules with high complexity, recent changes (`git log --since`), or known fragility. List the public functions and classes that serve as test entry points.

3. **Build an invocation graph for each target.** For each entry-point function, trace its call chain: what internal functions does it invoke? What modules does it import? Map at least two levels of dependencies. This reveals cross-module boundaries where specification mismatches are most likely.

4. **Extract documented intent for each function in the graph.** For every function in the invocation chain, extract: (a) what the docstring says it does, (b) parameter constraints from type hints or doc comments, (c) documented return values and side effects, (d) stated error conditions and edge cases. Record any ambiguities or contradictions.

5. **Formulate bug hypotheses.** For each function, ask: "If the documentation says X, what implementation mistakes could violate X?" Focus on: boundary conditions (off-by-one, empty inputs), type coercion edge cases, error handling paths that the docs say should raise exceptions, and cross-module contract mismatches (e.g., Module A documents it returns a sorted list; Module B consumes it assuming sorted order -- what if A doesn't actually sort?).

6. **Generate focused tests from hypotheses.** Write one test per hypothesis. Each test should: (a) set up minimal context, (b) call the function with an input derived from the documented spec, (c) assert the documented behavior -- not what the code currently does. Keep tests small; TestExplora shows that suites exceeding ~96 tests degrade in coherence.

7. **Run the test suite against the current codebase.** Execute the tests. Passing tests confirm the code matches its docs. Failing tests are bug candidates. For each failure, verify it's a genuine specification violation (not a doc error or test setup issue).

8. **Triage failures into true bugs vs. documentation drift.** For each failing test, read the implementation to confirm the behavior diverges from documented intent. If the code is clearly correct and the doc is wrong, flag it as a documentation bug. If the code violates its own stated contract, report a real defect.

9. **Report findings with evidence.** For each confirmed bug, provide: the failing test, the documentation excerpt that defines expected behavior, the actual behavior observed, and the source location of the defect. Suggest a fix or at minimum identify the root cause.

10. **Iterate on low-coverage areas.** Check which parts of the invocation graph your tests didn't reach. Generate additional tests targeting uncovered change-focused lines (lines modified in recent commits), since recent changes are statistically more likely to contain defects.

## Concrete Examples

**Example 1: Proactive testing of a utility library**

```
User: "Find bugs in our string utilities module. Here's the repo."

Approach:
1. Read src/utils/strings.py docstrings and README section on string utilities.
2. Docstring for `truncate(s, max_len)` says: "Truncates string to max_len
   characters, appending '...' if truncated. Returns original string if shorter
   than max_len."
3. Hypothesis: does it handle max_len < 3 correctly? The '...' suffix is 3 chars.
   If max_len=2, the output could exceed max_len or the function might crash.
4. Hypothesis: what about empty string input? Doc says "returns original" --
   does it return "" or does it append "..." to ""?
5. Generate tests:

   def test_truncate_short_max_len():
       # Doc says truncate to max_len with "..." suffix
       # If max_len=2, result must be at most 2 chars
       result = truncate("hello world", 2)
       assert len(result) <= 2

   def test_truncate_empty_string():
       # Doc says returns original if shorter than max_len
       assert truncate("", 10) == ""

   def test_truncate_exact_length():
       # Doc says returns original if shorter than max_len
       # Boundary: what about equal length?
       assert truncate("hello", 5) == "hello"

6. Run tests. If test_truncate_short_max_len fails because the function
   returns "..." (3 chars) when max_len=2, that's a real bug.

Output:
  BUG FOUND: `truncate("hello world", 2)` returns "..." (3 chars),
  violating the max_len=2 contract.
  Location: src/utils/strings.py:45
  Doc says: "Truncates string to max_len characters"
  Actual: returns string longer than max_len when max_len < len("...")
  Suggested fix: handle max_len < 3 by returning s[:max_len] without suffix.
```

**Example 2: Cross-module bug discovery in an API layer**

```
User: "Audit the payment processing module for bugs before we release."

Approach:
1. Read docs/api.md and src/payments/processor.py docstrings.
2. `process_payment(amount, currency)` docstring says: "amount is in the
   smallest currency unit (e.g., cents for USD). Raises ValueError if
   amount <= 0."
3. Trace invocation graph: process_payment -> validate_amount -> convert_currency
   -> charge_provider.
4. Read convert_currency docstring: "Converts between currencies using live
   rates. Returns integer amount in target currency's smallest unit."
5. Hypothesis: convert_currency returns a float from rate multiplication.
   If process_payment expects int (smallest unit) but gets float, downstream
   charge_provider may fail or lose precision.
6. Hypothesis: the ValueError for amount <= 0 -- does validate_amount check
   this before or after currency conversion? If after, a positive amount in
   one currency could convert to 0 in another due to rounding.

   def test_process_payment_rejects_zero():
       with pytest.raises(ValueError):
           process_payment(0, "USD")

   def test_currency_conversion_returns_integer():
       # Doc says "returns integer amount"
       result = convert_currency(100, "USD", "JPY")
       assert isinstance(result, int)

   def test_small_amount_conversion_not_rounded_to_zero():
       # 1 cent USD -> JPY could round to 0
       result = convert_currency(1, "USD", "JPY")
       assert result > 0  # doc says smallest unit, should not be zero

Output:
  BUG FOUND: `convert_currency(1, "USD", "JPY")` returns 0 due to
  int(0.67) truncation. This means process_payment(1, "USD") targeting
  JPY silently charges 0 yen.
  Location: src/payments/converter.py:23 -- `return int(amount * rate)`
  Doc contract violated: "Returns integer amount in target currency's
  smallest unit" -- 0 is not a valid charge amount.
  Fix: use `math.ceil()` or `round()` instead of `int()` truncation.
```

**Example 3: Black-box testing from API documentation only**

```
User: "We have an internal API with OpenAPI docs but I can't share the source.
Test it for bugs based on the API spec."

Approach:
1. Parse the OpenAPI/Swagger spec to extract endpoints, parameters,
   response schemas, and documented error codes.
2. For POST /users: spec says "email must be unique, returns 409 on
   duplicate." Also says "name is required, max 255 chars."
3. Hypotheses from spec:
   - Does it actually return 409 on duplicate email, or 500?
   - What happens with name of exactly 255 chars? 256 chars?
   - Does it validate email format, or accept "not-an-email"?
4. Generate tests:

   def test_duplicate_email_returns_409():
       create_user(email="test@example.com", name="A")
       resp = create_user(email="test@example.com", name="B")
       assert resp.status_code == 409  # not 500

   def test_name_at_max_length():
       resp = create_user(email="long@test.com", name="x" * 255)
       assert resp.status_code == 201

   def test_name_over_max_length():
       resp = create_user(email="over@test.com", name="x" * 256)
       assert resp.status_code == 400  # spec says max 255

5. Run against the API. Report any status code mismatches.

Output:
  BUG FOUND: POST /users with 256-char name returns 201 (created)
  instead of 400. Server does not enforce the documented max length.
  Spec: "name: string, maxLength: 255"
  Actual: accepts 256+ character names without validation.
```

## Best Practices

- **Do:** Always read the documentation FIRST, before reading the implementation. This prevents anchoring your expectations to what the code does rather than what it should do.
- **Do:** Test boundary conditions explicitly -- off-by-one, empty inputs, maximum values, type boundaries. The TestExplora data shows assertion mismatches on edge cases are the dominant failure mode.
- **Do:** Keep individual tests small and focused on one hypothesis. Suites with fewer, precise tests outperform large exhaustive suites. Aim for quality over quantity.
- **Do:** Trace cross-module call paths. Bugs cluster at module boundaries where one module's assumptions meet another module's implementation.
- **Avoid:** Treating the current implementation as the expected behavior. If you write `assert f(x) == [whatever the code currently returns]`, you've fallen into the compliance trap.
- **Avoid:** Generating tests for trivially correct code (simple getters, pass-through wrappers). Focus effort on functions with documented complex behavior, error handling, or cross-module interactions.
- **Avoid:** Overwhelming the test suite. Beyond ~50-100 focused tests per module, suite-level coherence degrades. Prioritize high-risk hypotheses.

## Error Handling

| Problem | Resolution |
|---|---|
| No documentation exists for the target code | Fall back to type hints, parameter names, and return types as implicit specs. Inform the user that bug detection confidence is lower without explicit docs. |
| Test fails due to test setup error, not a real bug | Verify the failure by reading the implementation. If the function works correctly and the test premise was wrong, discard it. Check mock configurations carefully -- misconfigured mocks are the second most common false positive. |
| Documentation contradicts itself | Flag both the contradiction and any tests affected. Test both interpretations and report which behavior the code actually implements. |
| Cannot determine if failure is a code bug or doc bug | Report it as a "specification-implementation mismatch" and let the user decide. Provide both the doc excerpt and the actual behavior. |
| Cross-module dependencies are too deep to trace | Limit invocation graph depth to 3 levels. Beyond that, treat deeper dependencies as black boxes and test the boundary contracts only. |

## Limitations

- **Documentation quality is the ceiling.** If docs are absent, vague, or wrong, this technique degrades to educated guessing. The oracle is only as good as the specification.
- **Not a replacement for runtime fuzzing.** This approach finds logic bugs (wrong behavior) but won't catch memory corruption, race conditions, or performance regressions that require dynamic analysis.
- **False positive rate is non-trivial.** Approximately 70-84% of generated tests in the TestExplora benchmark pass (no bug found), and some failures are test errors, not real bugs. Human triage remains necessary.
- **Domain-specific code is harder.** Scientific computing and security domains showed much higher difficulty in the benchmark. Specialized domain knowledge may be needed to write meaningful test oracles.
- **Single-language focus.** The TestExplora benchmark covers Python repositories. The methodology generalizes conceptually, but test generation patterns (pytest idioms, mock setup) are language-specific.

## Reference

**TestExplora: Benchmarking LLMs for Proactive Bug Discovery via Repository-Level Test Generation**
Liu et al., 2026. arXiv:2602.10471v1
https://arxiv.org/abs/2602.10471v1

Key takeaway: documentation-as-oracle proactive testing reveals real bugs that compliance-style regression tests miss, and iterative agentic exploration of the repository (tracing call graphs, running incremental tests) substantially outperforms single-shot test generation.