---
name: "agenttrace-structured-logging-framework"
description: "Implement structured, multi-surface observability logging for LLM agent systems using the AgentTrace pattern: operational, cognitive, and contextual trace surfaces with unified envelopes, span hierarchies, and dual-path storage. Use when asked to: 'add observability to my agent', 'log agent reasoning traces', 'instrument LLM tool calls', 'build agent telemetry', 'trace agent decisions', 'monitor agent security'."
---

# AgentTrace: Structured Multi-Surface Logging for LLM Agent Systems

This skill enables Claude to implement the AgentTrace observability pattern from AlSayyad et al. (AAAI 2026 Workshop LaMAS) for any LLM agent codebase. AgentTrace instruments agents at runtime across three distinct logging surfaces -- operational (method calls, timing), cognitive (LLM reasoning chains, plans, reflections), and contextual (external system interactions) -- unified under a common structured envelope with trace/span ID hierarchies. Unlike bolting on generic logging, this approach treats agent reasoning as a first-class telemetry signal, enabling security forensics, accountability auditing, and real-time monitoring of nondeterministic agent behavior.

## When to Use

- When the user asks to add observability, tracing, or structured logging to an LLM agent or agentic system
- When building a new agent framework and needs a telemetry layer from the start
- When the user wants to trace agent reasoning (chain-of-thought, plans, reflections) alongside tool calls
- When instrumenting agent interactions with external APIs, databases, vector stores, or file systems
- When the user needs audit logs for compliance, security forensics, or post-incident investigation of agent behavior
- When adding real-time monitoring or anomaly detection to deployed agents
- When debugging nondeterministic agent failures where traditional logging is insufficient

## Key Technique

AgentTrace's core insight is that LLM agents produce three fundamentally different categories of observable signals, and collapsing them into a single log stream destroys the structure needed for security analysis and debugging. The framework defines three **logging surfaces**: (1) **Operational** -- every method invocation with arguments, return values, timing, and status; (2) **Cognitive** -- LLM prompts, completions, extracted reasoning segments (thoughts, plans, reflections), model identifiers, and token counts; (3) **Contextual** -- outbound interactions with HTTP APIs, SQL/NoSQL databases, caches, vector stores, and file systems, captured via auto-instrumentation patches on standard libraries.

All three surfaces share a **common envelope schema**: a UUID, surface type tag, trace ID, span ID, UTC timestamp, and a surface-specific event body. This envelope enables cross-surface causal analysis -- linking an agent's internal plan (cognitive) to the method that executed it (operational) to the database write it produced (contextual). Storage is dual-path: JSONL for low-latency local debugging, and OpenTelemetry spans for scalable remote observability via backends like Jaeger or Grafana Tempo.

Instrumentation is applied via a **decorator injection pattern** that wraps target methods at runtime without modifying agent source code. Each wrapper emits a start event (method name, args summary, timestamp), executes the original function, optionally extracts cognitive content from LLM responses, then emits a completion or error event. The approach generates exactly two events per successful call path, keeps overhead low through batched async export, and degrades gracefully to local JSONL when remote backends are unavailable.

## Step-by-Step Workflow

1. **Define the envelope schema.** Create a base dataclass or TypedDict with fields: `id` (UUID4), `surface` (enum: operational | cognitive | contextual), `trace_id` (propagated string), `span_id` (fresh UUID4 per event), `timestamp` (UTC ISO-8601), and `body` (dict, surface-specific).

2. **Implement surface-specific body schemas.** For operational: `method`, `status` (start | complete | error), `duration_ms`, `args_summary`, `result_type`. For cognitive: `thought`, `plan`, `reflection`, `model`, `prompt_tokens`, `completion_tokens`. For contextual: `operation` (http | sql | cache | vectordb | filesystem), `target`, `query_summary`, `response_summary`, `row_count` or `status_code`.

3. **Build the ALogger class.** Implement a logger that accepts surface-typed events, validates them against the schema, serializes to JSONL (one line per event), and optionally exports as OpenTelemetry spans. Include a batch processor for async export and defensive serialization with safe type coercion.

4. **Create the `instrument_agent()` decorator injector.** Write a function that takes an agent instance and an optional method allowlist, iterates over public callables, and replaces each with a wrapper that: generates/propagates trace_id and span_id, emits an operational start event, calls the original method, extracts cognitive content if the return contains LLM output, emits an operational complete/error event, and re-raises any exceptions.

