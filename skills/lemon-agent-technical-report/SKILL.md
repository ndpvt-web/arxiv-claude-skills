---
name: "lemon-agent-technical-report"
description: "Orchestrate multi-agent workflows using the Lemon Agent orchestrator-worker pattern with hierarchical scheduling, progressive context compression, and self-evolving memory. Use when asked to 'break this into parallel subtasks', 'orchestrate agents for a complex task', 'manage context across long-running agent workflows', 'set up an orchestrator-worker pipeline', 'compress agent context efficiently', or 'build a multi-agent system with memory'."
---

# Lemon Agent: Orchestrator-Worker Multi-Agent System

This skill teaches Claude to apply the Lemon Agent architecture -- a two-tier orchestrator-worker system built on the AgentCortex framework -- to decompose complex tasks into parallel subtasks, manage context across concurrent execution streams, and accumulate reusable knowledge from execution traces. The core insight is that task complexity should dynamically determine computational intensity: simple tasks route to a single worker, while complex tasks fan out to parallel specialist workers, each of which can further parallelize their own tool calls. Combined with three-tier progressive context compression and a self-evolving semantic memory, this pattern keeps agent systems fast, context-efficient, and continuously improving.

## When to Use

- When the user asks you to coordinate multiple agents or subtasks working in parallel on a complex problem (e.g., "research these 5 topics simultaneously and synthesize the results")
- When building or designing a multi-agent orchestration system that needs dynamic task routing based on complexity
- When agent workflows are hitting context window limits and need progressive compression strategies
- When the user wants to build a system that learns from past execution traces to improve future task performance
- When designing a planner-executor-memory loop for autonomous agent systems
- When implementing concurrent tool invocation within individual agent workers to maximize throughput
- When the user asks to "set up an agent swarm" or "build an orchestrator that delegates to workers"

## Key Technique

**Hierarchical Self-Adaptive Scheduling.** The Lemon Agent operates at two tiers. At the macro level, the orchestrator evaluates incoming task complexity and decides whether to route to a single worker (minimizing overhead for simple tasks) or fan out to multiple specialist workers executing in parallel (for tasks with orthogonal sub-goals). At the micro level, each worker dynamically adjusts its tool parallelization degree from 1 to 5 concurrent tool calls. Information-gathering tasks (e.g., searching multiple sources) maximize parallelization, while sequential reasoning chains execute tools one at a time to preserve logical coherence. This two-tier approach avoids both the waste of over-parallelizing simple work and the bottleneck of serializing independent subtasks.

**Three-Tier Progressive Context Compression.** Context bloat is the silent killer of long-running agent workflows. Lemon Agent addresses this with three escalating compression stages: (1) **Intra-tool truncation** -- when a single tool's output exceeds a size threshold, truncate it but record metadata anchors (source, type, key fields) so the truncated content can be reconstructed later if needed. (2) **Intra-round adaptive summarization** -- when the cumulative tool outputs within one execution round exceed a threshold, synthesize the entire round's execution trace into a dense summary, prioritizing reconstruction of previously truncated segments. (3) **Cross-round retroactive compression** -- when total history approaches context capacity and truncated tools exist, backtrack through earlier rounds and apply secondary summarization, replacing raw results in-place with compressed versions. This layered strategy preserves critical logical links while aggressively reducing context footprint.

**Self-Evolving Semantic Memory (SES-Memory).** Unlike systems that only learn from successes, SES-Memory extracts reusable knowledge fragments -- code snippets, tool usage patterns, technical insights, critical decision points -- from *all* execution trajectories regardless of outcome. Retrieved memories are filtered by similarity threshold to prevent noise, and when new memories are too similar to existing ones, storage is skipped to prevent redundancy and unbounded growth. This gives the system a continuously improving knowledge base that accelerates similar future tasks.

## Step-by-Step Workflow

1. **Analyze task complexity.** Read the user's request and classify it: Is it a single-focus task (one clear goal, sequential steps) or a multi-faceted task (multiple independent sub-goals that can be pursued in parallel)? Determine the number of orthogonal subtask streams needed (1 for simple, 2-5 for complex).

2. **Retrieve relevant memory.** Before planning, search for any prior execution traces, learned patterns, or stored knowledge relevant to this task type. If you have access to persistent storage or session history, retrieve top-k entries by semantic similarity. Apply a similarity threshold to filter out low-relevance results.

3. **Decompose into subtasks with dependency analysis.** Break the task into concrete subtasks. For each subtask, identify: (a) what it needs to accomplish, (b) what tools it requires, (c) whether it depends on outputs from other subtasks. Independent subtasks are candidates for parallel execution; dependent ones must be sequenced.

4. **Route subtasks to workers.** For each independent subtask stream, spawn a worker (using the Task tool or team coordination). Give each worker a focused prompt containing only the context relevant to its subtask -- not the entire conversation history. This is critical for context efficiency.

