---
name: "from-assumptions-actions-turning"
description: "Uncertainty-aware planning using the PCE (Planner-Composer-Evaluator) decision-tree framework. Converts implicit assumptions in LLM reasoning into structured decision trees scored by likelihood, gain, and cost for rational action selection. Use when: 'plan under uncertainty', 'build a decision tree for agent actions', 'handle partial observability in multi-agent system', 'design uncertainty-aware agent planner', 'score alternative plans with cost-benefit', 'reduce communication overhead in multi-agent coordination'."
version: 1
---

# PCE: Uncertainty-Aware Planning via Planner-Composer-Evaluator Decision Trees

This skill enables Claude to design and implement uncertainty-aware planning systems for autonomous agents using the PCE (Planner-Composer-Evaluator) framework from Seo et al. (ICLR 2026). The core technique extracts implicit assumptions buried in LLM chain-of-thought reasoning, organizes them into a binary decision tree where internal nodes are environment assumptions and leaves are candidate actions, then scores each root-to-leaf path by scenario likelihood, goal-directed gain, and execution cost. This replaces expensive inter-agent communication with structured internal deliberation, enabling agents to act rationally under partial observability.

## When to Use

- When building a multi-agent system where agents must act under **partial observability** (hidden objects, unknown collaborator intentions) and you want to minimize communication overhead
- When a user asks to **plan under uncertainty** — e.g., "my agent doesn't know the full state, how should it pick actions?"
- When designing an **agent planner** that must weigh multiple hypothetical scenarios before committing to an action
- When implementing **cost-benefit action selection** for embodied or simulated agents that have both physical actions and communication options
- When the user wants to **structure LLM chain-of-thought reasoning** into an explicit decision tree rather than relying on a single greedy action
- When reducing **token and latency costs** in multi-agent LLM pipelines by replacing frequent message exchanges with internal deliberation

## Key Technique

Standard LLM-based agent planners generate a single chain-of-thought and pick one action. The problem is that each candidate action rests on a **single implicit assumption** about the uncertain environment — e.g., "the cup is probably in the kitchen cabinet" — while ignoring alternative scenarios. PCE makes these assumptions explicit and covers the combinatorial space systematically.

The framework has three stages. The **Planner** prompts an LLM with the agent's context (goal, progress, message history, recent actions, available action list) and extracts both a candidate action and the reasoning trace behind it. The **Composer** then builds a binary decision tree top-down: it identifies the most uncertainty-reducing assumption from the trace, creates True/False branches, and recurses up to depth D. Internal nodes are atomic assumptions ("the cup is in the bathroom cabinet" — True/False); leaves are concrete actions. The Composer uses a local ranking policy that prioritizes assumptions which most influence subsequent action choice and can generate new assumptions grounded in entities already in context. The **Evaluator** scores every root-to-leaf path using: scenario likelihood L(S) (LLM-assessed probability the assumption chain holds), conditional gain G(a) (how much the leaf action advances the goal given the scenario), and execution cost C(a) = alpha * d(a) * 1{move} + beta * l(a) * 1{comm}, where d(a) is traversal distance and l(a) is message length. The final utility is U(S,a) = L(S) * G(a) - lambda * C(a). The agent executes the leaf action with maximum U.

This structured approach consistently outperformed communication-heavy baselines across benchmarks (C-WAH, TDW-MAT) and three LLM backbones, cutting communication actions by 5-6x while improving success rate and task efficiency.

## Step-by-Step Workflow

1. **Define the agent context structure.** Create a data model that holds: the common goal G, current task progress, collaborator state (last known), message logs (keep last K_message=3), recent action history (last K_action=10), and the available action list (physical actions + communication actions).

2. **Implement the Planner module.** Prompt the LLM with the full agent context and ask it to (a) propose a candidate action and (b) output its full chain-of-thought reasoning trace. Parse the response to extract both the action and the reasoning text.

3. **Extract assumptions from the reasoning trace.** Parse the Planner's chain-of-thought to identify atomic, binary assumptions about the environment — statements that can be True or False. Examples: "Object X is at location Y", "Partner agent is currently doing task Z", "The door to room R is unlocked". Each assumption should be grounded in entities present in the context.

4. **Build the decision tree via the Composer.** Starting from the root, select the assumption that most reduces uncertainty and most influences action choice. Create True and False child branches. For each branch, recurse: filter remaining assumptions to those consistent with the current path, select the next most informative one, and split again. Stop at depth D (default D=3) or when further splits would not materially change the leaf action. At each leaf, assign the best action for that scenario.

