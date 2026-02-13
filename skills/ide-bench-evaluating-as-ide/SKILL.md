---
name: "ide-bench-evaluating-as-ide"
description: "Apply IDE-Bench's structured agent workflow for tackling real-world software engineering tasks: systematic exploration before editing, intent-driven tool transitions, and iterative test validation. Use when asked to 'fix a bug across multiple files', 'implement a feature in an unfamiliar codebase', 'refactor this module', 'optimize performance of this system', 'act like an IDE agent', or 'solve this engineering task systematically'."
---

# IDE-Bench Structured Agent Workflow for Software Engineering Tasks

This skill teaches Claude to operate as a high-performance IDE agent by applying the structured exploration-edit-verify workflow identified by IDE-Bench, a benchmark that evaluated frontier LLMs on 80 real-world engineering tasks across C/C++, Java, TypeScript, and Python codebases. The key insight: agents that spend 80-99% of iterations on deliberate exploration before editing — and that validate every edit by re-reading context before testing — dramatically outperform action-biased agents. This skill encodes those winning behavioral patterns into a repeatable workflow.

## When to Use

- When the user asks you to fix a bug in a codebase you haven't fully explored yet
- When implementing a new feature that touches multiple files across a project
- When refactoring code and you need to understand all callers and dependencies first
- When optimizing performance and you must locate bottlenecks before changing code
- When working in an unfamiliar repository with unknown structure, conventions, or test setup
- When a task spans multiple languages or layers (e.g., backend + frontend, C library + Python bindings)
- When the user says "figure out what's wrong and fix it" without pointing to a specific file

## Key Technique

IDE-Bench found that the highest-performing agents (Claude Opus at 86.25%, GPT-5.2 at 95%) share a distinctive behavioral signature: they spend the vast majority of their iterations on exploration rather than editing. Claude Opus dedicates 99.1% of iterations to exploration; even the most action-biased successful model (Grok 4.1 Fast) still spends 58.4%. The median successful run takes 21 iterations with the first edit not appearing until iteration 7. This is not indecisiveness — it is systematic context-gathering that prevents costly rework.

The tool transition data reveals a critical pattern: after editing a file, successful agents return to reading that file 55.9% of the time (verifying the edit landed correctly) and only 8% of edits transition directly to running tests. The dominant search pattern is self-chaining: `codebase_search → codebase_search` occurs 81.5% of the time, meaning effective agents run multiple sequential searches to triangulate relevant code before acting. Agents that skip this exploration phase or jump straight to testing after edits show significantly higher failure rates.

The practical implication: structure your work in distinct phases — explore broadly, localize precisely, edit minimally, verify thoroughly, then iterate. Never edit code you haven't read. Never run tests without first re-reading the file you changed. And never consider a task done without running the project's actual test suite.

## Step-by-Step Workflow

1. **Read the task description carefully and identify the task type** — classify it as feature implementation, bug fix, refactoring, or performance optimization. Each type has a different exploration pattern: bug fixes require finding the fault site, features require understanding extension points, refactoring requires mapping all usages, optimization requires profiling.

2. **Map the repository structure** — list the top-level directory, identify the build system (Makefile, CMake, package.json, pom.xml, Cargo.toml), locate the test directory, and find the entry point. Do NOT start reading source files yet; understand the project's shape first.

3. **Chain multiple searches to triangulate the relevant code** — run at least 3-5 targeted searches using different terms: the error message, the function name, the feature area, related types or interfaces. IDE-Bench data shows 81.5% of successful search activity is chained searches. Cast a wide net before narrowing.

4. **Read all files in the modification zone** — once searches converge on a set of files, read each one fully. Identify the coding style, conventions (naming, error handling, logging patterns), and any guards or invariants. Note the exact line numbers where changes are needed.

5. **Formulate an explicit plan before any edit** — state what you will change, in which files, and why. This is the "intent declaration" that IDE-Bench tracks. An explicit plan prevents drift and lets you verify that your changes match your stated goal.

6. **Make minimal, targeted edits** — change only what is necessary. IDE-Bench tasks are graded by test passage, not by style points. Smaller diffs have fewer failure modes. Prefer editing existing code over writing new files.

