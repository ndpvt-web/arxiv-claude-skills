---
name: "rapid-real-time-deterministic-trajectory"
description: "Distill diffusion-based trajectory planners into fast deterministic policies using score-regularized optimization and safety-critic supervision. Use when: 'distill diffusion model into real-time policy', 'speed up diffusion planner for deployment', 'score-regularized policy extraction', 'build real-time autonomous driving planner', 'convert iterative diffusion to single-pass inference', 'add safety critic to trajectory planning'."
---

# RAPiD: Deterministic Policy Extraction from Diffusion Behavior Priors

This skill enables Claude to implement the RAPiD framework — a method for distilling pretrained diffusion-based trajectory planners into deterministic, single-forward-pass policies. The core idea is to use the score function (gradient of the log-density) of a trained diffusion model as a behavior regularizer during policy optimization, combined with a critic network trained to mimic a predictive driver controller (PDC) for safety-focused supervision. The result is an 8x inference speedup over diffusion baselines while preserving multimodal driving behavior fidelity.

## When to Use

- When the user has a trained diffusion model (DDPM, score-based, or flow-matching) generating trajectories and needs to deploy it in real-time with strict latency budgets
- When building an autonomous driving planner that must produce trajectories in a single forward pass instead of iterative denoising
- When the user wants to regularize a policy network against a pretrained generative model to stay within the distribution of human-like behavior
- When adding a safety-oriented critic to trajectory optimization that goes beyond standard imitation learning
- When distilling any iterative generative model into a feedforward network while retaining distributional fidelity
- When evaluating planners on nuPlan closed-loop or interPlan generalization benchmarks

## Key Technique

**Score-Regularized Policy Optimization.** Standard diffusion planners (e.g., Diffuser, CTG++) model the distribution of human trajectories p(τ) and sample via iterative denoising — typically 10-100 forward passes per plan. RAPiD eliminates this by training a deterministic policy π_θ(τ|s) that outputs a trajectory in one pass. The trick is regularization: during policy optimization, the score function ∇_τ log p(τ) from the frozen diffusion model penalizes the policy whenever its output drifts away from high-density regions of the learned trajectory distribution. This acts as a soft constraint keeping the policy's outputs "in-distribution" without requiring sampling from the diffusion model at inference time. The loss combines a task reward, the score-based regularizer, and a KL-style divergence term.

**Critic as PDC Imitator.** Instead of relying solely on imitation loss (L2 to expert trajectories), RAPiD trains a critic network Q_ψ(s, τ) via behavioral cloning of a Predictive Driver Controller — a rule-based or model-based controller that evaluates trajectories for collision risk, lane adherence, comfort (jerk/acceleration), and progress. This critic provides dense, per-trajectory value estimates during policy optimization, giving richer gradients than sparse reward signals. The critic is trained offline and frozen during policy optimization.

**Three-Phase Training.** Phase 1: Train a diffusion model on human driving trajectories to learn p(τ). Phase 2: Train the critic Q_ψ by imitating PDC evaluations on trajectory-state pairs. Phase 3: Optimize the deterministic policy π_θ using the combined objective of critic value maximization + score regularization + optional behavioral cloning anchor.

## Step-by-Step Workflow

1. **Prepare trajectory dataset.** Collect or load driving trajectories as sequences of (x, y, heading, velocity) waypoints at fixed time intervals (typically 0.5s steps over an 8s horizon = 16 waypoints). Each trajectory is paired with scene context: ego state, map features (lane centerlines, traffic signals), and nearby agent states.

2. **Train the diffusion behavior prior.** Implement a DDPM or score-based diffusion model that takes scene context as conditioning and generates trajectory sequences. Use a U-Net or Transformer backbone over the trajectory dimension. Train with standard denoising score matching loss:
   ```python
   # Denoising score matching
   noise = torch.randn_like(trajectory)
   noisy_traj = sqrt_alpha * trajectory + sqrt_one_minus_alpha * noise
   predicted_noise = diffusion_model(noisy_traj, timestep, scene_context)
   loss = F.mse_loss(predicted_noise, noise)
   ```

3. **Extract the score function.** After training, the score ∇_τ log p(τ) is available via the diffusion model's noise prediction: `score = -predicted_noise / sqrt(1 - alpha_bar)`. Freeze the diffusion model weights entirely.

4. **Train the safety critic.** Build a critic network Q_ψ(s, τ) that takes scene context and a candidate trajectory, outputting a scalar value. Train it to regress PDC scores — composite metrics combining collision probability, lane deviation, comfort (lateral/longitudinal jerk), and route progress:
   ```python
   pdc_score = w_safety * collision_free + w_comfort * comfort_score + w_progress * progress_score
   critic_loss = F.mse_loss(critic(scene_context, trajectory), pdc_score)
   ```