5. **Configure per-worker tool parallelization.** Within each worker's instructions, specify the parallelization strategy: information-gathering subtasks (searching, fetching, reading multiple files) should invoke up to 5 tools concurrently. Reasoning-heavy subtasks (code analysis, logical deduction, sequential debugging) should use sequential single-tool execution to preserve reasoning trajectory integrity.

6. **Apply intra-tool truncation as results arrive.** When any tool returns output exceeding ~2000 lines or ~30K characters, truncate the result but record metadata: the tool name, input parameters, output size, and a brief description of what was truncated. This metadata serves as an anchor for later reconstruction.

7. **Apply intra-round summarization when cumulative output is large.** After each worker completes a round of tool calls, if the combined output from that round exceeds a workable threshold, synthesize it into a dense summary. Prioritize preserving: (a) direct answers to the subtask question, (b) key data points and evidence, (c) reconstruction hints for truncated segments.

8. **Apply cross-round retroactive compression when approaching context limits.** If the total conversation history is growing large, go back to earlier rounds and compress their raw results into tighter summaries. Replace verbose tool outputs with their compressed versions in-place. Preserve the logical chain linking rounds together.

9. **Aggregate worker results with confidence scoring.** When all parallel workers complete, collect their results. Assess each result for completeness and confidence. Synthesize a unified answer that reconciles any conflicts between worker outputs, noting areas of uncertainty.

10. **Extract and store reusable knowledge.** After task completion, identify transferable knowledge from the execution: effective tool combinations, useful search strategies, code patterns that worked, pitfalls encountered. Store these as discrete memory entries tagged by task category for future retrieval.

## Concrete Examples

**Example 1: Multi-source research synthesis**

```
User: Research the current state of WebAssembly support across major browsers,
      its performance characteristics vs JavaScript, and its use in production
      applications. Give me a comprehensive summary.

Approach:
1. Classify as multi-faceted (3 independent sub-goals) -> spawn 3 parallel workers
2. Worker A: Search for browser compatibility data (WebAssembly support tables,
   feature coverage across Chrome, Firefox, Safari, Edge)
3. Worker B: Search for performance benchmarks (Wasm vs JS in compute-heavy tasks,
   startup time, memory usage)
4. Worker C: Search for production case studies (companies using Wasm, what they
   built, what results they saw)
5. Each worker uses parallel tool calls (up to 3-5 web searches simultaneously)
6. Apply intra-tool truncation on lengthy benchmark reports, keeping key numbers
7. Aggregate: synthesize all three streams into a structured summary with sections
   for compatibility, performance, and real-world adoption

Output:
## WebAssembly in 2026: State of the Art

### Browser Support
[Synthesized from Worker A: specific version numbers, feature gaps, notable
limitations in Safari's implementation...]

### Performance vs JavaScript
[Synthesized from Worker B: 2-5x faster for compute-heavy tasks, comparable
startup after streaming compilation, specific benchmark numbers...]

### Production Adoption
[Synthesized from Worker C: Figma, Google Earth, Adobe Photoshop web,
specific metrics on load time improvements...]
```

**Example 2: Complex codebase debugging with context management**

```
User: Our API is returning 500 errors intermittently. The error logs mention
      a database connection timeout but we're not sure where it originates.
      The codebase has 200+ files across services/api, services/db, and lib/.

Approach:
1. Classify as complex (multiple investigation angles) -> spawn 2-3 workers
2. Worker A: Search for database connection handling code (pool config,
   connection lifecycle, timeout settings) -- parallel grep for "connection",
   "pool", "timeout" across services/db/ and lib/
3. Worker B: Search for error handling paths (where 500s are generated,
   how DB errors propagate) -- parallel grep for "500", "InternalServerError",
   "catch" across services/api/
4. Worker C: Search for configuration (env vars, connection strings, pool
   size settings) -- parallel read of config files, .env templates, docker-compose
5. Apply truncation: Worker A finds 40+ files mentioning "connection" --
   truncate to files with timeout-related logic, record metadata for others
6. Intra-round summarization: Worker B's error handler search yields verbose
   results -- summarize to "3 error paths lead to 500: middleware catch-all
   at api/middleware/error.ts:45, route handler in api/routes/users.ts:112,
   and DB wrapper at lib/db.ts:78"
7. Aggregate: Cross-reference connection pool config (Worker C) with timeout
   handling (Worker A) and error propagation (Worker B) to identify root cause

Output:
Root cause: The connection pool in lib/db.ts is configured with a 5-second
acquire timeout (line 23) but the health check query in services/db/health.ts
holds connections for up to 10 seconds under load (line 67). When the pool is
exhausted, new requests timeout and the error at lib/db.ts:78 throws a generic
error that gets caught by the middleware catch-all and returned as 500.

Fix: Either increase pool size, reduce health check hold time, or add a
dedicated pool for health checks.
```

