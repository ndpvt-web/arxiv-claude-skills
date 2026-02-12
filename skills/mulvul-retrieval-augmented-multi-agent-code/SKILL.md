---
name: "mulvul-retrieval-augmented-multi-agent-code"
description: "Multi-agent vulnerability detection using coarse-to-fine routing, contrastive retrieval, and cross-model prompt evolution. Use when: 'scan this code for vulnerabilities', 'detect CWE weaknesses in this function', 'security audit this C/C++ code', 'find buffer overflows and injection flaws', 'analyze this code for common weakness enumerations', 'run a multi-agent vulnerability check on this codebase'."
---

# MulVul: Retrieval-Augmented Multi-Agent Code Vulnerability Detection

This skill enables Claude to perform structured, multi-agent vulnerability detection on source code using the MulVul framework. Instead of scanning code with a single monolithic prompt, Claude applies a coarse-to-fine routing strategy: first classifying code into broad vulnerability categories (Memory Buffer Errors, Injection, Logic Errors, etc.), then dispatching specialized detection passes for each candidate category. Each pass uses contrastive retrieval -- pulling in similar vulnerable code, clean code, and hard-negative examples -- to ground its analysis in concrete evidence rather than parametric guessing. The result is broader coverage across CWE types with significantly fewer hallucinated findings.

## When to Use

- When the user asks to scan a function or file for security vulnerabilities and wants structured CWE-level findings
- When reviewing C/C++ code (or any language) for memory safety, injection, input validation, or cryptographic flaws
- When the user wants a security audit that goes beyond surface-level pattern matching and covers 10+ vulnerability categories systematically
- When triaging code changes in a pull request for potential security regressions
- When the user asks to detect specific CWE types (e.g., CWE-119 Buffer Overflow, CWE-89 SQL Injection) across a codebase
- When a single-pass vulnerability scan produces too many false positives and the user wants higher-precision results
- When analyzing code with known class-imbalance problems (rare vulnerability types mixed with common ones)

## Key Technique: Coarse-to-Fine Routing with Contrastive Retrieval

MulVul's core insight is that vulnerability detection fails when a single model tries to handle all 130+ CWE types simultaneously. Different vulnerability classes have fundamentally different code patterns -- a buffer overflow looks nothing like a cryptographic flaw. By routing code through category-specific detectors, each agent can focus on a narrow domain with specialized reasoning.

The framework operates in two phases. **Phase 1 (Router):** A Router agent examines the code and predicts the top-k (typically k=3) coarse vulnerability categories from a taxonomy of 10 groups: Memory Buffer Errors, Injection, Logic Errors, Input Validation, Cryptographic Flaws, Resource Management, Access Control, Information Disclosure, Numeric Errors, and API Misuse. The Router uses globally-retrieved examples spanning all categories to make this coarse prediction. **Phase 2 (Detectors):** For each predicted category, a specialized Detector agent analyzes the code using contrastive retrieval -- it gathers three types of evidence: (1) in-category vulnerable examples (positives), (2) clean non-vulnerable code (negatives), and (3) out-of-category vulnerable code that looks superficially similar (hard negatives). This contrastive context prevents confirmation bias and forces the detector to justify why this specific code matches this specific weakness.

The second key innovation is **Cross-Model Prompt Evolution**. Rather than hand-writing prompts for each of the 10+ detector categories, MulVul uses one LLM (the generator) to propose and mutate detection prompts while a different LLM (the executor) evaluates them on real code samples. This decoupling prevents the self-correction bias where a single model reinforces its own blind spots. The evolved prompts consistently include negative constraints ("Do NOT infer vulnerabilities beyond these patterns"), disambiguation rules ("Injection executes instructions; Input flaws only mishandle data"), and explicit signal definitions for each category.

## Step-by-Step Workflow

