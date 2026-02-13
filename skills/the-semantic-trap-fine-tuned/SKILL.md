---
name: "the-semantic-trap-fine-tuned"
description: >
  Evaluate code vulnerability detection for semantic traps -- where analysis fixates on
  functional context (e.g., "this is crypto code, so it's probably vulnerable") instead of
  reasoning about the actual root cause of a vulnerability. Applies the TrapEval methodology
  to distinguish genuine vulnerability reasoning from pattern-matching shortcuts.
  Trigger phrases: "check this code for vulnerabilities", "is this patch secure",
  "audit this function for security issues", "compare vulnerable vs patched code",
  "does this fix actually address the vulnerability", "evaluate my vulnerability detector"
---

# The Semantic Trap: Root-Cause Vulnerability Analysis over Pattern Matching

This skill equips Claude to perform vulnerability analysis that targets the actual **root cause** of security flaws rather than relying on superficial functional-domain associations. Based on the TrapEval framework (Huang et al., 2026), it operationalizes the finding that LLMs -- including fine-tuned ones -- routinely fall into "semantic traps": they flag code as vulnerable because it belongs to a security-adjacent domain (cryptography, memory management, network parsing) rather than because they identified the specific logic defect. This skill forces structured root-cause reasoning on every vulnerability assessment, using vulnerable-to-patched (V2P) comparison as the gold standard for whether an identified issue is real.

## When to Use

- When a user asks you to review code for security vulnerabilities and you need to avoid false positives driven by "this looks like risky code"
- When comparing a vulnerable function against its patched version to verify whether a fix actually addresses the root cause
- When a user presents code in a security-sensitive domain (crypto, parsers, memory management) and you must separate domain anxiety from actual flaws
- When auditing a diff or patch and determining if the security-critical logic change is sufficient
- When a user asks "is this vulnerability real or a false positive?" and you need a structured method to answer
- When evaluating whether an automated vulnerability scanner's finding reflects genuine understanding or functional-pattern shortcutting

## Key Technique

**The Semantic Trap Problem.** Fine-tuned vulnerability detectors learn shortcut associations: code that handles buffer operations, cryptographic primitives, or network input gets flagged at higher rates regardless of whether a specific flaw exists. This happens because training datasets pair vulnerable code (drawn from security-relevant domains) against benign code from unrelated domains. The model learns "buffer code = vulnerable" rather than "unchecked length before memcpy = vulnerable." The result: high accuracy on benchmarks that collapses when the model must distinguish vulnerable code from its nearly-identical patched version.

**The V2P Diagnostic.** The core actionable insight is the Vulnerable-to-Patched (V2P) test. Given a vulnerable function and its patched counterpart -- which differ only in the security-critical fix -- can the analysis correctly identify which is vulnerable and articulate *why*? If the reasoning cannot survive this test, it was never real vulnerability understanding; it was domain-pattern matching. A complementary diagnostic is **semantic-preserving perturbation**: rename variables, reorder independent statements, or insert dead code. If the assessment changes, the original reasoning was surface-level.

**Practical Application.** Instead of asking "is this code vulnerable?", the analyst should ask: "What is the specific logical predicate that makes this code exploitable, and does removing exactly that predicate (and nothing else) eliminate the vulnerability?" This forces root-cause identification. Every vulnerability claim must be anchored to a concrete execution path, a specific missing check, or a precise data-flow violation -- not to the code's functional domain.

## Step-by-Step Workflow

1. **Identify the functional domain and set it aside.** Note what the code does (e.g., "parses XML input", "performs AES encryption") but explicitly bracket this information. Domain membership is not evidence of vulnerability. State: "The functional domain is X; this is context, not a finding."

2. **Isolate the security-critical logic.** Identify the specific lines or conditions where a security property could be violated: bounds checks, authentication gates, input validation, resource lifecycle management, cryptographic parameter choices. Focus only on these sites.

3. **Formulate the root-cause hypothesis.** For each candidate vulnerability, state the precise logical defect: "The length of `user_input` is not checked against `buf_size` before the call to `memcpy` at line 47, allowing a heap buffer overflow." The hypothesis must reference specific variables, operations, and control-flow paths.

4. **Apply the V2P test mentally.** Ask: "If I changed *only* the defective logic (e.g., added the bounds check), would the vulnerability disappear while all functional behavior is preserved?" If you cannot construct such a minimal patch, your hypothesis is too vague or wrong.

5. **Apply semantic-preserving perturbation checks.** Mentally rename variables, reorder independent statements, or imagine the same logic with different identifier names. Does your vulnerability assessment survive? If renaming `buf` to `data_buffer` would change your confidence, you were pattern-matching on names, not reasoning about logic.

