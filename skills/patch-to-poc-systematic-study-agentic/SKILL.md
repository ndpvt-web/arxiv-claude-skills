---
name: "patch-to-poc-systematic-study-agentic"
description: >
  Agentic kernel vulnerability reproduction from security patches. Implements the K-Repro methodology:
  controlled code browsing, hypothesis-driven root cause analysis, iterative PoC generation,
  VM-based testing, and GDB debugging for Linux kernel N-day reproduction.
  Trigger phrases: "reproduce this kernel vulnerability", "generate a PoC from this patch",
  "analyze this security patch for exploitability", "kernel bug reproduction from commit",
  "patch-to-poc analysis", "N-day vulnerability reproduction"
---

# Patch-to-PoC: Agentic Kernel Vulnerability Reproduction from Security Patches

This skill enables Claude to systematically reproduce Linux kernel vulnerabilities given only a security patch commit. It applies the K-Repro methodology — a hypothesis-driven, iterative pipeline combining controlled source code browsing, static root cause analysis, PoC generation in C or via subsystem utilities, QEMU-based VM testing with KASAN, and GDB-based kernel debugging. The approach achieves 56% reproduction on real-world KernelCTF vulnerabilities and outperforms directed fuzzing by 50x in wall-clock time.

## When to Use

- When a user provides a kernel security patch (commit hash, diff, or patch file) and asks to understand or reproduce the vulnerability it fixes
- When analyzing whether a specific kernel commit addresses a UAF, OOB, race condition, or other memory safety bug
- When building a proof-of-concept exploit or crash trigger for a known kernel N-day
- When setting up a QEMU environment to test kernel vulnerabilities with KASAN
- When debugging a kernel PoC that compiles but does not trigger the expected crash
- When assessing the exploitability or severity of a patched kernel bug from offensive or defensive perspectives
- When designing an agentic pipeline for automated vulnerability reproduction in systems code

## Key Technique

K-Repro separates the reproduction pipeline into a **deterministic environment stage** and an **LLM-driven analysis stage**. The environment stage is non-agentic: it extracts the patch, checks out the vulnerable kernel source (one commit before the fix), compiles with GCC 10 using a syzbot-derived config with KASAN enabled, and prepares a QEMU VM with snapshot-based startup for deterministic baselines. This separation ensures the agent never wastes reasoning tokens on reproducible infrastructure.

The analysis stage is where the LLM agent operates. It receives only the patch commit identifier and uses four tool categories: **code browsing** (list symbols, query definitions/references globally, retrieve targeted snippets), **VM management** (start/restart VM, compile and upload binaries), **VM interaction** (send commands, read output, issue control signals), and **GDB debugging** (set breakpoints, inspect registers/memory, execute raw GDB commands). The agent is instructed to perform static analysis before dynamic validation — forming hypotheses about the vulnerability class, affected objects, and triggering conditions from the patch diff, then validating through iterative PoC execution.

The critical insight is that **hypothesis-driven exploration massively outperforms brute-force approaches**. Rather than fuzzing inputs, the agent reads the patch to identify what changed, reasons about what code path reaches the vulnerable state, and constructs a targeted trigger. Successful reproductions average 4.5-4.9 iterations with a median time under 12 minutes. Race conditions (29% success) and temporal memory bugs (UAF/DF) are significantly harder than spatial bugs (OOB) — knowing this upfront lets the agent allocate effort appropriately.

## Step-by-Step Workflow

1. **Extract and parse the security patch.** Given a commit hash or diff, identify all modified files, functions, and specific changed lines. Categorize the change: is it adding a bounds check (OOB), a lock/synchronization primitive (race), a reference count adjustment (UAF), or a null check? The patch type strongly predicts reproduction difficulty.

2. **Analyze the commit message for reproduction hints.** Classify the message into three tiers: (a) explicit reproduction steps or triggering syscalls mentioned (75% expected success), (b) basic issue acknowledgment naming the bug class (56%), or (c) minimal/no description (46%). Adjust confidence and strategy accordingly.

3. **Perform controlled code browsing around the patch site.** Use targeted queries — not full file reads — to examine the patched function, its callers, and the data structures involved. List symbols in large files first, then retrieve only the relevant snippets. Average ~40 queries per case; stay hypothesis-driven rather than exhaustive.

4. **Form a root cause hypothesis.** Based on the patch diff and surrounding code, state explicitly: (a) the vulnerability class (UAF, OOB, race, DF, type confusion), (b) the specific object or buffer affected, (c) the syscall or subsystem entry point that reaches the vulnerable code, and (d) whether a race window or specific timing is required.

5. **Choose a PoC strategy.** If the vulnerability is in a subsystem with user-space utilities (netfilter/nft, tc, BPF), prefer a command-line PoC using those tools. Otherwise, write a C program using raw syscalls and library APIs. Both approaches achieve comparable success rates (~46-56%).

