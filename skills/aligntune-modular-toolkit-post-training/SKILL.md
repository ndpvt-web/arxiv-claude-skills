---
name: "aligntune-modular-toolkit-post-training"
description: >
  Set up and run reproducible LLM post-training alignment pipelines using AlignTune's unified
  factory API. Covers SFT, DPO, PPO, GRPO, and 10+ RL algorithms with interchangeable TRL/Unsloth
  backends, composite reward functions, and integrated evaluation.
  Trigger phrases: "align a model with RLHF", "fine-tune with DPO", "set up SFT training pipeline",
  "configure GRPO reward functions", "compare TRL vs Unsloth backends", "run post-training evaluation"
---

# AlignTune: Modular Post-Training Alignment Pipelines

This skill enables Claude to scaffold, configure, and debug LLM post-training alignment workflows
using the AlignTune toolkit. AlignTune exposes a single factory boundary (`create_sft_trainer` /
`create_rl_trainer`) that abstracts away backend-specific logic for TRL and Unsloth, letting you
switch backends, swap RL algorithms (DPO, PPO, GRPO, GSPO, DAPO, Dr. GRPO, GBMPO, Counterfactual
GRPO, PACE), compose reward signals, and evaluate results -- all without rewriting training code.

## When to Use

- When the user wants to fine-tune an LLM with supervised fine-tuning (SFT) and needs a clean training script
- When the user asks to align a model using preference optimization (DPO, PPO, GRPO, or variants)
- When the user needs to compare training performance across TRL and Unsloth backends on the same config
- When the user wants to compose multiple reward signals (rule-based + learned) for RL training
- When the user needs pre/post-training evaluation with metrics like math accuracy, ROUGE, BLEU, win rate, or KL divergence
- When the user is debugging backend interference issues (Unsloth patching conflicting with TRL)
- When the user wants a reproducible alignment experiment with controlled configuration and seed management

## Key Technique

AlignTune solves three problems identified in alignment research: **backend interference** (Unsloth's
monkey-patching breaks TRL when both are installed), **reward fragmentation** (ad-hoc reward functions
scattered across scripts), and **irreproducible pipelines** (config drift between experiments).

The core architectural insight is the **factory boundary pattern**. Both `create_sft_trainer()` and
`create_rl_trainer()` accept identical high-level parameters (model name, dataset, backend, hyperparameters)
and return a trainer object with a uniform `.train()` / `.evaluate()` interface. Internally, the factory
manages environment variables (`PURE_TRL_MODE`, `DISABLE_UNSLOTH_FOR_TRL`) to prevent backend
cross-contamination, uses lazy imports for Unsloth to avoid premature patching, and provides automatic
fallback (Unsloth -> TRL) when the preferred backend is unavailable. This means switching from
`backend="unsloth"` to `backend="trl"` is a one-parameter change with no other code modifications.

The **reward layer** supports weighted composition of named reward functions (e.g., `"math_correctness"`
with weight 1.0, `"math_reasoning"` with weight 0.5, or `"coherence"` / `"safety"` / `"length"` for
preference tasks). Evaluation uses `EvalConfig` + `run_eval()` with task-type-aware metric selection,
supporting LoRA adapter merging, vLLM acceleration, and reference model comparisons out of the box.

## Step-by-Step Workflow

1. **Install AlignTune** from source with `git clone https://github.com/Lexsi-Labs/aligntune.git && cd aligntune && pip install -e .` -- requires Python 3.12+, PyTorch 2.0+, and a CUDA GPU.

2. **Choose the training paradigm** based on available data: use SFT (`create_sft_trainer`) for instruction-response pairs; use RL (`create_rl_trainer`) with DPO for preference pairs (chosen/rejected), GRPO/PPO for reward-signal-driven optimization, or GSPO/DAPO for advanced policy optimization.

3. **Select the backend explicitly** rather than relying on `"auto"`. Set `backend="trl"` for full algorithm coverage and battle-tested stability, or `backend="unsloth"` for 2x memory efficiency and faster training on supported algorithms. Note: GBMPO is TRL-only.

4. **Configure the trainer** by calling the factory function with model name, dataset, hyperparameters, and LoRA settings. Pass algorithm-specific parameters as keyword arguments (e.g., `boost_factor` for Counterfactual GRPO, `gbmpo_divergence_type` for GBMPO).

5. **Define reward functions** for RL training by passing a list of reward configurations with names and weights. Use domain-appropriate rewards: `"math_correctness"` + `"math_reasoning"` for math tasks, `"coherence"` + `"safety"` + `"length"` for general alignment, or custom reward functions via the extensible reward layer.

6. **Run pre-training evaluation** using `EvalConfig` and `run_eval()` to establish baseline metrics before training begins. Set `task_type` to match your domain (math, code, text) for automatic metric selection.

