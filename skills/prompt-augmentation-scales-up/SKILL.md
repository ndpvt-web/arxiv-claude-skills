---
name: "prompt-augmentation-scales-up"
description: |
  Apply prompt augmentation to scale up reinforcement learning training for LLM reasoning.
  Implements diverse prompt templates with format-aware rewards to prevent entropy collapse
  during GRPO or similar RL post-training. Use this skill when:
  - "Set up GRPO training with prompt augmentation"
  - "Prevent entropy collapse in RL fine-tuning"
  - "Train a math reasoning model with diverse prompts"
  - "Implement prompt augmentation for reinforcement learning"
  - "Scale RL training beyond 20 epochs without collapse"
  - "Add format rewards to GRPO training"
---

# Prompt Augmentation for Scaled-Up RL Training on Reasoning

This skill enables Claude to implement **prompt augmentation**, a training strategy from Lu et al. (2026) that replaces a single fixed reasoning template with a diverse set of prompt formats during reinforcement learning post-training. By randomly sampling from multiple template categories (DeepSeek-style tags, freeform chain-of-thought, explicit CoT prompting, and reflection-based formats), the technique maintains rollout diversity, prevents the entropy collapse that normally limits GRPO training to 5-20 epochs, and allows stable training for 50+ epochs on a fixed dataset -- all without KL regularization.

## When to Use

- When setting up GRPO (Group Relative Policy Optimization) or similar RL post-training for LLM reasoning tasks
- When training runs collapse after a few epochs due to monotonically decreasing policy entropy
- When a model's reasoning outputs become repetitive and near-deterministic during RL training
- When you want to extend effective training duration on a fixed dataset without collecting more data
- When implementing reward functions that combine correctness verification with format compliance
- When building a training pipeline for math reasoning that needs to match or exceed DeepSeek-R1-style results on a small model

## Key Technique

**The Problem: Entropy Collapse.** Standard GRPO training with a single prompt template causes policy entropy to decrease monotonically. The model converges to a narrow set of reasoning patterns, loses exploratory capacity, and training becomes unstable -- typically collapsing between steps 1,000-2,000. Prior work mitigated this with KL regularization (penalizing divergence from the reference policy), but strong KL constraints suppress the very exploration that drives improvement.

**The Solution: Prompt Augmentation.** Instead of presenting every training question with the same system prompt and format, each question is paired with a uniformly random template drawn from a bank of 13 templates across 4 categories. This forces the model to generate reasoning traces in structurally different formats -- some with `<think>/<answer>` tags, some freeform with `\boxed{}`, some with explicit "Let's think step by step" conditioning, and some with `<solution>/<check>` reflection structure. The structural diversity in rollouts sustains entropy even when the model becomes confident about individual answer strategies. Critically, this eliminates the need for KL regularization entirely, which means the reference model can be offloaded from GPU memory, freeing compute for larger batch sizes.

**Format Rewards Are Essential.** The technique pairs each template with a format-specific reward computed via simple string matching: check for presence and correct count of required tags (e.g., exactly one `<think>` and one `</think>` yields +0.25 each, totaling up to 1.0). Without format rewards, models ignore template instructions and collapse just as fast as single-template baselines. The combined signal of binary correctness reward (0 or 1 from answer verification) plus format reward drives both accurate reasoning and format compliance.

## Step-by-Step Workflow

1. **Define the template bank.** Create 10-15 prompt templates across 4 categories:
   - **Structured tags** (e.g., DeepSeek-style): System prompt instructs `<think>...</think><answer>...</answer>` format. Include teacher-forced variants that prefill the assistant turn with `<think>`.
   - **Freeform**: Instruct step-by-step reasoning with final answer in `\boxed{}`. No structural tags required.
   - **Explicit CoT**: Condition generation on phrases like "Let's think step by step" or "Please reason carefully before answering."
   - **Reflection-based**: Require `<solution>...</solution><check>...</check>` structure for self-verification.

2. **Implement template sampling.** For each training question, uniformly sample one template from the bank. Insert the question into the template's `{question}` placeholder. Each rollout within a group for the same question uses the same template.

3. **Implement format reward functions.** For each template category, define a tag-counting reward:
   - **Structured tags**: +0.25 per required tag present exactly once (e.g., `<think>`, `</think>`, `<answer>`, `</answer>` = max 1.0).
   - **Freeform / Explicit CoT**: Constant 1.0 (no format penalty since no tags are required).
   - **Reflection**: Distribute reward equally across required sections (e.g., 1/3 per section for 3 required tags).
   - Penalize duplicate or missing tags by withholding their portion of the reward.

4. **Implement answer correctness reward.** Extract the final answer from `\boxed{}` (or `<answer>` tags depending on template). Use symbolic verification (SymPy or equivalent) to compare against ground truth. Return binary 0 or 1.

5. **Configure GRPO training without KL regularization.** Set KL coefficient to 0. Use asymmetric clipping (`epsilon_high=0.28`, `epsilon_low=0.20`). Group size of 8 rollouts per question. Learning rate `1e-6` constant schedule.

