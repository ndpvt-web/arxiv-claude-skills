---
name: "self-improving-world-modelling-latent"
description: |
  Implement the SWIRL (Self-improving World modelling with Inverse and foRward Latent dynamics) framework
  for building self-improving world models that learn from unlabelled state transitions by treating actions
  as latent variables. Uses coordinate ascent between a Forward World Model and Inverse Dynamics Model,
  each trained with GRPO reinforcement learning using the other model's log-probability as reward.

  Trigger phrases:
  - "Build a world model from unlabelled trajectories"
  - "Implement SWIRL self-improving framework"
  - "Train forward and inverse dynamics models with latent actions"
  - "Set up coordinate ascent between FWM and IDM"
  - "Predict state transitions without action labels"
  - "Self-improving loop for environment modelling"
---

# Self-Improving World Modelling with Latent Actions (SWIRL)

This skill enables Claude to implement the SWIRL framework: a self-improvement loop that builds world models from unlabelled state-only sequences. Instead of requiring costly action-annotated trajectories, SWIRL treats actions as a latent variable and alternates training between a Forward World Model P_θ(Y|X,Z) and an Inverse Dynamics Model Q_φ(Z|X,Y) using GRPO reinforcement learning, where each model's log-probability serves as the other's reward signal. This creates a coordinate ascent procedure that provably maximises both conditional mutual information and the evidence lower bound.

## When to Use

- When the user wants to learn a world model from state-only data (video frames, DOM snapshots, text states) without action annotations
- When building a forward predictor (given current state + action, predict next state) and the training data lacks action labels
- When implementing a self-play or self-improvement loop between two complementary models (forward prediction and inverse dynamics)
- When the user needs to predict environment transitions for planning, tool calling, web interaction, or physical simulation
- When setting up GRPO-based reinforcement learning where one model's output scores another model's generations
- When the user asks to implement variational information maximisation or ELBO maximisation for sequential decision tasks

## Key Technique

SWIRL decomposes world modelling into two complementary problems solved via coordinate ascent. The **Forward World Model (FWM)** P_θ(Y|X,Z) takes a current state X and action Z to predict the next state Y. The **Inverse Dynamics Model (IDM)** Q_φ(Z|X,Y) takes a state pair (X,Y) and infers what action Z caused the transition. When action labels are unavailable, each model bootstraps the other: the IDM proposes latent actions for state pairs, and the FWM generates predicted next states that the IDM must be able to "explain." Both are language models trained with GRPO.

The training alternates two phases. **Phase I (Variational Information Maximisation):** Freeze the IDM. Sample a latent action z ~ Q_φ(z|x,y), then generate G candidate next-states from P_θ(·|x,z). Score each candidate ŷ_k using reward r_k = log Q_φ(z|x,ŷ_k) — high reward means the IDM can recover the original action from the generated state, ensuring the FWM produces action-distinguishable outputs. Update θ with GRPO. **Phase II (ELBO Maximisation):** Freeze the FWM. Sample G candidate actions from Q_φ(·|x,y). Score each ẑ_k using reward r_k = log P_θ(y|x,ẑ_k) — high reward means the FWM can reconstruct the real next state from the inferred action, ensuring the IDM produces semantically meaningful actions. Update φ with GRPO.

This loop is theoretically grounded: Phase I maximises a variational lower bound on conditional mutual information I(Z;Ŷ|X), ensuring predicted states carry information about actions. Phase II maximises the ELBO of log P_θ(Y|X), ensuring inferred actions explain real transitions. Together they perform coordinate ascent on a joint objective, with proven convergence guarantees.

## Step-by-Step Workflow

1. **Prepare an unlabelled state-pair dataset.** Collect (x_t, y_{t+1}) pairs from your domain: consecutive video frames, sequential DOM snapshots, before/after text states, or API call sequences. No action labels needed. Format each pair so both states are representable as model inputs (text descriptions, image tokens, structured markup).

2. **Initialise the FWM and IDM.** Start from a pretrained language model (or VLM for visual tasks). Create two copies or two LoRA adapters — one for forward prediction P_θ(Y|X,Z), one for inverse dynamics Q_φ(Z|X,Y). Optionally warm-start with supervised fine-tuning on any available labelled data (even a small seed set helps).

3. **Define prompt templates for both models.** The FWM prompt takes the form: `"Given the current state: {X} and the action: {Z}, predict the next state."` The IDM prompt takes: `"Given the current state: {X} and the next state: {Y}, what action caused this transition?"` Tailor these to your domain (e.g., for web: "Given this HTML and the resulting HTML, what user interaction occurred?").

