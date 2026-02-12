---
name: "sifting-noise-comparative-study"
description: "Filter false positives from static analysis security tools (SAST) using LLM-agent-driven triage. Applies iterative code inspection, dataflow reasoning, and cross-file analysis to classify SAST alerts as true or false positives. Use when: 'triage these CodeQL findings', 'filter SAST false positives', 'review static analysis alerts', 'are these security warnings real?', 'reduce noise from Semgrep/SonarQube results', 'verify if this vulnerability is a false positive'."
---

# SAST False Positive Filtering with LLM Agent Triage

This skill enables Claude to systematically triage static application security testing (SAST) alerts by acting as an agentic security reviewer. Rather than naively classifying alerts from a single file snippet, Claude iterates through the codebase -- reading the flagged file, tracing dataflow into helper classes, checking configuration, and validating constant expressions -- to determine whether each alert represents a true vulnerability or a false positive. This approach, demonstrated in Xiong & Zhang (2026), reduced a 92%+ false positive rate to as low as 6.3% on the OWASP Benchmark and achieved 93.3% FP identification on real-world CodeQL alerts.

## When to Use

- When a user shares SAST output (CodeQL, Semgrep, SonarQube, Joern, SpotBugs) and asks which findings are real
- When triaging a batch of security alerts before a release and needing to prioritize developer time
- When a CI pipeline produces static analysis warnings and the user wants to suppress noise without missing true vulnerabilities
- When a user pastes a specific CWE-flagged code snippet and asks "is this actually exploitable?"
- When reviewing pull requests that introduce SAST findings and deciding which to block on
- When building an automated security triage pipeline and needing prompt/workflow design guidance

## Key Technique

**Agentic multi-round inspection beats single-shot prompting.** The core insight from this paper is that LLM agents that can iteratively read files, search code, and reason across multiple rounds dramatically outperform vanilla (single-prompt) LLM classification -- but only when backed by a strong model (Claude Sonnet 4 or GPT-5). With weaker backbones, the agentic overhead yields inconsistent gains. The practical takeaway: invest in multi-step code traversal when you have a capable model; fall back to careful single-prompt analysis otherwise.

**Cross-file dataflow tracing is the critical differentiator.** In the study, agents read at least one non-target file in 51.2% of cases. False positives often involve: (1) values that appear tainted but are actually hardcoded constants, (2) sanitization that happens in a helper class outside the flagged file, (3) configuration-level protections (e.g., HTTP-only cookie flags set in web.xml rather than in code). Single-file analysis misses these, leading to both missed FPs and false confidence.

**CWE category matters for strategy selection.** Some vulnerability classes (CWE-79 XSS, CWE-89 SQLi) require deep dataflow analysis across request handlers, while others (CWE-330 weak randomness, CWE-614 secure cookie) can often be resolved by checking a single configuration file or constant value. The agent should adapt its investigation depth based on the CWE.

## Step-by-Step Workflow

1. **Parse the SAST alert.** Extract the tool name, rule ID, CWE category, file path, line number, and alert message. If multiple alerts exist, group them by file to avoid redundant file reads.

2. **Establish the security reviewer persona.** Frame the analysis as: "You are a security engineer reviewing static analysis findings. Your task is to determine if each alert is a true positive (exploitable vulnerability) or a false positive (safe code incorrectly flagged)."

3. **Read the flagged file in full.** Do not rely on just the flagged line. Read the entire file to understand the method context, parameter sources, and any local sanitization.

4. **Identify the dataflow question.** For each alert, articulate the specific dataflow question: Where does the tainted input originate? Does it pass through any sanitizer, validator, or encoding function before reaching the sink? Is the input actually user-controlled or is it a constant/config value?

5. **Trace cross-file dependencies.** Search for imports, helper methods, utility classes, and configuration files referenced by the flagged code. Key searches include:
   - Grep for the method name of any sanitizer/encoder called on the tainted value
   - Read web.xml, application.properties, or framework config for security settings
   - Check if the flagged method is only called from test code (not production)

