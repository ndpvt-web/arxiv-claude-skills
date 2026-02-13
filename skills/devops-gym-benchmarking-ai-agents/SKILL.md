---
name: "devops-gym-benchmarking-ai-agents"
description: "Apply the DevOps-Gym methodology to systematically tackle full-cycle DevOps tasks: build/configuration repair, runtime monitoring and anomaly detection, issue resolving via code patches, and regression test generation for Java and Go projects. Trigger phrases: 'fix this build failure', 'diagnose this runtime anomaly', 'generate regression tests for this bug', 'resolve this issue in Java/Go', 'debug this CI pipeline', 'monitor this running service for anomalies'."
---

# DevOps-Gym: Full-Cycle DevOps Agent Methodology

This skill equips Claude to handle the four core DevOps workflow categories identified in the DevOps-Gym benchmark (arXiv:2601.20882): **build and configuration**, **runtime monitoring**, **issue resolving**, and **test generation**. Rather than treating these as isolated coding tasks, the methodology emphasizes sequential decision-making across the full DevOps cycle -- analyzing large-scale Java/Go projects, understanding dynamic runtime behavior, leveraging domain-specific build and monitoring tools, and producing verifiable outputs (patches, diagnostics, tests) that are validated against real execution.

## When to Use

- When the user asks to fix a failing build, resolve dependency conflicts, or repair Maven/Gradle/Go module configuration
- When the user needs to diagnose a runtime anomaly (memory leak, CPU saturation, disk exhaustion, I/O bottleneck) in a running service
- When the user provides a bug description and asks for a minimal code patch in a Java or Go repository
- When the user asks to generate regression tests that reproduce a described bug, without access to the fix
- When the user wants to set up or migrate build toolchains (framework migration, plugin integration, version upgrades)
- When the user needs to debug CI/CD pipeline failures from build logs or error output
- When the user asks to evaluate or benchmark agent performance on DevOps-style tasks

## Key Technique

**The DevOps-Gym benchmark reveals that AI agents fail at DevOps tasks for three specific, addressable reasons**: (1) toolchain knowledge gaps -- agents don't understand the internal mechanics of Maven, Gradle, goreleaser, and Go modules well enough to fix configuration issues; (2) premature convergence -- agents stop after partial fixes instead of running iterative fix-run-verify loops; (3) cross-language capability gaps -- performance drops dramatically from Python to Java/Go due to compiled-language complexity (multi-stage compilation, linking, type systems). The benchmark shows Claude Code achieving 58% on build tasks but only 14-24% on monitoring and test generation, meaning these harder categories require deliberate strategies.

**The actionable methodology is structured verification**: for every DevOps category, the agent must produce output in a specific format (diff patch, structured diagnostic, test file) and verify it against execution. Build patches must compile and pass tests. Monitoring diagnoses must cite quantitative evidence (memory growth rates, process IDs). Issue patches must pass fail-to-pass tests without regressions. Generated tests must fail on buggy code and pass on patched code. The key insight is that agents that enforce iterative fix-run-verify loops outperform those that attempt single-shot solutions.

**For monitoring tasks specifically**, the benchmark identifies four failure modes that agents must avoid: inadequate monitoring methodology (37% of failures) -- solved by systematic multi-tool sampling over time; premature conclusions (26%) -- solved by requiring temporal evidence across multiple observation windows; insufficient temporal granularity (11%) -- solved by collecting data at regular intervals; and interpretation failures (26%) -- solved by comparing against baselines before diagnosing anomalies.

## Step-by-Step Workflow

### 1. Classify the DevOps Task Category
Determine which of the four categories applies: **build/config** (compilation failures, dependency errors, toolchain issues), **monitoring** (runtime anomalies, performance degradation), **issue resolving** (bug description to code patch), or **test generation** (bug description to regression test). This determines the tool set, output format, and verification strategy.

### 2. Gather Context from the Repository and Environment
For build tasks: read `pom.xml`, `build.gradle`, `go.mod`, CI config files, and recent build logs. For monitoring: use `top`, `free -m`, `ps aux`, `netstat`, `iostat` to capture baseline system state. For issue/test tasks: read the bug description, identify the affected module, and map the relevant source files and existing test suites.

