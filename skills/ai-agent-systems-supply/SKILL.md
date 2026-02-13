---
name: "ai-agent-systems-supply"
description: >
  Build LLM-based multi-agent systems for supply chain inventory management using
  structured decision prompts and memory-retrieval (AIM-RM). Implements the beer game
  multi-echelon supply chain simulation with per-stage agents that use stepwise ordering
  prompts, safety-stock calculations, and Euclidean-distance memory retrieval of similar
  historical episodes. Use when asked to: "build a supply chain agent", "implement inventory
  management with LLMs", "create a beer game simulation with AI agents", "multi-agent
  ordering system", "AIM-RM memory retrieval agent", "supply chain decision prompt design".
---

# AI Agent Systems for Supply Chain Inventory Management

This skill enables Claude to build LLM-based multi-agent systems (MAS) for supply chain inventory management, applying the AIM-RM (Agent with Iterative Memory-Retrieval Manager) architecture from Yoshizato et al. (2026). The core technique assigns one LLM agent per supply chain stage (retailer, wholesaler, distributor, factory), each guided by structured decision prompts that encode stepwise inventory calculations and safety-stock policies. Agents retrieve K-nearest historical episodes from a vector memory store using Euclidean distance, treating past state-action-reward tuples as evidence to inform current ordering decisions. This approach outperforms both heuristic baselines (base-stock, tracking-demand) and reinforcement learning methods (IPPO, MAPPO) across diverse demand patterns.

## When to Use

- When the user asks to build a multi-agent system for inventory management or supply chain optimization
- When implementing a beer game simulation where LLM agents make ordering decisions at each supply chain tier
- When designing structured prompts that guide an LLM through stepwise inventory calculations (inventory position, safety stock, order quantity)
- When building a memory-retrieval system where agents learn from past supply chain episodes via similarity matching
- When the user wants to compare LLM-agent ordering policies against heuristic or RL baselines
- When creating a decentralized multi-echelon supply chain where each stage makes independent ordering decisions with limited visibility

## Key Technique

**Structured Decision Prompts.** Instead of asking an LLM to "decide how much to order," the AIM-RM approach decomposes each ordering decision into explicit calculation steps encoded in the prompt. Three prompt components work together: (1) a **Decision-Maker prompt** (P_DM) that provides the agent with its current state (round number, stage location, lead time, inventory, backlog, arriving deliveries, downstream orders) and requires a numerical order quantity with rationale; (2) a **Step-wise Description prompt** (P_SD) that walks through the four-step period flow (receive delivery, make order decision, ship items, calculate profit) so the agent understands temporal mechanics; and (3) a **Safety-Stock prompt** (P_SS) that encodes the formula: compute inventory position as `IP = inventory + sum(in-transit deliveries) - backlog`, set target as `(lead_time + 1) * mean_demand + z * std_demand * sqrt(lead_time + 1)`, then order `max(0, min(target - IP, capacity))`.

**Memory Retrieval (AIM-RM).** Each agent maintains a per-stage memory store M of `(state_vector, action, reward)` tuples. The state vector has dimension `4 + 2 * lead_time`, encoding `[inventory, backlog, upstream_backlog, recent_shipments..., recent_deliveries...]`. When making a new decision, the agent computes Euclidean distance `d = ||phi(s) - v||_2` between its current state embedding and all stored vectors, retrieves the K=6 nearest neighbors filtered by threshold tau=2, and injects them into a **Memory Usage prompt** (P_MU) that instructs the agent to treat retrieved cases "as evidence, not rules." After each decision, the new experience is appended to memory. This gives agents the ability to adapt across demand patterns (constant, increasing, decreasing) and supply chain configurations (uniform vs. diverse lead times/capacities) without prompt re-engineering.

## Step-by-Step Workflow

1. **Define the supply chain topology.** Specify the number of stages (typically 4: retailer, wholesaler, distributor, factory), lead times per stage, production capacities, initial inventories, and cost parameters (holding cost, backlog cost, sales price). Use a configuration object or YAML file.

2. **Implement the environment simulator.** Build a period-step engine that processes the four-phase cycle: (a) deliver arriving shipments to each stage, (b) collect order decisions from all agents, (c) ship items downstream (limited by inventory), (d) compute per-stage profit as `revenue - holding_cost * inventory - backlog_cost * backlog`. Track demand patterns (constant, increasing `D = 2 + ceil(t/3)`, decreasing, or custom).