6. **Evaluate CWE-specific heuristics.** Apply category-appropriate reasoning:
   - **Injection (CWE-78/89/90/643):** Is the input parameterized or concatenated? Are prepared statements used?
   - **XSS (CWE-79):** Is output encoding applied? Does the framework auto-escape?
   - **Path traversal (CWE-22):** Is the path validated/canonicalized?
   - **Crypto (CWE-327/328):** Which algorithm is used? Is it contextually appropriate (e.g., SHA-256 for checksums vs. passwords)?
   - **Randomness (CWE-330):** Is SecureRandom used? Is the random value security-sensitive?
   - **Cookie (CWE-614):** Check both code-level and config-level secure flag settings
   - **Trust boundary (CWE-501):** Is the value validated before being placed in a trusted scope?

7. **Render a verdict for each alert.** Produce a structured classification:
   - **Verdict:** TRUE POSITIVE or FALSE POSITIVE
   - **Confidence:** HIGH / MEDIUM / LOW
   - **Reasoning:** 2-3 sentence explanation citing specific code evidence
   - **Evidence files:** List of files examined beyond the flagged file

8. **Flag uncertain cases explicitly.** When confidence is LOW, mark the alert for human review rather than suppressing it. Aggressive FP filtering risks suppressing true vulnerabilities (the paper documents this trade-off).

9. **Produce a summary report.** Aggregate results: total alerts reviewed, FPs identified, TPs confirmed, uncertain cases, and per-CWE breakdown.

10. **Recommend suppression actions.** For confirmed FPs, suggest the appropriate suppression mechanism for the tool (e.g., `// codeql[rule-id]` inline suppression, `.semgrepignore`, SonarQube issue resolution).

## Concrete Examples

**Example 1: SQL Injection alert in a Java Spring application**

User: "CodeQL flagged SQL injection at `UserDAO.java:47`. Is this real?"

Approach:
1. Read `UserDAO.java` -- find that line 47 does `String query = "SELECT * FROM users WHERE id = " + userId`
2. Trace `userId` parameter: it comes from `UserService.getUser(int id)` which receives it from `UserController`
3. Check `UserController` -- the `id` parameter has `@PathVariable int id`, meaning Spring parses it as an integer
4. Integer path variables cannot contain SQL metacharacters -- the type constraint acts as implicit sanitization

Output:
```
| Alert | File | CWE | Verdict | Confidence | Reasoning |
|-------|------|-----|---------|------------|-----------|
| java/sql-injection | UserDAO.java:47 | CWE-89 | FALSE POSITIVE | HIGH | The `userId` parameter is typed as `int` at the controller level (@PathVariable int id). Integer values cannot carry SQL injection payloads. While string concatenation into SQL is a code smell, it is not exploitable here. |

Evidence files examined: UserDAO.java, UserService.java, UserController.java
Recommendation: Suppress with `// lgtm[java/sql-injection]`. Consider refactoring to PreparedStatement for defense-in-depth.
```

**Example 2: Batch triage of Semgrep findings**

User: "Semgrep found 12 XSS warnings across our Thymeleaf templates. Most look like noise. Can you triage?"

Approach:
1. Read each flagged template file and identify the expression syntax
2. Check if Thymeleaf's `th:text` (auto-escaping) vs `th:utext` (raw HTML) is used
3. For `th:utext` cases, trace the model attribute back to the controller to check if it contains user input
4. Check for any custom Thymeleaf dialect that adds sanitization

Output:
```
## Triage Summary: 12 Semgrep XSS Alerts

- FALSE POSITIVE: 9 alerts (all use th:text which auto-escapes by default)
- TRUE POSITIVE: 2 alerts (th:utext with user-controlled content in comment-display.html:23, profile-bio.html:41)
- NEEDS REVIEW: 1 alert (th:utext in admin-panel.html:88 -- content from DB, unclear if user-sourced)

