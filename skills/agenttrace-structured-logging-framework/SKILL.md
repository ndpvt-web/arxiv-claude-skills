---
name: "agenttrace-structured-logging-framework"
description: "Implement structured observability and telemetry for LLM agent systems using the AgentTrace three-surface logging pattern (operational, cognitive, contextual). Use when user says 'add agent tracing', 'instrument my agent', 'add observability to my agent system', 'structured logging for LLM agents', 'trace agent decisions', or 'monitor agent reasoning'."
---

# AgentTrace: Structured Logging for Agent System Observability

This skill enables Claude to implement the AgentTrace three-surface structured logging framework from AlSayyad et al. (AAAI 2026 Workshop LaMAS). AgentTrace captures runtime telemetry across three complementary surfaces — operational (what the agent did), cognitive (how the agent reasoned), and contextual (the environment and state surrounding each action) — using decorator-based instrumentation with minimal performance overhead. Unlike flat logging or ad-hoc print statements, this approach produces correlated, queryable trace graphs that support security auditing, debugging, real-time monitoring, and trust calibration for LLM-powered agents.

## When to Use

- When the user is building an LLM agent system and needs structured observability beyond basic `print()` or `logging.info()` calls
- When the user asks to trace agent decision-making, tool calls, or reasoning chains for debugging or auditing
- When the user needs to add security-oriented telemetry to an autonomous agent handling sensitive operations
- When the user wants to correlate LLM API calls with downstream tool invocations and state changes
- When the user is deploying agents in production and needs real-time monitoring with structured, queryable logs
- When the user asks to implement OpenTelemetry-style tracing specifically for agentic workflows
- When the user needs to debug nondeterministic agent behavior by reconstructing decision traces

## Key Technique

**The Three-Surface Model.** Traditional logging captures a flat stream of events. AgentTrace instead decomposes agent telemetry into three orthogonal surfaces that together provide full observability:

1. **Operational surface** — captures the mechanical execution layer: function calls, tool invocations, API requests, execution duration, return values, and errors. This is closest to traditional structured logging but scoped to agent-relevant operations.
2. **Cognitive surface** — captures the reasoning layer: LLM prompts, completions, token counts, sampling parameters, intermediate chain-of-thought outputs, and decision branch points. This surface is unique to LLM agent systems and is critical for auditing why an agent chose a particular action.
3. **Contextual surface** — captures the environmental layer: timestamps, state snapshots (memory, variable bindings, configuration), dependency relationships between traces, and correlation IDs that link parent-child operations into a trace tree.

**Decorator-Based Instrumentation.** Rather than requiring agents to be rewritten, AgentTrace uses Python decorators (or equivalent wrapper patterns in other languages) to intercept function entry/exit, serialize parameters and results, and emit structured trace records. This non-invasive approach means existing agent code can be instrumented by adding a single decorator per function, with surface selection controlled by configuration so you only capture what you need.

**Correlated Trace Graphs.** Each trace record carries a unique `trace_id` and optional `parent_trace_id`, allowing reconstruction of full execution trees. A single user request fans out into LLM calls, tool invocations, and state mutations — all linked by correlation IDs. This makes it possible to ask questions like "show me every state change caused by this specific LLM decision" or "what was the full reasoning chain that led to this tool call failing."

## Step-by-Step Workflow

1. **Define the trace data model.** Create dataclasses or TypedDicts for the three surface record types: `OperationalTrace`, `CognitiveTrace`, and `ContextualTrace`. Each must include `trace_id`, `parent_trace_id`, `timestamp`, and `surface`-specific fields (see examples below).

2. **Implement a trace context manager.** Build a thread-local (or async-context-local via `contextvars`) trace context that holds the current `trace_id` and propagates `parent_trace_id` automatically when nested operations occur. This is the backbone of correlation.

3. **Build the operational surface decorator.** Create an `@trace_operation` decorator that wraps any function to capture: function name, input arguments (sanitized), return value, execution duration, and any exceptions. Emit an `OperationalTrace` record on exit.

