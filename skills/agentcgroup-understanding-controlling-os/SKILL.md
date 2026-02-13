---
name: "agentcgroup-understanding-controlling-os"
description: "Design and implement OS-level resource controls for sandboxed AI agents using hierarchical cgroups, eBPF enforcement, and tool-call-level resource management. Use when: 'set up cgroups for AI agent containers', 'control memory for coding agents', 'isolate tool-call resources with eBPF', 'manage multi-tenant agent resource limits', 'prevent OOM kills in agent sandboxes', 'configure agent resource policies with cgroup v2'."
---

# AgentCgroup: OS-Level Resource Control for AI Agent Workloads

This skill enables Claude to design, configure, and implement Linux cgroup v2 hierarchies and eBPF-based enforcement policies specifically tuned for AI agent workloads. Unlike traditional container resource limits that operate at a single level, this approach creates per-tool-call sub-cgroups, uses in-kernel eBPF hooks for sub-second enforcement, and applies graduated responses (throttle before kill) to handle the extreme memory spikes and unpredictable resource demands characteristic of AI coding agents.

## When to Use

- When the user needs to configure cgroup v2 resource limits for containers running AI coding agents (e.g., SWE-bench, Devin-style, or Claude Code sandboxes)
- When deploying multi-tenant AI agent infrastructure and needs isolation between concurrent agent sessions
- When debugging OOM kills or memory pressure issues in sandboxed agent environments
- When the user asks to write eBPF programs for resource monitoring or enforcement in agent containers
- When designing resource policies that need to handle 15x memory spikes from tool calls (shell commands, test suites, builds)
- When the user wants to profile or characterize resource usage patterns of AI agent tool calls
- When setting up Kubernetes pod resource limits for agent workloads and finding that static limits cause either waste or OOM kills

## Key Technique

AI coding agents have a fundamentally different resource profile than web services, batch jobs, or serverless functions. Research on 144 SWE-bench tasks shows that OS-level execution (tool calls, container init, agent init) accounts for 56-74% of end-to-end latency, and memory -- not CPU -- is the primary concurrency bottleneck. Memory spikes are driven by individual tool calls (running `pytest`, `git diff`, compilation) and exhibit a peak-to-average ratio of up to 15.4x, with bursts lasting only 1-2 seconds. Critically, these patterns are non-deterministic: the same task on the same model produces 1.8x variance across runs.

This creates three mismatches with existing resource controls. First, a **granularity mismatch**: container-level limits (Kubernetes QoS, Docker `--memory`) apply one budget to the entire agent session, but demands fluctuate per tool call -- a static limit that accommodates peaks wastes 93% of allocated memory on average. Second, a **responsiveness mismatch**: user-space controllers (kubelet, custom autoscalers) react on millisecond-to-minute timescales via polling, but agent bursts are sub-second and unpredictable, so the controller misses the window entirely. Third, an **adaptability mismatch**: history-based prediction fails because agent workloads are stateful and non-deterministic, and an OOM kill destroys minutes of accumulated LLM context that cannot be cheaply re-created.

AgentCgroup addresses these through three mechanisms: (1) a **hierarchical cgroup v2 structure** where each agent maps to a parent cgroup and each tool call spawns a child sub-cgroup, isolating the stable framework baseline (~185 MB) from tool-driven bursts; (2) **in-kernel eBPF enforcement** via `sched_ext` for CPU scheduling priority and `memcg_bpf_ops` hooks (e.g., `get_high_delay_ms`) for graduated memory throttling instead of termination; and (3) **runtime-adaptive policies** where eBPF programs trace process creation and memory allocation in-kernel to detect tool-call boundaries automatically, applying priority-based throttling and freezing without user-space round-trips.

## Step-by-Step Workflow

1. **Audit the agent runtime structure.** Identify the agent framework process (Python/Node), its child processes per tool call (bash, pytest, git, compilers), and the container runtime (Docker, gVisor, Firecracker). Map the process tree to understand which PIDs belong to the framework baseline vs. tool-call bursts.

