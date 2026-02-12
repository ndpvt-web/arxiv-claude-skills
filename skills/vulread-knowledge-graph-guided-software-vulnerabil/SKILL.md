---
name: "vulread-knowledge-graph-guided-software-vulnerabil"
description: "Knowledge-graph-guided vulnerability reasoning and CWE-level detection for source code. Analyzes functions for security flaws using structured CWE taxonomy reasoning rather than binary vulnerable/not-vulnerable labels. Trigger phrases: 'analyze this code for vulnerabilities', 'what CWE does this bug fall under', 'explain why this function is vulnerable', 'find security flaws with CWE classification', 'vulnerability reasoning for this code', 'CWE-level security audit'"
---

This skill enables Claude to perform **CWE-taxonomy-guided vulnerability reasoning** on source code, applying the VulReaD methodology from arXiv:2602.10787. Instead of simply flagging code as "vulnerable" or "safe," Claude extracts security-relevant entities from functions (API calls, memory operations, file handles, user-controlled inputs), maps them through a mental model of CWE abstraction classes, and produces structured reasoning that ties specific code constructs to specific CWE categories with root-cause explanations. This approach yields dramatically more useful security analysis than binary classification alone.

## When to Use

- When the user asks to **review a function or file for security vulnerabilities** and wants to know *which* CWE categories apply, not just yes/no
- When the user provides a code snippet and asks **"what kind of vulnerability is this?"** or **"what CWE is this?"**
- When performing a **security audit** and the user needs structured, taxonomy-aligned explanations for each finding
- When the user wants to understand **why** code is vulnerable, with reasoning grounded in the specific weakness mechanism (buffer overflow vs. use-after-free vs. injection, etc.)
- When comparing two code versions (patched vs. unpatched) and the user wants **contrastive reasoning** about what changed and why
- When triaging vulnerability reports and needing to **validate or correct CWE attributions** (e.g., a scanner says CWE-119 but the real issue is CWE-416)

## Key Technique

VulReaD's core insight is that vulnerability detection improves substantially when reasoning is **anchored to a structured security knowledge graph** rather than performed in open-ended fashion. The method defines 13 vulnerability abstraction classes — File and Path Handling, Input Validation, Access Control, Memory Management, Cryptographic Operations, Resource Lifecycle, Numeric Processing, Concurrency and Synchronization, Error Handling, Information Disclosure, Configuration and Environment, Code Injection Surfaces, and Authentication and Session Management — that serve as intermediate semantic categories between raw code and specific CWE identifiers. Each CWE maps to one or more abstraction classes via keyword matching and embedding similarity against CWE descriptions.

The practical workflow is: (1) extract security-relevant **entities** from code — function names, API calls, variable names, pointer operations, system calls, file descriptors; (2) associate each entity with one or more **abstraction classes** based on what security domain it belongs to; (3) use the abstraction class to retrieve **candidate CWE categories** from the knowledge graph; (4) generate **structured reasoning** that explains whether and how the code's entities interact to produce a vulnerability matching a specific CWE. This structured pipeline prevents the common failure mode where an LLM correctly detects a vulnerability exists but misattributes it to the wrong CWE (e.g., calling a memory leak CWE-401 a use-after-free CWE-416).

The contrastive dimension is equally important: for every piece of valid reasoning ("this function is vulnerable to CWE-120 because `strcpy` copies user input into a fixed-size buffer without bounds checking"), VulReaD also considers what **flawed reasoning** looks like ("this function is safe because `strcpy` is a standard library function") — and explicitly trains to prefer the grounded explanation over the superficial one. When applying this skill, Claude should similarly construct both the vulnerability argument and the counter-argument, then evaluate which is better supported by the code.

## Step-by-Step Workflow

1. **Extract the target function(s).** Read the complete function body including its signature, local variables, called APIs, and any relevant type definitions or macros. If the user provides a file, identify individual function boundaries.

