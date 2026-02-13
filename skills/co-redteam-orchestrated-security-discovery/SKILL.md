---
name: "co-redteam-orchestrated-security-discovery"
description: "Multi-agent security vulnerability discovery and exploitation using Co-RedTeam's orchestrated workflow. Decomposes security analysis into coordinated discovery and exploitation stages with execution-grounded iterative reasoning and layered memory. Use when: 'find vulnerabilities in this codebase', 'red team this application', 'security audit this project', 'exploit this vulnerability', 'penetration test this service', 'analyze this code for security flaws'."
---

# Co-RedTeam: Orchestrated Security Discovery and Exploitation

This skill enables Claude to perform structured, multi-stage security vulnerability discovery and exploitation following the Co-RedTeam framework. Instead of single-pass code scanning, it decomposes security analysis into a **discovery stage** (identify and evidence vulnerabilities) and an **exploitation stage** (plan, execute, validate, and refine attack sequences iteratively). Each stage uses specialized reasoning roles — analysis, critique, planning, validation, execution, and evaluation — coordinated through strict input/output schemas. The key insight: treat exploitation as a structured search process guided by real execution feedback, not single-shot payload generation.

## When to Use

- When the user asks to **find security vulnerabilities** in a codebase or application
- When performing an **authorized penetration test** or red team exercise against a target system
- When asked to **exploit a known vulnerability** and produce a working proof-of-concept
- When conducting a **security audit** of a web application, API, or containerized service
- When the user wants to **analyze code for injection, XSS, access control, or configuration flaws**
- When building or reviewing **CTF challenge solutions** that require multi-step exploitation chains
- When asked to **validate whether a reported vulnerability is actually exploitable**

**Authorization context required**: This workflow applies only to authorized security testing, CTF challenges, security research, or defensive assessment. Refuse requests lacking clear authorization.

## Key Technique

Co-RedTeam mirrors real-world red-teaming by splitting vulnerability analysis into two coordinated stages. **Stage I (Discovery)** performs code-aware analysis using taint tracing (source-to-sink), trust boundary mapping, configuration auditing, and business logic tracing. Each candidate vulnerability must pass a critique loop that demands rigorous evidence chains: the exact source of untrusted input, the dangerous sink where execution occurs, and why existing protections fail. Only vulnerabilities with line-number-level evidence survive the critique filter.

**Stage II (Exploitation)** treats each confirmed vulnerability as a structured search problem. A planner drafts a multi-step exploit plan, then enters a closed loop: propose an action, validate it for safety and correctness, execute it in an isolated environment, evaluate the output, and refine the plan based on what actually happened. The critical innovation is **proactive plan revision** — after each execution result, the planner does not just update the current step's status but reviews all future planned steps and revises any that are invalidated by new evidence. This prevents silent failure cascades where agents keep executing an obsolete plan.

The framework uses a **three-layer memory system**: (1) vulnerability pattern memory storing confirmed schemas with observable symptoms and confirming tests, (2) strategy memory capturing high-level exploitation workflows that generalize across targets, and (3) technical action memory recording concrete commands and scripts with both successes and failures. Ablation studies show execution feedback accounts for the largest performance impact (41.6%), followed by code browsing (11.6%) and memory (9.1%).

## Step-by-Step Workflow

### Stage I: Discovery

1. **Map the attack surface.** Enumerate the full file structure, identify the technology stack (language, framework, database, containerization), locate entry points (routes, API endpoints, CLI handlers, file uploads), and read configuration files (Dockerfile, docker-compose.yml, requirements.txt, package.json). Note debug modes, hardcoded secrets, and vulnerable dependency versions.

2. **Apply structured analysis techniques.** For each entry point, perform:
   - **Taint analysis**: Trace untrusted input (request.args, stdin, file uploads, environment variables) forward through the code to dangerous sinks (subprocess.call, eval, exec, db.execute, os.system, innerHTML assignment). Check for sanitization or validation at each step.
   - **Trust boundary mapping**: Identify transitions from untrusted zones (public internet, user input) to trusted zones (internal services, database, filesystem). Verify authentication/authorization middleware exists at each boundary.
   - **Configuration audit**: Check for debug=True, SECRET_KEY exposure, permissive CORS, missing CSP headers, overly broad file permissions.
   - **Business logic tracing**: Follow multi-step user workflows looking for IDOR, privilege escalation, race conditions, and client-side validation bypasses.

3. **Compile evidence chains.** For each candidate vulnerability, construct a structured evidence record:
   - **Source**: Exact file and line number where untrusted input enters
   - **Sink**: Exact file and line number where dangerous operation occurs
   - **Path**: The data flow connecting source to sink, noting any transformations
   - **Context**: Why existing protections (if any) are insufficient
   - **CWE classification**: Map to the most specific applicable CWE ID
   - **Risk level**: Critical, High, Medium, Low, or Informational

