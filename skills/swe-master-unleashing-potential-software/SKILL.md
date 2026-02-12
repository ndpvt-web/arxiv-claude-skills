---
name: "swe-master-unleashing-potential-software"
description: "Systematic software engineering agent workflow inspired by SWE-Master's post-training pipeline. Applies structured exploration, LSP-augmented navigation, multi-turn reasoning with budget awareness, iterative test-driven patching, and self-verification to solve real GitHub issues and complex bugs. Trigger phrases: 'fix this GitHub issue', 'debug this failing test', 'resolve this bug with tests', 'patch this repository issue', 'diagnose and fix this software defect', 'systematic bug resolution'."
---

# SWE-Master: Systematic Software Engineering Agent Workflow

This skill equips Claude with a disciplined, multi-phase workflow for resolving real-world software engineering tasks — bugs, failing tests, and GitHub issues — modeled on the SWE-Master framework (arXiv:2602.03411). The core insight is that effective SWE agents follow a structured trajectory of **explore → localize → hypothesize → patch → validate → verify**, with explicit budget tracking, LSP-guided code navigation, and iterative revision driven by actual test execution feedback. Instead of jumping to a fix, the agent systematically narrows the search space, writes reproducing tests, and only submits patches confirmed by execution.

## When to Use

- When the user provides a GitHub issue (or description of one) and asks you to produce a fix
- When a test suite is failing and the user needs the root cause diagnosed and patched
- When the user says "fix this bug" in a repository with a reproducible failure
- When tackling multi-file defects that require understanding call chains and dependencies
- When the user wants a patch validated against both fail-to-pass and pass-to-pass tests
- When resolving issues in large, unfamiliar codebases where naive grep is insufficient

## Key Technique

SWE-Master's core contribution is demonstrating that a structured agent trajectory — not just a powerful model — is what makes SWE tasks solvable. The pipeline decomposes issue resolution into distinct phases: **repository exploration** (understanding project structure and conventions), **fault localization** (narrowing to the exact functions and lines involved), **patch generation** (minimal, targeted edits), and **execution-based verification** (running real tests in the environment). Each phase feeds the next; the agent never skips ahead to editing without first building a mental model of the relevant code paths.

Two techniques from the paper are directly actionable. First, **LSP-augmented navigation**: instead of relying solely on text search, the agent uses semantic tools — go-to-definition, find-references, call-hierarchy analysis — to trace how code flows through a repository. This dramatically reduces wasted exploration turns and catches non-obvious dependencies (e.g., a method override three inheritance levels deep). Second, **budget-aware reasoning**: the agent explicitly tracks how many interaction steps remain and adjusts strategy accordingly — exploring broadly early on and converging to targeted edits as the budget shrinks.

The verification phase uses **test-driven confirmation**: the agent writes or identifies a test that reproduces the bug *before* patching, then confirms the patch makes that test pass while keeping existing tests green. When multiple candidate patches exist, the agent can generate parallel attempts and select the one with the best test outcome — a form of test-time scaling that the paper shows boosts resolve rates from 61.4% to 70.8%.

## Step-by-Step Workflow

1. **Parse the issue into structured requirements.** Extract: (a) the expected behavior, (b) the observed (buggy) behavior, (c) any reproduction steps, (d) affected files or modules if mentioned, and (e) the relevant test commands. Write these down explicitly before doing anything else.

2. **Explore repository structure.** Use `ls`, directory listing, and file globbing to understand the project layout — source directories, test directories, configuration files, and build system. Identify the language, framework, and testing tool (pytest, jest, cargo test, etc.).

3. **Localize the fault using semantic navigation.** Start from the symptom (error message, failing test, or user-reported behavior) and trace inward. Use grep/ripgrep to find the error string or relevant function name, then follow the call chain: read the function, find its callers (find-references pattern), read those callers, and repeat until you reach the root cause. Prefer reading definitions over guessing. For each file you examine, note the file path and line numbers.

4. **Build a fault hypothesis.** Before editing anything, write a clear 1-3 sentence explanation of *why* the bug occurs. Include: which function is wrong, what it does incorrectly, and what the correct behavior should be. If you are uncertain, identify what additional information would confirm or refute the hypothesis and gather it.

5. **Write or identify a reproducing test.** If the issue includes a reproduction script, verify it fails. If not, write a minimal test case that demonstrates the buggy behavior. Run it and confirm it fails with the expected error. This is your fail-to-pass (F2P) test.

6. **Generate a minimal patch.** Edit only the code necessary to fix the root cause. Prefer the smallest change that addresses the issue — do not refactor surrounding code, add unrelated improvements, or change formatting. If the fix requires changes in multiple files, make each edit deliberately and explain its purpose.

