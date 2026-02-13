---
name: "supchain-bench-benchmarking-real-world-supply"
description: "Build reliable long-horizon supply chain agents using the SupChain-ReAct pattern: multi-path ReAct trajectories with majority voting for autonomous tool orchestration without handcrafted SOPs. Use when asked to 'build a supply chain agent', 'orchestrate multi-step tool calls for order management', 'diagnose fulfillment issues', 'create an SOP-free agent workflow', 'implement long-horizon tool calling', or 'build an e-commerce order diagnostic system'."
---

This skill teaches Claude to build reliable, long-horizon supply chain agent systems using the SupChain-ReAct framework from the SupChain-Bench paper. The core technique replaces brittle, hand-authored Standard Operating Procedures (SOPs) with autonomous multi-path ReAct reasoning and majority-vote aggregation, enabling agents to synthesize their own executable procedures for tool orchestration across complex supply chain workflows spanning order management, fulfillment tracking, warehouse operations, cancellation analysis, and error diagnosis.

## When to Use

- When the user asks to build an agent that orchestrates 10-30+ sequential tool calls to resolve supply chain or e-commerce order issues
- When designing a diagnostic pipeline that traces orders through trade, fulfillment, and warehouse layers
- When the user needs an agent framework that works without hand-authored SOPs or rigid procedural scripts
- When implementing multi-step tool-calling workflows where early termination and execution drift are failure risks
- When building order investigation systems that must handle branching logic (cancelled vs. error vs. in-transit statuses)
- When the user wants to improve tool-calling reliability through parallel reasoning paths and consensus voting
- When creating agents for any domain requiring long-horizon, multi-entity traversal across linked database records

## Key Technique

**The Problem**: LLMs performing multi-step tool orchestration in supply chain settings suffer from three failure modes: (1) premature termination, where the model stops calling tools before exhausting all entities; (2) schema mismatches, where field names drift between tool calls; and (3) faithfulness errors, where the model's final response contradicts what tools actually returned. Providing hand-written SOPs helps but requires expensive domain expertise and still fails for models that prioritize conversational brevity over exhaustive coverage.

**SupChain-ReAct**: Instead of authoring SOPs, run N independent ReAct trajectories (the paper uses N=5) in parallel against the same task prompt and tool schema. Each trajectory alternates between a reasoning step ("I need to check the fulfillment status for each ID") and a tool invocation, continuing until it produces a final answer or hits a step limit. The final output is selected by majority vote over the textual answers from successful trajectories. This approach works because: (a) different trajectories explore different orderings and branching paths, reducing the chance that all paths prematurely terminate at the same point; (b) majority voting filters out hallucinated or unfaithful answers since they are unlikely to appear in a majority of independent runs; and (c) the model leverages its existing domain knowledge and tool-schema understanding to self-organize procedural steps without external instruction.

**Results**: SupChain-ReAct consistently outperformed both SOP-free and SOP-guided baselines across models. For example, Gemini-2.5-Pro jumped from 11.22% (no SOP) to 72.44% (SupChain-ReAct), and Claude-4-Sonnet went from 31.63% to 75.51%. The technique is model-agnostic and requires no training or fine-tuning.

## Step-by-Step Workflow

1. **Define the entity graph and tool schema.** Model your domain as linked entities (e.g., TradeOrder -> FulfillmentOrder -> WarehouseOrder) with foreign key relationships. For each entity type, define tool functions that retrieve status, error details, or related records. Expose these as typed function signatures with clear parameter names and return schemas.

2. **Design tool functions that traverse one hop each.** Each tool should do exactly one thing: `query_buyer_and_related(order_id)` returns linked IDs, `get_fulfillment_status(fulfillment_id)` returns a status enum, `get_warehouse_error_details(fulfillment_id, warehouse_order_id)` returns error codes. Keep tools atomic so the agent controls the traversal order.

3. **Build the ReAct loop for a single trajectory.** Implement a reasoning-action cycle where the agent: (a) receives the user question and full tool schema, (b) produces a "Thought" explaining what information it needs next, (c) emits a tool call with exact arguments, (d) receives the tool result, and (e) repeats until it produces a "Final Answer" or hits the step limit (30-40 steps for complex workflows).

4. **Launch N parallel trajectories (N=5 recommended).** Execute N independent ReAct loops against the same prompt and tool environment. Each trajectory has its own reasoning chain and may traverse entities in different orders or follow different conditional branches. Use independent random seeds or temperature > 0 to ensure diversity.

5. **Implement conditional branching within each trajectory.** The agent must interpret tool outputs and branch accordingly: if `get_fulfillment_status` returns "cancelled", the next calls should be `get_cancel_scenes` and `get_cancel_error_code`; if "error", call `get_error_reason`; otherwise proceed to warehouse-level enrichment. Encode these conditions in the system prompt as domain hints, not rigid step sequences.

