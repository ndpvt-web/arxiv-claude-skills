---
name: "outrunning-cutoffs-live-kernel"
description: "Build live-evolving kernel crash resolution benchmarks and agent environments using the Live-kBench/kEnv methodology. Set up compile-boot-reproduce feedback loops for automated kernel bug fixing. Triggers: 'kernel crash resolution', 'syzbot bug fix', 'live kernel benchmark', 'kEnv crash environment', 'Linux kernel patch agent', 'kernel fuzzer bug triage'"
---

# Live Kernel Crash Resolution with Feedback-Driven Agents

This skill enables Claude to build and operate automated kernel crash resolution pipelines inspired by the Live-kBench framework. The core technique pairs a continuously-updated bug corpus (scraped from syzbot/Syzkaller) with a standardized compile-boot-reproduce environment (kEnv) that gives agents structured feedback on every patch attempt. By exposing compilation errors, boot failures, and reproducer outcomes back to the agent, crash resolution rates improve by ~29% compared to single-shot patching. The methodology applies broadly to any systems-level crash triage that follows a compile-run-validate cycle.

## When to Use

- When the user wants to build an automated pipeline for triaging and patching Linux kernel crashes from syzbot
- When setting up a QEMU-based compile-boot-reproduce loop for validating kernel patches
- When designing a benchmark that continuously scrapes new bugs and evaluates agent performance over time
- When the user needs to resolve a specific kernel crash given a stack trace, reproducer, and source checkout
- When building an agent framework that iteratively refines patches using compilation and runtime feedback
- When evaluating LLM-generated kernel patches for both crash resolution and semantic equivalence to developer fixes

## Key Technique

**Decoupled Agent-Environment Architecture.** The central insight from Live-kBench is separating the *agent* (which reads crash context and proposes patches) from the *execution environment* (which compiles the kernel, boots it in QEMU, and runs the reproducer). This decoupling via kEnv means any agent framework — whether SWE-agent, OpenHands, or a custom script — gets identical compilation toolchains, kernel configs, and reproducers. The agent submits a git diff; kEnv returns one of three outcomes: compilation error, crash reproduced (with a new stack trace), or crash resolved.

**Feedback-Driven Iteration.** Single-shot patch generation achieves roughly 74% plausible patch rate but only ~20% equivalence to developer fixes. The breakthrough is the Crash Resolution Feedback (CRF) loop: after each failed attempt, the agent receives the exact compiler error or the new crashing stack trace, then refines its patch. This iterative loop — typically 5-10 rounds — boosts resolution rates by 29%. The feedback is structured: compilation errors include line numbers and messages; boot failures include kernel panic logs; reproducer failures include the crash signature.

**Live Benchmark Evolution.** Static benchmarks suffer from data contamination when bugs fall within an LLM's training cutoff. Live-kBench addresses this with a weekly scraper that pulls fresh syzbot bugs, verifies reproducibility (5 attempts), and adds them to the benchmark. This keeps the evaluation honest — agents show up to 25% lower equivalent patch rates on post-cutoff bugs, revealing genuine generalization gaps.

## Step-by-Step Workflow

1. **Acquire the crash context.** Collect the syzbot crash report (including full stack trace), the C reproducer program, the kernel commit hash where the crash occurs, and the `.config` file used during fuzzing. Store these as structured metadata (bug ID, subsystem, crash type, commit SHA).

2. **Check out the kernel at the correct commit.** Clone the Linux kernel repository and `git checkout` to the exact commit referenced in the bug report. This ensures the agent patches against the same code state where the crash was discovered.

3. **Set up the kEnv execution environment.** Build a Docker image containing: the checked-out kernel source, the compiled reproducer binary, a QEMU disk image with a minimal root filesystem, and the CRF feedback tool. Install the kernel build toolchain (GCC/Clang, make, binutils, flex, bison, libelf-dev).

4. **Synthesize the agent prompt.** Construct input for the agent containing: the crash report with stack trace, the reproducer C source code, instructions for using the `run_kernel` feedback tool, and the path to the Linux source tree. Do not include the developer's fix — the agent must derive a patch independently.

