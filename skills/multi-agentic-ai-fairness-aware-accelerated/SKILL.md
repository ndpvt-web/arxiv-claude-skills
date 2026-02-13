---
name: "multi-agentic-ai-fairness-aware-accelerated"
description: "Design and implement multi-agent systems for fairness-aware, low-latency inference orchestration across distributed edge infrastructure. Use when: 'schedule LLM inference across edge nodes', 'build a fair multi-model routing system', 'deploy AI agents for edge orchestration', 'optimize latency and fairness for distributed inference', 'create a prompt routing scheduler with Jain fairness', 'orchestrate containerized LLM serving on heterogeneous hardware'."
---

# Multi-Agentic AI for Fairness-Aware Accelerated Inference on Edge Networks

This skill enables Claude to design, implement, and operate hierarchical multi-agent systems that route prompts and deploy large language models across heterogeneous edge infrastructure while jointly optimizing for latency and fairness. The core technique decomposes the orchestration problem into three cooperating LLM-powered agents -- a long-term planner, a short-term scheduler, and per-node deployment managers -- that reason over runtime telemetry and historical logs in natural language rather than requiring hand-tuned heuristics or model fine-tuning. This approach, from Li et al. (2026), achieved 80%+ latency reduction and a Normalized Jain Index of 0.90 across city-wide edge testbeds.

## When to Use

- When the user needs to route inference requests across multiple GPU/CPU edge nodes with different model capabilities and resource profiles.
- When building a scheduling system that must balance per-user or per-request fairness (not just throughput) using a quantitative fairness metric like the Jain Index.
- When designing containerized multi-model serving infrastructure (e.g., Docker/K8s) that dynamically places and scales LLMs on heterogeneous hardware.
- When the user wants LLM-powered agents (not hardcoded rules) to make orchestration decisions by reasoning over telemetry data in natural language.
- When deploying multi-modal models (text, image, audio) with different resource footprints and the system must handle mixed-modality prompt streams.
- When the user asks to build a planning/scheduling system with separate long-horizon strategy and short-horizon reactive scheduling layers.

## Key Technique

The paper's central insight is that traditional optimization-based schedulers struggle with the combinatorial complexity of multi-modal LM deployment on heterogeneous edge networks. Instead of solving a monolithic optimization problem, the framework decomposes it into three agent roles with distinct time horizons. The **long-term planning agent** ingests network topology, node specs, and historical workload patterns to produce a deployment blueprint -- which models should run on which nodes and what resource budgets to allocate. The **short-term prompt scheduling agent** operates on a per-request basis, reading live telemetry (queue depths, inference latencies, CPU/GPU utilization) and the current deployment plan to route each incoming prompt to the best available node-model pair. The **on-node deployment agents** manage local container lifecycle, resource isolation, and model loading/unloading on each edge server.

All three agent types are powered by foundation language models. They receive structured telemetry as natural-language-formatted context and produce decisions as structured JSON outputs with natural-language justifications. This means the system adapts to new hardware, new models, or shifting traffic patterns without retraining -- the agents simply reason over updated context. Communication between agents uses structured message passing (JSON task descriptions, resource status updates) with event-driven triggers for anomalies like queue buildup or deadline violations.

The fairness objective is formalized via the **Normalized Jain Index**: `J = (sum(x_i))^2 / (n * sum(x_i^2))` where `x_i` is the normalized latency for request `i` and `n` is the request count. A value of 1.0 means perfectly equal treatment; the framework targets J >= 0.85 by penalizing routing decisions that consistently starve certain request types or users. The scheduler maintains a sliding window of per-user latency history and injects this fairness signal into its prompt context before each routing decision.

## Step-by-Step Workflow

1. **Model the infrastructure as a typed graph.** Define each edge node with its hardware spec (CPU cores, RAM, GPU type/VRAM, network bandwidth) and each deployable model with its resource requirements (min VRAM, expected tokens/sec, supported modalities). Store this as a JSON topology file.

2. **Implement the long-term planning agent.** Build a system prompt that accepts the infrastructure graph, historical workload distributions (requests/minute per modality), and SLA targets. The agent outputs a deployment plan: a mapping of model-to-node assignments with resource budgets. Re-run this agent on a slow cadence (every 5-15 minutes or on topology changes).

3. **Implement the short-term scheduling agent.** Build a system prompt that accepts: (a) the current deployment plan, (b) live telemetry snapshot (per-node queue depth, p50/p99 latency, utilization %), (c) the incoming prompt metadata (modality, estimated token count, user ID), and (d) a fairness context window (recent per-user latency stats). The agent outputs a routing decision as JSON: `{node, model, priority, justification}`.

4. **Implement per-node deployment agents.** Each edge node runs an agent that manages container lifecycle. It receives deployment directives from the planner and executes them: pulling model images, allocating GPU memory via cgroups/nvidia-container-runtime, health-checking inference endpoints, and reporting status back.