5. **Initialize the deterministic policy network.** Build π_θ as a feedforward network (MLP or lightweight Transformer) mapping scene context → trajectory. Initialize from the behavioral cloning solution (L2 to expert trajectories) for stable starting point.

6. **Optimize with score-regularized objective.** Combine three loss terms to train the policy:
   ```python
   # Forward pass
   predicted_traj = policy(scene_context)

   # Critic value (maximize)
   value = critic(scene_context, predicted_traj)

   # Score regularization (keep in-distribution)
   score = get_score(diffusion_model, predicted_traj, scene_context)
   score_reg = -torch.sum(score * predicted_traj, dim=-1).mean()

   # Behavioral cloning anchor
   bc_loss = F.mse_loss(predicted_traj, expert_traj)

   # Combined objective
   loss = -lambda_v * value + lambda_s * score_reg + lambda_bc * bc_loss
   ```

7. **Tune regularization weights.** Start with lambda_s dominant (e.g., 1.0) and lambda_v moderate (e.g., 0.1) to prevent early policy collapse. Anneal lambda_bc from 1.0 toward 0.0 over training as the score regularizer takes over distributional constraint duties.

8. **Validate on closed-loop simulation.** Test the trained policy in nuPlan's closed-loop simulator or equivalent. Measure: collision rate, driveable area compliance, comfort (max jerk), progress along route, and overall planner score. Compare against the original diffusion planner and pure BC baselines.

9. **Benchmark inference latency.** Profile single-pass policy inference vs. K-step diffusion sampling. Target: policy forward pass under 10ms on GPU (vs. 80ms+ for 20-step diffusion). Verify the 8x speedup holds on your target hardware.

10. **Deploy with fallback monitoring.** In production, run the deterministic policy as primary planner. Optionally keep the diffusion model available as a monitor: if the score ∇_τ log p(τ_predicted) falls below a threshold (trajectory is out-of-distribution), trigger a fallback to the full diffusion planner or a safety controller.

## Concrete Examples

**Example 1: Distilling an existing Diffuser model for real-time deployment**

User: "I have a trained Diffuser model that takes 80ms per plan with 20 denoising steps. I need it under 10ms for our AV stack. Can you help me distill it?"

Approach:
1. Load the pretrained Diffuser and freeze its weights. Extract its score function by wrapping the noise prediction head.
2. Build a lightweight MLP policy: `[scene_dim] -> 256 -> 256 -> 256 -> [16*4]` (16 waypoints, 4 features each).
3. Pre-train the policy with BC loss against the dataset for 50 epochs as initialization.
4. Implement the PDC critic: compute collision-free, comfort, and progress scores from the dataset's scenario metadata.
5. Train the critic on (context, trajectory, pdc_score) tuples for 30 epochs.
6. Run score-regularized policy optimization for 100 epochs with lambda_s=1.0, lambda_v=0.1, lambda_bc=0.5 (annealed to 0.0 by epoch 80).
7. Profile: MLP forward pass should be ~2ms vs. 80ms for 20-step diffusion.

Output:
```
Distilled Policy Performance:
  Inference latency:  2.1ms  (vs. 80ms diffusion, 38x speedup)
  Collision rate:     1.2%   (vs. 1.0% diffusion)
  Comfort score:      0.91   (vs. 0.93 diffusion)
  Route progress:     0.97   (vs. 0.97 diffusion)
  nuPlan composite:   84.2   (vs. 85.1 diffusion)
```

**Example 2: Adding score regularization to an existing BC policy**

User: "My behavioral cloning planner works but produces erratic trajectories in rare scenarios. How can I use a diffusion prior to regularize it?"

Approach:
1. Train a diffusion model on the same trajectory dataset as a density estimator. This captures the full multimodal distribution of driving behavior.
2. Compute the score function from the diffusion model for any candidate trajectory.
3. Add a score-regularization term to the existing BC training loop:
   ```python
   # Existing BC loss
   bc_loss = F.mse_loss(policy(ctx), expert_traj)
   # New: score regularization
   with torch.no_grad():
       score = diffusion_score(policy(ctx), ctx)
   score_penalty = -torch.sum(score * policy(ctx), dim=-1).mean()
   loss = bc_loss + 0.5 * score_penalty
   ```
4. The score penalty pushes erratic predictions back toward high-density regions of the learned trajectory distribution without changing the policy architecture.

Output: The policy's worst-case trajectories improve significantly — out-of-distribution predictions are pulled toward plausible driving modes, reducing erratic lane departures by ~40% while maintaining average-case performance.

**Example 3: Building a safety critic from a rule-based controller**

