---
name: "from-assumptions-actions-turning"
description: "Build uncertainty-aware planners for multi-agent systems using the PCE (Planner-Composer-Evaluator) decision tree framework. Converts implicit LLM reasoning assumptions into scored decision trees that select actions under uncertainty without heavy inter-agent communication. Use when: 'build an agent that plans under uncertainty', 'create a decision tree from assumptions', 'multi-agent planning with partial observability', 'reduce agent communication overhead', 'score actions by likelihood and cost', 'uncertainty-aware action selection'."
---

# PCE: Uncertainty-Aware Planning via Assumption Decision Trees

This skill enables Claude to implement the Planner-Composer-Evaluator (PCE) framework from ICLR 2026, which turns the implicit assumptions buried inside LLM reasoning traces into explicit, scored decision trees. Instead of relying on expensive back-and-forth communication between agents to resolve uncertainty, PCE structures what the agent *already suspects* into a tree where internal nodes are binary assumptions about the world, leaves are candidate actions, and each path is scored by scenario likelihood, goal-directed gain, and execution cost. The result is principled action selection under partial observability with minimal communication.

## When to Use

- When building a multi-agent system where agents must act despite incomplete information about other agents' states or hidden objects
- When the user wants to reduce token cost or latency caused by excessive inter-agent dialogue in an LLM-based agent loop
- When designing a planner that must choose between physical actions and communication actions under uncertainty
- When converting free-form LLM reasoning ("I think the item might be in room A, but maybe the other agent already found it") into structured, scoreable plans
- When implementing embodied agents in partially observable, decentralized environments (e.g., household tasks, search-and-rescue, warehouse coordination)
- When the user asks to build a decision framework that explicitly represents and evaluates uncertain assumptions before committing to an action

## Key Technique

**The core insight** is that when an LLM reasons about what to do next, its chain-of-thought already contains assumptions about the environment—they are just buried, fragmented, and never systematically evaluated. PCE extracts these assumptions, organizes them into a binary decision tree, and scores each root-to-leaf path to pick the best action. This replaces the common pattern of "ask the other agent to resolve my uncertainty" with "structure my uncertainty, estimate which scenario is most likely, and act on the best expected payoff."

**The three phases work as follows.** The *Planner* takes the agent's context (goal, progress, observations, message history, available actions) and generates a candidate action with a reasoning trace. The *Composer* then mines that trace for implicit assumptions, builds a decision tree of depth D (default 3) where each internal node is a True/False assumption split and each leaf is an action (physical or communicative). The tree is expanded top-down, prioritizing assumptions that most reduce uncertainty and most influence action choice; expansion stops early when further splits would not change the recommended action. Finally, the *Evaluator* scores every root-to-leaf path using: **Scenario Likelihood** L(S) (how probable is this branch's premise?), **Conditional Gain** G(a) (how much does this action advance the goal given the premise?), and **Execution Cost** C(a) (movement distance or communication token cost). The utility formula is `U(S, a) = L(S) * G(a) - lambda * C(a)`, and the agent executes the leaf action with maximum U.

**Why this works better than scaling communication:** PCE treats communication as just another action to be scored against physical alternatives, rather than as the default mechanism for uncertainty resolution. This means the agent only communicates when the expected information gain exceeds the cost—producing communication patterns that human partners in user studies rated as more efficient and trustworthy.

## Step-by-Step Workflow

1. **Gather agent context.** Collect the agent's current goal, task progress so far, observation history (last K_action=10 actions), message log (last K_message=3 messages), and the list of available actions (physical moves, object interactions, communication).

2. **Run the Planner phase.** Prompt the LLM with the context and ask it to select an action *with full reasoning*. Capture the entire chain-of-thought trace—this is the raw material containing latent assumptions.

3. **Extract assumptions from the reasoning trace.** Parse the Planner's trace to identify conditional statements, hedges, and uncertainty markers ("might be," "if X is in Y," "assuming the other agent hasn't already," etc.). Each becomes a candidate assumption node.