6. **Compute group-normalized advantages.** For each group of rollouts, normalize rewards: `A_i = (R_i - mean(R)) / std(R)`. The reward `R_i` is the sum (or weighted combination) of correctness and format rewards for rollout `i`.

7. **Train for extended horizons.** With prompt augmentation active, train for 50+ epochs on the fixed dataset. Monitor policy entropy -- it should stabilize rather than monotonically decrease. If entropy drops below 0.05 bits but training remains stable, this is expected and acceptable.

8. **Evaluate with a fixed template.** At inference time, use a single consistent template (typically the DeepSeek-style structured format) across all test questions for reproducible evaluation.

9. **Validate on held-out benchmarks.** Test on diverse math reasoning benchmarks (MATH500, AIME, AMC, Minerva, OlympiadBench) to confirm generalization beyond the training format distribution.

## Concrete Examples

**Example 1: Defining a template bank in Python**

User: "I need to set up prompt augmentation templates for GRPO training on math reasoning."

Approach:
1. Define templates as a list of dicts with system prompt, user template, optional assistant prefix, and format reward function
2. Implement uniform sampling
3. Wire into the data collator

Output:
```python
TEMPLATES = [
    # Category 1: DeepSeek-style structured tags
    {
        "category": "structured",
        "system": "You are a helpful math assistant. Show your reasoning inside <think></think> tags, then give your final answer inside <answer></answer> tags.",
        "user": "{question}",
        "assistant_prefix": "",
        "format_reward_fn": "structured_tag_reward",
    },
    {
        "category": "structured",
        "system": "You are a helpful math assistant. Show your reasoning inside <think></think> tags, then give your final answer inside <answer></answer> tags.",
        "user": "{question}",
        "assistant_prefix": "<think>\n",  # teacher-forced
        "format_reward_fn": "structured_tag_reward",
    },
    # Category 2: Freeform
    {
        "category": "freeform",
        "system": "You are a helpful math assistant. Solve the problem step by step. Put your final answer in \\boxed{}.",
        "user": "{question}",
        "assistant_prefix": "",
        "format_reward_fn": "constant_reward",
    },
    {
        "category": "freeform",
        "system": "Please solve this math problem carefully, showing all work. Enclose your final answer in \\boxed{}.",
        "user": "{question}",
        "assistant_prefix": "",
        "format_reward_fn": "constant_reward",
    },
    # Category 3: Explicit CoT
    {
        "category": "explicit_cot",
        "system": "You are a math tutor. Think through problems rigorously.",
        "user": "{question}\n\nLet's think step by step.",
        "assistant_prefix": "",
        "format_reward_fn": "constant_reward",
    },
    {
        "category": "explicit_cot",
        "system": "You are an expert mathematician.",
        "user": "{question}\n\nPlease reason step by step, then provide your final answer in \\boxed{}.",
        "assistant_prefix": "",
        "format_reward_fn": "constant_reward",
    },
    # Category 4: Reflection-based
    {
        "category": "reflection",
        "system": "Solve the problem in a <solution></solution> block, then verify your work in a <check></check> block. Give your final answer in <answer></answer>.",
        "user": "{question}",
        "assistant_prefix": "",
        "format_reward_fn": "reflection_tag_reward",
    },
    {
        "category": "reflection",
        "system": "Solve the problem in a <solution></solution> block, then verify your work in a <check></check> block. Give your final answer in <answer></answer>.",
        "user": "{question}",
        "assistant_prefix": "<solution>\n",  # teacher-forced
        "format_reward_fn": "reflection_tag_reward",
    },
]

import random

def sample_template(question: str) -> dict:
    """Uniformly sample a template and fill in the question."""
    tmpl = random.choice(TEMPLATES)
    return {
        "system": tmpl["system"],
        "user": tmpl["user"].replace("{question}", question),
        "assistant_prefix": tmpl["assistant_prefix"],
        "format_reward_fn": tmpl["format_reward_fn"],
    }
```

**Example 2: Implementing format reward functions**

User: "How do I write the format reward functions for prompt augmentation?"

Approach:
1. Count required tags using simple string matching
2. Verify each tag appears exactly once
3. Assign partial credit proportional to tag compliance

Output:
```python
def structured_tag_reward(response: str) -> float:
    """Reward for DeepSeek-style <think>/<answer> format."""
    score = 0.0
    for tag in ["<think>", "</think>", "<answer>", "</answer>"]:
        if response.count(tag) == 1:
            score += 0.25
    return score

def reflection_tag_reward(response: str) -> float:
    """Reward for <solution>/<check>/<answer> reflection format."""
    score = 0.0
    for tag_pair in [("<solution>", "</solution>"),
                     ("<check>", "</check>"),
                     ("<answer>", "</answer>")]:
        if response.count(tag_pair[0]) == 1 and response.count(tag_pair[1]) == 1:
            score += 1.0 / 3.0
    return score

def constant_reward(response: str) -> float:
    """Freeform and explicit CoT templates have no format penalty."""
    return 1.0

REWARD_FN_MAP = {
    "structured_tag_reward": structured_tag_reward,
    "reflection_tag_reward": reflection_tag_reward,
    "constant_reward": constant_reward,
}

def compute_reward(response: str, answer: str, format_reward_fn: str) -> float:
    """Combined reward: binary correctness + format compliance."""
    correctness = 1.0 if verify_answer(response, answer) else 0.0
    format_score = REWARD_FN_MAP[format_reward_fn](response)
    return correctness + format_score  # range [0, 2]
```

