---
name: "identifying-adversary-tactics-techniques"
description: "Identify MITRE ATT&CK Tactics, Techniques, and Procedures (TTPs) in decompiled malware binaries using the TTPDetect methodology: hybrid retrieval for entry point narrowing, incremental context exploration along call graphs, and TTP-specific reasoning guidelines for precise attribution. Use when: 'map this binary to MITRE ATT&CK', 'identify TTPs in this decompiled code', 'what malware techniques does this function implement', 'analyze this stripped binary for adversary tactics', 'attribute MITRE techniques to these functions', 'generate a TTP report for this malware sample'."
---

# Identifying Adversary Tactics and Techniques in Malware Binaries

This skill enables Claude to systematically identify MITRE ATT&CK TTPs (Tactics, Techniques, and Procedures) in decompiled or disassembled malware code using the TTPDetect methodology. The approach combines hybrid retrieval to find relevant analysis entry points across large binaries, on-demand context exploration along call graphs to resolve partial observability, and TTP-specific reasoning guidelines that distinguish behavior-based TTPs from intent-critical TTPs -- avoiding the false positives that plague naive pattern matching.

## When to Use

- When the user provides decompiled C/C++ code (e.g., from IDA Pro, Ghidra, or Binary Ninja) and asks what MITRE ATT&CK techniques it implements
- When analyzing a stripped binary's function listings to produce a TTP coverage report for threat intelligence
- When the user asks to triage which functions in a large binary are security-relevant and map them to specific ATT&CK technique IDs
- When reviewing malware analysis reports and validating or expanding the documented TTP attributions
- When building automated malware analysis pipelines that need structured TTP output
- When the user wants to distinguish superficially similar techniques (e.g., T1070 Indicator Removal vs. T1562 Impair Defenses) in decompiled code

## Key Technique

**TTPDetect** (Xuan et al., 2026) is the first LLM agent designed for TTP recognition in stripped malware binaries. The core challenge is that real-world malware has thousands of functions with address-based names, malicious behavior spread across multiple code regions, and no symbols or debug information. Directly prompting an LLM with all decompiled code fails because of context limits, noisy compiler-generated functions, and misalignment between general code understanding and TTP-specific decision logic.

The method uses a three-stage pipeline. **Stage 1 (Binary Renaming)** walks the call graph bottom-up, assigning semantic names to stripped functions by analyzing their decompiled code and callee summaries. **Stage 2 (Hybrid Retrieval)** narrows the analysis space through two complementary channels: dense retrieval embeds both TTP definitions and renamed functions into a shared vector space (using code-capable embeddings) to retrieve top-k candidates per TTP, while neural retrieval uses the LLM to generate over-inclusive TTP candidate sets per function with confidence filtering. This combination reduces candidate pairs by ~86% compared to dense retrieval alone. **Stage 3 (Function-Level Analysis)** deploys two components: a **Context Explorer** that selectively traverses callers/callees to gather surrounding context on demand (avoiding exhaustive call graph traversal), and **TTP-Specific Reasoning Guidelines** -- pre-synthesized checklists that classify each TTP as either *behavior-focused* (a characteristic action suffices, e.g., T1055 Process Injection) or *intent-critical* (adversarial purpose must be evidenced, e.g., T1082 System Information Discovery), with positive/negative contrasts and required decision criteria. This inference-time alignment is what drives the 10%+ precision gain over direct prompting.

## Step-by-Step Workflow

1. **Rename stripped functions semantically.** For each function in the decompiled output, assign a descriptive name based on its operations, API calls, string references, and callee summaries. Process in bottom-up call graph order: leaf functions first, then their callers. For cycles, assign provisional names and revisit after accumulating context.

2. **Build a TTP reference knowledge base.** For each candidate MITRE ATT&CK technique, assemble: the formal definition from MITRE ATT&CK, all sub-techniques, natural-language procedure examples, and classify the TTP as *behavior-focused* (characteristic action sufficient) or *intent-critical* (requires adversarial purpose evidence).

3. **Perform dense retrieval to identify candidate function-TTP pairs.** Represent each renamed function and each TTP definition as embeddings. Compute cosine similarity and retain the top-k function matches per TTP. This maximizes initial recall at the cost of false positives.

4. **Perform neural retrieval to refine candidates.** For each function, prompt the LLM to generate an over-inclusive set of plausible TTP candidates with confidence scores. Filter pairs below a confidence threshold. Take the union of dense and neural retrieval results as the final candidate set.

5. **For each candidate function-TTP pair, run the Context Explorer.** Starting from the seed function, selectively retrieve callers and callees whose names suggest security-relevant behavior. Prune generic utilities (memory allocation, string formatting). Terminate exploration when sufficient context has accumulated to make a determination.

6. **Apply TTP-Specific Reasoning Guidelines.** For the target TTP, load its pre-synthesized guideline containing: required behavioral components, intent signals (if intent-critical), positive/negative example contrasts, and differentiation criteria against confusable TTPs. Reason step-by-step against the checklist.