3. **Construct the structured decision prompt for each agent.** Compose three sub-prompts:
   - **P_SD** (step-wise description): Explain the period mechanics and lead-time calculation with a worked example.
   - **P_SS** (safety-stock strategy): Encode the inventory-position formula, target calculation with safety stock, and capacity-constrained ordering.
   - **P_DM** (decision-maker): Inject the current state variables and request a JSON response with `{"order_quantity": int, "reasoning": str}`.

4. **Build the memory store per agent.** Initialize an empty list or vector database for each stage. Define the state vector schema as `[inventory, backlog, upstream_backlog, shipments[-L:], deliveries[-L:]]` where L is the stage's lead time. Implement Euclidean distance search with K=6 neighbors and threshold tau=2.

5. **Implement the memory-retrieval prompt (P_MU).** When retrieved cases exist, format them as a list of `{state, action, reward, distance}` objects and append to the decision prompt with the instruction: "Use these similar past experiences as evidence to inform your decision. Do not blindly copy past actions — assess how the current situation differs."

6. **Run the sequential decision loop.** For each period t=1..T, iterate through stages from downstream (retailer) to upstream (factory). Each agent: (a) encodes its current state vector, (b) retrieves similar cases from memory, (c) calls the LLM with the composed prompt, (d) parses the order quantity from the response, (e) submits the action to the environment.

7. **Update memory after each period.** After the environment computes rewards, append `(state_vector, order_quantity, reward)` to each agent's memory store. This enables learning within an episode and across episodes.

8. **Evaluate against baselines.** Implement at least two comparison policies: (a) **Base-Stock**: order `capacity - current_inventory` every period; (b) **Tracking-Demand**: target inventory `= recent_average_demand * lead_time + backlog`. Compute total cost and optimality gap `= (agent_cost - optimal_cost) / optimal_cost * 100%`.

9. **Run multiple episodes with memory carryover.** Execute 3-5 episodes per scenario, carrying memory across episodes so agents accumulate experience. Track per-episode cost improvement to verify learning.

10. **Tune and diagnose.** If agents over-order (bullwhip effect), reduce the safety factor z. If agents under-order (frequent stockouts), increase z or K. If performance degrades with more complex prompts, simplify — the paper found that excessive reasoning effort ("overthinking") can hurt performance.

## Concrete Examples

**Example 1: Basic Beer Game with Structured Prompts**

User: "Build a 4-stage supply chain simulation where LLM agents make ordering decisions using safety-stock logic."

Approach:
1. Define topology: 4 stages, lead times [2,2,2,2], capacities [20,20,20,20], initial inventory [12,12,12,12], holding cost=1, backlog cost=1
2. Set demand: constant at 4 units/period for 12 periods
3. Build the environment loop and agent prompts

Output (agent prompt for stage 2, period 5):
```
SYSTEM: You are an inventory manager at Stage 2 (wholesaler) in a 4-stage
supply chain. Each period follows four steps: (1) receive deliveries,
(2) decide order quantity, (3) ship to downstream, (4) compute profit.

Your lead time is 2 periods. Items you order now arrive in 2 periods.

SAFETY-STOCK ORDERING STRATEGY:
- Compute inventory position: IP = inventory + in_transit - backlog
- Estimate future consumption: target = (lead_time + 1) * avg_demand + z * std_demand * sqrt(lead_time + 1)
- Order quantity = max(0, min(target - IP, capacity))

CURRENT STATE (Period 5, Stage 2):
- Inventory: 8
- Backlog: 2
- In-transit deliveries: [4 arriving period 6, 5 arriving period 7]
- Recent downstream orders: [4, 4, 5, 4]
- Average demand: 4.25, Std demand: 0.43
- Production capacity: 20

Respond with JSON: {"order_quantity": <int>, "reasoning": "<1-2 sentences>"}
```

Agent response:
```json
{"order_quantity": 5, "reasoning": "IP = 8 + 9 - 2 = 15. Target = 3 * 4.25 + 1.0 * 0.43 * 1.73 = 13.49. IP exceeds target but ordering 5 to cover recent demand uptick and maintain buffer."}
```

**Example 2: AIM-RM with Memory Retrieval**

User: "Add memory retrieval so the supply chain agents learn from past episodes."

Approach:
1. After Example 1 runs, each agent has 12 stored experiences
2. On episode 2, before each decision, retrieve similar past states
3. Inject retrieved cases into the prompt

Output (memory-augmented prompt addition for stage 1, period 3, episode 2):
```
SIMILAR PAST EXPERIENCES (from memory, sorted by relevance):
Case 1 [distance=0.8]: state=[inv:10, backlog:0, upstream_bl:1, ship:[4,4], deliv:[4,4]]
  -> Ordered 4, received reward 6.0
Case 2 [distance=1.2]: state=[inv:11, backlog:1, upstream_bl:0, ship:[4,5], deliv:[3,4]]
  -> Ordered 5, received reward 5.0
Case 3 [distance=1.7]: state=[inv:9, backlog:0, upstream_bl:2, ship:[4,4], deliv:[5,3]]
  -> Ordered 6, received reward 4.0

Use these similar past experiences as evidence to inform your decision.
Do not blindly copy past actions -- assess how the current situation differs.
```