### 3. Identify the Root Cause Using Domain-Specific Analysis
For build failures: parse error messages to distinguish dependency conflicts, version mismatches, missing plugins, and toolchain incompatibilities. For monitoring: collect system metrics at 3+ time intervals to establish trends (e.g., monotonically increasing memory = leak, sustained >90% CPU = saturation). For issue resolving: trace the bug description to specific code paths using grep, call graph analysis, and test failure output.

### 4. Formulate a Minimal, Targeted Fix
Produce the smallest change that addresses the root cause. For build config: edit only the specific dependency version, plugin configuration, or build script line. For code patches: generate a unified diff that touches only the buggy logic. Avoid refactoring or unrelated improvements -- the DevOps-Gym evaluation penalizes patches that introduce new test failures.

### 5. Execute the Iterative Fix-Run-Verify Loop
This is the critical differentiator. After applying each change: (a) run the build/test/monitoring check, (b) analyze the output for remaining failures, (c) apply incremental fixes. Do NOT stop after the first attempt. The benchmark shows agents that iterate achieve significantly better results than single-shot approaches.

### 6. Validate Output Against Execution Criteria
- **Build tasks**: Run `mvn clean install`, `gradle build`, or `go build` -- must complete with exit code 0, and any associated test suite must pass.
- **Monitoring tasks**: Write a structured one-line diagnosis specifying anomaly type and quantitative evidence. Compare observations against healthy baselines.
- **Issue patches**: Run the fail-to-pass test(s) -- they must now pass. Run the full test suite -- no new failures.
- **Test generation**: Run generated tests against the buggy code (must fail) and against patched code (must pass).

### 7. Handle Multi-Step Reasoning for Compiled Languages
Java and Go introduce compilation stages absent in Python. For Java: check that all imports resolve, generics are type-safe, and the build tool's dependency resolution is consistent. For Go: verify module paths in `go.mod`, ensure interface implementations are complete, and check that cross-package references compile.

### 8. Document the Change with Rationale
Produce output in the expected format: unified diff for patches, structured diagnostic for monitoring, or test file for test generation. Include a brief explanation of what was wrong and why the fix is correct, so the user can verify the reasoning.

## Concrete Examples

**Example 1: Build Configuration Repair (Maven Dependency Conflict)**

User: "My Java project fails to build with `NoSuchMethodError` at runtime after upgrading Spring Boot to 3.2. The build itself succeeds but tests fail."

Approach:
1. Read `pom.xml` to identify Spring Boot version and all transitive dependencies
2. Run `mvn dependency:tree` to find conflicting library versions pulled in by different dependencies
3. Identify that an older `jackson-databind` is being pulled in transitively, conflicting with Spring Boot 3.2's expected version
4. Add an explicit `<dependencyManagement>` entry pinning `jackson-databind` to the version compatible with Spring Boot 3.2
5. Run `mvn clean test` to verify all tests pass
6. If tests still fail, check for additional transitive conflicts and repeat

Output:
```xml
<!-- pom.xml patch -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>2.16.1</version>
    </dependency>
  </dependencies>
</dependencyManagement>
```
Verification: `mvn clean test` exits 0, all 247 tests pass.

**Example 2: Runtime Monitoring -- Memory Leak Detection**

User: "Our Go service is getting OOM-killed in production after ~2 hours. Diagnose what's happening."

Approach:
1. Run `free -m` at t=0 to establish memory baseline
2. Run `ps aux --sort=-rss | head -20` to identify the top memory consumers
3. Wait 30 seconds, repeat measurements 3 more times to establish a trend
4. Calculate memory growth rate: if the Go service's RSS grows by ~15MB every 30 seconds with no corresponding increase in request volume, this indicates a memory leak
5. Check for goroutine leaks with `curl localhost:6060/debug/pprof/goroutine?debug=1` if pprof is exposed
6. Write structured diagnosis with quantitative evidence

Output:
```
memory_leak: process=myservice pid=4821 rss_growth=30MB/min baseline=512MB current=1847MB goroutine_count=increasing(2340->8901)
```

**Example 3: Regression Test Generation from Bug Description**

User: "Bug report says: 'When a Go HTTP handler receives a request with a Content-Length header of 0 but a non-empty body, the server panics with nil pointer dereference in parseBody().' Generate a regression test."

