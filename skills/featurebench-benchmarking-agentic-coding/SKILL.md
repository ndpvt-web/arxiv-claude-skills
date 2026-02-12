---
name: "featurebench-benchmarking-agentic-coding"
description: "Extract feature-level coding tasks from repositories using test-driven dependency graph tracing. Use when the user says 'create a feature benchmark', 'extract coding tasks from tests', 'build a test-driven benchmark', 'evaluate agent coding ability', 'isolate features from a codebase', or 'generate feature development tasks'."
---

# FeatureBench: Test-Driven Feature Task Extraction and Agentic Coding Evaluation

This skill enables Claude to extract feature-level coding tasks from real code repositories by tracing unit test dependencies through a call graph, separating feature code from the codebase, and constructing executable evaluation environments. The core technique — derived from the FeatureBench paper (ICLR 2026) — produces tasks that span multiple files, commits, and PRs, yielding far more realistic feature-development challenges than single-PR bug-fix benchmarks.

## When to Use

- When the user wants to create coding benchmarks from an existing open-source repository
- When evaluating an LLM coding agent's ability to implement multi-file features end-to-end
- When the user needs to automatically identify which code constitutes a "feature" by tracing from its tests
- When constructing sandboxed Docker environments for reproducible agent evaluation
- When the user wants to separate a feature from a codebase without breaking other functionality
- When building training data with verifiable correctness for agentic coding systems
- When assessing whether an agent can implement features from scratch vs. extend existing code

## Key Technique

FeatureBench's insight is that unit tests already encode feature boundaries. By executing tests under a dynamic tracer, you capture every function call a test triggers. This produces a directed dependency graph where nodes are functions (with source locations) and edges are call relationships. Functions touched only by the target feature's tests — and never by other passing tests — are the feature's extractable code. Removing that code should make the feature's tests fail while all other tests continue to pass. This "fail-to-pass / pass-to-pass" (F2P/P2P) invariant is the correctness guarantee.

The method uses an LLM classifier to distinguish test *targets* (functions under test) from test *utilities* (helpers, fixtures, mocks) among a test file's imports. Targets become the entry points for a breadth-first traversal of the dependency graph. Nodes encountered during P2P test execution are marked "remained" (must not be removed); nodes absent from P2P runs are marked "extracted" (safe to remove). The traversal is bounded to produce patches of 3,000–5,000 lines, yielding tasks of realistic feature complexity.

The evaluation protocol is strictly execution-based: an agent's solution is correct only if every F2P test passes *and* every P2P test still passes. Soft metrics (fraction of F2P tests passed) and efficiency metrics (token I/O) provide further signal. Environments are Dockerized for full reproducibility.

## Step-by-Step Workflow

1. **Select a repository and set up the environment.** Choose a Python repository with a pytest-based test suite. Write a minimal config specifying install commands (pip/conda), then build a Docker image with all dependencies. This is the only manual step.

2. **Collect and validate tests.** Use `pytest --collect-only` to enumerate all test files. Execute each test file; record which tests pass (these become P2P candidates) and which fail (potential F2P candidates if they test unimplemented features). Discard flaky tests by running twice.

3. **Build the dynamic dependency graph.** Re-run passing tests under Python's `sys.settrace` to capture every function call event. Construct a directed graph: each node stores `(function_name, file_path, line_number, [callees], is_p2p_involved)`. Serialize the graph as an adjacency list (JSON).

4. **Classify test imports as targets vs. utilities.** For each F2P test file, extract all imported symbols. Use an LLM prompt to classify each import: "Is this symbol a function/class being tested, or a helper used to set up the test?" Symbols classified as targets become BFS entry points.

5. **Traverse the graph to identify extractable code.** Starting from target entry points, perform BFS over the dependency graph. Mark each visited node: if it was also touched during any P2P test execution, label it "remained"; otherwise label it "extracted." Stop traversal when extracted code reaches 3,000–5,000 lines.

6. **Generate the code patch.** Produce a `patch.diff` that removes all "extracted" nodes from the repository source files. Ensure that removed function bodies are replaced with stubs that raise `NotImplementedError` or are deleted entirely, depending on whether other code imports them.