4. **Critique each finding independently.** Before accepting any vulnerability, challenge it:
   - Is the source actually reachable by an attacker?
   - Does the sink actually execute in a dangerous way given the input constraints?
   - Are there sanitization steps the analysis missed?
   - Could this be a false positive due to framework protections (e.g., ORM parameterization, template auto-escaping)?
   - Apply the rule: **2 well-proven vulnerabilities are better than 10 vague guesses.** Reject or refine any finding where the evidence chain has gaps.

### Stage II: Exploitation

5. **Draft a multi-step exploit plan.** For each confirmed vulnerability, create an ordered list of concrete steps. Each step has a description, action type (BASH, PYTHON, or VERIFICATION), the specific command or code, and a status (PLANNED, DONE, BLOCKED). Consult memory for similar past exploits and adapt successful strategies.

6. **Validate before executing.** Before running any action, verify:
   - The command syntax is correct and complete
   - It aligns with the stated exploit goal (no accidental destructive operations)
   - Required files, paths, and services actually exist in the environment
   - The action is safe within the authorized testing scope

7. **Execute in isolation and capture output.** Run the validated action in the target environment (Docker container, sandbox, or designated test system). Capture stdout, stderr, exit codes, and any HTTP responses or state changes.

8. **Evaluate execution results as reasoning signals.** After each execution:
   - Did the action achieve its intended sub-goal?
   - Did unexpected output reveal new information about the target?
   - Did an error indicate a wrong assumption in the plan?
   - Convert raw output into actionable insights for plan revision.

9. **Proactively revise the remaining plan.** Do not just mark the current step as DONE or BLOCKED — review every remaining PLANNED step. If execution feedback invalidates future steps, update them immediately. Insert corrective steps for failures. Remove steps that are no longer relevant. This prevents cascading errors from an obsolete plan.

10. **Iterate until exploitation succeeds or the iteration budget is exhausted.** Continue the execute-evaluate-revise loop (default cap: 20 iterations). Terminate when: (a) a working proof-of-concept is produced with validated output, (b) all reasonable exploitation paths are exhausted, or (c) the iteration cap is reached. Document the final exploit chain, PoC payload, and execution trace.

## Concrete Examples

**Example 1: SQL Injection Discovery and Exploitation in a Flask App**

User: "Find and exploit vulnerabilities in this Flask web application (authorized pentest)"

Approach:
1. Map the file structure: find app.py, routes/, models/, templates/, requirements.txt
2. Taint analysis on routes/: find `request.args.get('id')` passed directly into `db.execute(f"SELECT * FROM users WHERE id = {user_id}")` at routes/users.py:47
3. Evidence chain: Source=routes/users.py:32 (request.args), Sink=routes/users.py:47 (db.execute with f-string), Context=no parameterized query, no input validation. CWE-89 (SQL Injection), Risk=Critical
4. Critique: Confirm db.execute uses raw SQL (not ORM), confirm route is publicly accessible (no @login_required), confirm no WAF or input sanitization middleware
5. Exploit plan:
   - Step 1 (BASH): `curl "http://target:5000/users?id=1 OR 1=1"` — test for boolean-based SQLi
   - Step 2 (BASH): `curl "http://target:5000/users?id=1 UNION SELECT username,password FROM users--"` — extract credentials
   - Step 3 (VERIFICATION): Confirm extracted data matches database contents
6. Execute Step 1: Response returns all users instead of one — confirms SQLi
7. Execute Step 2: Returns usernames and password hashes — exploitation confirmed
8. Output: PoC payload, full exploit chain, remediation recommendation (use parameterized queries)

Output:
```
## Vulnerability Report

### Finding 1: SQL Injection (CWE-89) — CRITICAL

**Source**: routes/users.py:32 — `user_id = request.args.get('id')`
**Sink**: routes/users.py:47 — `db.execute(f"SELECT * FROM users WHERE id = {user_id}")`
**Protection gaps**: No parameterized queries, no input validation, no WAF

**Proof of Concept**:
  curl "http://target:5000/users?id=1 UNION SELECT username,password FROM users--"

**Impact**: Full database read access, credential extraction
**Remediation**: Use parameterized queries: `db.execute("SELECT * FROM users WHERE id = ?", (user_id,))`
```

**Example 2: Multi-Step Exploitation of Command Injection in a CI/CD Pipeline**

User: "This CI/CD service lets users submit build configs. Check if it's exploitable (CTF challenge)"

Approach:
1. Map attack surface: find config parser at services/builder.py, identify user-supplied YAML parsed at line 23, build commands constructed at line 67
2. Taint analysis: User YAML field `build_command` flows to `subprocess.call(config['build_command'], shell=True)` at services/builder.py:67 — no sanitization
3. Evidence: CWE-78 (OS Command Injection), Risk=Critical
4. Critique: Confirm shell=True is used, confirm no allowlist on build_command values, confirm service runs as root in container
5. Exploit plan:
   - Step 1: Submit config with `build_command: "echo test"` — verify normal execution
   - Step 2: Submit config with `build_command: "id && cat /etc/passwd"` — test command chaining
   - Step 3: Submit config with `build_command: "cat /flag.txt"` — retrieve CTF flag
