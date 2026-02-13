---
name: "dynaweb-model-based-reinforcement-learning"
description: "Build model-based RL training pipelines for web agents using learned world models (environment simulators) that predict next web states from actions. Implements the DynaWeb Dyna-style architecture: train a world model on interaction traces, generate imagined rollouts ('dreaming'), interleave real expert trajectories, and optimize the agent policy with group sequence policy optimization. Trigger phrases: 'build a web agent with world model', 'train web agent with model-based RL', 'create a web environment simulator', 'DynaWeb-style agent training', 'dream rollouts for web navigation', 'Dyna architecture for browser agents'."
---

# DynaWeb: Model-Based Reinforcement Learning for Web Agents

This skill enables Claude to design and implement model-based reinforcement learning (MBRL) pipelines for training autonomous web agents. The core idea from DynaWeb (Ding et al., 2026) is to train a *world model* that predicts how web pages change in response to agent actions, then use that model as a simulator ("dream environment") to generate massive quantities of training trajectories without touching live websites. This eliminates the cost, latency, and safety risks of live web interaction while dramatically improving sample efficiency.

## When to Use

- When building a web navigation agent and the user needs a training pipeline that avoids live-web interaction during policy optimization
- When the user wants to implement a Dyna-style architecture that mixes real demonstrations with model-generated ("imagined") rollouts
- When designing an environment simulator for browser-based tasks using LLM-predicted state transitions
- When the user asks about training web agents with reinforcement learning and needs a scalable approach beyond pure supervised fine-tuning
- When implementing Group Sequence Policy Optimization (GSPO) or similar trajectory-level RL objectives for agentic tasks
- When the user needs to represent web pages as accessibility trees (AXTree) for structured agent observation

## Key Technique

**World Model as Environment Simulator.** Instead of having the agent interact with real websites (slow, expensive, risky), DynaWeb trains a separate LLM to act as the environment. Given the current page state (represented as an accessibility tree) and the agent's action, the world model predicts *what changes* on the page. Crucially, it does not predict the entire next page from scratch -- it predicts a natural-language *state change description* (delta), which is then applied to the current state. This decomposition (`o_{t+1} = apply(o_t, delta(o_t, a_t))`) dramatically reduces hallucination compared to generating full pages and keeps predictions grounded in the actual current state.

**Dyna-Style Mixed Training.** The training loop interleaves two data sources: (1) *imagined rollouts* where the agent policy proposes actions and the world model predicts outcomes for up to 5 steps, and (2) *real expert trajectories* sampled from demonstration datasets. The paper finds that replacing ~40-50% of dreamed trajectories with real expert data provides the best tradeoff -- enough imagination for exploration, enough real data for stability. Both trajectory types feed into the same policy optimization objective (GSPO), which computes advantages at the trajectory level using a geometric mean of token-wise likelihood ratios.

**Practical Architecture Choices.** The agent policy is a fine-tuned Llama-3.1-8B-Instruct that outputs chain-of-thought reasoning followed by atomic browser actions (click, type, scroll, goback, stop). The world model is a larger LLM (120B parameters) fine-tuned on real web interaction traces. Rollouts are capped at 5 steps to prevent compounding errors, and a model-based reward signal (binary task completion) is used for the RL objective. This means you need two models: a smaller policy model and a larger world model.

## Step-by-Step Workflow

1. **Define the observation format as an accessibility tree (AXTree).** Write a parser that extracts the structured accessibility tree from web pages, capturing element IDs, types (link, button, textbox), text content, and hierarchical nesting. This is the agent's view of the world -- it should include the current URL and all visible interactive elements.

2. **Define the action space as atomic browser operations.** Implement a fixed set of actions: `click [element_id]`, `type [element_id] [text]`, `scroll [up|down]`, `goback`, `stop [answer]`. Each action must reference elements by their AXTree ID, making actions grounded in the observation.

3. **Collect real interaction trajectories.** Gather expert demonstrations of web tasks as sequences of `(observation, reasoning, action, next_observation)` tuples. Each trajectory starts from a task instruction and ends at task completion or failure. These serve dual purpose: training the world model and interleaving as real data during RL.

4. **Compute state-change deltas for world model training data.** For each consecutive `(o_t, a_t, o_{t+1})` pair in the real trajectories, compute `delta(o_t, o_{t+1})` -- a natural-language description of what changed (e.g., "The search results page now shows 10 items matching query 'headphones'. The URL changed to /search?q=headphones."). Train or prompt a model to generate these deltas from observation pairs.

5. **Train the world model.** Fine-tune a large LLM on the objective: given task instruction `I`, current observation `o_t`, and action `a_t`, predict the reward `r` and state-change delta `delta`. The loss is: `L = -sum(log p(r, delta | I, o_t, a_t))`. Use the real trajectory data with computed deltas as training examples.

