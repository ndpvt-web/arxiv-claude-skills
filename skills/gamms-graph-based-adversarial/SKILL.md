---
name: "gamms-graph-based-adversarial"
description: >
  Build and run graph-based multi-agent adversarial simulations using the GAMMS framework.
  Covers agent creation, graph environments (including OpenStreetMap road networks),
  sensor configuration, adversarial rules, potential-field strategies, recording/replay,
  and integration with ML libraries. Trigger phrases:
  "simulate multi-agent on a graph", "GAMMS simulation", "adversarial agent modeling",
  "graph-based agent environment", "multi-agent road network simulation",
  "build a capture-the-flag simulation"
---

# GAMMS: Graph-Based Adversarial Multiagent Modeling Simulator

This skill enables Claude to scaffold, implement, and iterate on multi-agent simulations using the GAMMS framework -- a lightweight Python library where environments are represented as graphs and agents traverse nodes and edges. GAMMS is policy-agnostic (heuristic, optimization, RL, or LLM-driven agents all work), provides built-in pygame visualization, supports OpenStreetMap ingestion for real-world road networks, and runs on standard hardware. Use this skill whenever a user needs to prototype adversarial or cooperative multi-agent scenarios on graph-structured worlds.

## When to Use

- When the user asks to **simulate multiple agents competing or cooperating on a network** (road network, communication graph, grid world, supply chain)
- When the user wants to **load a real-world map from OpenStreetMap** and run agents on it
- When the user needs to **prototype a capture-the-flag, pursuit-evasion, or territorial control game** with configurable rules
- When the user asks to **benchmark different agent policies** (heuristic vs. learned) in a graph environment
- When the user wants **recording and replay** of multi-agent simulations for post-hoc analysis
- When the user asks to **integrate external ML models or planning solvers** with a lightweight simulation loop
- When the user wants to **visualize agent movement** on a graph with minimal setup

## Key Technique

GAMMS represents every environment as a graph of nodes (locations with `id`, `x`, `y` coordinates) and edges (connections with `id`, `source`, `target`, `length`). This abstraction means any domain expressible as a network -- city streets, communication topologies, game boards, logistics routes -- can be simulated with the same core API. Agents live on nodes, perceive their surroundings through registered sensors, choose an adjacent node as their action, and advance each simulation step.

The framework's adversarial modeling power comes from its rule system. Rules are plain Python functions executed each step that inspect agent states (positions, teams, scores) and mutate the simulation accordingly -- tagging opponents, scoring captures, triggering resets. Because rules are user-defined functions rather than hardcoded mechanics, GAMMS supports arbitrary game semantics without framework modifications. This is the critical architectural insight: the simulation loop is a thin orchestrator; all domain logic lives in composable rule functions and agent policies.

Integration is first-class. Since GAMMS agents receive state dictionaries and return actions, any external system -- a PyTorch policy network, an LLM API call, a MILP solver -- can drive agent decisions. The recording subsystem serializes differential state per step into compact `.ggr` files, enabling headless batch runs followed by visual replay, which is essential for training pipelines where thousands of episodes run without rendering.

## Step-by-Step Workflow

1. **Install GAMMS** and verify the import:
   ```bash
   pip install gamms
   ```
   ```python
   import gamms
   print(gamms.__version__)
   ```

2. **Create a simulation context** with a visualization engine (or `NO_VIS` for headless):
   ```python
   ctx = gamms.create_context(vis_engine=gamms.visual.Engine.PYGAME)
   graph = ctx.graph.graph
   ```

3. **Build or load the graph environment**:
   - **Programmatic grid**: Add nodes with `graph.add_node({'id': i, 'x': x, 'y': y})` and edges with `graph.add_edge({'id': j, 'source': a, 'target': b, 'length': d})` in a loop.
   - **OpenStreetMap**: Use `gamms.osm.create_osm_graph(location_string, resolution, tolerance, bidirectional)` or `gamms.osm.graph_from_xml(path)` for real-world road networks.

4. **Configure visualization** for the graph:
   ```python
   ctx.visual.set_graph_visual(width=1920, height=1080)
   ```

5. **Create agents** with names, starting nodes, and team metadata, then assign visual properties:
   ```python
   ctx.agent.create_agent(name='red_0', start_node_id=0)
   ctx.visual.set_agent_visual(name='red_0', color=(255, 0, 0), size=10)
   ```

6. **Register sensors** so agents perceive their surroundings:
   ```python
   ctx.sensor.create_sensor(sensor_id='arc_0', sensor_type=gamms.sensor.SensorType.ARC)
   agent = ctx.agent.get_agent('red_0')
   agent.register_sensor(name='arc_0', sensor=ctx.sensor.get_sensor('arc_0'))
   ```
   Sensor types: `NEIGHBOR` (adjacent nodes), `ARC` (environment features in range/FOV), `AGENT_ARC` (other agents in range/FOV), or `CUSTOM`.

7. **Implement agent policies** as functions that read `agent.get_state()` and set `state['action']` to a neighbor node ID:
   ```python
   def potential_field_policy(agent, ctx):
       state = agent.get_state()
       neighbors = agent.get_sensor('neigh').data
       best = max(neighbors, key=lambda n: compute_potential(n, ctx))
       state['action'] = best
       agent.set_state()
   ```