5. **Implement cognitive extraction.** Parse LLM responses to identify reasoning segments using marker detection (`<think>`, `## Plan`, `Reflection:`), XML tag parsing, or JSON field extraction. Store extracted `thought`, `plan`, and `reflection` excerpts with associated model name and token counts.

6. **Wire up contextual auto-instrumentation.** Patch standard libraries (e.g., `requests`, `httpx`, `sqlalchemy`, `redis`) using OpenTelemetry auto-instrumentation or lightweight monkey-patches to emit contextual events for every outbound call, capturing URL/headers for HTTP, query/row-count for SQL, and key/operation for caches.

7. **Initialize with `init()` entrypoint.** Provide a single configuration function that sets up local JSONL sinks (file path, rotation), optionally enables OpenTelemetry export (endpoint, service name, batch size), and activates contextual auto-instrumentation hooks.

8. **Add trace context propagation.** Ensure the trace_id is generated once per top-level agent invocation and propagated through all nested calls via thread-local or contextvars, so operational, cognitive, and contextual events from the same execution share one trace_id.

9. **Implement graceful degradation.** If OpenTelemetry export fails or the backend is unreachable, fall back silently to JSONL-only logging. Serialization errors should be caught and logged as warning-level events rather than crashing the agent.

10. **Add query and visualization helpers.** Provide utility functions to load JSONL traces, filter by surface/trace_id/time range, and reconstruct the causal chain (operational -> cognitive -> contextual) for a given agent invocation.

## Concrete Examples

**Example 1: Instrumenting a ReAct Agent**

User: "Add structured logging to my ReAct agent so I can trace its reasoning and tool calls."

Approach:
1. Identify the agent class and its key methods (e.g., `run`, `think`, `act`, `observe`)
2. Define the three-surface envelope schema
3. Apply `instrument_agent()` to wrap those methods
4. Add cognitive extraction for the LLM's chain-of-thought output
5. Initialize with JSONL output

Output (JSONL, one event per line):
```json
{"id": "a1b2c3d4", "surface": "operational", "trace_id": "tr-9f8e7d", "span_id": "sp-001", "timestamp": "2026-02-07T14:30:01.123Z", "body": {"method": "think", "status": "start", "args_summary": {"query": "What is the weather in NYC?"}}}
{"id": "a1b2c3d5", "surface": "cognitive", "trace_id": "tr-9f8e7d", "span_id": "sp-002", "timestamp": "2026-02-07T14:30:01.850Z", "body": {"thought": "I need to call the weather API for New York City", "plan": "Use get_weather tool with location=NYC", "reflection": null, "model": "claude-sonnet-4-20250514", "prompt_tokens": 312, "completion_tokens": 45}}
{"id": "a1b2c3d6", "surface": "contextual", "trace_id": "tr-9f8e7d", "span_id": "sp-003", "timestamp": "2026-02-07T14:30:02.100Z", "body": {"operation": "http", "target": "https://api.weather.com/v1/current", "query_summary": "GET ?location=NYC", "status_code": 200, "response_summary": "temp=42F, conditions=cloudy"}}
{"id": "a1b2c3d7", "surface": "operational", "trace_id": "tr-9f8e7d", "span_id": "sp-001", "timestamp": "2026-02-07T14:30:02.200Z", "body": {"method": "think", "status": "complete", "duration_ms": 1077, "result_type": "str"}}
```

**Example 2: Adding Agent Audit Logging for Compliance**

User: "We need to audit every decision our customer-service agent makes for regulatory compliance. Add traceability."

Approach:
1. Initialize AgentTrace with append-only JSONL files (one per day, rotated)
2. Instrument the agent's decision methods plus all database and API calls
3. Enable cognitive surface to capture why the agent chose each action
4. Add trace_id propagation so each customer interaction has one trace
5. Build a query helper to reconstruct the full decision chain per trace_id

Output (reconstructed trace for audit):
```
Trace: tr-abc123 | Customer Interaction #4892 | 2026-02-07T09:15:00Z
-----------------------------------------------------------------------
[operational] resolve_ticket.start  args={ticket_id: 4892}
[cognitive]   thought: "Customer reports duplicate charge. Checking transaction history."
[cognitive]   plan: "1. Query transactions DB  2. Verify duplicate  3. Issue refund if confirmed"
[contextual]  sql: SELECT * FROM transactions WHERE customer_id=771 AND amount=49.99  -> 2 rows
[cognitive]   reflection: "Found two identical charges 3 min apart. High confidence this is a duplicate."
[contextual]  http: POST /api/refunds {transaction_id: "tx-8821", amount: 49.99}  -> 201
[operational] resolve_ticket.complete  duration=3420ms  result=refund_issued
```

