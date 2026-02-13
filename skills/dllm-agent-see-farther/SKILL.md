---
name: "dllm-agent-see-farther"
description: "Design and implement multi-agent workflows using the DeepDiver hierarchical orchestration pattern with diffusion-inspired parallel planning. Applies DLLM Agent principles -- global planning signals, reduced backtracking, span-aware execution, and structured tool-call hardening -- to build agent pipelines that converge faster on correct action paths. Use when: 'build an agent pipeline with planner and workers', 'reduce backtracking in my agent loop', 'design a hierarchical agent workflow', 'optimize multi-step tool-use agent', 'implement DeepDiver-style agent orchestration', 'harden tool calls in my agent system'."
---

# DLLM Agent: Hierarchical Agent Orchestration with Global Planning

This skill teaches Claude to design and implement multi-agent systems using the DeepDiver hierarchical orchestration pattern described in "DLLM Agent: See Farther, Run Faster" (Zhen et al., 2026). The core insight is that agent workflows converge faster when the planner reasons globally over the full action space before committing to sequential steps -- analogous to how diffusion models refine all tokens simultaneously rather than generating left-to-right. You will apply this principle to build agent pipelines with a Planner/Seeker/Writer hierarchy, structured tool-call validation, span-aware context management, and early-convergence planning strategies that reduce interaction rounds by 10-30%.

## When to Use

- When the user asks to build a multi-step agent pipeline that coordinates planning, retrieval, and synthesis (e.g., a research agent, data analysis pipeline, or automated report generator)
- When an existing agent loop suffers from excessive backtracking, redundant tool calls, or late convergence on the correct action path
- When the user wants to implement a Planner/Worker agent hierarchy where a central planner decomposes goals and dispatches specialized sub-agents
- When designing tool-calling agents that need robust schema validation to prevent structured output failures
- When building multi-turn agent workflows where context and action spans must be cleanly separated to avoid information leakage between reasoning phases
- When optimizing an agent system for fewer interaction rounds while maintaining accuracy

## Key Technique

The DLLM Agent paper demonstrates that swapping an autoregressive backbone for a diffusion backbone inside the same agent framework produces agents that plan more globally: they identify viable paths with fewer exploratory detours, require fewer tool invocations (8.0 vs 10.4 per query), and use fewer interaction rounds (13.0 vs 14.8). The mechanism is that diffusion-style generation refines the entire action span in parallel, producing stronger "global planning signals" -- the planner establishes critical decisions early through parallel refinement before committing locally.

You do not need a diffusion LLM to benefit from this. The actionable insight is architectural: structure your agent workflow so the planner performs **batch constraint extraction and upfront goal decomposition** before dispatching any tool calls, rather than interleaving planning and execution token-by-token. This means the planner should output a complete sub-goal plan for the current iteration, the information seekers should execute tool calls against that plan, and the writer should synthesize results -- all within a clean think-act-observe loop. Each role has a distinct context window with explicit boundaries.

The paper also identifies two critical failure modes. First, diffusion-style (parallel) generation is 3x more likely to produce malformed tool calls (6.4% vs 1.9% invalid rate), so any parallel or batch planning approach must include **structural validation of tool-call schemas before execution**. Second, when multi-turn history mixes context and action spans, the model can leak information across boundaries; the fix is **span-aware attention masking** -- in practice, this means cleanly separating observation history from action generation in your prompt architecture.

## Step-by-Step Workflow

1. **Define the agent hierarchy.** Establish three roles: a **Planner** that decomposes the global objective into sub-goals, one or more **Information Seekers** that execute tool calls (search, database queries, API calls, file operations), and a **Writer** that synthesizes retrieved evidence into structured output. Each role gets its own system prompt and tool permissions.

2. **Implement the think-act-observe loop.** Structure each iteration as: (a) Planner receives full history and reasons about what sub-goals remain, (b) Planner emits a structured plan with 1-3 concrete sub-goals for this iteration, (c) Information Seekers execute the tool calls specified by the plan, (d) observations are appended to shared history, (e) loop repeats until Planner emits a Terminate action.

3. **Design the Planner for batch constraint extraction.** Instead of letting the planner reason one step at a time, prompt it to analyze ALL constraints from the user query upfront and produce a ranked decomposition. Use a structured output format:
   ```json
   {
     "constraints_extracted": ["constraint1", "constraint2"],
     "sub_goals": [
       {"id": 1, "goal": "...", "tools_needed": ["search"], "depends_on": []},
       {"id": 2, "goal": "...", "tools_needed": ["db_query"], "depends_on": [1]}
     ],
     "estimated_rounds": 3
   }
   ```

