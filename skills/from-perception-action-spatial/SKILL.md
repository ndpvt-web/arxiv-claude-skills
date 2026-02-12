---
name: "from-perception-action-spatial"
description: "Design and implement spatially-aware AI agent systems using hierarchical memory, GNN-LLM integration, and world models. Use when: 'build a spatial reasoning agent', 'implement scene graph navigation', 'create a world model for robot planning', 'add spatial memory to an agent', 'integrate GNN with LLM for spatial tasks', 'build a hierarchical planner for embodied AI'."
---

# Spatial AI Agent Architecture: From Perception to Action

This skill enables Claude to design and implement spatially-aware AI agent systems grounded in the three-axis taxonomy from Felicia et al. (2026). The core insight: perception alone does not confer agency. Spatial intelligence requires metric understanding of geometry and physics — not just semantic labeling. This skill applies the paper's framework to build agents with hierarchical memory for long-horizon tasks, GNN-LLM integration for structured spatial reasoning, and world models for safe predictive planning across micro (grasping), meso (room navigation), and macro (city-scale) spatial scales.

## When to Use

- When the user asks to build an embodied AI agent that must navigate, manipulate, or reason about 3D space
- When implementing a scene graph representation for object-relationship reasoning
- When adding persistent spatial memory (metric maps, topological graphs, semantic overlays) to an LLM-based agent
- When integrating graph neural networks with language models for spatial tasks like navigation or manipulation planning
- When building a world model (encoder-dynamics-decoder) for safe lookahead planning in physical environments
- When designing a hierarchical planner that decomposes high-level language instructions into spatially-grounded action primitives
- When working on traffic prediction, urban computing, or geospatial analysis using spatio-temporal graph networks

## Key Technique

**The Three-Axis Taxonomy.** The paper organizes spatial AI along three orthogonal dimensions: the *Capability axis* (how agents reason — memory, planning, tool use), the *Task axis* (what they do — navigation, manipulation, scene understanding, geospatial analysis), and the *Scale axis* (spatial granularity — micro <1m, meso 1-100m, macro >100m). No existing method achieves strong performance across all three simultaneously. The actionable implication: when designing a spatial agent, explicitly choose your target on each axis and select representations accordingly. Micro-scale tasks demand explicit pose representations; macro-scale tasks require learned implicit embeddings; meso-scale tasks benefit from hybrid semantic maps.

**Hierarchical Memory + GNN-LLM Integration.** The paper identifies two architectural patterns that unlock long-horizon spatial reasoning. First, agents need four memory tiers: short-term (transformer context), working memory (multi-step scratchpad), episodic memory (event-specific experience replay), and spatial memory (persistent geometric maps or topological graphs). Second, GNN-LLM integration bridges symbolic LLM reasoning with relational spatial structure by encoding scenes as graphs (nodes = objects, edges = spatial predicates) and either serializing them for LLM consumption, querying them as external memory, or jointly embedding graph and language tokens via cross-attention.

**World Models for Safe Planning.** Rather than trial-and-error in the real world, world models compress observations into latent states (encoder: `z_t = q(z_t | o_≤t, a_<t)`), predict forward dynamics in latent space (dynamics: `z_{t+1} = p(z_{t+1} | z_t, a_t)`), and decode when verification is needed. This enables tree search over imagined trajectories — the agent simulates multiple action sequences and selects the one with highest predicted reward before executing anything physical. This pattern is essential for safety-critical deployment.

## Step-by-Step Workflow

1. **Classify the spatial scale.** Determine whether the task is micro (<1m, e.g., grasping), meso (1-100m, e.g., room navigation), or macro (>100m, e.g., traffic prediction). This dictates representation choices — explicit coordinates for micro, semantic grid maps for meso, learned graph embeddings for macro.