### Action Items
1. Fix comment-display.html:23 -- switch to th:text or apply OWASP Java HTML Sanitizer
2. Fix profile-bio.html:41 -- sanitize bio HTML on write path
3. Manual review: admin-panel.html:88 -- verify data provenance
```

**Example 3: Weak cryptography finding in a non-security context**

User: "SonarQube reports CWE-328 (weak hashing) because we use MD5 in `CacheKeyGenerator.java`. Is this a real issue?"

Approach:
1. Read `CacheKeyGenerator.java` -- find MD5 is used to generate cache keys from request parameters
2. Verify the hash output is never used for authentication, password storage, or integrity verification
3. Confirm the cache key is internal-only and not exposed to users

Output:
```
| Alert | File | CWE | Verdict | Confidence | Reasoning |
|-------|------|-----|---------|------------|-----------|
| java:S4790 | CacheKeyGenerator.java:31 | CWE-328 | FALSE POSITIVE | HIGH | MD5 is used solely for cache key generation, not for security purposes (password hashing, integrity verification, or digital signatures). Cache keys are internal identifiers with no cryptographic requirements. MD5's collision properties are acceptable for this use case. |

Recommendation: Suppress with @SuppressWarnings("java:S4790") and add code comment explaining the non-security usage.
```

## Best Practices

- **Do:** Read the full file, not just the flagged line. Context within the same method and class is essential for accurate classification.
- **Do:** Always trace at least one hop beyond the flagged file. Cross-file sanitization, type constraints, and configuration are the most common sources of false positives.
- **Do:** Adapt investigation depth to CWE category. Injection flaws (CWE-78/89/79) need full dataflow tracing; configuration issues (CWE-614, CWE-330) often resolve with a single config file check.
- **Do:** State your confidence level. A LOW-confidence FP classification should go to human review, not be silently suppressed.
- **Avoid:** Classifying an alert as FP solely because the code "looks fine" at the flagged line. The paper shows that single-file analysis has substantially higher error rates.
- **Avoid:** Assuming all alerts from a single rule are uniformly FP or TP. Even within the same CWE, individual instances depend on their specific dataflow context.
- **Avoid:** Aggressively suppressing alerts to minimize noise at the cost of missing true positives. The paper explicitly documents that the lowest FP rate configurations also suppress some real vulnerabilities. Preserving TP recall is more important than achieving the lowest FP count.

## Error Handling

- **Incomplete source code:** If referenced files are unavailable (e.g., compiled dependencies, third-party libraries), note this in the verdict and lower confidence. Do not assume missing code is safe.
- **Ambiguous dataflow:** When input sources cross module boundaries or use reflection/dynamic dispatch, mark as NEEDS REVIEW rather than guessing.
- **Tool-specific false patterns:** Some SAST tools have known rule deficiencies (e.g., CodeQL may not model certain framework sanitizers). If a finding matches a known tool limitation, note it but still verify independently.
- **Large alert volumes:** When triaging 50+ alerts, prioritize by CWE severity (injection > crypto > config) and group identical rule violations in the same file for batch analysis.

## Limitations

- **Language scope:** The paper validates on Java projects specifically. The methodology transfers to other languages, but CWE-specific heuristics (e.g., PreparedStatement for SQLi, Thymeleaf auto-escaping) need language/framework adaptation.
- **Model dependency:** Multi-round agentic analysis yields strong results with capable models (Claude Sonnet 4, GPT-5) but inconsistent results with weaker models. With a less capable backbone, prefer a thorough single-prompt analysis with full context over multi-round iteration.
- **Cost trade-off:** Agentic approaches cost 3-4x more per alert than single-prompt analysis (study reports $0.047-$0.187 per alert with Claude). For large-scale triage (thousands of alerts), consider a two-phase approach: cheap single-prompt pre-filter followed by agentic deep analysis on uncertain cases.
- **True positive suppression risk:** The best FP reduction configurations also misclassify some real vulnerabilities as false positives. Never use automated FP filtering as the sole gate for security-critical code paths.
- **SAST tool coverage:** Results depend on the specific SAST tool's rule set and alert format. The approach was validated with CodeQL, Semgrep, SonarQube, and Joern; other tools may require prompt adaptation.

## Reference

Xiong, Y. & Zhang, T. (2026). *Sifting the Noise: A Comparative Study of LLM Agents in Vulnerability False Positive Filtering.* arXiv:2601.22952v1. Key takeaway: agentic LLM frameworks with cross-file code navigation reduce SAST false positive rates from 92%+ to 6.3%, but effectiveness is backbone-dependent and requires CWE-aware strategy selection.