4. **Add structured tool-call validation.** Before executing any tool call emitted by an agent, validate it against the tool's JSON schema. Check: (a) the tool name exists, (b) all required parameters are present, (c) parameter types match, (d) enum values are valid. On validation failure, return a structured error to the agent and request a corrected call -- do not silently fail or retry blindly.

5. **Implement span-aware context management.** In the prompt fed to each agent, clearly delimit observation history (read-only context) from the action generation region. Use explicit markers like `<context>...</context>` and `<action>...</action>`. When building multi-turn prompts, never interleave previous action attempts with new context -- keep all context tokens contiguous and all action tokens contiguous.

6. **Track planner hit rate and convergence.** Instrument your loop to record: (a) number of interaction rounds, (b) number of tool calls per round, (c) number of invalid/failed tool calls, (d) whether the planner revised a previous sub-goal (backtrack). Use these metrics to detect convergence problems early.

7. **Set a maximum round budget with early termination.** Cap the loop at T_max rounds (the paper uses 15). Implement early termination when the Planner determines all sub-goals are satisfied. The Planner should explicitly output either `{"action": "ToolCall", ...}` or `{"action": "Terminate", "reason": "..."}`.

8. **Reduce redundancy via observation deduplication.** Before each planning round, deduplicate observations that convey the same information from different tool calls. Summarize long observations into key findings. This reduces context window consumption and prevents the planner from re-exploring already-resolved sub-goals.

9. **Handle the writer synthesis phase.** Once the Planner terminates, pass all accumulated observations to the Writer agent with a structured prompt specifying the output format. The Writer should cite which observations support each claim, enabling traceability.

10. **Test with ablations.** Validate your pipeline by measuring: (a) accuracy with vs. without batch planning (step 3), (b) invalid tool-call rate with vs. without schema validation (step 4), (c) round count with vs. without span-aware context (step 5). Each should independently improve performance.

## Concrete Examples

**Example 1: Research Question Answering Agent**

User: "Build me an agent that can answer complex research questions by searching the web and synthesizing findings."

Approach:
1. Define three agents: `ResearchPlanner`, `WebSeeker`, `ReportWriter`
2. ResearchPlanner receives the question and extracts constraints:
   ```json
   {
     "query": "What are the environmental impacts of lithium mining in Chile?",
     "constraints_extracted": ["geographic: Chile", "topic: lithium mining", "aspect: environmental impacts"],
     "sub_goals": [
       {"id": 1, "goal": "Find recent data on lithium extraction volumes in Chile", "tools_needed": ["web_search"]},
       {"id": 2, "goal": "Find environmental impact assessments for Chilean lithium operations", "tools_needed": ["web_search"]},
       {"id": 3, "goal": "Find counterarguments or mitigation efforts", "tools_needed": ["web_search"], "depends_on": [1, 2]}
     ]
   }
   ```
3. WebSeeker executes searches for sub-goals 1 and 2 in parallel (no dependency), then sub-goal 3
4. Each observation is validated and deduplicated before the next planning round
5. Planner checks: all sub-goals resolved? If yes, Terminate. If a search returned no useful results, Planner revises that sub-goal with alternative search terms (one backtrack allowed)
6. ReportWriter synthesizes findings into a structured answer with citations

Output: A 3-round pipeline (vs. typical 5-7 rounds in naive sequential agents) producing a cited research summary.

**Example 2: Database Investigation Agent**

User: "Create an agent that investigates anomalies in a SQL database by querying tables and cross-referencing results."

Approach:
1. Define `InvestigationPlanner`, `DBSeeker`, `AnalysisWriter`
2. Planner extracts the anomaly description and decomposes:
   ```json
   {
     "constraints_extracted": ["anomaly: revenue spike on 2025-03-15", "tables: orders, payments, refunds"],
     "sub_goals": [
       {"id": 1, "goal": "Query orders table for 2025-03-15 volume and amounts", "tools_needed": ["sql_query"]},
       {"id": 2, "goal": "Query payments table for same date", "tools_needed": ["sql_query"]},
       {"id": 3, "goal": "Cross-reference order IDs between tables to find mismatches", "tools_needed": ["sql_query"], "depends_on": [1, 2]}
     ]
   }
   ```
3. Tool-call validation catches a malformed SQL query before execution:
   ```
   VALIDATION FAILURE: sql_query parameter "query" contains unmatched parenthesis.
   Requesting corrected tool call from DBSeeker.
   ```
4. DBSeeker corrects and re-submits; observations are appended with span markers
5. Planner sees cross-reference results, identifies a batch of duplicate payment records, adds one more sub-goal to quantify the duplicates
6. AnalysisWriter produces a root-cause analysis report

