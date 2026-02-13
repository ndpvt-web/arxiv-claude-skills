---
name: "seccodeprm-process-reward-code"
description: "Step-level security scoring for code generation and vulnerability detection using process reward model techniques. Use when asked to: 'review this code for vulnerabilities', 'generate secure code for [task]', 'score security of each code block', 'find the risky parts of this function', 'which lines introduce the vulnerability', 'help me write secure C/C++/Python code'."
---

# SecCodePRM: Step-Level Security Scoring for Code

This skill enables Claude to apply **process reward model reasoning** to code security — scoring code step-by-step rather than as a monolithic block. Inspired by the SecCodePRM paper, the technique segments code into logical steps (roughly paragraph-level blocks), assigns each step a security risk score considering all prior context, then aggregates scores with risk-sensitive weighting that amplifies the most dangerous regions. This produces dense, localized security feedback that works on complete programs, partial prefixes during generation, and as a ranking signal when choosing between alternative implementations.

## When to Use

- When the user asks to **review code for security vulnerabilities** and wants to know *which specific blocks* are dangerous, not just a binary safe/unsafe verdict
- When **generating security-sensitive code** (crypto, auth, memory management, input parsing, SQL queries) and you need to evaluate multiple candidate approaches
- When analyzing **incomplete or streaming code** to flag emerging risks before the function is finished
- When the user asks to **fix a vulnerability** and you need to precisely localize the root cause across inter-procedural call chains
- When comparing **two or more implementations** of the same functionality and choosing the most secure one
- When working with **long codebases** (1000+ lines) where whole-program reasoning degrades — step-level scoring stays precise

## Key Technique

**Step-level security scoring** breaks code into discrete logical steps — blocks separated by structural boundaries (blank lines, statement groups, function boundaries). Each step receives a security score that reflects its risk *in context of all preceding steps*. This is fundamentally different from line-level linting (no context) or whole-program classification (no localization). A `malloc` call is neutral in isolation but dangerous if a prior step set a user-controlled size without bounds checking. Step-level scoring captures this.

**Risk-sensitive aggregation** converts per-step scores into a program-level verdict by weighting steps inversely to their safety: `w_i = exp(-r_i / tau) / sum(exp(-r_j / tau))`. Steps with low safety scores get exponentially higher weight. This means a single dangerous step dominates the aggregate even if surrounded by safe code — matching real-world security where one exploitable flaw compromises the entire program regardless of how clean the rest is.

**Inference-time scaling for secure generation** uses best-of-N sampling: generate K candidate continuations at each step, score each with the process reward model, and select the candidate with highest advantage `A_t = r_t - mean(r_t)`. This steers generation toward secure trajectories without sacrificing functional correctness — the paper shows pass@1 rates actually improve slightly because security-aware selection also filters out structurally broken code.

## Step-by-Step Workflow

1. **Segment the code into logical steps.** Split at double-newline boundaries (`\n\n`). Merge fragments that are structurally incomplete (a lone `return 0;`, a variable declaration without its use). Each step should be a self-contained logical unit: a variable declaration block, a conditional branch, a loop body, a function call sequence.

2. **Build a context chain.** For each step `t`, the evaluation context is the concatenation of all steps `1..t`. Never score a step in isolation — a `free(ptr)` is safe on first occurrence but a use-after-free on second.

3. **Score each step for security risk.** Evaluate the margin between safe and unsafe: `r_t = P(safe | steps_1..t) - P(unsafe | steps_1..t)`. Scores range from -1 (certainly vulnerable) to +1 (certainly safe). Flag any step with `r_t < 0` as a risk point.

4. **Check for inter-procedural vulnerability propagation.** If step `t` calls a function defined elsewhere, trace the callee's steps. A tainted return value in a callee propagates vulnerability to the caller's step. Annotate the call chain.

5. **Aggregate with risk-sensitive weighting.** Compute the program-level score `R = sum(w_i * r_i)` where weights emphasize the lowest-scoring steps. Use temperature `tau` between 0.1 (very risk-sensitive, dominated by worst step) and 1.0 (balanced). Default to `tau = 0.3` for security-critical reviews.

