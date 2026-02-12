---
name: "adareasoner-dynamic-tool-orchestration"
description: "Dynamically orchestrate multi-step tool chains for complex reasoning tasks. Applies AdaReasoner's adaptive tool selection: evaluate tool utility from context and intermediate results, suppress irrelevant tools, and modulate invocation frequency. Use when asked to 'orchestrate tools for a multi-step task', 'build an adaptive tool pipeline', 'chain tools dynamically based on results', 'create a reasoning agent that picks its own tools', 'design a self-correcting tool workflow', or 'implement dynamic tool selection with reinforcement signals'."
---

# AdaReasoner: Dynamic Tool Orchestration for Iterative Reasoning

This skill teaches Claude to design and implement **adaptive multi-step tool orchestration systems** based on AdaReasoner's three core principles: (1) infer tool utility from task context and intermediate outcomes rather than hardcoding tool sequences, (2) dynamically regulate tool usage frequency based on task demands, and (3) generalize to unseen tools by learning tool-use as a reasoning skill decoupled from specific tool interfaces. The result is agent architectures that autonomously adopt beneficial tools, suppress irrelevant ones, and self-correct through reflection and backtracking.

## When to Use

- When the user needs to build an **agent that selects and sequences tools dynamically** rather than following a fixed pipeline (e.g., "build an agent that figures out which APIs to call based on what it learns at each step")
- When designing a **multi-step reasoning system** where the output of one tool informs whether and how to invoke the next (e.g., "create a pipeline that crops an image region, then OCRs only if text is detected")
- When the user wants **adaptive tool frequency** -- some tasks need many tool calls, others need few, and the system should decide autonomously
- When building **self-correcting workflows** where tool failures or unhelpful results trigger backtracking and alternative strategies
- When implementing **tool-agnostic orchestration** that can incorporate new tools at runtime without retraining or reconfiguring the routing logic
- When the user asks to **replace a rigid LangChain/LangGraph pipeline** with something that adapts tool selection based on reward signals or outcome quality

## Key Technique

AdaReasoner treats tool use as a **learned reasoning skill** rather than a hardcoded routing table. Traditional tool-augmented agents either use a fixed tool chain (always call Search then Summarize then Answer) or rely on a single LLM call to pick tools from a list. AdaReasoner instead trains a policy that optimizes the *entire trajectory* of tool calls -- which tool, in what order, how many times, and when to stop -- based on whether the final answer is correct. The key insight: **tool selection should be optimized end-to-end on task success, not on per-step heuristics.**

The method has three pillars. **First**, a data curation pipeline generates training trajectories that include long-horizon tool chains, reflection/backtracking scenarios, and explicit tool-failure cases where the model must fall back on intrinsic reasoning. This teaches the model that tools are optional aids, not mandatory steps. **Second**, Tool-GRPO (Group Relative Policy Optimization) assigns composite rewards: a format reward ensuring well-structured tool calls, a hierarchical tool reward scoring call quality (structure, name, parameters, content), and an accuracy reward for final correctness. Critically, correct answers get full reward *regardless* of tool usage, while incorrect answers get partial credit for informative tool reasoning -- this asymmetry prevents tool overuse. **Third**, an adaptive learning mechanism randomizes tool names and paraphrases tool descriptions during training, forcing the model to infer tool utility from functional descriptions rather than memorized names.

In practice, this produces emergent behaviors: the system learns to invoke pathfinding tools heavily for navigation tasks but suppress them for verification tasks; it learns to call OCR only after cropping relevant regions; and it adjusts total tool calls from 1 to 5+ depending on task complexity -- all without explicit rules.

## Step-by-Step Workflow

1. **Decompose the task into a perception-reasoning-action loop.** Identify the high-level goal, then classify what the system needs at each potential step: information gathering (search, OCR, API calls), transformation (crop, filter, compute), or verification (check, validate, compare). Do not fix the order yet -- just enumerate the *capabilities* needed.

