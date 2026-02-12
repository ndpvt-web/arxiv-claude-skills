---
name: "swe-master-unleashing-potential-software"
description: >
  Applies the SWE-Master agentic software engineering methodology to solve complex,
  multi-file bugs and feature requests. Uses structured trajectory planning, LSP-based
  semantic code navigation, context-aware budget management, and iterative verification
  to resolve real-world GitHub issues. Trigger phrases: "fix this GitHub issue",
  "debug this multi-file bug", "resolve this SWE-bench task", "trace this bug across
  the codebase", "find and fix this regression", "navigate this codebase semantically".
---

# SWE-Master: Structured Agentic Software Engineering

This skill equips Claude to tackle complex, real-world software engineering tasks — particularly multi-file bug fixes and feature implementations in large repositories — using the systematic methodology from the SWE-Master framework (arXiv:2602.03411). Instead of ad-hoc grepping and guessing, it applies a disciplined pipeline: semantic code navigation via LSP-style tools, structured exploration trajectories, context-managed long-horizon problem solving, budget-aware turn planning, and iterative test-driven verification. The approach transforms a 6.2% baseline resolve rate into 61.4%+ by replacing lexical search with semantic understanding and unstructured exploration with deliberate trajectory execution.

## When to Use

- When the user provides a GitHub issue (bug report, feature request, or test failure) and asks you to resolve it in a real codebase
- When debugging a non-crashing defect where the error context is ambiguous and grep alone cannot locate the root cause
- When a fix requires tracing definitions, call hierarchies, and references across multiple files and modules
- When the user asks to "fix this like a SWE agent" or wants a structured, agent-style approach to a complex codebase task
- When a bug spans multiple layers (e.g., API handler, service logic, database query) and needs coordinated multi-file changes
- When you need to navigate an unfamiliar large codebase efficiently to understand architecture before making changes

## Key Technique

**Semantic Navigation over Lexical Search.** The core insight of SWE-Master is that traditional agent approaches relying on `grep` and `find` hit a wall on non-crashing, ambiguous bugs. These tools return noisy results, miss semantic relationships, and waste interaction turns scrolling through irrelevant matches. SWE-Master replaces this with LSP-style semantic navigation: go-to-definition, find-references, call-hierarchy analysis, and workspace symbol search. This mirrors how a human developer uses an IDE — jumping to definitions, tracing callers, and understanding type hierarchies — rather than reading files linearly. In practice, this reduces resolution steps by ~37% and input tokens by ~24%.

**Structured Trajectory with Budget Awareness.** Rather than open-ended exploration, SWE-Master operates with explicit turn budgets and a phased approach: reproduce the issue, localize the fault, understand the context, implement the fix, and verify. The agent receives remaining-budget signals at each step, enabling it to switch from exploration to exploitation when turns are scarce. This prevents the common failure mode where an agent exhausts its context exploring tangents without ever attempting a fix.

**Context Management via Summarization.** Long-horizon tasks (50-150 turns) overflow context windows. SWE-Master uses a hybrid strategy: recent interactions are kept verbatim in a sliding window, while older interactions are compressed into natural language summaries. This prevents both context explosion (appending everything) and amnesia (discarding old state), maintaining a coherent working memory throughout the session.

## Step-by-Step Workflow

1. **Parse the issue into a structured problem statement.** Extract the bug description, reproduction steps, expected vs. actual behavior, affected files or modules (if mentioned), and any stack traces or error messages. Formulate a one-sentence hypothesis of the root cause category (logic error, missing validation, incorrect API usage, etc.).

2. **Reproduce or confirm the failure.** If tests are available, run the specific failing test(s) to confirm the issue exists and capture the exact error output. If no test exists, attempt to construct a minimal reproduction from the issue description. Record the precise failure signature (exception type, assertion message, incorrect output value).

3. **Localize the fault using semantic navigation, not grep.** Start from the failure point (test file, error location, or entry point mentioned in the issue). Use go-to-definition to trace the call chain from the failure to the implementation. Use find-references to understand all callers of suspicious functions. Use workspace-symbol search to locate relevant classes or functions by name when the entry point is ambiguous. Prefer definition-tracing over text search — follow the code's own import/call structure.

4. **Map the affected code region.** Once you identify the suspicious function or module, read it in full context. Check its callers (incoming calls), its dependencies (outgoing calls), and related test files. Build a mental map of the 2-5 files that are relevant to the fix. Document this map explicitly before proceeding.

