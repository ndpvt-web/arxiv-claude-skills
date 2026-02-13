---
name: "internet-agentic-ai-incentive-compatible"
description: >
  Design and implement incentive-compatible multi-agent coalition systems where
  heterogeneous AI agents dynamically form teams to execute workflows, using
  capability coverage, network locality, and Shapley-value-based fair payment
  allocation. Operates as a coordination layer above MCP or any tool-calling protocol.
  Trigger phrases: "design a multi-agent coalition system", "incentive-compatible
  agent teaming", "fair task allocation across agents", "decentralized agent
  coordination", "Shapley value agent payments", "coalition formation for workflows".
---

# Internet of Agentic AI: Incentive-Compatible Distributed Teaming and Workflow

This skill enables Claude to design, implement, and reason about **distributed multi-agent systems** where autonomous agents with different capabilities dynamically form coalitions to execute multi-step workflows. The core technique from Yang & Zhu (2026) formalizes three jointly-necessary conditions for a viable coalition: (1) the union of agent capabilities must cover every workflow step, (2) agents must satisfy network locality and latency constraints, and (3) payment allocation via Shapley values must be individually rational, core-stable, and budget-balanced -- so no agent has an incentive to defect. Claude uses this framework to architect real coordination layers, write coalition-formation algorithms, and validate that agent team designs are economically sound.

## When to Use

- When the user asks to build a system where multiple AI agents (or microservices) must cooperate to complete a multi-step task and they need fair cost/reward splitting.
- When designing an MCP-based architecture where different tool servers have different capabilities and a coordinator must select the minimal viable subset.
- When implementing a task-routing layer that must respect latency constraints (e.g., cloud vs. edge placement of agents).
- When the user needs to verify that a proposed agent coalition is stable -- that no sub-group would prefer to break away and work alone.
- When building healthcare, logistics, or finance pipelines where regulatory isolation requires distinct agents for distinct steps and economic viability matters.
- When the user asks "how do I fairly split payment among agents that contributed to a workflow?"

## Key Technique

**Workflow-Coalition Feasibility.** A workflow W is a directed acyclic graph of steps, each requiring specific capabilities. Each agent i has a capability set C_i. A coalition S is *feasible* if and only if: (a) the union of all C_i for i in S covers every required capability in W, (b) inter-agent communication latencies satisfy the workflow's timing constraints, and (c) the Shapley-value payment vector is in the *core* of the cooperative game -- meaning every agent earns at least what it could earn alone, and no sub-coalition is underpaid relative to its standalone value. When all three conditions hold simultaneously, the coalition is **incentive-compatible**: no agent benefits from leaving or free-riding.

**Minimum-Effort Coalition Selection.** Among all feasible coalitions, the framework selects the one that minimizes total coordination overhead (number of agents, cross-region links, capability redundancy). This is formulated as a set-cover-like optimization: find the smallest subset of agents whose capabilities cover the workflow, subject to locality and incentive constraints. The paper proposes a decentralized algorithm where agents broadcast capabilities, propose partnerships based on local information, verify Shapley allocations, and iterate until a stable coalition emerges -- no central orchestrator required.

**MCP Coordination Layer.** The framework sits above the Model Context Protocol. It translates workflow requirements into capability queries, routes multi-step tasks through the agent network respecting MCP's message-passing semantics, and manages state consistency. When an individual MCP tool call fails, the coordination layer handles re-routing to an alternative agent with overlapping capabilities.

## Step-by-Step Workflow

1. **Model the workflow as a DAG.** Parse the user's task into discrete steps with explicit dependencies. Each step gets a required-capability label (e.g., `diagnose`, `query_drugs`, `schedule`, `document`). Output a JSON or YAML workflow definition with `steps`, `dependencies`, and `required_capabilities` per step.

2. **Define the agent registry.** For each available agent (or MCP server, or microservice), enumerate its capability set C_i, its network region (cloud/edge/on-prem), its latency to other agents, and its standalone value v({i}) -- what it could earn executing tasks alone.

3. **Enumerate candidate coalitions.** For small agent pools (< 15), enumerate all subsets whose capability union covers the workflow. For larger pools, use a greedy set-cover heuristic: iteratively add the agent that covers the most uncovered capabilities, breaking ties by preferring agents in the same network region.

4. **Apply network locality filter.** Discard any candidate coalition where the maximum pairwise latency between agents exceeds the workflow's end-to-end latency budget. Penalize cross-region links by adding a coordination cost proportional to round-trip time.

