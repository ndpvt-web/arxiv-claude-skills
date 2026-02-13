---
name: "confounding-robust-continuous-control"
description: "Implement confounding-robust reward shaping for continuous control RL using causal Bellman upper bounds and PBRS. Use when: 'train RL with confounded data', 'reward shaping from offline dataset', 'handle unobserved confounders in RL', 'causal Bellman equation reward shaping', 'robust continuous control with missing state variables', 'potential-based reward shaping from observational data'."
---

# Confounding-Robust Continuous Control via Automatic Reward Shaping

This skill enables Claude to implement the causal upper-bound reward shaping framework from Juliani, Li & Bareinboim (AAMAS 2026). The core idea: learn a tight upper bound on optimal state values from offline data that may contain unobserved confounders, then use that bound as a potential function in Potential-Based Reward Shaping (PBRS) to accelerate online SAC training. This produces RL agents that are robust to missing state variables in the offline dataset — a common real-world scenario where logged data was collected under conditions the agent cannot fully observe.

## When to Use

- When the user wants to train a continuous control RL agent but their offline dataset has missing or hidden state variables (unobserved confounders)
- When the user asks to implement reward shaping from offline/observational data for MuJoCo or similar continuous control environments
- When the user needs to accelerate SAC training using learned potential functions derived from a separate offline dataset
- When the user wants to implement the causal Bellman equation for computing conservative value upper bounds
- When the user is building an RL pipeline that must be robust to distribution shift caused by confounding in behavioral data
- When the user asks about PBRS with learned potentials rather than hand-designed ones

## Key Technique

**The Problem.** Standard offline RL methods assume the logged behavioral policy's state observations are fully observable. When some state dimensions are missing (unobserved confounders), naive value estimation produces biased results — the agent learns a policy that exploits spurious correlations instead of causal structure.

**The Solution — Causal Bellman Upper Bound.** The method builds on the causal Bellman equation, which decomposes the value of a state into contributions from the action actually taken and counterfactual actions not taken, weighted by their respective probabilities. Concretely, for each state `s`:

```
V_upper(s) = π(a|s) * [r(s,a) + γ*V(s')] + (1 - π(a|s)) * [r_max + γ*max V(s_cf)]
```

where `s_cf` are counterfactual next-states produced by sampling alternative actions from the learned behavioral policy and predicting transitions via a learned dynamics model. The `r_max` term and the `max` over counterfactual values ensure this is an **upper bound** — it is pessimistic about what unseen confounders might have caused. Twin critics (minimum of two networks) tighten this bound.

**Integration with PBRS.** Once `V_upper(s)` is learned offline, it serves as the potential function `Φ(s)` in the PBRS framework. During online SAC training, the shaped reward becomes `r_shaped = r + γ*Φ(s') - Φ(s)`. PBRS guarantees that the optimal policy is preserved (policy invariance), while the shaped reward accelerates learning by encoding causal structure from the offline data.

## Step-by-Step Workflow

1. **Collect or load the offline dataset.** Use Minari to load standard datasets (e.g., `hopper-medium-v0`, `halfcheetah-expert-v0`) or prepare custom data as lists of `(s, a, r, s', done, truncated, step_in_episode)` transitions. Store as PyTorch tensors.

2. **Simulate confounding by removing state dimensions.** Identify which observation indices to drop (e.g., `--state_to_remove "[2]"` removes the third observation dimension from Hopper). Delete those columns from both `s` and `s'` in the offline dataset. This models real-world scenarios where the logging policy had access to variables the learning agent does not.

3. **Normalize states and rewards.** Compute per-dimension mean and std across the offline dataset. Z-score normalize states: `s_norm = (s - mean) / (std + 1e-7)`. Normalize rewards via z-score or min-max scaling. Track `max_r` and `min_r` for critic output clamping to `[min_r/(1-γ), max_r/(1-γ)]`.

4. **Pretrain the environment models (behavioral policy, dynamics, reward).** Train three neural networks on the offline data:
   - **Behavioral policy** (`GaussianNN`): 4-layer MLP (256 hidden) outputting mean and log-std of a TanhNormal distribution over actions. Loss: negative log-likelihood of observed actions.
   - **Dynamics model** (`RegressionNN`): 4-layer MLP predicting state deltas `Δs = s' - s`. Loss: MSE on state deltas.
   - **Reward model** (`RegressionNN`): 4-layer MLP predicting normalized reward. Loss: MSE.
   Save best checkpoints based on loss thresholds (policy > 0, reward > 0.001, dynamics > 0.005).

5. **Initialize twin critics with causal upper-bound targets.** Create two `Critic` networks (5-layer MLPs, 256 hidden, ReLU, output clamped to reward bounds) plus their target copies. Use separate Adam optimizers. Set soft-update rate `τ = 0.005`.