5. **Formulate a precise fix hypothesis.** Based on the localization, state exactly what needs to change and why. Identify whether the fix is: (a) a logic correction in a single function, (b) a missing edge-case handler, (c) an incorrect API call or argument, (d) a type/interface mismatch, or (e) a multi-site coordinated change. If multi-site, list all locations.

6. **Implement the minimal correct fix.** Make the smallest change that resolves the issue. Do not refactor surrounding code, add unrelated improvements, or change formatting. For multi-file fixes, make changes in dependency order (deepest dependency first). After each edit, verify the file is syntactically valid.

7. **Verify the fix by running tests.** Re-run the originally failing test(s) to confirm they now pass. Run the broader test suite for the affected module to check for regressions. If no tests exist, create a minimal test that captures the bug and verify it passes with your fix.

8. **Validate no regressions in related code.** Check that your fix doesn't break callers or dependents by running related test files. If the fix changes a public API or shared utility, verify all references still work. Use find-references to enumerate potential impact sites.

9. **Summarize the resolution.** Document what the root cause was, what was changed, and why, in a concise format suitable for a PR description. Include the specific files modified and the test results.

## Concrete Examples

**Example 1: Fixing a non-crashing data transformation bug**

```
User: Fix this issue — pandas DataFrame.groupby().transform() returns
wrong values when the transform function returns a scalar and the
group has mixed dtypes in other columns.

Approach:
1. Parse issue: transform() returns incorrect values; trigger condition
   is scalar return + mixed dtypes in non-grouped columns.
2. Reproduce: Write a test with a DataFrame having mixed-dtype columns,
   group by one column, and apply a scalar-returning transform. Confirm
   the output values are wrong.
3. Localize semantically: Go to definition of DataFrame.transform() →
   trace into groupby/generic.py → find the _transform_fast path →
   identify where scalar results are broadcast back to the group.
4. Map: groupby/generic.py (transform dispatch), groupby/ops.py
   (fast-path aggregation), core/dtypes/cast.py (dtype coercion).
5. Hypothesis: The fast path incorrectly coerces the scalar result
   through a dtype-sensitive code path that assumes homogeneous dtypes.
6. Fix: Add a dtype check before the fast path; fall back to the
   slow path when mixed dtypes are detected in the frame.
7. Verify: Original repro test passes. Existing groupby test suite
   passes (847 tests, 0 failures).
8. Regression check: find-references on _transform_fast confirms
   no other callers are affected.

Output:
- Modified: pandas/core/groupby/generic.py (lines 412-418)
- Root cause: _transform_fast assumed uniform dtypes across all columns
- Fix: Added dtype homogeneity check, routing mixed-dtype frames
  through _transform_general instead
- Tests: All 847 groupby tests pass; added 1 new regression test
```

**Example 2: Resolving a cross-module import/configuration issue**

```
User: After upgrading Django, our custom middleware is silently
not executing. No errors, just skipped. Issue #4521.

Approach:
1. Parse issue: Middleware exists but doesn't run post-upgrade. No
   error (non-crashing defect — exactly the ambiguous context where
   semantic navigation excels over grep).
2. Reproduce: Run request through the middleware stack, confirm via
   logging that the custom middleware's __call__ is never invoked.
3. Localize: Go to definition of the middleware class → check its
   __init__ signature → trace Django's middleware loading in
   django/core/handlers/base.py → find adapt_method_mode() which
   inspects middleware signatures to determine sync/async handling.
4. Map: our_app/middleware.py, django/core/handlers/base.py,
   django/utils/deprecation.py (MiddlewareMixin).
5. Hypothesis: Django upgrade changed middleware detection logic;
   our middleware lacks the expected method signature for the new
   sync/async adaptation layer.
6. Fix: Update middleware class to inherit from MiddlewareMixin
   (which provides the expected __call__ protocol) or add the
   explicit __acall__ method for async compatibility.
7. Verify: Request now passes through middleware. Middleware test
   suite passes.

Output:
- Modified: our_app/middleware.py (added MiddlewareMixin inheritance)
- Root cause: Django 4.x middleware adapter requires explicit
  sync/async protocol; plain classes are silently skipped
- Fix: Inherit from MiddlewareMixin to provide correct __call__
- Tests: 12/12 middleware tests pass, full integration suite green
```

**Example 3: Budget-aware resolution of a complex test failure**

