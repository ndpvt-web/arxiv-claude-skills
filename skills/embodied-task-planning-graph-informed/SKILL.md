---
name: "embodied-task-planning-graph-informed"
description: "Structure long-horizon task planning using graph-based memory and bounded lookahead. Use when asked to: 'plan a multi-step agent workflow', 'build a task planner for a robot or agent', 'decompose complex goals into grounded actions', 'implement graph-based planning memory', 'create an agent that learns from past executions', 'build a planning system that avoids hallucinated actions'."
---

# Graph-Informed Action Generation for Embodied Task Planning (GiG)

This skill enables Claude to design and implement long-horizon task planning systems for embodied or software agents using the GiG (Graph-in-Graph) architecture. The core idea: instead of relying on an LLM to plan purely from its context window, you encode environment states as graphs using a GNN, store past execution traces as action-connected graphs in a memory bank, cluster these graph embeddings for fast retrieval, and use a bounded lookahead module with symbolic transition logic to validate actions before committing. This produces planners that decompose high-level goals into executable sub-goals while respecting real environmental constraints.

## When to Use

- When the user needs to build a multi-step task planner for a robot, game agent, or autonomous software agent
- When implementing an agent that must plan over long horizons (10+ sequential actions) without losing coherence
- When building a system that stores and retrieves past execution experiences to inform new plans
- When the user wants to prevent an LLM-based planner from hallucinating invalid actions or impossible state transitions
- When designing a planning framework that must respect dynamic environmental constraints (object availability, spatial relationships, preconditions)
- When creating a cooking, assembly, household, or logistics automation pipeline that chains dependent sub-tasks
- When the user asks to implement graph-based memory or experience replay for an agent

## Key Technique

