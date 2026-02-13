---
name: "loca-bench-benchmarking-agents-under"
description: "Apply context management strategies from LOCA-bench to prevent context rot in long-running agent tasks. Implements programmatic tool calling, tool-result clearing, thinking-block clearing, context awareness, and memory tools to maintain agent accuracy as context grows. Use when: 'my agent loses track after many steps', 'context window filling up', 'agent stops exploring midway', 'handle long context in agentic workflow', 'prevent context rot', 'manage growing conversation context'."
---

# LOCA-bench: Context Management for Long-Running Agent Tasks

This skill teaches Claude to apply the context management strategies evaluated in the LOCA-bench benchmark to real agentic coding tasks. LOCA-bench systematically measured how agent accuracy degrades as context grows (a phenomenon called "context rot" -- e.g., Claude Opus dropping from 96% to 15% accuracy as environment complexity scales from 8K to 256K tokens) and identified which mitigation strategies actually work. The single most effective technique across all models is **programmatic tool calling** -- orchestrating tools via code execution rather than sequential individual calls -- which consistently doubles or nearly doubles success rates. This skill applies these findings to help you build, debug, and refactor agent systems that stay reliable under growing context.

## When to Use This Skill

- When building an agent workflow that involves many sequential tool calls and the agent starts "forgetting" earlier results or instructions
- When an agent prematurely concludes a task (e.g., checking only the first page of results and declaring "no matches found")
- When designing a scaffold or orchestration layer for LLM agents that must handle 32K+ tokens of accumulated context
- When an agent hallucinates or distorts values it correctly retrieved earlier in the conversation
- When refactoring an existing ReAct-style agent loop to improve reliability on long tasks
- When the user asks how to prevent context window overflow in multi-step tool-use workflows
- When evaluating or comparing context management strategies for an agent system

## Key Technique: Combating Context Rot in Agents

**Context rot** is the systematic degradation of agent performance as conversation history grows. LOCA-bench identified four specific failure modes: (1) declining complex reasoning -- agents fail multi-step retrieval across tools; (2) weaker instruction following -- agents miss explicit constraints, especially format/schema requirements; (3) insufficient exploration -- agents become conservative, stop exploring early, and mistake partial evidence for complete results; (4) hallucination-like inconsistencies -- agents distort values during reasoning despite correct initial retrieval.

The benchmark tested six context management strategies across frontier models. **Programmatic tool calling** (PTC) was the clear winner: it improved accuracy by 18-120% across all tested models while simultaneously *reducing* trajectory length. PTC works by having the agent write and execute code that orchestrates multiple tool calls in a single step, rather than issuing tools one at a time. This compresses what would be dozens of individual call/response pairs into a single code block and its output, dramatically reducing context consumption. Other strategies ranked: context awareness (giving the agent real-time token budget feedback) helps frontier models but can hurt weaker ones; memory tools (persistent key-value storage) provide moderate gains; tool-result clearing (removing 50% of old tool outputs at a threshold) and thinking-block clearing provide marginal improvements; context compaction (summarizing history) can create runaway trajectory lengths if used alone.

The critical insight is that **context management is not just about fitting within a window -- it changes agent behavior**. Agents with context awareness become more urgency-driven and explore sooner. Agents using PTC naturally batch related operations, reducing the noise-to-signal ratio in their history. The best approach combines PTC as the primary strategy with context awareness as a secondary signal.

## Step-by-Step Workflow

1. **Audit the agent's context growth pattern.** Before applying any strategy, instrument the agent loop to log cumulative token counts after each step. Identify where context grows fastest -- typically tool outputs (API responses, database results, file contents) dominate over agent reasoning tokens.

2. **Implement programmatic tool calling as the default execution mode.** Refactor the agent's tool-use layer so the agent writes code (Python or JavaScript) that calls multiple tools in sequence or parallel, processes intermediate results in code, and returns only the final relevant output. For example, instead of 10 separate `search_database` calls each returning full result sets, write one code block that queries, filters, and aggregates.

   ```python
   # BEFORE: 10 sequential tool calls, each adding ~2K tokens to context
   # result1 = search(query="...") -> 2K tokens
   # result2 = search(query="...") -> 2K tokens
   # ... (20K tokens consumed)

   # AFTER: Single PTC block
   code = """
   results = []
   for query in queries:
       batch = search(query=query)
       results.extend([r for r in batch if r['status'] == 'active'])
   return {'matching_count': len(results), 'items': results[:10]}
   """
   # Only the final filtered output enters context (~500 tokens)
   ```