7. **Run post-verification.** Apply the patch to the repository. Execute all P2P tests — they must all pass. Execute all F2P tests — they must all fail. If either invariant is violated, adjust the extraction boundary (expand "remained" set) and repeat.

8. **Package the task environment.** For each verified task, create a Docker image containing: the patched repository (feature removed), the F2P test files (hidden from the agent but available for evaluation), the P2P test files (visible, to define the contract), and a task description specifying what feature to implement.

9. **Define evaluation levels.** Create two difficulty variants: **L1 (incremental)** where the agent receives the patched codebase and must extend it, and **L2 (from-scratch)** where the agent receives only interface specs and test signatures without repository context.

10. **Evaluate agent solutions.** Run the agent in the Docker sandbox. Collect its output patch. Apply it and execute both F2P and P2P test suites via pytest. Compute resolved rate (all tests pass), passed rate (fraction of F2P tests passed), and token I/O.

## Concrete Examples

**Example 1: Extracting a feature task from a Flask extension**

```
User: I have a Flask-Login repository with 50 passing tests. Extract a
feature-level coding task from the "remember me" functionality.

Approach:
1. Run all tests under sys.settrace, building the dependency graph.
2. Identify F2P tests: test_remember_me_cookie_set, test_remember_me_expiry,
   test_remember_me_refresh — these test the remember-me feature.
3. Classify imports in those test files:
   - login_user() → target (function under test)
   - create_app() → utility (test fixture)
4. BFS from login_user() through the dependency graph:
   - login_user → _set_remember_cookie → _cookie_encode → (extracted)
   - login_user → _update_session → (remained, touched by P2P tests)
5. Generate patch removing _set_remember_cookie, _cookie_encode, and
   related code in login_manager.py and utils.py (~120 lines).
6. Verify: 47 P2P tests pass, 3 F2P tests fail. Task is valid.

Output (task description):
  Repository: flask-login (patched)
  Objective: Implement the "remember me" cookie functionality.
  Files to modify: flask_login/login_manager.py, flask_login/utils.py
  Tests to pass: test_remember_me_cookie_set, test_remember_me_expiry,
                 test_remember_me_refresh
  Constraint: All 47 existing tests must continue to pass.
```

**Example 2: Evaluating an agent on a multi-file task**

```
User: Evaluate Claude on a FeatureBench task for the "pagination" feature
extracted from a Django REST framework fork.

Approach:
1. Load the pre-built Docker image for this task.
2. Present the agent with:
   - The patched repository (pagination code removed from 4 files)
   - Task description: "Implement cursor-based and page-number pagination"
   - P2P test file showing existing serializer/view contracts
3. Run the agent. It produces edits to:
   - rest_framework/pagination.py (new file, 280 lines)
   - rest_framework/generics.py (adds pagination mixin, 35 lines)
   - rest_framework/settings.py (adds DEFAULT_PAGINATION_CLASS, 3 lines)
   - rest_framework/views.py (wires pagination into APIView, 12 lines)
4. Execute evaluation:
   - F2P tests: 8/12 pass (agent missed cursor pagination edge case)
   - P2P tests: 45/45 pass
   - Resolved: No (not all F2P tests pass)
   - Passed rate: 66.7%

Output:
  Task: pagination (L1)
  Resolved: false
  Passed rate: 0.667
  P2P integrity: true
  Tokens: 42,300 in / 8,100 out
  Failure analysis: AttributeError in CursorPagination.get_next_link()
    — agent hallucinated cursor encoding interface instead of reading
    the existing encode_cursor() utility.
```

**Example 3: Building a benchmark suite from scratch**

