---
name: "embocoach-bench-benchmarking-ai-agents"
description: |
  Closed-loop agentic workflow for autonomously engineering embodied robot policies.
  Implements the EmboCoach-Bench draft-debug-optimize cycle where executable code is the
  universal interface: the agent iteratively generates reward functions, policy architectures,
  and training configurations, then uses simulation feedback to self-correct and improve.
  Trigger phrases:
  - "Design a reward function for this robot task"
  - "Set up an RL training pipeline for embodied control"
  - "Debug my robot policy using simulation feedback"
  - "Iteratively optimize this manipulation/locomotion policy"
  - "Generate a closed-loop training workflow for MuJoCo/IsaacGym"
  - "Benchmark an LLM agent on embodied policy engineering"
---

# EmboCoach-Bench: Closed-Loop Agentic Workflow for Embodied Policy Engineering

This skill enables Claude to act as an autonomous embodied AI engineer, applying the EmboCoach-Bench methodology to iteratively draft, debug, and optimize robot control policies through executable code. Rather than producing static one-shot solutions, Claude follows a simulation-in-the-loop workflow: generating reward functions, policy architectures, and hyperparameter configurations as code, executing them against a simulator, interpreting feedback (training curves, success rates, error traces), and refining the solution across multiple iterations. This approach was shown to surpass human-engineered baselines by 26.5% in average success rate across 32 RL and IL tasks.

## When to Use

- When the user needs to design a reward function for a robotic manipulation or locomotion task in MuJoCo, IsaacGym, or similar physics simulators
- When the user asks to set up an end-to-end RL or IL training pipeline for embodied control (e.g., grasping, pushing, locomotion, assembly)
- When a robot policy is failing and the user wants to iteratively debug it using simulation feedback
- When the user wants to compare or benchmark different policy architectures (MLP, transformer, diffusion policy) on embodied tasks
- When the user needs to translate a natural-language task specification into executable training code with reward shaping
- When the user is tuning hyperparameters for robot learning and wants a systematic closed-loop optimization approach
- When the user wants to resurrect a near-totally-failed training run through iterative self-correction

## Key Technique

**Executable code as universal interface.** EmboCoach-Bench treats all components of an embodied learning pipeline — reward functions, observation/action spaces, policy network definitions, training hyperparameters, and evaluation scripts — as code artifacts that an LLM agent generates, executes, and refines. This eliminates ambiguity: every decision the agent makes is validated by actual simulation execution rather than theoretical reasoning alone.

**Draft-Debug-Optimize closed loop.** The workflow proceeds in three phases. In the *draft* phase, the agent generates an initial implementation (reward function, policy architecture, training config) based on the task specification and environment API. In the *debug* phase, the agent runs the code in simulation, captures execution errors, training metrics, and episode rollouts, then fixes bugs and structural issues. In the *optimize* phase, the agent analyzes success rates and reward curves to iteratively refine reward shaping terms, adjust hyperparameters, or swap policy architectures — for instance, replacing an MLP policy with a diffusion policy for multi-modal action distributions. Critically, this loop can self-correct from catastrophic failures: the agent can diagnose why a reward function produces degenerate behavior (e.g., reward hacking) and restructure it using physics-informed constraints.

**Physics-informed reward design.** Rather than relying on sparse binary rewards, the agent decomposes task objectives into physically meaningful sub-rewards: distance-to-target, contact forces, joint velocity penalties, orientation alignment, and progress-based shaping terms. Each sub-reward is weighted and the agent adjusts weights based on observed training dynamics — increasing a contact reward if the robot fails to grasp, or adding a smoothness penalty if trajectories are jerky.

## Step-by-Step Workflow

1. **Parse the task specification.** Extract the target behavior (e.g., "pick up the red block and place it on the shelf"), the simulation environment (MuJoCo, IsaacGym, robosuite, etc.), the robot model (Franka, UR5, humanoid), and any constraints (max episode length, safety limits, available sensors).

