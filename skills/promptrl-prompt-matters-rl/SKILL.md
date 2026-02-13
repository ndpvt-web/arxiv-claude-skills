---
name: "promptrl-prompt-matters-rl"
description: "Implement PromptRL-style joint prompt-refinement + RL training loops for flow-based image generation. Use when the user mentions 'PromptRL', 'prompt overfitting in RL', 'flow matching RL training', 'joint prompt and model optimization', 'reward-driven prompt rewriting', or 'sample-efficient RL for diffusion/flow models'."
---

# PromptRL: Joint Prompt Refinement with RL for Flow-Based Image Generation

This skill enables Claude to help users implement the PromptRL framework — a technique that couples a trainable language model (LM) prompt-refinement agent directly into the reinforcement learning optimization loop of a flow matching (FM) image generation model. The core insight is that RL-finetuned flow models suffer from prompt overfitting (memorizing specific phrasings and collapsing on paraphrases) and sample inefficiency (low generation diversity). PromptRL solves both by having an LM dynamically rewrite prompts during training, creating a synergistic co-training regime that achieves 2x sample efficiency and state-of-the-art scores (0.97 GenEval, 0.98 OCR, 24.05 PickScore).

## When to Use

- When the user is building or modifying an RL post-training pipeline for a flow matching model (FLUX, SD3, etc.) and wants to reduce prompt overfitting
- When the user needs to implement multi-reward RL training for text-to-image generation (GenEval + OCR + aesthetic scores simultaneously)
- When the user asks about sample-efficient RL for diffusion or flow-based generative models
- When the user wants to add a trainable prompt rewriter to an existing GRPO/REINFORCE training loop
- When the user is fine-tuning FLUX.1-Kontext or similar editing models with reward-based RL and wants to improve EditReward scores
- When the user encounters generation quality collapse after RL training and suspects prompt memorization

## Key Technique

**The Problem.** Standard RL pipelines for flow matching models (e.g., FlowGRPO) train only the denoising network on reward signals. This causes two failures: (1) the model sees the same prompts repeatedly and memorizes surface-level lexical patterns rather than learning semantic understanding — a phenomenon the authors call "prompt linguistic hacking"; (2) as the model improves, generation diversity drops, reducing the effective exploration and making reward-based optimization increasingly sample-inefficient.

**The Solution.** PromptRL introduces an LM (e.g., Qwen2.5-VL-3B) as a prompt refinement agent that runs inside each RL training step. Given a batch of n=8 samples per prompt, the LM rewrites the prompt for n-m of them (typically m=2 kept as originals). Both the LM and FM receive separate GRPO policy gradient updates from the same reward signals, but critically: the LM only gets gradients from its rewritten samples, while the FM gets gradients from all samples. The original-prompt samples serve as baselines for advantage estimation and prevent the FM from becoming over-dependent on the LM's rewrites. The two models are architecturally disjoint — no gradient flows between them — but they co-adapt through shared rewards.

**Multi-Reward Tagging.** Instead of using weighted sums of multiple reward functions (which requires fragile coefficient tuning), PromptRL assigns each training prompt a categorical reward tag. Advantage normalization occurs within each reward category independently, allowing GenEval, OCR, and PickScore objectives to coexist in their native scales without balancing hyperparameters.

## Step-by-Step Workflow

1. **Set up the base models.** Load a pretrained flow matching model (e.g., FLUX.1-dev 12B) as the FM policy and a small instruction-tuned LM (e.g., Qwen2.5-VL-3B-Instruct) as the prompt refinement agent. Initialize both with their pretrained weights; freeze nothing — both are trainable.

2. **Prepare the prompt dataset with reward tags.** Collect training prompts and assign each a reward category tag (e.g., `geneval`, `ocr`, `pickscore`). This eliminates the need for multi-reward coefficient tuning. For single-reward tasks, skip tagging.

3. **Configure the GRPO group structure.** For each prompt, set group size n=8 and prompt retention count m=2. This means 2 samples will use the original prompt verbatim and 6 will use LM-rewritten variants. The m=2 setting is empirically optimal — m=1 causes the single original to always receive negative advantage, collapsing gradients.

4. **Implement the LM prompt rewriting step.** At each training iteration, pass the original prompt to the LM with an instruction template requesting semantically equivalent but stylistically varied reformulations. Require XML-tagged output format and enforce RFormat(p) as a binary constraint. Generate k=n-m rewritten prompts per original.

5. **Run the FM rollouts.** Generate images for all n samples (m original-prompt + (n-m) rewritten-prompt) using the current FM policy. Use 20 inference steps for T2I at 512x512, or 8 steps with 4 SDE steps for editing at 1024x1024.