6. **Train critics using the causal Bellman target.** For each batch from the offline dataset:
   - Compute `π(a|s)` from the frozen behavioral policy.
   - Sample `N=25` counterfactual actions from the behavioral policy; predict their next-states via the dynamics model and rewards via the reward model.
   - Compute `V_target(s') = min(critic_target_1(s'), critic_target_2(s'))` for both the real next-state and each counterfactual next-state.
   - Enforce the upper-bound property: `V_cf = max(V_cf, V_real)`.
   - Combine: `target = π(a|s) * [r + γ*V(s')] + (1-π(a|s)) * [r_max + γ*V_cf_max]`.
   - Update critics via MSE or SmoothL1 loss against this target.
   - Soft-update target networks every `policy_delay` steps.

7. **Extract the learned potential function.** After training (typically 200 epochs, saving best after epoch 60), the critic network `V_upper(s)` becomes the potential `Φ(s)`. Save the critic checkpoint along with normalization statistics (mean, std).

8. **Integrate with online SAC training.** During SAC rollouts, at each step compute the shaped reward:
   ```python
   s_norm = (s - state_mean) / (state_std + 1e-7)
   sp_norm = (sp - state_mean) / (state_std + 1e-7)
   r_shaped = r + gamma * V_upper(sp_norm) - V_upper(s_norm)
   ```
   Feed `r_shaped` to SAC's replay buffer instead of `r`. The SAC algorithm itself is unchanged.

9. **Evaluate the trained agent.** Run the SAC policy in the full environment (with all state dimensions visible to the simulator). Compare cumulative returns against baselines: vanilla SAC (no shaping), SAC with naive offline value shaping (no causal correction), and oracle SAC (trained with full state).

10. **Tune key hyperparameters if performance is unsatisfactory.** The most impactful knobs are: number of counterfactual samples (`num_sample_neg_a`, default 30), action difference threshold (`neg_action_thres`, default 0.1), critic training epochs (`vs_epochs`, default 200), and the discount factor (`vs_gamma`, default 0.99).

## Concrete Examples

**Example 1: Reward shaping for Hopper with confounded offline data**

User: "I have offline Hopper data but the angular velocity of the thigh joint wasn't logged. I want to train a robust SAC agent."

Approach:
1. Load Minari Hopper datasets (simple, medium, expert):
   ```python
   import minari
   dataset = minari.load_dataset("hopper-medium-v0")
   ```
2. Remove the confounded state dimension (index 2 = thigh angular velocity):
   ```python
   states = states[:, [i for i in range(states.shape[1]) if i != 2]]
   next_states = next_states[:, [i for i in range(next_states.shape[1]) if i != 2]]
   ```
3. Train the causal upper-bound estimator:
   ```bash
   python main.py --data_set "hopper" --state_to_remove "[2]" \
       --vs_epochs 200 --vs_batch_size 1024 --vs_gamma 0.99
   ```
4. Load the saved critic checkpoint as `Φ(s)` and run SAC with shaped rewards.

Output: A SAC agent that achieves near-oracle performance despite never observing the thigh angular velocity during offline value estimation. Typical result: 85-95% of full-state oracle return vs. 60-70% for naive shaping.

**Example 2: Custom robotics dataset with hidden sensor readings**

User: "I have my own offline dataset from a robot arm where force-torque sensor data is missing. How do I apply confounding-robust reward shaping?"

Approach:
1. Format the custom data as transition tuples:
   ```python
   # Each transition: (state, action, reward, next_state, done, truncated, step)
   transitions = []
   for episode in your_data:
       for t in range(len(episode) - 1):
           transitions.append((
               torch.tensor(episode.obs[t], dtype=torch.float32),
               torch.tensor(episode.act[t], dtype=torch.float32),
               torch.tensor(episode.rew[t], dtype=torch.float32),
               torch.tensor(episode.obs[t+1], dtype=torch.float32),
               torch.tensor(episode.done[t], dtype=torch.float32),
               torch.tensor(episode.trunc[t], dtype=torch.float32),
               torch.tensor(t, dtype=torch.float32),
           ))
   ```
2. Modify the preprocessing at line 498 of `fin_train_value_state_new_continuous.py` to load your data format instead of Minari.
3. The missing force-torque dimensions are already absent from your observations — no need to manually remove them. The causal upper bound will account for the confounding.
4. Train the estimator, then integrate `Φ(s)` into your SAC loop.

Output: A potential function `Φ(s)` that provides reward shaping robust to the unobserved force-torque signals, improving sample efficiency of subsequent online SAC training.

**Example 3: Implementing the causal upper-bound critic from scratch**

User: "I want to implement the causal Bellman upper-bound value estimator in my own RL codebase."

Approach:
1. Define the critic architecture (5-layer MLP with clamped output):
   ```python
   class CausalCritic(nn.Module):
       def __init__(self, state_dim, min_v, max_v):
           super().__init__()
           self.net = nn.Sequential(
               nn.Linear(state_dim, 256), nn.ReLU(),
               nn.Linear(256, 256), nn.ReLU(),
               nn.Linear(256, 256), nn.ReLU(),
               nn.Linear(256, 256), nn.ReLU(),
               nn.Linear(256, 1),
           )
           self.min_v, self.max_v = min_v, max_v

       def forward(self, s):
           return torch.clamp(self.net(s), self.min_v, self.max_v)
   ```
