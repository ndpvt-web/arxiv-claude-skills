---
name: "issueguard-real-time-secret-leak"
description: "Scan text for leaked secrets using a two-stage pipeline: regex candidate extraction followed by contextual classification to eliminate false positives. Use when the user says 'scan for secrets', 'check for leaked credentials', 'find API keys in this text', 'detect hardcoded secrets', 'audit issue text for sensitive data', or 'prevent secret leaks'."
---

# IssueGuard: Two-Stage Secret Leak Detection

This skill enables Claude to detect accidentally leaked secrets (API keys, tokens, passwords, credentials) in unstructured text such as GitHub issue reports, logs, code snippets, configuration files, and documentation. It applies the IssueGuard two-stage pipeline: first, regex-based candidate extraction to catch anything that *looks* like a secret, then contextual classification to separate real secrets from false positives (placeholders, redacted values, dummy keys, example strings). This approach achieves 92.70% F1-score, dramatically outperforming regex-only scanners like TruffleHog (69.69% F1) and Gitleaks (63.60% F1) which suffer from massive false positive rates.

## When to Use

- When the user asks to scan a GitHub issue body, bug report, or support ticket for accidentally leaked secrets
- When reviewing text that contains logs, stack traces, or config snippets before posting publicly
- When the user pastes text and asks "is there anything sensitive in here?"
- When building or auditing a pre-submission hook that checks issue content for credentials
- When the user wants to implement a secret detection pipeline in their own codebase
- When triaging a batch of issue reports or comments for secret exposure incidents
- When the user asks to build regex patterns for detecting specific secret types (AWS keys, GitHub tokens, etc.)

## Key Technique: Regex Extraction + Contextual Classification

**Stage 1 — Regex Candidate Extraction.** The first stage casts a wide net using regular expressions matched against known secret formats. IssueGuard uses 761 regex patterns covering API keys, OAuth tokens, private keys, database connection strings, and more. This stage deliberately prioritizes recall over precision — it achieves ~100% recall but only ~6.8% precision. The goal is to never miss a real secret, even if that means flagging many non-secrets. Each regex match produces a *candidate* with its surrounding text.

**Stage 2 — Contextual Classification.** Each candidate is passed through a classifier that examines a 200-character context window around the matched string. The original IssueGuard uses a fine-tuned CodeBERT model for this, but the core insight is applicable without ML: by examining context (variable names like `example_key`, surrounding text like `# replace with your key`, placeholder patterns like `xxx` or `REDACTED`), you can reliably distinguish real secrets from benign strings. The binary decision is: **Secret** (a real, exploitable credential) vs. **Non-sensitive** (placeholder, redacted value, dummy key, documentation example).

**Why this matters.** Pure regex scanning is nearly useless for unstructured text — at 6.8% precision, over 93% of alerts are false positives. The contextual second stage is what makes the pipeline practical. When implementing this for users, Claude should replicate both stages: broad pattern matching followed by context-aware filtering.

## Step-by-Step Workflow

1. **Collect the input text.** Read the full content to be scanned — issue body, log output, config file, or any unstructured text the user provides. Preserve the original formatting; secrets often appear in code blocks, inline code, or raw text.

2. **Apply regex candidate extraction.** Scan the text against patterns for known secret formats. At minimum, cover these categories:
   - AWS access keys (`AKIA[0-9A-Z]{16}`)
   - AWS secret keys (40-character base64 strings near AWS context)
   - GitHub tokens (`ghp_[A-Za-z0-9]{36}`, `gho_`, `ghu_`, `ghs_`, `ghr_`)
   - GitLab tokens (`glpat-[A-Za-z0-9\-]{20,}`)
   - Generic API keys (high-entropy strings 20+ characters following keywords like `key`, `token`, `secret`, `password`, `apikey`, `api_key`)
   - Private keys (`-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----`)
   - Database connection strings (URIs with embedded credentials: `://user:pass@host`)
   - JWT tokens (`eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}`)
   - Slack tokens (`xox[bpas]-[A-Za-z0-9-]+`)
   - Generic high-entropy strings (32+ hex or base64 chars adjacent to secret-indicating keywords)