5. **Compute Shapley values for each surviving coalition.** For agent i in coalition S, compute phi_i as the average marginal contribution of i across all orderings of S. The marginal contribution of i to a subset T is v(T union {i}) - v(T), where v(T) is the total workflow value T can deliver.

6. **Check core stability.** For each sub-coalition T within S, verify that the sum of Shapley payments to T is at least v(T). If any sub-coalition is underpaid, the coalition is unstable -- discard it or adjust the value function.

7. **Check budget balance.** Verify that the sum of all Shapley payments does not exceed the total workflow compensation. If it does, apply a proportional scaling factor.

8. **Select the minimum-effort coalition.** Among all feasible, stable, budget-balanced coalitions, pick the one with the fewest agents (and lowest total coordination cost as tiebreaker).

9. **Generate the orchestration code.** Output a coordinator module that: dispatches each workflow step to the assigned agent, passes intermediate results along DAG edges, monitors for failures, and re-routes to backup agents if available.

10. **Emit the payment ledger.** Output a final allocation table mapping each agent to its Shapley-value payment, along with a proof that the allocation is individually rational and core-stable.

## Concrete Examples

**Example 1: Healthcare Workflow Coalition**

```
User: I have four MCP tool servers -- a diagnostics agent, a drug-interaction
checker, an appointment scheduler, and a records manager. A patient-intake
workflow needs all four steps in sequence. Each agent charges differently.
Design the coalition and fair payment split.

Approach:
1. Define workflow DAG:
   intake -> diagnose -> drug_check -> schedule -> document
   Required capabilities: {symptom_analysis, drug_query, scheduling, hipaa_docs}

2. Agent registry:
   - DiagnosticsAgent:  C={symptom_analysis},       region=cloud-us, v({d})=20
   - DrugDBAgent:       C={drug_query},              region=cloud-us, v({db})=15
   - SchedulerAgent:    C={scheduling},              region=edge-clinic, v({s})=10
   - RecordsAgent:      C={hipaa_docs},              region=cloud-us, v({r})=8

3. Only one feasible coalition: S = {d, db, s, r} (each covers exactly one
   required capability -- no redundancy).

4. Latency check: cloud-us <-> edge-clinic = 45ms. Budget = 200ms. Pass.

5. Shapley values (workflow total value V=100):
   phi_d = 32, phi_db = 27, phi_s = 23, phi_r = 18
   (Each agent's marginal contribution is indispensable, but weighted by
   standalone value and ordering effects.)

6. Core stability: Every singleton gets at least its standalone value. Check
   all 14 non-trivial subsets -- all pass.

7. Budget balance: 32+27+23+18 = 100 = V. Balanced.

Output:
{
  "coalition": ["DiagnosticsAgent","DrugDBAgent","SchedulerAgent","RecordsAgent"],
  "workflow": ["intake","diagnose","drug_check","schedule","document"],
  "payments": {"DiagnosticsAgent":32,"DrugDBAgent":27,"SchedulerAgent":23,"RecordsAgent":18},
  "core_stable": true,
  "budget_balanced": true
}
```

**Example 2: Selecting Minimal Coalition from Overlapping Agents**

```
User: I have 6 agents with overlapping capabilities. Find the smallest
incentive-compatible coalition for a 3-step data pipeline (extract, transform, load).

Approach:
1. Agent capabilities:
   A: {extract, transform}  v=30   region=cloud
   B: {transform, load}     v=25   region=cloud
   C: {extract}             v=12   region=edge
   D: {load}                v=10   region=edge
   E: {extract, load}       v=28   region=cloud
   F: {transform}           v=15   region=cloud

2. Feasible coalitions (cover {extract, transform, load}):
   {A,B}, {A,D}, {E,F}, {A,B,C}, {C,F,B}, ... (12 total)

3. Minimum-size feasible: {A,B} and {E,F} (both size 2).

4. Latency filter: {A,B} both cloud (5ms). {E,F} both cloud (5ms). Both pass.

5. Shapley values (V=80):
   {A,B}: phi_A=44, phi_B=36  (A covers 2/3 capabilities)
   {E,F}: phi_E=46, phi_F=34  (E covers 2/3 capabilities)

6. Core stability:
   {A,B}: phi_A=44 >= v({A})=30, phi_B=36 >= v({B})=25. Stable.
   {E,F}: phi_E=46 >= v({E})=28, phi_F=34 >= v({F})=15. Stable.

7. Tiebreak by coordination cost: both identical. Pick {A,B} (lower total
   standalone value means less "outside option" risk).

Output:
Selected coalition: {A, B}
Payments: A=$44, B=$36
Reason: Minimal size, core-stable, lowest defection risk.
```