```
User: The CI is failing on test_serialize_nested_generic_types.
I have no idea where to start — the codebase is huge.

Approach:
1. Parse: Single test failure, unknown location in large codebase.
   Budget: allocate ~30% of effort to localization, ~20% to
   understanding, ~30% to fix, ~20% to verification.
2. Reproduce: Run the specific test, capture the assertion error —
   it expects a serialized dict but gets None.
3. Localize (budget-aware — use semantic tools, not grep):
   - Go to the test definition, read the setup and assertion
   - Go to definition of the serialize() function under test
   - Trace into the type-dispatch logic for generic types
   - Use find-references on the GenericSerializer class to
     understand registration
   - Identify that nested generics (e.g., List[Optional[int]])
     hit an unhandled branch returning None
4. Map: tests/test_serialize.py, core/serializers/dispatch.py,
   core/serializers/generic.py (3 files total).
5. Fix: Add recursive handling for nested generic type arguments
   in the dispatch function.
6. Verify: Failing test now passes. Run full serializer test suite
   (203 tests) — all green.

Output:
- Modified: core/serializers/dispatch.py (lines 89-97)
- Root cause: Type dispatch unwrapped one level of generic args
  but not nested generics
- Fix: Recursive unwrap of typing.get_args() for nested generics
- Tests: 203/203 serializer tests pass
```

## Best Practices

**Do:**
- Always start from the failure point and trace inward using definition/reference chains — never start by reading random files
- Use go-to-definition and find-references as your primary navigation tools; reserve grep/find for string literals, config values, and error messages that aren't function names
- Keep an explicit mental map of the 2-5 relevant files before attempting any edit
- Track your remaining budget (turns, context) and switch from exploration to action when ~40% remains
- Verify fixes with the specific failing test first, then expand to the module test suite
- Compress your understanding of explored-but-irrelevant code into a one-line summary and move on

**Avoid:**
- Do not grep for function names when you can go-to-definition — grep returns all string matches including comments, documentation, and unrelated code
- Do not read entire large files top-to-bottom; use document-symbols to get the structure, then read specific functions
- Do not attempt a fix before confirming you can reproduce the failure — you need a verification signal
- Do not make speculative multi-site changes without first mapping all affected locations via find-references
- Do not exhaust your context exploring tangential code paths; set a localization budget and stick to it

## Error Handling

- **Cannot reproduce the issue:** Check if the issue requires specific environment setup, data fixtures, or configuration. Look for setup instructions in the test file or conftest. If still unreproducible, state this clearly and ask for reproduction steps.
- **Semantic navigation tools unavailable:** Fall back to structured grep patterns — search for `def function_name`, `class ClassName`, and `import` statements. Use file structure (directory names, `__init__.py` exports) as a secondary navigation aid.
- **Fix passes the target test but breaks others:** Your change likely violated an assumption made by other callers. Use find-references to enumerate all callers, understand the broken test's expectation, and adjust your fix to satisfy both constraints. Avoid special-casing.
- **Context window approaching limit on long investigations:** Summarize your findings so far (files mapped, hypothesis formed, changes planned) and continue from that summary. Prioritize executing the fix over further exploration.
- **Multiple plausible root causes identified:** Rank by proximity to the failure point. Fix the most proximate cause first and re-test. Deeper causes often resolve themselves or become clearer after the proximate fix.

## Limitations

- This methodology is optimized for bug fixes and localized feature additions in existing codebases. It is less applicable to greenfield development or large-scale architectural redesigns.
- Semantic navigation (go-to-definition, find-references) requires that the codebase has reasonable structure — heavily metaprogrammed or dynamically-generated code may not resolve through static analysis.
- The budget-aware approach assumes a finite interaction horizon. For truly open-ended exploratory tasks with no clear acceptance criteria, the phased structure may feel overly rigid.
- Test-driven verification requires that the project has a working test infrastructure. In codebases without tests, verification must rely on manual inspection or ad-hoc scripts.
- The approach achieves its best results on Python codebases (where SWE-bench is evaluated). The principles transfer to other languages, but tool availability and navigation quality may vary.

## Reference

**Paper:** [SWE-Master: Unleashing the Potential of Software Engineering Agents via Post-Training](https://arxiv.org/abs/2602.03411v1) — Song et al., 2026. Look for: Section 3 (inference framework and LSP tool design), Section 4 (trajectory synthesis and difficulty-based filtering), Section 5 (RL with forced submission and clip-higher GRPO), and Section 6.5 (summary-based context management). Code: [github.com/RUCAIBox/SWE-Master](https://github.com/RUCAIBox/SWE-Master).