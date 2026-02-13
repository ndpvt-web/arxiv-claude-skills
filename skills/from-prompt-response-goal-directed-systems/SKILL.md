---
name: "from-prompt-response-goal-directed-systems"
description: "Design production-grade agentic AI architectures with separated cognition/execution layers, typed tool interfaces, multi-agent topologies, and enterprise hardening. Use when: 'design an agent system', 'build a multi-agent architecture', 'add governance to my AI pipeline', 'harden my LLM agent for production', 'create a tool registry for agents', 'architect agent-to-agent coordination'."
---

This skill enables Claude to architect goal-directed agentic AI systems by applying the reference architecture from Alenezi (2026). Instead of treating LLM agents as monolithic prompt-response functions, this skill teaches you to decompose agent systems into separated layers — cognitive reasoning, control flow, memory, tool execution, and governance — connected by typed contracts. It covers single-agent loop design, multi-agent topology selection with failure-mode awareness, and a concrete enterprise hardening checklist for production deployment.

## When to Use

- When the user asks to design or architect an LLM-based agent system from scratch
- When building a multi-agent pipeline and choosing between orchestrator-worker, router-solver, hierarchical, or swarm topologies
- When adding production hardening to an existing agent — governance, observability, budgets, or policy enforcement
- When designing a tool registry or typed tool interface layer for agent-tool communication
- When refactoring a monolithic LLM chain into a layered agent architecture with separated concerns
- When reviewing an agent system for failure modes like unbounded loops, context pollution, or cascading tool calls
- When implementing memory tiers (working, episodic, semantic, preference) for a long-running agent

## Key Technique

The paper's core insight is that production-grade LLM agents must separate **cognition** (the LLM's reasoning) from **control flow** (planning, retries, circuit breakers), **memory** (tiered storage with access control), **tool execution** (sandboxed, typed, versioned), and **governance** (policy gates, audit, RBAC). This separation mirrors how web services matured: monolithic CGI scripts gave way to layered architectures with typed APIs, registries, and middleware. The same evolution applies to agents.

The architecture implements a **goal-directed loop** (perceive → plan → act → reflect) bounded by explicit resource budgets (max steps, token caps, cost limits, time limits). Every side-effecting action passes through a **policy enforcement gateway** before execution. Tools are not ad-hoc function calls — they are registry entries with typed schemas, version tracking, and sandboxed execution under least privilege. This makes every agent run auditable and reproducible.

For multi-agent systems, the paper provides a topology taxonomy with mapped failure modes. An orchestrator-worker topology risks silent worker failure (mitigated by heartbeats and ACK/NACK). A swarm topology risks herding behavior (mitigated by entropy-preserving incentives). Choosing the right topology is an architectural decision with direct reliability consequences, not a stylistic preference.

## Step-by-Step Workflow

1. **Identify the goal structure.** Determine whether the task is single-goal (one agent loop suffices) or multi-goal/decomposable (requires multi-agent coordination). Map user intent to explicit goals and constraints using the BDI frame: Beliefs (world state + memory), Desires (goals + policies), Intentions (planned actions).

2. **Design the layered stack.** Create these separated layers for each agent:
   - **Agent Core**: LLM reasoning component (model selection, prompt template, system instructions)
   - **Control Layer**: Planner with state machine, retry/backoff logic, circuit breakers, max-step limits
   - **Memory Layer**: Working memory (current context window), episodic store (past interaction summaries), semantic KB (vector store or knowledge graph), preference store (user-specific constraints)
   - **Tooling Layer**: Tool registry with typed schemas, sandboxed execution environments, RAG retrieval endpoints
   - **Governance Layer** (cross-cutting): RBAC, policy-as-code gates, audit logging, cost/rate limits

3. **Define typed tool interfaces.** For every tool the agent can invoke, create a schema specifying: input types and required fields, output types, preconditions, idempotency guarantees, version identifier, and required permissions. Treat the tool registry as an API gateway — tools are discoverable, versioned, and access-controlled.

4. **Implement the agent loop with budgeted autonomy.** Code the core loop:
   ```
   initialize state from goal
   for step in 1..K_MAX:
       context = build_context(state, memory, policies)
       action = llm.propose_action(context)
       if violates_policy(action): action = repair_or_escalate(action)
       if is_tool_call(action):
           result = execute_tool(action, sandbox, schema_validated=True)
           update_state(result)
           write_to_memory(action, result)
       elif is_final_answer(action):
           return action
       check_budget(tokens, cost, time, tool_calls)
   return graceful_degradation_response()
   ```