2. **Define the scene graph schema.** Create a typed graph structure where nodes represent entities (objects, rooms, intersections) with attribute vectors (position, class, state) and edges encode spatial predicates (supports, contains, adjacent_to, connects). Use a format like:
   ```python
   Node = {"id": str, "type": str, "pos": [x, y, z], "attrs": dict}
   Edge = {"source": str, "target": str, "relation": str, "weight": float}
   ```

3. **Implement the four-tier memory system.** Build distinct storage for: (a) short-term context (recent observations in the LLM prompt), (b) working memory (a scratchpad dict for intermediate spatial computations), (c) episodic memory (a buffer of (state, action, outcome) tuples for experience replay), and (d) spatial memory (a persistent map or graph updated incrementally as the agent perceives new regions).

4. **Build the GNN message-passing layer.** Implement neighbor aggregation over the scene graph to propagate spatial context:
   ```python
   def message_pass(graph, node_features, num_rounds=3):
       for _ in range(num_rounds):
           for node in graph.nodes:
               neighbor_msgs = [node_features[n] for n in graph.neighbors(node)]
               aggregated = mean(neighbor_msgs)
               node_features[node] = update_fn(node_features[node], aggregated)
       return node_features
   ```

5. **Serialize graph state for LLM consumption.** Convert the GNN-enriched graph into structured text the LLM can reason over. Use triplet format: `"Table supports Cup. Cup contains Coffee. Chair is_left_of Table."` Preserve spatial predicates explicitly — do not flatten to unstructured descriptions.

6. **Construct the world model pipeline.** Implement the three components: an encoder (compress observations to latent vectors via VAE or ResNet), a dynamics model (predict next latent state given current state + action via MLP or GRU), and a decoder (reconstruct observations for verification). Train on (observation, action, next_observation) tuples.

7. **Implement latent tree search for planning.** Use the dynamics model to simulate K candidate action sequences of horizon H in latent space. Score each trajectory using a learned reward predictor. Select the sequence with highest cumulative predicted reward. Execute only the first action, then re-plan (model-predictive control).

8. **Build the hierarchical planner.** Decompose high-level instructions via LLM into ordered subgoals, map each subgoal to a mid-level skill from a library, and execute skills with low-level collision-checked controllers:
   ```
   LLM("Make coffee") → ["go_to(kitchen)", "find(coffee_machine)", "operate(coffee_machine)"]
   Skill library maps each to motion primitives
   Low-level controller executes with spatial constraints
   ```

9. **Add spatial grounding validation.** Before executing any plan, check geometric feasibility: verify that referenced objects exist in the current spatial memory, that distances are physically traversable, and that spatial predicates are consistent (an object cannot be both inside and on_top_of the same container). Reject plans that violate metric constraints.

10. **Evaluate against scale-appropriate benchmarks.** Use R2R or ALFRED for meso-scale navigation, BEHAVIOR-1K for manipulation, and traffic datasets (METR-LA, PEMS-BAY) for macro-scale. Track success rate degradation as task horizon increases — this reveals memory and planning bottlenecks.

## Concrete Examples

**Example 1: Room-Scale Navigation Agent with Spatial Memory**

User: "Build an agent that can navigate a house floor plan to find objects based on natural language instructions."

Approach:
1. Parse the floor plan into a topological scene graph — rooms as nodes, doors as edges, objects as leaf nodes attached to rooms
2. Implement a semantic grid map (2D array where each cell stores an object class label) updated as the agent "explores"
3. Integrate with an LLM: serialize the current visible scene graph region as context, ask the LLM to select the next navigation waypoint
4. Store visited locations in episodic memory to avoid revisiting dead ends
5. Use the GNN layer to propagate "likely contains target" signals through the room graph based on co-occurrence priors (kitchens contain cups)

