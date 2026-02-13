---
name: "who-deserves-reward-sharp"
description: "Apply SHARP (Shapley-based credit attribution) to design and optimize multi-agent systems where each agent's individual contribution is measured and rewarded. Use when the user says 'assign credit to agents', 'which agent contributed most', 'optimize multi-agent rewards', 'Shapley credit assignment', 'decompose rewards for agents', or 'fair reward attribution in multi-agent system'."
---

# SHARP: Shapley Credit-Based Optimization for Multi-Agent Systems

This skill enables Claude to design, implement, and debug multi-agent systems that use **decomposed reward signals** to solve the credit assignment problem. Instead of broadcasting a single success/failure signal to all agents, you apply the SHARP framework: a three-part reward decomposition (broadcast accuracy, Shapley marginal credit, tool-process quality) combined with group-relative advantage normalization. This produces agent-specific feedback that tells you exactly which agent helped, which hurt, and which was irrelevant -- then uses that signal to improve each agent independently.

## When to Use

- When the user is building a multi-agent pipeline (planner + workers) and needs to figure out **which agent to improve** when the system fails or succeeds.
- When the user asks to implement **reward shaping** or **credit assignment** for a reinforcement learning system with multiple LLM agents.
- When an existing multi-agent system trains poorly because all agents receive the same global reward and the user wants per-agent attribution.
- When the user wants to add **tool-use quality scoring** alongside task-level accuracy in a multi-agent RL loop.
- When designing a **counterfactual evaluation** system to measure each agent's marginal contribution by ablation.
- When the user needs to stabilize multi-agent RL training with **group-relative advantage normalization** across trajectory samples.

## Key Technique

SHARP addresses the core bottleneck in multi-agent RL: when a pipeline of agents produces a correct or incorrect answer, standard approaches broadcast one reward to all agents equally. This is wasteful -- an agent that made a perfect tool call gets the same penalty as the agent that hallucinated. SHARP decomposes the reward into three orthogonal signals:

1. **Broadcast Accuracy Reward** (`R_b`): A binary terminal signal (correct/incorrect) shared by all agents. This provides a baseline learning signal but cannot distinguish individual contributions.
2. **Marginal Credit Reward** (`R_mc`): Computed via counterfactual masking -- re-run the trajectory with agent `m` ablated (replaced by a null/default action) and measure the accuracy delta: `credit(m) = R_acc(full_trajectory) - R_acc(trajectory_without_m)`. Positive delta means the agent helped; negative means it hurt. This is a tractable approximation of full Shapley values.
3. **Tool-Process Reward** (`R_tool`): The average validity score across all tool invocations by an agent, independent of final task outcome. This rewards correct tool usage even when the overall task fails.

These combine as: `R_total(agent_m) = alpha * R_b + beta * R_mc + gamma * R_tool` (with recommended weights alpha=0.9, beta=0.9, gamma=0.1). Training is stabilized by computing advantages per-agent relative to their trajectory group: `A_hat(m) = (R_total(m) - mean_m) / (std_m + epsilon)`, preventing any single dominant agent from swamping gradients.

## Step-by-Step Workflow

1. **Define the agent hierarchy.** Identify distinct roles in the pipeline: a planner that decomposes the task, and N worker agents that execute subtasks via tool calls. Each role gets a role-specific system prompt but can share the same underlying model (parameter-sharing self-play).

2. **Instrument trajectory logging.** For each input query, record the full trajectory: planner output (subtask decomposition), each worker's tool calls and responses, and the final assembled answer. Store these as structured objects with agent IDs, action sequences, and timestamps.

3. **Compute the broadcast accuracy reward.** Compare the final answer to ground truth. Assign `R_b = 1.0` if correct, `R_b = 0.0` otherwise. Broadcast this value identically to every agent in the trajectory.

4. **Compute marginal credit via counterfactual ablation.** For each worker agent `m`, construct a counterfactual trajectory where agent `m`'s output is replaced with a null/default response (e.g., empty string or "I don't know"). Re-evaluate the final answer accuracy on this ablated trajectory. The marginal credit is: `credit(m) = R_acc(original) - R_acc(ablated)`. For the planner, aggregate: `R_mc(planner) = lambda * mean(max(credit(m), 0) for m in workers)`.

5. **Score tool-process quality.** For each tool invocation by each agent, assign a validity score: 1.0 if the tool returned a valid, parseable result; 0.0 if it threw an error, timed out, or returned malformed output. Average across all calls: `R_tool(m) = sum(scores) / num_calls`. Agents with zero tool calls get `R_tool = 0`.

6. **Combine into aggregate per-agent reward.** `R(m) = 0.9 * R_b + 0.9 * R_mc(m) + 0.1 * R_tool(m)`. Adjust weights based on domain: increase gamma when tool reliability is a bottleneck; increase beta when you need sharper agent differentiation.