6. **Set up the vulnerable kernel environment.** Compile the kernel at the commit immediately before the patch with KASAN enabled. Boot in QEMU with snapshot-based startup. Verify the VM reaches a usable state before uploading any PoC.

7. **Compile, upload, and execute the PoC inside the VM.** Use a single-step compile-and-upload tool to minimize round trips. Observe kernel console output for KASAN reports, panics, or BUG_ON assertions. Any kernel-level crash constitutes successful reproduction.

8. **Iterate based on execution feedback.** If no crash occurs, analyze the output to determine why. Common refinements: adjust struct field offsets, add timing delays for race windows, change syscall argument values, or target a different code path. Successful cases average 4-5 iterations; budget up to 10.

9. **Use GDB for targeted state inspection when stuck.** Set breakpoints at the vulnerable function or allocation sites to confirm the PoC reaches the intended code path. Inspect register values and memory contents at the breakpoint. Do not attempt instruction-level stepping — use breakpoints for state validation only.

10. **Document the reproduction.** Record the vulnerability class, affected subsystem, PoC code, kernel crash log (KASAN trace or panic output), and the reasoning chain from patch to trigger. Note any race-condition timing dependencies or kernel config requirements.

## Concrete Examples

**Example 1: Out-of-bounds write in netfilter (non-race, spatial bug)**

User: "Here's a kernel patch that adds a bounds check in `nf_tables_newset`. Can you reproduce the vulnerability it fixes?"

Approach:
1. Parse the patch: a missing length validation on set element data allows writing beyond an allocated buffer in `nft_set_elem_init`.
2. Commit message says "fix OOB write in nft set element" — tier (b), basic acknowledgment.
3. Browse `nft_set_elem_init`, its caller `nf_tables_newsetelem`, and the `nft_set` struct to understand buffer allocation size vs. user-controlled length.
4. Hypothesis: user can send a netlink message creating a set element with data length exceeding the set's declared `dlen`, causing a heap OOB write.
5. PoC strategy: use `nft` command-line utility to create a set, then add an oversized element.
6. Build vulnerable kernel, boot QEMU with KASAN.
7. Execute nft commands inside VM. KASAN fires with "BUG: KASAN: slab-out-of-bounds in nft_set_elem_init".
8. First attempt succeeds — spatial bugs in netfilter are among the highest-success category.

Output:
```
[KASAN Report]
BUG: KASAN: slab-out-of-bounds in nft_set_elem_init+0x1a3/0x2b0
Write of size 64 at addr ffff88800xxxxxxx by task nft/1234
Call Trace:
 nft_set_elem_init+0x1a3/0x2b0
 nf_tables_newsetelem+0x5c2/0xb80
 ...
PoC: nft add table ip test; nft add set ip test s { type ipv4_addr\; size 1\; }
     nft add element ip test s { 1.2.3.4 . <oversized_data> }
```

**Example 2: Use-after-free with race condition in BPF (temporal + race)**

User: "Analyze commit abc123 which fixes a UAF in `bpf_map_free`. Generate a PoC if possible."

Approach:
1. Parse the patch: adds a synchronize_rcu() call before freeing a BPF map, indicating a classic RCU-based UAF race.
2. Commit message: minimal — "fix map free race" — tier (c).
3. Browse `bpf_map_free`, `bpf_map_put`, and the map's reference counting logic. Identify the window between the last `bpf_map_put` and the actual free where another thread can still access the map via an RCU-protected pointer.
4. Hypothesis: two threads — one closing the map fd (triggering free), one performing a map lookup — can race, causing the lookup to dereference freed memory.
5. PoC strategy: C program using `bpf()` syscalls. Thread A creates and pins a map, Thread B loops performing lookups. Thread A unpins and closes the fd. Timing loop to hit the race window.
6. Build kernel, boot QEMU. First 3 attempts: no crash (race window is narrow).
7. Iteration 4: add CPU affinity to pin threads to same core, insert `usleep` calibration. Still no crash.
8. Iteration 6: use GDB to set breakpoint at `bpf_map_free`, confirm Thread A reaches it. Confirm Thread B is in `bpf_map_lookup_elem` concurrently. Adjust timing.
9. Iteration 8: KASAN fires intermittently. Race conditions have ~29% reproduction rate; this is expected difficulty.

Output:
```
[Intermittent KASAN Report - triggered on ~1 in 5 runs]
BUG: KASAN: use-after-free in bpf_map_lookup_elem+0x42/0x190
Read of size 8 at addr ffff88800xxxxxxx by task poc_thread/5678
Freed by task poc_closer/5679:
 bpf_map_free+0x38/0x60
 ...
Note: Race-condition PoC. Requires multiple runs to trigger reliably.
```

**Example 3: Designing an agentic reproduction pipeline (architecture task)**

User: "I want to build an automated system that takes kernel patches and generates PoCs. How should I architect this?"

