---
name: "toolself-unifying-task-execution"
description: "Implement self-reconfiguring agent workflows where configuration (sub-goals, strategy, toolbox, context) is a mutable tool the agent calls at runtime. Use when: 'build an adaptive agent', 'self-reconfiguring pipeline', 'agent that adjusts its own strategy', 'dynamic tool selection workflow', 'multi-stage task with runtime adaptation', 'ToolSelf pattern'."
---

# ToolSelf: Self-Reconfiguring Agent Workflows

This skill teaches Claude to build agent systems that treat their own configuration as a callable tool, following the ToolSelf paradigm (arXiv:2602.07883). Instead of hard-coding agent behavior upfront, you expose four mutable parameters -- sub-goals, execution strategy, active toolbox, and accumulated context -- as a `reconfigure()` action the agent invokes between execution stages. This unifies task execution and self-adjustment into a single action loop, letting agents autonomously pivot strategy, prune tools, and compress context as tasks evolve.

## When to Use

- When building a multi-stage agent pipeline where later stages depend on discoveries from earlier ones (e.g., research-then-code, crawl-then-analyze)
- When the user asks for an agent that "adapts its approach" or "adjusts strategy mid-task" rather than following a rigid plan
- When implementing error recovery where the agent should autonomously try a different approach after failure
- When designing a workflow with a large tool catalog where only a subset is relevant at each stage
- When context windows risk pollution from earlier stages and you need stage-local context with a compressed global history
- When building orchestration code that dispatches sub-agents and needs to dynamically reassign goals

## Key Technique

**Configuration-as-Tool.** Traditional agent loops fix their system prompt, tool list, and goals before execution begins. ToolSelf replaces this static config with a dynamic 4-tuple `C = (q, sigma, T, K)`: the current sub-goal `q`, the execution strategy/persona `sigma`, the active toolbox `T` (a subset of all available tools), and task-relevant knowledge `K`. At the end of each execution stage, the agent can call a `reconfigure(proposed_goal, rationale, requirements)` tool that archives the current stage's summary into a global history and generates a fresh configuration for the next stage. This means the agent never accumulates raw conversation history across stages -- it works with a compressed summary plus fresh, stage-specific context.

**Why it works.** The reconfiguration boundary serves three purposes: (1) it forces the agent to articulate what it learned and what it needs next (the rationale), which improves planning quality; (2) it resets stage-local context to prevent token bloat -- the paper shows max input tokens grow only ~8% despite 4x more execution steps; (3) it enables toolbox pruning so the agent only sees tools relevant to its current sub-goal, reducing decision complexity. The pattern is implementable without any model fine-tuning -- the training stages (CAT) described in the paper improve internalization, but the architectural pattern alone yields significant gains.

## Step-by-Step Workflow

1. **Define the global tool catalog.** List every tool/function the agent could possibly need across all stages. Assign each a short name and category tag (e.g., `web_search:retrieval`, `code_exec:compute`, `file_write:output`).

2. **Design the configuration schema.** Create a data structure with four fields:
   ```python
   @dataclass
   class AgentConfig:
       sub_goal: str           # What the agent is trying to accomplish this stage
       strategy: str           # How it should approach the sub-goal (persona/method)
       active_tools: list[str] # Subset of tool names available this stage
       knowledge: str          # Compressed relevant context from prior stages
   ```

3. **Implement the reconfigure tool.** Write a function the agent can call that accepts three arguments: `proposed_next_goal` (what to do next), `rationale` (why the current stage is complete and what was learned), and `requirements` (what strategy/tools/knowledge the next stage needs). This function archives the current stage summary into a global history list and returns a new `AgentConfig`.

4. **Implement the terminate tool.** Write a companion function the agent calls when the overall task is complete, which collects the final answer from the last stage.

5. **Build the stage execution loop.** Each iteration: inject the current `AgentConfig` into the system prompt, restrict available tools to `active_tools` plus `reconfigure` and `terminate`, run the agent until it calls one of those two meta-tools, then either loop (reconfigure) or return (terminate).

6. **Seed the initial configuration.** Parse the user's request into an initial sub-goal, pick a default strategy (e.g., "analytical reasoning"), select a starting tool subset, and set knowledge to the raw user query.