2. **Define tools with function-level descriptions, not just names.** For each tool, write a description that specifies: what it accepts, what it returns, when it is useful, and when it is NOT useful. This mirrors AdaReasoner's semantic-level tool definitions that enable generalization. Avoid coupling logic to tool names.

   ```python
   tools = {
       "extract_text": {
           "description": "Extract text from an image region. Useful when text content is needed for answering. Not useful when the answer is purely visual.",
           "parameters": {"image": "PIL.Image", "region": "Optional[BBox]"},
           "returns": "str"
       }
   }
   ```

3. **Implement a trajectory-based orchestration loop.** Instead of a single tool-selection call, build an iterative loop where each step: (a) presents the current state (original input + all prior tool outputs), (b) asks the reasoning model to decide the next action (call a tool, reflect, or produce final answer), and (c) appends the result to the trajectory.

   ```python
   trajectory = [{"role": "user", "content": task}]
   for step in range(max_steps):
       action = reason(trajectory, available_tools)
       if action.type == "final_answer":
           break
       elif action.type == "tool_call":
           result = execute_tool(action.tool, action.params)
           trajectory.append({"role": "tool", "name": action.tool, "result": result})
       elif action.type == "reflect":
           trajectory.append({"role": "assistant", "content": action.reflection})
   ```

4. **Build in reflection and backtracking.** After each tool result, prompt the model to evaluate whether the result was useful. If not, explicitly allow it to (a) try a different tool, (b) retry with different parameters, or (c) abandon tool use and reason directly. Include this in your system prompt: *"If a tool result is unhelpful or a tool call fails, reflect on why and choose an alternative strategy. You are not required to use tools -- only use them when they provide information you cannot derive yourself."*

5. **Implement outcome-based reward scoring for trajectory evaluation.** When comparing multiple candidate approaches (e.g., during testing or optimization), score complete trajectories using AdaReasoner's composite reward: format correctness (are tool calls well-formed?), tool quality (did the tool calls make sense given the context?), and final accuracy (is the answer correct?). Weight accuracy highest.

   ```python
   def score_trajectory(trajectory, ground_truth):
       format_ok = all(is_well_formed(step) for step in trajectory if step["role"] == "tool")
       tool_score = mean(rate_tool_relevance(step, trajectory) for step in trajectory if step["role"] == "tool")
       accuracy = 1.0 if trajectory[-1]["content"] == ground_truth else 0.0
       return int(format_ok) * (0.3 * tool_score + 0.7 * accuracy)
   ```

6. **Apply the asymmetric reward principle.** When the system arrives at a correct answer without tools, reward it fully -- do not penalize tool-free solutions. When incorrect, give partial credit if the tool calls were contextually relevant and well-formed. This prevents two failure modes: gratuitous tool calling (tool overuse) and complete tool avoidance.

7. **Decouple tool identity from tool routing.** Reference tools by capability descriptions rather than fixed names in your routing logic. This mirrors AdaReasoner's token-level randomization. In practice: use a tool registry where the orchestrator matches task needs against tool descriptions, not a hardcoded `if task_type == "ocr": call_ocr()` dispatch.

8. **Add dynamic frequency regulation.** Track tool invocation counts per trajectory and allow the orchestrator to modulate. Simple implementation: after N tool calls without progress toward the answer, raise the threshold for making additional calls by injecting a prompt like *"You have already called {N} tools. Only make another call if it will provide critical new information."*

9. **Handle tool failures gracefully with fallback chains.** Define a fallback strategy for each tool: if OCR fails, try a different image preprocessing then retry; if search returns nothing, broaden the query; if all tool options are exhausted, produce a best-effort answer from available context. Never let a tool failure halt the pipeline.

10. **Validate with held-out scenarios including unseen tools.** Test the orchestrator with tools it hasn't seen during development. A well-built adaptive orchestrator should be able to read a new tool's description and decide whether and when to use it, without code changes to the routing logic.

## Concrete Examples

**Example 1: Adaptive Document Analysis Pipeline**

User: "Build me a pipeline that answers questions about scanned documents. Sometimes they have tables, sometimes just text, sometimes mixed. It should figure out what tools to use."