2. **Inspect the environment API.** Read the environment source code or documentation to identify the observation space (joint positions, end-effector pose, object states, images), action space (joint torques, delta positions, gripper commands), and available `step()`/`reset()` interfaces. Map these to the task requirements.

3. **Draft the reward function as executable code.** Write a Python function that computes a scalar reward from the observation dict. Decompose it into physics-informed sub-rewards:
   ```python
   def compute_reward(obs, action, info):
       # Distance-based shaping
       reach_reward = -np.linalg.norm(obs["ee_pos"] - obs["target_pos"])
       # Contact/grasp reward
       grasp_reward = float(obs["gripper_contact"] and obs["object_grasped"])
       # Placement accuracy
       place_reward = -np.linalg.norm(obs["object_pos"] - obs["goal_pos"]) * grasp_reward
       # Action smoothness penalty
       smooth_penalty = -0.01 * np.linalg.norm(action)
       return 0.3 * reach_reward + 1.0 * grasp_reward + 2.0 * place_reward + smooth_penalty
   ```

4. **Draft the policy architecture and training configuration.** Select an appropriate algorithm (PPO, SAC, BC, diffusion policy) and define the network architecture, learning rate, batch size, and training horizon as a config dict or YAML file. For multi-modal tasks, prefer diffusion policies; for simple continuous control, start with SAC or PPO with MLP.

5. **Execute the initial training run in simulation.** Launch training for a short horizon (e.g., 10-50k steps) and capture: (a) any runtime errors or exceptions, (b) reward curves, (c) success rate metrics, (d) sample rollout trajectories if available.

6. **Debug execution failures.** If the run crashes, parse the traceback to identify shape mismatches, missing observation keys, NaN gradients, or API incompatibilities. Fix the code and re-run. Common fixes:
   - Observation key not in dict → inspect `env.observation_space` and correct the key name
   - Action dimension mismatch → check `env.action_space.shape` and adjust policy output
   - NaN in reward → add clipping or check for division-by-zero in distance computations

7. **Analyze training dynamics and diagnose reward pathology.** If the policy converges to degenerate behavior (e.g., reward hacking by exploiting a loophole, or stalling at a local optimum), diagnose the root cause:
   - Reward hacking → add constraints or reweight sub-rewards
   - Sparse reward stalling → add denser shaping terms (curriculum of sub-goals)
   - Oscillatory training → reduce learning rate, increase batch size, or add gradient clipping

8. **Optimize through iterative refinement.** Adjust reward weights, swap policy architectures, or modify hyperparameters based on the analysis. Each iteration should target one specific hypothesis (e.g., "the grasp reward weight is too low relative to the reach reward"). Re-run training and compare success rates.

9. **Validate on full training horizon.** Once short-horizon results are promising, run a complete training session and evaluate on held-out initial conditions. Report success rate, average return, and qualitative behavior.

10. **Document the final solution.** Output the finalized reward function, policy config, and training script with inline comments explaining each design choice and the iterative refinements that led to it.

## Concrete Examples

**Example 1: Designing a Reward Function for Block Stacking**

User: "I have a MuJoCo environment with a Franka arm that needs to stack a red block on a blue block. The policy keeps dropping the block. Help me fix the reward function."

Approach:
1. Read the existing reward function and the environment's observation space
2. Identify that the current reward only has a sparse success signal with no grasp shaping
3. Decompose into sub-rewards: reaching, grasping, lifting, aligning, placing, releasing
4. Add a curriculum: reward reaching first, then grasping (contact + lift height), then alignment over the target block, then placement accuracy
5. Run 20k steps of PPO training, observe reward curve
6. If the robot grasps but doesn't lift, increase lift-height reward weight
7. If it lifts but drops during transport, add a grasp-maintenance bonus

