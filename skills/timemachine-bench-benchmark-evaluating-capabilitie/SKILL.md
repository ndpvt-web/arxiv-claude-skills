---
name: "timemachine-bench-benchmark-evaluating-capabilitie"
description: "Systematic dependency migration for Python projects. Diagnose and fix test failures caused by dependency updates using a minimal-edit, test-driven agent loop. Triggers: 'migrate dependencies', 'fix breaking changes from package update', 'upgrade library version', 'dependency migration', 'tests broke after pip upgrade', 'adapt code to new API version'"
---

# Dependency Migration Agent: Systematic Python Library Upgrade Workflow

This skill enables Claude to perform repository-level dependency migration — adapting Python codebases when dependency updates cause test failures. Based on the TimeMachine-bench methodology (Fujii et al., EACL 2026), it implements a disciplined agent loop: run tests, analyze failures, apply minimal targeted edits, re-test, and revert when wrong. The key insight from the paper is that naive LLM-driven migration produces two failure modes — *spurious solutions* (changes that pass tests by exploiting low coverage rather than fixing the real issue) and *unnecessary edits* (over-broad changes from poor tool-use strategy). This skill teaches the disciplined approach that avoids both.

## When to Use

- When the user reports test failures after upgrading a Python dependency (e.g., `pytest`, `numpy`, `pandas`, `django`, `pydantic`)
- When the user asks to migrate a codebase from one library version to another (e.g., "upgrade from Pydantic v1 to v2")
- When `pip install --upgrade` or a lockfile update breaks existing tests
- When the user needs to adapt code to breaking API changes in a new library release
- When performing a planned major version bump of a dependency across a project
- When investigating which code changes are required by a dependency changelog

## Key Technique

The TimeMachine-bench paper establishes that software migration is fundamentally a **test-driven, iterative repair task**. The benchmark constructs real-world migration problems by finding GitHub repositories where test suites pass under old dependencies but fail under new ones. The agent's job: modify source code (never test code) until tests pass again under the new dependency versions.

The agent operates with a constrained tool set: file navigation (`list_dir`, `search_dir`, `search_file`, `view_file`), surgical editing (`edit_file` with line ranges, `replace_all_in_file` for bulk renames), revert capability (`revert_last`), and containerized test execution (`execute_tests`). The critical constraint is **minimality**: only change what the dependency update broke. The paper's precision metric (`prec@1`) measures whether edits align with the gold-standard fix locations. High pass rate with low precision signals a spurious solution — the code passes tests but via incorrect changes.

The paper's main finding is that even strong models (GPT-4o, Claude) achieve only ~30-40% pass@1 on verified migration tasks, with precision often much lower. The primary failure modes are: (1) making unnecessary edits to files unrelated to the breaking change, driven by over-exploration, and (2) applying surface-level fixes (deleting tests, stubbing functions, catching all exceptions) that mask rather than resolve the incompatibility. The disciplined workflow below avoids these traps.

## Step-by-Step Workflow

1. **Establish the migration context.** Identify the exact dependency that was updated, its old version, and its new version. Read the dependency's changelog or release notes for the specific version range to understand what APIs changed, what was deprecated, and what was removed.

2. **Run the full test suite to capture the baseline failure.** Execute the project's test command (e.g., `pytest`) and record every failing test with its full traceback. Do not skip this — you need the exact error messages to drive targeted fixes.

3. **Categorize failures by root cause.** Group the failing tests by the type of breakage: removed API, renamed function/class, changed return type, altered default behavior, removed parameter, new required parameter, or changed import path. Each category maps to a specific fix pattern.

4. **Locate affected source files using search, not guessing.** For each failure root cause, use `grep`/search to find all call sites of the broken API in the source code (excluding test files). Map the full scope of changes needed before editing anything.

5. **Apply minimal, targeted edits to source code only.** Fix one failure category at a time. Use precise line-range edits or regex-based bulk replacements. Never modify test files. Never add broad exception handlers. Never delete or stub out functionality. Each edit should directly address a specific API incompatibility.