5. **Build the telemetry pipeline.** Instrument each inference container to emit structured logs: request ID, arrival time, queue wait, inference duration, tokens processed, GPU utilization. Aggregate these into a time-series store (Prometheus, InfluxDB, or a simple SQLite ring buffer) queryable by the scheduling agent.

6. **Compute the Normalized Jain Index continuously.** Over a sliding window (e.g., last 100 requests), calculate `J = (sum(latency_i))^2 / (n * sum(latency_i^2))`. Inject this metric into the scheduler's context. If J drops below threshold (e.g., 0.85), flag it so the scheduler prioritizes underserved users.

7. **Wire inter-agent communication.** Use a lightweight message bus (Redis streams, NATS, or even a shared filesystem for prototyping) where agents post structured JSON messages. The planner publishes deployment plans; the scheduler publishes routing decisions; node agents publish status heartbeats and completion events.

8. **Implement experience logging and replay.** Log every scheduling decision alongside its outcome (actual latency, fairness impact). Periodically summarize these logs into a "lessons learned" document that is prepended to the planning agent's context, enabling it to improve deployment strategies without fine-tuning.

9. **Add overflow and failover logic.** When a node's queue exceeds a threshold or a node goes offline, the scheduler must reroute. Encode this as an explicit rule in the scheduler's system prompt: "If queue_depth > X or node is unreachable, exclude it from candidates and select the next-best option."

10. **Test with synthetic load and validate fairness.** Generate a mixed-modality request stream with skewed user distributions. Measure both average latency and Jain Index. Iterate on the planner and scheduler prompts until fairness targets are met without unacceptable latency regression.

## Concrete Examples

**Example 1: Designing a prompt routing config for 3 edge nodes**

User: "I have 3 edge servers: node-A (NVIDIA A100, 80GB), node-B (NVIDIA T4, 16GB), node-C (CPU-only, 64GB RAM). I need to serve Llama-3-70B, Stable Diffusion XL, and Whisper-large. Design the deployment and routing."

Approach:
1. Map resource requirements: Llama-3-70B needs ~40GB VRAM (quantized), SDXL needs ~8GB VRAM, Whisper-large needs ~4GB VRAM or can run on CPU.
2. Generate deployment plan: Llama-3-70B -> node-A (fits in 80GB with room for batching), SDXL -> node-B (fits in 16GB), Whisper-large -> node-C (CPU inference acceptable for audio).
3. Define routing rules: text prompts -> node-A, image generation -> node-B, audio transcription -> node-C. If node-B queue > 10, overflow small image tasks to node-A (which has spare VRAM).

Output:
```json
{
  "deployment_plan": {
    "node-A": {"models": ["llama-3-70b-q4"], "gpu_budget_gb": 45, "overflow_accepts": ["sdxl"]},
    "node-B": {"models": ["sdxl"], "gpu_budget_gb": 12},
    "node-C": {"models": ["whisper-large"], "cpu_cores": 16, "ram_gb": 32}
  },
  "routing_rules": {
    "text": {"primary": "node-A", "fallback": null},
    "image": {"primary": "node-B", "fallback": "node-A", "fallback_trigger": "queue_depth > 10"},
    "audio": {"primary": "node-C", "fallback": null}
  },
  "fairness_target": {"jain_index_min": 0.85, "window_size": 100}
}
```

**Example 2: Implementing the short-term scheduling agent prompt**

User: "Write the system prompt for the scheduling agent that picks which node handles each incoming request."

Approach:
1. Define the agent's role, inputs, and output schema.
2. Embed the fairness computation logic.
3. Include decision criteria prioritized by: feasibility > fairness > latency.

Output:
```markdown
## System Prompt: Short-Term Scheduling Agent

You are an inference request scheduler for a distributed edge network.

### Input (provided per request)
- `deployment_plan`: Current model-to-node mapping with resource budgets.
- `telemetry`: Per-node snapshot: {node_id, queue_depth, p50_latency_ms, p99_latency_ms, gpu_util_pct, cpu_util_pct}.
- `request`: {request_id, modality, estimated_tokens, user_id}.
- `fairness_window`: Last 100 requests: [{user_id, latency_ms}]. Current Jain Index: <value>.

### Decision Process
1. Filter to nodes that have a model matching the request modality.
2. Exclude nodes where gpu_util_pct > 95% or queue_depth > 20.
3. If Jain Index < 0.85, check if this user_id has above-average latency. If so, boost their priority by selecting the node with the shortest queue.
4. Among remaining candidates, select the node with the lowest estimated completion time: queue_depth * avg_latency + estimated_inference_time.
5. Output your decision as JSON.

### Output Schema
```json
{"node": "<node_id>", "model": "<model_id>", "priority": "normal|boosted", "reason": "<one sentence>"}
```

### Constraints
- Never route a request to a node that lacks the required model unless overflow is explicitly allowed in the deployment plan.
- Always provide a reason. This will be logged for experience replay.
```

**Example 3: Computing and using the Jain Index in Python**