7. **Execute training** with `trainer.train()`. Monitor with standard HuggingFace callbacks. The factory handles gradient checkpointing, mixed precision (bf16/fp16), and QLoRA quantization configuration.

8. **Run post-training evaluation** with the same `EvalConfig` pointing at the output adapter path. Compare against the baseline to compute absolute improvement.

9. **Save and compare results** to JSON. When running backend comparisons, keep all parameters identical except `backend=` and compare metrics side-by-side.

10. **Export the trained adapter** for deployment. The output directory contains the LoRA adapter weights, tokenizer, and training configuration for reproducibility.

## Concrete Examples

**Example 1: SFT on instruction-following data with TRL**

User: "Fine-tune Gemma 7B on the Dolly dataset for instruction following"

Approach:
1. Import `create_sft_trainer` from `aligntune.core.backend_factory`
2. Configure with LoRA (r=16, alpha=32) and ChatML formatting
3. Run evaluation before and after training

```python
from aligntune.core.backend_factory import create_sft_trainer
from aligntune.eval.runner import EvalConfig, run_eval

# Baseline evaluation
baseline_config = EvalConfig(
    model_path="google/gemma-7b",
    task_type="text",
    metrics=["perplexity", "rouge", "bleu"],
    dataset_name="philschmid/dolly-15k-oai-style",
    split="test",
    batch_size=8,
    max_samples=200
)
baseline = run_eval(baseline_config)

# Training
trainer = create_sft_trainer(
    model_name="google/gemma-7b",
    dataset_name="philschmid/dolly-15k-oai-style",
    backend="trl",
    output_dir="./output/gemma-sft",
    num_epochs=3,
    batch_size=4,
    learning_rate=2e-4,
    max_seq_length=512,
    lora_r=16,
    lora_alpha=32,
    load_in_4bit=True,
    bf16=True
)
trainer.train()

# Post-training evaluation
post_config = EvalConfig(
    model_path="./output/gemma-sft",
    base_model="google/gemma-7b",
    use_lora=True,
    task_type="text",
    metrics=["perplexity", "rouge", "bleu"],
    dataset_name="philschmid/dolly-15k-oai-style",
    split="test",
    batch_size=8,
    max_samples=200
)
post = run_eval(post_config)
```

Output: JSON file with baseline vs post-training perplexity, ROUGE, and BLEU scores.

---

**Example 2: GRPO for math reasoning with composite rewards**

User: "Train Llama 3.2 3B on GSM8K with GRPO using math correctness and reasoning rewards"

Approach:
1. Import `create_rl_trainer` and evaluation utilities
2. Configure GRPO with two weighted reward signals
3. Use group size of 4 generations per prompt for relative ranking

```python
from aligntune.core.backend_factory import create_rl_trainer
from aligntune.eval.runner import EvalConfig, run_eval

# Pre-training eval
eval_cfg = EvalConfig(
    model_path="meta-llama/Llama-3.2-3B-Instruct",
    task_type="math",
    metrics=["math_accuracy"],
    dataset_name="openai/gsm8k",
    split="test",
    batch_size=8,
    temperature=0.1,
    max_samples=100
)
baseline = run_eval(eval_cfg)

# GRPO training with composite reward
trainer = create_rl_trainer(
    model_name="meta-llama/Llama-3.2-3B-Instruct",
    dataset_name="openai/gsm8k",
    algorithm="grpo",
    backend="trl",
    output_dir="./output/llama-grpo-math",
    max_steps=200,
    max_samples=1000,
    batch_size=8,
    gradient_accumulation_steps=2,
    num_generations=4,
    learning_rate=5e-6,
    lr_scheduler_type="cosine",
    max_seq_length=1024,
    bf16=True,
    lora_r=16,
    lora_alpha=32,
    reward_funcs=[
        {"name": "math_correctness", "weight": 1.0},
        {"name": "math_reasoning", "weight": 0.5}
    ]
)
trainer.train()

# Post-training eval
eval_cfg.model_path = "./output/llama-grpo-math"
eval_cfg.base_model = "meta-llama/Llama-3.2-3B-Instruct"
eval_cfg.use_lora = True
post = run_eval(eval_cfg)
print(f"Accuracy gain: {post['math_accuracy'] - baseline['math_accuracy']:.2%}")
```

Output: Pre/post math accuracy comparison with absolute gain percentage.

---

**Example 3: DPO alignment with safety and coherence rewards**

User: "Align Gemma 2 2B on Anthropic HH-RLHF using DPO with safety and coherence rewards"

Approach:
1. Use `create_rl_trainer` with `algorithm="dpo"` and preference pair dataset
2. Compose three reward signals with different weights
3. Evaluate with DPO-specific metrics (win rate, reward margin)

