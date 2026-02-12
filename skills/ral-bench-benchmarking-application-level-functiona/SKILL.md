---
name: "ral-bench-benchmarking-application-level-functiona"
description: "Generate and evaluate complete multi-file application repositories with both functional correctness and non-functional quality (maintainability, security, robustness, efficiency, resource usage). Use when: 'generate a complete application from requirements', 'build a multi-file project', 'evaluate code quality of a repository', 'check application-level correctness', 'assess non-functional quality attributes', 'scaffold a production-grade project with tests'."
---

# Application-Level Code Generation with Functional and Non-Functional Quality Assurance

This skill enables Claude to generate complete, runnable multi-file repositories from natural-language requirements while systematically ensuring both functional correctness and non-functional quality attributes. Based on the RAL-Bench framework (Pan et al., 2026), which revealed that no LLM exceeds 45% functional pass rate on application-level tasks, this skill applies the paper's evaluation methodology in reverse: using its quality dimensions as generation constraints to produce code that passes the tests real applications must survive. The core insight is that application-level code generation is a software engineering process requiring cross-module alignment, dependency management, and end-to-end executability — not just function-level correctness.

## When to Use

- When the user asks to generate a complete application, CLI tool, or multi-file project from a description or set of requirements
- When the user wants to scaffold a production-grade repository with proper structure, dependencies, and test coverage
- When the user asks to evaluate or audit an existing codebase for both functional correctness and quality attributes (maintainability, security, robustness, efficiency)
- When the user requests a code review that goes beyond style to assess cross-module consistency and runtime behavior
- When the user wants to add system-level tests that verify end-to-end executability and non-functional properties
- When generating code that must handle edge cases, invalid inputs, and resource constraints as first-class concerns

## Key Technique

RAL-Bench identifies three dominant failure modes in LLM-generated applications: requirement-implementation mismatch (51%), non-functional quality failures (31.8%), and executability/dependency failures (17.2%). The critical finding is that once code is executable, the bottleneck shifts to meeting the full set of behavioral contracts across modules and sustaining reliable runtime behavior. Existing repair strategies (self-repair, environment repair, planning-driven generation) all fail because they apply isolated corrections rather than addressing cross-module consistency.

The benchmark evaluates five non-functional quality dimensions weighted by the Analytic Hierarchy Process (AHP): **Maintainability** (weight 0.36) measured by Static Maintainability Index; **Security** (0.24) measured by static analysis findings; **Robustness** (0.16) measured by edge-case test pass rate; **Efficiency** (0.12) measured by execution time relative to reference; and **Resource Usage** (0.12) measured by memory and CPU utilization. These weights reflect real-world priorities: maintainability dominates because unmaintainable code accumulates cost fastest, followed by security because vulnerabilities carry outsized risk.

The actionable methodology is: (1) produce explicit functional requirement specifications before writing code, (2) define module boundaries and API surfaces up front, (3) generate code module-by-module with cross-module alignment checks, and (4) treat non-functional attributes as hard constraints during generation — bounding execution time, adding input validation at module boundaries, and structuring code for low cyclomatic complexity.

## Step-by-Step Workflow

1. **Extract and formalize requirements.** Parse the user's description into explicit functional requirements: expected inputs/outputs, core behaviors, error-handling expectations, and edge cases. List each requirement as a testable assertion. If the description is vague, ask clarifying questions before proceeding.

2. **Define the API surface and module boundaries.** Identify the top-level modules/packages the application needs. Specify the public interfaces (function signatures, class APIs, CLI commands, HTTP endpoints) each module must expose. This prevents cross-module mismatch, which causes 51% of application-level failures.

3. **Design the dependency and configuration layer.** List all external dependencies with version constraints. Define configuration files (pyproject.toml, package.json, Cargo.toml, etc.) and environment requirements. Ensure the project can be installed and executed from a clean environment.

4. **Generate code module-by-module with interface contracts.** Implement each module while respecting the API surface defined in Step 2. After each module, verify it imports correctly and its public interface matches the specification. Do not proceed to the next module until the current one is internally consistent.

5. **Apply non-functional quality constraints during generation.** For each module:
   - **Maintainability**: Keep cyclomatic complexity low, use descriptive names, limit function length to ~30 lines, maintain a Maintainability Index above 20.
   - **Security**: Validate all external inputs, avoid shell injection vectors, use parameterized queries, never hardcode secrets.
   - **Robustness**: Add explicit handling for invalid inputs, boundary values, and type errors at every public interface.
   - **Efficiency**: Avoid O(n^2) algorithms where O(n log n) exists, use streaming for large data, cache repeated computations.
   - **Resource Usage**: Bound memory allocations, close file handles and connections, use generators for large sequences.

