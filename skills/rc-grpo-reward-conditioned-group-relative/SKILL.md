---
name: "rc-grpo-reward-conditioned-group-relative"
description: "Implement reward-conditioned training pipelines for multi-turn tool-calling agents using RC-GRPO. Injects discrete reward tokens into prompts to steer trajectory quality, then uses within-group reward-token diversity to prevent advantage collapse during RL. Triggers: 'train a tool-calling agent', 'reward-conditioned policy optimization', 'fix vanishing gradients in GRPO', 'multi-turn agent RL training', 'improve tool-calling with reinforcement learning', 'reward token conditioning for LLM agents'"
---

# RC-GRPO: Reward-Conditioned Group Relative Policy Optimization

This skill enables Claude to design and implement training pipelines for multi-turn tool-calling LLM agents using the RC-GRPO method. The core idea: instead of treating reward sparsity as an unsolvable exploration problem, you condition the model on discrete reward tokens (`<|high_reward|>`, `<|low_reward|>`) so it learns to generate trajectories of controllable quality. During RL, sampling diverse reward tokens within each GRPO group guarantees within-group variance, preventing the advantage collapse that kills standard GRPO training on binary-reward tasks.

## When to Use

- When the user wants to train or fine-tune an LLM for multi-turn tool calling (function calling, API orchestration, agent loops)
- When a user reports that GRPO or PPO training has stalled with vanishing gradients on sparse-reward tasks
- When building a training pipeline where most rollouts receive the same binary reward (all 0 or all 1), making group-normalized advantages uninformative
- When the user needs to improve within-group diversity during on-policy RL without increasing group size or compute
- When implementing a two-stage SFT-then-RL pipeline for tool-calling agents and wants to maximize the RL stage effectiveness
- When the user asks how to condition an LLM on reward signals at inference time to steer output quality

## Key Technique

**The Vanishing Advantage Problem.** Standard GRPO normalizes advantages within a group of rollouts. When the policy is already strong from SFT, most rollouts in a group converge to the same reward (e.g., all succeed or all fail). The normalized advantage becomes near-zero, gradients vanish, and RL training stalls. This is formally guaranteed: after strong SFT with KL distance epsilon to optimal, all G trajectories collapse to identical rewards with probability at least `1 - G*T*epsilon`.

**Reward-Conditioned Trajectory Policy (RCTP).** The fix starts in Stage 1. Collect mixed-quality trajectories (successes and failures), label each with a reward token (`<|high_reward|>` or `<|low_reward|>`), and prepend this token to the prompt before each assistant turn. Fine-tune the model on this augmented dataset. The model learns a conditional distribution: given `<|high_reward|>`, produce expert-like tool calls; given `<|low_reward|>`, produce plausible but suboptimal trajectories. This is not reward hacking -- it is teaching the model to understand what distinguishes good from bad trajectories.

**RC-GRPO Sampling.** In Stage 2, for each GRPO group of G rollouts, sample reward tokens stochastically (e.g., p=0.5 for `<|high_reward|>`, 1-p for `<|low_reward|>`). Condition each rollout on its sampled token. High-reward-conditioned rollouts tend to succeed; low-reward-conditioned ones tend to fail. This injects controlled variance into the group, guaranteeing `E[sigma_g^2] >= kappa * epsilon^2` where kappa depends on group size and sampling ratio. The advantage signal stays informative, and RL training proceeds effectively. On BFCLv4, this took Qwen-2.5-7B from 48.75% (SFT+GRPO) to 85.00%, surpassing all closed-source models including Opus-4.5 at 61.25%.

## Step-by-Step Workflow

1. **Collect mixed-quality trajectory data.** Run your base model on multi-turn tool-calling tasks, collecting both successful and failed trajectories. Aim for a roughly balanced dataset (1:1 success-to-failure ratio). Each trajectory is a sequence of (observation, action) pairs across turns.

2. **Define the reward function.** Implement two complementary signals: (a) **State consistency** -- compare the final environment state against a golden replay, and (b) **Action coverage** -- verify all required tool calls appear with correct parameters. Combine into a binary reward: R=1 if both pass, R=0 otherwise.

3. **Assign reward tokens to trajectories.** For each trajectory in your dataset, prepend the appropriate token based on its reward:
   ```
   R = 1  -->  <|high_reward|>
   R = 0  -->  <|low_reward|>
   ```
   Add these as special tokens to your tokenizer. The token is injected before each assistant turn in the multi-turn conversation.

4. **Format the RCTP training data.** Structure each training example so the reward token appears in the prompt context before the model's response at every turn:
   ```
   [System prompt with tool definitions]
   User: Book a flight from NYC to London for next Friday.
   <|high_reward|>
   Assistant: {"name": "search_flights", "arguments": {"origin": "NYC", "destination": "LHR", "date": "2026-02-20"}}
   Tool: {"flights": [{"id": "BA117", "price": 450, ...}]}
   <|high_reward|>
   Assistant: {"name": "book_flight", "arguments": {"flight_id": "BA117"}}
   ```