8. **Define game rules** as functions that run each step (tag, capture, score, terminate):
   ```python
   def tag_rule(ctx, agents):
       for a in agents['red']:
           for b in agents['blue']:
               if a.current_node_id == b.current_node_id:
                   b.current_node_id = b.start_node_id  # reset
   ```

9. **Run the simulation loop**:
   ```python
   step = 0
   while not ctx.is_terminated():
       step += 1
       for agent in ctx.agent.create_iter():
           potential_field_policy(agent, ctx)
       tag_rule(ctx, agents_by_team)
       if step >= MAX_STEPS:
           ctx.terminate()
       ctx.visual.simulate()
   ```

10. **Record and replay** for analysis:
    ```python
    # During simulation
    ctx.record.start(path="my_run")
    # ... simulation loop ...
    ctx.record.stop()

    # Later, replay
    replay_ctx = gamms.create_context(
        vis_engine=gamms.visual.Engine.PYGAME,
        vis_kwargs={'simulation_time_constant': 0.3}
    )
    for _ in replay_ctx.record.replay("my_run"):
        continue
    replay_ctx.terminate()
    ```

## Concrete Examples

**Example 1: Grid-world capture-the-flag**

User: "Build a 10x10 grid where two teams of 3 agents play capture-the-flag. Red starts top-left, blue starts bottom-right. If an agent reaches the opponent's start, their team scores."

Approach:
1. Create context and build a 10x10 grid (100 nodes, bidirectional edges).
2. Place red agents at nodes (0,0), (0,1), (1,0) and blue agents at nodes (9,9), (9,8), (8,9).
3. Register `NEIGHBOR` sensors on all agents.
4. Implement a greedy shortest-path policy toward the opponent start zone.
5. Add a capture rule: if any red agent reaches node (9,9) area, increment red score and reset that agent.
6. Add a tag rule: if agents from opposing teams share a node, the intruder resets.
7. Run for 200 steps with pygame visualization.

Output structure:
```
project/
  config.py        # Grid size, team rosters, max steps, colors
  agents.py        # Policy functions (greedy, potential-field)
  rules.py         # tag_rule(), capture_rule(), termination_rule()
  main.py          # Context setup, graph construction, simulation loop
```

Key code in `main.py`:
```python
import gamms
from config import GRID_N, MAX_STEPS, RED_AGENTS, BLUE_AGENTS
from agents import greedy_policy
from rules import tag_rule, capture_rule

ctx = gamms.create_context(vis_engine=gamms.visual.Engine.PYGAME)
graph = ctx.graph.graph

# Build grid
node_id = 0
for y in range(GRID_N):
    for x in range(GRID_N):
        graph.add_node({'id': node_id, 'x': x * 50, 'y': y * 50})
        node_id += 1

edge_id = 0
for y in range(GRID_N):
    for x in range(GRID_N):
        nid = y * GRID_N + x
        if x + 1 < GRID_N:
            graph.add_edge({'id': edge_id, 'source': nid, 'target': nid + 1, 'length': 1.0})
            edge_id += 1
            graph.add_edge({'id': edge_id, 'source': nid + 1, 'target': nid, 'length': 1.0})
            edge_id += 1
        if y + 1 < GRID_N:
            graph.add_edge({'id': edge_id, 'source': nid, 'target': nid + GRID_N, 'length': 1.0})
            edge_id += 1
            graph.add_edge({'id': edge_id, 'source': nid + GRID_N, 'target': nid, 'length': 1.0})
            edge_id += 1

ctx.visual.set_graph_visual(width=800, height=800)

for name, start in RED_AGENTS.items():
    ctx.agent.create_agent(name=name, start_node_id=start)
    ctx.visual.set_agent_visual(name=name, color=(255, 0, 0), size=12)

for name, start in BLUE_AGENTS.items():
    ctx.agent.create_agent(name=name, start_node_id=start)
    ctx.visual.set_agent_visual(name=name, color=(0, 0, 255), size=12)

step = 0
while not ctx.is_terminated():
    step += 1
    for agent in ctx.agent.create_iter():
        greedy_policy(agent, ctx)
    tag_rule(ctx)
    capture_rule(ctx)
    if step >= MAX_STEPS:
        ctx.terminate()
    ctx.visual.simulate()
```

---

**Example 2: Real-world pursuit-evasion on San Diego streets**

User: "Load the La Jolla road network from OpenStreetMap and simulate a pursuit-evasion game with 2 pursuers and 1 evader."

Approach:
1. Load the OSM graph with `gamms.osm.graph_from_xml('la_jolla.osm')` at 50m resolution.
2. Create 2 pursuer agents (green) and 1 evader agent (red) at distant nodes.
3. Register `ARC` sensors (range=250m, fov=1.0 radian) and `AGENT_ARC` sensors on all agents.
4. Pursuer policy: move toward the neighbor node closest to the evader's last known position (from sensor data).
5. Evader policy: potential-field repulsion from detected pursuers, attraction toward open graph regions.
6. Capture rule: if a pursuer shares a node with the evader, terminate with "captured" status.
7. Enable recording for batch analysis of capture times.

