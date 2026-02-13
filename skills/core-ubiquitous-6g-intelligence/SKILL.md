---
name: "core-ubiquitous-6g-intelligence"
description: "Design and implement multi-LLM agent orchestration systems over hierarchical compute tiers using the CORE framework pattern: role affinity scheduling, DAG-based pipeline-parallel execution, and dynamic role reassignment. Triggers: 'orchestrate agents across edge and cloud', 'distribute LLM tasks across tiers', 'build a hierarchical agent pipeline', 'role-based agent scheduling', 'multi-agent edge orchestration', 'pipeline-parallel LLM execution'"
---

# CORE: Collaborative Orchestration of LLM Agents Over Hierarchical Edge

This skill enables Claude to design and build multi-agent LLM orchestration systems that distribute specialized agent roles across a hierarchy of compute tiers (device, edge, cloud). Based on the CORE framework, it applies role affinity scheduling, DAG-based pipeline-parallel execution, and real-time perception routing to match computational demands with heterogeneous resources -- turning fragmented infrastructure into a coherent reasoning system.

## When to Use

- When the user wants to orchestrate multiple LLM agents across different compute tiers (local devices, edge servers, cloud GPUs)
- When building a system where different agents need distinct functional roles (perception, reasoning, coordination) distributed by capability
- When designing a pipeline that decomposes complex tasks into subtasks and executes them in parallel across heterogeneous hardware
- When implementing dynamic load-aware scheduling that reassigns agent roles based on bandwidth, latency, or device utilization
- When the user asks to build an edge-AI orchestration layer for IoT, autonomous systems, or smart city applications
- When designing a multi-agent system that needs to handle real-time multimodal inputs (video, sensor data, text) with latency constraints

## Key Technique

CORE departs from monolithic single-agent architectures by distributing specialized LLM roles across a three-tier hierarchy: **device layer** (lightweight, latency-sensitive local tasks like feature extraction), **edge layer** (intermediate compute for semantic analysis and real-time reasoning), and **cloud layer** (heavy cross-domain model training and global optimization). Each agent is assigned a functional role -- perception orchestrator, dynamic reasoning engine, or network intelligence coordinator -- matched to its tier's capabilities.

The framework's core innovation is **role affinity scheduling**: an algorithm that scores agent-task compatibility based on three factors: (1) task properties (complexity, data volume, real-time constraints), (2) network conditions (bandwidth, latency, reliability), and (3) device capabilities (processing power, memory, energy). When an agent becomes overloaded or latency spikes, the scheduler dynamically reassigns roles to maintain throughput. In benchmarks, this approach (DynaRole-HEFT) reduced high-load latency by 52% compared to static scheduling.

Tasks are modeled as **directed acyclic graphs (DAGs)** where nodes are subtasks and edges encode dependencies. This enables pipeline-parallel execution: independent subtasks run concurrently across agents while dependent subtasks wait for upstream results. Inter-agent coordination uses a shared context protocol (inspired by MCP) so agents in different tiers maintain unified workflow awareness despite operating on different data modalities.

## Step-by-Step Workflow

1. **Decompose the task into a DAG of subtasks.** Analyze the user's goal and break it into discrete operations with explicit dependencies. Each node represents an atomic unit of work (e.g., "extract objects from video frame", "classify anomaly severity", "generate response plan"). Draw edges where one subtask's output feeds another's input.

2. **Define the compute tier hierarchy.** Map available infrastructure into tiers. At minimum, define two tiers (edge and cloud); three tiers (device, edge, cloud) for systems with local devices. Document each tier's constraints: latency budget, memory, GPU availability, network bandwidth to adjacent tiers.

3. **Assign functional roles to agents.** Create specialized agent profiles with distinct system prompts and tool access. Typical roles:
   - **Perception Agent** (device/edge): processes raw multimodal inputs, extracts structured features
   - **Reasoning Agent** (edge/cloud): performs complex analysis, planning, decision-making
   - **Coordinator Agent** (cloud): orchestrates workflow, aggregates results, handles cross-domain logic
   - **Execution Agent** (device/edge): carries out actions, returns status

4. **Implement the role affinity scoring function.** For each (agent, subtask) pair, compute an affinity score:
   ```python
   def role_affinity(agent, subtask):
       capability_match = score_capability(agent.tier, subtask.compute_requirement)
       latency_fit = score_latency(agent.network_latency, subtask.deadline)
       load_factor = 1.0 - agent.current_utilization
       return (0.4 * capability_match + 0.3 * latency_fit + 0.3 * load_factor)
   ```
   Assign each subtask to the agent with the highest affinity score.

