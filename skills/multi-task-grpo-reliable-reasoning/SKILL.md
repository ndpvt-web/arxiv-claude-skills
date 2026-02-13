---
name: "multi-task-grpo-reliable-reasoning"
description: |
  Implement Multi-Task GRPO (MT-GRPO) training pipelines that balance LLM reasoning performance
  across diverse tasks using dynamic task weighting and ratio-preserving sampling. Use this skill when:
  - "Set up multi-task GRPO training for my reasoning model"
  - "My RL fine-tuning is dominated by one task, how do I balance it?"
  - "Implement worst-case task optimization for multi-task LLM training"
  - "Add dynamic task weighting to my GRPO training loop"
  - "Build a ratio-preserving sampler for multi-task RL post-training"
  - "Balance performance across math, code, and logic tasks during training"
---

# Multi-Task GRPO: Reliable LLM Reasoning Across Tasks

This skill enables Claude to design and implement Multi-Task GRPO (MT-GRPO) training pipelines that produce LLMs with balanced, reliable reasoning across multiple task families. Rather than optimizing average accuracy (which lets easy tasks dominate while hard tasks stagnate), MT-GRPO dynamically reweights tasks to maximize worst-task performance and uses a ratio-preserving sampler to compensate for zero-gradient prompt filtering. The result is 16-28% better worst-task accuracy than standard GRPO with 50% fewer training steps to convergence.

## When To Use

- When a user is training an LLM with GRPO/DAPO on multiple reasoning tasks (math, code, logic puzzles) and one task's accuracy plateaus while others improve
- When building a multi-task RL post-training pipeline and needing to ensure no single task dominates gradient updates
- When configuring training data sampling ratios across heterogeneous task difficulties
- When a user reports that their multi-task fine-tuned model performs unevenly across benchmarks
- When implementing custom RL training loops in frameworks like TRL, OpenRLHF, or veRL that need task-balanced optimization
- When designing reward functions and sampling strategies for GRPO-based training on 3+ task families

## Key Technique

**The core problem:** Standard multi-task GRPO samples tasks uniformly and averages their losses. This fails because (1) tasks have different reward scales and gradient magnitudes, causing easy tasks to dominate optimization, and (2) tasks produce zero-advantage prompts at different rates -- when all sampled responses for a prompt get identical rewards, the advantage normalizes to zero and that prompt contributes no gradient. Tasks with high zero-advantage rates get silently underrepresented even with equal sampling weights.

**MT-GRPO's two-part solution:** First, an Improvement-Aware Weight Update (IWU) mechanism treats multi-task training as a minimax game -- maximize a weighted combination of task rewards while the adversary (the weight updater) concentrates weight on the worst-performing tasks. The signal combines per-step task improvement `I_k` with absolute task performance `J_k`, controlled by a parameter `lambda` that trades off robustness vs. average accuracy. Task logits are updated via softmax gradient descent each training step. Second, a Ratio-Preserving (RP) sampler inflates pre-filtering sample counts for tasks with high zero-advantage rates, ensuring the post-filtered batch composition matches the intended task weights. It estimates each task's filtering rate `rho_k` online and adjusts sampling accordingly.

**Why it works:** The IWU acts as a self-correcting feedback loop -- tasks falling behind receive more weight, which increases their gradient contribution, which improves their performance, which reduces their weight back toward uniform. The RP sampler removes the hidden bias from zero-advantage filtering, so the actual optimization matches the intended task distribution. Together, they achieve 16-28% absolute improvement in worst-task accuracy over standard GRPO and 6% over DAPO, while maintaining competitive average accuracy.

## Step-by-Step Workflow

1. **Define task families and reward functions.** Enumerate each distinct reasoning task (e.g., math word problems, code generation, logical puzzles). Assign a binary or scalar reward function `R_k(x, y)` to each task `k`. Verify rewards are comparable in scale; if not, normalize them to [0, 1].