7. **Validate the patch against the reproducing test.** Run the F2P test and confirm it now passes. If it still fails, revise the hypothesis (step 4) and iterate. Do not proceed until the reproducing test passes.

8. **Run the full relevant test suite (pass-to-pass check).** Execute the broader test suite for the affected module to ensure the patch does not introduce regressions. If any previously passing tests now fail, diagnose whether the patch caused the regression or whether the failure is pre-existing/flaky.

9. **Review the final diff for correctness and minimality.** Read through every changed line. Verify: no debug prints left behind, no unintended whitespace changes, no accidentally deleted code, and the patch is consistent with the project's coding conventions.

10. **Report the resolution.** Summarize: what the root cause was, what the fix does, which tests confirm it, and any edge cases or follow-up items the user should be aware of.

## Concrete Examples

**Example 1: Fixing a Django ORM query bug**

```
User: "Fix this issue — QuerySet.annotate() with a Subquery containing OuterRef
raises FieldError when used with .values(). See django/django#34567."

Approach:
1. Parse issue: annotate() + Subquery + OuterRef + values() triggers FieldError.
   Expected: query executes normally. Observed: FieldError on column resolution.

2. Explore: ls django/db/models/ — find query.py, sql/query.py, expressions.py,
   subqueries.py. Test dir: tests/annotations/.

3. Localize: grep for "FieldError" in django/db/models/sql/query.py. Find that
   resolve_expression() in Subquery fails to propagate outer references when
   the parent queryset uses .values(). Trace: values() calls set_values() →
   which modifies selected columns → resolve_expression() can't find the
   OuterRef target because it looks only in the narrowed column set.

4. Hypothesis: set_values() strips columns from the outer query's annotation
   mask before Subquery.resolve_expression() runs, so OuterRef resolution
   fails because the referenced field is no longer visible.

5. Reproducing test: Write test in tests/annotations/tests.py:
   qs = Book.objects.values('title').annotate(
       latest_rating=Subquery(
           Rating.objects.filter(book=OuterRef('pk')).order_by('-created')
           .values('score')[:1]
       )
   )
   list(qs)  # Should not raise FieldError

6. Patch: In django/db/models/sql/query.py, modify set_values() to preserve
   fields referenced by OuterRef in annotations before narrowing the mask.
   ~5 line change.

7. Run reproducing test → passes.
8. Run tests/annotations/ full suite → all green.
9. Diff review: only query.py changed, minimal edit.

Output: Root cause was premature column narrowing in set_values() hiding
OuterRef targets from Subquery resolution. Fix preserves outer-referenced
fields in the annotation mask. Confirmed by new test and full annotation
suite passing.
```

**Example 2: Debugging a race condition in a Python async library**

```
User: "Our aiohttp middleware sometimes returns a 500 with 'RuntimeError:
Response already started'. Can you find and fix it?"

Approach:
1. Parse: intermittent 500, RuntimeError on response already started.
   Likely a double-write or missing await causing concurrent response writes.

2. Explore: ls src/middleware/ — find auth.py, logging.py, error_handler.py.
   Test dir: tests/middleware/.

3. Localize: grep "Response already started" in aiohttp source to understand
   when it's raised (in StreamResponse.prepare()). Then search the middleware
   chain for places where response.write() or response.prepare() is called.
   Find that error_handler.py calls response.prepare() in the except block
   but doesn't check if the response was already prepared by a downstream
   handler.

4. Hypothesis: When a downstream handler starts the response (calls prepare())
   and then raises an exception, the error_handler middleware tries to
   prepare() a new error response on the same StreamResponse object, which
   is already started.

5. Reproducing test: Write test that has a handler which calls
   response.prepare(), writes partial data, then raises ValueError.
   Assert the middleware returns 500 without RuntimeError.

6. Patch: In error_handler.py, add a guard:
   if not response.prepared:
       await response.prepare(request)
       await response.write(error_body)
   else:
       # Response already started; log and let aiohttp close it
       logger.warning("Cannot send error response: already started")
   ~6 line change.

7. Reproducing test → passes (no RuntimeError).
8. Full middleware test suite → all green.

Output: The error_handler middleware didn't check response.prepared before
attempting to send an error response. Added a guard that skips error writing
when the response stream is already open. Test confirms no RuntimeError.
```

**Example 3: Multi-file type error in a TypeScript project**

