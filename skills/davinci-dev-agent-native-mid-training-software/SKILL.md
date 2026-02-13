---
name: "davinci-dev-agent-native-mid-training-software"
description: "Apply daVinci-Dev's agent-native workflow to software engineering tasks: navigate repos, localize bugs, plan edits, apply structured patches, and verify with tests. Use when asked to 'fix this issue in the repo', 'resolve this bug across files', 'apply agentic SWE workflow', 'navigate and patch this codebase', 'use agent-native approach to solve this', or 'debug and fix with test verification'."
---

This skill enables Claude to tackle complex, multi-file software engineering tasks using the agent-native workflow from daVinci-Dev. Instead of generating a single code block, Claude systematically navigates the repository to build context, localizes the relevant code, plans precise edits using search-and-replace operations, and verifies correctness through test execution — mirroring the think-act-observe loop that produces state-of-the-art results on SWE-Bench Verified (58.5% resolution rate).

## When to Use

- When the user reports a bug described in a GitHub issue and asks you to fix it across a real repository
- When resolving a failing test requires understanding multiple interacting files before editing
- When a task requires navigating an unfamiliar codebase to find the right files to change
- When the user wants a structured, verifiable approach to patching code (not just a suggested diff)
- When fixing an issue that spans multiple files with interdependent changes
- When the user asks you to work through a software engineering problem the way an autonomous agent would

## Key Technique: Agent-Native Workflows

daVinci-Dev identifies that effective software engineering agents need two complementary types of experience. **Contextually-native trajectories** preserve the full information flow an agent encounters: the issue description, repository structure, relevant file contents, and the reasoning chain that leads to a specific edit. **Environmentally-native trajectories** go further — they capture actual tool invocations (bash commands, file views, string replacements) and real execution feedback (test output, error messages) from running against live repositories.

The critical insight is **distribution fidelity**: training data must match the actual experience of working in a codebase. Static code-diff pairs miss the navigation, search, dead-ends, and iterative refinement that characterize real development. The daVinci-Dev workflow structures each task as a loop of *think* (reason about what to do next), *act* (invoke a tool — search, view, edit, or run tests), and *observe* (read the tool output and update your plan). This loop continues until tests pass or the problem is resolved.

The action space consists of three core tools: **bash** (run arbitrary shell commands for searching, grepping, running tests), **str_replace_editor** (view files, create files, perform exact string replacements, insert lines), and **submit** (finalize when done). Edits are expressed as precise search-and-replace operations rather than full-file rewrites, minimizing unintended changes and making each modification auditable.

## Step-by-Step Workflow

1. **Parse the issue**: Read the bug report, feature request, or failing test. Extract the expected behavior, actual behavior, reproduction steps, and any stack traces or error messages. Identify key symbols (function names, class names, error strings) to search for.

2. **Explore repository structure**: Run `find` or `ls` on the project root to understand the directory layout. Identify the language, framework, test infrastructure, and entry points. Read the README or setup files if the project is unfamiliar.

3. **Localize with targeted search**: Use `grep -rn` or `rg` with the key symbols from step 1 to find candidate files. Prioritize: error messages first, then function/class names, then module paths mentioned in stack traces. Read each candidate file to confirm relevance.

4. **Build deep context on relevant files**: For each file you plan to modify, read the full file (or the relevant section). Understand the surrounding logic, data flow, and any callers/callees. Check imports and type definitions that constrain the change.

5. **Formulate an edit plan**: Before touching any code, articulate the precise changes needed. State which file, which lines, what the current code does, and what the replacement should do. If multiple files need changes, determine the correct order (e.g., update the interface before the implementation).

6. **Apply edits as exact string replacements**: Use the Edit tool with `old_string` / `new_string` pairs. Each replacement should be minimal and surgical — change only what is necessary. Never rewrite entire files when a targeted replacement suffices.

7. **Run the relevant tests**: Execute the test suite (or the specific test file) that covers the changed code. Read the output carefully. If tests fail, return to step 3 with the new error information and iterate.

8. **Verify no regressions**: Run a broader test sweep if available (e.g., the full test suite or a related test directory). Check that your changes don't break unrelated functionality.

9. **Review the final diff**: Examine all changes together. Confirm each edit is necessary, no debug code remains, and the fix is minimal. Remove any exploratory changes that aren't part of the solution.

10. **Summarize the resolution**: Explain what caused the issue, what was changed, and why each change is correct. Reference the specific files and line numbers modified.

## Concrete Examples

**Example 1: Fixing a KeyError in a Django view**

```
User: "I'm getting a KeyError: 'user_id' in our API endpoint at /api/profile/.
The traceback points to views.py line 42."

Approach:
1. Read the traceback to identify views.py and the specific line
2. Search for 'user_id' usage: grep -rn "user_id" src/api/
3. Read views.py to understand the profile endpoint logic
4. Discover that request.data['user_id'] assumes the key exists,
   but the frontend sends 'userId' (camelCase) after a recent refactor
5. Read the serializer and frontend code to confirm the field name change
6. Plan: update views.py to use request.data.get('userId') or update
   the serializer to accept both formats

Edit applied (views.py):
  old_string: "user_id = request.data['user_id']"
  new_string: "user_id = request.data['userId']"

7. Run: python manage.py test api.tests.test_profile -v2
   Result: 4 tests pass, 0 failures
8. Run: python manage.py test api/ -v2
   Result: 47 tests pass, 0 failures
```