```python
from aligntune.core.backend_factory import create_rl_trainer
from aligntune.eval.runner import EvalConfig, run_eval

trainer = create_rl_trainer(
    model_name="google/gemma-2-2b-it",
    dataset_name="Anthropic/hh-rlhf",
    algorithm="dpo",
    backend="trl",
    output_dir="./output/gemma-dpo-safety",
    num_epochs=1,
    batch_size=4,
    learning_rate=5e-6,
    max_seq_length=512,
    load_in_4bit=True,
    fp16=True,
    reward_funcs=[
        {"name": "coherence", "weight": 0.4},
        {"name": "safety", "weight": 0.4},
        {"name": "length", "weight": 0.2, "params": {"min_tokens": 20, "max_tokens": 400}}
    ]
)
trainer.train()

# DPO-specific evaluation
eval_cfg = EvalConfig(
    model_path="./output/gemma-dpo-safety",
    base_model="google/gemma-2-2b-it",
    reference_model_path="google/gemma-2-2b-it",
    use_lora=True,
    task_type="text",
    metrics=["kl_divergence", "win_rate", "reward_margin"],
    dataset_name="Anthropic/hh-rlhf",
    split="test",
    batch_size=8,
    max_samples=200
)
results = run_eval(eval_cfg)
```

Output: Win rate, reward margin, and KL divergence metrics showing alignment improvement.

## Best Practices

**Do:**
- Always set `backend` explicitly (`"trl"` or `"unsloth"`) instead of `"auto"` in production scripts to ensure reproducibility across environments.
- Run pre-training evaluation with the same `EvalConfig` as post-training to get a valid before/after comparison.
- Use `load_in_4bit=True` with LoRA for large models (7B+) to fit within single-GPU memory.
- Set a fixed `seed` in both training and evaluation configs for reproducible experiments.
- Compose reward functions with explicit weights that sum to a meaningful scale -- heavier weights on primary objectives, lighter on auxiliary signals.

**Avoid:**
- Do not mix Unsloth and TRL imports manually -- let the factory manage environment variables and import order to prevent backend interference.
- Do not set `max_steps` and `num_epochs` simultaneously without understanding precedence -- `max_steps > 0` overrides epoch-based training.
- Do not use GBMPO with the Unsloth backend -- it is TRL-only and will fail silently or error.
- Do not skip pre-training evaluation -- without a baseline, accuracy gains are meaningless.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| `ImportError: unsloth not found` | Unsloth not installed or incompatible CUDA | Install unsloth separately, or switch to `backend="trl"` |
| Backend interference (corrupted outputs) | Unsloth monkey-patches applied before TRL init | Set `backend="trl"` explicitly; the factory sets `PURE_TRL_MODE=1` |
| OOM during RL training | Large model + multiple generations per prompt | Reduce `num_generations`, enable `load_in_4bit`, lower `batch_size` |
| Reward function not found | Misspelled reward name | Check available rewards in `aligntune.rewards` registry; use exact names |
| Evaluation metrics all zero | Wrong `task_type` for dataset domain | Match `task_type` to data: `"math"` for GSM8K, `"text"` for general, `"code"` for coding |
| LoRA adapter not loading | Missing `base_model` in EvalConfig | Always set `base_model` when `use_lora=True` in evaluation |
| Fallback backend selected unexpectedly | `backend="auto"` chose differently than expected | Set backend explicitly and check availability with `BackendFactory._check_backend_availability()` |

## Limitations

- **GPU required**: AlignTune targets CUDA GPUs. CPU-only training is technically possible for tiny models but impractical.
- **Algorithm coverage varies by backend**: Unsloth supports most algorithms but not GBMPO. Check the compatibility matrix before choosing a backend.
- **No multi-node training**: The factory API is single-node. For distributed training across multiple machines, you need to wrap the trainer with DeepSpeed or FSDP manually.
- **Reward functions are predefined**: While the reward layer is extensible, adding custom reward functions requires writing Python code in the AlignTune reward registry -- there is no YAML-only reward definition.
- **Dataset format assumptions**: SFT expects ChatML/instruction-response format; DPO expects chosen/rejected pairs; GRPO expects prompt + verifiable answers. Format mismatches produce silent training failures.
- **Python 3.12+ only**: Does not support older Python versions.

## Reference

**Paper**: [AlignTune: Modular Toolkit for Post-Training Alignment of Large Language Models](https://arxiv.org/abs/2602.09621v2) -- Read Section 3 (Architecture) for the factory boundary pattern and Section 4 (Reward Layer) for composite reward design. The key insight is that isolating backend-specific logic behind a single factory function eliminates cross-contamination and enables controlled A/B comparisons.

**Repository**: [github.com/Lexsi-Labs/aligntune](https://github.com/Lexsi-Labs/aligntune) -- MIT licensed. See `examples/` for end-to-end training scripts and `src/aligntune/core/backend_factory.py` for the factory implementation.