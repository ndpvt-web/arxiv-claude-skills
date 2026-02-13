---
name: "can-we-classify-flaky"
description: "Analyze test suites for flaky tests using LLM-based classification with context-augmented reasoning. Applies findings from Berndt et al. (2026) showing that test code alone is insufficient — the skill teaches Claude to gather surrounding project context (configs, dependencies, environment, production code) before classifying. Trigger phrases: 'find flaky tests', 'classify flaky tests', 'detect test flakiness', 'why is this test flaky', 'analyze test reliability', 'flaky test triage'"
---

# Flaky Test Classification with Context-Augmented LLM Analysis

This skill enables Claude to analyze test code for flakiness — the property where a test yields inconsistent pass/fail results on the same code revision. Based on empirical findings from Berndt et al. (2026), which demonstrated that LLMs performing zero-shot, few-shot, or chain-of-thought classification on isolated test code perform only marginally better than random guessing, this skill implements a **context-augmented approach**: gathering project artifacts (build configs, production code under test, environment setup, CI configuration) before making a flakiness judgment. The paper's key insight is that flakiness signals live outside the test method itself, so this skill systematically retrieves that missing context.

## When to Use

- When a user reports intermittent test failures and asks "is this test flaky?"
- When triaging a test suite to identify tests likely to produce non-deterministic results
- When a user pastes a test method and asks why it might be unreliable
- When reviewing a PR that adds new tests and checking for flakiness risk factors
- When a CI pipeline has inconsistent test results and the user wants to identify root causes
- When migrating or refactoring tests and assessing which ones need hardening against flakiness

## Key Technique

**Why test code alone fails.** Berndt et al. evaluated GPT-4o, GPT-4o-mini, and CodeLlama across zero-shot, few-shot, and chain-of-thought prompting on two established datasets (IDoFT and FlakeFlagger). Even the best prompt-model combination achieved results only marginally above random chance (MCC near zero). A manual investigation of 50 samples confirmed the root cause: isolated test methods lack the contextual signals needed to determine flakiness. A `Thread.sleep(1000)` in a test might be harmless or catastrophic depending on what it waits for — information that only exists in production code, CI config, or runtime environment.

**The six flakiness root-cause categories** identified in flaky test literature — and confirmed as undetectable from test code alone — are: (1) **async/timing dependencies** (waits, sleeps, timeouts whose adequacy depends on external systems), (2) **concurrency and shared state** (tests that modify shared resources without isolation), (3) **test-order dependencies** (tests that assume execution order or prior state), (4) **external service dependencies** (network calls, databases, APIs that may be unavailable), (5) **environment sensitivity** (file paths, OS-specific behavior, timezone/locale), and (6) **randomness and non-determinism** (unseeded RNGs, hash iteration order). Detecting most of these requires seeing what the test interacts with, not just the test itself.

**The context-augmented approach.** Instead of classifying from test code alone (which the paper proves ineffective), this skill implements what the paper recommends as future work: a retrieval-augmented strategy that gathers production code, configuration, and environment details before reasoning about flakiness. This transforms classification from a code-only task into a project-aware analysis.

## Step-by-Step Workflow

1. **Extract the test method(s) under analysis.** Read the full test file and isolate each test method, preserving class-level setup/teardown (`@Before`, `@After`, `setUp`, `tearDown`, `beforeEach`, etc.) and any shared fields or fixtures.

2. **Identify the production code under test.** Trace imports and method calls from the test to locate the actual classes/functions being tested. Read those source files to understand what the test exercises — this is the single most important context the paper found missing.

3. **Gather build and dependency configuration.** Read `pom.xml`, `build.gradle`, `package.json`, `requirements.txt`, or equivalent. Check for test framework versions, parallelism settings (`forkCount`, `parallel`, `--workers`), and timeout configurations that affect test execution.

4. **Check CI/environment configuration.** Read `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`, `docker-compose.yml`, or equivalent. Identify whether tests run in parallel, in containers, with specific environment variables, or against external services.