3. **Add context awareness signals.** After each tool invocation, inject a brief system message reporting remaining context capacity (e.g., `[Context: 45K/200K tokens used, 78% remaining]`). This nudges the agent to prioritize and compress when capacity is low, without requiring explicit truncation logic.

4. **Implement tiered tool-result clearing.** Set a context threshold (e.g., 60% of max capacity). When exceeded, remove the oldest 50% of tool call/response pairs from conversation history, keeping the most recent interactions intact. Preserve any tool results the agent explicitly referenced in its reasoning.

5. **Add a persistent memory/scratchpad layer.** Provide the agent with `memory_write(key, value)` and `memory_read(key)` operations backed by a key-value store outside the conversation. Instruct the agent to save critical intermediate findings (IDs, computed values, confirmed facts) immediately upon discovery, so they survive context clearing.

6. **Guard against premature termination.** Add an exploration completeness check: before the agent issues a final answer, verify it has explored the full scope of the task. For paginated results, check if all pages were fetched. For multi-source tasks, confirm all required sources were consulted. Inject a prompt like: "Before concluding, verify: have you checked all data sources mentioned in the task? Are there remaining pages or items you haven't examined?"

7. **Structure agent output to separate reasoning from data.** Use a format where the agent's plan/reasoning is in one clearly marked section and raw data/evidence is in another. This makes it possible to selectively clear data sections while preserving the reasoning chain.

8. **Test at multiple context scales.** Validate your agent at 8K, 32K, 64K, and 128K+ token workloads. Performance that looks fine at 8K can collapse at 64K. Use LOCA-bench's methodology: keep the task fixed but scale the environment complexity (more records, more noise, more edge cases).

9. **Combine strategies deliberately.** The recommended stack: PTC as the primary execution mode + context awareness for real-time feedback + memory tools for critical fact persistence + tool-result clearing as a safety net at high utilization. Do not rely on a single strategy alone.

## Concrete Examples

**Example 1: Refactoring a data-processing agent that loses track of findings**

User: "My agent queries a WooCommerce store to find products below a restock threshold, but when there are hundreds of products it only checks the first batch and gives up."

Approach:
1. Identify that the agent is making individual `list_products(page=1)` calls and running out of context budget before completing pagination
2. Refactor to programmatic tool calling -- write a single code block that paginates through all products:
   ```python
   all_products = []
   page = 1
   while True:
       batch = list_products(page=page, per_page=100)
       if not batch:
           break
       all_products.extend(batch)
       page += 1
   low_stock = [p for p in all_products if p['stock_quantity'] < p['restock_threshold']]
   return {'low_stock_count': len(low_stock), 'products': low_stock}
   ```
3. Add memory persistence: after the code block runs, store `memory_write("low_stock_products", low_stock)` so results survive any future context clearing
4. Add context awareness to alert the agent if it's approaching limits during the subsequent notification/email-sending phase

Output: The agent now processes all 500+ products in a single context step (~1K tokens) instead of 6 sequential paginated calls (~12K tokens), and retains the filtered results in persistent memory.

**Example 2: Building a multi-source research agent with context budget**

User: "I need an agent that checks Canvas LMS for upcoming exams, cross-references with Google Calendar for conflicts, and sends notification emails. It breaks down when there are many courses."

Approach:
1. Implement PTC for each data-gathering phase:
   ```python
   # Phase 1: Gather all exams across courses in one code block
   exams = []
   for course_id in course_ids:
       assignments = get_assignments(course_id, type='exam')
       exams.extend([{'course': course_id, 'date': a['due_at'], 'name': a['name']}
                     for a in assignments])
   return exams
   ```
2. Store intermediate results in memory: `memory_write("all_exams", exams)`
3. Phase 2 code block reads from memory and checks calendar:
   ```python
   exams = memory_read("all_exams")
   conflicts = []
   for exam in exams:
       events = get_calendar_events(date=exam['date'])
       if any(e['type'] != 'exam' for e in events):
           conflicts.append({**exam, 'conflicting_events': events})
   return conflicts
   ```
4. Add context awareness after each phase. If context usage exceeds 60%, trigger tool-result clearing before the email-sending phase
5. Add exploration guard: verify exam count matches expected course count before concluding

