---
name: "secure-code-generation-via"
description: |
  Generates secure, vulnerability-free code by applying the SecCoderX reasoning framework — systematically analyzing code for CWE-classified vulnerabilities while preserving full functionality. Combines vulnerability-aware task decomposition with a reasoning-based security audit pass on every code output.
  Trigger phrases:
  - "Write secure code for..."
  - "Generate code without vulnerabilities"
  - "Security-hardened implementation of..."
  - "Check this code for CWE vulnerabilities"
  - "Fix the security issues in this code"
  - "Write this function with security best practices"
---

# Secure Code Generation via Vulnerability Reasoning (SecCoderX)

This skill enables Claude to generate code that is both functionally correct and free of common security vulnerabilities, applying the SecCoderX framework's core insight: use structured vulnerability reasoning as an inline verification step during code generation. Rather than writing code and hoping it's secure, Claude decomposes every coding task through a security lens — identifying which CWE weakness classes are relevant to the task, generating code with explicit mitigations, then self-auditing the output with a reasoning chain that checks each vulnerability surface before delivering the result.

## When to Use

- When the user asks to write any code that handles user input (forms, APIs, CLI arguments, file uploads)
- When generating database queries, ORM code, or any data-access layer
- When writing authentication, authorization, session management, or credential-handling code
- When the user asks to "make this code secure" or "fix vulnerabilities" in existing code
- When implementing file I/O, process spawning, or system command execution
- When writing cryptographic operations, token generation, or secret management
- When the user requests a security review or CWE audit of a code snippet
- When generating web handlers, middleware, or API endpoints that process external data

## Key Technique: Vulnerability-Aware Reasoning Before Code

The SecCoderX paper identifies a critical problem: when LLMs are pushed to generate "secure" code through naive alignment, they often break functionality — refusing to implement features, adding excessive guards that change behavior, or stripping logic to avoid risk. This **functionality-security paradox** means that prior methods degrade Effective Safety Rate (ESR — the percentage of outputs that are both secure AND functionally correct) by 14-54%.

SecCoderX solves this through two innovations. First, it builds a **vulnerability-inducing task taxonomy** grounded in real CWE/CVE data. Instead of generic "write secure code" instructions, it maps each coding task to the specific CWE weakness classes that are relevant — SQL injection (CWE-89) for database code, buffer overflow (CWE-120) for C/C++ memory operations, XSS (CWE-79) for web output, OS command injection (CWE-78) for subprocess calls, and so on. This targeted scoping prevents over-correction.

Second, it trains a **reasoning-based vulnerability reward model** that doesn't just flag "vulnerable or not" but produces a chain-of-thought explaining *which* vulnerability surfaces exist, *whether* the code addresses them, and *how* each mitigation preserves the intended functionality. This skill replicates that reasoning chain as an explicit step in Claude's code generation workflow — identify applicable CWEs, write code with targeted mitigations, then self-audit with a structured security reasoning pass.

## Step-by-Step Workflow

1. **Classify the task's vulnerability surface.** Before writing any code, identify which CWE categories are relevant to the task. Map the request to specific weakness classes:
   - User input handling → CWE-89 (SQL Injection), CWE-79 (XSS), CWE-78 (OS Command Injection)
   - Memory operations (C/C++) → CWE-120 (Buffer Overflow), CWE-416 (Use After Free), CWE-476 (NULL Pointer Dereference)
   - Authentication → CWE-287 (Improper Authentication), CWE-798 (Hard-coded Credentials)
   - Cryptography → CWE-327 (Broken Crypto), CWE-330 (Insufficient Randomness)
   - File operations → CWE-22 (Path Traversal), CWE-377 (Insecure Temp File)
   - Web output → CWE-79 (XSS), CWE-352 (CSRF)
   - Deserialization → CWE-502 (Deserialization of Untrusted Data)

2. **Write the functionally correct implementation first.** Produce code that fully satisfies the user's requirements. Do not omit features or add unnecessary restrictions in the name of security.

3. **Apply targeted mitigations for each identified CWE.** For each vulnerability class from step 1, apply the specific mitigation pattern:
   - CWE-89: Use parameterized queries or prepared statements. Never interpolate user input into SQL strings.
   - CWE-79: Escape output contextually (HTML entity encoding for HTML body, JavaScript encoding for script contexts, URL encoding for URLs).
   - CWE-78: Use language-native APIs (e.g., `subprocess.run([...])` with list arguments) instead of shell string concatenation.
   - CWE-120: Use bounds-checked functions (`strncpy`, `snprintf`), or use safe abstractions (std::string, Vec).
   - CWE-22: Canonicalize paths and validate they fall within an allowed directory before file operations.
   - CWE-287: Use constant-time comparison for secrets, enforce password complexity, implement rate limiting.
   - CWE-327: Use current standard algorithms (AES-256-GCM, SHA-256+, Ed25519). Never use MD5/SHA1 for security purposes.
   - CWE-330: Use cryptographically secure random generators (`secrets` in Python, `crypto/rand` in Go, `SecureRandom` in Java).