Approach:
1. Define tool registry: `ocr_extract` (full-page text extraction), `table_detect` (find table regions), `table_parse` (structured table extraction), `crop_region` (isolate document sections), `text_search` (find relevant passages).
2. Implement the iterative orchestration loop -- the agent first inspects the document, then decides:
   - If tables detected: crop table region -> parse table -> answer from structured data
   - If dense text: OCR full page -> search for relevant passage -> answer
   - If mixed: crop regions -> process each with appropriate tool -> synthesize
3. Include reflection: if OCR output is garbled, retry with image preprocessing; if table parse fails, fall back to OCR on the table region.

Output architecture:
```python
class AdaptiveDocQA:
    def __init__(self, tools: ToolRegistry):
        self.tools = tools
        self.max_steps = 8

    def answer(self, document: Image, question: str) -> str:
        trajectory = [{"input": document, "question": question}]
        for step in range(self.max_steps):
            # Present full trajectory to reasoner
            action = self.reason(trajectory)
            if action.is_final:
                return action.answer
            result = self.tools.execute(action.tool_name, action.params)
            trajectory.append({
                "tool": action.tool_name,
                "result": result,
                "useful": None  # evaluated next iteration
            })
            # Reflection: was the last tool call helpful?
            if step > 0:
                trajectory[-1]["useful"] = self.evaluate_utility(trajectory)
                if not trajectory[-1]["useful"]:
                    trajectory.append({"reflection": "Tool result unhelpful, adjusting strategy"})
        return self.best_effort_answer(trajectory)
```

**Example 2: Self-Correcting Code Analysis Agent**

User: "I want an agent that analyzes a codebase to answer architectural questions. It should use tools like grep, AST parsing, dependency graphing, and test running -- but only the ones that actually help for each specific question."

Approach:
1. Register tools with rich descriptions:
   - `grep_search`: "Find text patterns across files. Useful for locating usages, imports, and string literals. Not useful for understanding control flow."
   - `ast_parse`: "Parse a file into an abstract syntax tree. Useful for understanding class hierarchies, function signatures, and call graphs. Not useful for runtime behavior."
   - `dep_graph`: "Generate module dependency graph. Useful for architecture-level questions about coupling and layering. Not useful for function-level questions."
   - `run_tests`: "Execute test suite. Useful for verifying behavior claims. Not useful for static analysis questions."
2. Orchestration loop with frequency regulation:
   - "How is authentication handled?" -> grep for auth patterns -> AST parse the auth module -> answer (2 tools)
   - "What happens when a payment fails?" -> grep for error handling -> AST parse payment module -> AST parse retry logic -> grep for fallback patterns -> answer (4 tools)
   - "Is the code well-tested?" -> run tests -> dep graph to find uncovered modules -> answer (2 tools)
3. The agent suppresses `run_tests` for static questions and suppresses `dep_graph` for function-level questions -- adaptively, based on intermediate results.

Output:
```
Q: "How does the app handle database connection failures?"

Step 1: grep_search("database.*connection.*error|db.*fail|connection.*retry")
  -> Found 3 files: db/pool.py, db/retry.py, config/resilience.py

Step 2: ast_parse("db/retry.py")
  -> Class RetryPolicy with exponential backoff, max_retries=3

Step 3: [Reflection] Have enough to answer without further tools.

Answer: Database connection failures are handled by RetryPolicy in db/retry.py,
which implements exponential backoff with a maximum of 3 retries. The connection
pool in db/pool.py marks failed connections and resilience config is in
config/resilience.py.
```

**Example 3: Dynamic API Orchestration for Data Enrichment**

User: "Build a data enrichment pipeline that takes company names and enriches them with whatever data it can find -- sometimes the company has a website, sometimes only social media, sometimes nothing. The pipeline should adapt."

Approach:
1. Tool registry: `web_search`, `scrape_url`, `social_lookup`, `whois_lookup`, `extract_structured_data`, `validate_data`.
2. Adaptive orchestration:
   - Start with `web_search` for every company
   - If website found: `scrape_url` -> `extract_structured_data` (2-3 tool chain)
   - If only social found: `social_lookup` -> `extract_structured_data` (2 tool chain)
   - If nothing found: produce partial result with confidence score (0 tools after search)