6. **Trace the exploit path.** Construct a concrete scenario: what input triggers the flaw, what state does the program reach, and what is the security consequence (code execution, information leak, denial of service)? If you cannot articulate this, downgrade confidence.

7. **Check for patch sufficiency.** If a patch is provided, verify it addresses the *exact* root cause from step 3. A patch that adds input validation is insufficient if the root cause is a race condition. Map patch changes to the root-cause hypothesis 1:1.

8. **Rate confidence using the semantic-gap principle.** If the vulnerable and safe versions of the code have high textual similarity (small semantic gap), your analysis must be proportionally more precise. The closer the code looks, the more specific your reasoning must be.

9. **Produce a structured finding.** Output the vulnerability assessment in a format that separates domain context, root-cause logic, exploit path, and confidence -- making it auditable.

## Concrete Examples

**Example 1: Buffer overflow -- real vulnerability vs domain anxiety**

User: "Is this function vulnerable?"
```c
void process_request(char *input, int len) {
    char buffer[256];
    memcpy(buffer, input, len);
    parse_header(buffer);
}
```

Approach:
1. Functional domain: network request processing. Set aside -- not evidence.
2. Security-critical logic: `memcpy(buffer, input, len)` where `buffer` is 256 bytes but `len` is caller-controlled.
3. Root-cause hypothesis: No check that `len <= 256` before `memcpy`. Attacker-controlled `len` can overflow `buffer`.
4. V2P test: Adding `if (len > sizeof(buffer)) return;` before the `memcpy` would eliminate the flaw while preserving function. Minimal patch exists -- hypothesis confirmed.
5. Perturbation check: Renaming `buffer` to `hdr_buf` and `input` to `raw_data` changes nothing about the analysis. Reasoning survives.
6. Exploit path: Caller passes `len = 512`, `memcpy` writes 512 bytes into 256-byte stack buffer, overwriting return address.

Output:
```
VULNERABILITY CONFIRMED
Root Cause: Missing bounds check on `len` parameter before `memcpy` into fixed-size stack buffer (256 bytes).
  Location: Line 3, `memcpy(buffer, input, len)`
  Defective predicate: No guard `len <= sizeof(buffer)`
Exploit path: Attacker supplies len > 256 -> stack buffer overflow -> potential RCE
Minimal fix: Add `if (len > sizeof(buffer)) return -1;` before memcpy
Confidence: HIGH -- root cause is specific, exploit path is concrete, V2P test passes
```

**Example 2: Crypto code that looks scary but is fine**

User: "Is this vulnerable? It's doing AES and I'm worried."
```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
import os

def encrypt(plaintext: bytes, key: bytes) -> bytes:
    iv = os.urandom(16)
    cipher = Cipher(algorithms.AES(key), modes.CBC(iv))
    encryptor = cipher.encryptor()
    padded = _pkcs7_pad(plaintext, 16)
    ct = encryptor.update(padded) + encryptor.finalize()
    return iv + ct
```

Approach:
1. Functional domain: AES-CBC encryption. Set aside -- "crypto code" is not a finding.
2. Security-critical logic: IV generation (`os.urandom(16)` -- proper), mode choice (CBC with random IV -- acceptable), padding (PKCS7 -- standard), key handling (passed in, not hardcoded).
3. Root-cause hypothesis: None identified. IV is random, mode is standard, padding is correct.
4. V2P test: Cannot construct a minimal patch that would "fix" anything -- no defect to remove.
5. Perturbation check: Renaming variables does not reveal any hidden concern.

Output:
```
NO VULNERABILITY FOUND
Domain note: This is cryptographic code, which warrants scrutiny, but domain membership alone is not a finding.
Analysis: IV is generated with os.urandom (CSPRNG), AES-CBC with random IV is acceptable for
  confidentiality, PKCS7 padding is standard. No hardcoded keys, no IV reuse, no ECB mode.
Potential concerns (informational, not vulnerabilities):
  - CBC mode does not provide authentication. Consider AES-GCM if integrity is also required.
  - Caller is responsible for key management -- not assessed here.
Confidence: HIGH -- no root-cause defect identified despite domain scrutiny
```

**Example 3: Evaluating whether a patch actually fixes the vulnerability**

User: "We patched CVE-2024-XXXX. Does this diff fix it?"
```diff
 int handle_packet(struct packet *pkt) {
-    if (pkt->type == PKT_DATA) {
+    if (pkt->type == PKT_DATA && pkt->length <= MAX_PKT_SIZE) {
         process_data(pkt->payload, pkt->length);
     }
     return 0;
 }
```