7. **Produce a binary present/absent decision with justification.** For each function-TTP pair, output whether the TTP is present, cite the specific code evidence (API calls, control flow patterns, data operations), and explain which checklist criteria were satisfied.

8. **Aggregate function-level results into a binary-level TTP report.** Deduplicate TTPs found across multiple functions, group by MITRE ATT&CK tactic, and flag any TTPs discovered beyond what existing reports document.

9. **Cross-validate confusable TTPs.** For TTPs that share surface-level similarity (e.g., file deletion for anti-forensics vs. file deletion for normal cleanup), re-examine the context and intent signals to confirm correct attribution.

10. **Output structured results.** Produce a mapping of TTP IDs to functions with evidence, organized by ATT&CK tactic categories (Initial Access, Execution, Persistence, etc.).

## Concrete Examples

**Example 1: Analyzing a single decompiled function for TTP attribution**

User: "What MITRE ATT&CK techniques does this decompiled function implement?"
```c
void sub_401230() {
    HANDLE hSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    PROCESSENTRY32 pe;
    pe.dwSize = sizeof(pe);
    Process32First(hSnap, &pe);
    do {
        if (strcmp(pe.szExeFile, "target.exe") == 0) {
            HANDLE hProc = OpenProcess(PROCESS_ALL_ACCESS, FALSE, pe.th32ProcessID);
            void* mem = VirtualAllocEx(hProc, NULL, 0x1000, MEM_COMMIT, PAGE_EXECUTE_READWRITE);
            WriteProcessMemory(hProc, mem, shellcode, sizeof(shellcode), NULL);
            CreateRemoteThread(hProc, NULL, 0, (LPTHREAD_START_ROUTINE)mem, NULL, 0, NULL);
        }
    } while (Process32Next(hSnap, &pe));
    CloseHandle(hSnap);
}
```

Approach:
1. Rename function: `inject_shellcode_into_target_process`
2. Identify API call sequence: `CreateToolhelp32Snapshot` -> `Process32First/Next` -> `OpenProcess` -> `VirtualAllocEx` -> `WriteProcessMemory` -> `CreateRemoteThread`
3. Load TTP guidelines for candidate techniques:
   - T1055.001 (Process Injection: DLL Injection) -- behavior-focused
   - T1055.012 (Process Hollowing) -- behavior-focused
   - T1057 (Process Discovery) -- intent-critical
4. Apply reasoning checklists against code evidence

Output:
```
TTP Attribution Results:
- T1055.001 - Process Injection: Dynamic-link Library Injection [PRESENT]
  Evidence: Classic remote thread injection pattern -- VirtualAllocEx allocates
  RWX memory in remote process, WriteProcessMemory copies payload,
  CreateRemoteThread executes it. All required behavioral components satisfied.
  Functions: inject_shellcode_into_target_process

- T1057 - Process Discovery [PRESENT]
  Evidence: CreateToolhelp32Snapshot + Process32First/Next enumerates running
  processes and compares against target name. Intent is adversarial: discovery
  is used to locate injection target, not for legitimate system management.
  Functions: inject_shellcode_into_target_process

- T1055.012 - Process Hollowing [ABSENT]
  Reasoning: No evidence of suspending a process, unmapping its image, or
  replacing it. The pattern here is remote thread injection, not hollowing.
```

**Example 2: Triaging a large binary with many functions**

User: "I have 200 decompiled functions from a stripped malware binary. Which ones are security-relevant and what TTPs do they map to?"

Approach:
1. Rename all 200 functions semantically using bottom-up call graph traversal
2. Dense retrieval: embed all functions and ATT&CK technique definitions, retrieve top-10 function matches per TTP
3. Neural retrieval: for each function, generate candidate TTPs with confidence scores, filter below 0.3 threshold
4. Union of both retrieval channels yields ~30 candidate function-TTP pairs (86% reduction)
5. For each candidate pair, run Context Explorer to gather caller/callee context
6. Apply TTP-Specific Reasoning Guidelines for each pair
7. Aggregate into binary-level report

Output:
```
Binary TTP Report - Sample: malware_x86_stripped.bin
=====================================================

Tactic: Defense Evasion
  T1070.004 - File Deletion          | cleanup_logs (0x40A100)
  T1562.001 - Disable Security Tools | kill_av_processes (0x40B230)
  T1027     - Obfuscated Files       | decode_config_xor (0x40C010)

Tactic: Discovery
  T1082 - System Information Discovery | get_system_info (0x40D100)
  T1057 - Process Discovery            | enumerate_procs (0x40D400)

Tactic: Collection
  T1005 - Data from Local System | scan_and_read_docs (0x40E200)

Tactic: Exfiltration
  T1041 - Exfiltration Over C2  | send_data_to_c2 (0x40F300)

Functions analyzed: 200 | Candidates after retrieval: 28
TTPs confirmed: 7 | False positives filtered: 21
```

**Example 3: Distinguishing confusable TTPs**

