---
name: "rethinking-value-agent-generated-tests"
description: "Optimize agent test-writing strategy for issue resolution by reallocating interaction budget from excessive test generation to direct implementation. Use when: 'fix this bug', 'resolve this issue', 'patch this repo-level problem', 'debug and fix this failing code', 'implement this change and verify it works', 'solve this SWE-bench task'."
---

# Budget-Aware Test Strategy for LLM Code Agents

This skill teaches Claude to apply an empirically-grounded test-writing strategy when resolving software issues autonomously. Based on a study of six frontier LLMs across 500 real GitHub issues (SWE-bench Verified), the core finding is that agent-generated tests provide marginal utility when they consume interaction budget that could be spent on implementation attempts. Agents that write fewer tests and invest more turns in iterative implementation achieve comparable or superior resolution rates (3-8 percentage points higher). This skill encodes that insight into a concrete workflow: write minimal, assertion-based tests only when they serve a clear diagnostic purpose, and default to spending your budget on trying fixes.

## When to Use

- When resolving a repository-level bug or issue that requires editing code, running tests, and validating a patch
- When you catch yourself planning to write a large test suite before attempting any fix
- When debugging a failure and tempted to add print-statement-heavy test files instead of reading the existing test suite
- When operating under turn or token constraints where every interaction step matters
- When reviewing your own agent workflow and noticing repeated test-write-then-rerun cycles without code changes
- When a user asks you to "fix this issue" or "resolve this bug" on a real codebase

## Key Technique

### The Interaction Budget Trade-Off

LLM agents resolving repository issues operate within a finite interaction budget — a limited number of tool calls, file reads, code edits, and test executions. The conventional wisdom is that writing tests helps agents validate patches. However, empirical analysis across Claude Opus 4.5, GPT-5, Gemini 3 Pro, DeepSeek V3.2, Kimi K2 Thinking, and MiniMax M2 shows that **resolved and unresolved tasks exhibit similar test-writing frequencies**. Test writing does not discriminate between success and failure. What discriminates is how agents spend their remaining budget.

### Observational Feedback vs. Assertion-Based Checks

When agents do write tests, they overwhelmingly prefer "observational feedback" — print statements that reveal program state — over formal assertions that provide pass/fail signals. This makes agent-written tests function as expensive debugging logs rather than validation gates. The study found that when prompts were revised to increase test writing, resolution rates dropped 2-5%; when revised to decrease test writing, resolution rates held steady or improved. The mechanism is straightforward: every turn spent writing, debugging, and re-running a test file is a turn not spent trying an alternative implementation approach.

### The Actionable Principle

Treat your interaction budget as a scarce resource. Default to implementation-first, test-light workflows. When you do write a test, make it a single focused assertion that answers a specific diagnostic question — never a sprawling test file with print statements. If the existing test suite already covers the behavior, run it directly instead of writing new tests.

## Step-by-Step Workflow

1. **Read the issue and existing test suite first.** Before writing any code or tests, identify what the expected behavior is and whether existing tests already cover it. Run `grep` or `find` for test files related to the affected module.

2. **Check if existing tests reproduce the bug.** Run the repository's own test suite (or the specific failing test) to confirm the failure. If existing tests already capture the problem, skip to step 5. This is the single highest-value action.

3. **If no existing test covers the bug, write exactly one minimal reproduction.** Write a single test function (5-15 lines) with one or two assertions that demonstrate the failure. Do not write a test file with multiple test cases, helper functions, or print statements. Use the project's existing test framework and conventions.

4. **Run the reproduction test to confirm it fails.** Execute it once. If it passes unexpectedly, re-read the issue — you may have misunderstood the bug. Do not iterate on the test more than once; move to implementation.

5. **Spend the majority of your budget on implementation attempts.** Read the relevant source code, form a hypothesis, make an edit, and run the failing test. If it still fails, read the error, revise your hypothesis, and try a different edit. Each cycle should be: read-edit-run, not read-edit-write-new-test-run.

6. **Avoid test-iteration loops.** If you find yourself editing a test file more than once, stop. You are spending budget on test maintenance instead of bug fixing. Delete the test, use the existing suite, or write a simpler one.

7. **Use assertions, never print statements.** If you need to understand program state, use a debugger invocation or add a temporary `assert value == expected, f"got {value}"` in the source code itself. Print-based "tests" provide no pass/fail signal and waste output parsing budget.

8. **Validate the final patch with the existing test suite.** Once your fix looks correct, run the original failing tests plus any related regression tests. This is the only test execution that matters for validation.

9. **Remove any temporary reproduction tests if they duplicate existing coverage.** If the repo's own tests now pass and cover the same behavior, delete your test to keep the patch minimal.

10. **Reserve 20% of your budget for alternative approaches.** If your first implementation attempt fails after 3-4 edit-run cycles, step back and try a fundamentally different approach rather than writing more tests to "understand" the problem better.

## Concrete Examples

**Example 1: Fixing a Django ORM bug**