6. **Write system-level tests covering functional requirements.** Create black-box tests that exercise the application end-to-end: invoke it as a user would (CLI commands, API calls, imports), verify outputs match requirement assertions, and test cross-module interactions — not just individual functions.

7. **Write non-functional quality tests.** Add tests for:
   - Robustness: invalid inputs, empty inputs, oversized inputs, malformed configuration
   - Efficiency: execution time within acceptable bounds for representative workloads
   - Security: attempted injection attacks, unauthorized access patterns

8. **Run the full test suite and perform cross-module alignment repair.** Execute all tests. When tests fail, diagnose whether the failure is a single-module bug or a cross-module mismatch. For mismatches, propagate fixes across all affected modules simultaneously rather than patching one module in isolation.

9. **Validate end-to-end executability.** Verify the project can be installed from scratch (clean virtual environment, fresh `npm install`, etc.), all entry points execute without import errors, and the application produces correct output for at least two representative use cases.

10. **Score and report quality.** Summarize the codebase against the five quality dimensions with a brief assessment for each. Flag any dimension scoring poorly and suggest targeted improvements.

## Concrete Examples

**Example 1: Generating a URL shortener service**

User: "Build me a URL shortener with a REST API, persistent storage, and analytics tracking."

Approach:
1. Extract requirements: POST /shorten accepts URL and returns short code; GET /:code redirects to original URL; GET /stats/:code returns click count, referrers, timestamps; data persists across restarts; invalid URLs rejected with 400.
2. Define modules: `app.py` (Flask/FastAPI entry), `storage.py` (SQLite persistence layer), `analytics.py` (click tracking), `validators.py` (URL validation).
3. Define API surface: `storage.create_short_url(url) -> code`, `storage.resolve(code) -> url`, `analytics.record_click(code, referrer, timestamp)`, `analytics.get_stats(code) -> dict`.
4. Generate each module, verifying imports resolve and interfaces match.
5. Apply constraints: validate URLs with urllib.parse (security), use parameterized SQL (security), keep functions under 25 lines (maintainability), handle missing codes with proper 404s (robustness), use connection pooling (efficiency).
6. Write system tests: test full POST-then-GET redirect cycle, test stats endpoint after multiple clicks, test invalid URL rejection, test nonexistent code handling.
7. Write non-functional tests: attempt SQL injection via URL parameter, submit 10,000 URLs and verify response time stays under 100ms each, submit empty and whitespace-only URLs.

Output structure:
```
url-shortener/
  pyproject.toml
  src/
    app.py           # FastAPI application with route handlers
    storage.py       # SQLite persistence with parameterized queries
    analytics.py     # Click tracking and stats aggregation
    validators.py    # URL validation and sanitization
  tests/
    test_functional.py      # End-to-end API tests
    test_robustness.py      # Edge cases and invalid inputs
    test_security.py        # Injection and abuse scenarios
```

**Example 2: Evaluating an existing repository**

User: "Review my Python CLI tool for code quality — it parses CSV files and generates reports."

Approach:
1. Read all source files to understand module structure and public interfaces.
2. Check functional coverage: Does the CLI handle all documented subcommands? Does each subcommand produce correct output for sample inputs?
3. Assess maintainability: Measure function lengths, nesting depth, naming clarity. Flag functions over 40 lines or cyclomatic complexity above 10.
4. Assess security: Check for path traversal in file arguments, unvalidated user input passed to format strings or eval, and unsafe deserialization.
5. Assess robustness: Does the CLI handle missing files, empty CSVs, malformed rows, non-UTF8 encoding? Are error messages actionable?
6. Assess efficiency: Are large CSVs streamed or loaded entirely into memory? Is there unnecessary recomputation?
7. Produce a scored report with specific file:line references for each finding.

Output:
```
## Quality Assessment (RAL-Bench Dimensions)

### Functional Correctness: 7/10
- `generate` subcommand works correctly for standard CSVs
- Missing: `--output-format json` documented in README but not implemented (src/cli.py:45)
- Edge case: crashes on CSV with zero data rows (src/parser.py:112)

### Maintainability (weight 0.36): 6/10
- src/reporter.py:generate_report() is 87 lines with 4 levels of nesting — extract helper functions
- Variable names `d`, `r`, `x` in src/parser.py:30-55 — use descriptive names

### Security (weight 0.24): 5/10
- src/cli.py:23 — file path from argv passed directly to open() without path traversal check
- src/reporter.py:71 — f-string with user data used in log message (potential log injection)

### Robustness (weight 0.16): 4/10
- No handling for non-UTF8 files (src/parser.py:10)
- No handling for CSV files with inconsistent column counts
- KeyboardInterrupt during generation leaves partial output file

### Efficiency (weight 0.12): 8/10
- CSV is streamed row-by-row — good
- Report generation caches intermediate aggregations — good

### Resource Usage (weight 0.12): 7/10
- File handles closed via context managers — good
- Large report strings built via concatenation rather than StringIO (src/reporter.py:50)
```

