---
name: "medsam-agent-empowering-interactive-medical"
description: |
  Build multi-turn agentic loops where a language model iteratively refines tool calls based on visual feedback, using the MedSAM-Agent paradigm of hybrid prompting, process rewards, and interaction parsimony. Applicable to any domain where an LLM orchestrates an external tool through sequential observe-act-refine cycles.
  Trigger phrases:
  - "build an agent that iteratively refines segmentation"
  - "multi-turn tool-use loop with visual feedback"
  - "implement process rewards for agentic RL training"
  - "create an interactive segmentation agent"
  - "design a reward function that penalizes redundant actions"
  - "agent that decides when to stop refining"
---

# MedSAM-Agent: Multi-Turn Agentic Tool Orchestration with Process Rewards

This skill teaches Claude to implement the **MedSAM-Agent** pattern: a multi-turn agentic framework where a language model autonomously orchestrates an external tool (like a segmentation model) through iterative observe-act-refine cycles. The core innovation is a hybrid prompting strategy for generating expert trajectories, combined with a two-stage training pipeline (SFT then GRPO reinforcement learning) that uses clinical-fidelity process rewards to enforce both quality and interaction parsimony — the agent learns not just *what* to do but *when to stop*. This pattern generalizes beyond medical imaging to any domain requiring an LLM to make sequential tool calls with visual or structured feedback.

## When to Use

- When building an agent that calls an external tool (segmentation model, image editor, code executor) multiple times and must decide when the output is good enough to stop
- When implementing reinforcement learning reward functions that need to balance output quality against action efficiency (fewer steps = better)
- When designing expert trajectory datasets for SFT cold-start of tool-using agents, combining multiple interaction paradigms (e.g., coarse-then-fine)
- When the user asks to create an agentic loop where an MLLM receives visual feedback (updated mask, rendered output) and generates the next tool call
- When implementing GRPO (Group Relative Policy Optimization) training for multi-turn tool-use agents
- When designing a process reward that penalizes overshoot (quality degradation after peak) and redundant iterations

## Key Technique

**The Problem with Single-Turn Tool Use.** Prior approaches (like SegAgent or Seg-R1) treat tool orchestration as a single-shot prediction: the LLM generates one prompt, the tool runs once, done. This fails because real-world tools benefit from iterative refinement — a bounding box gets you 70% of the way, then targeted point clicks fix the remaining errors. Single-turn agents cannot exploit this iterative potential.

**Hybrid Prompting + Multi-Turn Trajectories.** MedSAM-Agent defines an action space of three primitives: bounding boxes `[x1, y1, x2, y2]` for coarse localization, point clicks `[x, y, +/-1]` for targeted refinement, and a `stop_action` to terminate. Expert trajectories are generated using two complementary workflows: (1) **Box-to-Point** — start with a jittered bounding box, identify the largest false-negative/false-positive regions via distance transform, and place corrective clicks at their centroids; (2) **Sequential-Click** — point-only refinement without any box. Each action must improve IoU by at least threshold `tau`, enforcing that every step in the trajectory is meaningful. This hybrid data trains the agent to flexibly choose between coarse and fine strategies.

**Process Reward for Interaction Parsimony.** The reward function has four components: (1) `R_fmt` — format adherence (did the agent use valid tool calls and eventually stop?); (2) `R_qual` — weighted IoU + Dice of the final mask; (3) `R_imp` — progressive improvement bonus summing `max(0, IoU_t - IoU_{t-1})` across turns; (4) `R_over` — overshoot penalty `IoU_max - IoU_final` preventing quality degradation; plus (5) `R_cost` — linear penalty proportional to the number of turns. This composite reward teaches the agent that unnecessary actions are punished even if the final output is good. The total reward is: `R = w1*R_fmt + w2*clip(R_qual + lambda1*R_imp - lambda2*R_over - lambda3*R_cost, 0, 1)`.

