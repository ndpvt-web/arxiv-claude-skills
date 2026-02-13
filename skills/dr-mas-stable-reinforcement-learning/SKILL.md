---
name: "dr-mas-stable-reinforcement-learning"
description: "Design and implement stable reinforcement learning pipelines for multi-agent LLM systems using agent-wise advantage normalization (Dr. MAS). Use when: 'train multi-agent LLM system with RL', 'stabilize GRPO training for multiple agents', 'fix gradient spikes in multi-agent reinforcement learning', 'implement agent-wise reward normalization', 'build RL post-training pipeline for collaborating LLM agents', 'set up multi-agent math reasoning or search with RL'"
---

# Dr. MAS: Stable Reinforcement Learning for Multi-Agent LLM Systems

This skill enables Claude to design, implement, and debug reinforcement learning post-training pipelines for multi-agent LLM systems where multiple specialized agents (e.g., solver/verifier, searcher/answerer) collaborate on tasks. The core technique is **agent-wise advantage normalization** from the Dr. MAS paper, which replaces the standard global reward baseline in GRPO with per-agent statistics, eliminating the gradient-norm explosions that plague naive multi-agent RL training.

## When to Use

- When the user wants to **train a multi-agent LLM system with RL** (e.g., GRPO, PPO variants) and needs a stable training recipe
- When the user has a **solver-verifier** or **searcher-answerer** multi-agent pipeline and wants to fine-tune agents jointly with reinforcement learning
- When the user observes **gradient spikes or training instability** in a multi-agent RL setup and needs to diagnose/fix it
- When the user is implementing **GRPO (Group Relative Policy Optimization)** for multiple LLM agents with different roles or reward distributions
- When the user needs to build an **end-to-end orchestration framework** for multi-agent RL training with heterogeneous model assignments
- When the user asks how to **normalize rewards or advantages** in a system where different agents receive rewards of different scales or distributions

## Key Technique

### The Problem: Global Normalization Breaks Multi-Agent RL

Standard GRPO normalizes advantages across an entire group of sampled outputs using global reward statistics (mean `mu`, std `sigma`). In a multi-agent system, each agent contributes to the trajectory at different steps, and their reward distributions can differ significantly. Agent k has its own reward mean `mu_k` and variance `sigma_k^2`. The per-agent gradient second moment decomposes as:

```
E[||gradient_k||^2] ~ E[||z||^2] * (sigma_k^2 + (mu_k - mu)^2) / sigma^2
```

The multiplicative factor `(sigma_k^2 + (mu_k - mu)^2) / sigma^2` inflates whenever agent k's reward distribution deviates from the global distribution. This causes gradient-norm spikes that destabilize training -- especially when agents have different roles (a verifier that mostly outputs "correct"/"incorrect" vs. a solver producing long reasoning chains).

### The Fix: Agent-Wise Advantage Normalization

Dr. MAS replaces global normalization with per-agent normalization. For each agent k, advantages are computed using only the reward statistics from steps where agent k was active:

```
A_agent(i, k) = (R_i - mu_k) / sigma_k
```

where `mu_k` and `sigma_k` are the mean and standard deviation of rewards in the subset of samples where agent k acts (the "active set" for agent k). This forces the inflation factor to exactly 1, eliminating gradient-scale mismatch between agents. The fix is simple to implement -- it requires only filtering rewards by agent identity before computing normalization statistics -- yet it yields large gains: +5.6% on math reasoning, +15.2% on multi-turn search over vanilla GRPO.

### The Framework: End-to-End Multi-Agent RL Training

Beyond the normalization fix, Dr. MAS provides an architectural blueprint for multi-agent RL training: (1) an orchestrator that manages distributed rollout collection with user-defined multi-agent control flow, (2) an agent-to-model mapping that allows heterogeneous model assignments (e.g., a 4B solver with an 8B verifier), (3) per-agent optimization configs (separate learning rates, batch sizes), and (4) shared resource pooling across LLM backends.

## Step-by-Step Workflow

1. **Define the multi-agent topology**: Specify each agent's role, system prompt, and action space. Common patterns: solver + verifier (2-agent), or verifier + searcher + answerer (3-agent). Define the control flow (who acts first, termination conditions, max turns).

2. **Assign models to agents**: Map each logical agent to a physical LLM. Agents can share the same model (parameter-tied) or use different models/sizes. For cost efficiency, assign smaller models to simpler roles (e.g., 3B for verification, 7B for reasoning).

3. **Design the reward function**: Use binary rule-based rewards (1 for correct final answer, 0 otherwise). Add small penalties for invalid actions (e.g., 0.1 for malformed output, 0.01 for unnecessary tool calls). The reward is assigned to the trajectory but each agent's contribution is tracked separately.