4. **Build the cognitive surface decorator.** Create an `@trace_cognition` decorator (or integrate into your LLM client wrapper) that captures: the full prompt/messages sent to the LLM, the raw completion, token usage (prompt/completion/total), model name, sampling parameters (temperature, top_p), and latency. Emit a `CognitiveTrace` record.

5. **Build the contextual surface capture.** Create a `capture_context()` function that snapshots relevant agent state — memory contents, current plan, environment variables, tool availability — and emits a `ContextualTrace` record. Call this at key decision points (before tool selection, after state mutation).

6. **Configure a trace sink.** Implement at least one trace sink: structured JSON lines to a file (for development), or an async queue feeding into a telemetry backend (for production). The sink must accept any surface record type and serialize it with its correlation IDs intact.

7. **Instrument the agent's core loop.** Apply `@trace_operation` to tool-calling functions and the main agent loop. Apply `@trace_cognition` to the LLM call wrapper. Insert `capture_context()` calls before major decisions and after state mutations.

8. **Add surface filtering.** Implement a configuration flag (environment variable or config file) that controls which surfaces are active. In production, you may want only operational + contextual; during debugging, enable all three including full cognitive traces.

9. **Build a trace query utility.** Write a simple function that reads the JSON-lines trace log, reconstructs the trace tree from `trace_id`/`parent_trace_id` relationships, and can filter by surface type, time range, or error status. This is essential for post-hoc analysis.

10. **Validate with a smoke test.** Run the instrumented agent on a sample task and verify that: (a) all three surfaces emit records, (b) correlation IDs form a valid tree, (c) cognitive traces capture full LLM interactions, and (d) overhead is under 5% of baseline execution time.

## Concrete Examples

**Example 1: Instrumenting a tool-calling agent in Python**

User: "Add structured tracing to my LangChain-style agent so I can debug why it's calling the wrong tools."

Approach:
1. Define the three trace record types
2. Implement trace context propagation
3. Wrap the agent's tool execution and LLM calls with surface decorators
4. Output correlated JSON-lines traces

Output:

```python
import uuid
import time
import json
import contextvars
from dataclasses import dataclass, field, asdict
from typing import Any, Optional
from functools import wraps

# --- Trace Context Propagation ---
_current_trace = contextvars.ContextVar("current_trace", default=None)

def get_trace_context() -> Optional[str]:
    return _current_trace.get()

# --- Trace Data Model (Three Surfaces) ---
@dataclass
class OperationalTrace:
    surface: str = "operational"
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    parent_trace_id: Optional[str] = None
    timestamp: float = field(default_factory=time.time)
    function_name: str = ""
    args_summary: str = ""
    result_summary: str = ""
    duration_ms: float = 0.0
    error: Optional[str] = None

@dataclass
class CognitiveTrace:
    surface: str = "cognitive"
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    parent_trace_id: Optional[str] = None
    timestamp: float = field(default_factory=time.time)
    model: str = ""
    messages: list = field(default_factory=list)
    completion: str = ""
    token_usage: dict = field(default_factory=dict)
    temperature: float = 0.0
    latency_ms: float = 0.0

@dataclass
class ContextualTrace:
    surface: str = "contextual"
    trace_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    parent_trace_id: Optional[str] = None
    timestamp: float = field(default_factory=time.time)
    agent_state: dict = field(default_factory=dict)
    available_tools: list = field(default_factory=list)
    memory_snapshot: dict = field(default_factory=dict)

# --- Trace Sink ---
TRACE_LOG = "agent_traces.jsonl"

def emit_trace(record):
    with open(TRACE_LOG, "a") as f:
        f.write(json.dumps(asdict(record), default=str) + "\n")

# --- Surface Decorators ---
def trace_operation(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        trace = OperationalTrace(
            parent_trace_id=get_trace_context(),
            function_name=func.__name__,
            args_summary=str(args[:3])[:200],
        )
        token = _current_trace.set(trace.trace_id)
        start = time.perf_counter()
        try:
            result = func(*args, **kwargs)
            trace.result_summary = str(result)[:200]
            return result
        except Exception as e:
            trace.error = f"{type(e).__name__}: {e}"
            raise
        finally:
            trace.duration_ms = (time.perf_counter() - start) * 1000
            emit_trace(trace)
            _current_trace.reset(token)
    return wrapper

def trace_cognition(func):
    @wraps(func)
    def wrapper(messages, **kwargs):
        trace = CognitiveTrace(
            parent_trace_id=get_trace_context(),
            model=kwargs.get("model", "unknown"),
            messages=messages,
            temperature=kwargs.get("temperature", 0.0),
        )
        token = _current_trace.set(trace.trace_id)
        start = time.perf_counter()
        try:
            result = func(messages, **kwargs)
            trace.completion = result.get("content", "")[:500]
            trace.token_usage = result.get("usage", {})
            return result
        except Exception as e:
            trace.completion = f"ERROR: {e}"
            raise
        finally:
            trace.latency_ms = (time.perf_counter() - start) * 1000
            emit_trace(trace)
            _current_trace.reset(token)
    return wrapper

# --- Usage on an agent ---
@trace_operation
def execute_tool(tool_name: str, tool_input: dict) -> str:
    # Your existing tool execution logic
    return tools[tool_name].run(tool_input)

@trace_cognition
def call_llm(messages, **kwargs):
    # Your existing LLM call logic
    response = client.chat.completions.create(messages=messages, **kwargs)
    return {"content": response.choices[0].message.content,
            "usage": {"prompt": response.usage.prompt_tokens,
                      "completion": response.usage.completion_tokens}}

@trace_operation
def agent_step(user_query: str):
    # Capture context before decision
    emit_trace(ContextualTrace(
        parent_trace_id=get_trace_context(),
        agent_state={"step": "planning", "query": user_query},
        available_tools=list(tools.keys()),
    ))
    result = call_llm([{"role": "user", "content": user_query}], model="gpt-4")
    tool_call = parse_tool_call(result["content"])
    if tool_call:
        return execute_tool(tool_call["name"], tool_call["input"])
    return result["content"]
```