**Example 3: Retrofitting Tracing onto a LangChain Agent**

User: "I have an existing LangChain agent. Add AgentTrace without modifying my agent code."

Approach:
1. Import the AgentTrace init and instrument functions
2. Call `init(jsonl_path="./traces/", otel_endpoint="http://localhost:4317")`
3. Call `instrument_agent(agent, "my_langchain_agent")` -- this wraps all public methods via the decorator pattern without touching agent source code
4. LangChain's internal `requests`/`httpx` calls are auto-instrumented for contextual traces
5. Cognitive extraction parses LLM outputs from LangChain's callback structure

```python
from agenttrace import init, instrument_agent

# One-time setup
init(
    jsonl_path="./traces/agent.jsonl",
    otel_endpoint="http://localhost:4317",
    service_name="my-langchain-agent",
    auto_instrument=["requests", "httpx", "sqlalchemy"]
)

# Wrap without modifying agent code
instrument_agent(agent_executor, "customer_support_agent")

# Agent runs as normal -- traces are emitted automatically
result = agent_executor.invoke({"input": "Check my order status"})
```

## Best Practices

- **Do:** Use a single `trace_id` per top-level user interaction and propagate it through all nested calls via `contextvars`. This is what makes cross-surface causal analysis possible.
- **Do:** Store cognitive extracts (thought, plan, reflection) as separate fields, not a single blob. This enables filtering and anomaly detection on specific reasoning phases.
- **Do:** Implement dual-path storage (JSONL + OpenTelemetry) from the start. JSONL provides zero-dependency local access; OTel enables production dashboards.
- **Do:** Validate events against the envelope schema at write time. Malformed traces are worse than missing traces for security auditing.
- **Avoid:** Logging full LLM prompts/completions in the cognitive surface when they contain PII or secrets. Summarize or redact sensitive fields before recording.
- **Avoid:** Making instrumentation synchronous and blocking. Use batched async export so trace emission does not degrade agent response latency.
- **Avoid:** Catching and swallowing exceptions in the instrumentation wrapper. Always re-raise after recording the error event -- the wrapper must be transparent to the agent's error handling.

## Error Handling

- **Serialization failures:** If an argument or return value is not JSON-serializable, fall back to `repr()` with truncation. Log a warning-level event but do not crash the agent.
- **OpenTelemetry backend unreachable:** Degrade to JSONL-only mode silently. Queue a bounded buffer of spans for retry, but drop oldest spans if the buffer fills rather than consuming unbounded memory.
- **Cognitive extraction misparse:** If the LLM response does not contain recognizable reasoning markers (no `<think>` tags, no structured plan), record the cognitive event with `thought: null, plan: null, reflection: null` rather than skipping it. The absence of reasoning is itself a signal.
- **Trace ID propagation loss:** If a method is called outside an active trace context (e.g., from a background thread), generate a new trace_id and log a warning. Do not silently drop the event.
- **High-cardinality arguments:** Truncate `args_summary` and `result_type` fields to a configurable max length (default 512 chars) to prevent log bloat from large payloads.

## Limitations

- **No causal guarantees across async boundaries.** If the agent uses fire-and-forget async tasks or message queues, trace_id propagation requires explicit context passing that AgentTrace's decorator pattern cannot automatically inject.
- **Cognitive extraction is heuristic.** Parsing reasoning segments from LLM output depends on model-specific formatting (XML tags, markdown headers). Models that don't emit structured reasoning produce empty cognitive traces.
- **Overhead scales with method count.** Instrumenting every method on a deeply nested object graph adds measurable latency. Use the method allowlist to target only semantically meaningful methods.
- **Not a replacement for input/output guardrails.** AgentTrace observes and records behavior -- it does not prevent harmful actions. Pair it with runtime safety filters for defense in depth.
- **JSONL files grow unboundedly.** Production deployments need log rotation and retention policies. AgentTrace provides the emission layer, not the lifecycle management.

## Reference

- **Paper:** [AgentTrace: A Structured Logging Framework for Agent System Observability](https://arxiv.org/abs/2602.10133v1) -- AlSayyad, Huang, Pal (AAAI 2026 Workshop LaMAS). Focus on Section 3 (three-surface architecture), Algorithm 1 (runtime instrumentation wrapper), and Section 4 (security implications of structured agent telemetry).