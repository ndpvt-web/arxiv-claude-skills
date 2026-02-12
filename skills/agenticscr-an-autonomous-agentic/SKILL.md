---
name: "agenticscr-an-autonomous-agentic"
description: >
  Agentic secure code review for detecting immature vulnerabilities at pre-commit stage.
  Uses a two-phase Detector-Validator pipeline with SAST-rule semantic memory and CWE-tree
  validation to localize, classify, and explain security weaknesses in code diffs.
  Trigger phrases: "review this diff for security issues", "secure code review",
  "find vulnerabilities in my changes", "pre-commit security check",
  "check this PR for security weaknesses", "agentic security review"
---

# AgenticSCR: Agentic Secure Code Review for Immature Vulnerability Detection

This skill enables Claude to perform structured, agentic secure code review on code diffs and pull requests using the AgenticSCR methodology. Instead of scanning entire codebases or relying on single-pass LLM analysis, this technique uses a two-subagent pipeline — a **Detector** that applies SAST-rule patterns to localize vulnerabilities, followed by a **Validator** that filters false positives using CWE taxonomy knowledge. The approach targets **immature vulnerabilities**: incomplete, latent, or context-dependent weaknesses introduced through small incremental code changes that appear benign in isolation but evolve into exploitable flaws as surrounding code is added.

## When to Use

- When the user asks to review a git diff, patch, or PR for security vulnerabilities before committing
- When the user wants to find injection, authorization, information disclosure, resource management, or control flow vulnerabilities in changed code
- When the user asks "is this code change secure?" or "what vulnerabilities does this diff introduce?"
- When reviewing code changes in Python, JavaScript, or TypeScript for security weaknesses
- When the user wants actionable security review comments with line-level localization and CWE classifications
- When the user asks to reduce false positives from SAST tools like CodeQL, Semgrep, or Snyk by applying contextual validation

## Key Technique

AgenticSCR's core insight is that pre-commit vulnerabilities are **immature** — they don't match the fully-formed patterns that SAST tools are designed to catch. An insecure API call or a missing validation check may look harmless in a small diff, but becomes exploitable as surrounding code evolves. Detecting these requires combining pattern-based detection (SAST rules) with contextual reasoning (understanding what the code does in its repository context) and taxonomic validation (confirming the weakness maps to a real CWE category).

The system operates as a **two-subagent pipeline**. The Detector subagent loads SAST rule definitions (CodeQL-style patterns including rule ID, description, CWE mapping, severity, and vulnerable/secure code examples) into its working context, then navigates the repository using tools — reading diffs, expanding code hunks for surrounding context, grepping for security-relevant patterns, and inspecting directory structure. It produces review comments with file path, line number, vulnerability description, and predicted CWE. The Validator subagent then loads the CWE-1000 taxonomy tree and validates each comment: does the identified weakness match a real CWE category? Are the preconditions for that weakness present? Is the finding exploitable given the code context? Comments that fail validation are filtered as false positives.

This two-phase approach achieves 153% more correct review comments than single-pass LLM analysis and 71-85% fewer false positives than SAST tools, because the Detector casts a wide net using established patterns while the Validator prunes aggressively using domain knowledge.

## Step-by-Step Workflow

1. **Extract the diff scope.** Obtain the code diff (from `git diff`, a PR, or pasted text). Identify the modified files, their languages, and the specific line ranges changed. Focus only on modified code — do not ingest the entire repository.

2. **Expand context around changed hunks.** For each modified file, read 20-50 lines of surrounding context above and below the changed lines. This reveals function signatures, imports, class definitions, and control flow that the diff alone hides. Use file reading and grep to trace how changed variables are used elsewhere.

3. **Apply SAST-rule detection patterns.** Systematically check the diff against these five vulnerability categories:
   - **Injection (CWE-707):** Missing input sanitization, unsanitized user input flowing into SQL, shell commands, file paths, or template engines
   - **Authorization (CWE-284/287):** Broken authentication checks, missing access control, privilege escalation through insecure permission changes
   - **Information Disclosure (CWE-200):** Exposed logs containing secrets, sensitive data in error messages, credentials in configuration
   - **Resource Management (CWE-664):** Unbounded allocations, missing resource cleanup, loading untrusted binaries or plugins
   - **Control Flow (CWE-691):** Incorrect boolean conditions, missing validation branches, false-positive checks that always pass

4. **Localize each finding to a specific line.** For every potential vulnerability, identify the exact file path and line number where the weakness is introduced. The finding must point to a line within or very close to (within +-5 lines of) the actual changed code.