6. **Re-run tests after each edit batch.** After fixing one category of failures, run the test suite again. Confirm that the targeted failures are resolved and no new failures were introduced. If new failures appear, analyze whether they are secondary effects of the dependency change or regressions from your edit.

7. **Revert immediately if an edit does not improve results.** If a change does not reduce test failures or introduces regressions, undo it before proceeding. Do not accumulate speculative changes. The revert-and-retry loop is essential for maintaining edit precision.

8. **Validate the final state comprehensively.** Once all tests pass, review the complete diff. Every changed line should be directly attributable to a specific API incompatibility in the upgraded dependency. Remove any edits that are not strictly necessary (formatting changes, unused imports added then removed, debug prints).

9. **Check for spurious passes.** If the fix seems too simple for the scope of the migration, verify that the tests actually exercise the changed code paths. A test suite that passes because changed code is never called is a false positive — the migration is incomplete.

10. **Document the migration.** Summarize what changed in the dependency, what source files were modified, and why each change was necessary. This serves as both a commit message guide and a review aid.

## Concrete Examples

**Example 1: Pydantic v1 to v2 Migration**

User: "I upgraded pydantic from 1.10 to 2.0 and now 15 tests are failing with `AttributeError: type object 'BaseModel' has no attribute '__fields__'`"

Approach:
1. Run `pytest --tb=short` to capture all failures. Confirm they cluster around `__fields__`, `.dict()`, and `validator` decorator usage.
2. Search source code: `grep -rn "__fields__\|\.dict()\|@validator" src/ --include="*.py"` — find 8 files with affected calls.
3. Categorize: (a) `__fields__` -> `model_fields`, (b) `.dict()` -> `.model_dump()`, (c) `@validator` -> `@field_validator` with mode parameter.
4. Apply edits file by file:
   - Replace `cls.__fields__` with `cls.model_fields` (note: value format changed too — `.outer_type_` becomes `.annotation`)
   - Replace `.dict()` with `.model_dump()`
   - Replace `@validator("field")` with `@field_validator("field")` and adjust function signature (add `@classmethod` decorator, change `cls` parameter handling)
5. Run tests after each file group. Revert and re-examine if new failures appear.
6. Final diff review: confirm every edit maps to a documented Pydantic v2 migration item.

Output:
```
Modified 8 files, 34 lines changed.
All 15 previously-failing tests now pass. No new failures introduced.
Changes map to Pydantic v2 migration guide sections: field-access, serialization, validators.
```

**Example 2: NumPy Deprecation Removal**

User: "After upgrading numpy to 2.0, I'm getting `AttributeError: module 'numpy' has no attribute 'bool'`"

Approach:
1. Run tests. Confirm failures relate to removed type aliases: `np.bool`, `np.int`, `np.float`, `np.complex`, `np.object`, `np.str`.
2. Search: `grep -rn "np\.bool\b\|np\.int\b\|np\.float\b\|np\.str\b\|np\.object\b\|np\.complex\b" src/` — locate all occurrences.
3. These aliases were deprecated in NumPy 1.20 and removed in 2.0. Direct replacements: `np.bool` -> `np.bool_` (or Python `bool`), `np.int` -> `np.int_` (or Python `int`), etc.
4. Use bulk replacement per file, being careful with regex to avoid matching `np.bool_` or `np.boolean`:
   - Pattern: `np\.bool\b(?!_)` -> `np.bool_`
   - Pattern: `np\.int\b(?!_|e)` -> `np.int_`
5. Run tests. All pass.
6. Review diff: every change is a direct alias substitution. No unnecessary edits.

Output:
```
Modified 3 files, 12 lines changed.
Replaced deprecated NumPy type aliases with their concrete equivalents.
All tests pass under numpy 2.0.
```

**Example 3: Django REST Framework Serializer Changes**

User: "Upgraded djangorestframework from 3.14 to 3.15 and several serializer tests are failing"

