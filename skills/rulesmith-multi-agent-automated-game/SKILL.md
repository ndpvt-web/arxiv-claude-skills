---
name: "rulesmith-multi-agent-automated-game"
description: "Automated game balancing using multi-agent LLM self-play coupled with Bayesian optimization. Use when the user asks to 'balance a game', 'tune game parameters', 'optimize game rules', 'automate playtesting', 'build a game balancing pipeline', or 'use LLM agents for game simulation'."
---

# RuleSmith: Multi-Agent LLM Automated Game Balancing

This skill enables Claude to build automated game balancing systems that combine multi-agent LLM self-play with Bayesian optimization. Instead of manual playtesting, LLM agents interpret textual rulebooks and game states to play games against each other, generating win-rate data that a Bayesian optimizer uses to search a multi-dimensional parameter space for balanced configurations. The technique produces interpretable rule adjustments (e.g., "increase cavalry HP from 3 to 5") that converge to near-equal win rates across factions.

## When to Use

- When the user wants to automatically balance a game with tunable numeric parameters (unit stats, costs, resource rates, scoring weights)
- When building a system where LLM agents play a game via textual state descriptions and structured action outputs
- When the user needs to search a discrete parameter space using Bayesian optimization with a noisy objective function
- When implementing self-play evaluation loops where agents receive rulebooks, game state, and legal actions as text prompts
- When the user wants to replace expensive human playtesting with LLM-simulated games to estimate balance metrics like win-rate disparity
- When optimizing any multi-agent environment where the goal is symmetric or fair outcomes across heterogeneous participants

## Key Technique

RuleSmith closes the loop between game simulation and parameter optimization. The core insight is that LLM agents can serve as competent game-playing surrogates: given a textual rulebook and structured game state, they produce valid actions in JSON format, enabling thousands of simulated games without human players. This makes win-rate estimation cheap enough to embed inside an optimization loop.

The optimization layer uses a Gaussian process surrogate with a Matern 5/2 kernel to model the balance loss function across the parameter space. An Expected Improvement (EI) acquisition function proposes candidate parameter vectors, which are projected from continuous space to valid discrete values (integers or fixed-precision decimals). Critically, the system uses **adaptive sampling**: promising candidates (high EI) receive more evaluation games (up to 64) for accurate assessment, while exploratory candidates receive fewer (as low as 16), efficiently allocating compute. The balance loss is `L(theta) = |w_empire - 0.5| + |w_nomads - 0.5| + 0.5 * w_draw`, targeting 50/50 win rates with a penalty for excessive draws.

The agent prompt architecture is faction-specific: each agent receives a role system prompt, current turn/resources/unit positions, a strategy guide, the list of legal actions per unit, and RAG-retrieved relevant rules (via TF-IDF cosine similarity against the rulebook). Agents output simultaneous multi-unit actions as structured JSON, and the engine validates legality, defaulting illegal moves to PASS.

## Step-by-Step Workflow

1. **Define the parameter space.** Enumerate every tunable numeric parameter in the game (unit stats, costs, gather rates, scoring weights). For each parameter, specify its type (integer or decimal), valid range, and step size. Group parameters by subsystem (economy, combat, production, scoring) for interpretability.

2. **Build the game engine with textual state output.** Implement a deterministic game engine in Python that (a) accepts a parameter vector to configure rules, (b) outputs game state as structured text each turn (resources, unit positions, enemy positions, legal actions per unit), and (c) accepts player actions as JSON dictionaries mapping unit IDs to action types.

3. **Design faction-specific agent prompts.** For each faction, create a system prompt containing: the faction's identity and asymmetric strengths, a strategy guide, and output format instructions. At each turn, construct the user prompt with: turn index, max turns, current resources, full unit/enemy state, and the legal action list. Use a lightweight RAG system (TF-IDF cosine similarity) to retrieve the 3-5 most relevant rulebook sections for the current game state.

4. **Implement the self-play evaluation function.** Given a parameter vector theta, run N games between LLM agents controlling opposing factions. Parse agent JSON outputs, validate against legal moves (default to PASS on invalid), execute the game loop, and record outcomes. Compute win rates: `w_faction = (1/N) * sum(faction_wins)`. Return the balance loss `L = |w_a - 0.5| + |w_b - 0.5| + 0.5 * w_draw`.

5. **Set up Bayesian optimization with a Gaussian process surrogate.** Initialize a GP with Matern 5/2 kernel over the continuous relaxation of the parameter space. Seed with 5-10 random parameter configurations evaluated via self-play. Use Expected Improvement as the acquisition function: `EI(theta) = (y* - mu(theta)) * Phi(z) + sigma(theta) * phi(z)` where `z = (y* - mu(theta)) / sigma(theta)`.

