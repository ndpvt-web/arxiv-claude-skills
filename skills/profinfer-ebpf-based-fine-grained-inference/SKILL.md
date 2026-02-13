---
name: "profinfer-ebpf-based-fine-grained-inference"
description: "Profile and diagnose LLM inference engines (llama.cpp and similar GGML-based runtimes) using eBPF uprobes for non-intrusive, operator-level performance analysis. Trigger phrases: 'profile llama.cpp inference', 'eBPF LLM profiling', 'diagnose inference bottleneck', 'trace GGML operators', 'is my inference memory-bound or compute-bound', 'profile MoE routing overhead'"
---

# ProfInfer: eBPF-Based Fine-Grained LLM Inference Profiling

This skill enables Claude to help users build and deploy non-intrusive, fine-grained profiling instrumentation for LLM inference engines using eBPF (extended Berkeley Packet Filter). Based on the ProfInfer framework (MLSys 2026), the technique dynamically attaches uprobes to runtime functions across application, runtime, kernel, and hardware layers -- without modifying or recompiling the inference engine source code. It produces operator-level execution traces, computation graphs, timelines, and hardware counter trends that reveal whether workloads are memory-bound or compute-bound, where time is spent across GGML operators (matmul, softmax, RoPE, attention), and how MoE routing and CPU/GPU offloading behave in practice.

## When to Use

- When the user wants to profile llama.cpp or another GGML-based inference engine to find performance bottlenecks
- When the user asks whether their LLM inference workload is memory-bound or compute-bound
- When the user needs to trace individual operator execution times (matmul, softmax, RoPE, KV-cache operations) during inference
- When the user wants to analyze Mixture-of-Experts (MoE) routing behavior -- expert selection skew, per-expert latency, load imbalance
- When the user needs to profile CPU-to-GPU operator offloading decisions and data transfer overhead
- When the user wants to collect hardware performance counters (cache misses, IPC, branch mispredictions) correlated with specific inference operators
- When the user asks to build a profiling dashboard or visualization pipeline for LLM serving infrastructure
- When the user needs production-safe profiling with less than 4% overhead that does not require recompilation

## Key Technique

**Multi-layer eBPF instrumentation without source modification.** ProfInfer uses uprobes (user-space probes) to attach to exported symbols in the llama.cpp shared library at runtime. Unlike traditional profilers that require compile-time instrumentation (e.g., `-pg`, `-finstrument-functions`) or source-level annotations, uprobes hook into already-compiled binaries by patching a breakpoint instruction at the target function's entry and return addresses. This means you can profile any build of llama.cpp -- including release builds -- by resolving symbol names from the ELF binary with `nm` or `readelf` and attaching BCC/bpftrace probes to those symbols.

**Four-layer trace collection.** The framework instruments four distinct layers simultaneously: (1) the application layer captures high-level inference calls like `llama_decode` and prompt processing; (2) the runtime layer traces GGML graph execution via `ggml_graph_compute` and individual operator functions like `ggml_compute_forward_mul_mat`, `ggml_compute_forward_soft_max`, and `ggml_compute_forward_rope`; (3) the kernel layer captures scheduling events, context switches, and system calls via kprobes and tracepoints; (4) the hardware layer collects perf counters (LLC misses, IPC, memory stall cycles) via `perf_event_open` eBPF programs. By correlating events across all four layers with nanosecond timestamps, the framework produces a unified timeline showing exactly which operator was running when a cache miss spike occurred.

**Bound classification from counters.** A workload is classified as memory-bound when the LLC miss ratio is high and IPC is low (typically < 1.0 on modern CPUs), and compute-bound when IPC is high with low cache miss rates. ProfInfer automates this classification per-operator, so you can see that matmul is compute-bound while KV-cache lookups are memory-bound -- within the same inference pass.

## Step-by-Step Workflow

