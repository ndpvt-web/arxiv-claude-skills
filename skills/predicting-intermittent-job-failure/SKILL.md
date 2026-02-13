---
name: "predicting-intermittent-job-failure"
description: "Classify and diagnose intermittent CI/CD job failures from execution logs using the FlaXifyer few-shot approach and LogSift log reduction. Trigger phrases: 'diagnose flaky build', 'classify CI failure', 'why did this pipeline fail intermittently', 'triage job log', 'find root cause in CI log', 'reduce CI log noise'"
---

# Predicting Intermittent Job Failure Categories

This skill enables Claude to classify intermittent (flaky) CI/CD job failures into actionable diagnostic categories and surface the most influential log lines causing the failure — all from raw job execution logs. It implements the FlaXifyer classification taxonomy and LogSift log reduction technique from Aïdasso et al. (2026), which achieved 84.3% Macro F1 with only 12 labeled examples per category across 13 failure types on real-world GitLab pipelines.

## When to Use

- When a user pastes a CI/CD job log and asks why the pipeline failed or keeps failing intermittently
- When triaging flaky builds in GitLab CI, GitHub Actions, Jenkins, or any CI system that produces execution logs
- When a user wants to reduce a long job log to the most relevant lines for diagnosis
- When building an automated failure triage system or labeling pipeline for CI job failures
- When a user asks how to categorize intermittent failures across a codebase or organization
- When setting up few-shot classification of structured log data using contrastive learning with SetFit

## Key Technique

**FlaXifyer** uses contrastive few-shot learning (via the SetFit framework) to classify preprocessed CI job logs into failure categories. Unlike prompt-based LLM classification, it fine-tunes a sentence encoder (BGE or CodeBERT) using triplet contrastive loss: positive pairs are logs from the same failure category, negative pairs are from different categories. A logistic regression head maps the resulting 768-dimensional embeddings to one of K failure categories. This approach reaches strong performance with as few as 12 labeled examples per category — critical when labeled CI failure data is scarce.

**LogSift** is a model-agnostic interpretability technique that identifies the minimal set of log lines responsible for the classifier's prediction. It uses a binary-search-inspired recursive bisection: split the log in half, check which half preserves the original prediction, recurse into that half, and stop at a minimum segment size (default: 2 lines). This runs in O(log n) time when failure signals are localized, typically completing in under 1 second. On average it reduces logs by 74.4%, surfacing relevant failure information in 87% of cases.

**The 13 failure categories** identified from real-world data are: misconfigured environment variable, job execution timeout, dependency installation failure, runner pod waiting timeout, API gateway deployment error, container registry server error, git transient error, flaky UI test, external file invalid format, host resolution failure, runner image pull failure, remote call timeout, and Helm resource error. These cover the vast majority of intermittent CI failures in cloud-native environments.

## Step-by-Step Workflow

### Phase 1: Log Preprocessing

1. **Extract the raw job log** from the CI system (GitLab job trace, GitHub Actions log, Jenkins console output). Strip ANSI color codes and CI system wrapper lines (timestamps with no content, progress bars, section markers).

2. **Apply the 7-rule preprocessing pipeline** in order:
   - Replace URLs with `<URL>`, file paths with `<FILEPATH>`, directory paths with `<DIRPATH>`, durations with `<DURATION>`, version strings with `<VERSION>`
   - Replace alphanumeric identifiers (mixed letters+digits like commit SHAs, pod IDs, UUIDs) with `<ID>`
   - Remove non-alphanumeric special characters (keep spaces and newlines)
   - Remove standalone numbers except HTTP status codes (4xx, 5xx) and process exit codes
   - Remove trailing single-character suffixes on tokens
   - Collapse whitespace and remove blank lines
   - Deduplicate consecutive identical lines (keep one copy)

3. **Verify the preprocessed log** is reduced to a manageable size. Expect roughly 40% of original line count. If the log exceeds the context window, truncate from the middle (preserve the first 30% and last 30%, as failure signals typically appear at the start or end of job execution).

### Phase 2: Failure Classification