5. **Train the RCTP model (Stage 1).** Fine-tune the base model on the reward-token-augmented dataset using standard cross-entropy loss. Recommended hyperparameters: learning rate 5e-5, batch size 32, 3 epochs. The loss is `L_RCTP = -E[sum_t log pi(a_t | h_t, r)]` where r is the reward token and h_t is the history.

6. **Configure the RC-GRPO training loop (Stage 2).** Initialize from the RCTP checkpoint. Set group size G=4, clipping epsilon=0.2, KL coefficient beta=0.05, learning rate 1e-5. Define the reward-token sampling distribution: `P(<|high_reward|>) = p`, `P(<|low_reward|>) = 1-p`, where p matches the expert proportion in training data (typically 0.5).

7. **Generate diverse rollouts per group.** For each training prompt, generate G rollouts. For each rollout j, sample a reward token `r_j ~ P_sample(r)` and condition generation on it. This produces a mix of high-quality and low-quality trajectories within the same group, ensuring nonzero advantage variance.

8. **Compute clipped advantages and update.** Score each rollout with your reward function, compute group-normalized advantages `A_j = (R_j - mu_g) / (sigma_g + epsilon_stab)`, then apply the clipped PPO-style objective:
   ```
   L = -E[ (1/G) * sum_j min(rho_j * A_j, clip(rho_j, 1-eps, 1+eps) * A_j) - beta * KL(pi || pi_ref) ]
   ```

9. **Monitor entropy and reward correlation.** Track the correlation between policy entropy and reward across training. A healthy RC-GRPO run shows *negative* correlation (entropy decreases as reward increases, indicating the model is learning to be precise). Positive correlation signals the RL is relying on entropy alone -- a red flag.

10. **Remove reward tokens at inference.** At deployment, always condition on `<|high_reward|>` (or omit the token entirely if the model defaults to high-quality behavior). The reward token is a training scaffold, not a deployment feature.

## Concrete Examples

**Example 1: Training Pipeline for a Function-Calling Agent**

User: "I have a Qwen-2.5-7B model and want to train it for multi-turn function calling. Standard GRPO isn't improving after SFT. Help me set up RC-GRPO."

Approach:
1. Diagnose the problem: check if within-group reward variance is near zero (likely if SFT accuracy is already >60%)
2. Collect trajectory data by running the SFT checkpoint on training tasks, keeping both passes and failures
3. Create the RCTP dataset with reward tokens injected
4. Implement the two-stage training script

Output (RCTP data preparation code):
```python
import json
from transformers import AutoTokenizer

# Add special reward tokens
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-7B-Instruct")
tokenizer.add_special_tokens({
    "additional_special_tokens": ["<|high_reward|>", "<|low_reward|>"]
})

def prepare_rctp_example(trajectory: dict, reward: int) -> list[dict]:
    """Convert a multi-turn trajectory into RCTP training format."""
    reward_token = "<|high_reward|>" if reward == 1 else "<|low_reward|>"
    messages = []
    for turn in trajectory["turns"]:
        if turn["role"] == "user":
            messages.append({"role": "user", "content": turn["content"]})
        elif turn["role"] == "assistant":
            # Prepend reward token to assistant content
            messages.append({
                "role": "assistant",
                "content": f"{reward_token}\n{turn['content']}"
            })
        elif turn["role"] == "tool":
            messages.append({"role": "tool", "content": turn["content"]})
    return messages

# Build balanced dataset
rctp_dataset = []
for traj in all_trajectories:
    reward = evaluate_trajectory(traj)  # Returns 0 or 1
    rctp_dataset.append(prepare_rctp_example(traj, reward))
```

**Example 2: Implementing the RC-GRPO Rollout Sampler**

User: "How do I modify my GRPO training loop to sample diverse reward tokens per group?"

Approach:
1. Wrap the rollout generation to sample a reward token per trajectory
2. Condition each generation on the sampled token
3. Compute advantages with the resulting diverse rewards

Output (rollout generation logic):
```python
import random
import torch

def generate_rc_grpo_group(
    model, tokenizer, prompt: str, group_size: int = 4,
    p_high: float = 0.5, max_turns: int = 5
) -> list[dict]:
    """Generate a group of reward-conditioned rollouts."""
    rollouts = []
    for j in range(group_size):
        # Sample reward token for this rollout
        reward_token = (
            "<|high_reward|>" if random.random() < p_high
            else "<|low_reward|>"
        )
        # Generate multi-turn trajectory conditioned on reward token
        trajectory = generate_trajectory(
            model, tokenizer, prompt,
            reward_prefix=reward_token,
            max_turns=max_turns
        )
        trajectory["reward_token"] = reward_token
        rollouts.append(trajectory)
    return rollouts

def compute_group_advantages(rollouts: list[dict]) -> list[float]:
    """Compute group-normalized advantages with stability epsilon."""
    rewards = [evaluate_trajectory(r) for r in rollouts]
    mu = sum(rewards) / len(rewards)
    var = sum((r - mu) ** 2 for r in rewards) / len(rewards)
    sigma = var ** 0.5
    eps_stab = 1e-6
    return [(r - mu) / (sigma + eps_stab) for r in rewards]
```