Output:
```python
def compute_reward(obs, action, info):
    ee_pos = obs["robot0_eef_pos"]
    red_pos = obs["red_block_pos"]
    blue_pos = obs["blue_block_pos"]
    grasped = obs["red_block_grasped"]

    # Phase 1: Reach the red block
    reach_dist = np.linalg.norm(ee_pos - red_pos)
    reach_reward = 1.0 - np.tanh(5.0 * reach_dist)

    # Phase 2: Grasp and lift
    lift_height = red_pos[2] - blue_pos[2]
    grasp_reward = 2.0 * float(grasped)
    lift_reward = 3.0 * np.clip(lift_height - 0.04, 0, 0.15) * float(grasped)

    # Phase 3: Align over blue block and place
    xy_dist = np.linalg.norm(red_pos[:2] - blue_pos[:2])
    align_reward = 2.0 * (1.0 - np.tanh(10.0 * xy_dist)) * float(grasped)

    # Phase 4: Successful stack
    stack_height_target = blue_pos[2] + 0.04
    stack_dist = abs(red_pos[2] - stack_height_target)
    stack_reward = 5.0 * float(xy_dist < 0.02 and stack_dist < 0.01)

    # Regularization
    action_penalty = -0.005 * np.linalg.norm(action)

    return reach_reward + grasp_reward + lift_reward + align_reward + stack_reward + action_penalty
```

**Example 2: Self-Correcting a Failed Locomotion Policy**

User: "My humanoid locomotion policy in IsaacGym just falls over immediately. Training reward goes to zero after 1k steps. What's wrong?"

Approach:
1. Inspect the reward function — check for missing balance/uprightness terms
2. Check observation space — verify that joint velocities and torso orientation are included
3. Diagnose: likely missing uprightness reward and/or action penalty is too aggressive
4. Draft a corrected reward with explicit terms: torso height maintenance, forward velocity, uprightness (dot product of torso up-vector with world up), energy penalty
5. Add early termination condition for falls (torso height < threshold) to prevent learning from collapsed states
6. Run 50k steps with PPO, observe if the humanoid maintains balance for at least 20 steps
7. If balance is achieved but no forward motion, increase the forward velocity reward weight

Output:
```python
def compute_reward(obs, action, info):
    torso_height = obs["torso_pos"][2]
    torso_up = obs["torso_up_vec"]  # unit vector
    forward_vel = obs["root_lin_vel"][0]

    # Alive bonus — critical for preventing immediate collapse
    alive_bonus = 2.0 * float(torso_height > 0.7)

    # Uprightness — dot product with world up [0, 0, 1]
    upright_reward = 1.5 * torso_up[2]

    # Forward progress
    velocity_reward = 1.0 * np.clip(forward_vel, -0.5, 2.0)

    # Energy efficiency
    energy_penalty = -0.001 * np.sum(action ** 2)

    # Joint limit penalty
    joint_limit_penalty = -0.1 * info.get("joint_limit_violations", 0)

    return alive_bonus + upright_reward + velocity_reward + energy_penalty + joint_limit_penalty
```

**Example 3: Switching from MLP to Diffusion Policy for Multi-Modal Tasks**

User: "My BC policy for a pouring task has high variance — sometimes it pours correctly, sometimes it misses entirely. The demonstrations show two valid pouring strategies (from the left and from the right)."

Approach:
1. Diagnose: MLP policies with MSE loss average over multi-modal demonstrations, producing invalid intermediate actions
2. Recommend switching to a diffusion policy architecture that can model multi-modal action distributions
3. Generate the diffusion policy config: U-Net denoiser, 100 diffusion steps at training / 10 at inference (DDIM), observation-conditioned
4. Adjust the data pipeline to provide action chunks (sequences of 8-16 future actions) rather than single-step actions
5. Train with the same demonstration data, evaluate both modalities

