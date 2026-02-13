---
name: "autoregressive-yet-revisable-decoding-revision"
description: |
  Generate secure code using Stream of Revision — an in-decoding self-correction technique
  that backtracks and patches vulnerable code spans during generation rather than after it.
  Trigger phrases:
  - "generate secure code"
  - "fix security vulnerabilities in this code"
  - "write safe C/C++ code"
  - "review and revise code for security"
  - "backtrack and fix this vulnerability"
  - "self-correcting code generation"
---

# Stream of Revision: In-Decoding Revision for Secure Code Generation

This skill enables Claude to generate code using a **self-correcting revision loop** inspired by the Stream of Revision (SoR) framework. Instead of writing code linearly and fixing vulnerabilities after the fact, Claude generates code in a forward pass but pauses mid-generation when it detects a likely vulnerability, backtracks to the vulnerable span, and splices in a secure replacement — all within a single coherent generation. This mirrors how experienced programmers actually write code: forward drafting interleaved with on-the-fly revision.

## When to Use

- When the user asks to generate C, C++, Python, Java, JavaScript, or C# code that handles untrusted input, memory allocation, file I/O, or network data
- When writing code involving CWE Top-25 vulnerability categories (buffer overflows, SQL injection, path traversal, command injection, use-after-free, integer overflow)
- When the user asks to "write secure code" or "generate code without vulnerabilities"
- When reviewing and rewriting existing code to eliminate security flaws inline
- When generating cryptographic code, authentication logic, or access control implementations
- When the user wants code that handles memory management safely in C/C++

## Key Technique

**The core insight:** Traditional code generation is strictly monotonic — tokens are appended to an immutable prefix. Stream of Revision breaks this by introducing a revision episode mechanism. During generation, the model can emit a **backtracking trigger**, localize a vulnerable span in its own output using content-addressable matching, and then generate a patched replacement that is atomically spliced in. The critical advantage over post-hoc repair agents is a 6.5x reduction in input token cost (113 vs. 743 tokens) while achieving comparable or superior security.

**How it works mechanically:** The revision episode has three phases: (1) **Trigger** — the model recognizes it has generated a vulnerable pattern and signals a revision; (2) **Localization** — the model identifies the exact span to replace by repeating the vulnerable code bounded by scope delimiters; (3) **Patch** — the model generates the corrected code bounded by patch delimiters. A deterministic renderer then performs an in-place splice at the rightmost occurrence, maintaining syntactic validity 98.45% of the time.

**Why this matters for Claude:** While Claude cannot modify its own token stream mid-generation, we can simulate the SoR pattern by structuring generation as an explicit draft-then-revise workflow within a single response. Claude generates a code block, immediately audits it against known vulnerability patterns, and emits the corrected version — keeping the revision loop tight and internalized rather than requiring separate tool calls or user intervention.

## Step-by-Step Workflow

1. **Parse the security context.** Identify the language, the nature of untrusted inputs, and which CWE categories are relevant. For C/C++ code, prioritize buffer overflows (CWE-120), use-after-free (CWE-416), and integer overflow (CWE-190). For web languages, prioritize injection (CWE-89, CWE-79) and path traversal (CWE-22).

2. **Generate an initial code draft.** Write the code that satisfies the functional requirements. Do not over-optimize for security yet — focus on correctness and clarity first.

3. **Trigger a revision audit.** Immediately after the draft, scan the generated code span-by-span for vulnerability patterns. Check each function call, memory operation, string manipulation, and input handling site against the relevant CWE patterns.

4. **Localize vulnerable spans.** For each detected vulnerability, identify the exact lines or expressions that are unsafe. Quote them precisely — this is the "content-addressable localization" step. Be specific: not "the buffer handling code" but `char buf[256]; strcpy(buf, user_input);`.

5. **Generate secure patches.** For each localized span, produce a replacement that eliminates the vulnerability while preserving functional semantics. Apply the minimal change necessary — do not refactor surrounding code.

6. **Apply patches atomically.** Emit the final corrected code with all patches applied in-place. If multiple revision episodes overlap, apply them from rightmost to leftmost to maintain correct offset calculations.

7. **Verify syntactic integrity.** Confirm the patched code compiles/parses correctly. The SoR paper reports 98.45% of revisions are non-destructive — aim for 100% by checking that patches respect scope boundaries and type constraints.

8. **Annotate revisions.** Add brief inline comments at each patch site explaining what vulnerability was present and how the patch addresses it. This makes the revision transparent and auditable.

9. **Assess residual risk.** State any remaining security considerations that cannot be addressed purely through code revision (e.g., architectural issues, missing authentication layers, configuration-dependent risks).

## Concrete Examples

**Example 1: Buffer Overflow in C String Handling**

User: "Write a C function that reads a username from stdin and greets the user."

Approach:
1. Draft initial implementation with `gets()` or `scanf("%s", ...)`
2. Trigger revision: detect unbounded read into fixed buffer
3. Localize: `char name[64]; scanf("%s", name);`
4. Patch: replace with bounded read using `fgets()` and null-termination

Draft (vulnerable):
```c
void greet_user() {
    char name[64];
    scanf("%s", name);  // CWE-120: unbounded read
    printf("Hello, %s!\n", name);
}
```

Revision applied:
```c
void greet_user() {
    char name[64];
    if (fgets(name, sizeof(name), stdin) != NULL) {
        // Remove trailing newline if present
        name[strcspn(name, "\n")] = '\0';
        printf("Hello, %s!\n", name);
    }
}
```