4. **Match against the 13 failure categories** by analyzing the preprocessed log for characteristic patterns:

   | Category | Key Signals |
   |----------|-------------|
   | `misconfigured_env_variable` | Undefined/empty variable references, `envsubst` errors, missing secrets |
   | `job_execution_timeout` | "Job timed out", exceeded time limit messages, stuck stages |
   | `dependency_installation_failure` | npm/pip/maven install errors, version resolution failures, registry 503s |
   | `runner_pod_waiting_timeout` | "Waiting for pod", Kubernetes scheduling failures, pending pod status |
   | `api_gateway_deployment_error` | API gateway config errors, routing failures, deployment rollback |
   | `container_registry_server_error` | Docker push/pull 500 errors, registry authentication failures |
   | `git_transient_error` | "fatal: unable to access", SSL handshake failures, "Connection reset" |
   | `flaky_ui_test` | Element not found, timeout waiting for selector, screenshot diff |
   | `external_file_invalid_format` | Parse errors on downloaded configs, unexpected JSON/YAML format |
   | `host_resolution_failure` | "Could not resolve host", DNS lookup failures, NXDOMAIN |
   | `runner_image_pull_failure` | Image pull backoff, manifest not found, registry timeout during pull |
   | `remote_call_timeout` | HTTP request timeout to external services, connection timed out |
   | `helm_resource_error` | Helm upgrade/install errors, resource conflict, release stuck |

5. **Assign a primary category and confidence level** (high/medium/low). If the log contains signals for multiple categories, provide a ranked top-3 list — the FlaXifyer approach achieves 95.3% top-3 accuracy, so the correct category is almost always in the top 3.

### Phase 3: Log Reduction (LogSift)

6. **Apply the LogSift bisection algorithm** to surface influential lines:
   - Take the full preprocessed log as input
   - Split it into two halves
   - For each half, determine if it alone would lead to the same failure classification
   - Recurse into the half (or halves) that preserve the prediction
   - Stop when segments reach 2 lines or fewer
   - Return the union of all minimal influential segments with their line ranges

7. **Present the reduced log** showing only the influential segments with surrounding context (1-2 lines above/below each segment). Highlight the specific tokens or patterns that drove the classification.

### Phase 4: Diagnosis and Recommendations

8. **Map the failure category to actionable remediation steps**:
   - For infrastructure categories (runner pod timeout, image pull failure): recommend retry with backoff, check cluster health
   - For configuration categories (misconfigured env variable, external file format): identify the specific variable/file and suggest fixes
   - For transient network categories (git error, host resolution, remote call timeout): recommend retry strategies and network resilience patterns
   - For test categories (flaky UI test): identify the specific test and suggest quarantine or stabilization

9. **Generate a structured triage report** containing: failure category, confidence, influential log lines, root cause hypothesis, and recommended action.

10. **If building a classification system**, recommend the SetFit + BGE architecture: collect 12 labeled examples per category, preprocess logs with the 7-rule pipeline, fine-tune BAAI/bge-base-en-v1.5 with contrastive triplet loss, and add a logistic regression classification head.

## Concrete Examples

**Example 1: Diagnosing a flaky GitLab CI job**

User: "This CI job keeps failing intermittently. Here's the log — can you figure out what's going on?"
```
Running with gitlab-runner 15.8.1
Preparing the "kubernetes" executor
Using Kubernetes namespace: ci-runners
Waiting for pod ci-runners/runner-xyz-abc123 to be running, status is Pending
WARNING: Failed to pull image "registry.internal.io/node:18-alpine": rpc error: code = Unknown desc = Error response from daemon: Get "https://registry.internal.io/v2/": dial tcp 10.0.3.45:443: connect: connection timed out
ERROR: Job failed: prepare environment: waiting for pod running: timed out waiting for pod conditions
```

Approach:
1. Preprocess: Replace `registry.internal.io/node:18-alpine` with `<URL>`, `ci-runners/runner-xyz-abc123` with `<ID>`, IP with `<ID>`
2. Classify: Primary signal is "Failed to pull image" + "dial tcp...connection timed out" + "waiting for pod running"
3. Distinguish between `runner_image_pull_failure` (image pull timeout) and `runner_pod_waiting_timeout` (pod scheduling). The root cause here is the image pull, so classify as `runner_image_pull_failure`
4. LogSift reduction: Lines 4-6 are influential (the image pull error chain)

Output:
```
## Triage Report

**Category:** runner_image_pull_failure (high confidence)
**Runner-up:** runner_pod_waiting_timeout

**Influential Log Lines (3 of 7 — 57% reduction):**
> WARNING: Failed to pull image "registry.internal.io/node:18-alpine": ...connection timed out
> ERROR: Job failed: prepare environment: waiting for pod running

**Root Cause:** The container registry at registry.internal.io was unreachable
(TCP connection timeout on port 443). The pod failed to start because its
image could not be pulled.

**Recommended Actions:**
1. Check registry.internal.io availability and network path from CI runners
2. Add imagePullPolicy: IfNotPresent to reuse cached images on runner nodes
3. Configure a registry mirror or pull-through cache for resilience
4. Add retry logic to the runner pod spec (kubernetes executor retry settings)
```