5. **Scan for the six flakiness root-cause categories.** For each test, systematically check:
   - **Timing:** `sleep`, `wait`, `setTimeout`, `Thread.sleep`, polling loops, `@Timeout` annotations
   - **Concurrency:** shared mutable state, static fields, singleton access, missing synchronization
   - **Order dependency:** reliance on database state, file system artifacts, or class-level mutation without cleanup
   - **External services:** HTTP clients, database connections, message queues, file I/O to temp directories
   - **Environment:** hardcoded paths, locale-dependent formatting, OS-specific APIs, port binding
   - **Randomness:** `Math.random()`, `Random` without seed, `UUID.randomUUID()` in assertions, `HashMap` iteration

6. **Cross-reference test signals with production context.** A `Thread.sleep(500)` is only problematic if the operation it waits on can exceed 500ms. A database call is only flaky if the test uses a real (not mocked) connection. This step resolves ambiguities that the paper showed are impossible to resolve from test code alone.

7. **Classify each test with confidence and rationale.** Assign one of: `LIKELY_FLAKY` (clear root cause identified with context), `POSSIBLY_FLAKY` (suspicious patterns but mitigating factors present), or `UNLIKELY_FLAKY` (no detectable risk factors). Always state the specific root-cause category and the evidence from both test and production code.

8. **Recommend targeted fixes.** For each flaky or possibly-flaky test, provide a concrete remediation: replace `sleep` with polling/await, mock external services, add test isolation, seed random generators, use `@TempDir` instead of hardcoded paths, etc.

## Concrete Examples

**Example 1: Timing-dependent test in a Java project**

User: "This test fails intermittently in CI. Is it flaky?"
```java
@Test
public void testAsyncNotification() {
    notificationService.sendAsync("user@example.com", "Hello");
    Thread.sleep(2000);
    List<Notification> sent = notificationRepository.findAll();
    assertEquals(1, sent.size());
}
```

Approach:
1. Read the test — sees `Thread.sleep(2000)` and async call pattern
2. Read `NotificationService.sendAsync()` source — discovers it uses a thread pool with configurable queue size and sends via SMTP
3. Read CI config — finds tests run with `-DforkCount=4` (parallel execution) on a shared CI runner
4. Check `application-test.properties` — finds it points to a real SMTP test server, not a mock

Output:
```
Classification: LIKELY_FLAKY
Root cause: Async/timing dependency
Confidence: High

Evidence:
- Test code: Thread.sleep(2000) used as a hard wait for async operation
- Production code: NotificationService.sendAsync() submits to thread pool, then
  sends SMTP email — total latency depends on SMTP server response time
- CI config: 4 parallel forks compete for resources, increasing latency variance
- No mock: Test hits a real SMTP server, adding network non-determinism

Recommendation:
Replace Thread.sleep with Awaitility polling:
  await().atMost(10, SECONDS)
         .untilAsserted(() ->
             assertEquals(1, notificationRepository.findAll().size()));
Consider mocking the SMTP transport in unit tests.
```

**Example 2: Shared-state test in a Python project**

User: "Some tests pass alone but fail when the full suite runs. Help me find which ones are flaky."

Approach:
1. Scan test files for shared module-level state: global variables, class attributes, singleton patterns
2. Read `conftest.py` fixtures — check scope (`session`, `module`, `function`) and whether fixtures mutate shared state
3. Read `pytest.ini` / `pyproject.toml` — check for `--randomly-seed`, parallel execution (`-n auto`), test ordering plugins
4. Identify tests that write to shared resources (database, files, caches) without cleanup

Output:
```
Found 3 tests with order-dependency risk:

1. test_user_creation (test_auth.py:45)
   Classification: LIKELY_FLAKY (order dependency)
   - Inserts into `users` table using module-scoped db fixture
   - No teardown/rollback — subsequent tests see leftover rows
   - Fix: Use function-scoped fixture with transaction rollback

2. test_cache_expiry (test_cache.py:78)
   Classification: POSSIBLY_FLAKY (shared state)
   - Reads from module-level `_cache = {}` dict populated by test_cache_set
   - If test_cache_set runs first, passes; otherwise, KeyError
   - Fix: Initialize cache in each test or use a fixture

3. test_config_override (test_settings.py:22)
   Classification: LIKELY_FLAKY (environment + order dependency)
   - Monkeypatches os.environ["APP_MODE"] but uses module scope
   - Other tests in same module inherit the patched env
   - Fix: Use function-scoped monkeypatch or `with mock.patch.dict()`
```