**Example 3: Building a multi-agent pipeline with memory**

```
User: Help me design and implement an agent system that processes customer
      support tickets -- classifying them, routing to specialists, and
      learning from resolved tickets.

Approach:
1. Single orchestrator task but with clear architectural sub-components
2. Design the orchestrator-worker hierarchy:
   - Orchestrator: receives tickets, classifies complexity, routes
   - Worker pool: billing-specialist, technical-specialist, general-specialist
3. Implement adaptive scheduling:
   - Simple tickets (password reset, FAQ) -> single general worker
   - Complex tickets (multi-issue, escalation needed) -> parallel specialists
4. Implement context management:
   - Tier 1: Truncate long ticket histories to last 10 messages + summary
   - Tier 2: Summarize each specialist's investigation round
   - Tier 3: Compress resolved ticket context before archiving to memory
5. Implement SES-Memory:
   - After each ticket resolution, extract: resolution pattern, tools used,
     customer sentiment indicators, escalation triggers
   - Store as tagged memory entries (billing, technical, account, etc.)
   - On new tickets, retrieve top-5 similar past resolutions
   - Filter by similarity > 0.7 to prevent noise

Output:
[Architectural diagram + implementation code for the orchestrator routing
logic, worker specialization prompts, context compression functions,
and memory storage/retrieval module]
```

## Best Practices

- **Do:** Classify task complexity before deciding on parallelization. Single-worker execution for simple tasks avoids coordination overhead that slows things down.
- **Do:** Give each parallel worker only the context it needs for its specific subtask. Shipping the entire conversation history to every worker wastes context and confuses focus.
- **Do:** Record metadata when truncating tool outputs. The source URL, tool name, and content type act as anchors for later reconstruction when a detail turns out to be important.
- **Do:** Extract knowledge from failed executions too. A tool combination that didn't work, a search query that returned noise, or a reasoning path that hit a dead end are all valuable signals for future tasks.
- **Avoid:** Parallelizing subtasks that have data dependencies. If Worker B needs Worker A's output, running them in parallel wastes compute and produces incorrect results.
- **Avoid:** Setting tool parallelization to maximum for reasoning-heavy tasks. Sequential tool execution preserves the chain-of-thought integrity needed for logical deduction and debugging.
- **Avoid:** Storing every execution trace in memory without similarity deduplication. Unbounded memory growth introduces noise and increases retrieval latency. Always check for redundancy before storing.

## Error Handling

- **Worker failure or timeout:** If a parallel worker fails or hangs, the orchestrator should collect results from successful workers and either retry the failed subtask with a simplified prompt or report partial results with a clear indication of what's missing.
- **Context overflow despite compression:** If all three compression tiers are applied and context still exceeds limits, prioritize the most recent round's results and the task's core question. Drop intermediate exploration rounds first, keeping only their summaries.
- **Memory retrieval returns irrelevant results:** If retrieved memories have low relevance scores (below threshold), discard them entirely rather than injecting noise into the current task's context. It's better to work without memory than with misleading memory.
- **Conflicting worker results:** When parallel workers return contradictory findings, don't silently pick one. Flag the conflict explicitly, present both findings with their evidence, and either resolve through a follow-up investigation round or present the conflict to the user for judgment.
- **Tool parallelization causing rate limits:** If concurrent tool calls hit API rate limits or resource constraints, automatically fall back to sequential execution with exponential backoff rather than failing the entire subtask.

## Limitations

- **Not suitable for purely sequential reasoning tasks.** If a task is fundamentally a single chain of dependent steps (e.g., step-by-step mathematical proof), the orchestrator-worker pattern adds overhead without benefit. Use a single-agent sequential approach instead.
- **Memory system requires persistence.** The self-evolving memory only helps if execution traces can be stored and retrieved across sessions. In stateless environments without persistent storage, the memory tier provides no value.
- **Parallelization gains diminish with coordination costs.** For tasks with fewer than 3 independent sub-goals, the overhead of spawning workers, splitting context, and aggregating results can exceed the time saved by parallel execution.
- **Context compression is lossy.** Each compression tier discards information. For tasks requiring exact reproduction of tool outputs (e.g., precise data extraction), aggressive compression may lose critical details. Use higher truncation thresholds in these cases.
- **Depends on accurate complexity classification.** If the orchestrator misclassifies a complex task as simple (or vice versa), the system either under-resources the task or wastes compute. The classification step is a single point of failure that benefits from explicit user guidance.

## Reference

- **Paper:** [Lemon Agent Technical Report](https://arxiv.org/abs/2602.07092v1) (Jiang et al., 2026)
- **Key sections to study:** Section on hierarchical self-adaptive scheduling (macro/micro level parallelization decisions), the three-tier progressive context management thresholds (intra-tool, intra-round, cross-round), and Algorithm 1 which formalizes the full execution loop from memory retrieval through task routing, parallel execution, compression, and memory storage.