2. **Initialize task weight logits and filtering rate estimates.** Create a logit vector `xi` of length K (number of tasks) initialized to zeros (giving uniform softmax weights `z = 1/K`). Initialize filtering rate estimates `rho_k = 0` for each task. Set hyperparameters: learning rate `beta` for weight updates (try 0.1), robustness parameter `lambda` (try 0.5-1.2; higher = more worst-task focus), and max inflation cap `M_acc` (try 5-10).

3. **Implement the Ratio-Preserving Sampler.** For each training step: (a) compute target post-filter counts `(n_1,...,n_K) ~ Multinomial(B, z)` where B is batch size, (b) inflate sampling weights by `z_hat_k = z_k / (1 - rho_k)` normalized, capped by `M_acc`, (c) sample prompts from each task according to inflated weights, (d) generate N rollouts per prompt and compute advantages, (e) discard zero-advantage prompts, (f) if any task has fewer non-zero samples than its target `n_k`, resample from that task until met or budget exhausted, (g) update `rho_k` estimates with exponential moving average.

4. **Compute per-task GRPO loss.** For the filtered batch, compute the standard GRPO objective per task: for each prompt, normalize advantages within the group of N rollouts using mean/std, compute importance-weighted policy gradient with KL penalty against the reference model. Track per-task reward `J_k(theta)` as the mean reward across that task's prompts.

5. **Perform the optimizer step.** Combine per-task losses using current weights `z_k` and backpropagate. Update model parameters with your optimizer (Adam, etc.).

6. **Measure per-task improvement.** After the optimizer step, compute `I_k = J_k(theta_new) - J_k(theta_old)` for each task. This can use the training batch rewards before and after the update, or a held-out validation set for more stable estimates.

7. **Run the Improvement-Aware Weight Update (IWU).** Compute the combined signal `s_k = I_k + lambda * J_k(theta)` for each task. Update logits via: `g_k = z_k * (s_k - sum_j(z_j * s_j))`, then `xi_k = xi_k - beta * g_k`. Recompute `z = softmax(xi)`. Tasks with low improvement or low absolute reward gain higher weight.