5. **Build the pipeline-parallel execution engine.** Traverse the DAG topologically. At each level, dispatch all independent subtasks concurrently to their assigned agents. Use an event-driven loop: when an agent completes a subtask, check if downstream subtasks are unblocked and dispatch them immediately.

6. **Implement inter-agent context passing.** Define a shared context object (JSON or protobuf) that flows along DAG edges. Each agent reads upstream context, appends its results, and passes the enriched context downstream. This maintains unified workflow awareness without requiring agents to share full conversation history.

7. **Add dynamic reassignment monitoring.** Run a lightweight health-check loop that polls agent latency and utilization every N seconds. When an agent exceeds a latency threshold or utilization cap, trigger reassignment: recompute affinity scores for its pending subtasks and migrate them to a better-scoring agent.

8. **Implement fallback and retry logic.** If an agent fails or times out, mark its subtask as retriable. Reassign to the next-best agent by affinity score. Cap retries at 2 attempts before escalating the subtask to a higher compute tier.

9. **Aggregate results and return.** Once all DAG leaf nodes complete, the coordinator agent merges outputs into a coherent response. Apply any post-processing (formatting, validation, deduplication) before returning to the user.

10. **Log metrics for tuning.** Record per-subtask latency, agent utilization, reassignment events, and end-to-end completion time. Use these to iteratively adjust affinity weights and tier boundaries.

## Concrete Examples

**Example 1: Emergency incident analysis pipeline**

User: "Build a system that analyzes traffic camera feeds to detect vehicle fires and coordinate emergency response across edge and cloud servers."

Approach:
1. Decompose into DAG: `detect_anomaly (device)` -> `classify_severity (edge)` -> `gather_context (edge, parallel: weather + traffic data)` -> `generate_response_plan (cloud)` -> `dispatch_alerts (edge)`
2. Define tiers: roadside cameras (device, 2GB RAM, 50ms latency budget), regional edge server (NVIDIA 4090, 200ms budget), cloud (A100 cluster, 2s budget)
3. Assign roles:
   - Perception Agent on device: runs lightweight object detection (MiniCPM-scale model)
   - Reasoning Agent on edge: classifies fire severity from extracted features
   - Context Agent on edge: parallel fetches weather API + traffic density
   - Coordinator on cloud: synthesizes all inputs into emergency response plan
4. Implement affinity scoring, dispatch DAG, return structured alert

Output:
```json
{
  "pipeline": "vehicle_fire_response",
  "stages": [
    {"id": "detect", "agent": "perception-cam-07", "tier": "device", "latency_ms": 45},
    {"id": "classify", "agent": "reasoning-edge-west", "tier": "edge", "latency_ms": 180},
    {"id": "weather", "agent": "context-edge-west", "tier": "edge", "latency_ms": 90},
    {"id": "traffic", "agent": "context-edge-west", "tier": "edge", "latency_ms": 110},
    {"id": "plan", "agent": "coordinator-cloud", "tier": "cloud", "latency_ms": 850},
    {"id": "alert", "agent": "exec-edge-west", "tier": "edge", "latency_ms": 30}
  ],
  "total_latency_ms": 1105,
  "response_plan": "Deploy fire unit from Station 12 (ETA 4 min). Divert lanes 2-3 via Route 9. Notify hospital trauma center."
}
```

**Example 2: Multi-agent coding assistant with tiered compute**

User: "Design an agent system where a lightweight local model handles code completion, an edge model does code review, and a cloud model handles architecture decisions."

Approach:
1. DAG for a feature request: `parse_request (local)` -> `generate_code (local)` + `check_architecture (cloud, parallel)` -> `review_code (edge)` -> `apply_fixes (local)`
2. Tiers: developer laptop (local, 8GB model), team edge server (70B model on 4x A40), cloud API (frontier model)
3. Role assignment:
   - Local Agent: fast code completion, syntax-level tasks (affinity: low-latency, small context)
   - Edge Agent: code review, security scanning, test generation (affinity: mid-compute, code-aware)
   - Cloud Agent: architecture decisions, cross-repo analysis (affinity: high-compute, large context)
4. Affinity scoring ensures architecture queries never hit the local model, and completions never wait for cloud round-trips

Output:
```python
# Orchestrator configuration
TIERS = {
    "local":  {"latency_budget_ms": 100,  "max_context_tokens": 8192},
    "edge":   {"latency_budget_ms": 500,  "max_context_tokens": 32768},
    "cloud":  {"latency_budget_ms": 5000, "max_context_tokens": 128000},
}

ROLES = {
    "completer":   {"tier": "local", "prompt": "Fast code completion agent..."},
    "reviewer":    {"tier": "edge",  "prompt": "Thorough code review agent..."},
    "architect":   {"tier": "cloud", "prompt": "System architecture advisor..."},
}

dag = DAG()
dag.add("parse", role="completer", deps=[])
dag.add("generate", role="completer", deps=["parse"])
dag.add("arch_check", role="architect", deps=["parse"])  # parallel with generate
dag.add("review", role="reviewer", deps=["generate", "arch_check"])
dag.add("apply", role="completer", deps=["review"])
```

