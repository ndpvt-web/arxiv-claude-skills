---
name: "beyond-function-level-analysis-context-aware"
description: |
  Inter-procedural vulnerability detection using context-aware reasoning. Analyzes functions
  alongside their callers, callees, and global state to find vulnerabilities that single-function
  analysis misses. Uses code property graph traversal, security-focused context profiling,
  relevance scoring, and structured reasoning traces.

  Trigger phrases:
  - "Check this code for vulnerabilities across function boundaries"
  - "Analyze this function with its callers and callees for security issues"
  - "Find inter-procedural vulnerabilities in this codebase"
  - "Review this code for vulnerabilities that depend on how it's called"
  - "Do a deep security audit with cross-function context"
  - "Analyze whether this function is safe given how callers use it"
---

# Context-Aware Inter-Procedural Vulnerability Detection (CPRVul)

This skill enables Claude to detect vulnerabilities that are invisible at the function level by systematically gathering inter-procedural context — callers, callees, global variables — profiling each for security relevance, and applying structured reasoning over the combined evidence. Based on the CPRVul framework, this approach improved detection accuracy by 22.9% over function-only baselines on the PrimeVul benchmark. The core insight: raw context hurts; only *profiled, scored, and reasoned-over* context helps.

## When to Use

- When the user asks to review a function for vulnerabilities and the function takes user-controlled input from callers, delegates security-critical work to callees, or reads/writes shared global state
- When a security audit requires understanding how data flows across function boundaries (e.g., "is this buffer size validated before reaching this memcpy?")
- When triaging a CVE fix to determine if a patch is complete — the vulnerability root cause may be in a caller or callee, not the patched function itself
- When reviewing PRs that modify functions called from many sites, where a change could introduce vulnerabilities depending on caller assumptions
- When analyzing C/C++ code for memory safety issues (CWE-119, CWE-416, CWE-476, CWE-787) that frequently involve inter-procedural data flow
- When the user asks "is this function safe?" and the answer depends on preconditions enforced (or not) by callers

## Key Technique

**Why function-level analysis fails.** Most vulnerability detectors examine one function in isolation and predict "vulnerable or not." But real vulnerabilities often depend on context: a function that trusts its input is only vulnerable if callers pass unsanitized data; a null-pointer dereference only matters if callers can trigger the null path; a lock release is only dangerous if the caller didn't check for null first. Naively appending all caller/callee code makes things worse — the noise overwhelms the signal, and fine-tuned models actually perform *worse* with raw context than without it.

**Context Profiling and Selection.** CPRVul first builds a code property graph (CPG) for the repository and extracts three categories of context for a target function: (1) **callers** — every function that invokes the target, (2) **callees** — every function the target invokes, and (3) **global variables** the target reads or writes. Each context element is then *profiled* through security-focused summarization: for callers, the profile captures data origin (user input, network, file), transformations applied (sanitized vs. raw), and how return values are used; for callees, it captures security risk level with justification; for globals, it captures the role and security implications. Each profiled element receives a relevance score, and only the highest-scoring elements that fit the analysis budget are retained.