2. **Create a hierarchical cgroup v2 layout.** Under the container's top-level cgroup, create a parent cgroup per agent session and child sub-cgroups per tool-call type. Enable the `memory` and `cpu` controllers at each level:
   ```bash
   # Enable controllers on parent
   echo "+memory +cpu +pids" > /sys/fs/cgroup/agent-session-01/cgroup.subtree_control
   # Create tool-call sub-cgroups
   mkdir /sys/fs/cgroup/agent-session-01/tool-call-current
   ```

3. **Set graduated memory limits on the parent cgroup.** Configure `memory.high` (soft throttle threshold) well below `memory.max` (hard OOM boundary) to create a throttling zone that buys time before termination:
   ```bash
   echo 800M > /sys/fs/cgroup/agent-session-01/memory.high
   echo 1200M > /sys/fs/cgroup/agent-session-01/memory.max
   ```

4. **Assign tool-call processes to sub-cgroups.** When the agent framework spawns a tool call, move the child PID into the tool-call sub-cgroup. Implement this in the agent wrapper script or via an eBPF program that traces `fork`/`exec` and auto-assigns based on process ancestry:
   ```bash
   echo $TOOL_PID > /sys/fs/cgroup/agent-session-01/tool-call-current/cgroup.procs
   ```

5. **Set per-tool-call memory limits on sub-cgroups.** Apply tighter limits to the tool-call sub-cgroup so that a runaway `pytest` or build doesn't consume the entire agent budget:
   ```bash
   echo 600M > /sys/fs/cgroup/agent-session-01/tool-call-current/memory.high
   echo 900M > /sys/fs/cgroup/agent-session-01/tool-call-current/memory.max
   ```

6. **Write eBPF programs for in-kernel enforcement (advanced).** Use `memcg_bpf_ops` hooks to customize throttle delays when `memory.high` is breached, and `sched_ext` to prioritize latency-sensitive tool calls. Develop with libbpf and BPF CO-RE for portability:
   ```c
   SEC("memcg/get_high_delay_ms")
   int BPF_PROG(throttle_delay, struct mem_cgroup *memcg, unsigned int *delay_ms) {
       u32 priority = get_cgroup_priority(memcg);
       if (priority == PRIORITY_HIGH)
           *delay_ms = 10;   // Brief throttle for high-priority agents
       else
           *delay_ms = 200;  // Aggressive throttle for low-priority
       return 0;
   }
   ```

7. **Implement tool-call boundary detection.** Attach eBPF tracepoints to `sched_process_fork` and `sched_process_exec` to detect when the agent framework launches a new tool call. Use BPF maps to track the mapping between PIDs and tool-call sub-cgroups.

8. **Configure priority-based graduated responses.** Define a policy hierarchy: first throttle via `memory.high` delays, then freeze low-priority sub-cgroups via `cgroup.freeze`, and only OOM-kill as a last resort. Store priority mappings in BPF maps shared between the kernel and a lightweight user-space daemon.

9. **Deploy a lightweight user-space daemon for lifecycle management.** Write a small daemon that manages cgroup creation/teardown for each tool call, updates BPF map entries for policy configuration, and handles cgroup cleanup when tool calls complete. This daemon does not make enforcement decisions -- it only manages structure.

10. **Monitor and tune using cgroup pressure metrics.** Read `memory.pressure`, `cpu.pressure`, and `memory.events` from each sub-cgroup to observe throttle frequency, OOM events, and pressure stall durations. Adjust `memory.high` thresholds based on observed tool-call profiles for the specific agent framework.

## Concrete Examples

**Example 1: Configuring cgroup limits for a SWE-bench agent container**

User: "I'm running AI coding agents on SWE-bench tasks in Docker containers. They keep getting OOM-killed during test execution. How do I set up resource controls?"

Approach:
1. Identify that test execution (pytest) is the tool call causing memory spikes
2. Create a two-level cgroup hierarchy: agent session parent + tool-call child
3. Set graduated limits with a throttle zone before OOM