4. **Run Phase I — FWM update (freeze IDM).** For each training batch of (x, y) pairs: (a) sample latent action z ~ Q_φ(z|x,y) using the frozen IDM, (b) generate G candidate next-states {ŷ_1,...,ŷ_G} from P_θ(·|x,z), (c) compute rewards r_k = log Q_φ(z|x,ŷ_k), (d) compute group-relative advantages A_k = (r_k - mean(r)) / std(r), (e) update θ using the GRPO policy gradient with these advantages.

5. **Run Phase II — IDM update (freeze FWM).** For each training batch: (a) sample G candidate actions {ẑ_1,...,ẑ_G} from Q_φ(·|x,y), (b) compute rewards r_k = log P_θ(y|x,ẑ_k), (c) compute group-relative advantages, (d) update φ using GRPO. This ensures the IDM produces actions that the FWM can use to reconstruct observed transitions.

6. **Iterate until convergence.** Alternate Phase I and Phase II. Monitor both reward signals — if FWM reward plateaus but IDM reward still increases (or vice versa), the models are still improving. Typically 3-5 iterations suffice, with each iteration training for one epoch over the dataset.

7. **Implement GRPO correctly.** For each group of G rollouts, compute advantages as: A_k = (r_k - mean({r_1,...,r_G})) / std({r_1,...,r_G}). The policy gradient loss is: L = -E[min(ρ_k * A_k, clip(ρ_k, 1-ε, 1+ε) * A_k)] + β * KL(π_θ || π_ref), where ρ_k is the importance ratio and π_ref is the initial policy (for KL regularisation).

8. **Handle multi-step rollouts.** For tasks requiring T-step predictions (e.g., planning), autoregressively apply the FWM: predict y_2 from (x_1, z_1), then y_3 from (y_2, z_2), etc. Use the IDM to infer intermediate actions from the ground-truth trajectory, then evaluate the FWM's multi-step accuracy.

9. **Evaluate on held-out transitions.** Measure FWM accuracy (does the predicted next state match ground truth?) and IDM accuracy (does the inferred action, when fed to FWM, reconstruct the correct next state?). For text domains, use exact match or ROUGE; for visual domains, use perceptual similarity or VLM-based scoring.

10. **Deploy the trained models.** The FWM serves as a world simulator for planning (predict outcomes of candidate actions). The IDM serves as an action labeller (annotate unlabelled trajectories with inferred actions for downstream use).

## Concrete Examples

**Example 1: Web interaction prediction without action labels**

User: "I have 10K pairs of before/after HTML DOM snapshots from a website, but no labels for what user actions caused each transition. Build a model that can predict the next DOM state given a current state and a user action."

Approach:
1. Parse each (before_DOM, after_DOM) pair into simplified text representations
2. Initialise FWM and IDM from a 7B instruction-tuned model with LoRA adapters
3. IDM prompt: "Before: {before_DOM}\nAfter: {after_DOM}\nWhat user action caused this change?"
4. FWM prompt: "Current page: {before_DOM}\nUser action: {action}\nPredict the page after this action."
5. Run 3 SWIRL iterations, each with G=4 rollouts per sample
6. The IDM learns to output actions like "Click the 'Submit' button" or "Type 'hello' in the search box"
7. The FWM learns to predict DOM changes conditioned on these inferred actions

Output:
```python
# Phase I: Update FWM with frozen IDM
for x, y in batch:
    z = idm.generate(f"Before: {x}\nAfter: {y}\nAction:")  # sample latent action
    candidates = [fwm.generate(f"State: {x}\nAction: {z}\nNext state:") for _ in range(G)]
    rewards = [idm.log_prob(z, f"Before: {x}\nAfter: {c}\nAction:") for c in candidates]
    advantages = (rewards - mean(rewards)) / std(rewards)
    fwm.grpo_update(advantages)

# Phase II: Update IDM with frozen FWM
for x, y in batch:
    action_candidates = [idm.generate(f"Before: {x}\nAfter: {y}\nAction:") for _ in range(G)]
    rewards = [fwm.log_prob(y, f"State: {x}\nAction: {a}\nNext state:") for a in action_candidates]
    advantages = (rewards - mean(rewards)) / std(rewards)
    idm.grpo_update(advantages)
```

**Example 2: Tool-calling world model for API planning**

User: "I want to build a world model for API tool calling. I have logs of API request/response sequences but the 'intent' behind each call isn't labelled."

Approach:
1. Structure data as state pairs: state = system context before API call, next_state = system context after response
2. IDM learns to infer the latent intent/action: "search for restaurants" or "book appointment at 3pm"
3. FWM learns to predict API responses given the current context and an intent description
4. Train with SWIRL for 3 iterations using G=8 rollouts