6. **Implement the dream rollout generator.** Starting from a real initial observation `o_1` sampled from the dataset, run the agent policy to get action `a_1`, feed `(o_1, a_1)` to the world model to get predicted delta and reward, apply the delta to get `o_2`, and repeat for up to 5 steps. Terminate early if the world model predicts a terminal state or the agent outputs `stop`.

7. **Implement trajectory interleaving.** Build a data loader that, for each training batch, randomly replaces ~40-50% of dreamed trajectories with real expert trajectories. Both trajectory types should be formatted identically: a sequence of `(observation, reasoning, action)` tuples with a final reward signal.

8. **Implement Group Sequence Policy Optimization (GSPO).** For each task prompt, sample G rollout trajectories. Compute binary advantage: `A_i = 1` if reward is above the group mean, `-1` otherwise. Optimize the clipped surrogate objective using the geometric mean of per-token likelihood ratios across the full trajectory (not per-token updates). Use learning rate `1e-6`, clip epsilon `0.2`, and 8 rollout samples per prompt.

9. **Run the Dyna training loop.** Alternate between: (a) generating a batch of dreamed rollouts using the current policy + world model, (b) sampling real expert trajectories, (c) interleaving them, and (d) running one GSPO update step. Repeat for ~10 epochs. Validate on held-out tasks periodically.

10. **Evaluate with a self-assessment reward model.** At evaluation time, have the agent interact with the real environment and use a model-based judge to assess binary task completion (`r in {0, 1}`). Compare against supervised fine-tuning baselines to confirm the RL training improved performance.

## Concrete Examples

**Example 1: Building a Web Shopping Agent Trainer**

User: "I want to train an agent that can navigate e-commerce sites to find and compare products. How do I set up a DynaWeb-style training pipeline?"

Approach:
1. Define AXTree observations for shopping pages -- product listings, filters, search bars, cart buttons each get element IDs
2. Collect 500+ expert trajectories of shopping tasks (search for item, apply filters, compare prices, add to cart)
3. Compute state-change deltas: "After clicking 'Sort by Price', the product list reordered with cheapest first. Item at position 1 changed from 'Wireless Mouse $45' to 'Basic Mouse $12'."
4. Fine-tune a world model (e.g., Llama-70B) on these (state, action, delta) triples
5. Initialize policy from Llama-8B fine-tuned on the expert demos (SFT baseline)
6. Run Dyna loop: generate 5-step dream rollouts from random initial product pages, interleave 50% real expert trajectories, optimize with GSPO

Output structure:
```python
# World model training data format
{
    "instruction": "Find the cheapest wireless mouse under $30",
    "observation": "[AXTree] URL: /products\n  [1] searchbox 'Search products'\n  [2] link 'Electronics'\n  ...",
    "action": "type [1] wireless mouse",
    "reward": 0,
    "state_delta": "Search box now contains 'wireless mouse'. Page unchanged until submit."
}

# Dream rollout format (fed to GSPO)
{
    "task": "Find cheapest wireless mouse under $30",
    "trajectory": [
        {"obs": "[AXTree]...", "thought": "I need to search for wireless mouse", "action": "type [1] wireless mouse"},
        {"obs": "[AXTree]...", "thought": "Now I submit the search", "action": "click [3]"},
        {"obs": "[AXTree]...", "thought": "I see results, let me sort by price", "action": "click [15]"},
        {"obs": "[AXTree]...", "thought": "The cheapest is $18, under budget", "action": "stop [$18 Basic Wireless Mouse]"}
    ],
    "reward": 1,
    "source": "dreamed"  # or "expert"
}
```

**Example 2: Implementing the State-Delta World Model**

User: "How do I implement the world model that predicts page changes instead of whole pages?"

Approach:
1. Structure training examples as `(instruction, current_axtree, action) -> (reward_token, delta_description)`
2. Fine-tune with standard causal LM loss on the concatenated output
3. At inference, parse the predicted delta and apply it to the current AXTree

```python
# World model inference pseudocode
def world_model_step(current_obs: str, action: str, task: str, wm_model) -> tuple[str, float]:
    prompt = f"""Task: {task}
Current page (AXTree):
{current_obs}

Agent action: {action}

Predict the reward (0 or 1) and describe what changes on the page:"""

    response = wm_model.generate(prompt, temperature=0.7, top_p=0.9)

    # Parse response: "Reward: 0\nChanges: The page navigated to /search?q=mouse.
    # The product grid now shows 24 items. Filter sidebar appeared with
    # Price, Brand, Rating options."
    reward = parse_reward(response)
    delta = parse_delta(response)

    # Apply delta to current observation
    next_obs = apply_state_change(current_obs, delta)
    return next_obs, reward

def generate_dream_rollout(initial_obs: str, task: str, policy, wm_model, max_steps=5):
    trajectory = []
    obs = initial_obs
    for step in range(max_steps):
        thought, action = policy.generate(task, obs, trajectory)
        next_obs, reward = world_model_step(obs, action, task, wm_model)
        trajectory.append({"obs": obs, "thought": thought, "action": action})
        obs = next_obs
        if action.startswith("stop") or reward == 1:
            break
    return trajectory, reward
```