7. **Implement global history compression.** When the agent calls `reconfigure`, summarize the current stage's trajectory into 2-3 sentences and append to a running history list. Pass only this compressed history (not raw conversation) into the next stage's knowledge field.

8. **Add guardrails.** Set a maximum stage count (e.g., 8) to prevent infinite reconfiguration loops. Require the rationale field to be non-empty. Log each reconfiguration event for debugging.

9. **Wire up error-triggered reconfiguration.** When a tool call fails or returns unexpected results within a stage, the agent should call `reconfigure` with a rationale explaining the failure and a revised strategy, rather than retrying blindly.

10. **Test with a multi-hop task.** Run a task requiring at least 3 distinct phases (e.g., "find data, analyze it, generate a report") and verify that the agent produces distinct configurations per stage with appropriate tool subsets.

## Concrete Examples

**Example 1: Adaptive Research-to-Code Pipeline**

```
User: "Find the current population of the 5 largest EU countries,
       then write a Python script that creates a bar chart of the results."

Stage 1 Config:
  sub_goal: "Retrieve current population figures for the 5 largest EU countries"
  strategy: "web research with cross-reference verification"
  active_tools: [web_search, web_fetch]
  knowledge: "User wants populations of top 5 EU countries by population"

Agent executes: searches for populations, collects data for Germany,
France, Italy, Spain, Poland.

Agent calls reconfigure():
  proposed_goal: "Generate a Python bar chart script using the collected data"
  rationale: "Retrieved verified populations: DE=84.5M, FR=68.2M,
              IT=58.9M, ES=48.6M, PL=36.8M. Research phase complete."
  requirements: "Need code generation tools. No further web access needed.
                 Carry forward the population data."

Stage 2 Config:
  sub_goal: "Generate a Python bar chart script with the population data"
  strategy: "code generation with matplotlib best practices"
  active_tools: [code_write, code_exec]
  knowledge: "Populations: DE=84.5M, FR=68.2M, IT=58.9M, ES=48.6M, PL=36.8M"

Agent executes: writes and tests the matplotlib script.
Agent calls terminate() with the final script.
```

**Example 2: Error Recovery via Self-Reconfiguration**

```
User: "Calculate the driving distance between the two most
       recently founded state capitals in the US."

Stage 1 Config:
  sub_goal: "Identify the two most recently founded US state capitals"
  strategy: "systematic web lookup"
  active_tools: [web_search, web_fetch]
  knowledge: "Need founding dates of all 50 US state capitals"

Agent executes: searches for "most recently founded state capitals."
Initial results are ambiguous -- conflicting dates for several cities.

Agent calls reconfigure():
  proposed_goal: "Verify founding dates by checking individual city pages"
  rationale: "Bulk search returned conflicting data. Need per-city
              verification. Narrowed candidates to: Juneau (1881),
              Oklahoma City (1889), Phoenix (1881), Carson City (1858)."
  requirements: "Keep web tools. Strategy should shift to targeted
                 per-city lookups rather than aggregate searches."

Stage 2 Config:
  sub_goal: "Verify founding dates of candidate capitals"
  strategy: "targeted per-city Wikipedia lookups with date extraction"
  active_tools: [web_search, web_fetch]
  knowledge: "Candidates: Juneau, Oklahoma City, Phoenix, Carson City.
              Need verified founding/incorporation dates."

Agent executes: confirms Oklahoma City (1889) and Juneau (1906 as capital).

Agent calls reconfigure():
  proposed_goal: "Calculate driving distance between Oklahoma City and Juneau"
  rationale: "Confirmed: Juneau became capital 1906, OKC became capital 1910.
              These are the two most recent. Now need distance calculation."
  requirements: "Need distance/maps API. No more web search needed."

Stage 3 Config:
  sub_goal: "Get driving distance between Oklahoma City, OK and Juneau, AK"
  strategy: "Use mapping API or known route data"
  active_tools: [web_search, code_exec]
  knowledge: "Two most recent state capitals: Oklahoma City (1910), Juneau (1906)"

Agent executes and calls terminate() with the answer.
```

**Example 3: Implementation in Code**