**Example 3: Python Implementation Skeleton**

User: "Give me the code structure for an AIM-RM supply chain agent system."

Output:
```python
import numpy as np
from dataclasses import dataclass, field

@dataclass
class SupplyChainConfig:
    num_stages: int = 4
    num_periods: int = 12
    lead_times: list[int] = field(default_factory=lambda: [2, 2, 2, 2])
    capacities: list[int] = field(default_factory=lambda: [20, 20, 20, 20])
    init_inventory: list[int] = field(default_factory=lambda: [12, 12, 12, 12])
    holding_cost: float = 1.0
    backlog_cost: float = 1.0
    safety_factor_z: float = 1.0

class MemoryStore:
    """Per-stage memory of (state_vector, action, reward) tuples."""
    def __init__(self, k: int = 6, tau: float = 2.0):
        self.k = k
        self.tau = tau
        self.memories: list[tuple[np.ndarray, int, float]] = []

    def add(self, state_vec: np.ndarray, action: int, reward: float):
        self.memories.append((state_vec, action, reward))

    def retrieve(self, query_vec: np.ndarray) -> list[dict]:
        if not self.memories:
            return []
        distances = [(np.linalg.norm(query_vec - m[0]), m) for m in self.memories]
        distances.sort(key=lambda x: x[0])
        return [
            {"state": m[1][0], "action": m[1][1], "reward": m[1][2], "distance": round(m[0], 2)}
            for m in distances[:self.k] if m[0] < self.tau
        ]

class SupplyChainEnv:
    """Beer-game style multi-echelon environment."""
    def __init__(self, config: SupplyChainConfig, demand_fn):
        self.config = config
        self.demand_fn = demand_fn  # callable(period) -> int
        self.inventory = list(config.init_inventory)
        self.backlog = [0] * config.num_stages
        self.pipeline = [[0] * lt for lt in config.lead_times]  # in-transit per stage

    def step(self, orders: list[int], period: int) -> list[dict]:
        """Execute one period: deliver, order, ship, profit."""
        # Phase 1: Deliver arriving shipments
        for m in range(self.config.num_stages):
            arriving = self.pipeline[m][0]
            self.inventory[m] += arriving
            self.pipeline[m] = self.pipeline[m][1:] + [0]

        # Phase 2: Orders placed into pipeline (upstream fills them)
        for m in range(self.config.num_stages):
            capped = min(orders[m], self.config.capacities[m])
            self.pipeline[m][-1] = capped

        # Phase 3: Ship downstream (stage 0 faces end customer)
        demand = self.demand_fn(period)
        rewards = []
        for m in range(self.config.num_stages):
            d = demand if m == 0 else orders[m - 1]
            shipped = min(self.inventory[m], d + self.backlog[m])
            self.inventory[m] -= shipped
            self.backlog[m] = max(0, d + self.backlog[m] - shipped)
            reward = -(self.config.holding_cost * self.inventory[m]
                       + self.config.backlog_cost * self.backlog[m])
            rewards.append(reward)

        return [{"inventory": self.inventory[m], "backlog": self.backlog[m],
                 "reward": rewards[m]} for m in range(self.config.num_stages)]

    def get_state_vector(self, stage: int) -> np.ndarray:
        """Encode state for memory storage/retrieval."""
        vec = [self.inventory[stage], self.backlog[stage],
               self.backlog[min(stage + 1, self.config.num_stages - 1)]]
        vec.extend(self.pipeline[stage])  # in-transit deliveries
        return np.array(vec, dtype=float)

class AIMRMAgent:
    """One agent per supply chain stage with structured prompts + memory."""
    def __init__(self, stage: int, config: SupplyChainConfig, llm_call):
        self.stage = stage
        self.config = config
        self.memory = MemoryStore(k=6, tau=2.0)
        self.llm_call = llm_call  # callable(prompt) -> str

    def build_prompt(self, state: dict, similar_cases: list[dict]) -> str:
        prompt = f"""You are inventory manager at Stage {self.stage}.
Lead time: {self.config.lead_times[self.stage]}, Capacity: {self.config.capacities[self.stage]}.

ORDERING STRATEGY:
1. Compute IP = inventory + in_transit - backlog
2. Target = (lead_time + 1) * avg_demand + {self.config.safety_factor_z} * std_demand * sqrt(lead_time + 1)
3. Order = max(0, min(target - IP, capacity))

CURRENT STATE: {state}
"""
        if similar_cases:
            prompt += "\nSIMILAR PAST EXPERIENCES:\n"
            for i, c in enumerate(similar_cases):
                prompt += f"Case {i+1} [dist={c['distance']}]: action={c['action']}, reward={c['reward']}\n"
            prompt += "\nUse these as evidence, not rules.\n"

        prompt += '\nRespond JSON: {"order_quantity": <int>, "reasoning": "<str>"}'
        return prompt

    def decide(self, env: SupplyChainEnv) -> int:
        state_vec = env.get_state_vector(self.stage)
        cases = self.memory.retrieve(state_vec)
        state_info = {"inventory": env.inventory[self.stage],
                      "backlog": env.backlog[self.stage],
                      "pipeline": env.pipeline[self.stage]}
        prompt = self.build_prompt(state_info, cases)
        response = self.llm_call(prompt)
        order = parse_order(response)  # extract order_quantity from JSON
        return order

    def update_memory(self, state_vec: np.ndarray, action: int, reward: float):
        self.memory.add(state_vec, action, reward)
```