**Example 3: GSPO Implementation**

User: "How do I implement the Group Sequence Policy Optimization from DynaWeb?"

```python
import torch

def compute_gspo_loss(policy, trajectories, old_log_probs, clip_epsilon=0.2):
    """
    trajectories: list of G rollouts for the same task prompt
    Each trajectory has .tokens, .reward (0 or 1), .length
    """
    # Compute group-level advantage
    rewards = torch.tensor([t.reward for t in trajectories])
    mean_reward = rewards.mean()
    advantages = torch.where(rewards > mean_reward, 1.0, -1.0)

    total_loss = 0.0
    for i, traj in enumerate(trajectories):
        # Compute current log probs for full trajectory
        current_log_probs = policy.log_prob(traj.tokens)  # per-token

        # Geometric mean of per-token ratios (sequence-level ratio)
        token_ratios = torch.exp(current_log_probs - old_log_probs[i])
        seq_ratio = token_ratios.prod() ** (1.0 / len(token_ratios))

        # Clipped surrogate
        clipped_ratio = torch.clamp(seq_ratio, 1 - clip_epsilon, 1 + clip_epsilon)
        loss_i = -torch.min(seq_ratio * advantages[i], clipped_ratio * advantages[i])
        total_loss += loss_i

    return total_loss / len(trajectories)
```

## Best Practices

- **Do:** Cap dream rollouts at 4-5 steps. Longer rollouts compound world model errors and degrade training signal. The paper shows performance peaks at 5 steps and drops sharply after.
- **Do:** Predict state *deltas* rather than full next states. This grounds predictions in the actual current page and dramatically reduces hallucination in the world model.
- **Do:** Interleave 40-50% real expert trajectories with dreamed rollouts. This ratio provides the best stability-exploration tradeoff. Going below 30% real data destabilizes training; going above 60% underutilizes the world model.
- **Do:** Use a larger model for the world model than for the agent policy. The world model needs to accurately simulate diverse web environments, which is harder than selecting actions. DynaWeb uses 120B for the world model vs 8B for the policy.
- **Avoid:** Training the world model and policy jointly. Train the world model first on real trajectories, freeze it, then train the policy through interaction with the frozen world model.
- **Avoid:** Using the world model without fine-tuning on domain-specific web interaction data. A generic LLM used as a frozen world model achieves ~21% success rate vs ~31% with fine-tuning (a 48% relative drop).

## Error Handling

- **World model hallucination:** If the world model generates observations that reference nonexistent elements or impossible page states, implement a consistency checker that validates predicted AXTree structure (all referenced IDs exist, URLs are plausible). Discard hallucinated rollouts and resample.
- **Compounding errors in long rollouts:** Monitor rollout quality by periodically comparing dreamed 5-step trajectories against real trajectories from the same initial state. If divergence exceeds a threshold, reduce max rollout length or increase the real trajectory interleaving ratio.
- **Reward model disagreement:** When the self-assessment reward model is uncertain (e.g., partial task completion), default to reward = 0. False positives (rewarding failed trajectories) are more damaging than false negatives.
- **Policy collapse during GSPO:** If the policy starts generating repetitive or degenerate actions, increase the proportion of expert trajectories temporarily (to 70-80%) for a few epochs to recover, then gradually reduce back to 40-50%.

## Limitations

- **World model fidelity ceiling:** The world model cannot perfectly simulate all web dynamics -- JavaScript-heavy pages, CAPTCHAs, third-party authentication flows, and rapidly changing content (live feeds, stock tickers) will be poorly modeled. DynaWeb underperforms on sites requiring long-horizon planning with highly branching actions (e.g., GitHub, arXiv).
- **Two-model cost:** Requires maintaining both a large world model (120B+ parameters) and the policy model. The world model needs significant compute for fine-tuning and rollout generation.
- **Domain shift:** The world model is only as good as its training data. If the target website differs significantly from the training distribution (e.g., training on English e-commerce, deploying on Japanese government portals), dream quality degrades.
- **Static world model:** The current approach trains the world model once and freezes it. Real websites evolve over time, so the world model becomes stale and needs periodic retraining.
- **Accessibility tree dependency:** Assumes web pages can be reliably converted to structured AXTrees. Pages with heavy canvas rendering, iframes, or shadow DOM may produce poor AXTree representations.

## Reference

- **Paper:** [DynaWeb: Model-Based Reinforcement Learning of Web Agents](https://arxiv.org/abs/2601.22149) (Ding et al., 2026). Focus on Section 3 (DynaWeb Framework) for the world model design and Dyna training loop, Section 4 for GSPO derivation, and Table 2/Figure 3 for ablation results on dream length and expert trajectory mixing ratios.