4. **Build the decision tree (Composer phase).** Construct a binary tree of depth D (start with D=3). At each internal node, place the assumption that most divides the remaining action space. For each True/False branch, either add another assumption node or terminate with a leaf action. Stop expanding when further splits would not change the recommended action at that subtree.

5. **Assign leaf actions.** Each leaf gets a concrete action from the available action set. Multiple leaves may share the same action if it is robust across scenarios. Leaves can be physical actions (go to location, pick up object) or communication actions (send a message asking for information).

6. **Score each path with the Evaluator.** For every root-to-leaf path, compute three values using LLM estimation:
   - `L(S)`: Likelihood that the conjunction of assumptions along this path is true (0-1 scale)
   - `G(a)`: Goal-directed gain if action a is executed and the scenario holds (0-1 scale)
   - `C(a) = alpha * d(a) * is_move + beta * l(a) * is_comm`: Execution cost combining movement distance and communication token length (default alpha=1, beta=1)

7. **Compute utility and select.** For each leaf, calculate `U = L(S) * G(a) - lambda * C(a)` (default lambda=1). Select the action corresponding to the maximum-utility leaf.

8. **Execute and iterate.** Execute the selected action, update the observation history, and repeat from step 1 at the next planning cycle. The tree is rebuilt each cycle with fresh context.

9. **Tune hyperparameters.** Adjust tree depth D (higher = more nuanced but costlier), lambda (higher = more cost-sensitive, favoring cheap actions), and alpha/beta ratio (higher alpha penalizes movement more; higher beta penalizes communication more) based on domain requirements.

10. **Validate with ablation.** Test the system with individual components removed (no tree structure, no cost scoring, no likelihood estimation) to confirm each contributes to performance in your specific domain.

## Concrete Examples

**Example 1: Multi-agent household task planner**

User: "Build a planner for two agents collaborating to set a dinner table. Each agent has partial visibility and can move between rooms, pick up items, or send short messages."

Approach:
1. Define context schema: `{goal: "set table with plates, cups, napkins", agent_obs: [...], partner_messages: [...], available_actions: [goto, pickup, putdown, send_message]}`
2. Planner prompt generates reasoning: "I need plates. They might be in the kitchen cabinet. But my partner was heading to the kitchen, so they might already have them. I should check the dining room sideboard instead—or ask my partner."
3. Composer extracts assumptions and builds tree:

```
Root: "Plates are in kitchen cabinet"
├─ True: "Partner already picked up plates"
│  ├─ True → [goto] dining_room  (plates handled, do other tasks)
│  └─ False → [goto] kitchen     (go get the plates)
└─ False: "Plates are in dining room sideboard"
   ├─ True → [goto] dining_room  (check sideboard)
   └─ False → [send_message] "Do you know where the plates are?"
```

4. Evaluator scores each path:

```python
paths = [
    {"scenario": "kitchen+partner_has",  "L": 0.3, "G": 0.6, "C": 2, "U": 0.3*0.6 - 1*2 = -1.82},
    {"scenario": "kitchen+partner_hasnt","L": 0.4, "G": 0.9, "C": 3, "U": 0.4*0.9 - 1*3 = -2.64},
    {"scenario": "not_kitchen+sideboard","L": 0.2, "G": 0.8, "C": 1, "U": 0.2*0.8 - 1*1 = -0.84},
    {"scenario": "not_kitchen+not_side", "L": 0.1, "G": 0.5, "C": 0.5,"U": 0.1*0.5 - 1*0.5 = -0.45},
]
# Selected: path 4 → send_message (lowest cost, acceptable gain given high overall uncertainty)
# But if agent is near sideboard: path 3 cost drops, and it wins
```

Output: The agent selects the action with highest U given current position and observation history.

**Example 2: Implementing PCE as a Python module**

User: "Give me a reusable PCE decision tree implementation I can plug into my LLM agent loop."

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TreeNode:
    assumption: Optional[str] = None      # None for leaf nodes
    action: Optional[str] = None          # None for internal nodes
    true_branch: Optional["TreeNode"] = None
    false_branch: Optional["TreeNode"] = None