5. **Select a multi-agent topology** (if multi-agent). Choose based on task structure:
   - **Orchestrator-Worker**: Central manager decomposes and delegates. Best for well-defined subtask boundaries. Guard against silent worker failure with heartbeats.
   - **Router-Solver**: Classifier routes incoming requests to specialized solvers. Best for heterogeneous request types. Guard against misrouting with confidence thresholds.
   - **Hierarchical**: Recursive decomposition through management layers. Best for deeply nested problems. Guard against command distortion with signed intent propagation.
   - **Swarm/Market**: Decentralized agents bid for tasks. Best for embarrassingly parallel work. Guard against herding with anti-correlation penalties.

6. **Wire failure-mode mitigations into the architecture.** For each topology, implement the specific mitigations:
   - Heartbeats + ACK/NACK for worker liveness
   - DAG enforcement to prevent delegation deadlocks
   - Circuit breakers when error rates spike above threshold
   - Idempotent tool design to prevent duplicate side effects
   - Human-in-the-loop gates for high-risk actions (financial transactions, data deletion, external API writes)

7. **Implement the governance layer.** Apply the enterprise hardening checklist:
   - Identity: Short-lived credentials, per-agent RBAC, least privilege
   - Policy: Central policy gate evaluating every action before execution, policy-as-code with version control
   - Observability: End-to-end structured traces capturing model ID, prompt version, tool versions, policy decisions, memory operations, principal identity, and resource budgets
   - Budgets: Explicit caps on tokens, time, cost, and tool invocations per run

8. **Configure memory with access control.** Implement tiered memory with PII filtering, retention policies, and policy-aware retrieval. Episodic memory should summarize past interactions indexed by task and time. Semantic memory should use vector stores with access scoping.

9. **Set up CI/CD evaluation.** Create a continuous eval pipeline with regression benchmarks, safety tests (prompt injection, adversarial inputs), and schema contract tests for all tool interfaces.

10. **Validate with trace analysis.** Run the system end-to-end and verify that every trace contains: the complete action sequence, policy decisions at each step, resource consumption metrics, and a reproducible execution path.

## Concrete Examples

**Example 1: Designing a customer support agent system**

User: "Design an agent architecture for handling customer support tickets — it needs to read tickets, query our knowledge base, escalate to humans when needed, and log everything for compliance."

Approach:
1. Map the BDI frame — Beliefs: ticket content + KB articles + customer history. Desires: resolve ticket within SLA, comply with data policies. Intentions: search KB, draft response, escalate if low confidence.
2. Design a single-agent layered stack:
   - Agent Core: LLM with support-domain system prompt
   - Control: State machine with states `CLASSIFY → SEARCH_KB → DRAFT_RESPONSE → REVIEW → RESPOND | ESCALATE`
   - Memory: Working (current ticket), Episodic (past resolutions for this customer), Semantic (KB vector store)
   - Tools: `read_ticket(id) → Ticket`, `search_kb(query) → Article[]`, `send_response(ticket_id, message) → Ack`, `escalate(ticket_id, reason) → Ack`
   - Governance: PII filtering on all memory writes, RBAC restricting `send_response` to verified agent identity, audit trail on every action
3. Set K_MAX=15, token budget=8000, cost cap=$0.50 per ticket
4. Policy gate: if confidence < 0.7 on draft → mandatory escalation

Output structure:
```python
# Tool registry entry example
tool_registry = {
    "search_kb": {
        "version": "1.2.0",
        "input_schema": {"query": "string", "max_results": "int", "filters": "dict?"},
        "output_schema": {"articles": "Article[]", "scores": "float[]"},
        "preconditions": ["authenticated", "ticket_context_loaded"],
        "idempotent": True,
        "permissions": ["kb:read"],
        "sandbox": "network_restricted"
    }
}
```

**Example 2: Choosing a multi-agent topology for a data pipeline**

User: "I have an ETL pipeline where different data sources need different extraction logic, then everything gets transformed and loaded. Should this be one agent or multiple?"

Approach:
1. Identify decomposable goals: extraction is heterogeneous (different sources), transformation and loading are uniform
2. Select **Router-Solver** topology: a router agent classifies incoming data source type, routes to specialized extractor agents (one per source type), results converge to a shared transform-load agent
3. Map failure modes:
   - Misrouting risk: router sends Salesforce data to the PostgreSQL extractor → mitigate with schema fingerprinting and confidence threshold (reject if < 0.85)
   - Solver overload: one source generates 10x more data → mitigate with backpressure and per-solver queue limits