1. **Identify the target binary and resolve symbols.** Run `nm -D /path/to/libllama.so | grep ggml_` (or `readelf -Ws`) to enumerate all instrumentable GGML operator functions. Key targets include `ggml_graph_compute`, `ggml_compute_forward_mul_mat`, `ggml_compute_forward_soft_max`, `ggml_compute_forward_rope`, `ggml_compute_forward_rms_norm`, and backend dispatch functions like `ggml_backend_cpu_graph_compute`.

2. **Write BCC uprobe scripts for operator-level tracing.** Create a Python BCC script that attaches uprobes and uretprobes to each target symbol. On entry, record `bpf_ktime_get_ns()` into a BPF hash map keyed by `(pid, tid, function_id)`. On return, compute the delta and emit it to a perf buffer or BPF ring buffer with the operator name, thread ID, and duration in nanoseconds.

3. **Add hardware counter collection with perf_event eBPF programs.** Open `perf_event_open` file descriptors for `PERF_COUNT_HW_CACHE_MISSES`, `PERF_COUNT_HW_INSTRUCTIONS`, `PERF_COUNT_HW_CPU_CYCLES`, and `PERF_COUNT_HW_BRANCH_MISSES`. Attach BPF programs to these events that sample at a configurable frequency (e.g., every 10,000 events) and store the counter values alongside the current uprobe context so counters can be attributed to specific operators.

4. **Add kernel-layer context probes.** Attach tracepoints to `sched:sched_switch` and `sched:sched_wakeup` to detect when inference threads are preempted. This reveals whether high operator latency is caused by the operator itself or by OS scheduling interference.

5. **Correlate events using a unified timestamp.** All eBPF programs use `bpf_ktime_get_ns()` (monotonic clock). In the user-space collector, merge operator entry/exit events, hardware counter samples, and scheduling events into a single sorted timeline keyed by `(timestamp, thread_id, event_type)`.

