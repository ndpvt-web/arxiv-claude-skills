---
name: "joint-continual-learning-local"
description: "Implement DA-GRPO (Dual-Advantage Group Relative Policy Optimization) for jointly training local small language models with adaptive cloud offloading under budget constraints. Use when: 'implement DA-GRPO training loop', 'build continual learning with cloud offloading', 'train local model with budget-constrained LLM assistance', 'add dual-advantage GRPO to my RL pipeline', 'implement adaptive cloud routing for SLM training', 'build edge-cloud collaborative fine-tuning'."
---

# Joint Continual Learning of Local Models with Budget-Constrained Cloud Offloading (DA-GRPO)

This skill enables Claude to implement the DA-GRPO algorithm from Chen et al. (2026), which jointly trains a local Small Language Model (SLM) on sequential tasks while learning *when* to offload queries to a cloud Large Language Model (LLM) -- all under a strict cloud-usage budget. The key innovation is embedding cloud-cost constraints directly into GRPO's advantage computation as a separate "cost advantage" signal, avoiding brittle reward shaping and enabling stable offloading behavior that adapts automatically across task switches without catastrophic forgetting.

## When to Use

- When building a training pipeline where a small local model must learn when to call a larger cloud API for help, subject to a cost budget
- When implementing continual/sequential fine-tuning of an SLM across multiple task distributions and needing to prevent catastrophic forgetting
- When the user needs a reinforcement learning post-training loop that jointly optimizes task accuracy and resource allocation
- When designing an edge-cloud inference system where the routing decision should be learned end-to-end rather than through a separate classifier
- When adapting GRPO (Group Relative Policy Optimization) to handle constrained optimization problems beyond simple reward maximization
- When the user wants to replace a fixed reward-penalty scheme for API costs with an adaptive dual-variable approach

## Key Technique

**Standard GRPO** generates a group of G responses per prompt, computes each response's reward, normalizes rewards within the group to get advantages, and updates the policy to favor above-average responses. DA-GRPO extends this by computing *two* group-relative advantages: a **task advantage** `A_i^r = r_i - mean(r)` measuring response quality, and a **cost advantage** `A_i^c = c_i - mean(c)` measuring whether a response used cloud assistance (`c_i in {0,1}`) relative to the group average. These combine into a dual-weighted advantage: `A_hat_i = A_i^r - lambda * A_i^c`, where `lambda >= 0` is an adaptive Lagrange multiplier.

The dual variable `lambda` is updated each batch via projected gradient ascent on the constraint violation: `lambda <- max(0, lambda + eta_lambda * (empirical_cloud_rate - tau))`, where `tau` is the target budget. This means lambda increases when the model over-uses the cloud (penalizing offloading more) and decreases when under-budget (relaxing the penalty). The constraint is satisfied at any stationary point where `lambda > 0`. This avoids the need to manually tune reward coefficients per task -- the dual variable self-adjusts across task switches.

**Why it helps continual learning**: When the local model can offload truly hard problems, it concentrates its limited capacity on learnable patterns rather than wasting gradient updates on unsolvable queries. This reshapes the optimization trajectory to reduce interference between tasks, substantially cutting forgetting rates compared to edge-only fine-tuning or fixed routing.

## Step-by-Step Workflow

1. **Define the model architecture and cloud interface.** Set up the local SLM (e.g., Qwen2.5-1.5B/3B) with a generation head that can emit a special `<CLOUD_REQUEST>` token. Wrap the cloud LLM (e.g., DeepSeek-R1) behind an API call that returns a completion given the prompt and partial local generation.

2. **Configure the constrained optimization parameters.** Set group size `G=8`, target cloud budget `tau` per task (e.g., 0.3 for math, 0.5 for code), dual learning rate `eta_lambda=1e-2`, initial dual variable `lambda_init=0.5`, and policy learning rate `eta_theta=2e-6`.