**Example 3: Diagnosing Advantage Collapse in Existing GRPO**

User: "My GRPO training loss is flat and the model isn't improving. What's wrong?"

Approach:
1. Log within-group reward variance across training steps
2. Check if most groups have all-same rewards (variance = 0)
3. Recommend RC-GRPO if variance collapse is confirmed

Output (diagnostic script):
```python
def diagnose_grpo_health(training_log: list[dict]) -> dict:
    """Analyze GRPO training for advantage collapse."""
    zero_variance_groups = 0
    total_groups = len(training_log)
    avg_variance = 0.0

    for group in training_log:
        rewards = group["rewards"]  # List of rewards per rollout
        mu = sum(rewards) / len(rewards)
        var = sum((r - mu) ** 2 for r in rewards) / len(rewards)
        avg_variance += var
        if var < 1e-8:
            zero_variance_groups += 1

    avg_variance /= total_groups
    collapse_ratio = zero_variance_groups / total_groups

    diagnosis = {
        "avg_within_group_variance": avg_variance,
        "zero_variance_ratio": collapse_ratio,
        "recommendation": (
            "ADVANTAGE COLLAPSE DETECTED. "
            f"{collapse_ratio:.0%} of groups have zero variance. "
            "Switch to RC-GRPO: train RCTP first, then use reward-token "
            "diversity to guarantee within-group variance."
            if collapse_ratio > 0.5
            else "Variance looks healthy. Check learning rate or KL penalty."
        )
    }
    return diagnosis
```

## Best Practices

- **Do:** Maintain a balanced success/failure ratio (close to 1:1) in RCTP training data. Imbalanced data causes the model to ignore one reward token.
- **Do:** Set the reward-token sampling probability `p` to match the expert trajectory proportion in your training set (typically 0.5 for balanced data).
- **Do:** Monitor entropy-reward correlation during training. Negative correlation (entropy drops as reward rises) is the healthy signal for RC-GRPO.
- **Do:** Use state consistency AND action coverage as complementary reward signals -- action-only rewards miss side effects, state-only rewards miss correct intermediate steps.
- **Avoid:** Skipping the RCTP stage and going directly to RC-GRPO from a standard SFT checkpoint. The ablations show this yields minimal gains because the model hasn't learned to differentiate trajectory quality by reward token.
- **Avoid:** Using group sizes smaller than 4. The variance guarantee `kappa = (G-1)/G * p*(1-p)` degrades rapidly below G=4, and the diversity benefit diminishes.

## Error Handling

- **Reward token not learned (model ignores conditioning):** Verify the special tokens were properly added to the tokenizer and the model's embedding layer was resized. Check that reward tokens appear in the correct position (before assistant turns, not buried in system prompts).
- **Advantage still near-zero after RC-GRPO:** Inspect whether the RCTP model actually produces different quality trajectories for different tokens. Generate 50 rollouts with `<|high_reward|>` and 50 with `<|low_reward|>` -- if their success rates are similar, the RCTP training failed and needs more data or epochs.
- **KL divergence exploding:** Reduce learning rate or increase beta. RC-GRPO's diverse sampling can push the policy further from the reference than standard GRPO, so KL regularization is more important.
- **Reward hacking on tool calls:** If the model learns to game the reward function (e.g., calling tools in the right order but with wrong parameters), strengthen the action coverage component of the reward by checking parameter values, not just function names.

## Limitations

- **Binary rewards only in the published method.** The paper uses R in {0, 1}. Extending to continuous or multi-level rewards (e.g., `<|medium_reward|>`) is plausible but unvalidated -- it would require careful bucketing and more training data per bucket.
- **Requires mixed-quality trajectory data.** If your base model is too weak to produce any successes, or too strong to produce any failures, you cannot construct balanced RCTP training data without external trajectory sources.
- **Compute overhead.** The two-stage pipeline (RCTP-FT then RC-GRPO) requires roughly 2x the training compute of single-stage SFT+GRPO. The payoff is much higher final accuracy, but the cost is real.
- **Task-specific reward engineering.** The state-consistency and action-coverage rewards must be defined per task domain. There is no universal reward function -- you need ground-truth environment states or golden action sequences.
- **Tested primarily on function-calling benchmarks.** The BFCLv4 results are strong, but generalization to other multi-turn agent settings (web browsing, code execution, embodied agents) is not yet demonstrated.

## Reference

[RC-GRPO: Reward-Conditioned Group Relative Policy Optimization for Multi-Turn Tool Calling Agents](https://arxiv.org/abs/2602.03025v1) -- Zhong et al., 2026. Key sections: Section 4 for the variance collapse analysis (Propositions 4.2-4.3), Section 5 for the RCTP-FT and RC-GRPO algorithms, Table 1 for BFCLv4 results showing 85% accuracy on Qwen-2.5-7B surpassing closed-source models.