```python
# Minimal ToolSelf loop implementation
import json
from dataclasses import dataclass, asdict

@dataclass
class AgentConfig:
    sub_goal: str
    strategy: str
    active_tools: list[str]
    knowledge: str

def run_toolself_agent(user_query: str, all_tools: dict, max_stages: int = 8):
    global_history = []
    config = seed_initial_config(user_query, all_tools)

    for stage in range(max_stages):
        # Build stage-local prompt with current config
        prompt = build_stage_prompt(config, global_history)
        available = {k: all_tools[k] for k in config.active_tools}
        available["reconfigure"] = reconfigure_tool
        available["terminate"] = terminate_tool

        # Run agent within this stage
        result = execute_stage(prompt, available)

        if result.tool_called == "terminate":
            return result.final_answer

        if result.tool_called == "reconfigure":
            # Archive and compress current stage
            summary = compress_stage(stage, config, result.trajectory)
            global_history.append(summary)
            # Generate next config from reconfiguration request
            config = generate_next_config(
                result.proposed_goal,
                result.rationale,
                result.requirements,
                all_tools,
                global_history
            )

    return "Max stages reached. Last result: " + str(result)
```

## Best Practices

- **Do:** Make the rationale field mandatory and substantive. The quality of self-reconfiguration depends entirely on the agent articulating what it learned and why it needs to change. A rationale like "done with stage 1" is useless; "Found that the API returns paginated results, need to switch to batch fetching" drives good next-stage configuration.

- **Do:** Aggressively prune the toolbox per stage. If a stage is about code generation, remove web search tools. Fewer tools means fewer irrelevant options for the agent to consider and fewer hallucinated tool calls.

- **Do:** Keep global history entries short (2-3 sentences each). The entire point of stage boundaries is context compression. If you pass full conversation logs, you lose the benefit.

- **Do:** Log every reconfiguration event (stage number, old config, rationale, new config) for debugging. Reconfiguration traces are the primary diagnostic artifact.

- **Avoid:** Letting the agent reconfigure without a meaningful state change. If the proposed next goal is identical to the current one, reject the reconfiguration and force continued execution or a genuinely different approach.

- **Avoid:** Setting max stages too high. Most tasks resolve in 3-5 stages. More than 8 suggests the task decomposition is too granular or the agent is thrashing.

## Error Handling

| Problem | Response |
|---------|----------|
| Agent loops between two configurations | Detect repeated sub-goals in global history; force a strategy change or escalate to the user |
| Reconfigure called with empty rationale | Reject the call and instruct the agent to explain what it learned before reconfiguring |
| Tool call fails within a stage | Agent should call `reconfigure` with failure details rather than retrying the same approach indefinitely |
| Max stages exhausted | Return the best partial result from the last completed stage with an explanation of where progress stalled |
| Knowledge field exceeds token budget | Truncate oldest history entries first; keep the most recent 2-3 stage summaries intact |

## Limitations

- **Not for single-shot tasks.** If a task can be completed in one tool call or one reasoning step, the reconfiguration overhead adds latency for no benefit.
- **Requires a meaningful tool catalog.** If the agent only has 2-3 tools, toolbox pruning provides negligible value. The pattern shines with 10+ tools.
- **Configuration generation quality.** Without the CAT fine-tuning described in the paper, the quality of auto-generated configurations depends on the base model's instruction-following ability. Explicit schema validation helps.
- **No backtracking.** The linear stage progression doesn't support reverting to a prior stage's configuration. If the agent reconfigures incorrectly, it must reconfigure again forward, which can waste stages.
- **Debugging complexity.** Multi-stage traces are harder to debug than single-pass agent runs. Invest in logging from the start.

## Reference

[ToolSelf: Unifying Task Execution and Self-Reconfiguration via Tool-Driven Intrinsic Adaptation](https://arxiv.org/abs/2602.07883v1) -- Zhou et al., 2026. Key sections: Section 3 for the configuration-as-tool formalism (`C = (q, sigma, T, K)` and the reconfigure/terminate tool definitions), Section 4 for the CAT training pipeline, and Section 5.3 for the case studies showing multi-stage reconfiguration traces on GAIA benchmarks.