---
name: "embodied-task-planning-graph-informed"
description: |
  Implement the GiG (Graph-in-Graph) framework for embodied task planning that uses graph-structured memory
  and bounded lookahead to decompose complex, long-horizon tasks into grounded action sequences.
  Applies graph neural network encoding, execution trace graphs, and symbolic lookahead to prevent
  hallucinated actions and maintain coherence across extended multi-step plans.
  Trigger phrases:
  - "plan a multi-step task with graph memory"
  - "build an embodied agent planner"
  - "implement graph-informed action planning"
  - "decompose a long-horizon task into grounded actions"
  - "create a task planner that avoids hallucinated steps"
  - "build a GiG-style planner for sequential tasks"
---

# Embodied Task Planning via Graph-Informed Action Generation (GiG)

This skill enables Claude to implement the GiG (Graph-in-Graph) planning framework from Li et al. (2026), which structures an agent's memory as a hierarchy of graphs — state graphs encoding environment snapshots nested inside execution trace graphs recording action-connected trajectories. By clustering graph embeddings and retrieving structure-aware priors, then applying bounded symbolic lookahead, the framework produces long-horizon plans that stay grounded in real environmental constraints instead of hallucinating invalid transitions. This technique applies to any domain where an LLM must generate a sequence of constrained actions: robotics task planning, workflow automation, game AI, CI/CD pipeline orchestration, or multi-step API call chains.

## When to Use

- When the user needs to build a planner that decomposes a high-level goal into a valid sequence of constrained actions (e.g., cooking recipes, assembly instructions, deployment pipelines)
- When implementing an agent that must reason over environment state and avoid invalid action sequences (robotics, game AI, workflow engines)
- When building a retrieval-augmented planning system that learns from past execution traces to improve future plans
- When the user wants to prevent LLM hallucination in sequential task planning by grounding actions in graph-structured state representations
- When orchestrating multi-step workflows where actions have preconditions and effects that must be tracked (CI/CD, data pipelines, infrastructure provisioning)
- When building systems where parallel and sequential sub-tasks must be coordinated without violating dependency constraints

## Key Technique

**Graph-in-Graph Architecture.** GiG organizes planning memory at two levels. The *inner graph* (state graph) represents a single environment snapshot: nodes are entities or objects (e.g., "tomato", "cutting_board", "pot") and edges are spatial/relational connections (e.g., "on", "inside", "adjacent_to"). A Graph Attention Network (GAT) encodes each state graph into a fixed-dimensional embedding that captures the structural configuration. The *outer graph* (execution trace graph) links these state embeddings sequentially: nodes are state embeddings and edges are the actions that caused transitions between them. This two-level representation lets the system reason about both "what the world looks like" and "what sequence of actions got us here."

**Experience Memory and Retrieval.** Completed execution traces are stored in a memory bank. Graph embeddings are clustered (enabling efficient nearest-neighbor lookup), so when the agent faces a new situation, it retrieves past trace segments where the environment had a structurally similar configuration. These retrieved priors are injected into the LLM prompt as concrete action-sequence examples, grounding the model's generation in patterns that actually worked before — not hypothetical chains.

**Bounded Lookahead with Symbolic Transitions.** Before committing to an action, the system simulates the next K steps using a symbolic transition model (precondition/effect rules). Each candidate action is projected forward: does it lead to a valid state? Does the resulting state move closer to the goal? This lookahead prunes infeasible or counterproductive actions before the LLM ever sees them, dramatically reducing hallucinated plans. The bounded depth (typically 2-4 steps) keeps computation tractable while catching most constraint violations.

## Step-by-Step Workflow

1. **Define the state graph schema.** Enumerate the entity types (nodes) and relationship types (edges) relevant to your domain. For a cooking domain: nodes are ingredients, tools, containers; edges are spatial relations like `on`, `inside`, `held_by`. For a CI/CD domain: nodes are services, artifacts, environments; edges are `depends_on`, `deployed_to`, `triggers`.

2. **Build the state graph constructor.** Write a function that takes a raw environment observation (JSON state, API response, sensor data) and produces a typed graph. Use `networkx` or a similar library. Each node gets a type label and attribute dict; each edge gets a relation label.

   ```python
   import networkx as nx

   def build_state_graph(observation: dict) -> nx.DiGraph:
       G = nx.DiGraph()
       for entity in observation["entities"]:
           G.add_node(entity["id"], type=entity["type"], **entity["attrs"])
       for rel in observation["relations"]:
           G.add_edge(rel["source"], rel["target"], relation=rel["type"])
       return G
   ```