2. **Identify security-relevant entities.** Scan the function for: memory allocation/deallocation calls (`malloc`, `free`, `new`, `delete`, `realloc`), string/buffer operations (`strcpy`, `strncpy`, `memcpy`, `sprintf`), file/path operations (`fopen`, `open`, `unlink`, file descriptor usage), input sources (`recv`, `read`, `scanf`, `argv`, `getenv`, HTTP parameters), authentication/session calls, cryptographic operations, numeric casts/arithmetic, and concurrency primitives (`pthread_mutex`, atomics).

3. **Map entities to abstraction classes.** Assign each entity to one or more of the 13 abstraction classes:
   - `malloc`/`free`/pointer arithmetic → **Memory Management**
   - `strcpy`/`sprintf`/`gets` → **Input Validation** + **Memory Management**
   - `fopen`/path concatenation → **File and Path Handling**
   - `recv`/`argv`/`getenv` → **Input Validation** + **Code Injection Surfaces**
   - SQL string building → **Code Injection Surfaces**
   - `rand()`/weak hash → **Cryptographic Operations**
   - Integer overflow-prone arithmetic → **Numeric Processing**
   - Missing `fclose`/`free` on error paths → **Resource Lifecycle**
   - Missing permission checks → **Access Control**

4. **Retrieve candidate CWEs for each abstraction class.** Use the mapping to narrow the search space. For example, Memory Management entities point to CWE-119 (Buffer Overflow), CWE-120 (Classic Buffer Overflow), CWE-125 (Out-of-bounds Read), CWE-416 (Use After Free), CWE-401 (Memory Leak), CWE-415 (Double Free), CWE-476 (NULL Pointer Dereference). Input Validation points to CWE-20, CWE-78, CWE-79, CWE-89, CWE-134, etc.

5. **Trace data flow through the function.** Follow user-controlled or external inputs from their source to their use. Identify whether bounds checks, sanitization, or validation occur between source and sink. Note any missing error-handling branches.

6. **Construct the vulnerability argument.** For each candidate CWE, write a specific claim: "Entity X [from abstraction class Y] flows to sink Z without [specific protection], matching CWE-NNN because [mechanism]." Include the specific lines of code.

7. **Construct the counter-argument.** For each candidate, also articulate why the code might be safe: "Although `strcpy` is used, the source buffer is statically bounded to N bytes and the destination is N+1 bytes, so CWE-120 does not apply here."

8. **Evaluate and select.** Compare the vulnerability argument against the counter-argument for each candidate CWE. Select the determination (vulnerable or safe) that is better supported by the actual code. Discard CWEs where the counter-argument is stronger.

9. **Produce structured output.** For each confirmed finding, report: the CWE identifier, the abstraction class, the specific code lines involved, the root-cause mechanism, and a remediation suggestion. For safe code, briefly state which abstraction classes were checked and why no vulnerability was found.

10. **Assign confidence and severity.** Rate each finding as high/medium/low confidence based on how complete the data-flow trace is (e.g., cross-function flows are lower confidence). Note if full context is missing.

## Concrete Examples

**Example 1: Buffer overflow in C function**

User: "Is this function vulnerable?"

```c
void process_input(char *user_data) {
    char buffer[64];
    strcpy(buffer, user_data);
    log_message(buffer);
}
```

Approach:
1. Entities: `strcpy` (string copy), `buffer` (stack-allocated 64 bytes), `user_data` (function parameter, externally controlled)
2. Abstraction classes: Memory Management, Input Validation
3. Candidate CWEs: CWE-119, CWE-120, CWE-121 (Stack-based Buffer Overflow)
4. Data flow: `user_data` (external, unbounded) → `strcpy` → `buffer` (64 bytes, no bounds check)
5. Vulnerability argument: `strcpy` copies `user_data` into a 64-byte stack buffer with no length check. If `user_data` exceeds 63 bytes, stack corruption occurs. This is CWE-121 (Stack-based Buffer Overflow), a specific instance of CWE-120.
6. Counter-argument: Only valid if all callers guarantee `user_data` is < 64 bytes — no such guarantee is visible.