6. **Generate operator execution graphs.** Parse the GGML computation graph structure (accessible via `ggml_graph_compute`'s argument -- a pointer to `struct ggml_cgraph`) to build a DAG of operator dependencies. Overlay measured execution times onto each node to identify the critical path.

7. **Build timeline visualizations.** Export the merged event stream as a Chrome Trace Format JSON (`catapult` / `chrome://tracing`), with one track per thread, operator spans as duration events, and hardware counter samples as instant events. Alternatively, generate a Perfetto protobuf trace for richer analysis.

8. **Classify operators as memory-bound or compute-bound.** For each operator span, compute: `IPC = instructions / cycles` and `LLC_miss_rate = llc_misses / instructions`. Flag operators with `IPC < 1.0` and `LLC_miss_rate > 0.01` as memory-bound. Flag operators with `IPC > 2.0` and low miss rate as compute-bound. Output a summary table.

9. **Profile MoE routing (if applicable).** Attach uprobes to the expert-selection and expert-dispatch functions in MoE model inference. Track which experts are selected per token, measure per-expert execution time, and compute load imbalance metrics (e.g., coefficient of variation across expert execution times).

10. **Validate overhead.** Run the inference workload with and without probes attached, measuring end-to-end tokens-per-second. Confirm that overhead stays below 4% by limiting the number of simultaneously active uprobes and using BPF ring buffers instead of perf buffers for high-frequency events.

## Concrete Examples

**Example 1: Profiling llama.cpp operator latency breakdown**

User: "I'm running llama.cpp with a 7B model and generation feels slow. Can you help me profile which operators are taking the most time?"

Approach:
1. Resolve symbols from the llama.cpp binary
2. Write a BCC uprobe script targeting key GGML operators
3. Run inference with probes attached and collect traces
4. Produce a latency breakdown

Output:
```python
#!/usr/bin/env python3
# profinfer_operator_trace.py -- Trace GGML operator latencies via uprobes
from bcc import BPF
import ctypes, time, sys

LLAMA_BIN = sys.argv[1] if len(sys.argv) > 1 else "/usr/local/bin/llama-server"

bpf_text = r"""
#include <uapi/linux/ptrace.h>

struct event_t {
    u64 ts;
    u64 duration_ns;
    u32 pid;
    u32 tid;
    char func[64];
};

BPF_HASH(start_ts, u64, u64);
BPF_PERF_OUTPUT(events);

static inline void trace_entry(struct pt_regs *ctx) {
    u64 key = bpf_get_current_pid_tgid();
    u64 ts = bpf_ktime_get_ns();
    start_ts.update(&key, &ts);
}

static inline void trace_return(struct pt_regs *ctx, const char *name, int len) {
    u64 key = bpf_get_current_pid_tgid();
    u64 *tsp = start_ts.lookup(&key);
    if (!tsp) return;
    struct event_t evt = {};
    evt.ts = *tsp;
    evt.duration_ns = bpf_ktime_get_ns() - *tsp;
    evt.pid = key >> 32;
    evt.tid = key & 0xFFFFFFFF;
    __builtin_memcpy(evt.func, name, len);
    events.perf_submit(ctx, &evt, sizeof(evt));
    start_ts.delete(&key);
}
"""

# Operators to trace -- extend as needed
OPERATORS = [
    "ggml_compute_forward_mul_mat",
    "ggml_compute_forward_soft_max",
    "ggml_compute_forward_rope",
    "ggml_compute_forward_rms_norm",
    "ggml_compute_forward_add",
    "ggml_compute_forward_silu",
    "ggml_graph_compute",
]

# Generate per-operator entry/return BPF functions
for i, op in enumerate(OPERATORS):
    bpf_text += f"""
int entry_{i}(struct pt_regs *ctx) {{ trace_entry(ctx); return 0; }}
int return_{i}(struct pt_regs *ctx) {{ trace_return(ctx, "{op}", {len(op)}); return 0; }}
"""

b = BPF(text=bpf_text)
for i, op in enumerate(OPERATORS):
    b.attach_uprobe(name=LLAMA_BIN, sym=op, fn_name=f"entry_{i}")
    b.attach_uretprobe(name=LLAMA_BIN, sym=op, fn_name=f"return_{i}")

print("Tracing GGML operators... Ctrl+C to stop.")
latencies = {}  # op -> list of durations

def handle_event(cpu, data, size):
    evt = b["events"].event(data)
    name = evt.func.decode().rstrip('\x00')
    latencies.setdefault(name, []).append(evt.duration_ns)

b["events"].open_perf_buffer(handle_event)
try:
    while True:
        b.perf_buffer_poll()
except KeyboardInterrupt:
    pass

print(f"\n{'Operator':<45} {'Calls':>8} {'Total ms':>10} {'Avg us':>10} {'% Time':>8}")
print("-" * 85)
total_ns = sum(sum(v) for v in latencies.values())
for op, durs in sorted(latencies.items(), key=lambda x: -sum(x[1])):
    total_ms = sum(durs) / 1e6
    avg_us = (sum(durs) / len(durs)) / 1e3
    pct = (sum(durs) / total_ns * 100) if total_ns else 0
    print(f"{op:<45} {len(durs):>8} {total_ms:>10.1f} {avg_us:>10.1f} {pct:>7.1f}%")
```

Expected output table:
```
Operator                                        Calls   Total ms     Avg us   % Time
-------------------------------------------------------------------------------------
ggml_compute_forward_mul_mat                     1920     4823.1     2512.0    72.3%
ggml_compute_forward_soft_max                     640      521.4      814.7     7.8%
ggml_compute_forward_rope                         640      418.2      653.4     6.3%
ggml_compute_forward_rms_norm                     640      312.5      488.3     4.7%
ggml_compute_forward_add                         1280      298.1      232.9     4.5%
ggml_compute_forward_silu                         640      201.3      314.5     3.0%
ggml_graph_compute                                 10       98.7     9870.0     1.5%
```

**Example 2: Classifying operators as memory-bound vs compute-bound**

User: "How do I determine if my matmul operations are memory-bound or compute-bound during inference?"

Approach:
1. Attach uprobes for operator entry/exit
2. Simultaneously collect hardware perf counters (cycles, instructions, LLC misses)
3. Compute IPC and cache miss rate per operator span
4. Output classification

Output:
```bash
#!/usr/bin/env bpftrace
# profinfer_bound_classify.bt -- Classify operator bound type
# Requires: bpftrace >= 0.17, CAP_BPF

uprobe:/usr/local/bin/llama-server:ggml_compute_forward_mul_mat {
    @start[tid] = nsecs;
    @hw_cycles_start[tid] = cpid;  // placeholder -- real impl uses perf_event
}

uretprobe:/usr/local/bin/llama-server:ggml_compute_forward_mul_mat {
    $dur = nsecs - @start[tid];
    printf("mul_mat tid=%d duration=%llu ns\n", tid, $dur);
    // In production, read perf counters here and compute IPC
}
```

In a full BCC implementation, the classification logic:
```python
for op, metrics in operator_hw_counters.items():
    ipc = metrics["instructions"] / metrics["cycles"] if metrics["cycles"] else 0
    miss_rate = metrics["llc_misses"] / metrics["instructions"] if metrics["instructions"] else 0

    if ipc < 1.0 and miss_rate > 0.01:
        bound = "MEMORY-BOUND"
    elif ipc > 2.0 and miss_rate < 0.005:
        bound = "COMPUTE-BOUND"
    else:
        bound = "MIXED"

    print(f"{op:<45} IPC={ipc:.2f}  LLC_miss_rate={miss_rate:.4f}  => {bound}")
```

Expected output:
```
ggml_compute_forward_mul_mat                  IPC=3.41  LLC_miss_rate=0.0002  => COMPUTE-BOUND
ggml_compute_forward_soft_max                 IPC=0.87  LLC_miss_rate=0.0180  => MEMORY-BOUND
ggml_compute_forward_rope                     IPC=1.24  LLC_miss_rate=0.0090  => MIXED
ggml_compute_forward_rms_norm                 IPC=0.62  LLC_miss_rate=0.0230  => MEMORY-BOUND
```

**Example 3: Generating a Chrome Trace timeline**

User: "I want to visualize the inference timeline so I can see operator overlap across threads."

Approach:
1. Collect uprobe traces with timestamps and thread IDs
2. Convert to Chrome Trace Event Format JSON
3. Open in `chrome://tracing` or Perfetto UI

Output:
```python
import json

def export_chrome_trace(events, output_path="trace.json"):
    """Convert collected operator events to Chrome Trace Format."""
    trace_events = []
    for evt in events:
        # Duration event (ph="X") with begin timestamp and duration
        trace_events.append({
            "name": evt["operator"],
            "cat": "ggml",
            "ph": "X",
            "ts": evt["start_ns"] / 1000,  # Chrome trace uses microseconds
            "dur": evt["duration_ns"] / 1000,
            "pid": evt["pid"],
            "tid": evt["tid"],
            "args": {
                "bound_type": evt.get("bound_type", "unknown"),
                "ipc": evt.get("ipc", 0),
            }
        })
    with open(output_path, "w") as f:
        json.dump({"traceEvents": trace_events}, f)
    print(f"Trace written to {output_path} -- open in chrome://tracing or ui.perfetto.dev")
```

## Best Practices

- **Do:** Resolve symbols from the actual deployed binary using `nm -D` or `readelf -Ws` before writing probes. Symbol names vary between llama.cpp versions and build configurations (static vs shared, C++ name mangling with `extern "C"` wrappers).
- **Do:** Use BPF ring buffers (`BPF_RINGBUF_OUTPUT`) instead of perf buffers for high-frequency operator tracing -- ring buffers have lower overhead and avoid per-CPU buffer allocation.
- **Do:** Filter by PID in the BPF program (`if (pid != TARGET_PID) return 0;`) to avoid tracing unrelated processes and reduce overhead.
- **Do:** Start with coarse-grained probes (`ggml_graph_compute`) first, then drill down to individual operators only after identifying which graph executions are slow.
- **Avoid:** Attaching uprobes to extremely hot inner-loop functions (called millions of times per inference) -- this can push overhead beyond the 4% threshold. Target the operator dispatch level, not the innermost kernel.
- **Avoid:** Running eBPF profiling without `CAP_BPF` and `CAP_PERFMON` capabilities (or root). The probes will silently fail to attach. Check `dmesg` if no events appear.

## Error Handling

| Problem | Symptom | Solution |
|---------|---------|----------|
| Symbol not found | `attach_uprobe` raises exception | Run `nm -D` on the binary; symbol may be mangled or stripped. Use `--keep-symbols` at build time or find the mangled name with `c++filt`. |
| No events received | Script runs but prints nothing | Verify the target process PID is correct and inference is actually running. Check `bpftool prog list` to confirm probes loaded. |
| Permission denied | BPF program fails to load | Run with `sudo` or grant `CAP_BPF` + `CAP_PERFMON` via `setcap`. Check `/proc/sys/kernel/unprivileged_bpf_disabled`. |
| High overhead (>4%) | Inference slows noticeably | Reduce the number of simultaneous uprobes. Avoid tracing per-element functions. Use sampling instead of tracing for hardware counters. |
| Kernel version too old | BPF verifier rejects program | eBPF features used here require Linux >= 5.8 for ring buffers, >= 4.17 for basic uprobes. Check `uname -r`. |
| Stale probes after binary update | Events stop or crash | Detach and re-attach probes after updating the llama.cpp binary, as function offsets change. |

## Limitations

- **Linux-only.** eBPF uprobes are a Linux kernel feature. This approach does not work on macOS or Windows. For non-Linux platforms, consider `dtrace` (macOS) or `ETW` (Windows) as alternatives, though they lack eBPF's programmability.
- **Requires symbol visibility.** Fully stripped binaries (`strip --strip-all`) remove the symbols needed for uprobe attachment. The target binary must retain at least dynamic symbols (`.dynsym`) or a separate debug info file.
- **No GPU kernel profiling.** eBPF operates in kernel/user space on the CPU. It can trace the CPU-side dispatch of GPU operations and measure data transfer overhead, but cannot profile GPU kernel execution itself. Pair with `nvprof`/`nsys` or `rocprof` for GPU-side visibility.
- **GGML-architecture specific.** The operator-level tracing targets GGML's function naming conventions (`ggml_compute_forward_*`). Other inference engines (vLLM/PyTorch, TensorRT-LLM) have different internal structures and require different symbol targets, though the eBPF methodology transfers.
- **Overhead scales with probe count.** While individual probes add < 4% overhead, attaching hundreds of probes simultaneously to hot paths can accumulate. Profile the profiler itself when adding new probe points.
- **No support for JIT-compiled operators.** If the inference engine JIT-compiles kernels (e.g., via XLA or TVM), function addresses are not stable in the ELF binary. Uprobes cannot attach to dynamically generated code without additional bookkeeping.

## Reference

**Paper:** Zou, B., Roy, D., Airao, D.Y., Xu, W., Sun, B. "ProfInfer: An eBPF-based Fine-Grained LLM Inference Profiler." MLSys 2026. [arXiv:2601.20755v2](https://arxiv.org/abs/2601.20755v2)

Look for: The four-layer instrumentation architecture (Section 3), the operator-level uprobe attachment methodology (Section 3.2), hardware counter correlation with operator spans (Section 3.4), and the overhead evaluation showing < 4% impact on inference throughput (Section 5).