6. **Compute rewards and group-wise advantages.** Evaluate each generated image with the reward function corresponding to its prompt's tag. Compute advantages within each prompt group using the formula: `A(xi, pi) = [R(xi, pi) - mean_j] / [std_j + eps]`, where mean and std are over the group j that prompt i belongs to.

7. **Update the FM with all-sample gradients.** Apply GRPO policy gradient to the flow matching model using advantages from all n samples (both original and rewritten prompts). Use learning rate 3e-7 and KL coefficient 4e-3.

8. **Update the LM with rewritten-sample-only gradients.** Apply GRPO policy gradient to the LM using advantages from only the n-m rewritten-prompt samples. Use learning rate 1e-6 and KL coefficient 1e-2. This asymmetry prevents the LM from being penalized for the FM's performance on original prompts.

9. **Iterate and evaluate.** Repeat steps 4-8. Monitor GenEval scores, reward convergence, and generation diversity. PromptRL typically reaches competitive performance in ~50% of the rollouts required by flow-only RL.

10. **Deploy with co-trained LM.** At inference time, use the co-trained LM to rewrite user prompts before passing them to the FM. The co-adapted pair outperforms using either component alone (0.97 vs 0.88 GenEval when swapping in an independently trained LM).

## Concrete Examples

**Example 1: Adding PromptRL to an existing FlowGRPO training script**

User: "I have a FlowGRPO training loop for FLUX.1-dev that optimizes GenEval. It works but overfits to my exact prompt phrasing. How do I add PromptRL?"

Approach:
1. Load a small LM (Qwen2.5-VL-3B-Instruct) alongside the existing FM
2. Modify the batch sampling to split each group: 2 originals + 6 LM-rewritten variants
3. Add the LM prompt rewriting call before FM image generation
4. Split the advantage computation: FM gets all 8 advantages, LM gets only the 6 rewritten ones
5. Add a second optimizer for the LM with lr=1e-6, KL coeff=1e-2

Output (pseudocode for the modified training step):
```python
for batch in dataloader:
    prompts_original = batch["prompts"]  # list of original prompts

    # Step 1: LM generates rewritten prompts (n-m=6 per original)
    rewritten_prompts = lm_agent.rewrite(
        prompts_original,
        num_rewrites=6,
        format_constraint="xml_tagged"
    )

    # Step 2: Build full prompt batch (m=2 originals + 6 rewrites per group)
    all_prompts = []
    prompt_sources = []  # track which are original vs rewritten
    for orig, rewrites in zip(prompts_original, rewritten_prompts):
        all_prompts.extend([orig, orig] + rewrites)
        prompt_sources.extend(["orig"] * 2 + ["rewritten"] * 6)

    # Step 3: FM generates images for all prompts
    images = fm_model.generate(all_prompts, num_steps=20, resolution=512)

    # Step 4: Compute rewards and group-wise advantages
    rewards = reward_fn(images, all_prompts)
    advantages = compute_groupwise_advantages(rewards, group_size=8)

    # Step 5: Update FM on ALL samples
    fm_loss = grpo_loss(fm_model, images, all_prompts, advantages)
    fm_optimizer.step(fm_loss)

    # Step 6: Update LM on REWRITTEN samples only
    rewritten_mask = [s == "rewritten" for s in prompt_sources]
    lm_loss = grpo_loss(lm_agent, rewritten_prompts, advantages[rewritten_mask])
    lm_optimizer.step(lm_loss)
```

**Example 2: Multi-reward training with reward tagging**

User: "I want to train FLUX on GenEval, OCR accuracy, and PickScore simultaneously without tuning loss weights."

Approach:
1. Tag each prompt in the dataset with its reward category
2. During advantage computation, normalize within each tag group independently
3. All rewards operate in their native scales — no weighting needed

Output (reward tagging configuration):
```python
# Dataset preparation: assign reward tags
dataset = [
    {"prompt": "a cat sitting on a mat", "reward_tag": "geneval"},
    {"prompt": "a sign that reads HELLO", "reward_tag": "ocr"},
    {"prompt": "beautiful sunset over mountains", "reward_tag": "pickscore"},
    # ... mix of all three types
]

# Reward computation respects tags
def compute_tagged_advantages(rewards, tags, group_size=8):
    advantages = torch.zeros_like(rewards)
    for tag in set(tags):
        mask = [t == tag for t in tags]
        tag_rewards = rewards[mask]
        # Normalize within each reward category
        for group_start in range(0, len(tag_rewards), group_size):
            group = tag_rewards[group_start:group_start + group_size]
            advantages[mask][group_start:group_start + group_size] = (
                (group - group.mean()) / (group.std() + 1e-8)
            )
    return advantages

# Results: GenEval 0.93, OCR 0.96, PickScore 23.94 — all competitive
# without any coefficient tuning
```