3. **Implement the group sampling procedure.** For each prompt in the batch, generate G responses from the current local policy. For any response that emits `<CLOUD_REQUEST>`, query the cloud LLM once per prompt (shared across all requesting responses in the group) and splice the cloud completion into those responses.

4. **Compute the binary cost indicator for each response.** Set `c_i = 1` if response i triggered a cloud request, `c_i = 0` otherwise. Compute the task reward `r_i` using your reward function (correctness, format compliance, etc.).

5. **Calculate group-relative dual advantages.** For each group of G responses: compute `A_i^r = r_i - mean(r_1..r_G)` and `A_i^c = c_i - mean(c_1..c_G)`. Form the combined advantage `A_hat_i = A_i^r - lambda * A_i^c`.

6. **Compute the policy gradient and update the local model.** Use the GRPO gradient estimator with the dual-weighted advantages: `grad = (G/(G-1)) * sum(grad_log_pi(y_i|x) * A_hat_i)`. Apply the gradient step to `theta` with your optimizer.

7. **Update the dual variable.** Estimate the empirical cloud usage rate `J_c_hat` on the current batch. Apply projected ascent: `lambda = max(0, lambda + eta_lambda * (J_c_hat - tau))`.

8. **Repeat across the current task's training steps.** Train for the designated number of steps (e.g., 840 for the first task group) monitoring both accuracy and cloud usage rate convergence.

9. **Switch to the next task distribution.** When transitioning to a new task, keep the model weights and dual variable, but update `tau` if the new task has a different budget. The dual variable will self-adjust. Train for the new task's steps (e.g., 400).

10. **Evaluate forgetting and budget compliance.** After each task switch, measure accuracy on all previous tasks and compute forgetting rate: `(Acc_before - Acc_after) / Acc_before`. Verify that the empirical cloud usage on each task stays within `tau +/- epsilon`.

## Concrete Examples

**Example 1: Implementing the core DA-GRPO training loop in PyTorch**

User: "I have a local 3B model and access to a cloud API. Implement the DA-GRPO training loop with a 30% cloud budget."

Approach:
1. Define the dual-advantage computation function
2. Implement group sampling with cloud fallback
3. Wire up the GRPO gradient with dual weighting
4. Add the dual variable update step

Output:
```python
import torch
import torch.nn.functional as F
from dataclasses import dataclass

@dataclass
class DAGRPOConfig:
    group_size: int = 8
    tau: float = 0.3           # target cloud usage rate
    eta_lambda: float = 1e-2   # dual variable learning rate
    lambda_init: float = 0.5
    cloud_request_token_id: int = 50256  # configure per tokenizer

class DAGRPOTrainer:
    def __init__(self, policy, cloud_client, tokenizer, config: DAGRPOConfig):
        self.policy = policy
        self.cloud = cloud_client
        self.tokenizer = tokenizer
        self.config = config
        self.lam = config.lambda_init  # dual variable (scalar, not a tensor)

    def generate_group(self, prompt_ids, max_len=512):
        """Generate G responses; query cloud for any that request help."""
        G = self.config.group_size
        responses, cloud_flags = [], []

        # Sample G responses from the local policy
        for _ in range(G):
            output = self.policy.generate(prompt_ids, max_length=max_len,
                                          do_sample=True, temperature=0.7)
            responses.append(output)
            cloud_flags.append(
                self.config.cloud_request_token_id in output[0].tolist()
            )

        # Query cloud once per prompt if any response requested it
        cloud_completion = None
        if any(cloud_flags):
            prompt_text = self.tokenizer.decode(prompt_ids[0])
            cloud_completion = self.cloud.complete(prompt_text)

        # Splice cloud completion into requesting responses
        final_responses = []
        for resp, used_cloud in zip(responses, cloud_flags):
            if used_cloud and cloud_completion is not None:
                final_responses.append(
                    self._splice_cloud(resp, cloud_completion)
                )
            else:
                final_responses.append(resp)

        cost_indicators = [1.0 if f else 0.0 for f in cloud_flags]
        return final_responses, cost_indicators

    def compute_dual_advantages(self, rewards, costs):
        """Compute A_hat_i = A_i^r - lambda * A_i^c for each response."""
        rewards_t = torch.tensor(rewards, dtype=torch.float32)
        costs_t = torch.tensor(costs, dtype=torch.float32)

        task_adv = rewards_t - rewards_t.mean()
        cost_adv = costs_t - costs_t.mean()
        dual_adv = task_adv - self.lam * cost_adv
        return dual_adv

    def update_dual_variable(self, empirical_cloud_rate):
        """Projected gradient ascent on the Lagrange multiplier."""
        self.lam = max(0.0, self.lam + self.config.eta_lambda * (
            empirical_cloud_rate - self.config.tau
        ))

    def train_step(self, prompt_batch, reward_fn):
        """One DA-GRPO training step over a batch of prompts."""
        total_cloud_usage = 0.0
        total_prompts = 0
        policy_loss = 0.0

        for prompt_ids in prompt_batch:
            responses, costs = self.generate_group(prompt_ids)
            rewards = [reward_fn(prompt_ids, r) for r in responses]
            dual_adv = self.compute_dual_advantages(rewards, costs)

            # GRPO policy gradient with dual advantages
            G = self.config.group_size
            scale = G / (G - 1)
            for i, resp in enumerate(responses):
                log_prob = self.policy.log_prob(prompt_ids, resp)
                policy_loss -= scale * log_prob * dual_adv[i]

            total_cloud_usage += sum(costs)
            total_prompts += G

        # Backward pass and optimizer step (handled externally)
        policy_loss /= len(prompt_batch)

        # Update dual variable based on empirical cloud rate
        empirical_rate = total_cloud_usage / max(total_prompts, 1)
        self.update_dual_variable(empirical_rate)

        return policy_loss, empirical_rate, self.lam
```