Approach:
1. Run `python manage.py test --verbosity=2`. Capture failing test names and tracebacks.
2. Analyze errors: `TypeError: __init__() got an unexpected keyword argument 'allow_blank'` on `IntegerField` — DRF 3.15 tightened which kwargs each field type accepts.
3. Search: `grep -rn "IntegerField.*allow_blank\|BooleanField.*allow_blank" src/ --include="*.py"` to find all incorrectly-parameterized fields.
4. Remove `allow_blank=True` from non-string field types (it was silently ignored before, now raises).
5. Run tests. If additional failures exist, repeat the categorize-search-fix cycle.
6. Final check: diff shows only removal of invalid kwargs. No other changes.

Output:
```
Modified 2 serializer files, removed `allow_blank` from 5 non-string fields.
All tests pass. Changes are minimal and directly address the stricter validation in DRF 3.15.
```

## Best Practices

- **Do:** Always run the test suite before and after every edit to maintain a clear signal of progress.
- **Do:** Read the dependency's changelog or migration guide before writing any code. Understanding the *intended* API changes prevents guesswork.
- **Do:** Use bulk regex replacement (`replace_all_in_file` or `sed`) for mechanical renames that apply uniformly across files. This is faster and less error-prone than individual line edits.
- **Do:** Keep a mental (or written) tally of which failures are resolved and which remain. Track progress by failure count.
- **Avoid:** Modifying test files. Tests define the correctness specification. Changing tests to pass is not migration — it is cheating.
- **Avoid:** Adding broad `try/except` blocks, `# type: ignore` comments, or monkey-patches as "fixes." These mask the real incompatibility and will break in production.
- **Avoid:** Making formatting, style, or refactoring changes alongside migration fixes. These pollute the diff, reduce precision, and make review harder.
- **Avoid:** Over-exploring the codebase. Search for specific broken APIs rather than reading every file. Unfocused exploration leads to unnecessary edits.

## Error Handling

- **Test suite won't run at all:** The dependency update may have broken imports at module level. Check for `ImportError` or `ModuleNotFoundError` in the traceback. Fix import-level breakages first before addressing test-level failures.
- **Circular fixes (fix A breaks B, fix B breaks A):** This indicates the two changes interact. View both call sites together, understand the dependency, and apply a combined fix that satisfies both constraints.
- **Tests pass but diff is suspiciously large:** Review every edit. If changes exist in files not related to the upgraded dependency, revert those files and re-run tests. They are likely unnecessary.
- **Some tests are flaky or timeout:** Distinguish between migration-caused failures (deterministic, same error on every run) and pre-existing flakiness. Only fix migration-caused failures.
- **No changelog available for the dependency:** Check the library's git history, release notes on GitHub/PyPI, or use `pip show <package>` to find the project URL. As a fallback, compare the old and new source of the library's changed modules directly.

## Limitations

- This approach requires a working test suite. If the project has no tests or very low test coverage, migration correctness cannot be verified through testing alone — manual review becomes necessary.
- The workflow targets Python dependency migrations specifically. Other ecosystems (npm, cargo, go modules) follow similar principles but have different tooling.
- Migrations involving fundamental architectural changes (e.g., sync-to-async conversion when upgrading to Django 4+ channels) may require more than mechanical API substitution and could need design-level decisions beyond this workflow.
- If the dependency update introduces new required configuration (environment variables, config files), test failures may not clearly point to the root cause. Check the dependency's "upgrading" documentation for setup-level changes.
- The minimal-edit principle optimizes for precision and reviewability, not for adopting new features. After migration, a separate pass may be warranted to adopt improved APIs from the new version.

## Reference

**Paper:** [TimeMachine-bench: A Benchmark for Evaluating Model Capabilities in Repository-Level Migration Tasks](https://arxiv.org/abs/2601.22597v1) (Fujii et al., EACL 2026)
**Repository:** [tohoku-nlp/timemachine-bench](https://github.com/tohoku-nlp/timemachine-bench)
**Key takeaway:** Read Section 5 (Results & Analysis) for the detailed breakdown of spurious solutions and unnecessary edits — understanding these failure modes is essential for building reliable migration agents. The precision metric (`prec@1`) that compares edit locations against gold-standard patches is particularly instructive for self-evaluating migration quality.