Approach:
1. Read the bug description and locate `parseBody()` in the source
2. Examine the function signature, understand what `nil` value causes the panic (likely an unchecked `req.Body` when Content-Length is 0)
3. Write a test that constructs an `http.Request` with `Content-Length: 0` and a non-empty `bytes.Buffer` body
4. Assert that calling the handler does NOT panic (using `recover()` or `httptest`)
5. Run the test against the current buggy code to confirm it triggers the panic
6. Verify the test framework matches the project's conventions (`testing` package, file naming)

Output:
```go
func TestParseBody_ZeroContentLengthWithBody(t *testing.T) {
    body := bytes.NewBufferString(`{"key":"value"}`)
    req := httptest.NewRequest("POST", "/api/data", body)
    req.Header.Set("Content-Length", "0")

    recorder := httptest.NewRecorder()

    defer func() {
        if r := recover(); r != nil {
            t.Fatalf("parseBody panicked with Content-Length 0 and non-empty body: %v", r)
        }
    }()

    handler.ServeHTTP(recorder, req)

    if recorder.Code == http.StatusInternalServerError {
        t.Error("expected successful parsing, got 500")
    }
}
```
Verification: Test fails on buggy code (panic), passes after nil-check fix in `parseBody()`.

## Best Practices

- **Do:** Always run the full build/test suite after applying a fix, not just the specific failing test. DevOps-Gym evaluates that no regressions are introduced.
- **Do:** Collect monitoring data at multiple time points (minimum 3 observations over 60+ seconds) before diagnosing. Single-point observations lead to the "premature conclusion" failure mode that causes 26% of monitoring errors.
- **Do:** For Java projects, use `mvn dependency:tree` or `gradle dependencies` to understand transitive dependency graphs before editing build files. 37% of build failures stem from domain-specific knowledge gaps about build tool internals.
- **Do:** When generating tests, write them in the project's existing test framework and conventions -- match import style, assertion libraries, and file placement.
- **Avoid:** Single-shot fixes without verification. The iterative fix-run-verify loop is the single biggest differentiator in DevOps-Gym performance.
- **Avoid:** Treating Java/Go like Python. Compiled languages require attention to type resolution, import paths, module boundaries, and multi-stage compilation that Python does not have. Performance drops 40-50% when agents ignore these differences.

## Error Handling

**Build tool not found or wrong version:** Check `which mvn`, `java -version`, `go version` first. Install or configure the correct toolchain before attempting fixes.

**Monitoring context exhaustion:** System monitoring can generate enormous output. Limit `top` and `ps` to targeted queries (specific PIDs, specific metrics). Avoid dumping full system state repeatedly -- summarize trends instead of storing raw output.

**Flaky tests during verification:** If a test passes/fails inconsistently, run it 3 times. If it's flaky independent of your change, note this to the user and focus on the fail-to-pass tests specific to the bug.

**Patch applies but introduces new failures:** Revert, re-read the failing tests to understand what invariant was violated, and produce a more targeted fix. Never submit a patch that trades one failure for another.

**Go module resolution failures:** Run `go mod tidy` after any dependency change. Check that `go.sum` is updated. For vendored projects, run `go mod vendor` as well.

## Limitations

- **Monitoring tasks require a running system.** If the user provides only source code without a running container or process, you cannot perform runtime anomaly detection -- shift to static analysis instead.
- **Java/Go performance gap is real.** Current AI agents (including Claude) perform significantly worse on Java and Go than on Python for issue resolving and test generation. Be explicit about confidence levels when working in these languages.
- **End-to-end pipeline completion is unsolved.** The DevOps-Gym benchmark shows 0% success rate on completing all four stages (build, monitor, resolve, test) for a single task. Treat each stage independently and verify outputs at each boundary.
- **Synthetic monitoring tasks may not match production complexity.** Real-world anomalies often involve multiple interacting issues. The structured approach here handles single-anomaly cases well but may need adaptation for compound failures.
- **Build tool internals are a known weakness.** Maven plugin resolution, Gradle build script Groovy/Kotlin DSLs, and goreleaser configuration have deep domain knowledge requirements that may exceed training data coverage.

## Reference

[DevOps-Gym: Benchmarking AI Agents in Software DevOps Cycle](https://arxiv.org/abs/2601.20882v1) -- Tang et al., 2026. Focus on Section 3 (task definitions and evaluation metrics), Table 1 (agent performance by category), and Section 5 (error analysis and failure modes) for the specific strategies that differentiate successful from unsuccessful agent behaviors.