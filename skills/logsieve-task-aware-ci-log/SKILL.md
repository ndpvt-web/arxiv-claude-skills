---
name: "logsieve-task-aware-ci-log"
description: "Reduce verbose CI/CD build logs before LLM analysis using RCA-aware semantic filtering. Removes boilerplate lines (dependency downloads, progress bars, timestamps, env setup) while preserving diagnostic content (error traces, test failures, compiler errors). Use when: 'analyze this CI log', 'why did my build fail', 'reduce this GitHub Actions log', 'summarize build failure from logs', 'filter noisy CI output', 'diagnose this workflow run'."
---

# LogSieve: Task-Aware CI Log Reduction for LLM-Based Analysis

This skill enables Claude to apply LogSieve, a semantics-preserving log reduction technique that filters low-information lines from CI/CD build logs before performing root-cause analysis. Instead of feeding an entire verbose CI log (often thousands of lines of dependency downloads, progress indicators, and environment setup) directly into analysis, LogSieve first classifies each line as RCA-relevant or RCA-irrelevant, removes the noise, and then reasons over the reduced log. This achieves ~42% line reduction and ~40% token reduction while preserving 93% semantic fidelity with the original analysis -- meaning you get the same diagnostic quality at significantly lower cost and latency.

## When to Use

- When a user pastes or points to a CI build log (GitHub Actions, GitLab CI, CircleCI, Jenkins) and asks "why did this fail?"
- When analyzing long workflow run logs that exceed context limits or are expensive to process in full
- When a user wants to extract only the diagnostically relevant portions of a build log
- When triaging multiple failed CI runs and needing quick root-cause categorization across all of them
- When building CI/CD tooling that pipes log output into LLM-based analysis and needs a pre-processing step
- When a user asks to "clean up" or "filter" CI output before debugging

## Key Technique

**RCA-Aware Semantic Filtering**: Unlike structural compression (e.g., LogZip, which collapses repeated patterns) or naive approaches (random line removal), LogSieve classifies each log line by its relevance to root-cause analysis (RCA). Lines containing diagnostic evidence -- compiler errors, stack traces, test failure summaries, exception messages, task-level failure indicators -- are retained. Lines serving routine functions -- dependency download progress, timestamp-only entries, environment variable echoes, decorative separators, package loading indicators -- are removed. This binary classification (RCA-relevant=1, RCA-irrelevant=0) was validated by human annotators achieving Cohen's kappa = 0.80 across 14,646 lines from 20 open-source Android projects.

**Why this beats alternatives**: LogZip achieves only 0.70 cosine similarity on downstream LLM explanations (vs. LogSieve's 0.93) because structural compression destroys diagnostic context. Random line removal at the same reduction ratio hits 0.90 cosine similarity but only 70% exact-match on failure categorization (vs. LogSieve's 80%). The key insight is that CI logs have a bimodal information distribution -- most lines are noise, and the signal lines cluster around failure points. Filtering by semantic relevance rather than structure exploits this distribution.

**Automation at scale**: An embedding-based classifier (TF-IDF or BERT features fed into Logistic Regression or SVM) achieves 97% accuracy at identifying RCA-relevant lines, making this approach viable as an automated pre-inference pipeline stage rather than requiring manual annotation.

## Step-by-Step Workflow

1. **Ingest the raw CI log**: Accept the full build log as input. Identify the CI system (GitHub Actions, GitLab CI, Jenkins, etc.) from log formatting conventions -- this informs which boilerplate patterns to expect.

2. **Segment the log into logical phases**: Split the log at CI step/stage boundaries. GitHub Actions logs use `##[group]` / `##[endgroup]` markers and step headers like `Run actions/checkout@v4`. Jenkins uses stage markers. Identify setup phases, build phases, test phases, and teardown phases.