Output: 4-round investigation with 6 validated SQL queries, catching 1 malformed query before execution.

**Example 3: Hardening an Existing Agent Loop**

User: "My agent pipeline keeps backtracking and taking 12+ rounds to answer simple questions. Help me optimize it."

Approach:
1. Audit the existing loop: instrument it to log round count, tool calls per round, backtrack events, and invalid call rate
2. Identify the root cause using DLLM Agent diagnostics:
   - If backtrack rate > 20%: the planner is not extracting constraints upfront. Refactor to batch constraint extraction (step 3 of workflow)
   - If invalid tool-call rate > 5%: add schema validation layer (step 4)
   - If round count is high but backtrack rate is low: observations are bloated. Add deduplication (step 8)
3. Refactor the planner prompt to output structured sub-goal plans instead of single next-actions
4. Add explicit `<context>` / `<action>` span markers to prevent context-action information leakage
5. Re-measure: expect 30%+ reduction in rounds for equivalent accuracy

Output: Refactored pipeline dropping from 12 rounds to 7-8 rounds with same accuracy, plus a monitoring dashboard tracking the four key metrics.

## Best Practices

- **Do:** Have the Planner output a complete sub-goal decomposition before any tool calls execute. This is the single highest-impact change -- it mimics the "global planning signal" that gives diffusion agents their advantage.
- **Do:** Validate every tool call against its schema before execution. The 3x higher invalid rate in parallel planning is real and will bite you in production.
- **Do:** Keep context spans and action spans physically separated in your prompts. Interleaving them degrades planning quality measurably (~1% accuracy per the paper).
- **Do:** Track planner hit rate (fraction of sub-goals that succeed on first attempt) as your primary efficiency metric. Target > 80%.
- **Avoid:** Letting the planner interleave reasoning and tool execution in an unbounded stream. This leads to token-level commitment and excessive backtracking.
- **Avoid:** Retrying failed tool calls without returning the structured error to the agent. The agent needs the error signal to correct its reasoning, not just a silent retry.
- **Avoid:** Using a single flat context window for all agents. The Planner, Seekers, and Writer should each see only the context they need, with explicit role boundaries.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Malformed tool call (invalid JSON, missing params) | Schema validation before execution | Return structured error to agent; request corrected call. Cap retries at 2. |
| Planner produces circular sub-goals | Dependency graph cycle detection | Flatten dependencies; force sequential execution of the cycle. |
| Seeker returns empty/irrelevant observations | Relevance check against sub-goal | Planner revises search terms or marks sub-goal as unresolvable. |
| Context window overflow in multi-turn loop | Token count tracking per round | Summarize oldest observations; drop raw tool outputs, keep key findings only. |
| Planner fails to Terminate (infinite loop) | Round counter exceeds T_max | Force Terminate; Writer synthesizes from whatever observations exist. |
| Information leakage across context/action spans | Attention analysis or ablation test | Re-check prompt structure; ensure `<context>` and `<action>` markers are present and contiguous. |

## Limitations

- **Not a diffusion model replacement.** This skill applies the *architectural principles* from DLLM Agents (global planning, span separation, tool-call hardening) to standard autoregressive agent workflows. You will not get the full 30%+ speedup without an actual diffusion backbone -- expect 10-30% round reduction from architectural changes alone.
- **Batch planning has a cold-start cost.** The upfront constraint extraction adds latency to the first round. For single-step tasks (one tool call, one answer), the overhead is not worth it. Use this pattern only for tasks requiring 3+ interaction rounds.
- **Schema validation requires known tool schemas.** If tools have dynamic or undocumented schemas, the validation layer cannot catch all malformed calls. You need explicit JSON schemas for each tool.
- **Sub-goal decomposition quality depends on the planner's domain knowledge.** For highly specialized domains, the planner may produce poor decompositions. Mitigate by providing domain-specific examples in the planner's system prompt.
- **The paper's benchmarks are on information-retrieval tasks (BrowseComp-zh).** The round-reduction gains may differ for other task types (code generation, mathematical reasoning, creative tasks).

## Reference

**Paper:** "DLLM Agent: See Farther, Run Faster" -- Zhen, Lin, Liu, Han, Li (2026). [arXiv:2602.07451v2](https://arxiv.org/abs/2602.07451v2)

**What to look for:** Section 3 for the DeepDiver workflow and agent-oriented fine-tuning setup; Section 4.2 for the span-aware attention masking technique (context-clean corruption + span-aware attention); Table 2 for benchmark comparisons showing round and tool-call reduction; Section 5 for attention dynamics analysis showing global vs. local planning patterns.