**Example 3: Applying PromptRL to image editing (FLUX.1-Kontext)**

User: "I'm RL-finetuning FLUX.1-Kontext for image editing. How do I apply PromptRL to improve EditReward?"

Approach:
1. Use the same joint LM+FM architecture but adapt for editing: the LM rewrites editing instructions rather than generation prompts
2. Use 8 inference steps with 4 SDE steps at 1024x1024
3. Train on OmniEdit dataset (10k samples) with EditReward as the reward function
4. With only 0.06M rollouts, expect EditReward to improve from 1.19 to ~1.43

Output (key configuration differences for editing):
```python
editing_config = {
    "base_model": "FLUX.1-Kontext",
    "resolution": 1024,
    "inference_steps": 8,
    "sde_steps": 4,
    "dataset": "OmniEdit",
    "dataset_size": 10_000,
    "reward_fn": "EditReward",
    "group_size": 8,
    "prompt_retention": 2,  # m=2 originals per group
    "lm_model": "Qwen2.5-VL-3B-Instruct",
    "lm_lr": 1e-6,
    "fm_lr": 3e-7,
    "total_rollouts": 60_000,  # 0.06M — remarkably efficient
}
# Expected: EditReward 1.19 -> 1.43
# Surpasses Gemini 2.5 Flash Image (1.37)
# Comparable to ReasonNet (1.44) which needs multi-stage training
```

## Best Practices

- **Do** keep m=2 original prompts per group of 8. This is the empirically validated sweet spot. Setting m=1 causes consistent negative advantages on the lone original, collapsing its gradient signal.
- **Do** use separate optimizers with different learning rates for the LM (1e-6) and FM (3e-7). The LM needs a higher rate because it starts from a pretrained instruction-following model and adapts quickly.
- **Do** deploy the co-trained LM+FM pair together at inference. Swapping in a different LM drops GenEval from 0.97 to 0.88 due to lost co-adaptation.
- **Do** use reward tagging instead of weighted reward sums when training on multiple objectives. It eliminates hyperparameter search and lets each reward function operate in its native scale.
- **Avoid** propagating gradients between the LM and FM. They must remain architecturally disjoint; the synergy comes from shared reward signals, not backpropagation coupling.
- **Avoid** giving the LM gradients from original-prompt samples. The LM policy gradient should only come from samples where its rewrites were actually used, otherwise it receives noise.

## Error Handling

- **LM generates degenerate rewrites (copies the original verbatim):** Enforce the XML format constraint RFormat(p) and add a minimum edit distance threshold. If the LM reward collapses, increase the KL coefficient for the LM to prevent mode collapse.
- **FM performance degrades on original prompts:** Check that m >= 2. If the FM is over-relying on LM rewrites, temporarily increase m to expose it to more unmodified prompts.
- **Reward hacking across tags:** If one reward category dominates training, verify that advantage normalization is happening per-tag-per-group, not globally. Global normalization defeats the purpose of reward tagging.
- **Training instability with large group sizes:** Group size n=8 is the tested default. Larger groups increase variance in advantage estimation without proportional benefit. Stick to n=8 unless you have a specific reason to change it.
- **Out-of-memory with both LM and FM loaded:** Use mixed precision (bf16) for both models. The LM (3B params) is small relative to the FM (12B), so the overhead is modest — roughly 6GB additional VRAM.

## Limitations

- **Requires a pretrained instruction-following LM.** The prompt refinement agent must already understand how to paraphrase and rewrite instructions. A base LM without instruction tuning will not work.
- **Co-adaptation dependency.** The trained LM and FM are tightly coupled. You cannot freely swap the LM at inference without significant performance degradation (0.97 -> 0.88 GenEval).
- **Tested primarily on FLUX architectures.** The paper validates on FLUX.1-dev and FLUX.1-Kontext. Applicability to other flow matching architectures (SD3, Stable Cascade) is plausible but not empirically confirmed.
- **Reward function quality is the ceiling.** PromptRL makes RL training more sample-efficient and robust, but it cannot exceed what the reward model can distinguish. Poor reward models will still produce poor alignment.
- **Not a replacement for data quality.** The technique is a post-training RL strategy. It assumes a strong pretrained FM and good reward signals. It does not fix deficiencies in the base model's pretraining data.

## Reference

**Paper:** [PromptRL: Prompt Matters in RL for Flow-Based Image Generation](https://arxiv.org/abs/2602.01382v1) (Wang et al., 2026). Focus on Section 3 (Method) for the joint LM+FM GRPO formulation, Section 3.3 for reward tagging, and Table 1 + Table 3 for the prompt retention ablation that motivates m=2.

**Code:** [https://github.com/G-U-N/UniRL](https://github.com/G-U-N/UniRL) — Apache 2.0 licensed implementation with training and inference scripts.