3. **Classify each line as RCA-relevant or RCA-irrelevant** using these heuristics:

   **Remove (RCA-irrelevant) -- lines matching these patterns:**
   - Dependency download/resolution lines (`Downloading`, `Resolving`, `Fetching`, `Installing package`, progress percentages)
   - Progress indicators (`[ ] 3%`, `====>`, spinner characters, repeated dots)
   - Timestamp-only lines or lines that are purely timestamps with routine status (`2024-01-15T10:30:00Z INFO: Starting...`)
   - Environment variable echoes and shell setup (`export JAVA_HOME=`, `Setting up JDK`, `PATH=`)
   - Decorative separators (lines of `=`, `-`, `*`, or blank lines)
   - Cache hit/miss status for routine operations (`Cache restored from key:`, `Post job cleanup`)
   - License acceptance and SDK component installation boilerplate
   - Git checkout verbose output (`Fetching submodules`, `Checking out ref`)

   **Retain (RCA-relevant) -- lines matching these patterns:**
   - Lines containing `error`, `Error`, `ERROR`, `FAILURE`, `FAILED`, `fatal`
   - Stack traces (lines starting with `at `, `Caused by:`, `Exception in thread`)
   - Compiler diagnostic output (`src/main/java/Foo.java:42: error:`)
   - Test result summaries (`Tests run:`, `FAILED`, `X tests failed`, assertion errors)
   - Task execution failures (`Execution failed for task`, `Task :app:compileDebugJava FAILED`)
   - Build tool exit codes and summary lines (`BUILD FAILED`, `Process finished with exit code 1`)
   - Warning lines that co-occur near errors (retain warnings within 5 lines of an error)
   - Gradle/Maven task names with failure status
   - Lint, static analysis, or code quality failure reports

4. **Handle ambiguous lines with proximity weighting**: Lines within a 3-5 line window of a confirmed RCA-relevant line should be retained even if they don't match removal patterns. Context around errors is often diagnostic (e.g., the command that produced an error, the file being processed).

5. **Preserve structural anchors**: Always retain CI step headers, phase transitions, and the final summary/exit status lines regardless of classification. These provide navigational context for the reduced log.

6. **Emit the reduced log**: Output the filtered log preserving original line ordering. Optionally annotate removed sections with a single placeholder line like `[... N low-information lines removed ...]` to indicate where content was stripped, preserving the reader's sense of log structure.

7. **Perform downstream RCA on the reduced log**: With the filtered log, answer the user's question -- typically "why did this build fail?" Provide: (a) a root-cause explanation in 2-5 sentences, (b) a failure category (compilation error, test failure, dependency resolution, configuration issue, flaky test, infrastructure/timeout, etc.), and (c) a suggested fix if possible.

8. **Validate reduction quality**: If the reduced log seems insufficient for diagnosis (e.g., no error lines were found after filtering), fall back to scanning the full log. Some failures manifest only through absence (e.g., a step that never ran, a timeout with no error message).

9. **Report reduction metrics**: Tell the user how many lines/tokens were in the original vs. reduced log, giving them confidence in the analysis scope.

## Concrete Examples

**Example 1: GitHub Actions Android Build Failure**

User: "Here's my GitHub Actions log from a failed Android build. Why did it fail?" (pastes 800-line log)

Approach:
1. Segment log into phases: checkout (120 lines), setup-java (80 lines), gradle-build (450 lines), post-cleanup (150 lines)
2. Filter: Remove dependency download lines from gradle-build (230 lines of `Downloading https://repo.maven.apache.org/...`), setup-java SDK installation boilerplate (65 lines), checkout git operations (100 lines), post-cleanup cache upload (140 lines)
3. Retain: 5 lines around `Execution failed for task :app:compileDebugKotlin`, the 12-line compiler error block showing unresolved references, the `BUILD FAILED in 2m 34s` summary, and step headers

Output:
```
Reduced log: 800 → 143 lines (82% reduction), 24,000 → 4,300 tokens (82% reduction)

Root cause: Compilation failure in :app:compileDebugKotlin. The class
`com.example.data.UserRepository` references `UserDao.getActiveUsers()` which
does not exist. This was likely introduced by a recent refactor of the DAO layer.

Category: Compilation error

Suggested fix: Add the `getActiveUsers()` method to `UserDao.kt`, or update
`UserRepository.kt` to use the renamed method.
```

**Example 2: Flaky Test Failure in CI**

User: "My CI keeps failing intermittently. Here's the latest log -- can you figure out what's flaky?" (provides 1,200-line log)

Approach:
1. Filter setup/teardown boilerplate, dependency resolution, and environment configuration (~600 lines removed)
2. Retain all test execution output, focusing on failure blocks and timing information
3. Identify that the retained lines show a `SocketTimeoutException` in `NetworkIntegrationTest.testApiCall` with a 5-second timeout

Output:
```
Reduced log: 1,200 → 380 lines (68% reduction)

Root cause: `NetworkIntegrationTest.testApiCall` fails with
java.net.SocketTimeoutException after 5000ms. This test makes a real HTTP call
to an external API endpoint, making it inherently flaky in CI environments with
variable network latency.

Category: Flaky test (network dependency)

Suggested fix: Mock the HTTP client in this test, or increase the timeout and
add a @RetryOnFailure annotation. For a proper fix, refactor to use a local
test server (e.g., MockWebServer from OkHttp).
```