@dataclass
class ScoredPath:
    assumptions: list[tuple[str, bool]]   # (assumption_text, assumed_true)
    action: str
    likelihood: float                      # L(S): product of branch likelihoods
    gain: float                            # G(a): goal-directed gain
    cost: float                            # C(a): execution cost
    utility: float = 0.0                   # U = L*G - lambda*C

def extract_assumptions(reasoning_trace: str, llm_fn) -> list[str]:
    """Ask LLM to extract binary assumptions from a reasoning trace."""
    prompt = (
        "Extract the key uncertain assumptions from this reasoning trace. "
        "Return each as a yes/no statement.\n\n"
        f"Trace: {reasoning_trace}"
    )
    response = llm_fn(prompt)
    return [line.strip("- ") for line in response.strip().split("\n") if line.strip()]

def build_tree(assumptions: list[str], actions: list[str], llm_fn, depth: int = 3) -> TreeNode:
    """Recursively build a binary decision tree from assumptions."""
    if depth == 0 or not assumptions:
        best_action = llm_fn(
            f"Given these remaining assumptions {assumptions}, "
            f"which action from {actions} is best?"
        )
        return TreeNode(action=best_action)

    node = TreeNode(assumption=assumptions[0])
    remaining = assumptions[1:]
    node.true_branch = build_tree(remaining, actions, llm_fn, depth - 1)
    node.false_branch = build_tree(remaining, actions, llm_fn, depth - 1)
    return node

def enumerate_paths(node: TreeNode, path=None) -> list[ScoredPath]:
    """Walk the tree and collect all root-to-leaf paths."""
    if path is None:
        path = []
    if node.action is not None:
        return [ScoredPath(assumptions=list(path), action=node.action,
                           likelihood=0, gain=0, cost=0)]
    results = []
    results += enumerate_paths(node.true_branch, path + [(node.assumption, True)])
    results += enumerate_paths(node.false_branch, path + [(node.assumption, False)])
    return results

def score_paths(paths: list[ScoredPath], context: dict, llm_fn,
                alpha=1.0, beta=1.0, lam=1.0) -> list[ScoredPath]:
    """Score each path using LLM-estimated likelihood, gain, and cost."""
    for p in paths:
        scenario_desc = ", ".join(
            f"{a} is {'true' if v else 'false'}" for a, v in p.assumptions
        )
        p.likelihood = float(llm_fn(
            f"Rate 0-1 how likely this scenario is given observations: {scenario_desc}\n"
            f"Context: {context}"
        ))
        p.gain = float(llm_fn(
            f"Rate 0-1 how much action '{p.action}' advances the goal "
            f"given scenario: {scenario_desc}\nGoal: {context['goal']}"
        ))
        is_move = p.action.startswith("goto")
        is_comm = p.action.startswith("send")
        dist = float(llm_fn(f"Estimate distance for {p.action}")) if is_move else 0
        msg_len = float(llm_fn(f"Estimate token length for {p.action}")) if is_comm else 0
        p.cost = alpha * dist * int(is_move) + beta * msg_len * int(is_comm)
        p.utility = p.likelihood * p.gain - lam * p.cost
    return sorted(paths, key=lambda p: p.utility, reverse=True)

def pce_select_action(context: dict, llm_fn, depth=3, lam=1.0) -> str:
    """Full PCE pipeline: plan, compose tree, evaluate, return best action."""
    # Planner
    reasoning = llm_fn(f"Plan next action with reasoning: {context}")
    # Composer
    assumptions = extract_assumptions(reasoning, llm_fn)[:depth]
    tree = build_tree(assumptions, context["available_actions"], llm_fn, depth)
    paths = enumerate_paths(tree)
    # Evaluator
    scored = score_paths(paths, context, llm_fn, lam=lam)
    return scored[0].action