User: "I want to build the PDC-imitating critic for my planner. I have a rule-based safety checker that scores trajectories."

Approach:
1. Generate a dataset of (scene, trajectory, safety_score) tuples by running the rule-based checker on both expert and perturbed trajectories:
   ```python
   for scene in dataset:
       for traj in [expert_traj, *perturbed_trajs]:
           score = rule_checker.evaluate(scene, traj)
           critic_data.append((scene, traj, score))
   ```
2. Perturb expert trajectories with Gaussian noise at multiple scales (sigma=0.1, 0.5, 1.0) to cover the full quality spectrum — the critic needs to see both good and bad trajectories.
3. Train a critic network `Q(scene_features, traj) -> scalar` with MSE regression on the safety scores.
4. Validate by checking that the critic correctly ranks: expert > slightly-perturbed > heavily-perturbed trajectories.
5. Freeze the critic and use it in the RAPiD policy optimization loop.

Output:
```python
class SafetyCritic(nn.Module):
    def __init__(self, scene_dim, traj_dim, hidden=256):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(scene_dim + traj_dim, hidden),
            nn.ReLU(),
            nn.Linear(hidden, hidden),
            nn.ReLU(),
            nn.Linear(hidden, 1)
        )

    def forward(self, scene_features, trajectory):
        x = torch.cat([scene_features, trajectory.flatten(-2)], dim=-1)
        return self.net(x).squeeze(-1)
```

## Best Practices

- **Do:** Initialize the policy with behavioral cloning before score-regularized optimization. Cold-starting from random weights with the combined objective leads to unstable training.
- **Do:** Generate a quality-diverse critic training set by adding noise at multiple scales to expert trajectories. A critic that only sees near-expert trajectories cannot distinguish moderate from catastrophic failures.
- **Do:** Monitor the score magnitude during policy training. If `||∇_τ log p(τ)||` grows unboundedly, the policy is diverging from the prior — increase lambda_s or reduce learning rate.
- **Do:** Use the score function as an out-of-distribution detector at deployment time. Low log-density under the diffusion prior signals an unreliable prediction.
- **Avoid:** Training the diffusion prior and policy jointly end-to-end. The prior must be fully converged and frozen before policy extraction — joint training destabilizes the score estimates.
- **Avoid:** Setting lambda_bc to zero too early. The BC anchor prevents catastrophic forgetting of expert behavior during the early phase when the critic and score gradients may conflict.

## Error Handling

- **Score function returns NaN/Inf:** This happens when the diffusion model receives trajectories far outside its training distribution. Clamp the score magnitude: `score = torch.clamp(score, -10, 10)`. If persistent, the policy has diverged — restart from the BC checkpoint with lower learning rate.
- **Critic produces constant outputs:** The critic training set lacks diversity. Ensure you include trajectories spanning the full quality range (collisions, near-misses, smooth driving). Verify the PDC scoring function produces varied outputs.
- **Policy collapses to mean trajectory:** Over-regularization. Reduce lambda_s gradually. Also check that the critic provides gradient signal distinguishing trajectory quality — a flat critic causes the score regularizer to dominate, pulling everything toward the mode.
- **Latency not meeting target:** Ensure the policy network is appropriately sized. A 3-layer MLP with 256 hidden units should run under 1ms on modern GPUs. If using Transformer-based context encoding, batch map and agent encoding separately and cache across planning cycles.

## Limitations

- The approach requires a well-trained diffusion prior. If the diffusion model has poor coverage of the trajectory distribution, the score regularizer will push the policy toward an incomplete manifold, missing valid driving behaviors.
- The safety critic is only as good as the PDC it imitates. If the rule-based controller has blind spots (e.g., does not penalize certain uncomfortable maneuvers), those gaps propagate to the learned policy.
- RAPiD produces unimodal outputs per forward pass. For scenarios requiring explicit multimodal predictions (e.g., presenting multiple route options to a decision module), the original diffusion sampler may still be necessary.
- The method is validated on structured driving domains (nuPlan, interPlan). Generalization to unstructured environments (parking lots, off-road) requires retraining both the prior and critic on appropriate data.
- Score-regularized optimization adds hyperparameter complexity (three lambda weights plus annealing schedule). These require scenario-specific tuning.

## Reference

- **Paper:** [RAPiD: Real-time Deterministic Trajectory Planning via Diffusion Behavior Priors](https://arxiv.org/abs/2602.07339v1) — Focus on Section 3 (score-regularized policy optimization objective), Section 4 (critic architecture and PDC imitation), and Table 1/2 (nuPlan and interPlan benchmark comparisons).
- **Code:** [github.com/ruturajreddy/RAPiD](https://github.com/ruturajreddy/RAPiD) — Pretrained models forthcoming.