Approach:
1. Separate into deterministic infrastructure (no LLM) and agentic analysis (LLM-driven).
2. Infrastructure layer: git checkout, kernel build with KASAN + syzbot config, QEMU snapshot management.
3. Agent tooling layer: implement four tool categories as MCP server endpoints:
   - Code browsing: symbol listing, global reference queries, snippet retrieval (avoid full-file reads)
   - VM management: start/stop/restart, compile-and-upload as single action
   - VM interaction: command send/receive with multiplexed kernel messages
   - GDB debugging: breakpoint management, register/memory inspection
4. Agent prompting: instruct static-before-dynamic analysis, hypothesis formation, iterative refinement.
5. Success criteria: any KASAN report or kernel panic in VM logs.
6. Budget: 10 hours max per case, expect ~12 min median for successes.

Output:
```
Architecture:
  [Patch Commit] → [Deterministic Setup]  → [LLM Agent Loop]  → [PoC + Report]
                    ├─ git checkout          ├─ Code Browse
                    ├─ kernel build          ├─ Hypothesize
                    ├─ QEMU snapshot         ├─ Generate PoC
                    └─ tool server start     ├─ Compile + Test
                                             ├─ Observe output
                                             ├─ GDB inspect
                                             └─ Iterate (avg 4-5x)
```

## Best Practices

- **Do:** Perform static analysis of the patch and surrounding code before any dynamic execution. Hypothesis-driven exploration is 50x faster than blind fuzzing.
- **Do:** Use controlled, targeted code browsing (symbol queries, snippet retrieval) rather than reading entire files. Kernel files are enormous and will exhaust context.
- **Do:** Enable KASAN in the kernel build — it converts silent memory corruption into immediate, detectable crashes, dramatically increasing reproduction signal.
- **Do:** Use snapshot-based VM startup for deterministic baselines between iterations. Non-determinism in environment setup wastes debugging effort.
- **Avoid:** Attempting instruction-level GDB stepping through kernel code. Use breakpoints for state validation only — stepping is too slow and token-expensive.
- **Avoid:** Spending excessive iterations on race-condition bugs without adjusting strategy. If 5+ iterations fail, try CPU pinning, timing calibration, or increasing thread counts before abandoning.
- **Avoid:** Full memory searches or brute-force input generation inside the agent loop. These are expensive operations better suited to dedicated fuzzers.

## Error Handling

| Problem | Diagnosis | Resolution |
|---------|-----------|------------|
| Kernel fails to boot | Config mismatch or build error | Verify syzbot config compatibility with target commit; check GCC version (use GCC 10) |
| PoC compiles but no crash | Wrong code path or insufficient trigger | Set GDB breakpoint at patched function to confirm reachability; review syscall arguments |
| KASAN not reporting | KASAN not enabled or wrong slab allocator | Verify `CONFIG_KASAN=y` in `.config`; ensure the vulnerable object uses SLAB/SLUB |
| Race condition never triggers | Timing window too narrow | Pin threads to same CPU core; calibrate `usleep` delays; increase iteration count to 20+ |
| Agent fixates on wrong hypothesis | Misread patch signals | Re-read the patch diff focusing on what was *added* (not removed); check if multiple functions were modified |
| Context window exhausted | Too much code browsed | Switch to targeted symbol queries; summarize findings before continuing; avoid reading full files |

## Limitations

- **Race conditions are fundamentally hard**: 29% success rate vs. 64% for non-race bugs. Reproducing timing-dependent vulnerabilities requires non-deterministic retries that an LLM agent cannot fully control.
- **Temporal memory bugs (UAF, double-free) are harder than spatial (OOB)**: The agent must reason about object lifetimes and allocation/free ordering, which requires deeper cross-function analysis.
- **Kernel subsystem expertise varies**: While no single subsystem shows statistically significant degradation, complex subsystem-specific prerequisites (e.g., specific socket states, elaborate netlink message sequences) can block reproduction.
- **No internet access by design**: The agent cannot look up public PoCs, CVE details, or syzkaller reproducers. It must derive everything from the patch and kernel source.
- **10-hour budget ceiling**: Some vulnerabilities require environment conditions (specific hardware, kernel modules, multi-step protocol interactions) that exceed practical time budgets.
- **LLM knowledge cutoff is not a significant factor** (52% post-cutoff vs. 59% pre-cutoff, not statistically significant) — the agent relies on code reading rather than memorized vulnerability patterns.

## Reference

**Paper:** [Patch-to-PoC: A Systematic Study of Agentic LLM Systems for Linux Kernel N-Day Reproduction](https://arxiv.org/abs/2602.07287v1) — Pu et al., 2026. Look for: Section 3 (K-Repro architecture and tool design), Section 4 (evaluation on 100 KernelCTF cases), Section 5 (factor analysis: race conditions, CWE types, commit message quality, prompt impact), and Table 2 (per-iteration success rates showing the value of iterative refinement).