3. **Extract context window for each candidate.** For every regex match, capture 200 characters of surrounding text (100 before, 100 after). This context is critical for classification.

4. **Classify each candidate contextually.** Examine the context window and the candidate string itself. Mark as **Non-sensitive** if any of these indicators are present:
   - Placeholder patterns: `xxx`, `your_key_here`, `REPLACE_ME`, `<token>`, `INSERT`, `example`, `dummy`, `test`, `fake`, `sample`
   - Redaction patterns: `***`, `REDACTED`, `[hidden]`, `...` truncation
   - Documentation context: the surrounding text is explaining how to set a variable, not providing a real value
   - Well-known dummy values: `sk_test_`, `pk_test_` (Stripe test keys), `AKIAIOSFODNN7EXAMPLE` (AWS example key)
   - Low entropy: the string has low Shannon entropy relative to its length (below ~3.5 bits/char for 20+ char strings)
   - Variable assignment to empty or clearly fake value

   Mark as **Secret** if:
   - High entropy and matches a known secret format
   - Context suggests a real credential (e.g., appears in a stack trace, error log, or config dump without placeholder markers)
   - No redaction or placeholder indicators in the surrounding text

5. **Calculate entropy for borderline cases.** For candidates where context is ambiguous, compute Shannon entropy: `-sum(p * log2(p))` for each character frequency. Real secrets typically have entropy > 4.0 bits/char; placeholders and common strings are lower.

6. **Generate a structured report.** For each confirmed secret, output:
   - The secret type (AWS key, GitHub token, etc.)
   - The matched value (partially redacted in output: show first 4 and last 4 characters)
   - The line number or position in the text
   - The surrounding context (to help the user verify)
   - Severity: HIGH (cloud provider keys, private keys), MEDIUM (API tokens, JWTs), LOW (possible passwords or generic keys)

7. **Provide remediation guidance.** For each detected secret:
   - Recommend immediate rotation of the credential
   - Identify which service the credential belongs to
   - Suggest using environment variables or secret managers instead
   - If the text hasn't been posted yet, recommend removing the secret before submission

8. **Summarize findings.** Present a count of total candidates found (Stage 1), confirmed secrets (Stage 2), and false positives filtered out, giving the user confidence in the results.

## Concrete Examples

**Example 1: Scanning a GitHub issue body for leaked AWS credentials**

User: "Check this issue text for any secrets before I post it"

```
I'm getting a 403 error when trying to access S3. Here's my config:

aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
region = us-east-1

The bucket policy looks correct. Any ideas?
```

Approach:
1. Regex extraction finds two candidates: `AKIAIOSFODNN7EXAMPLE` (matches `AKIA[0-9A-Z]{16}`) and `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` (40-char base64 near AWS context)
2. Contextual classification checks both against known dummy values
3. `AKIAIOSFODNN7EXAMPLE` is the AWS documentation example key — **Non-sensitive**
4. `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` is the AWS documentation example secret — **Non-sensitive**

Output:
```
Secret Scan Results: 0 confirmed secrets found

Stage 1 (regex): 2 candidates extracted
Stage 2 (classification): 0 confirmed, 2 filtered as non-sensitive

Details:
  - AKIA...MPLE → AWS Access Key format → Known AWS example key (Non-sensitive)
  - wJal...EKEY → AWS Secret Key format → Known AWS example secret (Non-sensitive)

Verdict: Safe to post. Both values are well-known AWS documentation examples.
```

**Example 2: Detecting a real leaked GitHub token in a bug report**

User: "Scan this for secrets"

```
Getting auth failures with the GitHub API. Here's the curl command I ran:

curl -H "Authorization: token ghp_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890" \
  https://api.github.com/repos/myorg/myrepo/issues

Response: {"message":"Bad credentials","documentation_url":"..."}
```