6. **Implement adaptive game allocation.** For each candidate theta proposed by the optimizer, compute its EI value and allocate games proportionally: `N_t = N_min + (N_max - N_min) * EI(theta_t) / max(EI)`. Use N_min=16 and N_max=64. This focuses compute on promising candidates while still exploring cheaply.

7. **Apply discrete projection before evaluation.** Map continuous candidates to valid game parameters: round integers to nearest int, round decimals to nearest valid step (e.g., 0.1 increments). Clamp all values to their defined ranges. This deterministic projection `D(theta)` ensures every evaluated configuration is a legal ruleset.

8. **Run the optimization loop for 50-100 iterations.** At each step: (a) select candidate via EI, (b) project to discrete space, (c) evaluate via adaptive self-play, (d) update the GP surrogate with the new observation. Track the best-seen loss and corresponding parameters.

9. **Validate the final configuration.** Take the best parameter vector and run a large number of evaluation games (200+) to confirm win rates are within the target range (50% +/- 5%). Report per-faction win rates, draw rate, and the full parameter vector with human-readable labels.

10. **Generate interpretable balance reports.** Compare the optimized parameters against the initial/default configuration. Produce a diff showing which parameters changed and by how much, grouped by subsystem. Include TTK (time-to-kill) calculations and resource-efficiency metrics to explain *why* the changes produce balance.

## Concrete Examples

**Example 1: Balancing a Two-Faction Strategy Game**

User: "I have a simple strategy game with two factions (Humans and Orcs) on a grid. Humans have archers and knights, Orcs have berserkers and shamans. The Orcs win 70% of the time. Help me set up an automated balancing system."

Approach:
1. Identify tunable parameters: archer damage, archer HP, knight damage, knight HP, berserker damage, berserker HP, shaman heal amount, unit costs per faction, starting gold
2. Build a text-based state renderer that outputs unit positions, HP, and legal actions as structured text each turn
3. Create two agent prompts — one for Humans (emphasize ranged advantage, formation play) and one for Orcs (emphasize aggression, healing sustain)
4. Wire up Bayesian optimization with balance loss targeting 50/50 win rates

Output structure:
```python
# Parameter space definition
PARAM_SPACE = {
    "archer_damage": {"type": "int", "range": [1, 5], "default": 2},
    "archer_hp": {"type": "int", "range": [2, 8], "default": 3},
    "knight_damage": {"type": "int", "range": [2, 6], "default": 3},
    "knight_hp": {"type": "int", "range": [3, 10], "default": 5},
    "berserker_damage": {"type": "int", "range": [2, 7], "default": 4},
    "berserker_hp": {"type": "int", "range": [3, 10], "default": 6},
    "shaman_heal": {"type": "int", "range": [1, 4], "default": 2},
    "human_unit_cost": {"type": "float", "range": [5.0, 15.0], "step": 0.5, "default": 8.0},
    "orc_unit_cost": {"type": "float", "range": [5.0, 15.0], "step": 0.5, "default": 7.0},
    "starting_gold": {"type": "int", "range": [20, 60], "default": 30},
}

# Balance loss function
def compute_balance_loss(results: list[str]) -> float:
    n = len(results)
    w_human = sum(1 for r in results if r == "human") / n
    w_orc = sum(1 for r in results if r == "orc") / n
    w_draw = sum(1 for r in results if r == "draw") / n
    return abs(w_human - 0.5) + abs(w_orc - 0.5) + 0.5 * w_draw

# Adaptive game count
def adaptive_game_count(ei_value, max_ei, n_min=16, n_max=64):
    return int(n_min + (n_max - n_min) * ei_value / max(max_ei, 1e-8))
```

**Example 2: Adding Bayesian Optimization to an Existing Game Engine**

User: "I already have a Python card game engine with 8 tunable parameters. I want to plug in LLM self-play and Bayesian optimization to find balanced settings."

Approach:
1. Wrap the existing engine to accept a parameter dict and return game state as text per turn
2. Build the LLM agent interface: system prompt with card rules, per-turn prompt with hand/board/legal plays
3. Integrate scikit-optimize or BoTorch for the GP-based optimizer

```python
from skopt import gp_minimize
from skopt.space import Integer, Real

# Define search space from game parameters
space = [
    Integer(1, 10, name="card_attack_base"),
    Integer(2, 12, name="card_hp_base"),
    Real(0.5, 2.0, name="mana_regen_rate"),
    # ... remaining parameters
]

def evaluate_balance(params):
    config = dict(zip([s.name for s in space], params))
    # Run adaptive self-play
    ei = current_acquisition_value  # from optimizer internals
    n_games = adaptive_game_count(ei, max_ei_seen)
    results = run_self_play(config, n_games=n_games, llm_model="gpt-4o-mini")
    return compute_balance_loss(results)

result = gp_minimize(
    evaluate_balance,
    space,
    n_calls=100,
    acq_func="EI",
    n_initial_points=10,
)
print(f"Best params: {result.x}, Loss: {result.fun:.4f}")
```