Key code snippet:
```python
ctx = gamms.create_context(vis_engine=gamms.visual.Engine.PYGAME)
gamms.osm.graph_from_xml('la_jolla.osm', ctx=ctx, resolution=50, tolerance=10, bidirectional=True)
ctx.visual.set_graph_visual(width=1920, height=1080)

ctx.record.start(path="pursuit_run_001")
# ... agent creation, sensor registration, simulation loop ...
ctx.record.stop()
```

---

**Example 3: Headless batch runs with ML policy training**

User: "Run 1000 episodes of my RL agent in GAMMS without visualization and log results for training."

Approach:
1. Set `vis_engine=gamms.visual.Engine.NO_VIS` for headless mode.
2. Wrap the GAMMS simulation step as a Gymnasium-compatible environment.
3. In each episode: reset graph positions, run the loop collecting (state, action, reward) tuples.
4. Use the recorded component system to track per-episode metrics.
5. After all episodes, replay selected runs for visual debugging.

Key code snippet:
```python
for episode in range(1000):
    ctx = gamms.create_context(vis_engine=gamms.visual.Engine.NO_VIS)
    # ... build graph, create agents ...

    @ctx.record.component(struct={'episode': int, 'steps': int, 'score': int})
    class Metrics:
        def __init__(self):
            self.episode = episode
            self.steps = 0
            self.score = 0

    metrics = Metrics(name="metrics")
    ctx.record.start(path=f"runs/episode_{episode:04d}")

    while not ctx.is_terminated():
        for agent in ctx.agent.create_iter():
            state = agent.get_state()
            action = rl_model.predict(state_to_tensor(state))
            state['action'] = action
            agent.set_state()
        apply_rules(ctx)
        metrics.steps += 1
        ctx.visual.simulate()

    ctx.record.stop()
```

## Best Practices

- **Do:** Separate configuration (grid size, agent rosters, sensor params, max steps) into a dedicated `config.py` -- GAMMS simulations become unwieldy when magic numbers are scattered through the loop.
- **Do:** Use `NO_VIS` engine for batch/training runs and only enable pygame for debugging or demo. Visualization is the primary bottleneck.
- **Do:** Save NetworkX graphs to disk after OSM loading (`pickle` or `graphml`) -- OSM parsing is slow and the graph doesn't change between runs.
- **Do:** Register sensors before the simulation loop starts. Adding sensors mid-simulation can cause inconsistent state reads.
- **Avoid:** Putting complex ML inference inside the visualization thread. Run headless for training; replay `.ggr` files for visual inspection.
- **Avoid:** Creating monolithic simulation files. Factor policies, rules, configuration, and the main loop into separate modules -- this matches GAMMS's composable design and makes swapping policies trivial.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| `KeyError` on `agent.get_state()` | Agent name not found | Verify name matches exactly what was passed to `create_agent()` |
| Agent doesn't move | `state['action']` set to non-neighbor node | Validate action against sensor `NEIGHBOR` data before setting |
| OSM graph is empty | Location string not recognized by OSM | Use `graph_from_xml()` with a downloaded `.osm` file instead |
| Pygame window freezes | `ctx.visual.simulate()` not called in loop | Ensure `simulate()` runs every step, even if no agents moved |
| Recording file corrupt | Simulation crashed before `record.stop()` | Wrap the loop in `try/finally` and call `ctx.record.stop()` in `finally` |
| Sensor returns empty data | Sensor not registered to agent | Call `agent.register_sensor()` before the loop; verify sensor ID matches |

## Limitations

- **Not a physics engine.** GAMMS has no collision dynamics, rigid bodies, or continuous motion. Agents teleport between nodes each step. If you need sub-edge movement or physical forces, use a different simulator.
- **Visualization is pygame-only.** There is no web-based or headless image export renderer. For publication-quality figures, export recorded data and plot with matplotlib or similar.
- **No built-in RL training loop.** GAMMS provides the environment, not the training harness. You must write the Gymnasium wrapper, replay buffer, and training loop yourself.
- **OSM graph loading is slow for large regions.** Cities with dense road networks can take minutes to parse. Always cache the resulting graph.
- **Single-threaded simulation.** The main loop is synchronous. For massive agent counts (10,000+), consider batching or external parallelism.
- **Young ecosystem.** The library is at v0.2.x with active development. APIs may change between minor versions; pin your dependency.

## Reference

**Paper:** Patil et al., "GAMMS: Graph based Adversarial Multiagent Modeling Simulator," arXiv:2602.05105, 2026. Read for the five design objectives (scalability, ease-of-use, integration-first, fast visualization, real-world grounding) and comparative analysis against MASON, NetLogo, and Agents.jl.

**Repository:** [https://github.com/GAMMSim/GAMMS](https://github.com/GAMMSim/GAMMS)

**Documentation:** [https://gammsim.github.io/gamms/stable/](https://gammsim.github.io/gamms/stable/)