Approach:
1. Regex extraction finds `ghp_aBcDeFgHiJkLmNoPqRsTuVwXyZ1234567890` (matches `ghp_[A-Za-z0-9]{36}`)
2. Context window: appears in an `Authorization` header in a curl command — this is a real usage, not documentation
3. No placeholder indicators (no `example`, `your_token`, `REPLACE`, etc.)
4. Entropy is high (mixed case alphanumeric, 40 chars)
5. Classification: **Secret** — HIGH severity

Output:
```
SECRET DETECTED — 1 confirmed secret found

Stage 1 (regex): 1 candidate extracted
Stage 2 (classification): 1 confirmed secret

  [HIGH] GitHub Personal Access Token
  Value: ghp_aBcD...7890
  Location: Line 3 (in Authorization header)
  Context: Used in curl command as bearer token — real credential, not a placeholder

Remediation:
  1. Do NOT post this text as-is
  2. Revoke this token immediately: GitHub → Settings → Developer settings → Personal access tokens
  3. Generate a new token with minimum required scopes
  4. Use environment variables: -H "Authorization: token $GITHUB_TOKEN"
```

**Example 3: Building a secret scanner script for a CI pipeline**

User: "Help me write a Python script that scans issue text for secrets using the IssueGuard approach"

Approach:
1. Implement Stage 1: a dictionary of compiled regex patterns by secret type
2. Implement Stage 2: context window extraction + heuristic classification (placeholder detection, entropy calculation)
3. Return structured results

Output (key implementation):
```python
import re
import math
from dataclasses import dataclass

@dataclass
class SecretCandidate:
    secret_type: str
    value: str
    start: int
    end: int
    context: str
    severity: str

SECRET_PATTERNS = {
    "AWS Access Key": (re.compile(r"AKIA[0-9A-Z]{16}"), "HIGH"),
    "GitHub Token": (re.compile(r"gh[pousr]_[A-Za-z0-9]{36,}"), "HIGH"),
    "GitLab Token": (re.compile(r"glpat-[A-Za-z0-9\-]{20,}"), "HIGH"),
    "Slack Token": (re.compile(r"xox[bpas]-[A-Za-z0-9\-]+"), "MEDIUM"),
    "JWT": (re.compile(r"eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]+"), "MEDIUM"),
    "Private Key": (re.compile(r"-----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----"), "HIGH"),
    "DB Connection String": (re.compile(r"(?:mongodb|postgres|mysql|redis)://\S+:\S+@\S+"), "HIGH"),
    "Generic High-Entropy Secret": (re.compile(
        r"(?:api[_-]?key|secret|token|password|passwd|credential)[\s]*[=:]\s*['\"]?([A-Za-z0-9/+=]{20,})['\"]?"
    , re.IGNORECASE), "MEDIUM"),
}

PLACEHOLDER_INDICATORS = re.compile(
    r"example|dummy|test|fake|sample|placeholder|replace|your[_\s]|insert|xxx|redacted|\*{3,}|<[^>]+>",
    re.IGNORECASE,
)

KNOWN_DUMMY_SECRETS = {
    "AKIAIOSFODNN7EXAMPLE",
    "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
}

def shannon_entropy(s: str) -> float:
    if not s:
        return 0.0
    freq = {}
    for c in s:
        freq[c] = freq.get(c, 0) + 1
    length = len(s)
    return -sum((count / length) * math.log2(count / length) for count in freq.values())

def extract_context(text: str, start: int, end: int, window: int = 100) -> str:
    ctx_start = max(0, start - window)
    ctx_end = min(len(text), end + window)
    return text[ctx_start:ctx_end]

def scan_text(text: str) -> list[SecretCandidate]:
    candidates = []
    for secret_type, (pattern, severity) in SECRET_PATTERNS.items():
        for match in pattern.finditer(text):
            value = match.group(0)
            context = extract_context(text, match.start(), match.end())

            # Stage 2: Contextual classification
            if value in KNOWN_DUMMY_SECRETS:
                continue
            if PLACEHOLDER_INDICATORS.search(context):
                continue
            if secret_type == "Generic High-Entropy Secret":
                secret_value = match.group(1) if match.lastindex else value
                if shannon_entropy(secret_value) < 3.5:
                    continue

            candidates.append(SecretCandidate(
                secret_type=secret_type,
                value=value,
                start=match.start(),
                end=match.end(),
                context=context,
                severity=severity,
            ))
    return candidates
```