**Example 3: Generating Coordinator Code**

```
User: Generate a Python coordinator for the coalition {A, B} executing the
extract->transform->load pipeline.

Output:
import asyncio
from dataclasses import dataclass

@dataclass
class Agent:
    name: str
    capabilities: set[str]
    endpoint: str

COALITION = {
    "extract":   Agent("A", {"extract","transform"}, "http://agent-a:8000"),
    "transform": Agent("A", {"extract","transform"}, "http://agent-a:8000"),
    "load":      Agent("B", {"transform","load"},    "http://agent-b:8000"),
}
WORKFLOW = ["extract", "transform", "load"]
PAYMENTS = {"A": 44, "B": 36}

async def execute_workflow(input_data: dict) -> dict:
    result = input_data
    for step in WORKFLOW:
        agent = COALITION[step]
        result = await call_agent(agent.endpoint, step, result)
    return {"result": result, "payments": PAYMENTS}

async def call_agent(endpoint: str, step: str, data: dict) -> dict:
    # Replace with actual MCP tool call or HTTP request
    import httpx
    async with httpx.AsyncClient() as client:
        resp = await client.post(f"{endpoint}/{step}", json=data, timeout=5.0)
        resp.raise_for_status()
        return resp.json()
```

## Best Practices

- **Do:** Always compute Shapley values rather than splitting payment equally. Equal splits incentivize free-riding by low-capability agents and penalize high-capability ones.
- **Do:** Model the workflow as an explicit DAG before selecting agents. Implicit workflows lead to missed capability requirements and unstable coalitions.
- **Do:** Include standalone values v({i}) for every agent. Without these, you cannot verify individual rationality or core stability.
- **Do:** Prefer same-region agents when capabilities overlap. Cross-region coordination costs compound with workflow depth.
- **Avoid:** Skipping the core-stability check. A coalition where Shapley payments are individually rational but not core-stable will fragment under rational agents -- a sub-group will defect.
- **Avoid:** Building monolithic single-agent systems when the workflow has natural capability boundaries. The framework's value comes from distributed specialization.

## Error Handling

- **No feasible coalition exists.** If no subset of available agents covers all workflow capabilities, report the uncovered capabilities explicitly and suggest either adding agents or decomposing the workflow into independently-solvable sub-workflows.
- **Core stability fails for all coalitions.** This means standalone agent values are too high relative to the workflow's total value. Either increase V (the task's budget), reduce agent outside options, or relax the stability requirement to epsilon-core (allow small subsidies).
- **Shapley computation is too expensive.** For coalitions larger than ~20 agents, exact Shapley computation (O(2^n)) is infeasible. Use Monte Carlo sampling: randomly sample 1000+ agent orderings and average marginal contributions. Report confidence intervals.
- **Agent fails mid-workflow.** The coordinator should detect the failure, identify alternative agents with overlapping capabilities from the original candidate pool, verify the replacement coalition is still feasible and stable, then resume from the last successful step.
- **Latency budget exceeded.** Re-run coalition selection with tighter locality constraints or decompose the workflow into parallel sub-DAGs that can execute within regional clusters.

## Limitations

- Shapley value computation is exponential in coalition size. Practical for up to ~20 agents; beyond that, requires Monte Carlo approximation with accuracy trade-offs.
- The framework assumes agents truthfully report capabilities. Strategic misreporting (claiming capabilities an agent lacks) is not addressed and requires separate mechanism design.
- Network locality is modeled as static pairwise latency. Dynamic network conditions (congestion, outages) require runtime re-evaluation not covered by the static formulation.
- The cooperative game model assumes transferable utility -- payment can be freely redistributed. In systems with non-monetary incentives (reputation, access), the core-stability analysis needs adaptation.
- Single-step workflows with a single capable agent don't benefit from this framework. It is designed for multi-step, multi-agent scenarios.

## Reference

Yang, Y.-T. & Zhu, Q. (2026). *Internet of Agentic AI: Incentive-Compatible Distributed Teaming and Workflow.* arXiv:2602.03145v1. Look for: Definition 3 (workflow-coalition feasibility), Algorithm 1 (decentralized coalition formation), and Section V (healthcare case study with quantitative results).