**Example 2: Configuring continual learning across two task phases**

User: "I need to train on math tasks first, then switch to code tasks, without forgetting math. Cloud budget is 30% for math, 50% for code."

Approach:
1. Set up phase-specific configurations
2. Train Phase 1 with tau=0.3
3. Carry forward model weights and lambda, switch tau to 0.5
4. Evaluate forgetting on Phase 1 tasks after Phase 2

Output:
```python
# Phase 1: Math reasoning
phase1_config = DAGRPOConfig(tau=0.3, lambda_init=0.5)
trainer = DAGRPOTrainer(policy, cloud_client, tokenizer, phase1_config)

for step in range(840):
    loss, cloud_rate, lam = trainer.train_step(
        math_dataloader.next_batch(), math_reward_fn
    )
    if step % 100 == 0:
        print(f"Step {step}: loss={loss:.4f}, cloud={cloud_rate:.2%}, lam={lam:.3f}")

# Evaluate Phase 1 accuracy
math_acc_before = evaluate(policy, math_eval_set)

# Phase 2: Code generation -- carry forward lambda, update tau
trainer.config.tau = 0.5
for step in range(400):
    loss, cloud_rate, lam = trainer.train_step(
        code_dataloader.next_batch(), code_reward_fn
    )

# Evaluate forgetting
math_acc_after = evaluate(policy, math_eval_set)
forgetting_rate = (math_acc_before - math_acc_after) / math_acc_before
print(f"Math forgetting rate: {forgetting_rate:.2%}")
code_acc = evaluate(policy, code_eval_set)
print(f"Code accuracy: {code_acc:.2%}")
```

**Example 3: Adding the cloud-request token to an existing model**

User: "How do I modify my tokenizer and model to support the cloud offloading signal?"

Approach:
1. Add a special `<CLOUD_REQUEST>` token to the tokenizer
2. Resize model embeddings
3. Train the model to emit this token when uncertain