## Best Practices

- **Do:** Always run both stages. Regex alone produces ~93% false positives on unstructured text. The contextual classification stage is not optional — it is what makes the results useful.
- **Do:** Extract at least 200 characters of surrounding context for each candidate. Without context, you cannot distinguish `api_key = "AKIAIOSFODNN7EXAMPLE"` (documentation) from a real key in a log dump.
- **Do:** Partially redact detected secrets in your output (show first 4 and last 4 characters). Avoid displaying the full secret, since Claude's output may itself be logged.
- **Do:** Maintain a list of known dummy/example credentials (AWS example keys, Stripe test keys, etc.) and filter them explicitly.
- **Avoid:** Relying solely on entropy thresholds. Some real secrets have moderate entropy (short passwords), and some benign strings have high entropy (UUIDs, commit hashes). Entropy is a supplementary signal, not a primary classifier.
- **Avoid:** Reporting private keys detected inside `-----BEGIN ... -----END` blocks without checking if they are part of documentation showing the *format* of a key. Check if the content between the markers has real key material or is truncated/placeholder text.

## Error Handling

- **Malformed regex matches:** Some patterns may match partial strings at text boundaries. Always validate that the matched string has the expected length and character set before classifying.
- **Extremely large input text:** For texts over 100KB, scan in chunks with overlapping windows (at least 200 chars overlap) to avoid splitting a secret across chunk boundaries.
- **Unicode and encoding issues:** Secrets are ASCII, but surrounding text may contain Unicode. Ensure regex patterns operate on the raw text without encoding errors. Use `re.ASCII` flag where appropriate.
- **Ambiguous classifications:** When context is insufficient to determine if a candidate is real or a placeholder, err on the side of caution and report it with a note that manual verification is needed. A false positive is better than a missed secret.
- **Nested code blocks:** Secrets inside markdown code fences or HTML `<code>` tags are still secrets. Do not skip content inside code blocks — these are actually where secrets most commonly appear in issue reports.

## Limitations

- **Novel secret formats:** The regex stage can only catch secrets matching known patterns. Custom or proprietary token formats that don't match standard patterns will be missed. Users should extend the pattern list for their specific services.
- **Obfuscated secrets:** Base64-encoded or otherwise transformed secrets embedded in larger strings may evade regex extraction. The scanner does not decode or decompose encoded content.
- **Context window edge cases:** When a secret appears at the very start or end of the text, the 200-character context window is truncated, potentially reducing classification accuracy.
- **Not a replacement for secret scanning in git:** This technique targets unstructured text in issues and comments. For committed code, dedicated tools like `git-secrets`, `truffleHog`, or `gitleaks` operating on git history are more appropriate.
- **No ML model inference:** Claude applies the two-stage logic heuristically rather than running the actual fine-tuned CodeBERT model. For production systems processing high volumes, deploying the IssueGuard backend (FastAPI + CodeBERT) will yield higher accuracy, especially on edge cases.

## Reference

- **Paper:** [IssueGuard: Real-Time Secret Leak Prevention Tool for GitHub Issue Reports](https://arxiv.org/abs/2602.08072v1) — Look for: the two-stage pipeline architecture (Section describing regex extraction + CodeBERT classification), the 200-character context window approach, the comparison showing 92.70% F1 vs. TruffleHog's 69.69% and Gitleaks' 63.60%, and the training data composition (54,148 labeled instances).
- **Source Code:** [github.com/nafiurahman00/IssueGuard](https://github.com/nafiurahman00/IssueGuard) — Reference implementation with FastAPI backend, Chrome extension frontend, and pre-trained model weights.