5. **Generate a patch as a git diff.** The agent reads the crash context, localizes the bug (using stack trace functions and file paths), and produces a patch in unified diff format. Target patches averaging 10-20 lines of code, matching the typical developer fix size.

6. **Submit the patch to the compile-boot-reproduce pipeline.** Apply the diff to the kernel source, run `make -j$(nproc)` with the bug's `.config`, boot the resulting kernel in QEMU with timeout protection (typically 5-10 minutes), and execute the reproducer binary 25 times to account for nondeterministic crashes.

7. **Parse and return structured feedback.** If compilation fails, extract error messages with file paths and line numbers. If the kernel panics during boot, capture the panic log. If the reproducer still triggers the crash, return the new stack trace. If all 25 reproducer runs pass, report "crash resolved."

8. **Iterate on failures.** Feed the structured feedback back to the agent as a new prompt turn. The agent analyzes the error, adjusts its patch, and resubmits. Allow 5-10 iterations maximum. Track per-iteration metrics: time, token cost, and outcome.

9. **Evaluate patch quality on two axes.** Measure *Crash Resolution Rate (CRR)*: does the patch prevent the crash across 25 reproducer runs? Measure *Equivalent Patch Rate (EPR)*: does the patch structurally and logically match the developer's ground-truth fix? Use an LLM judge with majority voting (3+ evaluations) for equivalence assessment. Also compute *Localization IoU* at file and function granularity.

10. **Log results and update the dashboard.** Record the bug ID, agent identity, number of iterations, CRR, EPR, localization accuracy, wall-clock time, and token cost. Aggregate across bug attributes (subsystem, type, pre/post-cutoff) for trend analysis.

## Concrete Examples

**Example 1: Resolving a use-after-free in the networking subsystem**

User: "I have a syzbot crash report for a use-after-free in net/core/sock.c at commit abc1234. The reproducer is a C program that triggers it via socket syscalls. Help me set up an automated resolution pipeline."

Approach:
1. Parse the crash report to extract the faulting function (`sock_release`) and stack frames
2. Check out the kernel at commit `abc1234` and apply the syzbot `.config`
3. Build the Docker environment with QEMU and the compiled reproducer
4. Prompt the agent with crash context: "KASAN: use-after-free in sock_release+0x42/0x80 at net/core/sock.c:3215"
5. Agent proposes a patch adding a NULL check after `sk_free()` — submit to compile-boot-reproduce
6. Compilation succeeds, but reproducer still crashes with a different trace — feed back the new trace
7. Agent revises: moves the socket reference count decrement before the free call
8. Second submission: all 25 reproducer runs pass — crash resolved in 2 iterations

Output:
```
Bug: syzbot-net-uaf-abc1234
Status: RESOLVED (iteration 2/10)
CRR: 100% (25/25 runs clean)
Patch: net/core/sock.c | +3 -1 lines
Feedback loop: compile_ok -> crash_reproduced -> compile_ok -> crash_resolved
```

**Example 2: Building a weekly live benchmark pipeline**

User: "Set up a pipeline that scrapes new syzbot bugs weekly and benchmarks our patch agent against them."

Approach:
1. Write a scraper that hits the syzbot dashboard (`syzkaller.appspot.com`) weekly via cron
2. For each new bug, extract: crash title, report, reproducer, kernel commit, config
3. Filter for reproducibility — attempt reproduction 5 times, keep only consistently reproducible bugs
4. Generate SuiteCache (pre-compiled kernel state at each commit) for incremental builds
5. For each reproducible bug, spin up a kEnv container and invoke the agent
6. Collect CRR and EPR metrics, partition by pre/post LLM knowledge cutoff date
7. Push results to a dashboard with filters for subsystem, bug type, and agent version

Output:
```
Weekly Run: 2026-02-10
New bugs scraped: 12
Reproducible: 8 (67%)
Agent results:
  CRR (first attempt): 6/8 (75%)
  CRR (with feedback, 5 rounds): 7/8 (88%)
  EPR: 2/8 (25%)
Post-cutoff bugs: 5/8
  CRR on post-cutoff: 3/5 (60%)
  CRR on pre-cutoff: 4/3 — wait, 3/3 (100%)
```

**Example 3: Diagnosing why an agent patch fails compilation**