7. **Re-read the edited file immediately after each edit** — 55.9% of successful agent transitions go from `edit_file` back to `read_file`. Verify your edit rendered correctly: check indentation, syntax, that you didn't accidentally delete adjacent lines, and that the surrounding context still makes sense.

8. **Run the project's test suite** — use the repository's existing test commands (`npm test`, `pytest`, `make test`, `mvn test`, `cargo test`). Do not write your own tests unless the task explicitly requires it. Parse the output for failures and regressions.

9. **If tests fail, return to step 3** — search for the failure cause, re-read context, and iterate. Do not blindly retry the same edit. IDE-Bench shows that non-productive iterations (repeating failed actions) correlate strongly with overall failure. Analyze the error, form a new hypothesis, search for evidence, then edit.

10. **Verify completion against the original task description** — re-read the task requirements and confirm every acceptance criterion is met. Run the full test suite one final time. Only declare completion when all tests pass.

## Concrete Examples

**Example 1: Fixing a bug in a C++ game engine service**

User: "The collision detection is reporting false positives when objects are at the boundary of grid cells. Fix it."

Approach:
1. List project structure to find source layout (`src/`, `include/`, `tests/`)
2. Search for "collision" across the codebase — find `CollisionDetector.cpp`, `Grid.cpp`, `SpatialHash.h`
3. Search for "boundary" and "grid cell" to narrow to the specific boundary logic
4. Read `CollisionDetector.cpp` fully, then `Grid.cpp` — identify that the boundary check uses `<` instead of `<=`, causing objects exactly on cell edges to be missed by one cell and double-counted by another
5. State plan: "Change the boundary comparison in `Grid::getCellForPosition()` from strict `<` to `<=` on the upper bound"
6. Edit the single line in `Grid.cpp`
7. Re-read `Grid.cpp` to confirm the edit is correct
8. Run `make test` — all 47 tests pass, including the collision boundary tests
9. Confirm the fix matches the task description

Output:
```
Fixed boundary condition in Grid.cpp:142. Changed `pos.x < cellBoundary`
to `pos.x <= cellBoundary` (and same for y-axis on line 143). The strict
inequality caused objects exactly on cell boundaries to be assigned to the
wrong cell, producing duplicate collision pairs. All 47 tests pass.
```

**Example 2: Implementing a new feature in a MERN stack application**

User: "Add webhook retry logic with exponential backoff to the Event Callback System. Failed webhooks should retry up to 3 times."

Approach:
1. Map the project: `package.json` shows Express + MongoDB, `src/` has `routes/`, `models/`, `services/`
2. Search for "webhook" — find `services/webhookDispatcher.ts`, `models/WebhookEvent.ts`, `routes/callbacks.ts`
3. Search for "retry" and "backoff" — nothing exists yet, so this is net-new logic
4. Read `webhookDispatcher.ts` fully — it currently fires webhooks with `axios.post()` and logs failures but does not retry
5. Read `WebhookEvent.ts` — the model has a `status` field but no `retryCount` or `nextRetryAt`
6. Plan: (a) Add `retryCount` and `nextRetryAt` fields to the WebhookEvent model, (b) wrap the dispatch call in retry logic with delays of 1s, 4s, 16s, (c) add a scheduled job or post-failure re-queue mechanism
7. Edit `WebhookEvent.ts` to add the two fields with defaults
8. Re-read `WebhookEvent.ts` — confirmed
9. Edit `webhookDispatcher.ts` to add the retry loop with exponential backoff (`Math.pow(4, retryCount) * 1000`ms delay), capping at 3 retries
10. Re-read `webhookDispatcher.ts` — confirmed
11. Run `npm test` — 2 new test failures because the test fixtures don't include the new fields
12. Search for test fixtures — find `tests/fixtures/webhookEvents.json`
13. Read and edit the fixture file to include `retryCount: 0` and `nextRetryAt: null`
14. Re-read fixture, run `npm test` — all tests pass

Output:
```
Added exponential backoff retry logic to webhookDispatcher.ts. Failed
webhooks now retry up to 3 times with delays of 1s, 4s, and 16s.
Added retryCount and nextRetryAt fields to the WebhookEvent model.
Updated test fixtures. All tests pass.
```

**Example 3: Refactoring a Java web service**

User: "The SmartHub Operations Center has a god class in DeviceController.java. Break it into separate controllers by device category."