6. Execute Step 1: Returns "test" — confirms command execution
7. Execute Step 2: Returns uid=0(root) and passwd contents — confirms injection with root privileges
8. Revise plan: Step 3 remains valid, proceed
9. Execute Step 3: Returns flag — CTF solved

**Example 3: Discovery-Only Security Audit (No Exploitation)**

User: "Audit this Node.js Express API for security issues, don't exploit anything"

Approach:
1. Map structure: package.json (check dependencies for known CVEs), app.js, routes/, middleware/
2. Taint analysis across all routes: trace req.body, req.params, req.query to sinks
3. Trust boundary mapping: check authentication middleware coverage on all routes
4. Configuration audit: check helmet usage, CORS policy, rate limiting, cookie settings
5. Findings with evidence chains only — no exploitation stage

Output:
```
## Security Audit Report — 4 Findings

1. **Stored XSS (CWE-79) — HIGH**: routes/comments.js:34 stores req.body.text
   without sanitization, rendered via res.send() at routes/posts.js:89 without escaping.

2. **Missing Authentication (CWE-862) — HIGH**: routes/admin.js:12-45 defines
   /admin/users and /admin/config endpoints with no auth middleware.

3. **Prototype Pollution (CWE-1321) — MEDIUM**: lib/merge.js:7 uses recursive
   object merge without __proto__ filtering on user-supplied JSON.

4. **Outdated Dependency (CWE-1395) — MEDIUM**: package.json pins lodash@4.17.15,
   which is vulnerable to CVE-2020-8203 (prototype pollution).
```

## Best Practices

**Do:**
- Always require and verify authorization context before performing any security testing
- Produce evidence at line-number granularity — vague findings are worthless
- Iterate on exploitation with real execution feedback rather than guessing payloads
- Revise the entire remaining plan after each execution step, not just the current step
- Prioritize quality over quantity: fewer well-proven vulnerabilities beat many speculative ones
- Record both successes and failures for each exploitation attempt to build reusable knowledge

**Avoid:**
- Never attempt single-shot exploit generation — use the iterative execute-evaluate-revise loop
- Never accept a vulnerability finding without a complete source-sink-context evidence chain
- Never execute commands without validation (check syntax, safety, scope alignment first)
- Never ignore framework protections — verify whether ORM parameterization, template auto-escaping, or middleware actually neutralize the threat before reporting
- Never continue executing an obsolete plan after execution feedback contradicts its assumptions
- Never exceed 20 exploitation iterations without reassessing the overall approach

## Error Handling

- **False positive after critique**: If the critique loop identifies that a framework protection neutralizes the vulnerability (e.g., Django ORM prevents raw SQL injection), discard the finding and document why. Do not pad reports with false positives.
- **Execution environment unavailable**: If no sandbox or Docker environment is available for exploitation, stop at Stage I (discovery) and produce a detailed evidence-based report with theoretical exploit paths. Mark findings as "unvalidated."
- **Exploit step fails unexpectedly**: Do not retry the same action blindly. Evaluate the error output, determine the root cause (wrong assumption, environment difference, missing dependency), revise the plan, and attempt a corrective action.
- **Iteration budget exhausted**: After 20 iterations without successful exploitation, summarize what was attempted, what was learned, which paths remain unexplored, and whether the vulnerability is likely exploitable with a different approach.
- **Ambiguous authorization scope**: If the user's authorization context is unclear, ask explicitly before proceeding. Never assume authorization for destructive actions or actions against systems not explicitly designated as targets.

## Limitations

- **Requires execution environment for Stage II**: The exploitation stage depends on running commands against a live or sandboxed target. Without this, the skill is limited to discovery-only analysis.
- **Not a replacement for specialized tools**: Static analysis tools (Semgrep, CodeQL), dynamic scanners (Burp Suite, OWASP ZAP), and fuzzing frameworks catch classes of bugs that code reading alone cannot. Use this workflow to complement, not replace, automated tooling.
- **Memory bootstrapping**: The three-layer memory system performs best when seeded with prior security experience. First-run analysis on a novel technology stack may miss patterns that would be caught after several engagements.
- **LLM knowledge boundary**: Vulnerabilities requiring deep binary analysis, hardware-level exploitation, or novel zero-day research in compiled code are beyond the scope of code-level reasoning.
- **Iteration cost**: The iterative exploitation loop can consume significant compute. Set appropriate iteration caps based on the complexity of the target and the authorization window.

## Reference

**Paper**: [Co-RedTeam: Orchestrated Security Discovery and Exploitation with LLM Agents](https://arxiv.org/abs/2602.02164v2) (He et al., 2026). Key sections: Section 3 for the full agent architecture and interaction protocol, Section 4 for the three-layer memory system design, and Tables 1-5 for benchmark results showing execution feedback as the single most impactful component (41.6% performance impact in ablation).