7. **Normalize advantages within trajectory groups.** Sample G trajectories per query (e.g., G=8). For each agent role `m`, compute mean and standard deviation of `R(m)` across the G samples. Normalize: `A_hat(m) = (R(m) - mean) / (std + 1e-6)`. This prevents reward scale differences between agent roles from destabilizing training.

8. **Apply policy gradient update.** Use the normalized advantages in a clipped surrogate objective (GRPO/PPO-style): `loss = min(ratio * A_hat, clip(ratio, 1-eps, 1+eps) * A_hat)` with eps=0.2. Aggregate across all agents and trajectories.

9. **Iterate and monitor convergence.** Track per-agent reward curves independently. A healthy system shows all agents improving; if one agent's marginal credit is consistently near zero, consider removing it or redesigning its role.

10. **Deploy with credit dashboards.** In production, log marginal credit scores per agent per query. Use these to identify which agent degrades on specific input types and target fine-tuning accordingly.

## Concrete Examples

**Example 1: Multi-agent QA pipeline with credit attribution**

User: "I have a planner agent that breaks questions into sub-questions, a retriever agent that searches documents, and a synthesizer agent that combines answers. How do I figure out which agent to blame when the system gets a wrong answer?"

Approach:
1. Log the full trajectory: planner's sub-questions, retriever's search results per sub-question, synthesizer's final answer.
2. Compute broadcast reward: compare final answer to ground truth (0 or 1).
3. Ablate the retriever: replace its search results with empty results, re-run the synthesizer, check if accuracy changes. If accuracy stays the same, the retriever's contribution was zero (it retrieved irrelevant content).
4. Ablate the synthesizer: feed perfect retrieval results from ground truth, check if the synthesizer produces the right answer. If it still fails, the synthesizer is at fault.
5. Score tool process: did the retriever's API calls return valid JSON? Did the synthesizer format output correctly?

Output:
```json
{
  "query": "What year did the company merge with Acme Corp?",
  "final_answer": "2019",
  "correct_answer": "2021",
  "broadcast_reward": 0.0,
  "agent_credits": {
    "planner": {"marginal_credit": 0.0, "tool_process": 1.0, "total": 0.09},
    "retriever": {"marginal_credit": -0.5, "tool_process": 0.75, "total": -0.375},
    "synthesizer": {"marginal_credit": 0.0, "tool_process": 1.0, "total": 0.09}
  },
  "diagnosis": "Retriever fetched outdated document (2019 annual report instead of 2021). Synthesizer correctly extracted the year from what it was given."
}
```

**Example 2: Implementing the reward computation in Python**

User: "Give me code to compute SHARP rewards for my multi-agent trajectories."

```python
from dataclasses import dataclass
from typing import Callable

@dataclass
class AgentTrajectory:
    agent_id: str
    actions: list[dict]       # tool calls and responses
    tool_scores: list[float]  # per-call validity (0 or 1)

@dataclass
class SystemTrajectory:
    query: str
    agents: list[AgentTrajectory]
    final_answer: str
    ground_truth: str

def compute_sharp_rewards(
    trajectory: SystemTrajectory,
    evaluate_fn: Callable[[SystemTrajectory], float],
    alpha: float = 0.9,
    beta: float = 0.9,
    gamma: float = 0.1,
) -> dict[str, float]:
    """Compute per-agent SHARP rewards for a single trajectory."""
    # 1. Broadcast accuracy reward
    r_broadcast = evaluate_fn(trajectory)

    rewards = {}
    for i, agent in enumerate(trajectory.agents):
        # 2. Marginal credit via counterfactual ablation
        ablated = ablate_agent(trajectory, i)
        r_ablated = evaluate_fn(ablated)
        r_marginal = r_broadcast - r_ablated

        # 3. Tool-process reward
        if agent.tool_scores:
            r_tool = sum(agent.tool_scores) / len(agent.tool_scores)
        else:
            r_tool = 0.0

        # 4. Aggregate
        rewards[agent.agent_id] = alpha * r_broadcast + beta * r_marginal + gamma * r_tool

    return rewards

def ablate_agent(traj: SystemTrajectory, agent_idx: int) -> SystemTrajectory:
    """Replace one agent's output with a null default."""
    ablated_agents = []
    for i, agent in enumerate(traj.agents):
        if i == agent_idx:
            ablated_agents.append(AgentTrajectory(
                agent_id=agent.agent_id,
                actions=[{"tool": "null", "result": ""}],
                tool_scores=[0.0],
            ))
        else:
            ablated_agents.append(agent)
    return SystemTrajectory(
        query=traj.query,
        agents=ablated_agents,
        final_answer=recompute_answer(ablated_agents),
        ground_truth=traj.ground_truth,
    )

def normalize_advantages(
    rewards_per_group: list[dict[str, float]],
    eps: float = 1e-6,
) -> list[dict[str, float]]:
    """Group-relative normalization across G sampled trajectories."""
    from statistics import mean, stdev
    agent_ids = rewards_per_group[0].keys()
    stats = {}
    for aid in agent_ids:
        vals = [r[aid] for r in rewards_per_group]
        stats[aid] = (mean(vals), stdev(vals) if len(vals) > 1 else 0.0)

    normalized = []
    for rewards in rewards_per_group:
        normalized.append({
            aid: (rewards[aid] - stats[aid][0]) / (stats[aid][1] + eps)
            for aid in agent_ids
        })
    return normalized
```