**Example 2: Classifying a dependency installation failure**

User: "Our nightly build failed again. Log attached."
```
$ npm ci --prefer-offline
npm ERR! code ERESOLVE
npm ERR! ERESOLVE could not resolve
npm ERR! While resolving: @company/shared-ui@3.2.1
npm ERR! Found: react@18.2.0
npm ERR! node_modules/react
npm ERR!   react@"^18.2.0" from the root project
npm ERR! Could not resolve dependency:
npm ERR!   peer react@"^17.0.0" from @company/shared-ui@3.2.1
npm ERR! Fix the upstream dependency conflict
npm ERR! ERESOLVE unable to resolve dependency tree
ERROR: Job failed with exit code 1
```

Approach:
1. Preprocess: Replace `@company/shared-ui@3.2.1` with `<ID>`, version numbers retained as semver signals
2. Classify: `npm ERR! ERESOLVE` + "could not resolve dependency" = `dependency_installation_failure` (high confidence)
3. LogSift: Lines 2-9 are influential; the final "Job failed" line is not

Output:
```
## Triage Report

**Category:** dependency_installation_failure (high confidence)

**Influential Log Lines (8 of 11 — 27% reduction):**
> npm ERR! ERESOLVE could not resolve
> peer react@"^17.0.0" from @company/shared-ui@3.2.1

**Root Cause:** @company/shared-ui@3.2.1 requires react ^17.0.0 but the
project has react 18.2.0. This is a peer dependency conflict.

**Recommended Actions:**
1. Update @company/shared-ui to a version compatible with React 18
2. If intentional, use --legacy-peer-deps (add to .npmrc for CI)
3. Pin the dependency version in package-lock.json to prevent drift
```

**Example 3: Building a few-shot failure classifier**

User: "I want to build an automated triage system for our CI failures. How should I approach it?"

Approach:
1. Define failure categories from your historical job logs (start with 8-10 categories, expand later)
2. Label 12 examples per category from real failures
3. Implement the 7-rule log preprocessing pipeline
4. Use SetFit with BAAI/bge-base-en-v1.5 as the encoder

Output:
```python
# Step 1: Preprocess logs
import re

def preprocess_log(raw_log: str) -> str:
    log = raw_log
    # Rule 1: Variable abstraction
    log = re.sub(r'https?://\S+', '<URL>', log)
    log = re.sub(r'/[\w./-]+/[\w.-]+', '<FILEPATH>', log)
    log = re.sub(r'/[\w./-]+/', '<DIRPATH>', log)
    log = re.sub(r'\d+[hms]\d*[ms]?\d*[s]?', '<DURATION>', log)
    log = re.sub(r'\d+\.\d+\.\d+[\w.-]*', '<VERSION>', log)
    # Rule 2: Identifier normalization
    log = re.sub(r'\b(?=[a-zA-Z0-9]*[a-zA-Z])(?=[a-zA-Z0-9]*\d)[a-zA-Z0-9]{8,}\b', '<ID>', log)
    # Rule 3: Special characters
    log = re.sub(r'[^a-zA-Z0-9\s<>]', ' ', log)
    # Rule 4: Numbers (keep HTTP status codes and exit codes)
    log = re.sub(r'(?<!\b[1-5])\b\d+\b(?!\d{2}\b)', '', log)
    # Rule 5: Trailing single chars
    log = re.sub(r'\b\w\b', '', log)
    # Rule 6: Whitespace
    log = re.sub(r'\n\s*\n', '\n', log)
    log = re.sub(r'  +', ' ', log)
    # Rule 7: Deduplication
    lines = log.strip().split('\n')
    deduped = [lines[0]] + [l for i, l in enumerate(lines[1:], 1) if l != lines[i-1]]
    return '\n'.join(deduped)

# Step 2: Train with SetFit (12 examples per category)
from setfit import SetFitModel, SetFitTrainer

model = SetFitModel.from_pretrained("BAAI/bge-base-en-v1.5")
trainer = SetFitTrainer(
    model=model,
    train_dataset=labeled_dataset,  # 12 examples x K categories
    num_iterations=200,
    batch_size=4,
    num_epochs=2,
    body_learning_rate=1e-5,
)
trainer.train()

# Step 3: LogSift for interpretability
def logsift(lines: list[str], classifier, threshold: int = 2) -> list[tuple[int, int]]:
    original_pred = classifier(lines)

    def find_influential(segment, pred, offset):
        if len(segment) <= threshold:
            return [(offset, offset + len(segment) - 1)]
        mid = len(segment) // 2
        top, bot = segment[:mid], segment[mid:]
        top_match = classifier(top) == pred
        bot_match = classifier(bot) == pred
        if top_match and bot_match:
            return find_influential(top, pred, offset) + \
                   find_influential(bot, pred, offset + mid)
        elif top_match:
            return find_influential(top, pred, offset)
        elif bot_match:
            return find_influential(bot, pred, offset + mid)
        else:
            return [(offset, offset + len(segment) - 1)]

    return find_influential(lines, original_pred, 0)
```