**Example 3: Test that looks suspicious but is actually fine**

User: "Is this test flaky? It uses random data."
```python
def test_sorting_algorithm():
    random.seed(42)
    data = [random.randint(0, 1000) for _ in range(100)]
    result = my_sort(data)
    assert result == sorted(data)
```

Approach:
1. Read the test — spots `random` usage but also `random.seed(42)` (deterministic)
2. Read `my_sort()` source — confirms it is a pure function with no side effects
3. Check for parallel test execution config — even with parallelism, the seeded RNG is local

Output:
```
Classification: UNLIKELY_FLAKY
Root cause: None detected

Evidence:
- Random is seeded with fixed seed (42) — output is deterministic
- my_sort() is a pure function with no external dependencies
- No shared state, no I/O, no timing sensitivity

No action needed. This test is well-constructed.
```

## Best Practices

- **Do:** Always read the production code under test before classifying. The paper's central finding is that test code alone lacks sufficient signal — the production code is where timing, concurrency, and external dependency details live.
- **Do:** Check build/CI configuration for parallelism settings. Many flakiness issues only manifest under parallel execution, which is invisible in the test code itself.
- **Do:** Distinguish between unit tests (which can often be classified from code alone if mocking is visible) and integration tests (which almost always require context about external systems).
- **Do:** Report confidence levels honestly. If you cannot access production code or CI config, say so and downgrade confidence rather than guessing.
- **Avoid:** Classifying flakiness from test code alone. The paper empirically proved this approach fails — even GPT-4o with chain-of-thought prompting achieved near-random results.
- **Avoid:** Assuming every `sleep()` or `random()` call indicates flakiness. Context determines whether these patterns are problematic. A seeded random or a sleep that vastly exceeds the operation's maximum latency may be perfectly safe.

## Error Handling

- **Cannot access production code:** If the user only provides the test method, explicitly state that classification confidence is low and request the production code. Cite the paper's finding that test-code-only classification is near random.
- **Cannot determine test execution environment:** Flag this as a gap. Flakiness often depends on CI parallelism, container resource limits, or network conditions that are not visible in code.
- **Ambiguous patterns:** When a test has suspicious patterns but also mitigating factors (e.g., `sleep` with a very generous timeout), classify as `POSSIBLY_FLAKY` and explain both the risk and the mitigation.
- **Large test suites:** When asked to scan an entire suite, prioritize integration tests, tests with external dependencies, and tests with known CI failure history. Do not attempt to classify hundreds of tests individually — focus on the highest-risk patterns first.

## Limitations

- **Test code alone is insufficient.** This is the paper's core finding and the fundamental constraint. Without production code, configuration, and environment details, flakiness classification accuracy approaches random chance.
- **Some flakiness is undetectable from static analysis.** Race conditions, resource contention under load, and infrastructure-level issues (DNS resolution, disk I/O variance) cannot be reliably detected by reading code — they require runtime observation or historical failure data.
- **Dataset bias.** The IDoFT and FlakeFlagger datasets used in the paper are Java-centric. Flakiness patterns in other ecosystems (Python, JavaScript, Go) may differ in prevalence and manifestation.
- **LLM classification is not a substitute for reruns.** The most reliable flakiness detection remains running tests multiple times (e.g., `pytest --count=10`, Maven Surefire `rerunFailingTestsCount`). LLM-based analysis is a triage tool, not a definitive oracle.

## Reference

Berndt, A., Bekmyradov, V., Gemulla, R., Kessel, M., & Bach, T. (2026). *Can We Classify Flaky Tests Using Only Test Code? An LLM-Based Empirical Study.* arXiv:2602.05465v1. SANER-RENE 2025. [https://arxiv.org/abs/2602.05465v1](https://arxiv.org/abs/2602.05465v1)

Key takeaway: Test-code-only flakiness classification with LLMs (GPT-4o, GPT-4o-mini, CodeLlama) across zero-shot, few-shot, and chain-of-thought prompting yields results near random chance (MCC ~ 0). The critical missing ingredient is project context — production code, build configuration, CI setup, and runtime environment. Any practical flakiness classifier must retrieve this context before reasoning.