8. **Log and monitor per-task metrics.** Track per-task accuracy, reward, weight `z_k`, filtering rate `rho_k`, and improvement `I_k` at each step. Plot worst-task accuracy as the primary metric. Alert if any task weight exceeds 0.5 (indicating severe imbalance) or if filtering rates exceed 0.8 (indicating the task's prompts are too easy/hard for current model capability).

9. **Tune lambda on a validation set.** Run short training runs with `lambda` in {0.0, 0.3, 0.6, 1.0, 1.2}. At `lambda=0`, MT-GRPO emphasizes balanced improvement only. At higher `lambda`, it explicitly upweights tasks with lower absolute reward. Select the value that gives acceptable worst-task accuracy without sacrificing too much average accuracy.

10. **Evaluate and iterate.** Evaluate the final model on held-out test sets for each task. Compare worst-task accuracy, average accuracy, and per-task accuracy against a uniform-sampling GRPO baseline. If one task still lags, check whether its reward function is too sparse or its prompts need difficulty adjustment.

## Concrete Examples

**Example 1: Balancing math, code, and logic training**

```
User: I'm training Qwen-2.5-3B with GRPO on three tasks -- Countdown (number
puzzles), Zebra logic puzzles, and ARC reasoning. Countdown trains fine but
Zebra accuracy is stuck at 12% after 500 steps. How do I fix this?

Approach:
1. Diagnose: Check Zebra's zero-advantage filtering rate. If most Zebra prompts
   produce all-wrong or all-right rollouts, gradients vanish for that task.
2. Implement RP sampler: Track rho_zebra online. If rho_zebra = 0.6, inflate
   Zebra's sampling weight by 1/(1-0.6) = 2.5x to compensate.
3. Add IWU: Since J_zebra(theta) is low (0.12) vs J_countdown(theta) ~ 0.7,
   the signal s_zebra = I_zebra + lambda * 0.12 will be low, causing the weight
   update to increase z_zebra.
4. Set lambda=1.0, beta=0.1, M_acc=8. Train for 500 more steps.

Implementation (pseudocode for the training loop):

```python
# Core MT-GRPO training loop
import torch
import torch.nn.functional as F

class MTGRPOTrainer:
    def __init__(self, tasks, batch_size=64, num_rollouts=8,
                 beta=0.1, lam=1.0, m_acc=8.0, rho_ema=0.95):
        self.K = len(tasks)
        self.tasks = tasks
        self.B = batch_size
        self.N = num_rollouts
        self.beta = beta
        self.lam = lam
        self.m_acc = m_acc
        self.rho_ema = rho_ema

        self.xi = torch.zeros(self.K)          # task weight logits
        self.rho = torch.zeros(self.K)          # filtering rate estimates
        self.z = F.softmax(self.xi, dim=0)      # task weights

    def rp_sample(self):
        """Ratio-preserving sampler."""
        # Inflate weights to compensate for filtering
        inflate = torch.clamp(1.0 / (1.0 - self.rho), max=self.m_acc)
        z_hat = self.z * inflate
        z_hat = z_hat / z_hat.sum()

        # Sample target counts
        targets = torch.multinomial(self.z, self.B, replacement=True)
        target_counts = torch.bincount(targets, minlength=self.K)

        # Sample with inflated weights, filter, resample if needed
        batch = self._sample_and_filter(z_hat, target_counts)
        return batch

    def iwu_step(self, improvements, rewards):
        """Improvement-Aware Weight Update."""
        s = improvements + self.lam * rewards     # combined signal
        baseline = (self.z * s).sum()
        grad = self.z * (s - baseline)
        self.xi = self.xi - self.beta * grad      # minimize worst-task
        self.z = F.softmax(self.xi, dim=0)

    def train_step(self, model, optimizer):
        batch = self.rp_sample()
        rewards_before = self.evaluate_tasks(model)

        loss = self.compute_weighted_grpo_loss(model, batch, self.z)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        rewards_after = self.evaluate_tasks(model)
        improvements = rewards_after - rewards_before
        self.iwu_step(improvements, rewards_after)

        return {
            'weights': self.z.tolist(),
            'worst_task': rewards_after.min().item(),
            'avg_task': rewards_after.mean().item(),
        }
```

Output: After 500 steps, expect Zebra accuracy to rise from 12% to ~40%+,
while Countdown and ARC maintain their levels. The z_zebra weight will
initially spike then stabilize as Zebra catches up.
```

**Example 2: Configuring MT-GRPO in an existing TRL/OpenRLHF pipeline**

```
User: I have a working GRPO training script using TRL's GRPOTrainer on a
single math task. I want to extend it to 5 tasks. What do I change?

Approach:
1. Replace the single dataset with a multi-task dataset that tags each
   prompt with its task ID.
2. Replace TRL's uniform sampler with the RP sampler:
   - Wrap the dataloader to sample according to z_hat weights
   - After rollout generation, filter zero-advantage prompts
   - Track per-task filtering rates with EMA
3. After each optimizer step, compute per-task reward deltas and run IWU.
4. Modify the loss computation to weight per-task losses by z_k.

Key config changes:
  - Add: task_weights_beta = 0.1
  - Add: task_weights_lambda = 0.8  (start moderate, tune up if needed)
  - Add: rp_sampler_m_acc = 8.0
  - Add: filtering_rate_ema = 0.95
  - Keep: per_prompt_rollouts = 8 (N, standard for GRPO)
  - Keep: kl_penalty_coeff = 0.01 (tau)

Output: A multi-task config dict and the three new components (RP sampler,
IWU updater, per-task metrics logger) to integrate into the existing loop.
```

**Example 3: Diagnosing task imbalance in an ongoing training run**

```
User: My 9-task GRPO training shows 85% average accuracy but worst task is
only 31%. The tasks are easy/medium/hard variants of math, code, and logic.

Approach:
1. Check per-task filtering rates. Hard variants likely have rho > 0.7
   (most prompts produce all-wrong rollouts, giving zero advantages).
2. Check if current sampling is uniform -- if so, hard tasks contribute
   almost no gradient despite 1/9 of the batch.
3. Implement MT-GRPO with lambda=1.2 (aggressive worst-task focus since
   the gap is 54 percentage points).
4. The RP sampler will inflate hard-task sampling by up to M_acc to
   compensate for high filtering rates.
5. IWU will quickly concentrate weight on the 31% task until it improves.

Expected outcome: Worst-task accuracy rises to ~55-65% within the same
training budget. Average may dip slightly to ~80% -- this is the expected
robustness/average tradeoff controlled by lambda.
```

## Best Practices

- **Do:** Start with `lambda=0.5` and increase if worst-task performance is unacceptable. The parameter directly controls the robustness-average tradeoff; sweeping {0.3, 0.6, 1.0, 1.2} covers the useful range.
- **Do:** Monitor per-task filtering rates `rho_k` throughout training. A rate above 0.8 means that task's prompts are either too easy (all correct) or too hard (all wrong) for the current model, and you may need to adjust prompt difficulty rather than just resampling.
- **Do:** Use at least N=8 rollouts per prompt. Fewer rollouts increase variance in advantage estimation and inflate zero-advantage rates, undermining the entire approach.
- **Do:** Normalize reward functions to comparable scales across tasks before training. MT-GRPO adapts weights based on reward magnitudes, so a task with rewards in [0, 100] will dominate one with rewards in [0, 1].
- **Avoid:** Setting `M_acc` too high (above 15). Extreme inflation in the RP sampler can cause a single hard task to consume most of the sampling budget, starving other tasks.
- **Avoid:** Using MT-GRPO with fewer than 3 tasks. With 2 tasks, simple loss weighting or alternating batches is sufficient and easier to tune. MT-GRPO's minimax formulation shines at 3+ tasks where manual weight tuning becomes impractical.
- **Avoid:** Evaluating MT-GRPO by average accuracy alone. The entire point is worst-task performance. Always report both metrics, and use worst-task accuracy as the primary selection criterion for hyperparameter tuning.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| One task weight collapses to ~0 | A task's `z_k` drops below 0.01 | Lower `lambda`; the task may be too easy and getting deprioritized correctly, or its reward signal is noisy |
| Filtering rate hits 1.0 for a task | Zero non-trivial gradients from that task | The task is either solved or impossible at current model capacity. Remove it or adjust difficulty |
| Training instability after IWU | Loss spikes when weights shift | Reduce `beta` (weight update learning rate) to 0.01-0.05 for smoother adaptation |
| RP sampler budget exhausted | Resampling loop hits max iterations without meeting targets | Increase `M_acc` slightly, or increase total sampling budget. Check if the task genuinely has near-100% filtering |
| Worst-task improves but others regress | Overly aggressive worst-task focus | Reduce `lambda` toward 0 to balance improvement-awareness vs. absolute-reward focus |

## Limitations

- **Requires per-task reward signals.** MT-GRPO needs to compute reward per task per step. If your setup only provides a single blended reward, you cannot apply the IWU mechanism.
- **Overhead from resampling.** The RP sampler generates extra rollouts to compensate for filtering, increasing compute cost by roughly `1/(1-max(rho_k))` in the worst case. With very high filtering rates, this cost compounds.
- **Not designed for single-task training.** The algorithm's benefits come entirely from multi-task balancing. For single-task GRPO, use standard GRPO or DAPO directly.
- **Assumes tasks are equally important.** The minimax formulation treats all tasks symmetrically. If some tasks genuinely matter more than others, you'd need to modify the objective with task-specific importance weights on top of the dynamic weights.
- **Tested at 3B scale.** The paper validates on Qwen-2.5-3B. While the algorithm is scale-agnostic in principle, the optimal hyperparameters (especially `beta`, `lambda`) may need re-tuning at larger model scales.

## Reference

**Paper:** [Multi-Task GRPO: Reliable LLM Reasoning Across Tasks](https://arxiv.org/abs/2602.05547v1) (Ramesh et al., 2026)
**What to look for:** Algorithm 1 (full MT-GRPO pseudocode), Equation 5 (IWU update rule), Section 3.2 (ratio-preserving sampler derivation), Table 1 (3-task results showing 16-28% worst-task gains), and the `lambda` ablation in Figure 3.