5. **Draft a review comment for each finding.** Each comment must include:
   - `file`: the file path
   - `line`: the specific line number
   - `comment`: a plain-language explanation of the vulnerability and why it's dangerous
   - `cwe`: the CWE ID (e.g., CWE-79) and name
   - `severity`: critical, high, medium, or low
   - `suggestion`: concrete remediation code or guidance

6. **Validate each comment against CWE taxonomy.** For every drafted comment, ask:
   - Does the identified weakness genuinely map to the claimed CWE category?
   - Are the preconditions for this CWE actually present (e.g., for injection: is there actually user-controlled input flowing to a sink)?
   - Could this be a false positive — does the surrounding code already mitigate this issue (e.g., sanitization upstream, type constraints, framework protections)?
   - Remove comments that fail validation. Downgrade severity if the exploitability is unclear.

7. **Check for cross-file dependencies.** If a changed function is called from other files, grep for its callers. Verify whether input sanitization happens at the call site rather than the definition site. A missing validation in the diff may be intentional if validation occurs upstream.

8. **Classify the overall risk.** Summarize how many validated findings exist per severity level. Flag any critical or high findings that need immediate attention before commit.

9. **Present findings as structured review comments.** Output a clean table or list of validated findings sorted by severity, each with file, line, CWE, description, and remediation suggestion.

10. **Acknowledge uncertainty.** For findings where context is incomplete (e.g., you cannot see the full call chain), mark the finding as "potential" rather than "confirmed" and explain what additional context would be needed to verify.

## Concrete Examples

**Example 1: SQL Injection in a Python Web Endpoint**

User: "Review this diff for security issues"
```python
# app/routes/users.py (changed lines)
+ @app.route('/users/search')
+ def search_users():
+     query = request.args.get('q')
+     results = db.execute(f"SELECT * FROM users WHERE name LIKE '%{query}%'")
+     return jsonify([dict(r) for r in results])
```

Approach:
1. Extract diff: single file `app/routes/users.py`, lines added to a Flask route
2. Expand context: check if there's a sanitization middleware or ORM layer imported
3. Detect: `request.args.get('q')` flows unsanitized into an f-string SQL query — classic SQL injection (CWE-89, child of CWE-707 Injection)
4. Localize: line 4 of the diff (`db.execute(f"SELECT...`)
5. Validate: CWE-89 requires user-controlled input reaching a SQL execution sink without parameterization — both conditions confirmed
6. No upstream sanitization found

Output:
| File | Line | CWE | Severity | Finding |
|------|------|-----|----------|---------|
| app/routes/users.py | 4 | CWE-89 (SQL Injection) | Critical | User input from `request.args.get('q')` is interpolated directly into SQL via f-string. Use parameterized queries: `db.execute("SELECT * FROM users WHERE name LIKE ?", (f"%{query}%",))` |

**Example 2: Missing Authorization Check in a JavaScript API**

User: "Check this PR for security weaknesses"
```javascript
// src/api/admin.js (changed lines)
+ router.delete('/admin/users/:id', async (req, res) => {
+   const userId = req.params.id;
+   await User.findByIdAndDelete(userId);
+   res.json({ success: true });
+ });
```

Approach:
1. Extract diff: admin route for deleting users
2. Expand context: check if `router` has authentication/authorization middleware applied globally
3. Grep for `router.use(auth` or `requireAdmin` in same file and route configuration
4. Detect: no authorization middleware visible on this route — any authenticated (or unauthenticated) user could delete arbitrary users (CWE-284 Improper Access Control)
5. Validate: CWE-284 requires a privileged action accessible without proper authorization checks — confirmed for user deletion endpoint
6. Also check: no input validation on `:id` parameter (potential CWE-20), but framework likely handles ObjectId validation

Output:
| File | Line | CWE | Severity | Finding |
|------|------|-----|----------|---------|
| src/api/admin.js | 1 | CWE-284 (Improper Access Control) | Critical | Admin user deletion endpoint lacks authorization middleware. Any request to `DELETE /admin/users/:id` will succeed. Add `requireAdmin` middleware: `router.delete('/admin/users/:id', requireAdmin, async (req, res) => {` |
| src/api/admin.js | 2 | CWE-20 (Improper Input Validation) | Low (potential) | `req.params.id` is not validated before database query. Verify that the ORM or framework rejects malformed IDs — if using MongoDB with Mongoose, ObjectId casting provides implicit validation. |

**Example 3: Information Disclosure via Error Logging**