**Example 2: Querying traces to debug a failed tool call**

User: "My agent keeps calling the search tool with malformed queries. How do I find out why?"

Approach:
1. Read the JSONL trace log
2. Reconstruct the trace tree around failed search calls
3. Correlate back to the cognitive trace that produced the decision

Output:

```python
import json

def load_traces(path="agent_traces.jsonl"):
    traces = []
    with open(path) as f:
        for line in f:
            traces.append(json.loads(line))
    return traces

def find_failed_tool_calls(traces, tool_name="search"):
    """Find operational traces where a specific tool errored."""
    return [t for t in traces
            if t["surface"] == "operational"
            and tool_name in t.get("function_name", "")
            and t.get("error")]

def get_parent_chain(traces, trace_id):
    """Walk up the parent chain to reconstruct the full decision path."""
    index = {t["trace_id"]: t for t in traces}
    chain = []
    current = index.get(trace_id)
    while current:
        chain.append(current)
        current = index.get(current.get("parent_trace_id"))
    return list(reversed(chain))

def debug_failed_search(path="agent_traces.jsonl"):
    traces = load_traces(path)
    failures = find_failed_tool_calls(traces, "search")
    for fail in failures[-3:]:  # Last 3 failures
        print(f"\n--- Failed call: {fail['error']} ---")
        chain = get_parent_chain(traces, fail["trace_id"])
        for step in chain:
            surface = step["surface"]
            if surface == "cognitive":
                print(f"  [COGNITIVE] Model: {step['model']}")
                print(f"    Completion: {step['completion'][:150]}...")
            elif surface == "operational":
                print(f"  [OPERATIONAL] {step['function_name']} "
                      f"({step['duration_ms']:.0f}ms)")
                if step.get("error"):
                    print(f"    ERROR: {step['error']}")
            elif surface == "contextual":
                print(f"  [CONTEXTUAL] State: {step['agent_state']}")

# Output shows the full chain: what state the agent was in (contextual),
# what the LLM produced (cognitive), and what tool call failed (operational).
```

**Example 3: Adding surface filtering for production deployment**

