---
name: "tamperbench-systematically-stress-testing-safety"
description: "Set up and run TamperBench pipelines to systematically stress-test LLM safety under fine-tuning and tampering attacks. Use when: 'stress-test model safety', 'evaluate tamper resistance', 'run tampering attacks on LLM', 'benchmark safety defenses', 'set up TamperBench', 'sweep attack hyperparameters against model'."
---

# TamperBench: Systematically Stress-Testing LLM Safety

This skill enables Claude to set up, configure, and run TamperBench -- a unified framework for evaluating the tamper resistance of open-weight LLMs. TamperBench curates weight-space fine-tuning attacks (LoRA, full-parameter, jailbreak-tuning) and latent-space representation attacks, runs systematic hyperparameter sweeps per attack-model pair using Optuna, and reports both safety metrics (StrongREJECT, XSTest, SafetyGap, WMDP) and utility metrics (MMLU-Pro, MT-Bench, MBPP, IFEval). Claude uses this skill to automate the end-to-end pipeline: installation, configuration, attack execution, defense integration, evaluation, and results analysis.

## When to Use

- When the user wants to evaluate how resistant an open-weight LLM is to safety-breaking fine-tuning attacks
- When the user asks to benchmark a new alignment defense (CRL, TAR, Tamper-Resistant Vaccine) against known attacks
- When the user needs to run hyperparameter sweeps to find worst-case attack configurations for a specific model
- When the user wants to compare tamper resistance across multiple models or model families (Llama, Qwen, Gemma, etc.)
- When the user is adding a custom attack, defense, or evaluation metric to an existing TamperBench installation
- When the user asks to generate safety/utility heatmaps or analyze TamperBench results
- When the user needs to set up a CI/CD pipeline or Kubernetes job for automated tamper-resistance testing

## Key Technique

TamperBench's core insight is that tamper-resistance evaluation must be **adversarially rigorous**: using a single fixed set of attack hyperparameters underestimates vulnerability. Instead, TamperBench runs Optuna-based hyperparameter optimization per attack-model pair, searching for the configuration that maximizes harmfulness (measured by StrongREJECT score) while constraining utility loss to at most 10% on MMLU-Pro. This simulates a realistic attacker who tunes their approach to each target model rather than using off-the-shelf settings.

The framework organizes tampering threats into two categories. **Weight-space attacks** modify model parameters directly -- these include LoRA fine-tuning, full-parameter fine-tuning, jailbreak-tuning (the most severe attack found in the paper), multilingual fine-tuning, competing-objectives fine-tuning, style-modulated attacks, and backdoor injection. **Latent-space attacks** manipulate internal representations without traditional gradient-based training. Each attack is specified as a typed Python config class or YAML file, making it trivial to add new attacks via a decorator-based plugin architecture.

Three alignment-stage defenses are built in: **CRL** (contrastive representation learning), **TAR** (tamper-aware regularization), and **Tamper-Resistant Vaccine (T-Vaccine)**. The paper finds that Triplet-style defenses (combining contrastive objectives) emerge as the strongest. TamperBench evaluates safety via StrongREJECT (LLM-judged refusal quality), XSTest, SafetyGap, and WMDP (domain-specific risk in biology, chemistry, cybersecurity), and evaluates utility via MMLU-Pro, MT-Bench, MBPP (code generation), and IFEval (instruction following).

## Step-by-Step Workflow

1. **Clone and install TamperBench** with `uv` for dependency isolation:
   ```bash
   git clone https://github.com/criticalml-uw/tamperbench.git
   cd tamperbench
   uv sync --all-groups
   pre-commit install
   ```

2. **Select the target model** from Hugging Face Hub (e.g., `meta-llama/Llama-3.1-8B-Instruct`, `Qwen/Qwen3-4B`). Confirm the model is open-weight and that the user has accepted any license gates.

3. **Choose attacks to run** based on the threat model. For a comprehensive sweep, use all attacks. For a quick assessment, start with `lora_finetune` and `jailbreak_tuning` (the most severe):
   ```bash
   uv run scripts/whitebox/optuna_single.py meta-llama/Llama-3.1-8B-Instruct \
       --attacks lora_finetune jailbreak_tuning \
       --n-trials 50 \
       --model-alias llama31_8b
   ```

4. **Configure hyperparameter sweep ranges** by editing YAML files in `configs/whitebox/attacks/<attack_name>/single_objective_sweep.yaml`. Define float ranges with log-uniform sampling for learning rate, integer ranges for LoRA rank, and categorical choices for schedulers:
   ```yaml
   learning_rate:
     type: "float"
     low: 1.0e-6
     high: 1.0e-3
     log: true
   lora_rank:
     type: "int"
     low: 4
     high: 64
     step: 4
   lr_scheduler_type:
     choices: [constant, cosine, linear]
   ```