**The Problem.** Standard LLM planners generate action sequences through open-ended text completion. Over long horizons this fails: the model forgets earlier state, hallucinates actions that violate physical constraints (e.g., picking up an object that hasn't been placed yet), and loses strategic coherence as the plan grows beyond the context window.

**The GiG Solution.** GiG introduces a Graph-in-Graph architecture with three interlocking components. First, a Graph Neural Network encodes each observed environment state as an embedding, where nodes represent entities (objects, agent state, locations) and edges capture spatial or functional relationships between them. Second, these state embeddings are organized into execution trace graphs -- directed acyclic graphs where nodes are state embeddings and edges are the actions that caused transitions between states. These trace graphs are stored in an experience memory bank and clustered by structural similarity, so when the agent encounters a new task, it retrieves the most structurally relevant past traces to use as priors. Third, a bounded lookahead module applies symbolic transition logic (PDDL-style precondition/effect checking) to project the consequences of candidate actions a few steps ahead, pruning infeasible branches before the LLM commits to a plan.

**Why It Works.** The graph encoding forces the planner to reason about structure -- which objects exist, how they relate, what actions are available -- rather than relying on token-level pattern matching. The experience memory bank provides few-shot exemplars that are structurally matched to the current situation, not just textually similar. The bounded lookahead prevents cascading errors by catching invalid transitions early. Together, these yield up to 22-37% improvement in task completion on standard benchmarks.

## Step-by-Step Workflow

1. **Define the environment state schema as a graph.** Identify the entity types (objects, locations, agent properties) that become nodes and the relationship types (spatial proximity, containment, functional dependency) that become edges. Represent this as an adjacency list or edge list with typed, attributed nodes.

2. **Implement a GNN encoder for state embeddings.** Use a 2-3 layer Graph Attention Network (GAT) or GraphSAGE model to encode each state graph into a fixed-dimensional embedding vector. Node features should include entity type, current properties (e.g., `is_cooked`, `is_held`), and any relevant numeric attributes. Edge features encode relationship type and strength.

3. **Define the action space with preconditions and effects.** For each action the agent can take (e.g., `pick_up(X)`, `cook(X, Y)`, `move_to(L)`), specify preconditions (predicates that must be true in the current state) and effects (predicates that change after execution). Use a PDDL-like format or a Python dictionary mapping actions to their pre/post conditions.

4. **Build the execution trace graph structure.** Record each episode as a directed acyclic graph where nodes are GNN-encoded state embeddings and edges are labeled with the action taken. Store the full trace (initial state through goal state) along with metadata: task description, success/failure, total steps.

5. **Populate and cluster the experience memory bank.** After collecting execution traces (from demonstrations, rollouts, or synthetic generation), cluster their graph embeddings using k-means or spectral clustering on the initial-state embeddings. Index clusters for O(1) lookup by maintaining a centroid-to-traces mapping.

6. **Implement the retrieval mechanism.** Given a new task, encode the current environment state with the GNN, find the nearest cluster centroid, and retrieve the top-k most similar execution traces from that cluster. Rank by cosine similarity between state embeddings. These become the structural priors passed to the LLM.

7. **Build the bounded lookahead module.** Implement a forward search that, given the current state and a candidate action sequence, symbolically applies precondition/effect rules to simulate the next 2-4 state transitions. Reject any candidate sequence where a precondition fails. Return the set of feasible action prefixes.

8. **Construct the LLM prompt with graph-grounded context.** Format the prompt with: (a) the current state description derived from the graph, (b) the retrieved execution traces as structured examples, (c) the set of currently feasible actions from the lookahead module, and (d) the high-level goal. Instruct the LLM to select the next action from the feasible set.

9. **Execute, observe, and update.** Execute the chosen action, observe the new environment state, re-encode with the GNN, and repeat from step 6. After the episode ends, store the new execution trace in the memory bank and re-cluster periodically.

10. **Evaluate with ablation.** Test each component independently: GNN encoding vs. flat state, memory retrieval vs. no retrieval, bounded lookahead vs. unconstrained generation. Measure Pass@1 (first-attempt success rate) and average plan length.

## Concrete Examples

**Example 1: Cooking Task Planner (Robotouille-style)**

User: "Build a task planner that can execute multi-step cooking recipes where ingredients must be prepared in the right order."

Approach:
1. Define state graph: nodes = `{ingredient: {name, state: [raw/chopped/cooked]}, station: {type, occupied_by}, agent: {holding, location}}`; edges = `at(ingredient, station)`, `near(agent, station)`.
2. Define actions with preconditions:
```python
actions = {
    "pick_up": {
        "params": ["ingredient"],
        "preconditions": ["agent.holding == None", "ingredient.location == agent.location"],
        "effects": ["agent.holding = ingredient", "ingredient.location = None"]
    },
    "chop": {
        "params": ["ingredient"],
        "preconditions": ["agent.holding == ingredient", "agent.location == cutting_board", "ingredient.state == 'raw'"],
        "effects": ["ingredient.state = 'chopped'"]
    },
    "cook": {
        "params": ["ingredient"],
        "preconditions": ["agent.holding == ingredient", "agent.location == stove", "ingredient.state == 'chopped'"],
        "effects": ["ingredient.state = 'cooked'"]
    }
}
```
3. Encode current kitchen state via GNN, retrieve 3 most similar past cooking traces, run 3-step lookahead to filter infeasible next actions, prompt LLM with feasible actions and retrieved traces.

Output:
```
State: tomato(raw) at counter, lettuce(raw) at counter, agent at counter, holding nothing
Goal: serve(salad) -- requires tomato(chopped), lettuce(chopped), both plated

Retrieved trace (similarity=0.94): [pick_up(tomato) -> move_to(board) -> chop(tomato) -> place(tomato, plate) -> move_to(counter) -> pick_up(lettuce) -> move_to(board) -> chop(lettuce) -> place(lettuce, plate) -> serve(plate)]

Feasible next actions (lookahead verified): [pick_up(tomato), pick_up(lettuce)]
LLM selects: pick_up(tomato)  # follows retrieved trace pattern
```

**Example 2: Household Robot (ALFWorld-style)**

User: "Implement a planner for a household robot that must find objects, move them, and interact with appliances to complete tasks like 'heat the mug and put it on the shelf'."

Approach:
1. State graph nodes: `{object: {type, location, temperature}, receptacle: {type, contains[]}, agent: {location, holding}}`. Edges: `inside(obj, receptacle)`, `accessible(agent, receptacle)`.
2. Experience memory seeded with 50 demonstration traces from simple household tasks, clustered into 8 groups (kitchen-tasks, cleaning-tasks, organization-tasks, etc.).
3. Bounded lookahead validates: can't heat what you're not holding, can't open what's already open, can't put in a full receptacle.

Output:
```
Task: "Heat the mug and put it on the shelf"
State: mug in cabinet_2, shelf_1 empty, microwave closed

Plan generated:
1. go_to(cabinet_2)        # precond: agent can navigate -> OK
2. open(cabinet_2)         # precond: cabinet_2 is closed -> OK
3. take(mug, cabinet_2)    # precond: mug in cabinet_2, agent.holding==None -> OK
4. go_to(microwave)        # precond: agent can navigate -> OK
5. open(microwave)         # precond: microwave is closed -> OK
6. heat(mug, microwave)    # precond: agent.holding==mug, microwave is open -> OK
7. go_to(shelf_1)          # precond: agent can navigate -> OK
8. put(mug, shelf_1)       # precond: agent.holding==mug, shelf_1 has space -> OK

All 8 steps verified by bounded lookahead. No precondition violations.
```

**Example 3: Software Deployment Pipeline as Embodied Planning**

User: "Apply this graph-based planning approach to orchestrate a multi-service deployment where services have dependency ordering and health-check constraints."

Approach:
1. State graph: nodes = `{service: {name, version, status: [stopped/deploying/healthy/unhealthy]}, dependency: {from, to}}`. Edges = `depends_on(service_a, service_b)`, `shares_resource(service_a, service_b)`.
2. Actions: `deploy(service)` (precond: all dependencies healthy), `rollback(service)` (precond: service is unhealthy), `health_check(service)` (precond: service is deploying).
3. Memory bank stores past deployment traces with outcomes, clustered by topology similarity (which services were involved, dependency shape).
4. Lookahead prevents deploying a service before its dependencies are healthy.

Output:
```python
# Graph-informed deployment planner
state_graph = {
    "nodes": [
        {"id": "db", "status": "healthy", "version": "3.1"},
        {"id": "cache", "status": "healthy", "version": "2.0"},
        {"id": "api", "status": "stopped", "version": "1.4"},  # target: deploy v1.5
        {"id": "web", "status": "healthy", "version": "4.2"},  # target: deploy v4.3
    ],
    "edges": [
        {"type": "depends_on", "from": "api", "to": "db"},
        {"type": "depends_on", "from": "api", "to": "cache"},
        {"type": "depends_on", "from": "web", "to": "api"},
    ]
}

# Lookahead result: deploy(web) before deploy(api) -> REJECTED (api not healthy)
# Lookahead result: deploy(api) then health_check(api) then deploy(web) -> VALID
# Retrieved similar trace: [deploy(api) -> health_check(api) -> deploy(web) -> health_check(web)]
# Plan follows retrieved trace with lookahead validation
```

## Best Practices

- **Do:** Define preconditions and effects exhaustively for every action. Missing a precondition is the most common source of invalid plans. Use unit tests that attempt illegal transitions to verify completeness.
- **Do:** Seed the experience memory bank with at least 20-50 diverse traces before expecting good retrieval. Cold-start with synthetic or hand-crafted traces if real executions are unavailable.
- **Do:** Keep the GNN shallow (2-3 layers) and the embedding dimension moderate (64-128). Deep GNNs over-smooth node features, losing the structural distinctions you need for clustering.
- **Do:** Re-cluster the memory bank periodically (every 50-100 new traces) to keep centroids representative as the distribution of experiences shifts.
- **Avoid:** Encoding raw text descriptions as graph node features. Always parse into structured typed attributes first. Text features defeat the purpose of graph-structured reasoning.
- **Avoid:** Setting the lookahead depth too high (>5 steps). The branching factor makes deep lookahead exponentially expensive. 2-4 steps catches most immediate constraint violations at practical cost.
- **Avoid:** Passing the entire retrieved trace verbatim to the LLM. Summarize the trace as a structured action sequence with key state checkpoints to stay within context limits.

## Error Handling

- **GNN encoding fails on unseen entity types:** Maintain a fallback embedding (mean of all known entity type embeddings) for novel objects. Log the unknown type and add it to the schema for future runs.
- **No similar traces found in memory bank:** When the nearest cluster centroid has cosine similarity below 0.5, fall back to unconstrained LLM planning but still enforce the bounded lookahead. Flag this as a coverage gap.
- **Lookahead rejects all candidate actions:** This indicates either a dead-end state or incomplete action definitions. Expand the action space check, attempt backtracking to the last state with valid actions, or escalate to the user with the current state description and the set of failed preconditions.
- **Plan diverges from retrieved trace mid-execution:** Re-run retrieval from the current (diverged) state to find a new matching trace segment rather than forcing alignment with the original trace. The memory bank should be consulted at every step, not just at the start.
- **Clustering produces degenerate groups:** If any cluster contains >60% of all traces, increase k or switch to hierarchical clustering. Overly broad clusters degrade retrieval precision.

## Limitations

- Requires a well-defined state schema and action space. Environments with continuous, unstructured observations (raw pixel input without an object detector) need an additional perception layer before GiG applies.
- The symbolic transition logic (preconditions/effects) must be manually authored or learned separately. GiG does not learn these rules from data -- it consumes them.
- Cold-start performance depends entirely on the quality of seed traces. Poor or unrepresentative seed demonstrations will produce misleading retrievals.
- The approach assumes discrete, turn-based action execution. Real-time continuous control (e.g., torque-level robot control) is out of scope without a discretization layer.
- Memory and compute scale with the number of stored traces and graph sizes. For environments with thousands of entities, the GNN encoding and clustering steps may need optimization (subgraph sampling, approximate nearest neighbor search).

## Reference

**Paper:** [Embodied Task Planning via Graph-Informed Action Generation with Large Language Models](https://arxiv.org/abs/2601.21841v1) (Li, Yan, Mortazavi, 2026). Look for: the Graph-in-Graph architecture diagram (Figure 2), the bounded lookahead algorithm (Algorithm 1), and the ablation study showing each component's individual contribution to Pass@1 scores on Robotouille and ALFWorld.