Output:
```bash
#!/bin/bash
# setup-agent-cgroups.sh -- Run inside the container or via Docker's cgroup parent

AGENT_CGROUP="/sys/fs/cgroup/agent-session-${SESSION_ID}"
TOOLCALL_CGROUP="${AGENT_CGROUP}/tool-call"

# Create hierarchy
mkdir -p "$TOOLCALL_CGROUP"
echo "+memory +cpu +pids" > "${AGENT_CGROUP}/cgroup.subtree_control"

# Agent-level limits: soft throttle at 800M, hard kill at 1200M
echo 838860800  > "${AGENT_CGROUP}/memory.high"    # 800 MB
echo 1258291200 > "${AGENT_CGROUP}/memory.max"     # 1200 MB

# Tool-call limits: tighter bounds for individual commands
echo 629145600  > "${TOOLCALL_CGROUP}/memory.high"  # 600 MB
echo 943718400  > "${TOOLCALL_CGROUP}/memory.max"   # 900 MB

# Agent wrapper: move tool-call PIDs into sub-cgroup
run_tool_call() {
    local cmd="$1"
    sh -c "echo \$\$ > ${TOOLCALL_CGROUP}/cgroup.procs && exec $cmd"
}

# Example: run pytest in the tool-call sub-cgroup
run_tool_call "python -m pytest tests/ -x --timeout=300"
```

**Example 2: eBPF-based auto-detection of tool-call boundaries**

User: "I want to automatically detect when my agent spawns tool calls and assign them to separate cgroups without modifying the agent framework code."

Approach:
1. Write an eBPF program that traces process creation events
2. Detect tool calls by matching the parent PID to the known agent framework PID
3. Auto-migrate child processes into the tool-call sub-cgroup

Output:
```c
// tool_call_detect.bpf.c
#include <vmlinux.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 256);
    __type(key, u32);       // PID of agent framework
    __type(value, u64);     // cgroup ID for tool-call sub-cgroup
} agent_pids SEC(".maps");

SEC("tp/sched/sched_process_fork")
int detect_tool_call(struct trace_event_raw_sched_process_fork *ctx) {
    u32 parent_pid = ctx->parent_pid;
    u32 child_pid = ctx->child_pid;

    u64 *tool_cgroup_id = bpf_map_lookup_elem(&agent_pids, &parent_pid);
    if (tool_cgroup_id) {
        // Parent is a known agent framework process.
        // Signal user-space daemon to migrate child_pid to tool-call cgroup.
        // (Use a ring buffer event to notify the daemon)
        struct tool_call_event evt = {
            .child_pid = child_pid,
            .cgroup_id = *tool_cgroup_id,
        };
        bpf_ringbuf_output(&events, &evt, sizeof(evt), 0);
    }
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

**Example 3: Multi-tenant isolation for concurrent agent sessions**

User: "I need to run 8 AI agents concurrently on a 16GB machine. How do I prevent one agent's memory spike from killing others?"

Approach:
1. Calculate per-agent memory budget: 16GB / 8 = 2GB per agent, but tool-call spikes need headroom
2. Use memory.high for soft isolation and memory.max for hard isolation
3. Prioritize agents so low-priority ones get throttled first under pressure

Output:
```bash
#!/bin/bash
# multi-tenant-setup.sh

TOTAL_MEM_MB=16384
NUM_AGENTS=8
PER_AGENT_HIGH=$((TOTAL_MEM_MB * 85 / 100 / NUM_AGENTS))  # 1740 MB soft
PER_AGENT_MAX=$((TOTAL_MEM_MB * 95 / 100 / NUM_AGENTS))    # 1945 MB hard

for i in $(seq 1 $NUM_AGENTS); do
    CGROUP="/sys/fs/cgroup/agent-${i}"
    mkdir -p "${CGROUP}/tool-call"
    echo "+memory +cpu +pids" > "${CGROUP}/cgroup.subtree_control"

    echo "${PER_AGENT_HIGH}M" > "${CGROUP}/memory.high"
    echo "${PER_AGENT_MAX}M"  > "${CGROUP}/memory.max"

    # Tool-call sub-cgroup gets 70% of agent budget
    TOOL_HIGH=$((PER_AGENT_HIGH * 70 / 100))
    echo "${TOOL_HIGH}M" > "${CGROUP}/tool-call/memory.high"
    echo "${PER_AGENT_MAX}M" > "${CGROUP}/tool-call/memory.max"

    # Monitor pressure for adaptive tuning
    echo "Monitor: cat ${CGROUP}/memory.pressure"
done