3. Frequency adapts: well-known companies need 2 calls; obscure ones may need 5+ with fallbacks.

```python
# The orchestrator dynamically adjusts per entity
# Company A (Apple): search -> scrape homepage -> done (2 calls)
# Company B (tiny startup): search -> no site -> social_lookup -> found LinkedIn
#   -> scrape LinkedIn -> extract data -> validate -> done (5 calls)
# Company C (defunct): search -> nothing -> social_lookup -> nothing
#   -> [reflect: no data available] -> return partial with low confidence (2 calls)
```

## Best Practices

- **Do:** Design tool descriptions as the primary routing mechanism. The orchestrator should select tools based on what they *do* (from descriptions), not what they're *called*. This is the single most important principle from AdaReasoner.
- **Do:** Include tool-failure trajectories in your test cases. The system must handle gracefully: tools returning empty results, tools timing out, tools returning malformed data, and tools that are simply wrong for the task.
- **Do:** Reward correct answers *regardless* of tool usage. If the agent can answer without tools, that's ideal -- never penalize efficiency. This asymmetric incentive prevents the agent from calling tools just to "show its work."
- **Do:** Cap tool calls per trajectory with a soft limit (warn after N calls) and a hard limit (force final answer after M calls). AdaReasoner found natural convergence around 3-5 calls per task, but complex tasks may need more.
- **Avoid:** Fixed tool chains like "always call A then B then C." This defeats the purpose of adaptive orchestration. Even if tool A is usually first, let the system discover that from context.
- **Avoid:** Exposing raw tool names in routing logic. If you rename a tool from `ocr_v2` to `text_extractor`, nothing in the orchestration logic should break. Coupling to names is the most common failure mode.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Tool returns empty/null | Check result is non-empty after each call | Log as unhelpful, reflect, try alternative tool or different parameters |
| Tool call is malformed | Validate tool name exists and parameters match schema before execution | Re-prompt the reasoner with the schema; if repeated, skip tool and reason directly |
| Tool timeout | Set per-tool timeouts (e.g., 30s for API calls, 5s for local computation) | Append timeout notice to trajectory; let reasoner decide whether to retry or proceed without |
| Infinite tool loop | Track (tool, params) pairs; detect repeated identical calls | Inject "You have already called this tool with these parameters" into context; force progress |
| All tools fail | Count consecutive failures | After 3 consecutive failures, switch to tool-free reasoning and produce best-effort answer |
| Trajectory exceeds context window | Monitor token count of accumulated trajectory | Summarize earlier tool results into compressed form; keep most recent 2-3 results verbatim |

## Limitations

- **Not suited for single-step tasks.** If a task requires exactly one tool call with no ambiguity (e.g., "convert this file from PNG to JPG"), adaptive orchestration adds unnecessary overhead. Use direct tool invocation instead.
- **Requires good tool descriptions.** The quality of adaptive routing depends entirely on how well tools are described. Vague descriptions like "processes data" will produce poor routing decisions.
- **Cold-start problem with novel tools.** While the architecture supports unseen tools via description-based routing, the first few uses of a genuinely novel tool may be suboptimal until the system has observed its outputs.
- **Evaluation cost.** Trajectory-based reasoning consumes more tokens than single-shot tool selection. For latency-critical applications, consider caching common trajectories or using a lightweight classifier as a fast path before falling back to full adaptive reasoning.
- **Not a replacement for domain-specific pipelines.** When tool ordering is physically constrained (e.g., you *must* authenticate before querying an API), encode those constraints as hard rules. Adaptive orchestration is for the soft decisions -- which tools, how many, and when to stop.

## Reference

**Paper:** [AdaReasoner: Dynamic Tool Orchestration for Iterative Visual Reasoning](https://arxiv.org/abs/2601.18631v2) (Song et al., 2026)

**What to look for:** Section 3 for the full Tool-GRPO algorithm and reward formulation; Section 3.3 for the adaptive learning mechanism (token randomization + semantic paraphrasing); Figure 3 for concrete multi-step tool trajectories showing perception-planning-verification patterns; Table 1 for the asymmetric reward design that prevents tool overuse.