5. **Score each root-to-leaf path with the Evaluator.** For every leaf, compute three values:
   - **Scenario likelihood L(S)**: Prompt the LLM to estimate the probability (0-1) that the conjunction of assumptions along this path is true, given current observations and message history.
   - **Conditional gain G(a)**: Prompt the LLM to rate (0-1) how much executing the leaf action advances the goal, assuming this scenario holds.
   - **Execution cost C(a)**: Calculate alpha * distance * is_movement + beta * message_length * is_communication (default alpha=beta=1).

6. **Compute utility and select the best action.** For each leaf, calculate U(S,a) = L(S) * G(a) - lambda * C(a) with lambda=1 as default. Select the action with the highest U. If the top two scores are within a small epsilon, prefer the lower-cost action.

7. **Execute the selected action and update context.** Carry out the chosen action in the environment, then update the agent's memory: append the action to history, incorporate any new observations, and update collaborator state if a message was received.

8. **Repeat at each decision step.** On every planning cycle, rebuild the tree from scratch with fresh context. The tree is ephemeral — it is a deliberation structure, not a persistent plan.

9. **Tune hyperparameters for your domain.** Adjust tree depth D (deeper = more scenarios but more LLM calls), cost weights alpha/beta (raise beta to discourage unnecessary communication), and cost sensitivity lambda (raise to make the agent more cost-averse). The paper found D=3, alpha=beta=lambda=1 robust across benchmarks.

## Concrete Examples

**Example 1: Household task agent deciding where to search for an object**

```
User: I'm building a household robot agent. It needs to find a cup but doesn't
know which room the cup is in. It can move to rooms, open containers, or ask
its partner agent. Design the decision logic.

Approach:
1. Planner generates reasoning: "The cup might be in the kitchen cabinet
   (most cups are stored there) or the bathroom counter (partner was
   cleaning there earlier). I should check kitchen first."

2. Extract assumptions:
   A1: "Cup is in kitchen cabinet" (True/False)
   A2: "Partner already checked the kitchen" (True/False)

3. Build decision tree (depth 2):
           [A1: Cup in kitchen cabinet?]
          /  True                 \  False
   [A2: Partner checked       [A2: Partner checked
    kitchen?]                  kitchen?]
   / True     \ False         / True      \ False
  go_to       go_to          go_to        go_to
  bathroom    kitchen        bathroom     kitchen
  _cabinet    _cabinet       _counter     _cabinet

4. Score each leaf:
   Path 1 (A1=T, A2=T): L=0.15, G=0.9, C=3.0 → U = 0.135 - 3.0 = -2.87
   Path 2 (A1=T, A2=F): L=0.45, G=0.9, C=2.0 → U = 0.405 - 2.0 = -1.60
   Path 3 (A1=F, A2=T): L=0.10, G=0.6, C=4.0 → U = 0.060 - 4.0 = -3.94
   Path 4 (A1=F, A2=F): L=0.30, G=0.5, C=2.0 → U = 0.150 - 2.0 = -1.85

5. Select: Path 2 wins → go_to kitchen_cabinet

Output: Agent moves to kitchen cabinet (highest expected gain per unit cost,
exploiting the likely scenario without needing to ask the partner).
```

**Example 2: Multi-agent coordination with communication cost trade-off**

```
User: My two agents are collecting objects in a simulated house. Agent A found
3 of 5 target objects. Agent B's status is unknown. Should A communicate or
keep searching?

Approach:
1. Planner reasoning: "B might have found the remaining 2 objects already
   (in which case we're done), or B might be stuck and needs help."

2. Extract assumptions:
   A1: "Agent B has found remaining objects" (True/False)
   A2: "Agent B is in an accessible room" (True/False)

3. Build decision tree:
           [A1: B found remaining?]
          /  True           \  False
   do_nothing           [A2: B accessible?]
   (task done)         / True        \ False
                    send_message     go_search
                    ("status?")      _living_room

4. Score each leaf:
   Path 1 (A1=T):       L=0.25, G=1.0, C=0.0 → U = 0.250
   Path 2 (A1=F, A2=T): L=0.30, G=0.7, C=0.5 → U = 0.210 - 0.5 = -0.29
   Path 3 (A1=F, A2=F): L=0.45, G=0.6, C=2.0 → U = 0.270 - 2.0 = -1.73

5. Select: Path 1 has highest U, but it requires doing nothing. Since A1=T
   has only 0.25 probability, the agent should consider: if wrong, the cost
   of inaction is high. Reframe: the *expected* utility across scenarios is
   what matters. Weighted sum: the "do_nothing" action only pays off 25% of
   the time. The search action (Path 3) has low U but covers the most likely
   scenario. Adjusted selection: go_search_living_room.

Output: Agent A continues searching rather than communicating — the search
has better expected value than the communication cost of messaging Agent B.
```