1. **Structure the code for analysis.** Extract each function or logical unit from the input. For each unit, identify: function signature, parameter types, memory operations, external input sources, control flow branches, and library calls. This structured representation (analogous to MulVul's SCALE format) makes retrieval and comparison more effective.

2. **Run the Router pass -- predict top-k coarse categories.** Analyze the structured code against 10 coarse vulnerability categories. Act as a senior security analyst. For each category, check for explicit signals:
   - **Memory Buffer Errors:** Direct memory manipulation (malloc, memcpy, pointer arithmetic) without bounds checks
   - **Injection:** String concatenation into commands, queries, or interpreters without sanitization
   - **Logic Errors:** Flawed conditionals, missing state transitions, race conditions
   - **Input Validation:** Missing or incomplete validation of external inputs
   - **Cryptographic Flaws:** Weak algorithms, hardcoded keys, improper random number generation
   - **Resource Management:** Unclosed handles, leaked memory, missing cleanup in error paths
   - **Access Control:** Missing authorization checks, privilege escalation paths
   - **Information Disclosure:** Sensitive data in logs, error messages, or unprotected storage
   - **Numeric Errors:** Integer overflow/underflow, signed/unsigned confusion, truncation
   - **API Misuse:** Incorrect argument order, ignored return values, deprecated function usage

   Select the top 3 categories with the strongest signal. If no strong pattern matches, default to "Benign."

3. **For each predicted category, gather contrastive evidence.** Search the codebase (or your knowledge of common vulnerability patterns) for three types of examples:
   - **Positive examples:** Known vulnerable code patterns matching this category (e.g., for Memory Buffer Errors: a classic `strcpy` without length check)
   - **Negative examples:** Similar-looking but safe code (e.g., `strncpy` with proper bounds)
   - **Hard negatives:** Code that triggers a different category but looks superficially similar (e.g., unsanitized input that's an Input Validation issue, not Injection)

4. **Run each Detector pass with contrastive context.** For each of the top-k categories, analyze the code with the category-specific lens. Include the contrastive examples as grounding context. Apply these constraints:
   - Only flag vulnerabilities with concrete evidence in the code -- no speculation about hypothetical execution paths
   - Distinguish between the current category and adjacent categories using disambiguation rules
   - Map findings to specific CWE IDs where possible (e.g., CWE-119 for buffer overflow, CWE-125 for out-of-bounds read)
   - Provide the exact code location (line numbers, function names) for each finding

5. **Aggregate and deduplicate findings.** Merge results from all Detector passes. Remove duplicates where multiple detectors flagged the same code location. Rank findings by severity (Critical > High > Medium > Low) and confidence.

6. **Apply the benign default rule.** If no detector produced any findings with sufficient confidence, classify the code as benign. Do not force-fit vulnerabilities. False negatives on rare types are preferable to hallucinated findings.

7. **Generate structured output.** For each finding, produce: CWE ID, category, severity, affected code location, explanation of the vulnerability mechanism, a contrastive justification (why this is X and not Y), and a concrete fix recommendation.

8. **Self-check with negative constraints.** Review each finding against these evolved-prompt rules:
   - "Do NOT infer vulnerabilities beyond the patterns present in the code"
   - "Do NOT speculate about hypothetical vulnerabilities in code paths not shown"
   - "Do NOT conflate similar-sounding categories" (e.g., Injection vs. Input Validation)
   - Remove any finding that violates these constraints

## Concrete Examples

**Example 1: Buffer Overflow Detection in C**

User: "Scan this function for vulnerabilities"
```c
void process_input(char *user_data) {
    char buffer[64];
    strcpy(buffer, user_data);
    printf("Processed: %s\n", buffer);
}
```

Approach:
1. **Router pass:** Identifies Memory Buffer Errors (strcpy with no bounds) and Input Validation (user_data unchecked) as top categories
2. **Contrastive retrieval:** Compares against safe pattern `strncpy(buffer, user_data, sizeof(buffer) - 1)` and hard negative where input is validated but still used in `sprintf` (different category)
3. **Detector pass (Memory Buffer):** Confirms CWE-119 / CWE-120 -- `strcpy` copies unbounded input into fixed 64-byte buffer
4. **Detector pass (Input Validation):** No independent input validation issue -- the root cause is the unbounded copy, not missing validation per se. Avoids double-counting.

Output:
```
Finding 1:
  CWE: CWE-120 (Buffer Copy without Checking Size of Input)
  Category: Memory Buffer Errors
  Severity: Critical
  Location: line 3, strcpy(buffer, user_data)
  Mechanism: strcpy copies user_data into a 64-byte stack buffer with no
    length check. If user_data exceeds 63 bytes, this overwrites adjacent
    stack memory, enabling code execution.
  Contrastive justification: This is a Memory Buffer Error (unbounded copy),
    not an Input Validation issue (the input itself may be valid data --
    the flaw is the copy operation).
  Fix: Replace with strncpy(buffer, user_data, sizeof(buffer) - 1);
       buffer[sizeof(buffer) - 1] = '\0';
```

**Example 2: SQL Injection in Python**

User: "Check this endpoint for security issues"
```python
@app.route('/users')
def get_users():
    name = request.args.get('name')
    query = f"SELECT * FROM users WHERE name = '{name}'"
    result = db.execute(query)
    return jsonify(result.fetchall())
```

Approach:
1. **Router pass:** Top categories -- Injection (string interpolation into SQL), Input Validation (unvalidated query param), Access Control (no auth check on endpoint)
2. **Detector pass (Injection):** Confirms CWE-89 -- f-string directly interpolates user input into SQL
3. **Detector pass (Input Validation):** The missing validation enables the injection but is not a standalone finding -- it's the same root cause
4. **Detector pass (Access Control):** No authentication decorator, but this may be intentional for a public endpoint -- flag as informational only

Output:
```
Finding 1:
  CWE: CWE-89 (SQL Injection)
  Category: Injection
  Severity: Critical
  Location: line 4, f"SELECT * FROM users WHERE name = '{name}'"
  Mechanism: User-controlled request parameter 'name' is interpolated
    directly into a SQL query string. An attacker can inject arbitrary SQL
    (e.g., name=' OR '1'='1) to dump or modify the database.
  Contrastive justification: This is Injection (user input becomes part of
    an executed query), not merely Input Validation (the issue is not
    malformed data but executed instructions).
  Fix: Use parameterized queries: db.execute(
       "SELECT * FROM users WHERE name = ?", (name,))

Finding 2 (Informational):
  CWE: CWE-862 (Missing Authorization)
  Category: Access Control
  Severity: Low (informational -- may be intentional)
  Location: line 1-2, @app.route('/users') with no auth decorator
  Mechanism: Endpoint is accessible without authentication. If this returns
    sensitive user data, it should require authorization.
  Fix: Add @login_required or equivalent auth middleware if endpoint
       should be protected.
```

**Example 3: Multi-file Codebase Scan**

User: "Run a security audit on the authentication module"

Approach:
1. Identify all files in the authentication module (handlers, middleware, token logic, password storage)
2. For each function, run the Router pass to identify candidate categories
3. Focus Detector passes on the categories most relevant to auth code: Cryptographic Flaws (password hashing, token generation), Access Control (authorization checks), Information Disclosure (token leakage in logs), Input Validation (credential format checks)
4. Aggregate findings across all files, grouping by category and severity
5. Produce a summary table plus detailed per-finding output

Output format:
```
Security Audit: Authentication Module
======================================
Files analyzed: 6
Functions analyzed: 23
Findings: 4 (1 Critical, 2 High, 1 Medium)

| # | CWE     | Severity | File           | Line | Summary                    |
|---|---------|----------|----------------|------|----------------------------|
| 1 | CWE-327 | Critical | auth/hash.py   | 14   | MD5 used for password hash |
| 2 | CWE-798 | High     | auth/config.py | 3    | Hardcoded JWT secret       |
| 3 | CWE-532 | High     | auth/login.py  | 45   | Password logged on failure |
| 4 | CWE-307 | Medium   | auth/login.py  | 22   | No brute-force protection  |

[Detailed findings follow...]
```

## Best Practices

- **Do:** Always run the Router pass first before deep-diving into specific vulnerability types. Skipping coarse classification leads to tunnel vision on one category while missing others.
- **Do:** Include contrastive examples when explaining findings. Showing why code is vulnerable (and what the safe version looks like) is far more useful than just flagging a line number.
- **Do:** Apply the benign default. If you cannot find concrete evidence of a vulnerability, say so. MulVul's key design principle is that no finding is better than a hallucinated finding.
- **Do:** Use disambiguation rules between adjacent categories. Injection vs. Input Validation, Memory Errors vs. Numeric Errors, and Access Control vs. Logic Errors are the most commonly confused pairs.
- **Avoid:** Reporting the same root cause as multiple findings across different categories. Deduplicate by identifying the primary CWE and noting related weaknesses as secondary.
- **Avoid:** Speculating about vulnerabilities in code paths not present in the input. Only analyze what you can see. If critical context is missing, state that explicitly rather than guessing.

## Error Handling

- **Incomplete code context:** If the function references external functions or types not provided, note the assumption explicitly (e.g., "Assuming `sanitize()` does not perform SQL escaping based on its name") and flag the finding as conditional.
- **Ambiguous category assignment:** When code could belong to multiple categories (e.g., integer overflow leading to buffer overflow), assign the primary category based on the root cause (Numeric Error) and note the downstream impact (Memory Buffer Error) in the explanation.
- **No vulnerabilities found:** This is a valid and expected outcome. Report it clearly: "No vulnerabilities detected in the analyzed code. The Router identified [categories] as candidates, but Detector analysis found no concrete evidence of exploitable weaknesses."
- **Language-specific gaps:** MulVul was evaluated primarily on C/C++. When analyzing other languages, adapt category signals accordingly (e.g., Memory Buffer Errors are less relevant in Python/Java but Resource Management and Injection remain critical).

## Limitations

- The coarse-to-fine approach depends on the Router correctly including the true category in its top-k predictions. If the Router misclassifies, the relevant Detector never runs. For safety-critical audits, consider increasing k or running all Detectors.
- Rare vulnerability types (the long tail of CWE types with few known examples) have inherently lower detection confidence. Be transparent when a finding maps to a rare CWE.
- This approach works best on function-level analysis. Vulnerabilities that span multiple functions, files, or require understanding the full application data flow (e.g., TOCTOU races across threads) may not be caught by single-function analysis.
- Without an actual indexed knowledge base of vulnerable code samples, Claude approximates the retrieval step using its training knowledge. A production MulVul system with a FAISS-indexed CWE knowledge base will have stronger contrastive grounding.

## Reference

[MulVul: Retrieval-augmented Multi-Agent Code Vulnerability Detection via Cross-Model Prompt Evolution](https://arxiv.org/abs/2601.18847v1) -- Wu et al., 2026. Key sections: Section 3 (framework architecture with Router/Detector agents), Section 3.3 (contrastive retrieval with positive/negative/hard-negative pools), Section 3.4 (cross-model prompt evolution with generator/executor LLM decoupling), and Table 2 (results across 130 CWE types demonstrating 41.5% improvement over baselines).