4. **Perform the security reasoning audit.** After writing the code, run a structured self-check for each applicable CWE. For each one, state: (a) whether the vulnerability surface exists in this code, (b) what specific mitigation is applied, and (c) confirm the mitigation does not alter functional behavior.

5. **Verify input validation at trust boundaries.** Identify every point where external data enters the system (HTTP parameters, file reads, environment variables, database results from user-controlled queries). Confirm each boundary has appropriate validation or sanitization.

6. **Check for information leakage.** Ensure error messages, logs, and responses do not expose stack traces, internal paths, database schemas, or credential material to external callers.

7. **Validate resource management.** Confirm that all acquired resources (file handles, database connections, memory allocations, locks) are released in all code paths including error paths. Use language idioms: `with` in Python, `defer` in Go, try-with-resources in Java, RAII in C++.

8. **Present the code with inline security annotations.** Add brief comments at each security-critical point explaining the mitigation choice, referencing the CWE number. Keep annotations minimal — one line per mitigation point.

9. **Summarize the security posture.** After the code block, provide a brief table or list mapping each identified CWE to the mitigation applied, confirming both security and functionality preservation.

## Concrete Examples

**Example 1: Secure Database Query Endpoint**

User: "Write a Python Flask endpoint that searches users by name and returns JSON results."

Approach:
1. Classify: HTTP input → CWE-89 (SQL Injection), CWE-79 (XSS in JSON less likely but check), CWE-200 (Information Exposure)
2. Write functional implementation with parameterized queries
3. Apply mitigations: parameterized query for CWE-89, input length limit, no stack trace leakage
4. Self-audit reasoning chain

Output:
```python
from flask import Flask, request, jsonify
import sqlite3

app = Flask(__name__)

@app.route("/api/users/search")
def search_users():
    name = request.args.get("name", "")

    # CWE-89 mitigation: limit input length at trust boundary
    if len(name) > 200:
        return jsonify({"error": "Query too long"}), 400

    try:
        conn = sqlite3.connect("app.db")
        cursor = conn.cursor()
        # CWE-89 mitigation: parameterized query — never interpolate user input
        cursor.execute(
            "SELECT id, name, email FROM users WHERE name LIKE ?",
            (f"%{name}%",),
        )
        rows = cursor.fetchall()
        results = [{"id": r[0], "name": r[1], "email": r[2]} for r in rows]
    except sqlite3.Error:
        # CWE-200 mitigation: generic error, no internal details exposed
        return jsonify({"error": "Search failed"}), 500
    finally:
        conn.close()  # Resource cleanup in all paths

    return jsonify({"results": results})
```

Security summary:
| CWE | Surface | Mitigation |
|-----|---------|------------|
| CWE-89 | User-supplied `name` in SQL | Parameterized query with `?` placeholder |
| CWE-200 | Exception details | Generic error message, no traceback |
| Resource mgmt | DB connection | `finally` block ensures `conn.close()` |

---

**Example 2: Secure File Download Handler**

User: "Write a Node.js Express handler that serves files from an uploads directory based on a filename parameter."

Approach:
1. Classify: User-controlled filename → CWE-22 (Path Traversal), CWE-200 (Information Exposure)
2. Write functional handler with path canonicalization
3. Self-audit for traversal bypass techniques

Output:
```javascript
const express = require("express");
const path = require("path");
const fs = require("fs");

const app = express();
const UPLOADS_DIR = path.resolve(__dirname, "uploads");

app.get("/download/:filename", (req, res) => {
  const filename = req.params.filename;

  // CWE-22 mitigation: reject path separators and traversal sequences
  if (filename.includes("/") || filename.includes("\\") || filename.includes("..")) {
    return res.status(400).json({ error: "Invalid filename" });
  }

  // CWE-22 mitigation: resolve and verify canonical path is within UPLOADS_DIR
  const filePath = path.resolve(UPLOADS_DIR, filename);
  if (!filePath.startsWith(UPLOADS_DIR + path.sep)) {
    return res.status(403).json({ error: "Access denied" });
  }

  if (!fs.existsSync(filePath)) {
    // CWE-200 mitigation: don't reveal directory structure
    return res.status(404).json({ error: "File not found" });
  }

  res.sendFile(filePath);
});
```

Security summary:
| CWE | Surface | Mitigation |
|-----|---------|------------|
| CWE-22 | User-controlled `filename` | Reject traversal chars + canonical path prefix check |
| CWE-200 | Missing file | Generic 404, no path information leaked |

---

**Example 3: Fixing Insecure Existing Code**