**Example 3: Implementing PCE as a Python module**

```
User: Give me a Python implementation of the PCE scoring logic.

Output:
```python
from dataclasses import dataclass

@dataclass
class Leaf:
    scenario: list[tuple[str, bool]]  # [(assumption, truth_value), ...]
    action: str
    likelihood: float   # L(S) in [0, 1]
    gain: float         # G(a) in [0, 1]
    move_distance: float
    is_movement: bool
    message_length: float
    is_communication: bool

def compute_utility(
    leaf: Leaf,
    alpha: float = 1.0,
    beta: float = 1.0,
    lam: float = 1.0,
) -> float:
    """PCE utility: U(S,a) = L(S) * G(a) - lambda * C(a)."""
    expected_gain = leaf.likelihood * leaf.gain
    cost = (
        alpha * leaf.move_distance * leaf.is_movement
        + beta * leaf.message_length * leaf.is_communication
    )
    return expected_gain - lam * cost

def select_action(leaves: list[Leaf], **kwargs) -> Leaf:
    """Select the leaf with maximum utility."""
    return max(leaves, key=lambda l: compute_utility(l, **kwargs))
```
```

## Best Practices

- **Do:** Extract assumptions as atomic, binary propositions grounded in entities the agent has actually observed or been told about. "The cup is in the kitchen" is good; "things might go wrong" is too vague.
- **Do:** Keep tree depth small (D=2 or D=3). Each level doubles the LLM scoring calls. Diminishing returns appear beyond D=3 in the paper's ablations.
- **Do:** Normalize likelihood scores across leaves so they sum to approximately 1.0 — this prevents the evaluator from assigning high probability to all scenarios simultaneously.
- **Do:** Reuse the same LLM call structure for Planner and Evaluator to keep implementation simple. The Evaluator just needs likelihood and gain ratings, which can be a single structured-output call per leaf.
- **Avoid:** Treating the decision tree as a persistent plan. Rebuild it every decision step with fresh observations — stale trees encode outdated assumptions.
- **Avoid:** Setting lambda=0 (ignoring cost). Without cost penalization, the agent will over-communicate and over-explore. The cost term is what gives PCE its efficiency advantage.
- **Avoid:** Using PCE for fully observable, single-agent tasks. The framework's value comes from handling uncertainty about hidden state and collaborator intentions. In fully observable settings, simpler planners suffice.

## Error Handling

- **LLM returns non-numeric likelihood/gain scores:** Constrain the output format with structured generation (JSON mode or function calling). Clamp values to [0, 1] and re-prompt if parsing fails.
- **All leaves score negatively:** This means every action's cost exceeds its expected gain. Fall back to the action with the least negative score — the agent still must act. Alternatively, reduce lambda temporarily.
- **Assumption extraction produces zero assumptions:** The Planner's reasoning was too shallow. Re-prompt with explicit instruction: "List 2-3 things you are uncertain about regarding the environment state."
- **Decision tree is unbalanced (all leaves choose same action):** The assumptions don't influence action choice. Replace them with more decision-relevant assumptions or reduce tree depth to avoid wasted computation.
- **Token budget exceeded with deep trees:** Reduce D, reduce K_message/K_action context windows, or batch Evaluator calls (score multiple leaves in one prompt).

## Limitations

- PCE adds LLM calls proportional to 2^D leaves for scoring. At D=3, that is 8 evaluator calls per decision step — acceptable for turn-based or slow-clock environments but potentially too slow for real-time control loops under 100ms.
- The quality of uncertainty handling depends entirely on the LLM's calibration. If the model systematically misjudges likelihood (e.g., always assigns 0.5), the decision tree degenerates to cost-only ranking.
- PCE assumes discrete, enumerable actions. Continuous action spaces (e.g., precise robot joint angles) require discretization before the framework applies.
- The framework was validated on cooperative multi-agent tasks. Adversarial or competitive settings (where other agents actively deceive) may require game-theoretic extensions beyond likelihood scoring.
- Communication cost modeling (beta * message_length) is a simplification. In real human-AI teams, the disruption cost of a message depends on timing and context, not just length.

## Reference

**Paper:** Seo et al., "From Assumptions to Actions: Turning LLM Reasoning into Uncertainty-Aware Planning for Embodied Agents," ICLR 2026. [arXiv:2602.04326](https://arxiv.org/abs/2602.04326v1)

**What to look for:** Section 3 for the full PCE formulation (Planner prompt structure, Composer tree-building algorithm, Evaluator scoring equations); Section 4 for benchmark configurations and hyperparameter settings; Appendix for prompt templates and ablation details on tree depth D.