Output:
```python
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-3B-Instruct")
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-3B-Instruct")

# Add the cloud request token
special_tokens = {"additional_special_tokens": ["<CLOUD_REQUEST>"]}
tokenizer.add_special_tokens(special_tokens)
model.resize_token_embeddings(len(tokenizer))

cloud_token_id = tokenizer.convert_tokens_to_ids("<CLOUD_REQUEST>")

# During reward computation, differentiate cloud-assisted responses:
def compute_reward(prompt, response, ground_truth, cloud_used):
    correct = check_answer(response, ground_truth)
    format_ok = check_format(response)  # <think>...</think><answer>...</answer>
    if correct and format_ok and not cloud_used:
        return 1.0    # full credit for local solve
    elif correct and format_ok and cloud_used:
        return 0.8    # partial credit when cloud assisted
    elif format_ok:
        return 0.1    # format-only reward
    else:
        return 0.0
```

## Best Practices

- **Do:** Keep the dual variable `lambda` as a simple scalar updated outside the computation graph. It does not need gradients -- projected ascent on a scalar is sufficient and stable.
- **Do:** Share a single cloud query across all G responses in a group that request help. This keeps cloud costs proportional to prompts, not to group size.
- **Do:** Monitor `lambda` over training. It should rise early (when the model is weak and over-requests) then fall as competence improves. If it stays pinned at 0, your budget `tau` is too generous.
- **Do:** Use differentiated rewards (correct-local > correct-cloud > format-only > wrong) so the task advantage naturally favors self-reliance when the model can solve the problem.
- **Avoid:** Manually tuning a fixed penalty coefficient for cloud usage. The entire point of the dual-variable approach is to let this adapt automatically. Fixed penalties break across task switches.
- **Avoid:** Generating responses with greedy decoding during training. The group-relative mechanism requires diversity within each group to produce meaningful advantages. Use `temperature >= 0.7`.

## Error Handling

- **Lambda diverges upward**: If `lambda` grows without bound, the budget `tau` is likely too tight for the model's current capability. Relax `tau` or verify the cloud interface is actually returning useful completions.
- **Cloud usage oscillates wildly**: Reduce `eta_lambda` (e.g., from 1e-2 to 1e-3). The dual update may be overshooting. Alternatively, increase batch size to reduce variance in the empirical cloud rate estimate.
- **Catastrophic forgetting persists despite offloading**: Ensure the cloud query is actually being used for hard problems (not random). Verify that the `<CLOUD_REQUEST>` token is being emitted by the policy for genuinely difficult prompts, not just as noise.
- **All responses in a group request cloud**: This collapses the cost advantage to zero (all c_i = 1, mean = 1, A_i^c = 0). Increase `lambda_init` or raise `tau` constraints to incentivize at least some local attempts during early training.
- **Reward hacking via format-only responses**: If the model learns to emit correct format tags without real reasoning, add a minimum-length or reasoning-step check to your reward function.

## Limitations

- Requires access to a cloud LLM API during training, not just inference. Training costs include cloud query costs for every batch, scaled by the fraction of prompts that trigger offloading.
- The method assumes tasks arrive sequentially with clear boundaries. It does not handle interleaved or non-stationary task streams where you cannot define clean phase transitions.
- Group size G must be large enough (8+) for group-relative normalization to be stable. This multiplies per-prompt generation cost by G.
- The `<CLOUD_REQUEST>` token approach assumes the model can learn to identify its own uncertainty. Very small models (< 1B parameters) may struggle to develop this meta-cognitive capability.
- Budget compliance is achieved in expectation at convergence, not per-batch. Short training runs or high variance in task difficulty can lead to temporary budget violations.

## Reference

Chen, E., Fang, W., Wang, S., & Brinton, C. (2026). *Joint Continual Learning of Local Language Models and Cloud Offloading Decisions with Budget Constraints.* arXiv:2602.00166v2. [https://arxiv.org/abs/2602.00166v2](https://arxiv.org/abs/2602.00166v2)

Key sections to study: Algorithm 1 (full DA-GRPO loop), Proposition 1 (unbiased gradient estimator proof), Section 4.2 (dual variable dynamics and fixed-point analysis), and Table 2 (forgetting rates across model sizes).