Output:
```python
class SpatialNavAgent:
    def __init__(self, floor_plan_graph, llm_client):
        self.scene_graph = floor_plan_graph          # topological graph
        self.semantic_map = np.zeros((H, W), dtype=int)  # spatial memory
        self.episodic_buffer = []                     # (location, action, result)
        self.llm = llm_client
        self.gnn = SceneGraphGNN(hidden_dim=64, num_layers=3)

    def navigate_to(self, target_description: str):
        # Enrich graph node features via GNN message passing
        enriched = self.gnn(self.scene_graph)
        # Serialize visible subgraph for LLM
        context = serialize_graph(enriched, self.current_room, radius=2)
        # LLM selects next waypoint
        plan = self.llm.query(
            f"You are at {self.current_room}. Find: {target_description}.\n"
            f"Nearby layout: {context}\n"
            f"Visited: {[e[0] for e in self.episodic_buffer]}\n"
            f"Choose next room to visit."
        )
        # Execute and record
        next_room = parse_room(plan)
        result = self.move_to(next_room)
        self.episodic_buffer.append((self.current_room, next_room, result))
        return result
```

**Example 2: Traffic Prediction with Spatio-Temporal GNN**

User: "Implement a traffic speed prediction system for a city road network that forecasts 15 minutes ahead."

Approach:
1. Model the road network as a graph — intersections as nodes, road segments as edges
2. Build a spatio-temporal GNN: spatial convolution via diffusion on the road graph, temporal convolution via 1D causal filters across time steps
3. Learn an adaptive adjacency matrix to capture non-local correlations (distant intersections sharing commuter patterns)
4. Train on historical speed data with (past 60 min → next 15 min) windows

Output:
```python
class SpatioTemporalTrafficGNN(nn.Module):
    def __init__(self, num_nodes, in_steps=12, out_steps=3):
        super().__init__()
        # Learnable adjacency: captures correlations beyond physical roads
        self.node_embed_src = nn.Parameter(torch.randn(num_nodes, 16))
        self.node_embed_tgt = nn.Parameter(torch.randn(num_nodes, 16))
        # Spatial diffusion convolution
        self.spatial_conv = DiffusionConv(in_channels=32, out_channels=32, K=2)
        # Temporal causal convolution
        self.temporal_conv = nn.Conv1d(32, 32, kernel_size=3, padding=2, dilation=2)
        self.output_layer = nn.Linear(32, out_steps)

    def learned_adjacency(self):
        return F.softmax(F.relu(self.node_embed_src @ self.node_embed_tgt.T), dim=-1)

    def forward(self, x):  # x: (batch, num_nodes, in_steps, features)
        adj = self.learned_adjacency()
        # Spatial: propagate traffic state across road graph
        h = self.spatial_conv(x, adj)  # (batch, nodes, time, channels)
        # Temporal: capture speed trends over time
        h = self.temporal_conv(h.permute(0,1,3,2)).permute(0,1,3,2)
        return self.output_layer(h[:, :, -1, :])  # predict future steps
```

**Example 3: World Model for Robot Manipulation Planning**

User: "Create a world model that lets a robot arm plan grasping sequences by imagining outcomes before acting."

Approach:
1. Encode workspace images into latent vectors via a convolutional encoder
2. Train a dynamics model (GRU) to predict next latent state given current state + proposed action
3. Implement latent tree search: simulate multiple grasp candidates, score by predicted success, execute the best
4. Decode top candidate to verify visual plausibility before committing

Output:
```python
class ManipulationWorldModel:
    def __init__(self, encoder, dynamics, decoder, reward_predictor):
        self.encode = encoder        # image → latent z
        self.dynamics = dynamics      # (z, action) → z_next
        self.decode = decoder         # z → reconstructed image
        self.reward = reward_predictor # z → scalar success score

    def plan(self, observation, candidate_actions, horizon=5):
        z_current = self.encode(observation)
        best_score, best_sequence = -float('inf'), None

        for action_sequence in candidate_actions:  # K candidate sequences
            z = z_current
            cumulative_reward = 0.0
            for t in range(min(horizon, len(action_sequence))):
                z = self.dynamics(z, action_sequence[t])
                cumulative_reward += self.reward(z)
            if cumulative_reward > best_score:
                best_score = cumulative_reward
                best_sequence = action_sequence

        # Verify: decode the predicted final state for visual sanity check
        z_final = z_current
        for a in best_sequence[:horizon]:
            z_final = self.dynamics(z_final, a)
        predicted_image = self.decode(z_final)
        return best_sequence[0], predicted_image  # execute first action only (MPC)
```

