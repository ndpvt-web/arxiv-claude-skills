---
name: "learning-rate-matters-vanilla"
description: "Configure optimal learning rates for LoRA fine-tuning of LLMs. Generates hyperparameter search configs, training scripts, and analysis code that ensure vanilla LoRA matches or beats fancy variants. Triggers: 'fine-tune with LoRA', 'set up LoRA training', 'compare LoRA variants', 'optimize LoRA hyperparameters', 'learning rate sweep for LoRA', 'LoRA fine-tuning config'"
---

# Learning Rate Matters: Optimal LoRA Fine-Tuning Configuration

This skill enables Claude to configure, generate, and debug LoRA fine-tuning pipelines with properly tuned learning rates. Based on the finding that vanilla LoRA matches the performance of PiSSA, DoRA, MiLoRA, and Init[AB] within 1-2% when learning rates are correctly swept, this skill produces hyperparameter search configurations, training scripts, and analysis tooling that avoid the single-configuration trap that plagues most LoRA setups.

## When to Use

- When the user asks to fine-tune an LLM with LoRA and needs a training configuration
- When the user is choosing between LoRA variants (PiSSA, DoRA, MiLoRA, etc.) and wants guidance
- When a LoRA fine-tuning run produces poor results and the user suspects hyperparameter issues
- When the user needs to set up a learning rate sweep or hyperparameter search for LoRA
- When the user asks to generate a training script for math reasoning or code generation tasks
- When the user wants to benchmark multiple LoRA methods fairly against each other
- When the user is debugging a LoRA training run that diverges or underperforms

## Key Technique

The core insight is that reported gains from LoRA variants (PiSSA, DoRA, MiLoRA, Init[AB]) over vanilla LoRA are largely artifacts of fixed hyperparameter settings. Different LoRA initialization strategies create different loss landscape curvatures at initialization -- measured by the maximum Hessian eigenvalue (lambda_max). PiSSA, which initializes trainable parameters along principal singular directions, produces significantly higher curvature and therefore requires a lower optimal learning rate. When each method gets its own properly tuned learning rate, performance gaps collapse to within 1-2%.

The practical rule is: **optimal learning rate is inversely proportional to the largest Hessian eigenvalue at initialization** (eta* ~ 1/lambda_max). Since vanilla LoRA (zero-initialized B, random A) has lower curvature than SVD-based methods like PiSSA, it tolerates higher learning rates. This means a single fixed learning rate (e.g., 2e-5) will favor whichever method's curvature happens to match that rate, creating the illusion of methodological superiority.

The recommended search strategy uses a logarithmic grid of 12-16 learning rates spanning 1e-5 to 6e-3, with batch sizes of {16, 64, 128} and ranks in {8, 16, 32, 64, 128}. Learning rate is the most critical axis: tuning only batch size with a fixed LR yielded 11.16% accuracy on one benchmark, while tuning LR alone yielded 20.5-21.0%. The optimal LR also scales proportionally with batch size, following the classical linear scaling rule from SGD theory.

## Step-by-Step Workflow

1. **Identify the task and model scale.** Determine whether the fine-tuning target is math reasoning, code generation, instruction following, or another task. Note the base model (e.g., Llama-2-7B, Qwen3-0.6B, Gemma-3-1B) since optimal LR ranges shift with model size.

2. **Default to vanilla LoRA.** Unless the user has a specific reason to use a variant, configure standard LoRA with zero-initialized B and Kaiming-initialized A. Set alpha = rank (scaling factor gamma_r = 1). This is the simplest setup and matches variant performance when tuned.

3. **Generate the logarithmic learning rate grid.** Produce 12-16 values spanning three orders of magnitude. Use the geometric progression: for each order (e.g., 1e-4), generate values at multipliers [1.12, 2.0, 3.56, 6.32]. A typical grid for vanilla LoRA: `[1.1e-5, 2e-5, 3.6e-5, 6.3e-5, 1.1e-4, 2e-4, 3.6e-4, 6.3e-4, 1.1e-3, 2e-3, 3.6e-3, 6.3e-3]`.

4. **Set the rank search space.** For initial experiments, use rank 64 or 128 as the default. If the user needs to explore rank sensitivity, sweep `{8, 16, 32, 64, 128, 256}`. Note rank-dependent behaviors: DoRA shows up to 1.1% gains at r=8 but the advantage vanishes at higher ranks; PiSSA underperforms at r<=32 but catches up at higher ranks.