User: "I want full traces in dev but only operational logs in production to reduce overhead."

Approach:
1. Add an environment-based surface configuration
2. Gate trace emission on the active surfaces

Output:

```python
import os

ACTIVE_SURFACES = set(
    os.environ.get("AGENTTRACE_SURFACES", "operational,cognitive,contextual")
    .split(",")
)

def emit_trace(record):
    if record.surface not in ACTIVE_SURFACES:
        return
    with open(TRACE_LOG, "a") as f:
        f.write(json.dumps(asdict(record), default=str) + "\n")

# In production: AGENTTRACE_SURFACES=operational
# In staging:    AGENTTRACE_SURFACES=operational,contextual
# In dev:        AGENTTRACE_SURFACES=operational,cognitive,contextual
```

## Best Practices

- **Do:** Sanitize sensitive data before logging. Strip API keys, user PII, and credentials from `args_summary` and `messages` fields. The cognitive surface captures full prompts — redact secrets before emission.
- **Do:** Use `contextvars` (Python) or `AsyncLocalStorage` (Node.js) for trace context propagation rather than thread-local storage, especially in async agent code. This ensures parent-child correlation survives across `await` boundaries.
- **Do:** Keep trace record serialization cheap. Use `default=str` in `json.dumps` and truncate large fields (completions, tool outputs) to a configurable max length. Full payloads can be stored in a separate blob store keyed by `trace_id`.
- **Do:** Emit traces asynchronously in production. Use a bounded queue with a background writer thread to avoid blocking the agent's hot path on file I/O.
- **Avoid:** Logging the full content of every LLM completion in production. Cognitive traces are valuable for debugging but generate high volume. Use surface filtering to disable them unless actively investigating an issue.
- **Avoid:** Nesting decorators without trace context propagation. If `@trace_operation` wraps a function that calls another `@trace_operation` function, ensure the inner call reads `parent_trace_id` from the context — otherwise you get a flat list instead of a tree.

## Error Handling

- **Missing parent trace context:** If `get_trace_context()` returns `None`, emit the trace with `parent_trace_id=None`. This creates a root trace. Do not skip emission — orphan traces are still valuable.
- **Serialization failures:** Wrap `json.dumps` in a try/except. If a trace record contains non-serializable objects, fall back to `repr()` for the problematic field rather than dropping the entire trace.
- **High-volume trace storms:** If the agent enters a tight retry loop, the trace sink can be overwhelmed. Implement a rate limiter (e.g., max 100 traces/second per surface) with a counter that logs a "traces dropped" summary rather than silently losing data.
- **Async context loss:** In Python, `contextvars` work correctly with `asyncio` but not with raw threads. If the agent uses thread pools, manually copy the context using `contextvars.copy_context()` before dispatching.

## Limitations

- **Not a security enforcement layer.** AgentTrace provides observability and audit trails, but it does not prevent malicious agent actions. It must be paired with guardrails, sandboxing, or policy enforcement for actual security.
- **Cognitive surface overhead.** Capturing full LLM prompts and completions adds non-trivial storage cost. For agents making many LLM calls per task, cognitive traces can dominate log volume by 10-100x over operational traces.
- **No standardized schema yet.** The AgentTrace paper proposes a framework, but there is no industry-standard schema for agent telemetry. Interoperability with OpenTelemetry or other tracing systems requires custom exporters.
- **Post-hoc analysis only.** The trace data supports debugging and auditing after the fact. Real-time anomaly detection on trace streams requires additional infrastructure (stream processing, alerting rules) not covered by the core framework.
- **Language-specific.** The decorator pattern maps naturally to Python. For other languages (TypeScript, Go, Rust), equivalent wrapper patterns (middleware, aspects, macros) require different implementation strategies.

## Reference

[AgentTrace: A Structured Logging Framework for Agent System Observability](https://arxiv.org/abs/2602.10133v1) — AlSayyad, Huang, Pal (AAAI 2026 Workshop LaMAS). Focus on Section 3 for the three-surface decomposition and Section 4 for the instrumentation architecture and trace correlation model.