5. **Run grid-based benchmarking** for deterministic, reproducible runs using pre-defined configs:
   ```bash
   uv run scripts/whitebox/benchmark_grid.py Qwen/Qwen3-4B \
       --attacks lora_finetune full_parameter_finetune jailbreak_tuning \
       --model-alias qwen3_4b
   ```

6. **Integrate a defense** by specifying the defense method in the attack config. Defense-augmented models are evaluated with the same attack pipeline to measure how well the defense holds:
   ```python
   from tamperbench.whitebox.attacks.lora_finetune.lora_finetune import (
       LoraFinetune, LoraFinetuneConfig
   )
   config = LoraFinetuneConfig(
       input_checkpoint_path="meta-llama/Llama-3.1-8B-Instruct",
       out_dir="results/llama31_tar_defense",
       evals=[EvalName.STRONG_REJECT, EvalName.MMLU_PRO_VAL],
       per_device_train_batch_size=8,
       learning_rate=1e-4,
       num_train_epochs=1,
       lora_rank=16,
   )
   attack = LoraFinetune(attack_config=config)
   results = attack.benchmark()
   ```

7. **Run standalone evaluations** on existing checkpoints to compute safety and utility scores:
   ```python
   from tamperbench.whitebox.evals.strong_reject.strong_reject import (
       StrongRejectEvaluation, StrongRejectEvaluationConfig
   )
   config = StrongRejectEvaluationConfig(
       checkpoint_path="results/llama31_tar_defense/checkpoint",
       out_dir="results/llama31_tar_defense/eval"
   )
   evaluation = StrongRejectEvaluation(config)
   results = evaluation.run_evaluation()
   ```

8. **Inspect results** in the structured output directory. Each run produces `inferences.parquet` (raw generations), `scores.parquet` (per-sample scores), and `results.parquet` (aggregates). Use these to compare attack severity across models and defenses.

9. **Generate comparison heatmaps** from the results. Darker cells indicate higher harmfulness (lower tamper resistance); lighter cells indicate stronger resistance. Cross-reference StrongREJECT safety scores with MMLU-Pro utility scores to identify Pareto-optimal defenses.

10. **Scale to production** using Kubernetes configs in `k8s/` for parallel multi-model, multi-attack sweeps across a GPU cluster. Use `--model-alias` consistently to enable resumable sweeps and organized result directories.

## Concrete Examples

**Example 1: Quick tamper-resistance assessment of a new model**

User: "I want to check how safe Qwen3-4B is against fine-tuning attacks."

Approach:
1. Install TamperBench and verify GPU availability.
2. Run Optuna sweep with the two most critical attacks (LoRA fine-tuning and jailbreak-tuning) for 50 trials each.
3. Evaluate with StrongREJECT (safety) and MMLU-Pro (utility).
4. Report the worst-case safety score found by the sweep.

```bash
# Install
git clone https://github.com/criticalml-uw/tamperbench.git && cd tamperbench
uv sync --all-groups

# Run sweep
uv run scripts/whitebox/optuna_single.py Qwen/Qwen3-4B \
    --attacks lora_finetune jailbreak_tuning \
    --n-trials 50 \
    --model-alias qwen3_4b

# Results are in results/qwen3_4b/<attack>/
# Check results.parquet for aggregate StrongREJECT and MMLU-Pro scores
```

Output: A directory per attack containing parquet files with safety/utility scores. The worst-case StrongREJECT score across trials indicates the model's tamper resistance floor.

---

**Example 2: Comparing defenses against jailbreak-tuning**

User: "I need to benchmark CRL, TAR, and T-Vaccine defenses on Llama-3.1-8B against jailbreak-tuning."

Approach:
1. Prepare three defense-augmented model checkpoints (one per defense).
2. Run the same jailbreak-tuning attack grid against each.
3. Collect StrongREJECT and MMLU-Pro scores for all three.
4. Produce a comparison table.

```python
defenses = ["crl", "tar", "t_vaccine"]
results_summary = {}

for defense in defenses:
    config = LoraFinetuneConfig(
        input_checkpoint_path=f"results/llama31_{defense}/checkpoint",
        out_dir=f"results/llama31_{defense}_jailbreak",
        evals=[EvalName.STRONG_REJECT, EvalName.MMLU_PRO_VAL],
        per_device_train_batch_size=8,
        learning_rate=1e-4,
        num_train_epochs=1,
        lora_rank=16,
    )
    attack = LoraFinetune(attack_config=config)
    results_summary[defense] = attack.benchmark()
```