**Example 3: IoT sensor fusion with dynamic reassignment**

User: "I have 50 IoT sensors feeding data to 3 edge nodes. Sometimes one edge node gets overloaded. Build an orchestration layer that dynamically rebalances."

Approach:
1. Each sensor streams data to its nearest edge node (affinity: network latency)
2. Edge agents run anomaly detection models; results flow to a cloud coordinator
3. Monitor loop checks edge utilization every 5 seconds
4. When utilization > 80%, recompute affinity scores for that node's pending tasks and migrate overflow to the least-loaded edge node
5. Implement graceful handoff: buffer 2 seconds of data during migration to prevent drops

Output:
```python
class DynamicRebalancer:
    def __init__(self, edge_nodes, utilization_threshold=0.8):
        self.edges = edge_nodes
        self.threshold = utilization_threshold

    def check_and_rebalance(self):
        for node in self.edges:
            if node.utilization > self.threshold:
                overflow_tasks = node.pending_tasks_sorted_by_priority()
                for task in overflow_tasks:
                    best_alt = max(
                        (n for n in self.edges if n != node),
                        key=lambda n: role_affinity(n, task)
                    )
                    if best_alt.utilization < self.threshold:
                        node.migrate_task(task, best_alt, buffer_sec=2)
                        log_reassignment(task, node, best_alt)
```

## Best Practices

- **Do:** Assign the smallest capable model to each role. Perception tasks rarely need a 70B model; an 8B vision model on edge is faster and cheaper.
- **Do:** Make affinity weights tunable. Start with equal weights (0.33 each) and adjust based on logged metrics from real workloads.
- **Do:** Design DAGs with maximum parallelism. If two subtasks don't depend on each other, they should run concurrently on separate agents.
- **Do:** Include a health-check heartbeat. Agents that stop responding within 2x their expected latency should be marked degraded immediately.
- **Avoid:** Sending raw high-bandwidth data (video frames, audio streams) between tiers. Always extract features locally and pass structured summaries upstream.
- **Avoid:** Static role assignment in production. The key insight of CORE is that roles should shift dynamically; hard-coded assignments defeat the purpose.
- **Avoid:** Overloading the coordinator agent with direct computation. Its job is orchestration and aggregation only; heavy reasoning belongs on edge/cloud reasoning agents.

## Error Handling

| Failure Mode | Detection | Response |
|---|---|---|
| Agent timeout | No response within 2x latency budget | Retry on same agent once, then reassign to next-best by affinity |
| Network partition between tiers | Heartbeat loss for >10s | Promote edge agent to temporary coordinator; queue cloud-bound tasks |
| Overloaded edge node | Utilization >80% sustained | Trigger dynamic rebalancing; migrate lowest-priority tasks first |
| DAG deadlock | Cycle detected in dependency graph | Reject task at submission time; DAGs must be validated acyclic before execution |
| Context object too large | Serialized context exceeds tier's token limit | Summarize upstream context before passing to constrained agents |
| Model hallucination in critical path | Validator agent detects inconsistency | Flag result, retry with higher-tier model, log for human review |

## Limitations

- **Not for single-machine setups.** If all your compute is on one server, standard single-agent or simple tool-use patterns are simpler and faster. CORE's value emerges with genuinely distributed, heterogeneous infrastructure.
- **Network overhead is real.** Inter-tier communication adds latency. For tasks completable in <200ms on a single capable model, the orchestration overhead outweighs the benefit.
- **Role affinity tuning requires production data.** The initial weight configuration is a guess; meaningful optimization requires logging and iterating on real workload patterns.
- **DAG design is manual.** The framework does not auto-decompose tasks into subtask graphs. You must design the DAG topology for each workflow type.
- **Security boundaries.** Passing context between tiers (especially device-to-cloud) requires careful handling of sensitive data. The framework does not include encryption or access control by default.

## Reference

**Paper:** [CORE: Toward Ubiquitous 6G Intelligence Through Collaborative Orchestration of Large Language Model Agents Over Hierarchical Edge](https://arxiv.org/abs/2601.21822v1) (Yu et al., 2026, IEEE Communications Magazine)

**What to look for:** Section III for the three-module architecture and role affinity scheduling algorithm, Section IV for the DAG-based pipeline-parallel execution strategy, and Section V for case studies (highway fire incident) and the 52% latency reduction benchmark with DynaRole-HEFT vs. static HEFT scheduling.