Output:
```python
diffusion_policy_config = {
    "policy_type": "DiffusionPolicy",
    "obs_encoder": "ResNet18",  # if image observations
    "noise_scheduler": "DDIMScheduler",
    "num_train_diffusion_steps": 100,
    "num_inference_diffusion_steps": 10,
    "action_horizon": 16,       # predict 16-step action chunks
    "observation_horizon": 2,   # condition on 2 past observations
    "unet_dims": [256, 512, 1024],
    "learning_rate": 1e-4,
    "batch_size": 256,
    "num_epochs": 300,
    "ema_decay": 0.999,
}
```

## Best Practices

- **Do:** Start with a short training run (10-50k steps) before committing to a full run. This lets you catch reward pathologies, crashes, and architectural issues cheaply.
- **Do:** Decompose rewards into physically meaningful sub-components with explicit weights. This makes each component independently adjustable during the optimize phase.
- **Do:** Use `np.tanh` or `np.clip` to bound reward terms — unbounded rewards cause training instability and make it hard to balance sub-reward magnitudes.
- **Do:** Log per-component reward values (not just the total) so you can diagnose which term is dominating or underperforming.
- **Do:** Change one thing per iteration. If you adjust both reward weights and the learning rate simultaneously, you cannot attribute any change in performance.
- **Avoid:** Sparse-only rewards for complex multi-step tasks. Without shaping, RL exploration is prohibitively slow for tasks like assembly or tool use.
- **Avoid:** Reward terms that can be exploited without achieving the task goal (e.g., rewarding proximity to an object without requiring a grasp, which leads to the arm hovering near the object indefinitely).
- **Avoid:** Skipping the debug phase — always run the generated code and verify it executes without errors before analyzing training performance.

## Error Handling

| Problem | Diagnosis | Fix |
|---|---|---|
| Training crashes with shape mismatch | Observation or action space dimensions don't match network input/output | Print `env.observation_space` and `env.action_space`, align policy network accordingly |
| Reward is NaN or Inf | Division by zero in distance calculation, or log of zero | Add `max(dist, 1e-8)` guards; use `np.tanh` instead of raw distances |
| Policy converges to zero reward | Reward is too sparse or early termination is too aggressive | Add dense shaping terms; relax termination thresholds during early training |
| Reward hacking (high reward, low success) | Reward function has an exploitable loophole | Inspect rollout videos/trajectories; add constraints or restructure reward to require actual task completion |
| Training oscillates without convergence | Learning rate too high, reward scale too large, or reward components conflicting | Reduce learning rate by 3-10x; normalize reward components to similar scales |
| Diffusion policy generates jittery actions | Insufficient diffusion steps at inference or action horizon too short | Increase inference steps (10→20); increase action chunk length; add temporal smoothing |

## Limitations

- This workflow requires access to a simulator with a step/reset API. It does not apply to real-robot-only settings without sim-to-real transfer.
- The iterative refinement loop depends on fast simulation. Environments with slow physics (e.g., deformable objects, fluids) may make the feedback loop impractically slow.
- Reward function design is task-specific. While the decomposition principles generalize, the specific sub-reward terms must be re-derived for each new task domain.
- The approach works best for tasks with clear, measurable success criteria. Open-ended tasks (e.g., "make the robot move gracefully") are harder to encode as executable reward code.
- Diffusion policies require demonstration data; the pure RL path requires a well-shaped reward. Mixing the two (RL fine-tuning of IL policies) adds complexity that may require manual judgment.
- Self-correction from catastrophic failure typically requires 3-5 iterations; some pathological reward designs may require more fundamental restructuring than incremental adjustment.

## Reference

**EmboCoach-Bench: Benchmarking AI Agents on Developing Embodied Robots**
Lei, Liu, Zhang, Liu, Wen (2026). arXiv:2601.21570
https://arxiv.org/abs/2601.21570v1

Key takeaway: LLM agents using a closed-loop draft-debug-optimize workflow with simulation feedback can autonomously engineer embodied policies that surpass human baselines by 26.5% in success rate. The critical insight is treating executable code as the universal interface and leveraging environment feedback for iterative self-correction.