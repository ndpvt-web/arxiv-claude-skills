---
name: "vulread-knowledge-graph-guided-software-vulnerabil"
description: "CWE-guided vulnerability reasoning and detection using knowledge-graph-structured analysis. Analyzes source code for security vulnerabilities with structured CWE-level explanations grounded in a security knowledge graph. Triggers: 'analyze this code for vulnerabilities', 'find CWE issues in this function', 'security audit with CWE classification', 'explain the vulnerability in this code', 'what CWE does this bug fall under', 'check this C function for memory safety issues'"
---

# VulReaD: Knowledge-Graph-Guided Software Vulnerability Reasoning and Detection

This skill enables Claude to perform structured vulnerability analysis on source code by following the VulReaD methodology: instead of making binary vulnerable/not-vulnerable judgments, Claude extracts security-relevant entities from code (API calls, variable names, file paths, memory operations), maps them to vulnerability abstraction classes (e.g., Memory Management, Input Validation, Access Control), and traces those classes to specific CWE identifiers. This produces explanations that are semantically grounded in the CWE taxonomy rather than surface-level pattern matching.

## When to Use

- When the user asks to audit a function or file for security vulnerabilities and wants CWE-level classification, not just "this looks unsafe"
- When reviewing C/C++ code for memory safety issues (buffer overflows, use-after-free, memory leaks) and needing precise CWE attribution
- When the user wants to understand *why* code is vulnerable with a structured explanation linking code entities to specific weakness categories
- When triaging vulnerability reports and needing to verify whether a reported CWE matches the actual code behavior
- When analyzing code diffs or patches to confirm whether a fix actually addresses the claimed CWE
- When building or reviewing static analysis rules and needing CWE-consistent reasoning to validate detection logic

## Key Technique

VulReaD's core insight is that LLMs produce fluent but often semantically misaligned vulnerability explanations. A model might flag code as vulnerable to CWE-763 (Use-After-Free) when the actual issue is CWE-401 (Memory Leak) -- the explanation reads convincingly but maps to the wrong weakness. VulReaD fixes this by grounding all reasoning in a security knowledge graph that enforces entity-to-class-to-CWE alignment.

The knowledge graph organizes vulnerabilities through 13 abstraction classes: File and Path Handling, Input Validation, Access Control, Memory Management, Cryptographic Operations, Error Handling, Resource Management, Concurrency, Numeric Operations, Code Injection, Information Disclosure, Configuration Management, and Authentication/Authorization. Each class captures a family of security flaws and maps to specific CWE identifiers. Code entities (function calls like `malloc`, `strcpy`, `fopen`; parameters; variable names) are linked to these classes, and the classes are linked to CWEs via both keyword matching and embedding-based similarity.

The contrastive reasoning approach is what makes VulReaD's explanations reliable. For each code sample, the method generates two reasonings: a *valid* reasoning conditioned on the true label (extracting entities, mapping to the correct abstraction class, citing the right CWE) and a *flawed* reasoning produced by conditioning on the wrong label (which generates plausible-sounding but structurally incorrect explanations). Training the model to prefer valid over flawed reasoning -- using Odds Ratio Preference Optimization -- teaches it to distinguish genuinely grounded explanations from superficially convincing ones.

## Step-by-Step Workflow

1. **Extract security-relevant entities from the code.** Identify all function calls, API invocations, library references, variable names, file path operations, memory allocations/deallocations, and input-handling operations. List each entity explicitly.

2. **Map each entity to one or more of the 13 vulnerability abstraction classes.** Use these categories:
   - Memory Management: `malloc`, `free`, `realloc`, `new`, `delete`, pointer arithmetic
   - Input Validation: `scanf`, `gets`, `argv`, `getenv`, user-controlled buffers
   - File and Path Handling: `fopen`, `open`, `stat`, path concatenation, symlink operations
   - Access Control: permission checks, `setuid`, capability operations
   - Error Handling: unchecked return values, missing null checks, exception handling gaps
   - Resource Management: file descriptors, sockets, locks not released
   - Concurrency: shared state without synchronization, race conditions, TOCTOU patterns
   - Numeric Operations: integer overflow, truncation, signed/unsigned mismatches
   - Code Injection: `system()`, `exec()`, SQL string building, `eval()`
   - Cryptographic Operations: weak algorithms, hardcoded keys, insufficient randomness
   - Information Disclosure: logging sensitive data, error messages exposing internals
   - Configuration Management: hardcoded credentials, insecure defaults
   - Authentication/Authorization: missing auth checks, broken session management

3. **Trace each abstraction class to candidate CWE identifiers.** For Memory Management entities, consider CWE-119 (Buffer Overflow), CWE-125 (Out-of-bounds Read), CWE-416 (Use-After-Free), CWE-401 (Memory Leak), CWE-415 (Double Free), CWE-476 (NULL Pointer Dereference). For Input Validation, consider CWE-20 (Improper Input Validation), CWE-787 (Out-of-bounds Write), CWE-120 (Buffer Copy without Size Check). Match based on the specific entity behavior, not just its category.