6. **Enforce exhaustive entity iteration.** In the system prompt, instruct the agent: "For every entity ID returned by the initial query, you must check its status and retrieve error details. Do not stop until all entities have been processed." This combats the premature-termination failure mode that affects most models.

7. **Collect final answers and apply majority voting.** From the N trajectories, extract the final textual answer from each successful run (discard trajectories that hit the step limit without producing an answer). Select the answer that appears most frequently. If there is a tie, prefer the answer from the trajectory with the most tool calls (indicating more thorough investigation).

8. **Validate tool-response faithfulness.** Before returning the voted answer, cross-check that all factual claims in the answer (status values, error codes, entity counts) match the actual tool outputs from the winning trajectory. Flag and correct any discrepancies where the model's summary contradicts its own tool results.

9. **Measure with Information Retrieval Accuracy.** Evaluate by comparing the set of facts retrieved by the agent's tool calls against a ground-truth set produced by an oracle script that programmatically executes the correct procedure. This metric captures both whether the right tools were called and whether the results were correctly propagated to the final answer.

10. **Iterate on the system prompt, not on SOPs.** When accuracy is low, tune the system prompt's domain hints and entity-iteration instructions rather than writing rigid step-by-step procedures. The key insight is that models already understand tool schemas; they need encouragement to be thorough, not micromanagement of call order.

## Concrete Examples

**Example 1: Order Diagnostic Agent**

User: "Build me an agent that can investigate any e-commerce order and tell me exactly what happened with each fulfillment and warehouse shipment."

Approach:
1. Define the entity schema with five tables: TradeOrders, FulfillmentOrders, WarehouseOrders, ErrorLogs, CancellationContext
2. Create seven tool functions:
   - `query_buyer_and_related(order_id)` -> buyer_id, list of (fulfillment_id, warehouse_order_id)
   - `get_fulfillment_status(fulfillment_id)` -> status enum
   - `get_warehouse_status(fulfillment_id, warehouse_order_id)` -> status, error_code
   - `get_warehouse_error_details(fulfillment_id, warehouse_order_id)` -> code, text
   - `get_cancel_scenes(fulfillment_id)` -> cancelType
   - `get_cancel_error_code(fulfillment_id)` -> cancelErrorCode, cancelErrorMsg
   - `check_fake_shipping(fulfillment_id)` -> boolean flag
3. Implement a ReAct executor that runs 5 parallel trajectories per query
4. System prompt includes domain hints but no rigid SOP

Output system prompt:
```
Role: You are a supply chain diagnostic agent. Given a trade order ID,
your job is to comprehensively investigate every linked fulfillment and
warehouse order to build a complete picture of what happened.

Available tools: [tool schemas provided here]

Instructions:
- Start by calling query_buyer_and_related to discover all linked entities.
- For EVERY fulfillment_id returned, call get_fulfillment_status.
- If status is "cancelled": call get_cancel_scenes AND get_cancel_error_code.
  If cancelType is null/unknown, also call check_fake_shipping.
- If status is "error": call get_error_reason.
- For EVERY warehouse_order_id returned, call get_warehouse_status
  AND get_warehouse_error_details.
- Do NOT stop until every entity has been fully investigated.
- Report all findings with exact values from tool outputs.
```

Executor pseudocode:
```python
async def supchain_react(question, tools, n_trajectories=5, max_steps=35):
    trajectories = await asyncio.gather(*[
        run_react_trajectory(question, tools, max_steps)
        for _ in range(n_trajectories)
    ])
    successful = [t for t in trajectories if t.has_final_answer]
    if not successful:
        return fallback_to_longest_trajectory(trajectories)
    return majority_vote([t.final_answer for t in successful])
```

**Example 2: Adapting SupChain-ReAct to a Non-Supply-Chain Domain (IT Incident Triage)**

User: "I want to use this multi-path ReAct pattern for our IT incident management system that investigates alerts across services, pods, and log streams."

Approach:
1. Map the entity graph: Incident -> AffectedServices -> Pods -> LogStreams
2. Define atomic tools:
   - `get_incident_details(incident_id)` -> affected_service_ids, severity, timestamp
   - `get_service_status(service_id)` -> health, error_rate, linked_pod_ids
   - `get_pod_logs(pod_id, time_range)` -> log entries
   - `get_pod_metrics(pod_id)` -> cpu, memory, restarts
   - `check_deployment_history(service_id)` -> recent deploys
3. Apply the same pattern: 5 parallel ReAct trajectories, exhaustive entity iteration, majority vote

Output:
```python
# Same executor, different domain
tools = [get_incident_details, get_service_status, get_pod_logs,
         get_pod_metrics, check_deployment_history]

system_prompt = """
Role: You are an IT incident triage agent. Given an incident ID,
investigate every affected service, every pod, and relevant logs.

Instructions:
- Call get_incident_details to discover affected services.
- For EVERY service_id, call get_service_status.
- For EVERY pod_id linked to affected services, call get_pod_logs
  AND get_pod_metrics.
- If any service had a recent deployment, call check_deployment_history.
- Do NOT stop until every entity has been fully investigated.
"""

result = await supchain_react(user_question, tools, n_trajectories=5)
```