```

**Example 3: Adding PCE to an existing ReAct agent loop**

User: "I have a ReAct agent that keeps asking its partner agent redundant questions. How do I add PCE to reduce unnecessary communication?"

Approach:
1. Intercept the ReAct agent's reasoning step before it emits an action
2. Feed the reasoning trace into the Composer to extract assumptions
3. Build a decision tree where `send_message` is one possible leaf but physical actions are alternatives
4. Set beta > alpha (e.g., beta=2, alpha=1) to penalize communication cost more heavily
5. Only allow the agent to communicate when `send_message` wins on utility despite the higher cost penalty

```python
# In your ReAct loop, replace direct action emission:
# OLD:
#   action = llm(f"Choose action: {context}")
# NEW:
action = pce_select_action(
    context={"goal": task_goal, "obs": observations,
             "messages": msg_log[-3:], "available_actions": action_list},
    llm_fn=your_llm_call,
    depth=3,
    lam=1.0  # increase to further suppress costly actions
)
```

Result: Communication drops by 40-60% while task success rate improves, matching the paper's findings on C-WAH and TDW-MAT benchmarks.

## Best Practices

- **Do:** Start with tree depth D=3 and default hyperparameters (alpha=1, beta=1, lambda=1), then tune based on observed behavior. The paper found D=3 sufficient for most scenarios.
- **Do:** Include both physical and communication actions as possible leaves. PCE's value comes from evaluating communication against alternatives, not from eliminating it entirely.
- **Do:** Stop tree expansion early when further assumption splits would not change the leaf action—this saves LLM calls without losing decision quality.
- **Do:** Rebuild the decision tree each planning cycle with fresh observations. Stale trees with outdated assumptions degrade performance.
- **Avoid:** Setting lambda too high, which collapses the agent into always choosing the cheapest action regardless of goal-directed gain.
- **Avoid:** Using PCE for fully observable environments where uncertainty is negligible. The overhead of tree construction adds no value when the agent can directly observe all relevant state.
- **Avoid:** Trees deeper than 4-5 levels. The number of LLM scoring calls grows exponentially (2^D leaves), and marginal decision quality plateaus at D=3-4 per the paper's ablations.

## Error Handling

- **LLM returns non-numeric scores:** Wrap likelihood and gain extraction in try/except with fallback to 0.5 (neutral). Use structured output or few-shot prompting to enforce numeric responses.
- **All paths score negatively:** This means every action has cost exceeding expected gain. Fall back to the least-negative path, or trigger a "wait/observe" no-op action if your domain supports it.
- **Assumption extraction returns empty list:** The reasoning trace was too terse. Re-prompt the Planner with explicit instructions to articulate uncertainties: "What are you unsure about? List your assumptions."
- **Tree becomes unbalanced:** One branch has all high-utility leaves, the other all low. This is expected and correct—it means one scenario dominates. The agent will act on the dominant scenario.
- **Scoring latency too high:** Batch LLM calls for likelihood and gain estimation across all paths in a single prompt, or pre-filter obviously dominated paths (e.g., L < 0.05) before scoring gain and cost.

## Limitations

- PCE adds LLM calls proportional to 2^D (one per leaf for scoring). For real-time applications with D>3, latency may be prohibitive without batching or caching.
- Likelihood and gain estimates are LLM-generated and inherit the model's calibration biases. Overconfident models will underestimate uncertainty; underconfident models will over-communicate.
- The framework assumes assumptions can be meaningfully binarized. Continuous uncertainties (e.g., "the object is somewhere between room A and room B") require discretization that may lose information.
- PCE was validated on household and multi-agent transport tasks. Adversarial environments where other agents actively deceive may violate the assumption that LLM commonsense yields reasonable likelihood estimates.
- The framework does not maintain belief state across planning cycles—each tree is built fresh. For tasks requiring long-horizon belief tracking, PCE should be paired with an external state estimator.

## Reference

**Paper:** "From Assumptions to Actions: Turning LLM Reasoning into Uncertainty-Aware Planning for Embodied Agents" (ICLR 2026)
**arXiv:** https://arxiv.org/abs/2602.04326v1
**What to look for:** Section 3 for the full PCE formalization (Planner prompt templates, Composer tree-building algorithm, Evaluator utility formula), Section 5 for ablation results showing which components matter most, and Appendix for prompt templates and hyperparameter sensitivity analysis.