Output:
```
**Vulnerability Detected: CWE-121 (Stack-based Buffer Overflow)**
- Abstraction Class: Memory Management, Input Validation
- Location: line 3, `strcpy(buffer, user_data)`
- Root Cause: Unbounded external input `user_data` copied into fixed 64-byte
  stack buffer via `strcpy`, which performs no length checking.
- Confidence: High (complete data flow visible within function)
- Remediation: Replace `strcpy` with `strncpy(buffer, user_data, sizeof(buffer) - 1)`
  and null-terminate, or use `snprintf`. Better: validate input length at the
  caller boundary.
```

**Example 2: Memory leak on error path**

User: "Audit this function for security issues."

```c
int load_config(const char *path) {
    FILE *fp = fopen(path, "r");
    if (!fp) return -1;

    char *buf = malloc(4096);
    if (!buf) return -1;  // <-- leak: fp not closed

    if (parse(fp, buf) < 0) {
        free(buf);
        return -1;  // <-- leak: fp not closed
    }

    fclose(fp);
    free(buf);
    return 0;
}
```

Approach:
1. Entities: `fopen`/`fclose` (file handle lifecycle), `malloc`/`free` (memory lifecycle), error-return paths
2. Abstraction classes: Resource Lifecycle, File and Path Handling, Memory Management
3. Candidate CWEs: CWE-401 (Memory Leak), CWE-772 (Missing Release of Resource after Effective Lifetime), CWE-403 (File Descriptor Leak)
4. Trace error paths: `malloc` failure at line 5 returns without `fclose(fp)`. `parse` failure at line 7 returns after `free(buf)` but without `fclose(fp)`.
5. Vulnerability argument: Two error paths leak the file descriptor opened at line 2. This matches CWE-772 (resource leak) rather than CWE-401 (which is memory-specific).
6. Counter-argument: Process exit would reclaim the handle — but this function may be called repeatedly in a long-running process, exhausting file descriptors.

Output:
```
**Vulnerability Detected: CWE-772 (Missing Release of Resource after Effective Lifetime)**
- Abstraction Class: Resource Lifecycle, File and Path Handling
- Locations: line 5 (malloc failure path), line 8 (parse failure path)
- Root Cause: File handle `fp` opened at line 2 is not closed on two
  early-return error paths. In a long-running process, repeated calls
  exhaust file descriptors.
- Confidence: High
- Remediation: Add `fclose(fp)` before each early return, or use a
  goto-cleanup pattern:
    error:
        if (buf) free(buf);
        if (fp) fclose(fp);
        return -1;

**No additional vulnerabilities found.**
- Memory Management: `malloc`/`free` are correctly paired on all paths.
- Input Validation: `path` is used directly in `fopen`, which is standard
  usage (path validation is the caller's responsibility).
```

**Example 3: False positive correction**

User: "A scanner flagged this as CWE-416 (Use After Free). Is that right?"

```c
void cleanup(struct conn *c) {
    if (c->buf) {
        free(c->buf);
        c->buf = NULL;
    }
    free(c);
}
```

Approach:
1. Entities: `free` (memory deallocation), `c->buf` (nested pointer), `c` (struct pointer), NULL assignment
2. Abstraction classes: Memory Management
3. Candidate CWEs from scanner: CWE-416 (Use After Free)
4. Trace: `c->buf` is freed, then immediately set to NULL. `c` is freed last. No access to `c->buf` occurs after its `free`. No access to `c` occurs after its `free` within this function.
5. Counter-argument is stronger: The NULL assignment after `free(c->buf)` is defensive coding. The access to `c->buf` for the NULL write occurs *between* the two `free` calls, which is a valid access (the struct `c` is still live). No use-after-free exists within this function.