**Example 3: Diagnosing a stalled multi-agent training run**

User: "My multi-agent system stopped improving after 50 steps. All agents get the same reward. How do I fix this?"

Approach:
1. The symptom (all agents get identical reward) indicates you are using only broadcast rewards with no credit decomposition.
2. Add marginal credit: for each training batch, ablate each agent and measure the accuracy delta. Log these per-agent credits.
3. Check if one agent dominates: if the planner's marginal credit is always high but workers are near zero, workers are not learning because they receive no differentiated signal.
4. Add tool-process reward (gamma=0.1) so workers get feedback on tool-call quality independent of final accuracy.
5. Apply group-relative normalization: sample 8 trajectories per query, normalize per-agent rewards within each group. This prevents a single high-performing trajectory from overwhelming gradients.

Output diagnosis:
```
Before SHARP: All 3 agents receive R=0.42 per trajectory (broadcast only)
After SHARP decomposition:
  - Planner:   R=0.42 (broadcast) + 0.31 (marginal) + 0.00 (tool) = 0.73
  - Retriever:  R=0.42 (broadcast) + 0.08 (marginal) + 0.85 (tool) = 1.35
  - Synthesizer: R=0.42 (broadcast) - 0.15 (marginal) + 0.92 (tool) = 1.19

Diagnosis: Synthesizer has negative marginal credit -- it is degrading
the retriever's good results. Focus fine-tuning on the synthesizer.
```

## Best Practices

- **Do:** Always log full trajectories with agent boundaries so counterfactual ablation is possible after the fact.
- **Do:** Start with the recommended weights (alpha=0.9, beta=0.9, gamma=0.1) and adjust only after observing per-agent reward distributions.
- **Do:** Sample multiple trajectories per query (G >= 4) for stable advantage normalization. Eight rollouts per query is the tested default.
- **Do:** Clamp marginal credit for the planner to positive values only (`max(credit, 0)`) since the planner cannot be meaningfully ablated without destroying the entire trajectory.
- **Avoid:** Using marginal credit alone without the broadcast reward. Agents need a baseline task-success signal; marginal credit only captures relative contribution.
- **Avoid:** Setting gamma too high. Tool-process reward is a proxy metric; over-weighting it rewards agents for making well-formed but useless tool calls.
- **Avoid:** Full Shapley value computation with more than 5-6 agents. The number of coalitions grows factorially. Use single-agent ablation (marginal contribution) as the tractable approximation.

## Error Handling

- **Ablation produces nonsensical output:** If removing an agent causes the pipeline to crash rather than degrade gracefully, wrap the ablated trajectory in a try/catch that returns `R_acc = 0`. This correctly assigns high marginal credit to essential agents.
- **Tool-process scores are all 1.0 or all 0.0:** Your validity function is too coarse. Add granularity: partial credit for valid-but-incomplete results, penalties for retries, score based on response latency relative to a baseline.
- **Advantage normalization divides by near-zero std:** This happens when all G trajectories produce identical rewards for an agent. The epsilon term (1e-6) handles numerical stability, but the root cause is insufficient trajectory diversity. Increase sampling temperature or add exploration noise.
- **One agent's reward dominates training:** Check that you are normalizing per-agent, not globally. Each agent role should have its own mean/std computed across the trajectory group.

## Limitations

- **Counterfactual ablation requires re-execution.** You must be able to re-run downstream agents with modified inputs. If agents have side effects (database writes, API calls with rate limits), ablation is expensive or impractical.
- **Assumes decomposable pipelines.** SHARP works best when agents operate sequentially in a clear pipeline. Highly parallel or message-passing architectures where agents interact bidirectionally make clean ablation difficult.
- **Marginal credit is an approximation.** Single-agent ablation captures first-order contributions but misses interaction effects (e.g., two agents that are individually useless but powerful together). Full Shapley values capture this but are computationally intractable beyond ~6 agents.
- **Requires ground truth for accuracy reward.** The broadcast reward needs a correct answer to compare against. For open-ended generation tasks without clear ground truth, you need a proxy evaluator, which introduces its own noise.
- **Training scale.** The original paper used 64 A100 GPUs with batch size 256. Smaller setups will need reduced batch sizes and more gradient steps, which may affect convergence properties.

## Reference

[SHARP: Shapley Credit-based Optimization for Multi-Agent System](https://arxiv.org/abs/2602.08335v1) -- Look for Section 3 (reward decomposition formulas), Algorithm 1 (training loop pseudocode), and Table 2 (ablation study showing the contribution of each reward component).