User: "Fix the security issues in this Python code:"
```python
import os
def run_report(user_input):
    os.system(f"generate_report --name {user_input} --output /tmp/report.pdf")
    return open(f"/tmp/{user_input}_log.txt").read()
```

Approach:
1. Classify: `os.system` with string interpolation → CWE-78 (OS Command Injection); unvalidated filename → CWE-22 (Path Traversal); unclosed file handle
2. Replace `os.system` with `subprocess.run` using list arguments
3. Sanitize filename and validate path
4. Self-audit the fixed version

Output:
```python
import subprocess
import os
import re

def run_report(user_input: str) -> str:
    # CWE-78 mitigation: validate input contains only safe characters
    if not re.match(r"^[a-zA-Z0-9_\-]+$", user_input):
        raise ValueError("Invalid report name")

    # CWE-78 mitigation: use subprocess with list args, no shell interpretation
    result = subprocess.run(
        ["generate_report", "--name", user_input, "--output", "/tmp/report.pdf"],
        capture_output=True,
        check=True,
    )

    # CWE-22 mitigation: construct path safely and verify it's within /tmp
    log_path = os.path.realpath(os.path.join("/tmp", f"{user_input}_log.txt"))
    if not log_path.startswith("/tmp/"):
        raise ValueError("Invalid log path")

    # Resource management: use context manager for file handle
    with open(log_path) as f:
        return f.read()
```

Vulnerabilities fixed:
| Original Issue | CWE | Fix Applied |
|---------------|-----|-------------|
| `os.system(f"...")` with user input | CWE-78 | `subprocess.run` with list args, input validation |
| Unvalidated path in `open()` | CWE-22 | `realpath` + prefix check |
| Unclosed file handle | Resource leak | `with` context manager |

## Best Practices

- **Do:** Map every coding task to specific CWE numbers before writing code. This targeted approach prevents both under-protection and the functionality-security paradox where over-aggressive hardening breaks features.
- **Do:** Apply mitigations at the trust boundary closest to where external data enters. Sanitize once at entry, then trust the sanitized value downstream.
- **Do:** Use language-native safe APIs over manual sanitization. Parameterized queries beat escaping. `subprocess` with list args beats shell escaping. Template engines with auto-escaping beat manual HTML encoding.
- **Do:** Keep the security reasoning audit visible. Briefly state which CWEs were considered and how each is addressed, so the user can verify the reasoning.
- **Avoid:** Adding authentication, rate limiting, or access control unless the user asks for it. The goal is to make the requested code secure, not to redesign the system architecture.
- **Avoid:** Refusing to implement functionality because it "could be insecure." Instead, implement it with the correct mitigation. A parameterized SQL query is not a security risk — it's the solution.
- **Avoid:** Using deprecated or weak security primitives even as fallbacks (MD5, SHA1 for hashing passwords, ECB mode, `Math.random()` for tokens).

## Error Handling

- **User provides incomplete context:** If the code's deployment context is unclear (e.g., is this an internal tool or public-facing?), ask the user. The threat model determines which CWEs are most critical.
- **Conflicting requirements:** If the user explicitly requests an insecure pattern (e.g., "use string concatenation for the SQL query"), explain the specific vulnerability, provide the secure alternative, and let the user decide. Never silently make it insecure.
- **Language-specific gaps:** Some languages lack certain safe APIs. Document the limitation and use the best available alternative (e.g., C lacks parameterized queries natively — use the database driver's prepared statement API).
- **False positive on vulnerability surface:** If the security audit flags something that isn't actually exploitable in context (e.g., a "hardcoded string" that is a SQL table name, not user input), note why it's safe rather than adding unnecessary mitigation.

## Limitations

- This approach addresses code-level vulnerabilities (OWASP/CWE categories). It does not cover infrastructure security, network configuration, or deployment hardening.
- The CWE classification step depends on correctly identifying trust boundaries. If the user doesn't describe how the code will be called, some vulnerability surfaces may not be identified. Ask when in doubt.
- Logic vulnerabilities (business logic flaws, race conditions in distributed systems, subtle authorization bypasses) require domain-specific knowledge that cannot be fully captured by CWE classification alone.
- This skill focuses on the top ~25 most common CWE categories. Esoteric or domain-specific vulnerability classes (e.g., automotive CAN bus injection, SCADA-specific flaws) are outside scope.
- The approach works best for single-function or single-module code. System-wide security architecture (e.g., designing a complete OAuth2 flow across microservices) benefits from dedicated architecture review beyond this skill's scope.

## Reference

**Paper:** [Secure Code Generation via Online Reinforcement Learning with Vulnerability Reward Model](https://arxiv.org/abs/2602.07422v1) (Wu et al., 2026). Key insight: mapping coding tasks to specific CWE vulnerability classes and using structured reasoning to verify mitigations preserves functionality while improving security — achieving ~10% ESR improvement where prior methods degraded ESR by 14-54%.