3. **Encode state graphs with a GNN.** Implement a Graph Attention Network (or use PyTorch Geometric's `GATConv`) that takes a state graph and produces a fixed-size embedding vector. Train or fine-tune on your domain's state transitions so that structurally similar states map to nearby vectors.

   ```python
   from torch_geometric.nn import GATConv, global_mean_pool
   import torch.nn as nn

   class StateEncoder(nn.Module):
       def __init__(self, in_dim, hidden_dim, out_dim):
           super().__init__()
           self.conv1 = GATConv(in_dim, hidden_dim, heads=4, concat=False)
           self.conv2 = GATConv(hidden_dim, out_dim, heads=1)

       def forward(self, x, edge_index, batch):
           x = self.conv1(x, edge_index).relu()
           x = self.conv2(x, edge_index)
           return global_mean_pool(x, batch)  # single embedding per graph
   ```

4. **Construct execution trace graphs.** As the agent executes actions, record each (state_embedding, action, next_state_embedding) triple. Build a directed graph where nodes are state embeddings and edges are labeled with the action taken. Store completed traces in the experience memory bank.

   ```python
   class ExecutionTrace:
       def __init__(self):
           self.trace = nx.DiGraph()
           self.step = 0

       def record(self, state_emb, action, next_state_emb):
           self.trace.add_node(self.step, embedding=state_emb)
           self.trace.add_node(self.step + 1, embedding=next_state_emb)
           self.trace.add_edge(self.step, self.step + 1, action=action)
           self.step += 1
   ```

5. **Cluster and index the memory bank.** Use k-means or HDBSCAN on the state embeddings from all stored traces. Build a nearest-neighbor index (FAISS or annoy) over the embeddings so retrieval is sub-linear. Each cluster groups structurally similar environment configurations.

   ```python
   import faiss
   import numpy as np

   def build_memory_index(traces, embed_dim=128):
       all_embeddings = []
       metadata = []  # (trace_id, step, action_taken, subsequent_actions)
       for tid, trace in enumerate(traces):
           for node, data in trace.trace.nodes(data=True):
               all_embeddings.append(data["embedding"])
               successors = list(trace.trace.successors(node))
               actions = [trace.trace.edges[node, s]["action"] for s in successors]
               metadata.append({"trace_id": tid, "step": node, "actions": actions})
       index = faiss.IndexFlatIP(embed_dim)
       index.add(np.stack(all_embeddings).astype("float32"))
       return index, metadata
   ```

6. **Retrieve structure-aware priors.** Given the current state embedding, query the FAISS index for the top-K nearest neighbors. Extract the action subsequences that followed those similar states in past traces. These become the grounded examples for the LLM prompt.

7. **Define symbolic transition rules.** Encode each action's preconditions and effects as logical rules. For example: `cut(X)` requires `holding(knife) AND on(X, cutting_board)` and produces `is_cut(X)`. These rules power the lookahead module.

   ```python
   TRANSITIONS = {
       "cut": {
           "preconditions": lambda s: s.has("holding", "knife") and s.has_relation("on", "cutting_board"),
           "effects": lambda s, target: s.set_attr(target, "is_cut", True),
       },
       "pick_up": {
           "preconditions": lambda s: s.get("hands") == "empty",
           "effects": lambda s, target: s.set("holding", target),
       },
   }
   ```

8. **Implement bounded lookahead.** For each candidate action, simulate K steps forward using the symbolic rules. Score each trajectory by goal proximity (how many goal predicates are satisfied). Prune actions that lead to dead-end states or violate preconditions within the lookahead window.

   ```python
   def bounded_lookahead(current_state, candidates, transitions, goal, depth=3):
       scored = []
       for action in candidates:
           if not transitions[action]["preconditions"](current_state):
               continue
           sim_state = transitions[action]["effects"](current_state.copy(), action.target)
           score = goal_proximity(sim_state, goal)
           # Recurse for deeper lookahead
           if depth > 1:
               sub_candidates = get_valid_actions(sim_state, transitions)
               _, best_sub_score = bounded_lookahead(sim_state, sub_candidates, transitions, goal, depth - 1)
               score = max(score, best_sub_score)
           scored.append((action, score))
       scored.sort(key=lambda x: -x[1])
       return scored[0] if scored else (None, 0)
   ```

9. **Compose the LLM prompt.** Combine: (a) the current state description, (b) the retrieved prior action sequences from memory, (c) the lookahead-pruned candidate actions, and (d) the high-level goal. Ask the LLM to select the next action from the pruned candidates, explaining its reasoning.

   ```
   System: You are an embodied planning agent. Choose the next action from
   the valid candidates. Ground your choice in the retrieved examples and
   current state.

   Current state: [serialized state graph]
   Goal: [goal description]
   Retrieved similar situations and actions taken:
     - [prior 1: state description -> action sequence that succeeded]
     - [prior 2: state description -> action sequence that succeeded]
   Valid next actions (after lookahead pruning):
     - [action A] (lookahead score: 0.85)
     - [action B] (lookahead score: 0.72)

   Select the best next action and explain why.
   ```

10. **Execute, observe, and loop.** Execute the chosen action, observe the new environment state, build its state graph, encode it, record the trace step, and repeat from step 6 until the goal is satisfied or a maximum horizon is reached.

## Concrete Examples

**Example 1: Cooking Task Planner (Robotouille-style)**

User: "Build a planner that generates a step-by-step plan for making a salad given available ingredients and tools."

Approach:
1. Define state graph schema: nodes = `{tomato, lettuce, knife, cutting_board, bowl, plate}`, edges = spatial relations
2. Build initial state graph from kitchen observation: `tomato:on(counter)`, `knife:on(rack)`, `bowl:on(counter)`
3. Encode state with GAT, retrieve similar past traces (e.g., past salad assemblies)
4. Symbolic rules: `pick_up(knife)` requires `hands_empty`, `cut(tomato)` requires `holding(knife) AND on(tomato, cutting_board)`
5. Lookahead prunes `cut(tomato)` until knife is picked up and tomato is on the board
6. LLM selects from valid candidates: `pick_up(knife)` first, then `move(tomato, cutting_board)`, then `cut(tomato)`, etc.

Output:
```
Plan:
  1. pick_up(knife)           # hands_empty -> holding(knife)
  2. move(tomato, cutting_board)  # tomato on counter -> on cutting_board
  3. cut(tomato)              # holding(knife) AND on(tomato, cutting_board) -> is_cut(tomato)
  4. put_down(knife)          # free hands for next step
  5. pick_up(cut_tomato)
  6. place_in(cut_tomato, bowl)
  7. pick_up(lettuce)
  8. tear(lettuce)            # holding(lettuce) -> is_torn(lettuce)
  9. place_in(torn_lettuce, bowl)
  10. serve(bowl, plate)

Status: All goal predicates satisfied. Plan valid.
```

**Example 2: CI/CD Pipeline Orchestration**

User: "Implement a deployment planner that sequences build, test, and deploy steps across microservices with dependency constraints."

Approach:
1. State graph schema: nodes = `{service_A, service_B, database, staging_env, prod_env}`, edges = `depends_on`, `deployed_to`
2. Transition rules: `deploy(X, prod)` requires `tests_passed(X) AND all_deps_deployed(X)`
3. Memory bank stores past deployment traces (successful rollouts)
4. Retrieve priors: past deployments of similar service graphs
5. Lookahead catches constraint violations: deploying service_A before its dependency (database) would fail

Output:
```python
planner = GiGPlanner(
    state_schema=ServiceGraphSchema(),
    transitions=DeploymentTransitions(),
    memory=DeploymentMemoryBank(past_traces),
    lookahead_depth=3,
)

plan = planner.generate_plan(
    initial_state=current_cluster_state(),
    goal=Goal("all services on prod, tests passing"),
)
# Result:
# 1. run_tests(database_migration)
# 2. deploy(database, staging)
# 3. run_tests(service_B)
# 4. deploy(service_B, staging)
# 5. run_tests(service_A)     # depends on service_B
# 6. deploy(service_A, staging)
# 7. integration_test(staging)
# 8. promote(staging, prod)
```

**Example 3: Game AI Task Decomposition (ALFWorld-style)**

User: "Build a household task agent that finds and heats an apple in a simulated kitchen."

Approach:
1. State graph: nodes are rooms, containers, objects; edges are `contains`, `connected_to`
2. Goal: `heated(apple) AND on(apple, counter)`
3. Retrieve past traces where agent found and heated objects
4. Symbolic transitions: `open(fridge)` requires `at(fridge)`, `take(apple)` requires `open(fridge) AND in(apple, fridge)`
5. Lookahead prevents searching containers the apple isn't in (using prior knowledge from memory retrieval)

Output:
```
> think: I need to find an apple and heat it. Memory shows apples are usually in the fridge.
> go_to(kitchen)
> go_to(fridge)
> open(fridge)
> take(apple, fridge)
> go_to(microwave)
> open(microwave)
> put(apple, microwave)
> close(microwave)
> heat(microwave)
> open(microwave)
> take(apple, microwave)
> go_to(counter)
> put(apple, counter)
Task completed: heated apple placed on counter.
```

## Best Practices

- **Do:** Define preconditions and effects for every action in your domain exhaustively. Missing a single precondition defeats the purpose of symbolic lookahead — the system will think invalid actions are valid.
- **Do:** Start with a shallow lookahead depth (2-3) and increase only if you observe the agent making locally greedy but globally suboptimal choices. Deeper lookahead is exponentially more expensive.
- **Do:** Periodically retrain or update the GNN encoder as new execution traces accumulate. State embeddings should evolve as the agent encounters more diverse situations.
- **Do:** Serialize state graphs to a canonical form before encoding so that isomorphic states produce identical embeddings regardless of node insertion order.
- **Avoid:** Encoding raw text observations directly as graph nodes. Always parse observations into typed, structured entities first. Unstructured text defeats the purpose of graph-structured memory.
- **Avoid:** Storing every execution trace indefinitely. Prune traces that led to failures or are redundant with existing high-quality traces. Memory quality matters more than quantity.

## Error Handling

| Problem | Symptom | Fix |
|---|---|---|
| No valid actions after lookahead | Lookahead prunes all candidates | Reduce lookahead depth by 1, or allow the highest-scoring "partial" action. Check if transition rules are overly restrictive. |
| Retrieval returns irrelevant priors | Agent takes actions that don't match current context | Increase the number of clusters, retrain the GNN on more diverse traces, or raise the similarity threshold for retrieval. |
| Plan loops (repeated state visits) | Agent cycles between two states | Add a visited-state penalty to the lookahead scoring. Track the last N state embeddings and penalize candidates that lead back to them. |
| State graph too large to encode | GNN OOM or slow inference | Prune the state graph to only entities relevant to the current goal. Use subgraph extraction based on goal predicates. |
| LLM ignores retrieved priors | Generated actions don't reflect memory | Move retrieved examples closer to the end of the prompt (recency bias). Reduce the number of priors to 2-3 high-quality matches. |

## Limitations

- **Requires a symbolic transition model.** The bounded lookahead depends on having precondition/effect rules for every action. Domains where actions have continuous, hard-to-model effects (e.g., physics simulations) are poor fits without heavy approximation.
- **GNN training needs data.** The state encoder requires a corpus of execution traces to train on. Cold-start scenarios with no historical data fall back to pure LLM planning without the retrieval or embedding benefits.
- **Scalability of state graphs.** Environments with hundreds of entities produce large graphs that are expensive to encode. The technique works best for environments with 10-100 discrete entities.
- **Static transition rules.** The symbolic rules don't adapt to novel actions not in the rule set. Domains where the action space changes dynamically (open-ended tool use) require rule-generation as an additional step.
- **Clustering granularity trade-off.** Too few clusters conflate dissimilar states; too many make retrieval fragile. Requires tuning per domain.

## Reference

Li, X., Yan, N., & Mortazavi, M. (2026). *Embodied Task Planning via Graph-Informed Action Generation with Large Language Model.* arXiv:2601.21841v1. [https://arxiv.org/abs/2601.21841v1](https://arxiv.org/abs/2601.21841v1)

Read for: The Graph-in-Graph architecture (Section 3), the bounded lookahead algorithm (Section 3.3), and the Robotouille/ALFWorld benchmark results showing 15-37% gains over baselines (Section 5).