User: "Find vulnerabilities in my changes"
```python
# services/payment.py (changed lines)
+ def process_payment(card_number, amount):
+     try:
+         result = gateway.charge(card_number, amount)
+     except GatewayError as e:
+         logger.error(f"Payment failed for card {card_number}: {e}")
+         raise PaymentError("Payment processing failed")
+     return result
```

Approach:
1. Extract diff: payment processing function with error handling
2. Expand context: check logger configuration — does it write to files, stdout, or external services?
3. Detect: `card_number` (PCI-sensitive data) is logged in plaintext in the error handler (CWE-532, child of CWE-200 Information Exposure)
4. Localize: line 5 (`logger.error(f"Payment failed for card {card_number}..."`)
5. Validate: CWE-532 requires sensitive data written to log files — card numbers are PCI DSS-regulated data, logging them violates compliance
6. The `raise PaymentError` is safe (generic message), but the logger line is the issue

Output:
| File | Line | CWE | Severity | Finding |
|------|------|-----|----------|---------|
| services/payment.py | 5 | CWE-532 (Information Exposure Through Log Files) | High | Full card number logged in error handler. This violates PCI DSS requirements. Mask the card number: `logger.error(f"Payment failed for card ***{card_number[-4:]}: {e}")` |

## Best Practices

- **Do:** Focus exclusively on the diff and its immediate context. Pre-commit review is about what changed, not auditing the entire codebase.
- **Do:** Always validate findings against CWE preconditions before reporting. A finding without confirmed preconditions is noise.
- **Do:** Provide concrete remediation code in every review comment — a vulnerability report without a fix is half the job.
- **Do:** Check for framework-level protections before flagging. Django's ORM prevents SQL injection by default; React escapes JSX by default. Don't report what the framework already handles.
- **Avoid:** Reporting stylistic issues, code quality concerns, or non-security findings. This is a security review, not a general code review.
- **Avoid:** Generating large numbers of low-confidence findings. The Validator phase exists to reduce false positives — err on the side of precision over recall.
- **Avoid:** Claiming a vulnerability is "confirmed" when you cannot see the full data flow. Use "potential" and explain what additional context is needed.

## Error Handling

- **Incomplete diff context:** If the diff alone doesn't show enough code to determine whether a vulnerability exists (e.g., sanitization might happen in an imported function), explicitly state the assumption and recommend the user verify. Use grep/search to check for upstream mitigations before concluding.
- **Unknown framework or library:** If the code uses a framework you're uncertain about (e.g., whether it auto-escapes output), say so rather than guessing. Flag the finding as "potential" pending framework documentation review.
- **Ambiguous CWE mapping:** When a weakness could map to multiple CWE IDs, choose the most specific one and note the alternatives. For example, a missing input check could be CWE-20 (Input Validation) or CWE-89 (SQL Injection) depending on the sink — pick the one matching the actual exploitation path.
- **Large diffs exceeding context limits:** For diffs over 100 changed lines across multiple files, prioritize files that handle user input, authentication, cryptography, and external system calls. Review these first and flag remaining files as needing separate review passes.

## Limitations

- **Immature vulnerabilities are inherently uncertain.** A code change that introduces a latent weakness may never become exploitable if subsequent code adds proper protections. Pre-commit review flags risks, not guaranteed exploits.
- **Cross-repository and dependency vulnerabilities are out of scope.** This technique reviews first-party code changes. Vulnerabilities in third-party dependencies require separate tools (e.g., `npm audit`, `pip-audit`).
- **Control flow vulnerabilities have the lowest detection rate.** The original paper found 0% correct detection for CWE-691 (Control Flow) issues, which require deep logical reasoning about program state that pattern-based detection struggles with.
- **Language coverage.** The technique was validated on Python, JavaScript, and TypeScript. It can be applied to other languages but with lower confidence in CWE pattern matching.
- **Single-commit scope.** The review analyzes one commit's diff at a time. Vulnerabilities spanning multiple commits (e.g., a validation function removed in commit A and a new unvalidated input path added in commit B) may be missed.

## Reference

**Paper:** [AgenticSCR: An Autonomous Agentic Secure Code Review for Immature Vulnerabilities Detection](https://arxiv.org/abs/2601.19138v1) — Charoenwet et al., 2026. Look for: the two-subagent Detector-Validator architecture, the SAST-rule and CWE-tree semantic memory design, the SCRBench benchmark with 144 pre-commit changes across 107 CVEs, and the five vulnerability type definitions (Injection, Authorization, Information, Resource, Control).