4. **Implement trajectory collection with agent tracking**: During rollout, tag each generation step with the agent identity that produced it. Store tuples of `(agent_id, prompt, response, reward)`. This agent-level metadata is critical for the per-agent normalization.

5. **Compute per-agent advantage normalization**: For each agent k in the batch, filter the reward set to only include samples where agent k was active. Compute `mu_k = mean(rewards_k)` and `sigma_k = std(rewards_k)`. Normalize: `advantage = (reward - mu_k) / max(sigma_k, epsilon)`. Use `epsilon = 1e-6` to avoid division by zero.

6. **Apply per-agent gradient updates**: Compute the policy gradient loss using the agent-wise normalized advantages. If agents share a model, accumulate gradients from all agents before stepping. If agents use separate models, update each model independently with its own optimizer config.

7. **Configure per-agent hyperparameters**: Set learning rate (typically 1e-6), group size (8 for math-style tasks, 5 for search-style tasks), and batch sizes per agent. Agents with higher-variance rewards may benefit from larger group sizes.

8. **Monitor gradient norms per agent**: Log the gradient L2 norm for each agent separately during training. If any agent's gradient norm exceeds 10x the median, this indicates the normalization is not working correctly or there is a reward distribution issue.

9. **Evaluate with avg@K and pass@K metrics**: Sample K completions (e.g., K=16) per problem. `avg@K` is the mean accuracy across samples; `pass@K` is the fraction of problems where at least one sample is correct. Both metrics matter -- avg@K measures reliability, pass@K measures coverage.

10. **Iterate on agent topology and reward shaping**: If training plateaus, consider adjusting the number of interaction turns, adding intermediate rewards for sub-tasks, or changing the agent-model assignment to give harder roles more capable models.

## Concrete Examples

**Example 1: Math Solver-Verifier with Agent-Wise RL**

User: "I have a 2-agent math reasoning system where a Solver generates solutions and a Verifier checks them. I want to fine-tune both with GRPO but training is unstable with gradient spikes."

Approach:
1. Identify the root cause: the Solver produces long reasoning with high reward variance; the Verifier outputs short judgments with low variance. Global normalization over-weights Verifier gradients.
2. Implement agent-wise normalization:

```python
import torch

def compute_agent_wise_advantages(rewards, agent_ids, epsilon=1e-6):
    """
    rewards: Tensor of shape (batch_size,) - per-trajectory rewards
    agent_ids: List[List[str]] - agent identities per step per trajectory
    Returns: dict mapping agent_id -> normalized advantages tensor
    """
    unique_agents = set(aid for traj in agent_ids for aid in traj)
    agent_advantages = {}

    for agent in unique_agents:
        # Mask: which trajectories had this agent active
        mask = torch.tensor([
            agent in traj_agents for traj_agents in agent_ids
        ], dtype=torch.bool)

        agent_rewards = rewards[mask]
        mu_k = agent_rewards.mean()
        sigma_k = agent_rewards.std().clamp(min=epsilon)
        advantages = (agent_rewards - mu_k) / sigma_k

        agent_advantages[agent] = (mask, advantages)

    return agent_advantages
```

3. Replace the global advantage computation in the GRPO loss with this per-agent version.
4. Train with lr=1e-6, group_size=8, batch_size=32 on competition math problems.

Output: Gradient norms stabilize within the first 100 steps. Expect ~5% avg@16 improvement over vanilla GRPO on AIME-level benchmarks.

**Example 2: Multi-Turn Search Pipeline (3 Agents)**

User: "I'm building a 3-agent search system: a Verifier decides if we have enough info, a Searcher retrieves evidence, and an Answerer synthesizes. How do I set up RL training?"

Approach:
1. Define the control flow:

```python
AGENT_CONFIG = {
    "verifier": {
        "model": "Qwen2.5-3B",  # lightweight judge
        "system_prompt": "Decide if the available evidence is sufficient to answer the question. Output SUFFICIENT or SEARCH_MORE.",
        "max_turns": 5,
    },
    "searcher": {
        "model": "Qwen2.5-7B",  # stronger retrieval reasoning
        "system_prompt": "Given the question and current evidence, generate a search query to find missing information.",
        "max_turns": 5,
        "tools": ["web_search"],
    },
    "answerer": {
        "model": "Qwen2.5-7B",  # shared with searcher
        "system_prompt": "Synthesize the evidence into a final answer.",
        "max_turns": 1,
    },
}

CONTROL_FLOW = """
loop(max=5):
    verifier -> if SUFFICIENT: break, else: continue
    searcher -> retrieves evidence
answerer -> final answer
"""
```