```
User: "After upgrading to TypeScript 5.3, our build fails with 'TS2345:
Argument of type X is not assignable to parameter of type Y' in
src/api/client.ts. Fix the type errors without changing runtime behavior."

Approach:
1. Parse: TS2345 after upgrade, in src/api/client.ts. Need type-only fix,
   no runtime changes.

2. Explore: ls src/api/ — find client.ts, types.ts, endpoints/. Check
   tsconfig.json for strictness settings. Run tsc --noEmit to get full
   error list.

3. Localize: Read the exact error lines in client.ts. The function
   fetchEndpoint<T>() returns Promise<T>, but the caller passes the result
   to a function expecting Promise<ApiResponse<T>>. In TS 5.3, stricter
   variance checking catches this mismatch that was previously allowed.

4. Hypothesis: fetchEndpoint's return type generic T was implicitly widened
   in TS 5.2 but TS 5.3 enforces the declared variance. The fix is to
   make fetchEndpoint return Promise<ApiResponse<T>> explicitly, matching
   what it actually returns at runtime.

5. Verification: read fetchEndpoint implementation — it does return
   ApiResponse<T> at runtime, confirming the type annotation is wrong,
   not the runtime code.

6. Patch: In client.ts, change return type from Promise<T> to
   Promise<ApiResponse<T>>. Update 2 call sites in endpoints/ that
   destructure the result (they already expect ApiResponse shape, just
   need type annotations updated).

7. Run tsc --noEmit → zero errors.
8. Run npm test → all tests pass, no runtime changes.

Output: TS 5.3's stricter generic variance exposed an incorrect return type
on fetchEndpoint(). Fixed the type annotation to match the actual runtime
return type (ApiResponse<T>). No runtime behavior changed. Build and tests
pass clean.
```

## Best Practices

- **Do:** Always read the code before editing it. Trace the call chain from symptom to root cause — never guess at the fix based on the error message alone.
- **Do:** Write or identify a failing test *before* writing the patch. This prevents false confidence from patches that don't actually address the issue.
- **Do:** Track your remaining budget explicitly. If you've spent 70% of your turns exploring without a clear hypothesis, switch strategy: narrow your focus to the most probable location and test a hypothesis directly.
- **Do:** Make the smallest possible patch. Each additional changed line is an additional risk of regression. If you can fix the bug by changing one condition, don't also refactor the function.
- **Avoid:** Jumping to `str_replace` edits before understanding the surrounding code. Context matters — a fix that looks correct in isolation may break an invariant relied upon elsewhere.
- **Avoid:** Running the entire project test suite on every iteration. Run the specific relevant test file first; only run the broader suite once you believe the fix is complete.
- **Avoid:** Ignoring test failures that seem "unrelated." Verify they are genuinely pre-existing by checking if they fail on the unmodified code before dismissing them.

## Error Handling

- **Hypothesis is wrong (test still fails after patch):** Do not iterate on the same patch blindly. Re-examine the fault localization. Read the test output carefully — the failure mode may have changed, pointing to a different root cause.
- **Patch introduces regressions:** Read the failing test to understand what behavior it asserts. Trace how your change affects that code path. Often the regression reveals an assumption you missed — incorporate that into a revised patch.
- **Cannot reproduce the bug:** Ask the user for exact reproduction steps, environment details, and dependency versions. If the bug is environment-specific, focus on understanding the environmental difference rather than guessing at code changes.
- **Repository is too large to explore exhaustively:** Use semantic search strategies — start from the error message, trace to the raising function, follow its callers. Avoid breadth-first exploration of unrelated modules. LSP-style navigation (go-to-definition, find-references) is far more efficient than grep for understanding call chains.
- **Test suite is slow or flaky:** Isolate the specific test file or test case relevant to the bug. Run only that test during iteration. Flag flaky tests to the user but do not attempt to fix them unless asked.

## Limitations

- This workflow assumes the project has a runnable test suite. If there are no tests and no way to execute the code, verification falls back to code review only, which is less reliable.
- For bugs that require deep domain knowledge (cryptographic protocols, database internals, kernel behavior), the explore-localize-patch loop still works, but hypothesis quality depends on the agent's domain understanding.
- Race conditions and concurrency bugs may not reproduce deterministically. The workflow helps structure the investigation, but the reproducing test may need to use stress-testing or mocking to trigger the bug reliably.
- This approach is optimized for "fix this specific bug" tasks. For open-ended feature requests or large-scale refactors, a planning-first workflow is more appropriate.
- The parallel test-time scaling strategy (generating multiple candidate patches and selecting the best) is a conceptual guide — in practice, Claude generates one trajectory per conversation, so the "multiple attempts" pattern requires the user to request retries or the agent to try alternative approaches sequentially.

## Reference

- **Paper:** [SWE-Master: Unleashing the Potential of Software Engineering Agents via Post-Training](https://arxiv.org/abs/2602.03411v1) — Song et al., 2026. Look for: the trajectory format (Thought + Action pairs), the LSP tool integration for semantic navigation, the budget-aware prompting strategy, the GRPO reward shaping (binary outcome + forced-submission penalty), and the SWE-World verifier for selecting among candidate patches.
- **Code:** [github.com/RUCAIBox/SWE-Master](https://github.com/RUCAIBox/SWE-Master)