**Example 3: Preventing Premature Termination in Existing Agents**

User: "My agent keeps stopping after checking 2 out of 6 fulfillment orders. How do I fix this?"

Approach:
1. This is the "metacognitive myopia" problem identified in the paper where models adopt a "good-enough stopping rule"
2. Apply three fixes from SupChain-ReAct:
   - Add explicit iteration instructions: "You MUST process ALL N entities, not just the first few"
   - Run multiple trajectories (N=5) so at least some complete the full traversal
   - Use majority voting to select the most complete answer

```python
# Add to system prompt:
ANTI_TERMINATION_INSTRUCTIONS = """
CRITICAL: The initial query may return multiple entity IDs.
You MUST call the appropriate status and detail tools for EVERY
entity ID. Count the entities and verify you have investigated
each one before producing your final answer.

Before your final answer, output a checklist:
- Total entities found: X
- Entities investigated: [list each ID and what you found]
- Remaining: [should be empty]
"""

# Run 5 trajectories and vote
results = await supchain_react(question, tools, n_trajectories=5)
# The trajectory that investigated all 6 orders will produce
# the most accurate answer and likely win the majority vote
```

## Best Practices

- **Do:** Run at least 5 parallel trajectories. The paper found this balances computational cost against reasoning diversity. Fewer trajectories reduce the self-correction benefit of voting.
- **Do:** Keep tool functions atomic (one hop, one entity). This gives the agent maximum control over traversal order and lets different trajectories explore different paths.
- **Do:** Include explicit exhaustive-iteration instructions in the system prompt. The single most common failure mode is premature termination after processing only a subset of entities.
- **Do:** Validate that the final answer's factual claims match the actual tool outputs. Faithfulness errors (model says "cancelled" when the tool returned "in_transit") are the second most common failure mode.
- **Avoid:** Writing rigid step-by-step SOPs. The paper shows that SupChain-ReAct without SOPs outperforms SOP-guided approaches for most models. Domain hints are better than procedural micromanagement.
- **Avoid:** Using temperature=0 for all trajectories. Diversity across trajectories is what makes majority voting effective. Use moderate temperature (0.5-0.7) or different random seeds.
- **Avoid:** Aggregating at the intermediate step level. Vote only on final answers. Trying to align or merge intermediate reasoning steps across trajectories adds complexity without improving accuracy.

## Error Handling

| Failure Mode | Frequency | Mitigation |
|---|---|---|
| **Missing warehouse status** | Most common (4,215 cases in benchmark) | Enforce per-entity iteration with explicit counting in the prompt. Verify entity coverage before finalizing. |
| **Schema mismatches** | Common | Use consistent parameter names across tools. In the prompt, list exact parameter names. Catch argument errors and retry with corrected field names. |
| **Faithfulness errors** | 974 cases | Post-process: extract factual claims from the final answer and cross-check against the tool call log. Reject answers where claims contradict tool outputs. |
| **Premature termination** | Pervasive across non-GPT-5 models | Multi-trajectory voting naturally mitigates this. Also add step-limit awareness: "You have a budget of 35 tool calls. Use them all if needed." |
| **Enum normalization** | Models map anomalous states to valid enums | Instruct the agent to preserve raw status values from tools rather than normalizing to a clean taxonomy. |
| **Trajectory timeout** | Some trajectories hit step limit without answering | Discard these and vote among successful trajectories. If all timeout, fall back to the trajectory with the most tool calls. |

## Limitations

- **Cost**: Running 5 parallel trajectories multiplies inference cost by ~5x. For latency-sensitive applications, consider reducing to 3 trajectories with the understanding that accuracy will decrease.
- **Simple queries don't benefit**: For tasks requiring fewer than 5 tool calls, multi-path voting adds overhead without meaningful accuracy improvement. Use single-trajectory ReAct for simple lookups.
- **Domain knowledge ceiling**: The technique assumes the model already has sufficient domain knowledge to reason about which tools to call. For highly specialized domains with no representation in training data, you may still need to provide domain context (the paper shows ~15-20 percentage point gains from adding domain documents).
- **Text-only**: The benchmark and technique operate on structured text data and tool APIs. They do not handle multimodal inputs like sensor data, images, or real-time streaming logs.
- **Voting assumes independence**: If the same systematic error appears in all trajectories (e.g., all models misunderstand a particular tool schema), majority voting will amplify rather than correct the error.

## Reference

[SupChain-Bench: Benchmarking Large Language Models for Real-World Supply Chain Management](https://arxiv.org/abs/2602.07342v1) (Guan, Liu, Cao, 2026). Key sections: Section 4.5 for the SupChain-ReAct framework design, Table 4 for comparative results showing SupChain-ReAct outperforming SOP-guided baselines, Table 5 for the exact SOP prompt structure, and Section 4.2-4.3 for the analysis of premature termination and error modes across models.