**Example 3: Bulk Triage of Multiple Failed Runs**

User: "I have 5 failed workflow runs. Can you categorize what's going wrong?" (provides 5 logs)

Approach:
1. Apply LogSieve reduction to each log independently
2. Extract the failure category and one-line root cause from each reduced log
3. Present a summary table

Output:
```
| Run   | Lines Reduced | Category           | Root Cause                              |
|-------|--------------|--------------------|-----------------------------------------|
| #412  | 74%          | Compilation error  | Missing import after merge conflict     |
| #413  | 68%          | Test failure       | Assertion in UserServiceTest.testCreate  |
| #414  | 81%          | Dependency error   | com.example:lib:2.0.0 not found in repo |
| #415  | 74%          | Compilation error  | Same as #412 (not fixed on this branch) |
| #416  | 45%          | Infrastructure     | Runner disk space exhausted              |

Pattern: Runs #412 and #415 share the same root cause on the same branch.
Run #416 is an infrastructure issue unrelated to code changes.
```

## Best Practices

- **Do** preserve a window of 3-5 lines around every detected error. The surrounding context (which command ran, which file was being processed) is often essential for diagnosis.
- **Do** retain all CI step/phase headers even when the step contents are entirely filtered. Headers like `Run: ./gradlew assembleDebug` tell you what was attempted.
- **Do** insert placeholder markers (`[... 230 lines of dependency downloads removed ...]`) when removing large contiguous blocks so users understand the log's original structure.
- **Do** report the reduction ratio to the user so they can judge whether the filtering was too aggressive.
- **Avoid** removing lines just because they look like "info" level logging. Many diagnostic messages use INFO level, especially in Gradle and Maven output. Only remove lines matching known boilerplate patterns.
- **Avoid** filtering logs from steps that actually failed. If a CI step has a non-zero exit code, be conservative and retain more of its output. Apply aggressive filtering only to steps that succeeded.

## Error Handling

- **No error lines found after filtering**: The failure may be a timeout, an infrastructure issue, or a silent failure (exit code != 0 but no error output). Fall back to examining the full log's final 50 lines and all step exit codes.
- **Log exceeds context window even after filtering**: Apply a second pass prioritizing lines closest to the failure point (typically the last failed step). Summarize earlier phases as single-line descriptions.
- **Unrecognized CI system**: If the log format is unfamiliar, skip phase segmentation and apply line-level classification only. The heuristic patterns (error keywords, stack traces, progress indicators) are largely CI-system-agnostic.
- **Mixed stdout/stderr interleaving**: Some CI systems interleave streams. If the log appears jumbled, look for explicit stream markers (`##[error]` in GitHub Actions) rather than relying on line ordering.

## Limitations

- **Structured system logs are out of scope**: LogSieve targets unstructured, verbose CI build logs. For structured logs (JSON-formatted application logs, syslog), use dedicated log analysis tools instead.
- **Novel failure modes may be missed**: The heuristic classification favors known error patterns. A completely novel failure type that doesn't produce standard error keywords may have its diagnostic lines incorrectly filtered. Always verify against the full log when the reduced log seems inconclusive.
- **Language/tool-specific patterns**: The filtering rules work best for JVM-based builds (Gradle, Maven) and common CI toolchains. Logs from highly custom build systems may have different boilerplate patterns requiring adjustment.
- **80% exact-match on categorization (not 100%)**: The 20% gap means that roughly 1 in 5 failure categorizations from the reduced log may differ from what the full log would produce. For critical production debugging, use the reduced log for triage but verify against the full log for the final diagnosis.
- **Binary classification is lossy**: Some lines are genuinely ambiguous (e.g., SDK license acceptance that is sometimes the root cause). The technique trades perfect recall for practical efficiency.

## Reference

**Paper**: [LogSieve: Task-Aware CI Log Reduction for Sustainable LLM-Based Analysis](https://arxiv.org/abs/2601.20148v1) (Barnes, Ghaleb, Hassan -- MSR'26). Key sections: the line classification taxonomy and annotation guidelines (Section 3), the embedding-based automation approach achieving 97% accuracy (Section 5), and the downstream evaluation showing 0.93 cosine similarity with 42% reduction (Section 4).