Approach:
1. Map the project structure — find `src/main/java/com/smarthub/controllers/`
2. Search for "DeviceController" — single 800-line file handling lights, thermostats, cameras, locks
3. Search for route annotations (`@Get`, `@Post`, Javalin `path()`) to inventory all endpoints
4. Read `DeviceController.java` fully — identify four clear clusters of methods by device type, plus shared helper methods
5. Read the Javalin app setup in `App.java` to understand how routes are registered
6. Plan: Create `LightController.java`, `ThermostatController.java`, `CameraController.java`, `LockController.java`. Extract shared helpers into `DeviceControllerBase.java`. Update route registration in `App.java`.
7. Create each controller file with extracted methods, preserving exact method signatures
8. Re-read each new file to verify correctness
9. Edit `App.java` to register the new controllers
10. Re-read `App.java`
11. Run `mvn test` — verify all integration tests pass
12. Delete the original `DeviceController.java` only after all tests pass

## Best Practices

- **Do: Spend at least 60% of your effort on exploration before making any edit.** IDE-Bench shows the best agents explore extensively. Reading code is cheap; incorrect edits are expensive.
- **Do: Chain 3-5 searches with different terms before concluding where to edit.** A single search often misses relevant code. Triangulate with function names, error strings, type names, and comments.
- **Do: Re-read every file immediately after editing it.** This catches malformed edits, wrong indentation, and accidental deletions before they cascade into test failures.
- **Do: Run the project's own test suite, not ad-hoc verification.** The repository's tests encode the actual acceptance criteria. Trust them over manual spot-checks.
- **Avoid: Jumping straight to editing after a single search.** This is the most common failure pattern in IDE-Bench — agents that act on incomplete context make wrong changes.
- **Avoid: Running tests immediately after an edit without re-reading first.** Only 8% of successful edit transitions go directly to testing. The extra read step catches errors before the slower test cycle.
- **Avoid: Repeating the same failed action.** If an edit or test fails, search for new context and form a different hypothesis. Non-productive iteration loops (18% of iterations for low-performing agents) are the hallmark of failure.

## Error Handling

- **Edit doesn't apply cleanly**: The target text wasn't found or was ambiguous. Re-read the file to get the exact current content, then retry with a precise match.
- **Tests fail after a correct-looking edit**: The change may have broken an invariant elsewhere. Search for other callers or dependents of the modified code. Read the failing test to understand what it actually asserts.
- **Build fails with syntax errors**: Re-read the edited file. Check for mismatched braces, missing semicolons, or indentation errors introduced by the edit. Fix the syntax before attempting any other changes.
- **Cannot find relevant code after multiple searches**: Broaden search terms. Try searching for the data type instead of the function, or for log messages, or for configuration keys. List directory contents to discover unexpected file locations.
- **Task seems done but some tests still fail**: Read the failing test carefully. It may test an edge case your fix didn't cover, or it may be a pre-existing failure. Distinguish between regressions you caused and pre-existing issues.

## Limitations

- This workflow is optimized for tasks where the codebase already has a test suite. If no tests exist, the verify phase must rely on manual validation or writing new tests — which is a different skill.
- The exploration-heavy approach works best when the codebase is of moderate size (hundreds to low thousands of files). For monorepos with millions of lines, additional strategies like architectural documentation review and package-boundary analysis are needed.
- Performance optimization tasks may require runtime profiling (e.g., `perf`, `valgrind`, browser DevTools) that goes beyond search-and-read exploration. The workflow still applies, but the "search" phase should include profiling tool output.
- The 80-99% exploration ratio is a statistical finding across many tasks. Some trivial fixes (typos, one-line config changes) genuinely need less exploration. Use judgment — but when in doubt, explore more.
- Language-specific tooling matters: IDE-Bench found that TypeScript async patterns and Java web frameworks (Javalin) have lower success rates across all models, suggesting these require extra care and deeper reading.

## Reference

**Paper**: [IDE-Bench: Evaluating Large Language Models as IDE Agents on Real-World Software Engineering Tasks](https://arxiv.org/abs/2601.20886v2) — Focus on Section 4 (tool transition analysis), Table 8 (exploration ratios), and the finding that the best agents spend 80-99% of iterations exploring before editing.