## Step-by-Step Workflow

1. **Define the action space.** Enumerate the discrete tool-call primitives your agent can issue. Each action must include structured parameters (e.g., coordinates, labels) formatted inside `<tool_call>` tags that the tool API can parse. Always include an explicit `stop_action` so the agent can terminate.

2. **Define the observation loop.** After each tool call, capture the tool's output (e.g., a predicted mask), overlay it on the original input as a visual prompt, and re-encode it through the vision encoder. The agent's state at turn `t` is the full history: `s_t = {(a_0, o_0), ..., (a_{t-1}, o_{t-1})}`.

3. **Generate expert trajectories using the hybrid strategy.** Create two trajectory generators:
   - *Coarse-to-fine*: Start with a bounding box (add random jitter of 5-20px to simulate human imprecision), then iteratively place corrective points at centroids of the largest error regions found via distance transform on the error mask.
   - *Fine-only*: Use only point clicks from scratch, selecting the most impactful click location at each step.
   Apply progress-constrained sampling: discard any action where `delta_IoU < tau` (typically 0.01-0.02). Retry up to N=5 times with different random seeds before abandoning.

4. **Format trajectories as multi-turn conversations.** Each trajectory becomes a conversation where the user turn contains the image + task description, and assistant turns contain `<tool_call>` actions followed by `<tool_response>` observations. Cap trajectories at 5 turns maximum.

5. **Stage 1: SFT cold-start.** Fine-tune the base MLLM on the trajectory dataset using standard next-token prediction loss. This teaches the model the action format, domain-specific vocabulary, and basic decision heuristics. Use a large dataset (MedSAM-Agent used 449K samples).

6. **Stage 2: RL with process rewards (GRPO).** Select a smaller subset of challenging samples requiring 3-5 interaction rounds. For each sample, generate G rollouts (groups), compute per-rollout rewards using the composite formula, then optimize using Group Relative Policy Optimization. The group-relative baseline removes the need for a separate value network.

7. **Implement the composite reward function.** Code four components:
   - `R_fmt`: Binary checks — 0.5 for using at least one tool call, 0.5 for ending with `stop_action`.
   - `R_qual`: `w_iou * IoU_final + w_dice * Dice_final` (or domain-appropriate quality metric).
   - `R_imp`: `sum(max(0, metric_t - metric_{t-1}))` across all turns — rewards monotonic improvement.
   - `R_over`: `metric_max - metric_final` — penalizes the agent if it peaked and then degraded.
   - `R_cost`: `lambda * num_turns / max_turns` — linear penalty for interaction length.

8. **Implement the inference loop.** At test time, run the agent in a loop: generate an action, execute the tool, overlay the result, feed back to the agent. Stop when the agent outputs `stop_action` or hits the maximum turn limit.

9. **Evaluate with both quality and efficiency metrics.** Report the final output quality (IoU, Dice, or domain metric) AND the average number of turns used. A good agent achieves high quality in fewer turns — MedSAM-Agent averaged 2.11 turns vs 4.34 for the base model.

10. **Test tool agnosticism.** Verify that the trained agent transfers to different tool backends without retraining. MedSAM-Agent trained on MedSAM2 transferred to SAM2.1 and IMISNet with negligible degradation, confirming the agent learns *strategy*, not tool-specific quirks.

## Concrete Examples

**Example 1: Building the Multi-Turn Agent Loop**

User: "I want to build an agent that iteratively refines a segmentation mask by choosing between bounding boxes and point clicks."

Approach:
1. Define the action schema with three primitives:
```python
ACTION_SCHEMA = {
    "box": {"params": ["x1", "y1", "x2", "y2"], "description": "Bounding box prompt"},
    "point": {"params": ["x", "y", "label"], "description": "Point click (+1=foreground, -1=background)"},
    "stop": {"params": [], "description": "Terminate interaction"}
}
```