## Best Practices

- **Do:** Apply all 7 preprocessing rules in order — variable abstraction before identifier normalization prevents false matches. Skipping rules degrades classification accuracy.
- **Do:** Start with 8 core categories and expand incrementally. Performance drops from 92.5% F1 (K=8) to 84.3% (K=13) as categories increase — add categories only when you have clear distinguishing signals.
- **Do:** Use BGE (bge-base-en-v1.5) over CodeBERT as the encoder. Despite CodeBERT being code-aware, BGE consistently outperformed it in log classification experiments.
- **Do:** Provide top-3 predictions rather than a single answer. The 95.3% top-3 accuracy means the correct category is almost always in the shortlist.
- **Avoid:** Classifying logs without preprocessing. Raw logs contain noise (UUIDs, timestamps, paths) that confuses embeddings and masks the actual failure signal.
- **Avoid:** Using fewer than 8 labeled examples per category. Performance gains plateau around 12 shots, but below 8 the contrastive learning has insufficient signal to form meaningful clusters.

## Error Handling

- **Log too short (< 5 lines after preprocessing):** The failure may be in a preceding job or external system. Check pipeline stage dependencies and upstream job logs.
- **No clear category match:** The failure may be a new category not in the taxonomy. Flag for manual review and collect as a candidate for a new category if it recurs.
- **LogSift returns the entire log (no reduction):** This happens when failure signals are diffuse across the log (worst-case O(n)). Fall back to showing the last 30 lines, where terminal errors typically appear.
- **Multiple categories with similar confidence:** Present top-3 with confidence levels. Categories like `host_resolution_failure` and `remote_call_timeout` share network-related signals — check for DNS-specific vs. TCP-timeout-specific patterns to disambiguate.
- **Extremely long logs (>10K lines):** Truncate to the first 1000 and last 1000 lines before preprocessing. Intermittent failure signals rarely appear in the middle of execution logs.

## Limitations

- The 13-category taxonomy was derived from TELUS GitLab CI data. Organizations using different CI systems or infrastructure may have failure modes not covered (e.g., AWS-specific, CircleCI-specific errors). Extend the taxonomy as needed.
- LogSift assumes the classifier is correct. If the classifier misclassifies, LogSift will surface lines that support the wrong prediction. Always validate LogSift output against the actual error.
- Few-shot contrastive learning requires at least some labeled data. For truly zero-shot scenarios (no labeled examples at all), use direct LLM prompting with the category definitions as a starting point, then collect labels for fine-tuning.
- Categories with overlapping log patterns (e.g., `host_resolution_failure` at 68.4% F1 and `remote_call_timeout` at 73.5% F1) are inherently harder to distinguish. Consider merging closely related categories if per-class accuracy matters more than granularity.
- The preprocessing rules target English-language CI logs with standard tooling output. Logs in other languages or from highly custom build systems may need adapted rules.

## Reference

**Paper:** Aïdasso, H., Bordeleau, F., & Tizghadam, A. (2026). "Predicting Intermittent Job Failure Categories for Diagnosis Using Few-Shot Fine-Tuned Language Models." arXiv:2601.22264v1. https://arxiv.org/abs/2601.22264v1

**Key sections to consult:** Table I for the complete failure category taxonomy; Section III-B for the 7-rule log preprocessing pipeline; Section III-C for SetFit contrastive training details; Algorithm 1 for the LogSift pseudocode; Table II for per-category F1 scores at different shot settings.