4. **Analyze data flow and control flow around each entity.** Track where user-controlled input enters, how it propagates through variables and function calls, and whether sanitization or bounds checking occurs before reaching a security-sensitive operation. This determines whether a potential weakness is actually reachable.

5. **Produce a structured vulnerability assessment.** For each identified issue, state: (a) the specific code entity, (b) its abstraction class, (c) the CWE identifier with rationale for why *this* CWE and not a related one, and (d) a concise description of the vulnerability mechanism.

6. **Generate contrastive validation.** For each finding, briefly consider the most likely *misclassification* -- the CWE that a surface-level analysis might incorrectly assign -- and explain why the chosen CWE is correct instead. This prevents semantic drift.

7. **Assess severity and exploitability.** Note whether the vulnerability requires specific preconditions (e.g., attacker-controlled input reaching the function, specific memory layout) and whether it is exploitable in practice.

8. **Recommend specific fixes.** For each CWE finding, suggest a concrete code change that addresses the root cause, not just a symptom.

## Concrete Examples

**Example 1: Memory Leak Detection with Correct CWE Attribution**

User: "Check this function for vulnerabilities"
```c
char* read_config(const char* filename) {
    FILE* fp = fopen(filename, "r");
    if (!fp) return NULL;
    char* buf = malloc(1024);
    if (!buf) return NULL;  // fp leaked here
    if (fread(buf, 1, 1024, fp) == 0) {
        free(buf);
        return NULL;  // fp leaked here too
    }
    fclose(fp);
    return buf;
}
```

Approach:
1. Extract entities: `fopen`, `malloc`, `fread`, `free`, `fclose`, `return NULL` statements
2. Map: `fopen`/`fclose` -> Resource Management + File Handling; `malloc`/`free` -> Memory Management
3. Trace data flow: `fp` is opened on line 2 but not closed before `return NULL` on lines 4 and 7

Output:
```
## Vulnerability Analysis

### Finding 1: Resource Leak — CWE-775 (Missing Release of File Descriptor after Effective Lifetime)
- **Entity**: `fopen()` return value stored in `fp`
- **Abstraction Class**: Resource Management
- **Mechanism**: When `malloc` fails (line 4) or `fread` returns 0 (line 6-8),
  the function returns without calling `fclose(fp)`. The file descriptor leaks.
- **Why CWE-775 and not CWE-401**: CWE-401 covers memory leaks (heap allocations).
  The leaked resource here is a file descriptor, not heap memory. The `malloc`
  allocation is correctly freed on the error path at line 7.
- **Severity**: Medium. Repeated calls exhaust file descriptor limits (typically 1024).
- **Fix**: Add `fclose(fp)` before each early return after successful `fopen`:

  if (!buf) { fclose(fp); return NULL; }
  ...
  if (fread(...) == 0) { free(buf); fclose(fp); return NULL; }
```

**Example 2: Distinguishing Buffer Overflow Variants**

User: "What CWE applies to this code?"
```c
void process_input(char* user_data) {
    char local_buf[64];
    int len = strlen(user_data);
    memcpy(local_buf, user_data, len);  // no bounds check
    local_buf[len] = '\0';
}
```

Approach:
1. Extract entities: `strlen`, `memcpy`, stack buffer `local_buf[64]`, `user_data` (external input)
2. Map: `memcpy` with unchecked length -> Memory Management + Input Validation
3. Trace: `user_data` is caller-controlled, `len` is unbounded, `memcpy` copies directly to a 64-byte stack buffer

Output:
```
## Vulnerability Analysis

### Finding 1: Stack Buffer Overflow — CWE-121 (Stack-based Buffer Overflow)
- **Entity**: `memcpy(local_buf, user_data, len)` where `len = strlen(user_data)`
- **Abstraction Class**: Memory Management, Input Validation
- **Mechanism**: `user_data` is externally controlled. `strlen(user_data)` can
  exceed 64, causing `memcpy` to write past `local_buf` boundaries on the stack.
- **Why CWE-121 and not CWE-120 or CWE-787**:
  - CWE-120 (Buffer Copy without Checking Size of Input) is the *cause* pattern
  - CWE-787 (Out-of-bounds Write) is the generic parent
  - CWE-121 is most precise: the target is a stack-allocated buffer, enabling
    return address overwrite and code execution
- **Severity**: Critical. Stack buffer overflows allow arbitrary code execution.
- **Fix**: Bound the copy length:
  size_t len = strlen(user_data);
  if (len >= sizeof(local_buf)) len = sizeof(local_buf) - 1;
  memcpy(local_buf, user_data, len);
  local_buf[len] = '\0';
```

**Example 3: Contrastive Reasoning Catches Misclassification**