```
User: Fix issue #4821 — QuerySet.union() drops ordering when combined with .values()

Approach:
1. Read the issue. Identify that existing test in tests/queries/test_qs_combinators.py
   covers union behavior but not the .values() + ordering interaction.
2. Run: python -m pytest tests/queries/test_qs_combinators.py — confirm no test
   currently fails for this specific case.
3. Write ONE test (7 lines):
   def test_union_preserves_order_with_values(self):
       qs1 = Number.objects.filter(num__gte=0).values('num').order_by('num')
       qs2 = Number.objects.filter(num__lte=5).values('num')
       result = list(qs1.union(qs2).order_by('num').values_list('num', flat=True))
       assert result == sorted(result), f"Expected sorted, got {result}"
4. Run it — confirms failure.
5. Read django/db/models/sql/compiler.py around the union/order_by handling.
6. Edit the compiler to preserve ORDER BY clause through union when .values() is used.
7. Run the test — passes. Run full test_qs_combinators.py — all pass.
8. Done in ~8 interaction steps. A test-heavy approach would spend 4+ steps writing
   multiple test cases, leaving fewer turns for the actual fix.
```

**Example 2: Debugging a Python library's incorrect return type**

```
User: The parse_duration() function returns None instead of raising ValueError
for invalid input strings. Fix it.

Approach:
1. Read existing tests for parse_duration — find tests/test_duration.py with
   10 existing test cases. None test invalid input explicitly.
2. Do NOT write a new test file. Instead, go straight to the source:
   read src/utils/duration.py, find the parse_duration function.
3. Identify the bug: a bare `except` clause catches the ValueError and returns None.
4. Edit: replace bare except with specific exception handling, let ValueError propagate.
5. Write ONE assertion as validation:
   def test_parse_duration_invalid_raises(self):
       with pytest.raises(ValueError):
           parse_duration("not-a-duration")
6. Run it — passes. Run full test suite — all pass.
7. Total: 6 interaction steps. Budget saved by not writing a comprehensive test
   matrix for every invalid input pattern.
```

**Example 3: Resisting the urge to over-test**

```
User: Fix the CSV export that silently drops rows containing unicode characters.

Anti-pattern (what NOT to do):
1. Write test_csv_unicode.py with 12 test cases covering various Unicode ranges
2. Add print statements to dump intermediate byte representations
3. Run tests, read output, add more prints to narrow down the issue
4. Discover the bug after 15 interaction steps, with only 5 left for fixing
5. Run out of budget before verifying the fix against edge cases

Correct approach:
1. Read the CSV export function in src/export/csv_writer.py
2. Spot the bug: file opened with encoding='ascii' instead of 'utf-8'
3. Fix the encoding parameter
4. Run existing test suite — if it has CSV tests, they now pass
5. If no existing test covers unicode, add ONE:
   def test_csv_export_unicode(self):
       result = export_to_csv([{"name": "cafe\u0301"}])
       assert "cafe\u0301" in result
6. Done in 5 interaction steps
```

## Best Practices

**Do:**
- Run existing tests before writing new ones — the repo's test suite is almost always more comprehensive than anything you'll write on the fly
- Write at most one small test with 1-2 assertions when no existing test covers the bug
- Treat each interaction step as currency: spend it on implementation attempts, not test scaffolding
- Use assertion messages (`assert x == y, f"got {x}"`) as a two-in-one: both validation and diagnostic output

**Avoid:**
- Writing test files with more than one test function during issue resolution
- Using print statements in tests — they provide no pass/fail signal and consume output parsing budget
- Iterating on test code more than once — if the test needs debugging, it's too complex
- Writing tests before reading the source code — you cannot test what you do not understand
- Generating "comprehensive" test suites as a substitute for reading and understanding the codebase
- Creating test infrastructure (fixtures, helpers, base classes) for a single-use reproduction test

## Error Handling

**Your reproduction test passes unexpectedly:** You misunderstood the issue. Re-read the issue description and the relevant source code. Do not write more tests to "explore" — read the code directly.

**Existing test suite fails in unrelated areas:** Ignore unrelated failures. Focus only on the tests relevant to the issue. Do not spend budget investigating pre-existing failures.

**Your fix breaks other tests:** Revert your change, re-read the failing tests to understand the expected behavior, and try a narrower fix. Do not modify the existing tests to accommodate your patch — they encode correct behavior.

**You're running low on budget with no fix:** Stop writing tests entirely. Use remaining turns for: (1) re-reading the issue, (2) reading stack traces or error messages from the last run, (3) trying one more focused code edit. A last-ditch test will not save you.

## Limitations

- This strategy is optimized for **issue resolution tasks** (bug fixes, feature patches) under budget constraints. For greenfield development or TDD-driven feature design, conventional test-first approaches may still be appropriate.
- The findings are based on SWE-bench Verified (500 Python repository issues). Results may differ for other languages, domains, or issue types, though the budget trade-off principle generalizes.
- When the bug is in complex concurrent, stateful, or distributed code where reading alone is insufficient to understand behavior, a carefully constructed test may be worth the budget cost.
- This skill does not apply to writing tests as the primary deliverable (e.g., "write tests for this module"). It applies only when tests are a means to an end (resolving an issue).
- Some organizations require tests as part of every patch. In that case, write the minimal test after the fix is confirmed, not before.

## Reference

[Rethinking the Value of Agent-Generated Tests for LLM-Based Software Engineering Agents](https://arxiv.org/abs/2602.07900v1) — Chen et al., 2026. Key finding: across six frontier LLMs on SWE-bench Verified, reducing agent test-writing volume did not hurt resolution rates and sometimes improved them by 3-8 percentage points, because interaction budget reallocated from test generation to implementation attempts yielded better outcomes.