Approach:
1. Identify the root cause of the original vulnerability: `process_data` is called with `pkt->length` without verifying it does not exceed `MAX_PKT_SIZE`. If `process_data` uses `length` to index into a fixed buffer, this is an overflow.
2. Map the patch to the root cause: The patch adds `pkt->length <= MAX_PKT_SIZE` as a guard. This directly addresses the missing bounds check.
3. V2P analysis: The patched version differs from the vulnerable version in exactly the security-critical predicate. All functional behavior for valid packets is preserved.
4. Check sufficiency: Does `MAX_PKT_SIZE` correctly correspond to the buffer size inside `process_data`? If yes, the fix is sufficient. If `process_data` has a different internal limit, the patch may be incomplete.

Output:
```
PATCH ASSESSMENT: LIKELY SUFFICIENT (with caveat)
Root cause addressed: Missing bounds check on pkt->length before passing to process_data.
Patch mechanism: Added guard `pkt->length <= MAX_PKT_SIZE` in the conditional.
Residual risk: Verify that MAX_PKT_SIZE matches the actual buffer size used inside process_data().
  If process_data allocates dynamically based on length, the check is redundant but harmless.
  If process_data uses a fixed buffer of size != MAX_PKT_SIZE, the patch is insufficient.
Confidence: MEDIUM-HIGH -- root cause is addressed, but constant correctness needs verification
```

## Best Practices

- **Do:** Always state the root cause as a specific logical predicate ("missing check X before operation Y") rather than a domain label ("this is unsafe crypto code").
- **Do:** Apply the V2P mental test on every finding -- if you cannot describe a minimal patch that fixes only the vulnerability, your analysis is too vague.
- **Do:** Separate informational domain concerns (e.g., "CBC doesn't provide authentication") from actual vulnerability findings (e.g., "IV is hardcoded, enabling chosen-plaintext attacks").
- **Do:** When analyzing a patch, map each changed line to a specific root-cause element. Unaccounted-for changes may indicate the patch is doing more or less than intended.
- **Avoid:** Flagging code as vulnerable solely because it operates in a security-sensitive domain (crypto, parsing, memory management). This is the semantic trap.
- **Avoid:** Changing your assessment based on variable names, comment text, or code style. If renaming `password` to `token` would change your finding, the finding was never about logic.
- **Avoid:** Reporting vulnerabilities without a concrete exploit path. "This could potentially be unsafe" is not a finding -- specify what input triggers what consequence.

## Error Handling

- **Incomplete code context:** If the function calls other functions you cannot see, state explicit assumptions: "Assuming `process_data` copies `length` bytes into a fixed buffer..." and flag this as reducing confidence.
- **Multiple interacting vulnerabilities:** When code has several flaws, analyze each independently with its own root cause. Do not conflate them -- a bounds check fix does not address a separate use-after-free.
- **Language/framework-specific semantics:** Some languages prevent certain vulnerability classes (e.g., Python prevents buffer overflows in native strings). Adjust analysis to the language's memory model and standard library guarantees.
- **Ambiguous patches:** If a diff changes functional behavior alongside the security fix, flag the functional changes separately and assess whether they introduce new attack surface.

## Limitations

- This methodology is strongest for **logic-level vulnerabilities** (missing checks, incorrect conditions, race conditions). It is less directly applicable to architectural flaws (insecure design patterns) or configuration issues.
- The V2P test requires you to envision or have access to the patched version. For novel vulnerability classes where the fix is non-obvious, the test is harder to apply.
- Semantic-preserving perturbation reasoning works for code you can read and reason about. For obfuscated code or extremely large functions, the manual approach does not scale -- static analysis tools are needed as a complement.
- This skill improves **precision** (reducing false positives from domain-pattern matching) but does not guarantee **recall** -- subtle vulnerabilities may still be missed if the security-critical logic is deeply buried.

## Reference

Huang, F., Sun, Y., Zhang, F., Yang, Z., & Liu, H. (2026). *The Semantic Trap: Do Fine-tuned LLMs Learn Vulnerability Root Cause or Just Functional Pattern?* arXiv:2601.22655v2. https://arxiv.org/abs/2601.22655v2

Key takeaway: Fine-tuned LLMs achieve high vulnerability detection scores by associating functional domains with vulnerability likelihood (the "semantic trap") rather than learning root-cause reasoning. The TrapEval framework -- particularly the V2P (Vulnerable-to-Patched) paired evaluation -- is the diagnostic that exposes this failure. Apply V2P reasoning to every vulnerability assessment to avoid the same trap.