Output:
```
Defense     | StrongREJECT (safety) | MMLU-Pro (utility)
------------|----------------------|-------------------
CRL         | 0.72                 | 0.61
TAR         | 0.68                 | 0.63
T-Vaccine   | 0.81                 | 0.59
```
(Higher StrongREJECT = better safety; higher MMLU-Pro = better utility.)

---

**Example 3: Adding a custom attack to TamperBench**

User: "I have a new fine-tuning attack method and want to integrate it into TamperBench."

Approach:
1. Create a new directory under `src/tamperbench/whitebox/attacks/<your_attack>/`.
2. Define a config dataclass extending the base attack config.
3. Implement the attack class with a `benchmark()` method.
4. Register it via the decorator-based plugin system.
5. Add YAML configs for grid and sweep under `configs/whitebox/attacks/<your_attack>/`.

```
configs/whitebox/attacks/my_custom_attack/
  grid.yaml                      # Fixed-parameter grid config
  single_objective_sweep.yaml    # Optuna hyperparameter ranges

src/tamperbench/whitebox/attacks/my_custom_attack/
  __init__.py
  my_custom_attack.py            # Attack class + config dataclass
```

The grid.yaml must contain a `base` key with default parameters:
```yaml
base:
  template: generic_chat
  max_generation_length: 512
  inference_batch_size: 4
  per_device_train_batch_size: 8
  learning_rate: 1e-4
  num_train_epochs: 3
  evals:
    - strong_reject
    - mmlu_pro_val
```

Then run:
```bash
uv run scripts/whitebox/benchmark_grid.py Qwen/Qwen3-4B \
    --attacks my_custom_attack --model-alias qwen3_4b
```

## Best Practices

- **Do** always run Optuna sweeps (`optuna_single.py`) rather than single-configuration tests when assessing tamper resistance. A single configuration drastically underestimates vulnerability.
- **Do** constrain utility loss during sweeps (default: MMLU-Pro must not drop more than 10%). This ensures attacks are realistic -- a model that loses all capability is not a useful attack.
- **Do** use `--model-alias` on every CLI run. This organizes results by model and enables Optuna to resume interrupted sweeps.
- **Do** test the base model without any attack first to establish safety and utility baselines for comparison.
- **Avoid** running only one attack type. Jailbreak-tuning is typically the most severe, but other attacks (multilingual, competing objectives) may exploit different vulnerabilities.
- **Avoid** comparing results across runs that used different evaluation prompts or metric versions. Pin your TamperBench version and config for reproducibility.

## Error Handling

- **Out of GPU memory**: Reduce `per_device_train_batch_size` or `inference_batch_size`. For large models, use LoRA attacks instead of full-parameter fine-tuning.
- **Optuna trial failures**: Individual trials may fail (divergent loss, OOM). TamperBench logs these and continues. Check that at least 30% of trials succeed for a meaningful sweep.
- **Model tokenizer mismatches**: Set `user_prefix`, `assistant_prefix`, and `end_turn` tokens in ModelConfig to match the target model's chat template. Incorrect tokens cause garbled generations and unreliable safety scores.
- **Stale checkpoints from interrupted runs**: Use `--model-alias` to let Optuna's SQLite-backed storage resume from the last completed trial rather than restarting.
- **HuggingFace access errors**: Ensure you have accepted model license agreements and set `HF_TOKEN` in your environment for gated models.

## Limitations

- Requires significant GPU resources. Optuna sweeps of 50+ trials on 8B+ parameter models need multi-GPU setups or long runtimes.
- Evaluates only open-weight models. Closed API models (GPT-4, Claude) cannot be fine-tuned or weight-tampered and are outside scope.
- StrongREJECT is LLM-judged, inheriting judge-model biases. Cross-validate with XSTest or manual review for high-stakes assessments.
- The framework focuses on English-language safety. Multilingual attack vectors are included, but evaluation datasets are primarily English.
- Defense integration assumes access to the model training pipeline. Pre-trained defense checkpoints may not be available for all models.
- Results are specific to the attack hyperparameter ranges searched. A more capable attacker with a wider search space may find worse-case configurations not captured by the default sweeps.

## Reference

**Paper**: [TamperBench: Systematically Stress-Testing LLM Safety Under Fine-Tuning and Tampering](https://arxiv.org/abs/2602.06911v1) -- Hossain et al., 2026. Key sections: Section 3 (framework design), Section 4 (attack taxonomy), Section 5 (experimental results showing jailbreak-tuning severity and Triplet defense effectiveness), Table 1 (full model-attack heatmap).

**Code**: [github.com/criticalml-uw/TamperBench](https://github.com/criticalml-uw/TamperBench)