2. Implement the agent loop:
```python
def agent_loop(model, seg_tool, image, task_description, max_turns=5):
    state = {"image": image, "task": task_description, "history": []}
    mask = None

    for t in range(max_turns):
        # Compose observation: overlay current mask on image
        if mask is not None:
            observation = overlay_mask_on_image(image, mask)
        else:
            observation = image

        # Agent decides next action
        action = model.generate_action(observation, state["history"], task_description)

        if action["type"] == "stop":
            break

        # Execute tool call
        mask = seg_tool.predict(image, action, previous_prompts=state["history"])
        state["history"].append({"action": action, "mask_iou": compute_iou(mask, gt)})

    return mask, len(state["history"])
```

3. The agent sees the updated mask at each step and decides whether to add a corrective point, try a new box, or stop.

Output: An agent that averages 2-3 turns for typical cases, issuing a box first then 1-2 corrective points before stopping.

---

**Example 2: Implementing the Composite Process Reward**

User: "Design a reward function that penalizes my agent for taking unnecessary actions even when the final result is correct."

Approach:
1. Track per-turn metrics during rollout
2. Compute each reward component separately
3. Combine with clipping

```python
def compute_reward(turn_metrics, final_iou, final_dice, num_turns, max_turns=5,
                   w_iou=0.5, w_dice=0.5, lam_imp=0.3, lam_over=0.2, lam_cost=0.1):
    # Format reward: did the agent use tools and stop properly?
    r_fmt = 0.0
    if num_turns >= 1:
        r_fmt += 0.5  # Used at least one tool call
    if turn_metrics[-1].get("stopped", False):
        r_fmt += 0.5  # Ended with stop_action

    # Quality reward
    r_qual = w_iou * final_iou + w_dice * final_dice

    # Progressive improvement: reward monotonic gains
    r_imp = sum(
        max(0, turn_metrics[t]["iou"] - turn_metrics[t - 1]["iou"])
        for t in range(1, len(turn_metrics))
    )

    # Overshoot penalty: did the agent peak then degrade?
    max_iou = max(m["iou"] for m in turn_metrics)
    r_over = max_iou - final_iou

    # Cost penalty: fewer turns = better
    r_cost = num_turns / max_turns

    # Composite
    r_total = r_fmt + max(0, min(1, r_qual + lam_imp * r_imp - lam_over * r_over - lam_cost * r_cost))
    return r_total
```

Output: An agent trained with this reward learns to stop at turn 2 when IoU plateaus, instead of blindly exhausting all 5 turns.

---

**Example 3: Generating Hybrid Expert Trajectories**

User: "I need to create training data for my segmentation agent with diverse interaction strategies."

Approach:
1. Implement the box-to-point trajectory generator:
```python
def generate_box_to_point_trajectory(image, gt_mask, seg_tool, tau=0.02, max_retries=5):
    trajectory = []
    # Step 1: Jittered bounding box
    bbox = get_bounding_box(gt_mask)
    bbox = jitter_box(bbox, noise_px=random.randint(5, 20))
    mask = seg_tool.predict(image, box=bbox)
    trajectory.append({"action": "box", "params": bbox, "iou": iou(mask, gt_mask)})

    # Steps 2-N: Corrective point clicks
    for _ in range(4):  # max 4 additional clicks
        error_mask = compute_error(mask, gt_mask)
        if error_mask.sum() == 0:
            break
        # Find largest error region centroid via distance transform
        click_point, click_label = find_best_click(error_mask, gt_mask)
        new_mask = seg_tool.predict(image, points=[(click_point, click_label)], prev_mask=mask)
        delta = iou(new_mask, gt_mask) - iou(mask, gt_mask)
        if delta < tau:
            continue  # Skip non-improving actions
        mask = new_mask
        trajectory.append({"action": "point", "params": (*click_point, click_label), "iou": iou(mask, gt_mask)})

    trajectory.append({"action": "stop", "params": []})
    return trajectory
```