User: "Show me how to compute the Normalized Jain Index and use it to flag unfairness in a request stream."

Output:
```python
from collections import deque
from dataclasses import dataclass

@dataclass
class RequestRecord:
    user_id: str
    latency_ms: float

class FairnessMonitor:
    def __init__(self, window_size: int = 100, threshold: float = 0.85):
        self.window: deque[RequestRecord] = deque(maxlen=window_size)
        self.threshold = threshold

    def record(self, user_id: str, latency_ms: float) -> None:
        self.window.append(RequestRecord(user_id, latency_ms))

    def jain_index(self) -> float:
        """Compute Normalized Jain Index over the sliding window."""
        if len(self.window) < 2:
            return 1.0
        latencies = [r.latency_ms for r in self.window]
        n = len(latencies)
        sum_x = sum(latencies)
        sum_x2 = sum(x ** 2 for x in latencies)
        if sum_x2 == 0:
            return 1.0
        return (sum_x ** 2) / (n * sum_x2)

    def underserved_users(self) -> set[str]:
        """Return user IDs whose avg latency exceeds the global average."""
        if not self.window:
            return set()
        from collections import defaultdict
        user_latencies = defaultdict(list)
        for r in self.window:
            user_latencies[r.user_id].append(r.latency_ms)
        global_avg = sum(r.latency_ms for r in self.window) / len(self.window)
        return {
            uid for uid, lats in user_latencies.items()
            if sum(lats) / len(lats) > global_avg * 1.2
        }

    def should_boost(self, user_id: str) -> bool:
        """True if fairness is low AND this user is underserved."""
        return self.jain_index() < self.threshold and user_id in self.underserved_users()
```

## Best Practices

- **Do** separate planning from scheduling. The planner runs on a slow cadence (minutes); the scheduler runs per-request (milliseconds). Mixing them creates either stale plans or slow scheduling.
- **Do** include the Jain Index and per-user latency history in every scheduling prompt. Without explicit fairness context, LLM-based schedulers default to throughput-maximizing behavior that starves minority request types.
- **Do** log every routing decision with its justification and outcome. This experience log is the primary mechanism for the system to improve over time without fine-tuning.
- **Do** use structured JSON for both agent inputs and outputs, with natural-language reasoning confined to a `reason` field. This keeps decisions machine-parseable while retaining interpretability.
- **Avoid** routing all requests through a single LLM call for scheduling -- the latency overhead defeats the purpose. Use the LLM-based scheduler only when heuristic fallbacks are insufficient (e.g., when fairness drops below threshold or topology changes).
- **Avoid** giving deployment agents write access to the scheduling policy. Maintain strict role separation: planners set policy, schedulers execute routing, node agents execute deployments.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Node goes offline | Heartbeat timeout (>10s) | Scheduler excludes node; planner redistributes models on next cycle |
| Model OOM on node | Container exit code 137 | Node agent evicts lowest-priority model, reports to planner |
| Scheduler LLM timeout | Response >500ms | Fall back to heuristic: route to node with shortest queue among feasible candidates |
| Jain Index collapse (<0.5) | Fairness monitor alert | Force round-robin scheduling for next N requests to reset fairness baseline |
| Telemetry pipeline lag | Stale timestamps (>30s old) | Scheduler uses last-known-good telemetry with a staleness penalty on estimated latency |
| Conflicting deployment directives | Two planner outputs within overlap window | Node agent applies the newer directive; acknowledges to planner with conflict flag |

## Limitations

- **LLM scheduling latency**: Using a foundation model for per-request routing adds overhead (50-200ms per decision). For ultra-low-latency requirements (<10ms routing), use heuristic scheduling with LLM-based agents only for periodic replanning.
- **Context window limits**: As the number of nodes and models grows (>50 nodes, >20 models), the telemetry snapshot may exceed the scheduler's context window. Summarize or sample telemetry for large deployments.
- **Fairness-latency tradeoff**: Boosting fairness to J > 0.95 may increase average latency by 10-20% because it forces suboptimal routing to equalize user experience. The threshold should be tuned per deployment.
- **Cold start**: The planning agent has no historical experience on first deployment. Seed it with synthetic workload traces or run a burn-in period with conservative round-robin routing.
- **Single-point-of-failure risk**: If the scheduling agent's backing LLM goes down, all routing stops. Always implement a heuristic fallback (e.g., least-loaded-node selection) that operates without LLM inference.

## Reference

Li, H., Madhukumar, H., Yan, S., Wu, Y., & Simeonidou, D. (2026). *Multi-Agentic AI for Fairness-Aware and Accelerated Multi-modal Large Model Inference in Real-world Mobile Edge Networks*. arXiv:2602.07215v1. https://arxiv.org/abs/2602.07215v1

Key sections to study: the three-agent decomposition (long-term planner, short-term scheduler, on-node deployer), the Normalized Jain Index formulation for fairness measurement, and the city-wide testbed architecture for containerized multi-model edge serving.