**Example 3: Designing the LLM Agent Prompt**

User: "How should I structure the prompts for the LLM agents that play the game?"

Approach — build a layered prompt with five sections:

```
SYSTEM PROMPT:
"You are playing as the Empire faction in CivMini. Your strengths:
specialized units (Farmers gather, Soldiers fight). Your weakness:
slower movement (1 cell/turn). Win by destroying the enemy city or
outscoring by the final turn."

USER PROMPT (per turn):
"Turn 5/16 | Resources: 12 | Score: Empire 8 vs Nomads 6

YOUR UNITS:
- Farmer_1 at (2,3): HP 3/3 | Legal: [GATHER, MOVE_N, MOVE_E]
- Soldier_1 at (4,4): HP 5/5 | Legal: [ATTACK_Cavalry_1, MOVE_S, MOVE_W]

ENEMY UNITS:
- Cavalry_1 at (4,5): HP 4/4

RELEVANT RULES (retrieved via RAG):
- Soldiers deal 2 damage to adjacent enemies
- Cavalry deal 3 damage but take 1 retaliation damage
- Gathering yields 3 resources per action

Respond with a JSON object mapping unit IDs to actions:
{\"Farmer_1\": {\"action\": \"GATHER\"}, \"Soldier_1\": {\"action\": \"ATTACK_Cavalry_1\"}}"
```

Key design choices:
- Legal actions listed per unit prevents hallucinated moves
- RAG retrieval (TF-IDF over rulebook chunks) surfaces relevant rules without overloading context
- Structured JSON output enables deterministic parsing; invalid actions default to PASS

## Best Practices

- **Do** validate every agent action against the legal action list and default to PASS for invalid outputs. LLM agents will occasionally hallucinate illegal moves, and graceful fallback prevents crashes.
- **Do** use deterministic game engines so the only randomness comes from LLM action selection. This makes win-rate estimates meaningful and reproducible.
- **Do** run at least 16 games per parameter configuration even for exploratory candidates. Fewer games produce unreliable win-rate estimates due to high variance.
- **Do** group parameters by subsystem (economy, combat, production, scoring) in reports so designers understand *what* changed and *why* it helps balance.
- **Avoid** using overly large LLMs for self-play agents. Smaller models (2B-8B parameters) are sufficient for structured game-playing and dramatically reduce evaluation cost. Reserve large models for complex strategy games.
- **Avoid** optimizing more than 15-20 parameters simultaneously. Bayesian optimization scales poorly with dimensionality. If the parameter space is larger, fix low-sensitivity parameters first using ablation studies.

## Error Handling

- **LLM output parsing failures:** Wrap JSON parsing in try/except. On failure, retry once with a simplified prompt containing only the legal action list. If still unparseable, default all units to PASS for that turn.
- **GP surrogate fitting errors:** If the Gaussian process fails to fit (e.g., singular kernel matrix), add a small jitter term (1e-6) to the diagonal or reduce the number of observations by removing duplicates.
- **Degenerate games:** If one agent plays PASS every turn (model collapse), detect this by checking if action diversity falls below a threshold, and re-run with higher temperature or a different model.
- **Non-convergence after 100 iterations:** The parameter space may be too large or the balance loss landscape too flat. Reduce dimensionality by fixing non-sensitive parameters, or increase N_min to get more reliable loss estimates.
- **Asymmetric model capacity:** If agents of different capability levels play each other, the optimizer will compensate by making the weaker agent's faction numerically stronger. Always use symmetric model configurations for fair balance assessment.

## Limitations

- LLM agents are not optimal players. Balanced configurations found by RuleSmith are balanced *for LLM-level play*, which may not transfer to expert human players who exploit strategies the LLMs miss.
- The approach requires a text-representable game state. Games with complex visual information (real-time 3D, physics-based) need a text abstraction layer that may lose critical information.
- Bayesian optimization becomes inefficient beyond ~20 continuous dimensions. Games with hundreds of tunable parameters need dimensionality reduction or hierarchical optimization.
- Evaluation cost scales linearly with game length and number of games. Long games (100+ turns) with high N_max (64 games) can be prohibitively slow without GPU parallelization.
- The balance loss formulation assumes two factions targeting 50/50 win rates. Games with 3+ factions, asymmetric objectives, or non-zero-sum outcomes need a modified loss function (e.g., pairwise win-rate matrix distance from uniform).

## Reference

**Paper:** [RuleSmith: Multi-Agent LLMs for Automated Game Balancing](https://arxiv.org/abs/2602.06232v1) (Zeng et al., 2026). Focus on Section 3 for the Bayesian optimization loop with adaptive sampling, Section 4 for the agent prompt architecture and RAG-based rule retrieval, and Table 5 for how optimized parameters shift across model capacities.