User: "My agent generated a patch for a NULL pointer dereference in fs/ext4/inode.c but it won't compile. Here's the error."

Approach:
1. Parse the compilation error: `fs/ext4/inode.c:1847:15: error: 'bh' undeclared`
2. Read the agent's diff — it references a variable `bh` that was renamed to `buffer_head` in this commit
3. Advise the agent (or user) to re-read the local variable declarations at the crash site
4. Generate a corrected patch using the actual variable name from the checked-out source
5. Resubmit through the compile-boot-reproduce loop

Output:
```
Diagnosis: Variable name mismatch — agent used 'bh' but source at this
commit uses 'buffer_head' (renamed in commit def5678).
Fix: Replace 'bh' with 'buffer_head' in the patch.
Resubmission: compile_ok -> boot_ok -> crash_resolved
```

## Best Practices

- **Do:** Always check out the kernel at the exact commit from the bug report. Even one commit difference can change variable names, function signatures, or file structure.
- **Do:** Run the reproducer at least 25 times per validation. Kernel crashes are nondeterministic — a single clean run means nothing.
- **Do:** Return structured feedback (compiler errors with line numbers, full panic traces) rather than binary pass/fail. The 29% improvement comes from rich feedback.
- **Do:** Partition benchmark results by LLM knowledge cutoff date. Pre-cutoff bugs inflate metrics due to potential data contamination.
- **Avoid:** Treating crash resolution as equivalent to a correct fix. A patch that disables the crashing code path resolves the crash but introduces regressions. Always measure EPR alongside CRR.
- **Avoid:** Running more than 10 feedback iterations. Diminishing returns set in quickly — if 5 rounds of feedback haven't resolved it, the agent likely lacks the context to fix the bug.
- **Avoid:** Skipping the SuiteCache (pre-compiled kernel state). Full kernel compilation averages ~20 minutes per attempt; incremental builds from cached state cut this dramatically.

## Error Handling

| Failure Mode | Cause | Resolution |
|---|---|---|
| Compilation timeout | Large kernel, slow machine | Use `ccache` and limit `make -j` to available cores; pre-build SuiteCache |
| QEMU boot hang | Kernel deadlock from patch | Set a hard boot timeout (300s); return "boot_hang" feedback to agent |
| Reproducer flaky | Nondeterministic crash | Run 25 attempts; require >80% resolution rate to mark as resolved |
| Diff doesn't apply | Agent targets wrong file/line | Verify `git checkout` commit matches bug report; provide `git apply` error to agent |
| Agent loops on same error | Insufficient context | Inject additional context: nearby function definitions, related header files, git blame for the crash site |
| Docker OOM | Kernel build exhausts memory | Allocate 8GB+ RAM to container; reduce parallel compilation jobs |

## Limitations

- **Hardware intensive.** Each compile-boot-reproduce cycle requires ~20 minutes and significant CPU/RAM. Running benchmarks at scale requires cloud infrastructure or dedicated build servers.
- **Reproducer dependency.** The pipeline requires a working C reproducer from syzbot. Bugs without reproducers (~30% of syzbot reports) cannot be evaluated.
- **Equivalence assessment is approximate.** EPR relies on LLM-based judging with majority voting, which introduces subjectivity. Patches that are functionally correct but structurally different from the developer fix may be unfairly penalized.
- **Linux-specific.** The kEnv environment, Syzkaller reproducers, and syzbot scraping are specific to the Linux kernel. Adapting this to other OS kernels (FreeBSD, Windows) requires significant retooling.
- **Nondeterminism.** Some kernel bugs are timing-sensitive and may not reproduce consistently even with 25 attempts, leading to false positives in crash resolution metrics.

## Reference

**Paper:** "Outrunning LLM Cutoffs: A Live Kernel Crash Resolution Benchmark for All" — Huang et al., 2026. [arXiv:2602.02690](https://arxiv.org/abs/2602.02690v1)

Key sections: Section 3 (kEnv architecture and CRF tool design), Section 4 (Live-kBench scraping pipeline and reproducibility filtering), Section 5 (empirical results showing the 29% feedback improvement and pre/post-cutoff performance gap).