4. Define typed contracts between agents:
   ```
   ExtractorOutput = { records: Record[], source_id: str, schema_version: str, extraction_ts: datetime }
   ```

Output: Architecture diagram description + implementation skeleton with router logic, solver registration, and typed message contracts.

**Example 3: Hardening an existing agent for production**

User: "I have a LangChain agent that works in dev. What do I need before deploying to production?"

Approach:
1. Audit against the enterprise hardening checklist:
   - Identity: Replace any hardcoded API keys with short-lived credentials via secrets manager. Add per-user RBAC.
   - Policy: Wrap all tool calls in a policy gate. Define policy-as-code rules (e.g., "no external API calls without user confirmation for writes").
   - Observability: Add structured tracing (OpenTelemetry) capturing model ID, prompt hash, tool versions, and latency per step.
   - Budgets: Set K_MAX, token ceiling, cost cap, and max tool calls. Add circuit breaker for error rate > 20%.
   - Memory: Add PII scrubbing before any data persists to episodic memory. Set 30-day retention.
   - Security: Add prompt injection test suite. Sandbox tool execution (no shell access, restricted network).
   - CI/CD: Add eval pipeline with golden-set regression tests and adversarial prompt tests.
2. Implement changes incrementally, starting with budgets and policy gates (highest risk reduction).

Output: Prioritized checklist with specific code changes mapped to files in the existing codebase.

## Best Practices

- **Do:** Treat every tool as a versioned, schema-validated registry entry — not an ad-hoc function. This enables audit, rollback, and contract testing.
- **Do:** Set explicit budget caps (K_MAX steps, token ceiling, cost limit, wall-clock timeout) on every agent loop. Unbounded loops are the most common production failure.
- **Do:** Implement policy gates as a separate layer that evaluates every proposed action before execution. Never embed policy logic inside prompts alone — prompts are not reliable enforcement mechanisms.
- **Do:** Record structured traces for every run. In stochastic systems, traces are your primary debugging tool. Include model ID, prompt version, tool versions, and all policy decisions.
- **Avoid:** Monolithic agents that combine reasoning, tool execution, memory, and governance in a single prompt chain. This creates context pollution and makes failures undiagnosable.
- **Avoid:** Skipping failure-mode analysis when choosing a multi-agent topology. Each topology has specific failure modes — designing without mitigations leads to silent failures in production.

## Error Handling

- **Unbounded loops**: Agent cycles without progress. Detect by tracking state deltas between steps — if state is unchanged for 3 consecutive steps, trigger circuit breaker and return graceful degradation response.
- **Context pollution**: Long-running agents accumulate irrelevant context. Mitigate by summarizing episodic memory rather than appending raw history, and by scoping working memory to the current subtask.
- **Tool schema violations**: Agent proposes a tool call with invalid parameters. Validate against the typed schema before execution. On violation, feed the validation error back to the LLM for self-correction (up to 2 retries, then escalate).
- **Cascading failures in multi-agent systems**: One agent's hallucinated output becomes another agent's input. Mitigate with output validation at agent boundaries and confidence-gated handoffs.
- **Policy bypass through delegation**: A restricted agent delegates a forbidden action to a higher-privilege agent. Enforce that delegated actions inherit the caller's permission scope, not the callee's.

## Limitations

- The reference architecture adds genuine complexity. For simple single-turn tool-use agents (one tool, no memory, no compliance requirements), the full layered stack is over-engineering. Apply proportionally.
- The paper's multi-agent topology taxonomy covers four patterns. Real systems often use hybrids (e.g., hierarchical orchestration with swarm-like leaf workers). The taxonomy is a starting point, not exhaustive.
- Enterprise hardening items like "signed intent propagation" and "bonding curves with stake slashing" (for swarm agents) are described conceptually — no reference implementations exist yet.
- The governance patterns assume you control the full deployment stack. When using third-party agent platforms, some layers (tracing, policy gates) may be constrained by platform capabilities.
- Verifiability of LLM reasoning remains an open problem. The architecture can audit *what* the agent did, but formally proving *why* it chose a particular plan is not yet tractable.

## Reference

**Paper**: Alenezi, M. (2026). "From Prompt-Response to Goal-Directed Systems: The Evolution of Agentic AI Software Architecture." arXiv:2602.10479v1. [https://arxiv.org/abs/2602.10479v1](https://arxiv.org/abs/2602.10479v1)

**What to look for**: Section 3 for the reference architecture and Algorithm 1 (agent loop pseudocode), Table 1 for the multi-agent failure mode taxonomy, and Table 2 for the complete enterprise hardening checklist with verification methods and evidence requirements.