User: "This function deletes log files. Is it T1070 (Indicator Removal) or T1107 or just normal cleanup?"

```c
void sub_402000() {
    remove("/var/log/auth.log");
    remove("/var/log/syslog");
    remove("/tmp/.bash_history");
}
```

Approach:
1. Rename: `delete_system_logs`
2. Context Explorer: retrieve callers -- called after `exfiltrate_data` and `establish_persistence`
3. Load guidelines for T1070.004 (Indicator Removal: File Deletion)
   - Behavior-focused TTP: deleting security-relevant logs is sufficient
   - Positive signal: targets auth.log, syslog, bash_history (forensic artifacts)
   - Negative contrast: normal cleanup targets temp files, caches, not system logs
4. Check T1562 (Impair Defenses) guideline -- requires disabling a security tool, not just deleting logs

Output:
```
T1070.004 - Indicator Removal: File Deletion [PRESENT]
  Evidence: Targets forensic artifacts (auth.log, syslog, bash_history).
  Called in post-exploitation phase after data exfiltration and persistence.
  This is anti-forensic log clearing, not routine cleanup.

T1562 - Impair Defenses [ABSENT]
  Reasoning: No security tool is disabled or modified. Log deletion removes
  evidence but does not impair defensive capabilities. Guideline requires
  active interference with security software operation.
```

## Best Practices

- **Do:** Classify each TTP as behavior-focused or intent-critical before reasoning. For behavior-focused TTPs (e.g., T1055 Process Injection), the presence of characteristic API sequences is sufficient. For intent-critical TTPs (e.g., T1082 System Information Discovery), you must find evidence of adversarial purpose -- legitimate software also queries system info.
- **Do:** Use caller/callee context to resolve ambiguity. A function that reads files could be benign or T1005 (Data from Local System) depending on what calls it and what happens to the data afterward.
- **Do:** Build positive/negative contrasts for confusable TTPs. When T1070 and T1562 both involve file operations, articulate the specific differentiating criteria before making a decision.
- **Do:** Process functions bottom-up in the call graph so that callee summaries are available when analyzing higher-level functions.
- **Avoid:** Attributing TTPs based solely on individual API calls. `CreateFile` alone does not indicate any specific TTP -- the full behavioral pattern and context matter.
- **Avoid:** Treating all network operations as exfiltration (T1041). Dense retrieval often conflates DDoS, C2 communication, and data exfiltration because they share network API surfaces. Apply the reasoning guideline to distinguish them.
- **Avoid:** Exhaustive call graph traversal. Only expand context along paths where function names or API patterns suggest security-relevant behavior. Pruning generic utilities prevents context pollution.

## Error Handling

- **Decompilation artifacts:** Compiler-generated functions (prologue/epilogue stubs, exception handlers, CRT initialization) will vastly outnumber real malware functions. Filter these during renaming by recognizing standard library patterns and compiler idioms before retrieval.
- **Obfuscated or packed code:** If the decompiled output is heavily obfuscated (opaque predicates, control flow flattening), note that TTP attribution confidence drops significantly. Flag these functions as requiring dynamic analysis or manual review rather than forcing an attribution.
- **Missing context:** When the Context Explorer cannot find sufficient callers/callees (e.g., the function is only called via indirect jumps), state the limitation explicitly and provide a conditional attribution: "If this function is called in a post-exploitation context, it would constitute T1070.004."
- **Overlapping TTPs:** Some code legitimately implements multiple TTPs simultaneously (e.g., process injection followed by credential dumping). Report all applicable TTPs rather than forcing a single attribution.
- **Stripped imports:** If the binary uses dynamic API resolution (GetProcAddress with hashed names), attempt to resolve the hashes against known API hash databases before TTP analysis. Note unresolved calls as gaps.

## Limitations

- This approach works on decompiled C/C++ code. Managed-language malware (.NET, Java) requires different decompilation pipelines and the reasoning guidelines may not directly transfer.
- The method assumes access to reasonably complete decompilation output. Heavily packed or VM-protected binaries that resist static decompilation will yield poor results -- dynamic analysis should be used first.
- TTP-Specific Reasoning Guidelines are synthesized against MITRE ATT&CK definitions. Novel techniques not yet cataloged in ATT&CK cannot be identified by this method.
- Function-level attribution cannot capture TTPs that emerge only from whole-binary behavioral sequences spanning dozens of functions with complex state dependencies.
- The 87.37% precision on real-world binaries means roughly 1 in 8 attributions may be incorrect. Always treat results as analytical hypotheses requiring human validation in high-stakes contexts.

## Reference

Xuan, Z., Xu, X., Zheng, M., Tan, L.Z., & Guo, J. (2026). *Identifying Adversary Tactics and Techniques in Malware Binaries with an LLM Agent*. arXiv:2602.06325. Key sections: Section 3 (three-stage pipeline architecture), Section 3.3 (TTP-Specific Reasoning Guidelines with behavior-focused vs. intent-critical classification), Section 4 (dataset construction with 42 distinct TTPs across 12 malware families).