User: "Is this vulnerable? A scanner flagged it as use-after-free."
```c
void update_cache(struct cache* c, int key) {
    struct entry* e = cache_lookup(c, key);
    if (e) {
        cache_remove(c, key);   // frees e internally
        log_removal(e->value);  // accesses freed memory
    }
}
```

Approach:
1. Extract entities: `cache_lookup`, `cache_remove`, `e->value` dereference after removal
2. Map: pointer `e` used after `cache_remove` frees it -> Memory Management
3. Contrastive check: the scanner's CWE-416 (Use-After-Free) classification is correct here

Output:
```
## Vulnerability Analysis

### Finding 1: Use-After-Free — CWE-416 (Use After Free)
- **Entity**: `e->value` accessed after `cache_remove(c, key)` on line 5
- **Abstraction Class**: Memory Management
- **Mechanism**: `cache_remove` frees the entry struct internally. The subsequent
  `log_removal(e->value)` dereferences freed memory via the dangling pointer `e`.
- **Scanner validation**: The CWE-416 classification is correct.
  - CWE-416 (not CWE-415/Double Free): only one free occurs; the issue is
    subsequent *use*, not a second free
  - CWE-416 (not CWE-476/NULL Deref): `e` is non-null (checked on line 3)
    but *freed*, not null
- **Severity**: High. UAF can lead to arbitrary code execution if the freed
  memory is reallocated with attacker-controlled content.
- **Fix**: Capture the value before removal:
  int val = e->value;
  cache_remove(c, key);
  log_removal(val);
```

## Best Practices

**Do:**
- Always extract concrete code entities before classifying -- never jump directly from "this looks like a memory bug" to a CWE number. Ground every classification in specific identifiable operations.
- Perform contrastive validation: for each CWE assignment, explicitly name the most likely *wrong* CWE and explain why it does not apply. This is the single most impactful step from VulReaD.
- Distinguish between the *cause* CWE (e.g., CWE-120: buffer copy without size check) and the *consequence* CWE (e.g., CWE-121: stack buffer overflow). Report the most specific applicable CWE.
- Track data flow from input sources to security-sensitive sinks. A dangerous function call without reachable attacker-controlled input is a code smell, not a vulnerability.

**Avoid:**
- Do not assign CWEs based on function names alone. `strcpy` is not automatically CWE-120; it depends on whether the source is bounded and the destination is appropriately sized.
- Do not conflate related CWEs. CWE-401 (Memory Leak) and CWE-416 (Use-After-Free) both involve memory management but have opposite mechanisms -- one is failure to free, the other is use after freeing. Mixing them undermines the entire analysis.
- Do not generate "fluent but misaligned" explanations. If you cannot trace a clear entity -> abstraction class -> CWE path, say so explicitly rather than producing a confident-sounding wrong classification.
- Do not ignore the "non-vulnerable" conclusion. Thoroughly analyzed code that turns out to be safe is a valid and valuable result. Not every suspicious pattern is exploitable.

## Error Handling

- **Incomplete code context**: When analyzing a single function without visibility into called functions (e.g., does `cache_remove` actually free the entry?), state assumptions explicitly: "Assuming `cache_remove` frees the entry internally, this is CWE-416. If it only unlinks without freeing, this is not a vulnerability."
- **Ambiguous CWE mapping**: When an entity maps to multiple plausible CWEs with similar confidence, list all candidates with the distinguishing condition for each. Let the user resolve based on broader codebase knowledge.
- **Language-specific gaps**: The 13 abstraction classes were designed primarily around C/C++ vulnerabilities. For managed languages (Java, Python, Go), skip Memory Management checks for heap issues and focus on Input Validation, Code Injection, Access Control, and Information Disclosure classes.
- **Large functions**: For functions exceeding 100 lines, break the analysis into segments around each security-sensitive operation rather than attempting a single holistic assessment. Track data flow across segments.

## Limitations

- This approach works best for C/C++ code where memory safety CWEs dominate. For web application vulnerabilities (XSS, CSRF, SSRF), the abstraction classes still apply but the entity extraction step requires adaptation to HTTP handlers, template rendering, and request parsing.
- CWE assignment precision depends on available context. Analyzing an isolated function without its callers or callees limits data-flow tracing and may produce false positives or overly broad CWE assignments.
- The 13 abstraction classes do not cover every CWE. Vulnerabilities in areas like cryptographic protocol design, business logic flaws, or race conditions in distributed systems may not map cleanly.
- This is static reasoning over source code. It cannot detect vulnerabilities that depend on runtime state, specific compiler behavior, or hardware-level conditions (e.g., speculative execution side channels).

## Reference

**Paper**: [VulReaD: Knowledge-Graph-guided Software Vulnerability Reasoning and Detection](https://arxiv.org/abs/2602.10787v1) (Mukhtar et al., 2026). Key sections: Section 3 (KG construction and 13 abstraction classes), Section 4 (contrastive reasoning pair generation), Section 5 (ORPO training objective). The core takeaway for practitioners is the contrastive validation pattern -- always check your CWE assignment against the most likely misclassification.