5. **Configure batch size as a secondary axis.** Test {16, 64, 128} if compute allows. When scaling batch size by factor k, scale learning rate by the same factor k (linear scaling rule). If compute is limited, fix batch size and sweep only LR -- this captures most of the performance variance.

6. **Generate the training script with sweep logic.** Output a script using HuggingFace PEFT + Transformers (or the user's preferred framework) with either a grid search loop, Optuna integration, or Weights & Biases sweep config. Include early stopping and validation checkpointing.

7. **Set fixed hyperparameters.** Keep these constant across the sweep: LoRA target modules (typically q_proj, v_proj, or all linear layers), alpha = rank, number of epochs, warmup ratio, weight decay, and LR scheduler (cosine is standard). These are secondary to LR and should not vary during the primary sweep.

8. **Run the sweep and collect results.** Generate analysis code that plots accuracy vs. learning rate for each configuration, identifies the peak LR, and reports the performance envelope (best accuracy +/- std across seeds).

9. **Diagnose failures.** If runs diverge at high LR, the Hessian curvature is too high for that rate -- lower the LR. If performance plateaus at low LR, the model is underfitting -- raise the LR. If a LoRA variant underperforms vanilla LoRA, check whether it was given the same LR (it likely needs a different one).

10. **Report results with proper context.** Present best accuracy per method at its own optimal LR, not at a shared LR. Flag when performance differences are within noise (1-2%) and recommend the simpler method (vanilla LoRA) in such cases.

## Concrete Examples

**Example 1: Generate a LoRA fine-tuning config for code generation**

User: "Set up LoRA fine-tuning for Llama-2-7B on Python code generation data"

Approach:
1. Configure vanilla LoRA with rank=64, alpha=64, targeting all linear layers
2. Generate a learning rate sweep from 1.1e-5 to 6.3e-3
3. Produce a PEFT training script with the sweep

Output:
```python
from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM, TrainingArguments
import numpy as np

# --- Learning Rate Grid (log-scale, 12 points) ---
lr_grid = []
for exp in range(-5, -2):
    for mult in [1.1247, 2.0, 3.5566, 6.3246]:
        lr_grid.append(round(mult * 10**exp, 8))

# --- LoRA Config (vanilla, zero-init B, Kaiming A) ---
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=64,
    lora_alpha=64,          # alpha = rank => gamma_r = 1
    lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                     "gate_proj", "up_proj", "down_proj"],
    init_lora_weights=True,  # default: A ~ Kaiming, B = 0
)

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf", torch_dtype="bfloat16"
)
model = get_peft_model(model, lora_config)

# --- Sweep over learning rates ---
for lr in lr_grid:
    training_args = TrainingArguments(
        output_dir=f"./output/lr_{lr:.2e}",
        learning_rate=lr,
        per_device_train_batch_size=16,
        num_train_epochs=3,
        lr_scheduler_type="cosine",
        warmup_ratio=0.03,
        weight_decay=0.01,
        bf16=True,
        evaluation_strategy="steps",
        eval_steps=500,
        save_strategy="steps",
        save_steps=500,
        load_best_model_at_end=True,
        logging_steps=50,
    )
    # ... initialize Trainer with training_args and run
```

**Example 2: Debug underperforming PiSSA vs LoRA comparison**

User: "I'm comparing PiSSA to LoRA on math tasks and PiSSA is doing worse. Both use LR=2e-4."

Approach:
1. Identify the problem: PiSSA has higher Hessian curvature at initialization, so its optimal LR is lower than vanilla LoRA's
2. Recommend a lower LR range for PiSSA (roughly 0.3x-0.5x of the LoRA-optimal LR)
3. Generate a targeted sweep for PiSSA

Output:
```
The issue is that PiSSA initializes along principal singular vectors,
creating higher loss curvature (larger Hessian eigenvalues). This means
PiSSA needs a LOWER learning rate than vanilla LoRA.

If LoRA works well at 2e-4, try PiSSA at:
  - 3.6e-5, 6.3e-5, 1.1e-4 (roughly 0.2x to 0.5x of LoRA's optimal)

To confirm, run both methods across the full LR grid and compare
each at its own best LR, not at a shared LR.
```

**Example 3: Set up a W&B sweep for comparing LoRA methods**

User: "Create a Weights & Biases sweep config to compare vanilla LoRA, DoRA, and PiSSA"

Approach:
1. Define a W&B sweep YAML with method, LR grid, and LoRA variant as categorical
2. Include the log-scale LR distribution and rank options
3. Set the metric to maximize

Output:
```yaml
# wandb_sweep.yaml
program: train_lora.py
method: grid
metric:
  name: eval/accuracy
  goal: maximize
parameters:
  lora_method:
    values: ["vanilla", "dora", "pissa"]
  learning_rate:
    values: [1.1e-5, 2.0e-5, 3.6e-5, 6.3e-5,
             1.1e-4, 2.0e-4, 3.6e-4, 6.3e-4,
             1.1e-3, 2.0e-3, 3.6e-3, 6.3e-3]
  rank:
    values: [16, 64, 128]
  batch_size:
    values: [16, 64]
  lora_alpha:
    value: -1  # sentinel: set alpha = rank in train script
  num_epochs:
    value: 3
  warmup_ratio:
    value: 0.03
```

## Best Practices

**Do:**
- Always sweep learning rate on a log scale spanning at least 2-3 orders of magnitude. A single LR is the most common source of misleading LoRA comparisons.
- Set `lora_alpha = rank` (gamma_r = 1) to decouple the scaling factor from the rank, making LR sweeps more interpretable.
- Scale learning rate proportionally when changing batch size (double the batch, double the LR).
- Compare LoRA variants at each method's own best LR, never at a shared fixed LR.

**Avoid:**
- Avoid adopting a LoRA variant (PiSSA, DoRA, etc.) solely based on a published comparison that used a single learning rate. The reported gains likely reflect LR mismatch, not method superiority.
- Avoid sweeping batch size while holding learning rate fixed -- this captures very little of the performance variance (11% vs 21% accuracy in the paper's worst case).
- Avoid using very high ranks (r > 128) with PiSSA at the same LR that works for vanilla LoRA. PiSSA's curvature increases with rank, requiring even lower LRs.
- Avoid concluding a method is better based on < 2% improvement -- this is within noise across seeds and LR sensitivity.

## Error Handling

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Training loss diverges (NaN/Inf) | Learning rate too high for this method's curvature | Reduce LR by 3-5x; PiSSA especially prone |
| Loss plateaus, no improvement | Learning rate too low | Increase LR by 3-5x; check if LR scheduler is decaying too aggressively |
| LoRA variant worse than vanilla | Using vanilla LoRA's optimal LR for the variant | Sweep LR independently per method |
| Performance degrades at high rank | Overfitting or LR-rank mismatch | Reduce rank, or lower LR (curvature grows with rank for SVD-init methods) |
| Results vary wildly across seeds | Operating near a steep region of the LR-performance curve | Use 3+ seeds at the current LR; shift to a flatter region of the curve |

## Limitations

- The paper's results cover math reasoning (GSM8K, MATH) and code generation (HumanEval, MBPP). The equivalence of LoRA variants may not hold identically for instruction tuning, RLHF, or domain-specific tasks with very different loss landscapes.
- Models tested go up to 7B parameters. At much larger scales (70B+), the relative curvature properties and optimal LR ranges may shift.
- The sweep budget (12-16 LRs x 3 batch sizes x multiple ranks) requires significant compute. For single-run constraints, vanilla LoRA at LR ~2e-4 to 3.6e-4 with rank 64 is a reasonable starting point for 1B-7B models.
- The Hessian analysis is performed at initialization. Training dynamics could introduce method-specific advantages later in training that the initial curvature analysis does not capture.
- Alpha is fixed at alpha = rank throughout. Alternative alpha-rank relationships (e.g., alpha = 16 regardless of rank, as some practitioners use) were not studied.

## Reference

**Paper:** [Learning Rate Matters: Vanilla LoRA May Suffice for LLM Fine-tuning](https://arxiv.org/abs/2602.04998v1) (Lee et al., 2026)
**Key takeaway:** Look at Table 1 and Figure 1 for the LR-performance curves showing how all methods converge to within 1-2% at their respective optimal LRs, and Section 4.2 for the Hessian eigenvalue analysis explaining *why* different methods need different LRs.