6. **For vulnerability detection:** Report each step with `r_t < 0` as a finding, ordered by severity (most negative first). Map each finding to a CWE category: buffer overflow (CWE-125), integer overflow (CWE-190), use-after-free (CWE-416), null pointer dereference (CWE-476), injection (CWE-89), etc.

7. **For secure code generation:** When writing new code, mentally evaluate 2-3 alternative implementations at each decision point. Choose the continuation where the cumulative reward is highest. Prefer bounds-checked alternatives, safe library functions, and validated inputs.

8. **For partial-code assessment:** When reviewing incomplete code, score all steps written so far. If the trajectory is trending negative (scores decreasing), warn the user before they continue building on a risky foundation.

9. **Verify functional correctness is preserved.** After any security-motivated change, confirm the code still fulfills its functional requirements. Security fixes that break functionality are not acceptable — the goal is improved security at equivalent or better utility.

10. **Produce a structured security report.** For each flagged step: quote the exact code, state the risk, identify the CWE, suggest a concrete fix, and show the fixed version with its improved step score.

## Concrete Examples

**Example 1: Localize a buffer overflow in C**

User: "Review this C function for security issues"

```c
void process_input(char *user_data, int len) {

    char buffer[256];

    int adjusted_len = len + 10;

    memcpy(buffer, user_data, adjusted_len);

    buffer[adjusted_len] = '\0';

    printf("Processed: %s\n", buffer);
}
```

Approach:
1. Segment into steps: (a) declaration of `buffer`, (b) `adjusted_len` computation, (c) `memcpy` call, (d) null termination, (e) printf
2. Score each step in context:
   - Step (a): `r=+0.8` — fixed-size stack buffer, neutral alone
   - Step (b): `r=+0.2` — arithmetic on user-controlled `len` without overflow check (CWE-190 risk emerging)
   - Step (c): `r=-0.7` — copies `adjusted_len` bytes into 256-byte buffer with no bounds check (CWE-125 triggered)
   - Step (d): `r=-0.5` — writes at `adjusted_len` offset, compounding the overflow
   - Step (e): `r=+0.6` — printf itself is safe given prior null termination
3. Risk-sensitive aggregate: `R = -0.41` (dominated by steps c and d)

Output:
```
SECURITY FINDINGS (2 issues, aggregate risk: HIGH)

[CRITICAL] Step 3 (line 7): Heap/stack buffer overflow — CWE-125
  memcpy(buffer, user_data, adjusted_len);
  `adjusted_len` can exceed 256. No bounds check on `len`.

  Fix: if (adjusted_len > sizeof(buffer) - 1) return;  // before memcpy

[HIGH] Step 2 (line 5): Integer overflow — CWE-190
  int adjusted_len = len + 10;
  If len is near INT_MAX, `len + 10` wraps negative, causing
  memcpy to receive a huge size_t value.

  Fix: if (len < 0 || len > 246) return;  // bounds check before arithmetic
```

**Example 2: Secure code generation for SQL query**

User: "Write a Python function to look up a user by username from a database"

Approach:
1. At the query construction step, evaluate candidates:
   - Candidate A: `f"SELECT * FROM users WHERE name = '{username}'"` — step score `r=-0.9` (CWE-89 SQL injection)
   - Candidate B: `cursor.execute("SELECT * FROM users WHERE name = ?", (username,))` — step score `r=+0.9` (parameterized)
2. Select candidate B (highest reward)
3. Continue generation with input validation step

Output:
```python
import sqlite3

def lookup_user(db_path: str, username: str) -> dict | None:
    """Look up a user by username using parameterized query."""
    if not username or len(username) > 255:
        return None

    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()

    cursor.execute(
        "SELECT id, username, email FROM users WHERE username = ?",
        (username,),
    )

    row = cursor.fetchone()
    conn.close()

    if row is None:
        return None
    return {"id": row[0], "username": row[1], "email": row[2]}
```

Step scores: input validation `r=+0.8`, connection `r=+0.7`, parameterized query `r=+0.9`, result handling `r=+0.7`. Aggregate: `R = +0.78`.

**Example 3: Partial-code prefix assessment**