2. Set up reward: binary (1 if final answer matches gold, 0 otherwise). Penalize unnecessary search calls with -0.01 per extra turn.
3. Implement agent-wise normalization -- critical here because the Verifier's reward distribution (binary classification) differs sharply from the Searcher's (reward depends on retrieval quality over multiple turns).
4. Train with group_size=5, batch_size=128. Use shared resource pooling: Searcher and Answerer share one model backend; Verifier uses a separate smaller one.

Output: Expect ~15% avg@16 improvement on multi-hop QA benchmarks (HotpotQA, MuSiQue) over single-agent or globally-normalized baselines.

**Example 3: Diagnosing Gradient Instability**

User: "My multi-agent GRPO training has periodic gradient spikes every ~50 steps. How do I fix this?"

Approach:
1. Add per-agent gradient norm logging:

```python
for agent_name, agent_model in agents.items():
    total_norm = torch.nn.utils.clip_grad_norm_(
        agent_model.parameters(), max_norm=float('inf')
    )
    wandb.log({f"grad_norm/{agent_name}": total_norm.item()})
```

2. Check if spikes correlate with a specific agent. If one agent's gradient norm is consistently 5-10x higher than others, the global normalization is the likely cause.
3. Verify reward distributions per agent: compute running mean and variance of rewards for each agent. Large `(mu_k - mu)^2 / sigma^2` values confirm the diagnosis.
4. Switch to agent-wise normalization. If spikes persist, also check for degenerate rewards (all 0s or all 1s in an agent's active set within a batch -- increase group size to fix).

## Best Practices

- **Do** track agent identity at every generation step during rollout collection. Without this metadata, per-agent normalization is impossible to implement correctly.
- **Do** use binary rule-based rewards rather than model-based reward signals. The paper shows binary rewards (correct/incorrect) work well and avoid reward model instability compounding with multi-agent instability.
- **Do** start with homogeneous model assignments (same model for all agents), verify training stability, then experiment with heterogeneous assignments for efficiency gains.
- **Do** set `epsilon >= 1e-6` in the standard deviation denominator. With small group sizes, a single-agent's reward set can have zero variance (all correct or all incorrect), causing NaN gradients.
- **Avoid** using a single global advantage normalization across all agents. This is the primary source of instability the paper identifies and fixes.
- **Avoid** very small group sizes (< 4) per agent. The per-agent statistics become unreliable with too few samples, defeating the purpose of normalization.
- **Avoid** mixing reward scales across agents (e.g., one agent gets rewards in [0, 1] and another in [0, 100]). If unavoidable, agent-wise normalization handles this, but consistent scales make debugging easier.

## Error Handling

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| NaN gradients | Zero variance in agent's reward subset (all same reward) | Increase group size or add epsilon to std computation |
| One agent collapses to constant output | Reward signal uninformative for that agent | Add intermediate rewards or adjust penalty scale |
| Training diverges after initial stability | Learning rate too high for one agent | Use per-agent learning rates; reduce for the diverging agent |
| No improvement despite stable gradients | Agent topology issue, not RL issue | Verify the multi-agent system works in supervised/prompting mode before applying RL |
| Memory errors with multiple models | Resource contention | Use shared model backends for agents with the same architecture; implement Ray-style placement groups |

## Limitations

- **Requires multiple samples per agent per batch**: Agent-wise normalization needs a reasonable number of reward samples per agent to compute stable statistics. With very small batches or many agents, per-agent sample counts can be too low.
- **Does not address reward attribution**: The method normalizes rewards but does not solve credit assignment between agents. If the Solver produces a wrong answer and the Verifier approves it, both receive reward 0 -- there is no decomposition of blame.
- **Binary rewards only in the paper**: The experiments use binary (correct/incorrect) rewards. The technique should generalize to continuous rewards, but this is not empirically validated.
- **Assumes static agent topology**: The number and roles of agents are fixed during training. Dynamic agent spawning or variable-length agent chains are not supported.
- **Compute intensive**: Multi-agent RL requires serving multiple LLM backends simultaneously. The paper uses H100 GPUs; smaller setups may need to time-share backends or reduce model sizes significantly.
- **Not validated beyond 3 agents**: The paper tests 2-agent (math) and 3-agent (search) systems. Scaling to many more agents may surface additional instability modes.

## Reference

**Paper**: [Dr. MAS: Stable Reinforcement Learning for Multi-Agent LLM Systems](https://arxiv.org/abs/2602.08847v1) (Feng et al., 2026)

**Key takeaway**: Look at Section 4 (Theoretical Analysis) for the gradient variance decomposition proving why global normalization fails, and Algorithm 1 for the complete training procedure with agent-wise normalization.