Annotation: Replaced `scanf("%s")` with `fgets()` bounded to buffer size, preventing buffer overflow (CWE-120). Added NULL check on return value.

---

**Example 2: SQL Injection in Python**

User: "Write a function to look up a user by email in our SQLite database."

Approach:
1. Draft using f-string interpolation into SQL
2. Trigger revision: detect string concatenation/interpolation in SQL query
3. Localize: `cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")`
4. Patch: replace with parameterized query

Draft (vulnerable):
```python
def find_user_by_email(db_path: str, email: str) -> dict | None:
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")  # CWE-89
    row = cursor.fetchone()
    conn.close()
    return row
```

Revision applied:
```python
def find_user_by_email(db_path: str, email: str) -> dict | None:
    conn = sqlite3.connect(db_path)
    try:
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE email = ?", (email,))
        row = cursor.fetchone()
        return row
    finally:
        conn.close()
```

Annotation: Replaced f-string SQL interpolation with parameterized query using `?` placeholder (CWE-89). Added `try/finally` to ensure connection cleanup.

---

**Example 3: Path Traversal in JavaScript**

User: "Write an Express route that serves files from the uploads directory."

Approach:
1. Draft using `path.join('uploads', req.params.filename)`
2. Trigger revision: detect unsanitized path component from user input
3. Localize: `const filePath = path.join('uploads', req.params.filename);`
4. Patch: add `path.basename()` sanitization and resolved-path containment check

Draft (vulnerable):
```javascript
app.get('/files/:filename', (req, res) => {
    const filePath = path.join(__dirname, 'uploads', req.params.filename);
    res.sendFile(filePath);  // CWE-22: path traversal via ../
});
```

Revision applied:
```javascript
app.get('/files/:filename', (req, res) => {
    const safeName = path.basename(req.params.filename);
    const filePath = path.resolve(path.join(__dirname, 'uploads', safeName));
    const uploadsDir = path.resolve(path.join(__dirname, 'uploads'));

    if (!filePath.startsWith(uploadsDir + path.sep)) {
        return res.status(403).send('Forbidden');
    }
    res.sendFile(filePath);
});
```

Annotation: Applied `path.basename()` to strip directory traversal sequences and added resolved-path containment check to ensure the final path stays within the uploads directory (CWE-22).

## Best Practices

**Do:**
- Always generate the vulnerable draft mentally or explicitly before producing the secure version — this forces systematic vulnerability identification rather than hoping the first pass is safe
- Apply the minimal patch that eliminates the vulnerability; resist refactoring unrelated code during revision
- Annotate every revision site with the CWE identifier and a one-line explanation
- When multiple vulnerabilities exist, address them independently — each gets its own localize-and-patch cycle
- Prioritize CWE Top-25 patterns: they cover the vast majority of real-world exploits

**Avoid:**
- Do not skip the revision audit even for "simple" code — the SoR paper shows that even expert models introduce vulnerabilities in straightforward string handling and file I/O
- Do not apply overly defensive patches that break functionality (e.g., rejecting all input containing special characters when only SQL metacharacters are dangerous)
- Do not conflate security revision with code review — focus strictly on vulnerability elimination, not style or performance
- Do not assume a single revision pass catches everything; for high-stakes code, state explicitly what was checked and what residual risks remain

## Error Handling

**Patch breaks compilation.** If a security patch introduces a syntax error or type mismatch, revert to the vulnerable span and try an alternative fix. The SoR framework achieves 98.45% non-destructive patches by respecting scope boundaries — always verify the patch matches the surrounding type context and control flow.

**Ambiguous localization.** When the same vulnerable pattern appears multiple times (e.g., multiple `strcpy` calls), localize each instance independently with enough surrounding context to uniquely identify it. Apply the rightmost-match-first strategy to avoid offset drift.

**False positive trigger.** If a span looks vulnerable but is actually safe due to prior validation (e.g., input was already bounds-checked upstream), do not patch it. Note the existing safeguard in the annotation rather than adding redundant protection.

**Overlapping patches.** When two vulnerability fixes touch the same lines, merge them into a single atomic patch to avoid conflicts. Test the merged result for both vulnerabilities.

## Limitations

- This technique addresses code-level vulnerabilities detectable from local context. It cannot fix **architectural security flaws** like missing authentication middleware, improper session management, or insecure deployment configurations.
- The revision loop is most effective for well-characterized vulnerability patterns (CWE Top-25). Novel or domain-specific vulnerability classes may not trigger revision.
- For languages with complex macro systems (C preprocessor) or metaprogramming, the localize-and-patch approach may not capture vulnerabilities hidden behind macro expansion.
- The approach adds modest output overhead (~6% more tokens per the paper's measurements). For latency-critical applications generating thousands of code snippets, this adds up.
- Security patches may conflict with performance requirements (e.g., bounds checking in hot loops). Flag these trade-offs explicitly rather than silently choosing one over the other.

## Reference

**Paper:** [Autoregressive, Yet Revisable: In Decoding Revision for Secure Code Generation](https://arxiv.org/abs/2602.01187v1) — Yang et al., 2026.
**Key takeaway:** Look at Section 3 for the formal revision episode structure (`trigger → localize → patch → render`), Table 2 for security pass rates across languages showing +7.1% improvement on CWE Top-10, and Table 3 for the 6.5x input token efficiency gain over post-hoc repair agents.