Output:
```python
# Data preparation
pairs = []
for log in api_logs:
    for t in range(len(log) - 1):
        pairs.append({
            "x": log[t]["context"],   # system state before call
            "y": log[t+1]["context"], # system state after response
        })

# SWIRL training loop
for iteration in range(3):
    # Phase I: FWM update
    train_fwm_with_idm_reward(fwm, idm, pairs, G=8, lr=1e-5)
    # Phase II: IDM update
    train_idm_with_fwm_reward(idm, fwm, pairs, G=8, lr=1e-5)

# Deployment: use FWM for planning
def plan_api_sequence(goal, current_state, fwm, idm, max_steps=5):
    trajectory = [current_state]
    for step in range(max_steps):
        # Generate candidate actions with IDM or propose them
        action = propose_action(goal, current_state)
        next_state = fwm.predict(current_state, action)
        trajectory.append((action, next_state))
        if goal_reached(next_state, goal):
            break
        current_state = next_state
    return trajectory
```

**Example 3: Video frame prediction with latent camera/object actions**

User: "I have unlabelled video data. I want to predict future frames conditioned on inferred motion descriptions."

Approach:
1. Extract consecutive frame pairs from videos (sample uniformly, ~30K pairs per iteration)
2. Use a VLM as the base model; IDM infers natural language action descriptions like "camera pans left" or "person raises hand"
3. FWM generates predicted next-frame descriptions (or image tokens) given current frame + action
4. Reward: IDM scores how well it can recover the action from (current_frame, predicted_frame)
5. Run 3-5 SWIRL iterations

Output:
```
Iteration 1: FWM accuracy 34%, IDM action recovery 41%
Iteration 2: FWM accuracy 49%, IDM action recovery 58%
Iteration 3: FWM accuracy 57%, IDM action recovery 64%
```

## Best Practices

- **Do:** Start with a supervised warm-up if any labelled data exists (even 1-5% of your dataset). SWIRL converges faster from a reasonable initialisation than from a cold start.
- **Do:** Use separate model weights (or separate LoRA adapters) for FWM and IDM. Shared weights are memory-efficient but less stable during coordinate ascent.
- **Do:** Monitor both reward signals across iterations. Both should trend upward; if one collapses, reduce its learning rate or add stronger KL regularisation against the reference policy.
- **Do:** Set group size G >= 4 for stable advantage estimation. G=8 is a good default; larger G gives better gradient estimates but costs more compute.
- **Avoid:** Updating both models simultaneously. The coordinate ascent guarantee requires one model to be frozen while the other updates. Violating this breaks the theoretical convergence properties.
- **Avoid:** Skipping KL regularisation in GRPO. Without the KL penalty against π_ref, models can diverge to degenerate solutions where the FWM ignores actions or the IDM outputs trivial action labels.

## Error Handling

- **Reward collapse (all rewards near-zero):** The frozen model assigns uniformly low scores. Fix by increasing temperature during generation, using a warmer start, or reducing the number of GRPO iterations per phase.
- **Action mode collapse (IDM always outputs the same action):** Increase KL penalty β, or add diversity regularisation (e.g., penalise low entropy in Q_φ). Check that the training data has sufficient diversity in state transitions.
- **FWM ignores the action input:** The FWM learns to predict a "mean" next state regardless of Z. Increase the IDM reward weight, verify the action is positioned prominently in the prompt template, and ensure G is large enough to differentiate action-conditioned outputs.
- **Divergence between phases:** If performance oscillates instead of improving, reduce learning rates for both models and increase the number of inner GRPO steps per phase to ensure each model converges before the other updates.

## Limitations

- Requires substantial compute: each SWIRL iteration involves G rollouts per sample for both phases, so total generation cost is ~2G times a single training pass.
- Latent actions are inferred, not grounded to a fixed action space. For tasks requiring precise action labels (e.g., exact API parameters), post-hoc alignment to a known action vocabulary may be needed.
- Multi-step rollouts accumulate prediction errors. For T > 5 steps, consider re-anchoring with ground-truth intermediate states or using a separate planner.
- The framework assumes state pairs (x, y) are temporally adjacent. Non-adjacent or variable-gap pairs degrade IDM accuracy.
- Works best when transitions are deterministic or low-variance. Highly stochastic environments (where the same action from the same state can lead to many different outcomes) weaken the mutual information signal.

## Reference

**Paper:** [Self-Improving World Modelling with Latent Actions](https://arxiv.org/abs/2602.06130v1) — Qiu et al., 2026.
**What to look for:** Algorithm 1 (full pseudocode for the coordinate ascent loop), Theorems 3.1-3.2 (learnability guarantees for each phase), and Section 5 (benchmark-specific implementation details for visual and textual environments).