**Example 2: Resolving a race condition in async code**

```
User: "Our background job processor sometimes processes the same job twice.
The relevant code is in workers/job_processor.py."

Approach:
1. Read workers/job_processor.py — find the job claim logic
2. Search for locking mechanisms: grep -rn "lock\|atomic\|transaction" workers/
3. Discover that claim_job() reads then updates the job status in two
   separate DB calls without a transaction
4. Read the database model to understand available locking primitives
5. Check if the ORM supports select_for_update or similar

Edit plan:
- Wrap the read-and-update in an atomic transaction with row-level locking
- File: workers/job_processor.py

Edit applied:
  old_string: |
    job = Job.objects.filter(status='pending').first()
    if job:
        job.status = 'processing'
        job.save()
  new_string: |
    from django.db import transaction
    with transaction.atomic():
        job = Job.objects.select_for_update(
            skip_locked=True
        ).filter(status='pending').first()
        if job:
            job.status = 'processing'
            job.save()

6. Run: python -m pytest workers/tests/test_job_processor.py -v
   Result: all 12 tests pass including the new concurrency test
7. Run the full worker test suite to check for regressions
```

**Example 3: Adding a missing feature across multiple files**

```
User: "Add CSV export to our report endpoint. We already have JSON export
in reports/views.py. The tests are in reports/tests/test_export.py."

Approach:
1. Read reports/views.py to understand the existing JSON export logic
2. Read reports/tests/test_export.py to see existing test patterns
3. Search for any existing CSV utilities: grep -rn "csv\|CSV" src/
4. Read reports/serializers.py to understand the data format

Edit plan (3 files):
a) reports/views.py — add CSVExportView class mirroring JSONExportView
b) reports/urls.py — add URL route for /reports/export/csv/
c) reports/tests/test_export.py — add test_csv_export test case

Apply edits sequentially:
- First: add the CSV view class to views.py
- Second: add the URL pattern to urls.py
- Third: add the test case

Run: python -m pytest reports/tests/test_export.py -v
Result: 6 tests pass (3 existing JSON + 3 new CSV)

Run: python -m pytest reports/ -v
Result: all 24 tests pass
```

## Best Practices

- **Do:** Always read a file before editing it. The agent-native approach treats "view then edit" as an atomic unit — never guess at file contents.
- **Do:** Use exact string replacement for edits rather than rewriting entire files. This mirrors the `str_replace_editor` action space and keeps changes minimal and auditable.
- **Do:** Run tests after every edit, not just at the end. The think-act-observe loop means each observation (test result) informs the next action.
- **Do:** Search broadly before editing narrowly. Cast a wide net with grep/search to find all relevant code, then focus edits on the minimal set of files.
- **Avoid:** Making multiple speculative changes at once without testing between them. Sequential, verified edits are more reliable than batch changes.
- **Avoid:** Skipping the context-building phase. The #1 cause of incorrect patches is insufficient understanding of the surrounding code. Read callers, callees, and tests before editing.

## Error Handling

- **Tests fail after your edit**: Do not immediately revert. Read the failure output carefully. The failing test may reveal a second location that needs updating, or it may expose an incorrect assumption in your edit. Iterate with the new information.
- **Cannot reproduce the bug**: Ask the user for exact reproduction steps, environment details, or the specific commit. Search for conditional logic (feature flags, environment checks) that might cause different behavior.
- **Search returns too many results**: Narrow with file-type filters (`--include='*.py'`), path restrictions, or more specific patterns. Use function signatures rather than common variable names.
- **Edit tool reports non-unique match**: Expand the `old_string` to include more surrounding context (additional lines before/after) until the match is unique.
- **Test suite is missing or broken**: Note this to the user. Write a minimal reproduction test if possible, verify the fix manually, and recommend adding proper test coverage.

## Limitations

- This workflow is most effective for **bug fixes and targeted feature additions** in codebases with existing test suites. For greenfield development or architectural redesigns, a planning-first approach may be more appropriate.
- The search-and-replace edit style works best for **surgical changes**. Large-scale refactors that touch dozens of files may benefit from automated refactoring tools instead.
- The approach assumes you can **run tests locally**. If the project requires special infrastructure (GPU, external services, CI-only tests), verification must be adapted accordingly.
- Complex issues involving **concurrency, distributed systems, or hardware-dependent behavior** may not be fully reproducible or verifiable through this workflow alone.

## Reference

**Paper**: [daVinci-Dev: Agent-native Mid-training for Software Engineering](https://arxiv.org/abs/2601.18418v2) — Look for Section 3 on agent-native data synthesis (contextually-native vs. environmentally-native trajectories) and Section 4 on the mid-training methodology that achieves 58.5% on SWE-Bench Verified with a 72B model.

**Code & Data**: [github.com/GAIR-NLP/daVinci-Dev](https://github.com/GAIR-NLP/daVinci-Dev) — The pipeline directory contains the data construction code; env_traj_utils shows the trajectory format with bash, str_replace_editor, and submit actions.