2. Generate sequential-click trajectories (no box) as a complementary paradigm.
3. Mix both at ~50/50 ratio in the SFT dataset.

Output: A dataset of 449K multi-turn trajectories where each action provably improves the result, teaching the agent efficient human-like refinement strategies.

## Best Practices

- **Do:** Always include an explicit `stop_action` in your action space. Without it, the agent has no way to signal confidence and will exhaust all turns.
- **Do:** Apply progress-constrained sampling when generating trajectories — discard any action that does not improve the metric by at least `tau`. This ensures every training example teaches meaningful decisions.
- **Do:** Use jittered/noisy initial prompts during trajectory generation (e.g., perturbed bounding boxes). This makes the agent robust to imprecise inputs at test time.
- **Do:** Track and report average number of turns alongside quality metrics. An agent that matches accuracy in 2 turns is strictly better than one needing 5 turns.
- **Avoid:** Training only with outcome rewards (final quality). Without process rewards, the agent learns to spam maximum turns regardless of whether they help.
- **Avoid:** Using only one interaction paradigm (e.g., box-only). The hybrid strategy of coarse + fine trajectories is critical — ablations show box-only reaches 0.649 IoU vs 0.686 for hybrid.

## Error Handling

- **Agent generates invalid action format:** Enforce structured output with `<tool_call>` tags and validate JSON before executing. If malformed, return a format error as the observation and let the agent retry (count toward turn budget).
- **Tool call returns degenerate output** (empty mask, all-zero prediction): Overlay the empty result as observation. A well-trained agent will recognize the failure and issue a corrective action. Set a minimum-quality threshold below which the turn is treated as a no-op.
- **Agent fails to stop** (exhausts max turns): Return the mask from the turn with the highest metric, not necessarily the final turn. The overshoot penalty `R_over` in training specifically addresses this failure mode.
- **IoU degrades after a refinement step:** This is the overshoot scenario. The `R_over` reward component penalizes this. At inference, consider a rollback heuristic: if `metric_t < metric_{t-1}`, revert to the previous mask.
- **SFT cold-start produces poor initial policy:** The RL stage (GRPO) cannot recover from an arbitrarily bad initial policy. Ensure the SFT dataset is large enough (hundreds of thousands of trajectories) and covers diverse interaction strategies.

## Limitations

- **Requires a differentiable or callable tool API.** The agent loop assumes the segmentation tool can be called programmatically with structured prompts. Purely GUI-based tools need an API wrapper.
- **Process reward requires ground truth at training time.** The IoU-based progressive improvement and overshoot rewards need per-turn ground truth labels. This limits applicability to domains where labeled data is available for training (though inference is label-free).
- **Maximum turns cap is a hyperparameter.** Setting it too low (1-2) eliminates the benefit of multi-turn refinement; too high (10+) increases training cost and may encourage verbosity. The paper found 5 turns optimal.
- **Computational cost of RL training.** GRPO with multi-turn rollouts requires significant compute (8x H20 GPUs in the paper). The SFT-only variant achieves ~97% of full performance and is much cheaper.
- **Tool agnosticism has limits.** While the agent transfers between SAM variants, it assumes the tool accepts the same prompt format (boxes and points). Fundamentally different tool APIs require re-generating trajectories.

## Reference

- **Paper:** [MedSAM-Agent: Empowering Interactive Medical Image Segmentation with Multi-turn Agentic Reinforcement Learning](https://arxiv.org/abs/2602.03320v1) — Look for Section 3 (Method) for the hybrid prompting strategy, Section 3.3 for the process reward formulas, and Table 3 for ablation results showing the contribution of each reward component.
- **Code:** [github.com/CUHK-AIM-Group/MedSAM-Agent](https://github.com/CUHK-AIM-Group/MedSAM-Agent) — Reference implementation using Qwen3-VL-8B + MedSAM2, with verl-based GRPO training.