**Example 3: Configuring GRPO training hyperparameters**

User: "What training config should I use for prompt augmentation GRPO?"

Approach:
1. Set KL coefficient to 0 (prompt augmentation replaces KL regularization)
2. Use asymmetric clipping for stability
3. Configure batch and group sizes

Output:
```python
training_config = {
    # Core GRPO settings
    "kl_coeff": 0.0,             # No KL penalty -- prompt augmentation handles entropy
    "clip_eps_high": 0.28,       # Asymmetric clipping upper bound
    "clip_eps_low": 0.20,        # Asymmetric clipping lower bound
    "group_size": 8,             # Rollouts per question per step

    # Batch sizes
    "batch_size": 128,           # Prompts per batch
    "mini_batch_size": 32,       # Mini-batch for gradient accumulation

    # Sequence lengths
    "max_prompt_tokens": 1024,
    "max_response_tokens": 3072,

    # Optimizer
    "learning_rate": 1e-6,
    "lr_schedule": "constant",   # No warmup or decay needed

    # Training duration (prompt augmentation enables long training)
    "num_epochs": 50,            # Can go 50+ without collapse

    # Prompt augmentation
    "num_templates": 13,
    "template_sampling": "uniform",
    "format_reward_weight": 1.0,
    "correctness_reward_weight": 1.0,
}
```

## Best Practices

- **Do:** Include teacher-forced variants (prefilling assistant turn with opening tags) for structured templates. This improves convergence speed for tag-based formats.
- **Do:** Use simple string matching for format rewards -- exact tag counting with `str.count()` is sufficient. Complex regex is unnecessary and can introduce false negatives.
- **Do:** Keep template categories balanced. Having roughly equal representation across structured, freeform, CoT, and reflection categories maximizes rollout diversity.
- **Do:** Monitor policy entropy during training. A stable (even if low) entropy curve confirms prompt augmentation is working. A monotonically decreasing curve signals a problem.
- **Avoid:** Adding KL regularization on top of prompt augmentation. The paper shows KL constraints actively harm performance by suppressing exploration. Set `kl_coeff=0`.
- **Avoid:** Using fewer than 8-10 templates. Insufficient diversity fails to prevent entropy collapse. The sweet spot in the paper is 13 templates across 4 categories.
- **Avoid:** Removing format rewards. Without format-specific rewards, the model ignores template instructions and collapses by step ~1,500 regardless of template count.

## Error Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Entropy drops to ~0 within 1,500 steps | Format rewards missing or misconfigured | Verify format reward functions return non-zero for compliant outputs |
| Model ignores template tags entirely | No format reward signal; model optimizes only for correctness | Add or increase format reward weight |
| Training loss spikes after epoch 30+ | Learning rate too high for long training | Reduce to `1e-6` or lower; verify asymmetric clipping bounds |
| All rollouts in a group produce identical outputs | Group size too small or template not varying per batch | Confirm template is sampled per-question and group size >= 8 |
| Answer extraction fails for some templates | Extraction logic doesn't handle all template formats | Implement per-category answer extraction: `\boxed{}` for freeform/CoT, `<answer>` tags for structured/reflection |

## Limitations

- **Domain specificity.** The technique is validated on mathematical reasoning where answers are verifiable. Extending to open-ended generation (creative writing, summarization) requires designing alternative correctness rewards.
- **Compute requirements.** Generating 8 rollouts per question per step across 50+ epochs is GPU-intensive. The paper used 8x L40S GPUs for a 1.5B parameter model.
- **Template design is manual.** The 13 templates were hand-crafted. There is no automated template generation or optimization -- the quality of the template bank matters.
- **Fixed dataset assumption.** The technique scales training duration on a fixed dataset. It does not address data scaling; combining prompt augmentation with curriculum learning or data augmentation is unexplored.
- **Small model validation only.** Results are demonstrated on Qwen2.5-Math-1.5B. Scaling behavior to 7B+ models is not established in this paper.

## Reference

**Paper:** Lu, Huang, & Balestriero (2026). *Prompt Augmentation Scales up GRPO Training on Mathematical Reasoning.* [arXiv:2602.03190v2](https://arxiv.org/abs/2602.03190v2)

Look for: Table 1 (template categories and format rewards), Figure 2 (entropy curves with/without augmentation), Appendix B (all 13 template definitions), and the ablation in Section 4.3 showing format rewards are non-negotiable.

**Code:** [github.com/wenquanlu/prompt-augmentation-GRPO](https://github.com/wenquanlu/prompt-augmentation-GRPO) -- built on the VERL framework.