Output: Agent handles 50+ courses reliably by compressing each phase into a single code execution, persisting critical data between phases, and clearing stale tool outputs when context pressure builds.

**Example 3: Debugging an agent that halluccinates retrieved values**

User: "My agent correctly retrieves invoice amounts from BigQuery but then uses wrong numbers when composing the final summary."

Approach:
1. Diagnose: this is the "hallucination-like inconsistency" failure mode from LOCA-bench -- the agent retrieved correct values but distorted them during subsequent reasoning steps because the original retrieval is now far back in context
2. Add memory persistence at the retrieval point:
   ```python
   invoices = query_bigquery("SELECT id, amount, vendor FROM invoices WHERE status='pending'")
   memory_write("pending_invoices", invoices)
   return invoices
   ```
3. In the summary-generation step, instruct the agent to re-read from memory rather than relying on conversation history: `invoices = memory_read("pending_invoices")`
4. Add a verification step: compare final summary values against the memory store before outputting
5. If context is very long, apply tool-result clearing to remove the original verbose BigQuery output (now safely persisted in memory)

Output: Agent composes accurate summaries because it reads canonical values from persistent memory rather than depending on values surviving through a long reasoning chain.

## Best Practices

- **Do:** Default to programmatic tool calling for any task involving 3+ sequential tool calls on the same data source. It is the single highest-impact intervention.
- **Do:** Persist critical intermediate results (IDs, computed values, confirmed facts) to memory immediately upon retrieval -- do not rely on them surviving in conversation history.
- **Do:** Inject context awareness signals so the agent can self-regulate its exploration depth and verbosity as context fills.
- **Do:** Test your agent at 2-4x the expected context length to catch degradation before production.
- **Avoid:** Using context compaction (full history summarization) as the sole strategy -- it tends to create runaway trajectory lengths and loses critical details.
- **Avoid:** Applying the Claude Agent SDK's built-in subagent/semantic-search features on unfamiliar environments without testing -- LOCA-bench found this *decreased* performance vs. plain ReAct because the agent over-relies on framework features it can't effectively control.
- **Avoid:** Assuming that a larger context window solves context rot -- the benchmark shows degradation is behavioral (exploration patterns, reasoning quality), not just a capacity issue.

## Error Handling

- **Agent stops exploring prematurely:** Add explicit exploration completeness prompts and pagination checks. Verify the agent has touched all required data sources before accepting a final answer.
- **Context limit exceeded mid-task:** Implement graceful degradation: trigger tool-result clearing at 70% capacity, fall back to memory-only operation at 90%, and surface a clear warning if the agent must truncate its approach.
- **Memory store conflicts:** When multiple code blocks write to the same key, use append semantics or versioned keys (`invoices_phase1`, `invoices_phase2`) to avoid silent overwrites.
- **PTC code execution fails:** Fall back to sequential tool calling for that specific operation. Log the failure for debugging. Common causes: tool API changes, unexpected response formats, or sandbox restrictions.
- **Values drift between retrieval and use:** Always re-read from persistent memory at the point of use rather than trusting values carried through reasoning. Add checksums or counts as sanity checks.

## Limitations

- Programmatic tool calling requires a code execution sandbox, which may not be available in all deployment environments.
- Memory tools add latency per read/write operation and introduce a new failure surface (store unavailability, key collisions).
- Context awareness signals consume tokens themselves -- at very high context pressure, the overhead of awareness messages may be counterproductive.
- These strategies mitigate but do not eliminate context rot. At extreme scales (256K+ tokens), even the best strategies show significant accuracy loss compared to short-context baselines.
- The LOCA-bench findings are measured on specific task types (database queries, API orchestration, scheduling). Performance gains may vary for fundamentally different agent workloads (e.g., long-form writing, code generation).
- Tool-result clearing is lossy -- if the agent needs to revisit old tool outputs that were cleared and not persisted to memory, it must re-execute the tool calls.

## Reference

**Paper:** [LOCA-bench: Benchmarking Language Agents Under Controllable and Extreme Context Growth](https://arxiv.org/abs/2602.07962v1) (Zeng, Huang, He, 2026). Look for: Table 3 (strategy comparison at 128K tokens), Section 5 (four failure modes of context rot), and Section 4.2 (programmatic tool calling implementation details). **Code:** [github.com/hkust-nlp/LOCA-bench](https://github.com/hkust-nlp/LOCA-bench)