**Example 3: Scaffolding a project with quality-aware generation**

User: "Create a Python package that wraps the GitHub API for managing issues — support create, list, update, close, and search."

Approach:
1. Requirements: `create_issue(repo, title, body, labels)`, `list_issues(repo, state, labels)`, `update_issue(repo, number, **kwargs)`, `close_issue(repo, number)`, `search_issues(query)`. All require auth token. Return typed dataclass objects.
2. Modules: `client.py` (HTTP layer), `models.py` (dataclasses), `issues.py` (business logic), `auth.py` (token management), `exceptions.py` (custom errors).
3. Generate with constraints: token never logged or included in error messages (security), all HTTP errors mapped to specific exceptions (robustness), pagination handled transparently (functional correctness), requests use timeouts and retries (efficiency/resource usage).
4. System tests using responses library to mock GitHub API: test full CRUD cycle, test pagination across 3 pages, test auth failure produces clear error, test rate-limit handling.
5. Non-functional tests: attempt to search with 5000-character query, verify token not in any exception string representation, verify connection pool is bounded.

## Best Practices

**Do:**
- Define module boundaries and API surfaces before writing any implementation code — 51% of application-level failures stem from cross-module requirement-implementation mismatch
- Treat non-functional attributes as hard constraints during generation, not afterthoughts — add input validation, error handling, and resource bounds as you write each function
- Write system-level black-box tests that exercise the application the way a user would, not just unit tests on internal functions
- When a test fails, diagnose whether the root cause is local to one module or a cross-module interface mismatch, and fix accordingly by propagating changes across all affected modules
- Use the AHP weights (maintainability 0.36 > security 0.24 > robustness 0.16 > efficiency 0.12 = resource usage 0.12) to prioritize improvement effort when time is limited

**Avoid:**
- Generating the entire application in a single pass without checking cross-module consistency — this is the primary cause of integration failures
- Relying on self-repair by running generated tests against generated code — RAL-Bench showed this actually degrades functional performance because generated tests are unreliable oracles
- Treating non-functional quality as a compensation for functional failures — high maintainability scores cannot rescue code that doesn't meet its behavioral requirements
- Over-investing in planning-driven generation without corresponding implementation discipline — RAL-Bench found planning alone provides no functional improvement and degrades non-functional quality

## Error Handling

**Cross-module import failures:** When modules cannot import each other, the issue is usually an inconsistent API surface. Re-examine the interface contract defined in Step 2 and ensure all module public names match exactly.

**Dependency resolution failures:** If the project fails to install, verify that all imports correspond to declared dependencies in the configuration file. Run a static import scan and cross-reference with the dependency list.

**Test failures from requirement-implementation mismatch:** When tests fail on assertion values rather than exceptions, the code runs but produces wrong behavior. Trace the failing assertion back to the specific requirement, identify which module is responsible, and fix the logic — do not simply adjust the test.

**Non-functional degradation after functional fixes:** Fixing functional bugs can worsen maintainability (longer functions, deeper nesting) or efficiency (added workarounds). After functional fixes, re-evaluate the affected module against non-functional constraints and refactor if thresholds are exceeded.

**Flaky tests from resource/timing sensitivity:** Tests checking execution time or memory usage may vary across environments. Use relative thresholds (e.g., no more than 3x the reference baseline) rather than absolute values, and run timing tests multiple times.

## Limitations

- This approach adds overhead to generation: defining API surfaces, writing system tests, and checking cross-module alignment takes significant effort. For simple single-file scripts or throwaway prototypes, skip the full workflow.
- Non-functional quality measurement (especially efficiency and resource usage) is environment-dependent. Absolute scores are not portable across machines; use relative comparisons against a reference implementation.
- The five quality dimensions and AHP weights are derived from ISO/IEC 25010 and the RAL-Bench authors' expert judgment. Different domains may need different weights — safety-critical systems should weight robustness higher; high-traffic services should weight efficiency higher.
- Static analysis for maintainability and security catches structural issues but misses semantic problems (e.g., correct but misleading variable names, logic bombs). Manual review remains necessary for high-stakes code.
- The 45% functional pass rate ceiling observed in RAL-Bench applies to zero-shot generation. This skill's structured workflow should improve on that, but complex applications with many interacting requirements will still challenge even guided generation.

## Reference

**RAL-Bench: Benchmarking for Application-Level Functional Correctness and Non-Functional Quality Attributes** — Pan et al., 2026. [arXiv:2602.03462](https://arxiv.org/abs/2602.03462v1). Key takeaway: application-level code generation fails primarily from cross-module requirement-implementation mismatch (51%), and existing repair strategies (self-repair, environment repair, planning) do not help — structured requirement analysis and cross-module alignment are the path forward.