**Structured Reasoning.** The selected context, target function, and vulnerability metadata (CWE classification, CVE description, commit context) are combined into a structured reasoning trace with five fields: `observation` (what the code does), `security_reasoning` (why it is or isn't vulnerable given context), `impact` (consequences), `is_vulnerable` (boolean verdict), and `confidence_score` (0-10). This structure prevents the model from pattern-matching on surface features and forces it to articulate the causal chain from context through to exploit.

## Step-by-Step Workflow

1. **Identify the target function.** Determine which function the user wants analyzed. Read it fully and note its parameters, return type, memory operations, and any security-sensitive API calls (allocation, I/O, crypto, string manipulation, pointer arithmetic).

2. **Extract callers.** Search the codebase for every call site of the target function. For each caller, read the relevant code surrounding the call. Focus on: what data is passed as arguments, whether inputs are validated or sanitized before the call, and how the return value is checked.

3. **Extract callees.** Identify every function the target invokes. Read each callee's implementation. Focus on: whether the callee performs bounds checking, whether it can fail or return null, whether it has security-relevant side effects (memory allocation, lock acquisition, privilege changes).

4. **Extract global state.** Find global variables, shared structs, and static state that the target reads or writes. Determine whether other functions modify this state concurrently and whether the target assumes invariants about it.

5. **Profile each context element for security relevance.** For each caller, callee, and global variable, write a concise security profile:
   - **Callers:** Data origin (user input / network / trusted internal), transformation status (sanitized / raw / partially validated), return value handling (checked / ignored / propagated)
   - **Callees:** Security risk level (high / medium / low) with one-line justification
   - **Globals:** Role (configuration / shared buffer / lock state) and mutation risk

6. **Score and select context.** Assign each profiled element a relevance score (0-10) based on how directly it affects whether the target function is exploitable. Retain only elements scoring 7+ or the top 3-5 elements if many qualify. Discard boilerplate, logging-only callers, and pure-utility callees with no security surface.

7. **Assemble the analysis input.** Combine: (a) the target function source, (b) the selected context profiles with code excerpts, and (c) any available vulnerability metadata (CWE category if suspected, known vulnerability patterns for the code pattern).

8. **Generate a structured reasoning trace.** Produce a verdict with these five fields:
   - `observation`: Factual description of what the target function does and what context reveals
   - `security_reasoning`: The causal argument — how does the context make this vulnerable or safe? What preconditions are met or violated?
   - `impact`: What could an attacker achieve if the vulnerability is real (denial of service, code execution, information leak)
   - `is_vulnerable`: true/false
   - `confidence_score`: 0-10 with justification for uncertainty

9. **Validate with counterfactual reasoning.** If the verdict is "not vulnerable," explicitly reason about what would make it vulnerable (e.g., "if caller X stopped validating the length parameter, this memcpy would overflow"). If the verdict is "vulnerable," reason about what defense would fix it and whether any caller already provides that defense.

10. **Report findings with full traceability.** Present the verdict with specific file:line references for the target, the relevant callers/callees, and the exact data flow path that constitutes the vulnerability (or the defense that prevents it).

## Concrete Examples

**Example 1: Null-pointer dereference across function boundary (CWE-476)**

User: "Is `put_busid_priv()` in drivers/usb/usbip safe?"

Approach:
1. Read `put_busid_priv()` — it calls `spin_unlock(&busid_table_lock)` and accesses `bid->status`
2. Extract callers — `stub_probe()` and `stub_disconnect()` both call it
3. Profile callers: `stub_disconnect()` can pass a NULL `bid` when the device is already unbound
4. Score: `stub_disconnect()` caller scores 9/10 (directly controls NULL path)
5. Profile callee: `spin_unlock` has no NULL guard — scores 8/10

Output:
```json
{
  "observation": "put_busid_priv() accesses bid->status and releases bid->busid_lock without null-checking bid. Caller stub_disconnect() can invoke this with bid=NULL when device is already unbound.",
  "security_reasoning": "stub_disconnect() calls busid_priv_get() which returns NULL when the bus ID is not found. This NULL pointer is passed directly to put_busid_priv(). The function dereferences bid without a guard, causing a kernel NULL pointer dereference.",
  "impact": "Kernel panic / denial of service. Local attacker with USB access can trigger by rapidly unbinding a USBIP device.",
  "is_vulnerable": true,
  "confidence_score": 9
}
```
Fix: Add `if (!bid) return;` at the top of `put_busid_priv()`.

---

**Example 2: Incorrect return value with caller-dependent impact (CWE-682)**

User: "Review `Item_direct_view_ref::const_item()` in MariaDB for correctness."

Approach:
1. Read `const_item()` — it unconditionally returns `true`
2. Extract callers — the query optimizer's condition pushdown logic calls `const_item()` to decide whether an expression can be evaluated early
3. Profile callers: The optimizer trusts `const_item()` return value to push conditions into derived tables. Score: 10/10 — directly determines optimization correctness
4. Extract callees: `const_item()` should check `used_tables() == 0` but doesn't call it
5. The correct implementation delegates to `(*ref)->const_item()` or checks `used_tables()`

Output:
```json
{
  "observation": "const_item() returns true unconditionally instead of checking used_tables() == 0. This tells the optimizer that a non-constant expression is constant.",
  "security_reasoning": "Callers in the query optimizer use this to push conditions into subqueries/derived tables. When a non-constant expression is treated as constant, the optimizer can evaluate it with wrong row context, producing incorrect query results. This is exploitable for data exfiltration if query results are user-visible.",
  "impact": "Incorrect query results. Potential information disclosure if exploited in multi-tenant query paths.",
  "is_vulnerable": true,
  "confidence_score": 8
}
```

---

**Example 3: Safe function confirmed via caller analysis**

User: "Is this `parse_header()` function vulnerable to buffer overflow?"

Approach:
1. Read `parse_header()` — uses `memcpy(buf, src, len)` where `len` comes from a parameter
2. Extract callers — two callers found: `handle_request()` and `process_batch()`
3. Profile callers: Both callers validate `len <= MAX_HEADER_SIZE` before calling. `handle_request()` gets `len` from network input but clamps it. `process_batch()` computes `len` from a trusted internal struct.
4. Score: Both callers score 8/10 for relevance but provide defensive validation
5. Counterfactual: If any future caller passes unchecked `len`, the memcpy overflows

Output:
```json
{
  "observation": "parse_header() copies len bytes into buf via memcpy without internal bounds checking. Both current callers validate len <= MAX_HEADER_SIZE before invoking.",
  "security_reasoning": "The function itself lacks bounds checking, but all current callers enforce the precondition. The function is safe in its current call context but is fragile — any new caller that omits the length check introduces a buffer overflow.",
  "impact": "No current impact. Latent risk if calling convention changes.",
  "is_vulnerable": false,
  "confidence_score": 7
}
```
Recommendation: Add a `assert(len <= MAX_HEADER_SIZE)` or explicit check inside `parse_header()` for defense in depth.

## Best Practices

- **Do:** Always profile context *before* reasoning. Raw caller/callee code without security profiling adds noise and leads to worse analysis than no context at all.
- **Do:** Score and rank context elements. Not all callers matter equally — prioritize those handling external input or controlling security-critical parameters.
- **Do:** Use counterfactual reasoning in both directions. For "safe" verdicts, articulate what would make it unsafe. For "vulnerable" verdicts, check if any existing defense was overlooked.
- **Do:** Trace specific data flow paths. Name the variable, the caller that sets it, the callee that consumes it, and what goes wrong.
- **Avoid:** Dumping all callers and callees into the analysis without filtering. The paper shows this *degrades* detection accuracy.
- **Avoid:** Making verdicts based on the target function alone when the user asked for inter-procedural analysis. A function that looks safe in isolation may be dangerous given its callers.
- **Avoid:** Conflating "no current callers pass bad input" with "the function is safe." Flag latent vulnerabilities with defense-in-depth recommendations.

## Error Handling

- **Cannot find callers/callees:** If the codebase is incomplete or the function is in a library with unknown callers, state this limitation explicitly. Analyze the function under worst-case caller assumptions (untrusted input, unchecked return values) and note that the verdict is conditional.
- **Too many callers to analyze:** For heavily-called utility functions (e.g., `malloc` wrappers), sample callers from security-sensitive code paths (network handlers, parsers, authentication) rather than analyzing all call sites.
- **Context exceeds practical limits:** If the combined context is too large, strictly use relevance scoring to keep only elements scoring 7+. Summarize excluded elements as "N additional callers examined, none handling external input."
- **Conflicting evidence from different callers:** If some callers are safe and others are not, report the vulnerability as real — a single exploitable call path is sufficient.

## Limitations

- This approach requires access to the broader codebase, not just an isolated function snippet. If the user provides only a single function with no surrounding code, fall back to function-level analysis and note the limitation.
- Inter-procedural analysis through indirect calls (function pointers, virtual dispatch, callbacks) is inherently incomplete. Flag these as "context gaps" when encountered.
- The technique is most effective for C/C++ memory safety and logic vulnerabilities (CWE-119, CWE-416, CWE-476, CWE-787, CWE-190). It is less proven for web application vulnerabilities (XSS, SQLi) where taint tracking frameworks are more appropriate.
- For very large codebases (10K+ callers), exhaustive analysis is impractical. The scoring/selection phase is essential — skipping it will produce worse results than function-only analysis.
- The structured reasoning format helps rigor but cannot substitute for domain expertise in cryptography, protocol design, or concurrency bugs where the vulnerability patterns are qualitatively different.

## Reference

**Paper:** [Beyond Function-Level Analysis: Context-Aware Reasoning for Inter-Procedural Vulnerability Detection](https://arxiv.org/abs/2602.06751v1) — Li et al., 2026. Look for: Table 2 (reasoning trace templates), Section 3 (context profiling pipeline), and the ablation in Section 5 showing that processed context + structured reasoning is the only combination that improves over baselines.