```
User: Generate a 20-task benchmark from the "httpx" Python library.

Approach:
1. Clone httpx, install via Docker, run pytest --collect-only → 847 tests.
2. Execute all tests: 831 pass (P2P pool), 16 fail (skip these).
3. Build dependency graph via tracing: 2,340 function nodes, 5,120 edges.
4. Group tests by top-level target objects (LLM classification):
   - HTTP/2 stream handling (12 tests, 6 target functions)
   - Redirect following (8 tests, 4 target functions)
   - Authentication flows (14 tests, 7 target functions)
   - ... (identify ~30 candidate feature clusters)
5. For each cluster, run BFS extraction + post-verification.
   - 22 clusters produce valid F2P/P2P splits.
   - 2 clusters fail verification (shared state with P2P tests).
6. Select 20 tasks spanning 3k–5k extracted lines each.
7. Package into Docker images with task descriptions.

Output:
  Benchmark: httpx-featurebench-v1
  Tasks: 20
  Repositories: 1 (httpx)
  Environments: 20 Docker images
  Avg extracted lines: 3,840
  Avg F2P tests per task: 9.2
  Avg P2P tests per task: 412
  Verification: all 20 tasks pass F2P-fail / P2P-pass invariant
```

## Best Practices

- **Do:** Always run post-verification (P2P pass + F2P fail) before accepting a task. A task that breaks P2P tests is useless — it means you removed shared infrastructure, not a separable feature.
- **Do:** Use the LLM classifier for import classification rather than heuristics. The paper reports 91.7% accuracy; manual rules on naming conventions perform significantly worse on real codebases.
- **Do:** Bound extracted code to 3,000–5,000 lines. Below 3,000, tasks are trivially small. Above 5,000, agents cannot realistically complete them within token budgets.
- **Do:** Implement cheating detection by scanning agent logs for access to installed package source (e.g., `/usr/local/lib/python*/`). Agents that read the answer from installed packages invalidate results.
- **Avoid:** Extracting features that share mutable global state with P2P tests — these produce flaky verification results and unreliable tasks.
- **Avoid:** Using non-execution-based evaluation (e.g., code similarity, AST diff). The paper demonstrates that only test execution reliably measures functional correctness for multi-file features.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| P2P tests fail after extraction | Removed code is shared by non-target features | Expand the "remained" set by including nodes touched by failing P2P tests, then re-extract |
| F2P tests pass after extraction | Extraction missed the actual feature code | Re-run LLM classification on imports; check for dynamic dispatch or monkey-patching that the static tracer missed |
| Dependency graph is too shallow | Library uses heavy metaprogramming or decorators | Supplement `sys.settrace` with AST-based import analysis to capture static dependencies |
| Docker build fails | Missing system-level dependencies | Add `apt-get` commands to the config; check for C extensions requiring build tools |
| Agent produces correct code but tests still fail | Test relies on specific file paths or import structure | Ensure task description specifies expected module paths; add path hints to the environment |
| Tracer captures too many nodes (>50k) | Large repository with deep call stacks | Filter out standard library and third-party calls; trace only project-internal modules |

## Limitations

- **Python-centric.** The dynamic tracing approach (`sys.settrace`) is Python-specific. Adapting to compiled languages requires different instrumentation (e.g., LLVM-based tracing for C/C++, JVM agents for Java).
- **Pytest dependency.** The pipeline assumes pytest as the test runner. Projects using unittest, nose, or custom frameworks need adapter logic.
- **Metaprogramming blind spots.** Dynamic dispatch, `__getattr__` overrides, and decorator-heavy patterns can create dependencies invisible to the call tracer, leading to incomplete extraction.
- **Manual environment setup.** The Docker config step (~3 minutes per repo) is manual. Complex build systems (Bazel, multi-stage builds) take longer.
- **LLM classifier accuracy.** At 91.7% accuracy, roughly 1 in 12 import classifications is wrong. This propagates to incorrect BFS entry points and potentially invalid tasks. Always run post-verification.
- **Single-language features only.** Cannot extract features that span multiple languages (e.g., Python backend + JavaScript frontend).

## Reference

**Paper:** [FeatureBench: Benchmarking Agentic Coding for Complex Feature Development](https://arxiv.org/abs/2602.10975v1) (ICLR 2026)
**Key insight:** Unit tests encode feature boundaries; tracing test execution through a dependency graph and applying F2P/P2P invariants automatically extracts realistic, multi-file feature development tasks with built-in correctness verification.