Output:
```
**No Vulnerability: CWE-416 does not apply here.**
- Abstraction Class: Memory Management
- Analysis: The scanner likely flagged the `c->buf = NULL` write after
  `free(c->buf)` as a use-after-free. However, `c->buf` is a *member*
  of struct `c`, and `c` itself has not been freed yet. Writing NULL to
  `c->buf` accesses the struct's memory (still valid), not the freed
  buffer. This is standard defensive cleanup.
- Caveat: If callers access `c` after calling `cleanup()`, that would
  be CWE-416 — but that is a caller-side issue, not in this function.
```

## Best Practices

- **Do:** Always identify the specific CWE, not just the abstraction class. "Memory issue" is insufficient; "CWE-416 Use After Free" is actionable.
- **Do:** Trace data flow from source to sink explicitly. Name the input source, every transformation, and the dangerous sink.
- **Do:** Construct counter-arguments for every finding. If you cannot articulate why the code *might* be safe, your analysis is incomplete.
- **Do:** Distinguish between intra-function vulnerabilities (high confidence) and cross-function vulnerabilities (require more context, lower confidence).
- **Avoid:** Attributing a vulnerability to the wrong CWE. A memory leak (CWE-401) is not a use-after-free (CWE-416). A stack overflow (CWE-121) is not a heap overflow (CWE-122). Precision matters for remediation.
- **Avoid:** Flagging standard library usage as vulnerable without tracing actual data flow. `strcpy` is not inherently a vulnerability — it depends on whether the source is bounded.
- **Avoid:** Producing binary "vulnerable/safe" verdicts without structured reasoning. The reasoning *is* the value.

## Error Handling

- **Incomplete code context:** If the function calls other functions whose implementations are not visible, state this explicitly and mark cross-function flows as lower confidence. Recommend the user provide callers/callees for higher fidelity.
- **Language-specific semantics:** Memory management vulnerabilities apply primarily to C/C++. For managed languages (Java, Python, Go), shift focus to injection, access control, cryptographic, and logic vulnerabilities. Do not report memory leaks in garbage-collected languages unless dealing with native bindings or resource handles.
- **Ambiguous CWE mapping:** When a vulnerability could map to multiple CWEs (e.g., CWE-119 vs CWE-120 vs CWE-121), prefer the most specific CWE. State the parent-child relationship if relevant.
- **Scanner disagreement:** When correcting a scanner's CWE attribution, explain both why the scanner's CWE is wrong and what the correct CWE is (or why the code is actually safe). Never dismiss a scanner finding without analysis.

## Limitations

- **Cross-file and whole-program analysis:** This skill operates best at function-level granularity. Vulnerabilities that span multiple files, involve complex callback chains, or require whole-program alias analysis may be missed or reported at lower confidence.
- **Concurrency vulnerabilities:** Race conditions (CWE-362) and TOCTOU issues (CWE-367) require understanding of execution interleaving that is difficult to reason about from static code alone. Findings in this class should be treated as hypotheses.
- **Custom frameworks and DSLs:** The 13 abstraction classes cover standard security domains. Vulnerabilities specific to custom middleware, proprietary APIs, or domain-specific languages may not map cleanly to existing CWE categories.
- **Quantitative bounds analysis:** Determining whether an integer overflow actually triggers (e.g., whether a multiply can exceed `INT_MAX` given domain constraints) often requires runtime information that is unavailable in static analysis.
- **Not a replacement for tooling:** This reasoning approach complements static analyzers and fuzzers. It excels at explaining *why* code is vulnerable and correcting misattributions, but should not be the sole detection mechanism for production codebases.

## Reference

**Paper:** Mukhtar et al., "VulReaD: Knowledge-Graph-guided Software Vulnerability Reasoning and Detection," arXiv:2602.10787, 2026. Look for: the 13 vulnerability abstraction classes (Section 3), the KG-guided entity-to-CWE mapping pipeline (Section 3.2), and the contrastive reasoning generation with ORPO training (Section 3.3) that demonstrates how structured CWE-anchored reasoning outperforms unconstrained LLM explanations by 8-10% binary F1 and 30% Macro-F1 on multi-class classification.