## Best Practices

- **Do:** Distinguish spatial grounding from symbolic grounding. An LLM describing a scene correctly does not mean it can navigate that scene. Always validate plans against metric constraints (distances, collision geometry, physical reachability).
- **Do:** Use topological graphs for meso-scale navigation and learned adjacency matrices for macro-scale prediction. Explicit metric maps become intractable at city scale; learned embeddings capture non-local correlations that physical adjacency misses.
- **Do:** Implement model-predictive control (plan, execute first action, re-plan) rather than open-loop execution. Long-horizon plans degrade rapidly — ALFRED success drops from 65% at 5 steps to 18% at 20 steps.
- **Do:** Store spatial memory separately from the LLM context. Scene graphs and metric maps should persist across reasoning steps and survive context window limitations.
- **Avoid:** Flattening scene graphs into unstructured natural language descriptions. Serialization must preserve relational structure — use explicit triplets, not prose paragraphs.
- **Avoid:** Training world models purely on reconstruction loss. Add contrastive objectives or reward prediction to ensure the latent space captures action-relevant features, not visual details irrelevant to planning.

## Error Handling

- **Spatial hallucination:** The LLM generates a plan referencing objects or spatial relationships that don't exist in the scene graph. Mitigation: validate every entity and predicate in the plan against current spatial memory before execution. Reject and re-prompt with the actual graph state.
- **Reference frame confusion:** Agent mixes egocentric ("turn left") and allocentric ("go north") coordinates. Mitigation: normalize all spatial instructions to a single canonical frame at the interface boundary. Include explicit frame labels in prompts.
- **Scale mismatch:** Agent applies micro-scale reasoning (exact coordinates) to macro-scale tasks or vice versa. Mitigation: gate representation choice on the classified spatial scale from step 1. Raise an error if the user requests centimeter precision for city-scale data.
- **Memory drift over long horizons:** Semantic maps accumulate errors as the agent explores (observed in VLMaps after 100+ steps). Mitigation: implement periodic map consistency checks — verify that recently observed regions match stored representations and correct discrepancies.
- **GNN over-smoothing:** Too many message-passing rounds collapse all node features to the same value. Mitigation: use 2-3 rounds maximum; add skip connections or use attention-based aggregation (Graph Transformer) for deeper architectures.

## Limitations

- This framework is a design methodology, not a drop-in library. It requires implementing each component (scene graph, GNN, memory system, world model) for your specific domain.
- World models trained in simulation may not transfer to real environments. The sim-to-real gap remains an open challenge — latent dynamics learned in simulation diverge from real physics.
- GNN-LLM serialization loses information. Even structured triplet formats cannot capture continuous spatial relationships (exact angles, distances) as precisely as the underlying graph representation.
- The hierarchical planner assumes a pre-defined skill library. If the required primitive skill doesn't exist, the system cannot improvise — it must fall back to lower-level control or fail gracefully.
- Multi-agent spatial coordination (multiple robots in a shared environment) is identified as a grand challenge but not solved by this framework. It requires extensions for decentralized planning and emergent communication.

## Reference

**Paper:** Felicia, G., Bryant, N., Putra, H., Gazali, A., & Lobo, E. (2026). "From Perception to Action: Spatial AI Agents and World Models." arXiv:2602.01644v1.
https://arxiv.org/abs/2602.01644v1

**What to look for:** The three-axis taxonomy (Section 3) for classifying your spatial task; the hierarchical memory architecture (Section 4) for designing agent memory; the GNN-LLM integration patterns (Section 5) for spatial reasoning; and the six grand challenges (Section 8) for understanding current limits.