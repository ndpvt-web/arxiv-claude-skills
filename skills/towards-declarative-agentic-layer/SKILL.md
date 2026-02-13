---
name: "towards-declarative-agentic-layer"
description: "Build grounded, declarative agentic architectures using the DALIA pattern: capability descriptors, discovery protocols, federated agent directories, and deterministic task graphs. Use when the user says 'build a multi-agent system', 'create an agent orchestration layer', 'design declarative agent workflows', 'set up agent capability discovery', 'make agent coordination reliable', or 'avoid hallucinated agent plans'."
---

# Declarative Agentic Layer (DALIA) for Reliable Agent Orchestration

This skill enables Claude to design and build multi-agent systems that avoid hallucinated actions, unexecutable plans, and brittle coordination by applying the DALIA architecture from Rodriguez-Sanchez et al. (2026). Instead of letting LLMs freely improvise tool calls and coordination, DALIA enforces a declarative layer where every agent capability is explicitly declared with typed inputs/outputs, preconditions, and postconditions. Task graphs are constructed deterministically from these declarations, guaranteeing that every planned action is actually executable.

## When to Use

- When the user asks to build a multi-agent system that coordinates across multiple tools or services
- When designing an orchestration layer for MCP servers or similar tool-serving ecosystems
- When an existing agent system suffers from unreliable plans (agents attempt actions that don't exist or pass wrong parameters)
- When the user wants deterministic, reproducible task execution rather than open-ended LLM reasoning chains
- When building a service registry or capability discovery system for AI agents
- When the user needs to formalize what agents can and cannot do before letting them plan

## Key Technique

The core insight of DALIA is that most agent failures come not from the LLM being unintelligent, but from the absence of architectural structure linking goals to actual capabilities. When an LLM plans freely, it can hallucinate tools that don't exist, misunderstand parameter types, or construct plans with impossible data dependencies. DALIA eliminates this by requiring every executable capability to be declared upfront in a structured descriptor that specifies its ID, role, domain, typed inputs/outputs, preconditions, and postconditions.

DALIA separates the agent lifecycle into three strict phases. **Discovery** queries available MCP servers and the federated Agent Directory to build a ground-truth index of what capabilities exist and which agents can execute them. **Planning** constructs task graphs exclusively from discovered capabilities -- the planner cannot reference any operation that wasn't declared. **Execution** walks the task graph in topological order, propagating typed outputs from one node as inputs to the next. This separation means you can validate a plan statically before running anything.

The architecture introduces two key protocols: the **Capability Semantic Model** (structured descriptors for individual operations) and the **Agentic Task Discovery Protocol (ATDP)** (which exposes composite tasks as combinations of capabilities). A **Federated Agent Directory** maps agents to their roles, domains, and accessible servers without duplicating capability definitions. Together, these components create a verifiable operational space that constrains agent behaviour to what is actually possible.

## Step-by-Step Workflow

1. **Define capability descriptors** for every atomic operation in your system. Each descriptor must include: `capability_id`, `role` (functional category), `domain`, `inputs` (typed parameter list), `outputs` (typed result list), `preconditions` (what must be true before execution), and `postconditions` (what becomes true after execution).

2. **Define agent directory entries** for each agent. Each entry declares: `agent_id`, `role`, `domains` the agent operates in, and `accessible_servers` (which MCP or tool servers it can reach). Do not duplicate capability details here -- reference capability IDs only.

3. **Define ATDP task declarations** for each user-facing task. Each task binds an `intent` (what the user wants) to a set of `capabilities` that fulfil it, along with the task-level `inputs` and `outputs`. This is the contract between user goals and system capabilities.

4. **Implement the discovery phase.** When a request arrives, query all registered MCP servers for their capability descriptors. Consult the Agent Directory to identify which agents can execute which capabilities. Build an in-memory capability index.

5. **Implement the planning phase.** Match the user's intent against ATDP task declarations. Resolve the required capabilities from the index. Construct a directed acyclic graph (DAG) where nodes are capability invocations and edges are data dependencies (output of one node feeds input of the next). Validate that all preconditions can be satisfied and all input types match.

6. **Validate the task graph before execution.** Check: (a) every capability node maps to a registered agent, (b) all input parameters are bound (either from user input or from upstream outputs), (c) no circular dependencies exist, (d) all preconditions are satisfiable given the execution order.

7. **Execute the task graph in topological order.** Invoke each capability through its agent's MCP server connection. Capture typed outputs and propagate them to downstream nodes. If a node fails, halt execution and report which capability failed and why.

8. **Log execution traces** at each node: inputs received, outputs produced, postconditions asserted. This creates a reproducible audit trail that can be replayed or debugged.

9. **Handle dynamic capability changes.** If agents or servers come online/offline, re-run discovery and revalidate any in-flight task graphs. Invalidate plans that reference unavailable capabilities rather than attempting fallback reasoning.

## Concrete Examples

**Example 1: Restaurant Booking System**

User: "Build a multi-agent system that can search for restaurants and make reservations."

Approach:
1. Define capability descriptors:

```json
{
  "capability_id": "restaurant.search",
  "role": "information_retrieval",
  "domain": "food",
  "inputs": [
    {"name": "location", "type": "string"},
    {"name": "date", "type": "date"},
    {"name": "party_size", "type": "integer"}
  ],
  "outputs": [
    {"name": "restaurant_list", "type": "array<Restaurant>"}
  ],
  "preconditions": ["location_known"],
  "postconditions": ["results_available"]
}
```

```json
{
  "capability_id": "restaurant.reserve",
  "role": "transaction",
  "domain": "food",
  "inputs": [
    {"name": "restaurant_id", "type": "string"},
    {"name": "date", "type": "date"},
    {"name": "party_size", "type": "integer"}
  ],
  "outputs": [
    {"name": "booking_confirmation", "type": "BookingConfirmation"}
  ],
  "preconditions": ["results_available"],
  "postconditions": ["reservation_confirmed"]
}
```

2. Define the agent directory entry:

```json
{
  "agent_id": "RestaurantAgent",
  "role": "booking_agent",
  "domains": ["food"],
  "accessible_servers": ["restaurant_mcp"]
}
```

3. Define the ATDP task:

```json
{
  "task_id": "restaurant.booking",
  "intent": "book_restaurant",
  "inputs": ["location", "date", "party_size"],
  "outputs": ["booking_confirmation"],
  "capabilities": ["restaurant.search", "restaurant.reserve"]
}
```

4. The planner constructs this task graph:

```
[user input: location, date, party_size]
        |
        v
  restaurant.search (RestaurantAgent)
        |
        | restaurant_list -> user selects -> restaurant_id
        v
  restaurant.reserve (RestaurantAgent)
        |
        v
  [output: booking_confirmation]
```

**Example 2: Multi-Agent Document Processing Pipeline**

User: "Design an agent system where one agent extracts text from PDFs, another summarizes it, and a third translates the summary."

Approach:
1. Define three capability descriptors:

```json
[
  {
    "capability_id": "doc.extract_text",
    "role": "extraction",
    "domain": "documents",
    "inputs": [{"name": "pdf_url", "type": "string"}],
    "outputs": [{"name": "raw_text", "type": "string"}],
    "preconditions": ["pdf_accessible"],
    "postconditions": ["text_extracted"]
  },
  {
    "capability_id": "doc.summarize",
    "role": "analysis",
    "domain": "documents",
    "inputs": [{"name": "raw_text", "type": "string"}, {"name": "max_length", "type": "integer"}],
    "outputs": [{"name": "summary", "type": "string"}],
    "preconditions": ["text_extracted"],
    "postconditions": ["summary_available"]
  },
  {
    "capability_id": "doc.translate",
    "role": "transformation",
    "domain": "language",
    "inputs": [{"name": "text", "type": "string"}, {"name": "target_lang", "type": "string"}],
    "outputs": [{"name": "translated_text", "type": "string"}],
    "preconditions": ["summary_available"],
    "postconditions": ["translation_complete"]
  }
]
```

2. Register three agents, each owning one capability and its MCP server.

3. The task graph becomes a linear three-node DAG:

```
extract_text -> summarize -> translate
```

4. Validation confirms: `raw_text` output type matches `summarize` input, `summary` output matches `translate` input. All preconditions chain correctly.

**Example 3: Retrofitting an Existing Agent System**

User: "My current agent system sometimes tries to call tools that don't exist. How do I fix this?"

Approach:
1. Audit every tool your agents currently call. For each, write a capability descriptor with explicit input/output types and pre/postconditions.
2. Build a capability registry (can be a JSON file, database, or MCP server).
3. Modify the agent's planning step to query the registry first, then construct plans only from registered capabilities.
4. Add a validation step between planning and execution that checks every node in the plan references a real capability with compatible types.
5. Reject plans that reference undeclared capabilities instead of attempting them.

## Best Practices

- **Do:** Declare every capability with typed inputs and outputs. Type mismatches between pipeline stages are the most common source of runtime failures.
- **Do:** Use preconditions and postconditions as a lightweight contract system. They enable static validation of execution order without running anything.
- **Do:** Keep the Agent Directory separate from capability definitions. Agents map to capabilities by reference, not by embedding. This avoids duplication when multiple agents share capabilities.
- **Do:** Validate task graphs before execution. A graph that passes validation is guaranteed to be executable given the current capability set.
- **Avoid:** Letting the LLM planner reference any operation not in the capability registry. The entire point of the declarative layer is to constrain planning to the verifiable operational space.
- **Avoid:** Embedding execution logic in capability descriptors. Descriptors declare *what* a capability does (interface), not *how* it does it (implementation). Keep these concerns separated.

## Error Handling

- **Missing capability:** If a user intent requires a capability not in the registry, report the gap explicitly rather than letting the LLM improvise. Return a structured error naming the missing capability.
- **Type mismatch in task graph:** If an output type doesn't match a downstream input type, fail at validation time with a message identifying the incompatible edge.
- **Agent unavailable:** If the Agent Directory shows an agent is offline, remove its capabilities from the index and re-plan. Do not attempt to invoke unreachable agents.
- **Precondition not met:** If a capability's precondition cannot be satisfied by any upstream postcondition in the graph, reject the plan and explain which precondition is unsatisfied.
- **Circular dependency:** If task graph construction produces a cycle, reject the graph. DALIA task graphs must be DAGs.

## Limitations

- DALIA adds upfront design cost: every capability must be formally declared before agents can use it. This is inappropriate for highly exploratory tasks where the set of needed operations is unknown.
- The architecture is best suited for structured, repeatable workflows. Open-ended creative tasks that benefit from free-form LLM reasoning are not the target use case.
- Precondition/postcondition checking is only as good as the declarations. If descriptors are incomplete or incorrect, validation provides false confidence.
- The federated directory adds infrastructure complexity. For systems with only 2-3 agents and a handful of tools, a simple hardcoded registry may be more practical.
- Dynamic capability negotiation (agents discovering new capabilities at runtime from unknown servers) is supported but requires periodic re-discovery, which may introduce latency.

## Reference

Rodriguez-Sanchez, M.J., Noguera, M., Ruiz-Zafra, A., & Benghazi, K. (2026). *Towards a Declarative Agentic Layer for Intelligent Agents in MCP-Based Server Ecosystems*. arXiv:2601.17435v1. Key sections: Section 3 (Capability Semantic Model and ATDP), Section 4 (Agent Directory MCP), Section 5 (Task Graph Construction), and the restaurant booking scenario walkthrough.