2. Implement the causal Bellman target computation:
   ```python
   def causal_target(s, a, r, sp, policy, dynamics, reward_model, critic_target, gamma, n_cf=25):
       # Probability of observed action
       pi_a = torch.exp(policy(s).log_prob(a))

       # Value of action taken
       v_taken = r + gamma * critic_target(sp)

       # Sample counterfactual actions and predict outcomes
       cf_actions = policy(s).rsample((n_cf,))            # [n_cf, batch, action_dim]
       cf_next = s.unsqueeze(0) + dynamics(s.unsqueeze(0).expand(n_cf,-1,-1),
                                            cf_actions)    # predicted delta
       cf_v = critic_target(cf_next).max(dim=0).values     # max over samples
       cf_v = torch.max(cf_v, v_taken)                     # enforce upper bound

       r_max = reward_model.max_reward
       v_not_taken = r_max + gamma * cf_v

       # Weighted combination
       target = pi_a * v_taken + (1 - pi_a) * v_not_taken
       return target.detach()
   ```
3. Train with MSE loss against this target; soft-update target networks.

Output: A reusable `CausalCritic` module and `causal_target` function that can plug into any PyTorch RL pipeline.

## Best Practices

- **Do:** Pretrain environment models (policy, dynamics, reward) thoroughly before critic training. Bad environment models produce noisy counterfactuals that loosen the upper bound. Save checkpoints based on validation loss.
- **Do:** Use twin critics (minimum of two independent estimates) to tighten the upper bound. This is analogous to TD3/SAC's twin-Q trick and is critical for stability.
- **Do:** Clamp critic outputs to `[min_r/(1-γ), max_r/(1-γ)]`. Without clamping, the upper bound can diverge during early training.
- **Do:** Normalize states with z-score normalization using offline dataset statistics. Pass the same mean/std to the online SAC phase for consistent potential evaluation.
- **Avoid:** Using too few counterfactual action samples. Below 10 samples, the upper bound becomes loose and noisy. 25-30 is the empirically validated range.
- **Avoid:** Training the critic for too few epochs or saving early checkpoints. The paper finds that checkpoints before epoch 60 are unreliable; the best models typically emerge between epochs 100-200.
- **Avoid:** Applying this method when confounding is absent. If the offline dataset has full state observability, standard offline RL or simpler reward shaping will be more efficient. The causal upper bound adds computational overhead that only pays off under genuine confounding.

## Error Handling

- **Divergent critic values:** If critic outputs hit the clamp bounds persistently, reduce the learning rate or increase batch size. Also verify that reward normalization is correct — unnormalized rewards with large magnitude cause instability.
- **Poor environment model fit:** If dynamics MSE loss stays above 0.05 after pretraining, increase pretraining epochs or use a larger hidden dimension. Alternatively, check that state normalization is applied consistently to both inputs and targets.
- **NaN in log-probabilities:** TanhNormal distributions can produce extreme log-probs. Ensure log-std is clamped to `[-20, 2]` in the GaussianNN policy network.
- **Truncation artifacts:** For time-limited environments, use predicted next-states (from the dynamics model) instead of terminal observations when computing critic targets at truncation boundaries. The reference code handles this with: `sp = torch.where(trunc == 1, predicted_sp, real_sp)`.
- **Action threshold sensitivity:** If `neg_action_thres` is too small, counterfactual actions collapse to the observed action; too large and they become unrealistic. Start with 0.1 and adjust based on the action space scale.

## Limitations

- **Continuous control only.** The method is designed for continuous action spaces with TanhNormal policies. Discrete or hybrid action spaces require a different counterfactual sampling strategy.
- **Requires offline data.** You need a pre-collected offline dataset to learn the potential function. The method does not work in a purely online setting.
- **Computational cost of counterfactual sampling.** Sampling 25+ counterfactual actions per transition and running them through the dynamics model is expensive. Critic training is roughly 25x slower than standard offline value estimation.
- **Assumes confounding structure.** The upper bound is tight when confounders affect the behavioral policy but not the transition dynamics directly. If confounders alter the environment physics themselves (e.g., hidden friction coefficients changing dynamics), the bound may be loose.
- **MuJoCo-focused evaluation.** The paper validates on Hopper, HalfCheetah, Walker2d, Ant, and Adroit (pen, door, relocate). Performance on substantially different domains (e.g., high-dimensional image observations) is untested.
- **Reward function must be bounded.** The clamping strategy requires known `min_r` and `max_r`. Environments with unbounded rewards need additional handling.

## Reference

**Paper:** Juliani, Li & Bareinboim, "Confounding Robust Continuous Control via Automatic Reward Shaping," AAMAS 2026. [arXiv:2602.10305](https://arxiv.org/abs/2602.10305v1). Look for: Section 4 (causal Bellman equation for continuous control), Algorithm 1 (upper-bound critic training), and Section 6 (experimental comparison on MuJoCo benchmarks).

**Code:** [github.com/mateojuliani/confounding_robust_cont_control](https://github.com/mateojuliani/confounding_robust_cont_control) — PyTorch implementation with Minari dataset integration.