User: "I'm writing a file upload handler, is this start safe so far?"

```python
from flask import request
import os

def upload_file():
    f = request.files['file']
    filename = f.filename

    save_path = os.path.join('/uploads', filename)
```

Approach:
1. Score steps written so far:
   - Step 1 (imports): `r=+0.7` — neutral
   - Step 2 (get file): `r=+0.5` — no check for missing key, minor
   - Step 3 (filename): `r=-0.3` — using raw user-supplied filename (CWE-22 path traversal risk emerging)
   - Step 4 (path join): `r=-0.8` — `os.path.join('/uploads', '../../../etc/passwd')` resolves outside uploads dir
2. Trajectory is trending negative — warn before user continues

Output:
```
WARNING: Security trajectory is negative at step 4.

Step 3-4 introduce path traversal risk (CWE-22):
  filename = f.filename                              # unsanitized user input
  save_path = os.path.join('/uploads', filename)     # attacker controls path

Before continuing, add:
  filename = secure_filename(f.filename)  # from werkzeug.utils
  save_path = os.path.join('/uploads', filename)
  if not os.path.abspath(save_path).startswith('/uploads'):
      abort(400)

This raises the step score from r=-0.8 to r=+0.8.
```

## Best Practices

- **Do:** Score every step in the context of all preceding steps. A `free()` call is safe or dangerous depending on what came before it.
- **Do:** Use risk-sensitive aggregation (low tau) for security reviews. A single critical flaw at step 15 of 50 should dominate the score, not be averaged away.
- **Do:** Map findings to specific CWE identifiers. "This is insecure" is less useful than "CWE-416: use-after-free due to double free on line 34."
- **Do:** When generating code, prefer the secure alternative at each decision point even when it requires a few extra lines. The paper shows this preserves or improves functional correctness.
- **Avoid:** Scoring steps in isolation without context. `ptr = malloc(n)` is meaningless without knowing if `n` was validated.
- **Avoid:** Treating all steps equally in aggregation. Uniform averaging hides critical vulnerabilities behind a majority of safe code. Always weight toward risk.

## Error Handling

- **False positives on defensive code:** Code that handles error paths (null checks, bounds validation) may look similar to vulnerable patterns. Score the *complete* conditional block as one step rather than splitting guard and body.
- **Cross-file vulnerabilities:** When a vulnerability spans multiple files (e.g., tainted data flows from a controller to a model), explicitly reconstruct the inter-procedural call chain before scoring. Scoring one file in isolation will miss the issue.
- **Very long functions:** If a function exceeds 50 steps, group related steps into macro-blocks (initialization, processing, cleanup) and score at both granularities. Step-level scores within each block, plus block-level aggregation for the function verdict.
- **Language-specific patterns:** Certain patterns are idiomatic and safe in one language but dangerous in another (e.g., pointer arithmetic in Rust vs C). Calibrate severity to the language's safety guarantees.

## Limitations

- This technique is strongest for **memory safety (C/C++)**, **injection (SQL/XSS/command)**, and **input validation** vulnerabilities. It is less effective for **logic bugs**, **cryptographic misuse** (wrong mode of operation), or **race conditions** that depend on runtime timing rather than code structure.
- Step-level scoring approximates but does not replace full **taint analysis** or **symbolic execution**. It cannot prove the absence of vulnerabilities — it identifies likely risk regions.
- The approach works best on **imperative, sequential code**. Heavily callback-based, event-driven, or reactive code may not segment cleanly into linear steps.
- For **production vulnerability scanning**, complement this technique with actual static analysis tools (CodeQL, Semgrep) and dynamic testing. This skill provides fast, contextual feedback during authoring, not a formal security audit.

## Reference

[SecCodePRM: A Process Reward Model for Code Security](https://arxiv.org/abs/2602.10418v1) — Yu et al., 2026. Focus on Section 3 (step segmentation and labeling), Section 4 (risk-sensitive aggregation formula), and Section 5.3 (inference-time scaling for secure generation). The key insight: scoring code security at the step level with risk-weighted aggregation outperforms both static analyzers and whole-program classifiers while preserving functional correctness.