## Best Practices

- **Do:** Encode the full stepwise calculation (IP, target, order) directly in the prompt rather than asking the LLM to figure out the formula on its own. Explicit calculation steps dramatically reduce ordering errors.
- **Do:** Set the memory retrieval threshold tau conservatively (start with tau=2.0). Retrieving dissimilar cases degrades performance more than retrieving no cases at all.
- **Do:** Run agents sequentially from downstream to upstream within each period, matching real supply chain information flow. Parallel execution breaks the demand signal propagation.
- **Do:** Store the raw numerical state vector for similarity search, not the text description. Euclidean distance on structured numerical features outperforms semantic text similarity for this domain.
- **Avoid:** Over-engineering the prompt with excessive reasoning instructions. The paper found that "high" reasoning effort decreased performance compared to "medium" — LLMs can overthink inventory decisions.
- **Avoid:** Sharing memory across stages. Each stage faces different demand signals (downstream orders, not end-customer demand), so memories are stage-specific. Cross-stage memory introduces noise.

## Error Handling

- **LLM returns non-numeric order:** Parse the response with a fallback regex `r'"order_quantity"\s*:\s*(\d+)'`. If parsing fails, fall back to the safety-stock formula computed deterministically from the current state.
- **Order exceeds capacity:** Always cap: `order = min(parsed_order, capacity)`. Do not trust the LLM to respect capacity constraints even when stated in the prompt.
- **Memory store grows too large:** For long-running simulations (>100 episodes), implement a sliding window or reservoir sampling to keep memory size bounded. Performance plateaus after ~50 episodes of experience.
- **Negative inventory position:** Clamp IP to zero before computing order quantity. A negative IP means severe backlog — order at full capacity as an emergency policy.
- **Divergent bullwhip behavior:** If upstream agents consistently order 2x+ downstream demand, reduce the safety factor z or add a prompt clause: "Your order should not exceed 2x your recent average downstream demand unless backlog exceeds inventory."

## Limitations

- The approach is validated on the beer game (4 stages, 12 periods) — scaling to larger networks (50+ nodes, tree topologies) is untested and may require prompt compression or hierarchical memory.
- Constant-demand scenarios achieve optimality (0% gap), but complex demand patterns (increasing/decreasing) still show ~74% optimality gap, indicating LLM agents are not yet replacements for tuned RL policies in non-stationary settings.
- Each agent decision requires an LLM API call, making real-time or high-frequency inventory management cost-prohibitive. Best suited for strategic/tactical planning horizons (weekly/monthly decisions).
- Memory retrieval uses raw Euclidean distance on state vectors — it does not account for temporal dynamics or causal relationships between states. Two states can be numerically close but represent fundamentally different supply chain phases.
- The method assumes decentralized agents with no inter-agent communication beyond order signals. Cooperative or information-sharing supply chain setups need a different architecture.

## Reference

**Paper:** Yoshizato, K., Shimizu, K., Higa, R., & Otsuka, T. (2026). *AI Agent Systems for Supply Chains: Structured Decision Prompts and Memory Retrieval.* arXiv:2602.05524v1. AAMAS 2026.
https://arxiv.org/abs/2602.05524v1

Look for: Section 3 (prompt templates P_DM, P_SD, P_SS, P_MU), Algorithm 1 (one-round decision procedure), Table 1 (scenario configurations), and Table 2 (optimality gap results across demand patterns).