echo "Provisioned $NUM_AGENTS agents with ${PER_AGENT_HIGH}MB soft / ${PER_AGENT_MAX}MB hard limits"
```

## Best Practices

- **Do:** Set `memory.high` 30-40% below `memory.max` to create a throttling buffer zone. This gives the system time to reclaim memory before triggering OOM kills, preserving expensive LLM context.
- **Do:** Create separate sub-cgroups for each tool call and tear them down after completion. This prevents memory accounting leaks and gives you per-tool-call metrics via `memory.peak` and `memory.events`.
- **Do:** Use `cgroup.freeze` for low-priority agents under memory pressure instead of killing them. Frozen agents can resume once pressure subsides, avoiding the cost of re-executing LLM inference.
- **Do:** Monitor `memory.events.local` to track `high` (throttle triggers), `max` (hard limit hits), and `oom_kill` counts per sub-cgroup to identify which tool calls cause spikes.
- **Avoid:** Setting only `memory.max` without `memory.high`. Without the throttle threshold, the kernel jumps straight from unlimited allocation to OOM kill with no graceful degradation.
- **Avoid:** Using container-level (single flat cgroup) limits for agent workloads. The 15.4x peak-to-average memory ratio means a limit sized for peaks wastes 93% of allocated memory, while a limit sized for averages triggers constant OOM kills.
- **Avoid:** Relying on user-space monitoring daemons (polling `/proc` or cgroup files) for burst detection. Memory spikes in agent tool calls last 1-2 seconds; user-space polling loops have 10-100ms latency plus reaction time, missing the burst window.

## Error Handling

- **OOM kills despite memory.high:** If `memory.events` shows `oom_kill` counts rising, the gap between `memory.high` and `memory.max` is too small. Increase the throttle zone or lower `memory.high` to trigger throttling earlier.
- **Agent hangs after throttle:** Aggressive `memory.high` settings can stall an agent indefinitely. Set a timeout in the agent wrapper that kills and restarts the tool call if it remains throttled beyond a threshold (e.g., 30 seconds).
- **eBPF program fails to load:** `memcg_bpf_ops` requires Linux 6.15+ with specific patches (currently under upstream review as of the paper). Fall back to user-space cgroup management with `memory.high`/`memory.max` on older kernels -- this sacrifices sub-second responsiveness but retains the graduated approach.
- **Sub-cgroup cleanup failure:** If tool-call sub-cgroups are not cleaned up (processes still attached), use `cgroup.kill` to terminate stragglers before `rmdir`, or implement a reaper in the lifecycle daemon.
- **Priority inversion:** If a high-priority agent depends on output from a low-priority agent that is frozen, the system deadlocks. Track inter-agent dependencies and exempt dependency-chain agents from freezing.

## Limitations

- **Kernel version requirement:** Full eBPF enforcement with `memcg_bpf_ops` and `sched_ext` requires Linux 6.15+ with patches that are not yet upstream. The cgroup v2 hierarchy approach works on Linux 5.8+, but without in-kernel enforcement hooks.
- **No workload prediction:** The paper explicitly shows that agent resource demands are non-deterministic (1.8x variance across identical tasks). This approach is reactive, not predictive -- it cannot pre-allocate resources for an upcoming spike.
- **Container runtime compatibility:** Some container runtimes (gVisor, Firecracker) have their own resource management layers that may conflict with direct cgroup manipulation. Test compatibility with your specific runtime.
- **Not applicable to CPU-bound agents:** The paper found memory, not CPU, is the bottleneck for coding agents. If your agent workload is CPU-bound (e.g., heavy compilation, ML inference on CPU), the memory-focused hierarchy may not address your primary constraint.
- **Single-machine scope:** AgentCgroup operates at the OS level on a single host. For distributed multi-node agent deployments, you need an orchestrator-level policy (e.g., Kubernetes) in addition to per-node cgroup controls.

## Reference

**Paper:** Zheng et al., "AgentCgroup: Understanding and Controlling OS Resources of AI Agents" (2026). arXiv: [2602.09345](https://arxiv.org/abs/2602.09345v1). Key sections: Section 3 for workload characterization data (memory spike ratios, latency breakdowns), Section 4 for the three-mismatch analysis, and Section 5 for the hierarchical cgroup + eBPF architecture.