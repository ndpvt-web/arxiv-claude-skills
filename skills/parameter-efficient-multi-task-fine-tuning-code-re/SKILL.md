---
name: "parameter-efficient-multi-task-fine-tuning-code-re"
description: "Configure and execute multi-task QLoRA fine-tuning of code models for code generation, translation, and summarization. Use when user says 'fine-tune a code model for multiple tasks', 'set up QLoRA for code generation and summarization', 'multi-task training for code LLM', 'parameter-efficient fine-tuning for code', 'train one model for code gen and translation', or 'QLoRA multi-task code model'."
---

This skill enables Claude to guide practitioners through setting up multi-task QLoRA (Quantized Low-Rank Adaptation) fine-tuning pipelines for code-related Large Language Models. Based on research showing that a single QLoRA-adapted model trained jointly on code generation, code translation, and code summarization achieves competitive or superior performance versus separate single-task models—while using a fraction of the compute—this skill covers adapter configuration, multi-task data mixing, evaluation with both correctness and quality metrics, and model-size-aware tradeoff decisions.

## When to Use

- When the user wants to fine-tune a single code model to handle multiple tasks (generation, translation, summarization) instead of maintaining separate models
- When the user needs to reduce GPU memory and compute costs for code model specialization (full fine-tuning is infeasible)
- When the user asks how to configure QLoRA hyperparameters (rank, alpha, target modules, quantization) for code LLMs
- When the user wants to evaluate fine-tuned code models beyond pass@1—incorporating static analysis and code quality metrics
- When the user is choosing between single-task and multi-task fine-tuning strategies and needs guidance on transfer learning tradeoffs
- When the user asks about data mixing strategies for combining code generation, translation, and summarization datasets

## Key Technique

**QLoRA** freezes the base model weights in 4-bit NormalFloat (NF4) quantization and injects small low-rank adapter matrices into attention and feed-forward projection layers. This reduces trainable parameters to roughly 1-2% of the full model while preserving most of the model's capacity. The critical innovation here is applying QLoRA in a **multi-task** setting: one adapter is trained on a shuffled mixture of code generation (NL→Code), code translation (Code→Code across languages), and code summarization (Code→NL) examples simultaneously.

The research demonstrates that multi-task QLoRA leverages **cross-task transfer learning**: shared representations for understanding code structure benefit all three tasks. At 3B parameters, multi-task QLoRA improved Java code generation pass@1 by 7.3% over single-task QLoRA while simultaneously reducing cyclomatic complexity and maintainability issues. The key practical insight is that **model scale matters for quality**: 3B+ models maintain both correctness and code quality under multi-task training, while smaller models (0.5B–1.5B) tend to preserve functional correctness but accumulate more style and complexity issues.

The data mixing strategy is deliberately simple: concatenate all task datasets and shuffle with a fixed seed. No explicit task weighting is applied—the model encounters examples proportional to their dataset size. This simplicity is itself a finding: sophisticated sampling schedules are unnecessary when the adapter capacity and base model are sufficient.

## Step-by-Step Workflow

1. **Select your base code model.** Choose a code-specialized instruction-tuned model. The research used Qwen2.5-Coder-Instruct at 0.5B, 1.5B, and 3B. Prefer 3B+ for the best correctness-quality balance. Smaller models work when correctness alone is sufficient.

2. **Configure the QLoRA adapter.** Apply these proven hyperparameters:
   - Rank (`r`): 8
   - Scaling factor (`lora_alpha`): 16
   - Dropout: 0.1
   - Target modules: `q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj`
   - Quantization: 4-bit NF4 with double quantization enabled
   - Use paged optimizers (AdamW 8-bit paged) to handle memory spikes

3. **Prepare task-specific datasets with uniform formatting.** For each task, structure examples as instruction–response pairs with a system prompt identifying the task:
   - **Code generation:** Input = NL docstring/description → Output = source code
   - **Code translation:** Input = source in language A → Output = equivalent code in language B
   - **Code summarization:** Input = source code → Output = NL summary
   Use datasets like CodeXGLUE Code-to-Text (generation/summarization) and CodeXGLUE Code Translation (Java↔C#). Ensure each example includes the task-specific system prompt so the model learns to distinguish tasks at inference time.

4. **Mix the datasets.** Concatenate all task datasets into a single training corpus. Shuffle with a fixed random seed (e.g., `seed=4242`) for reproducibility. Do not apply task weighting—proportional sampling from the natural dataset sizes is sufficient.

5. **Train with standard supervised fine-tuning.** Use a library like `peft` + `trl` (SFTTrainer) or `axolotl`. Train on the shuffled multi-task corpus. Monitor per-task validation loss separately to detect catastrophic forgetting on any individual task.

6. **Evaluate functional correctness with task-appropriate metrics.** Run execution-based evaluation for code generation (pass@1 on CoderEval or HumanEval). Use CodeBLEU for translation tasks. Use BLEU, METEOR, ROUGE-L, chrF, BERTScore, and SIDE for summarization.

7. **Run static analysis on generated code.** This is the step most practitioners skip. Analyze generated outputs with:
   - **Lizard:** cyclomatic complexity, lines of code, token count
   - **PMD** (Java): best practices, code style, design, error-prone patterns
   - **Pylint** (Python): convention, refactor, warning, error categories
   - **SonarCloud/SonarQube:** security hotspots, reliability, maintainability, cognitive complexity
   - **Roslyn analyzers** (C#): syntax errors, maintainability

8. **Compare against single-task baselines.** Train single-task QLoRA models using the same adapter config but only one task's data. Compare both correctness metrics and quality metrics. Multi-task should match or beat single-task on correctness for 3B+ models.

9. **Iterate on task composition if needed.** If translation tasks show degraded CodeBLEU (common at 0.5B), consider either scaling up the model or excluding translation from the multi-task mix and training it separately. Generation and summarization tend to share representations well.

10. **Deploy with task-routing via system prompt.** At inference, use the task-specific system prompt to route the model's behavior. A single adapter serves all tasks—no model switching required.

## Concrete Examples

**Example 1: Setting up multi-task QLoRA training config**

User: "I want to fine-tune Qwen2.5-Coder-3B-Instruct on code generation and summarization using QLoRA. How do I configure this?"

Approach:
1. Install dependencies: `peft`, `transformers`, `trl`, `bitsandbytes`, `datasets`
2. Configure the quantization and adapter
3. Prepare the multi-task dataset
4. Launch training

Output:
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from trl import SFTTrainer, SFTConfig

# Quantization config (4-bit NF4 with double quantization)
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_use_double_quant=True,
    bnb_4bit_compute_dtype=torch.bfloat16,
)

model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-Coder-3B-Instruct",
    quantization_config=bnb_config,
    device_map="auto",
)
model = prepare_model_for_kbit_training(model)

# QLoRA adapter config
lora_config = LoraConfig(
    r=8,
    lora_alpha=16,
    lora_dropout=0.1,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                     "gate_proj", "up_proj", "down_proj"],
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Expected: ~1-2% of total parameters are trainable

# Combine datasets (generation + summarization)
from datasets import concatenate_datasets, load_dataset
gen_data = load_dataset("code_x_glue_ct_code_to_text", "python", split="train")
sum_data = load_dataset("code_x_glue_ct_code_to_text", "python", split="train")
# Format each with task-specific system prompts (see step 3 above)
combined = concatenate_datasets([gen_formatted, sum_formatted]).shuffle(seed=4242)

# Train
trainer = SFTTrainer(
    model=model,
    train_dataset=combined,
    args=SFTConfig(
        output_dir="./multitask-qlora-3b",
        per_device_train_batch_size=4,
        gradient_accumulation_steps=4,
        num_train_epochs=3,
        learning_rate=2e-4,
        optim="paged_adamw_8bit",
        bf16=True,
        logging_steps=50,
    ),
)
trainer.train()
```

**Example 2: Evaluating code quality beyond pass@1**

User: "My fine-tuned model passes 30% of tests but I'm worried about code quality. How do I check?"

Approach:
1. Generate code outputs from the model on the evaluation set
2. Run static analysis tools on every generated file
3. Compare against single-task baseline outputs
4. Aggregate and interpret the results

Output:
```bash
# Generate outputs (assuming saved to ./generated_code/)
# Then run quality analysis:

# Cyclomatic complexity with Lizard
lizard ./generated_code/ --csv > lizard_report.csv

# Python quality with Pylint
find ./generated_code/ -name "*.py" -exec pylint --output-format=json {} \; > pylint_report.json

# Java quality with PMD
pmd check -d ./generated_code/ -R rulesets/java/quickstart.xml -f json > pmd_report.json

# SonarQube analysis (requires running SonarQube server)
sonar-scanner -Dsonar.projectKey=multitask-eval \
  -Dsonar.sources=./generated_code/ \
  -Dsonar.host.url=http://localhost:9000
```

Key metrics to compare between single-task and multi-task models:
| Metric | What it reveals | Healthy direction |
|--------|----------------|-------------------|
| Cyclomatic complexity (Lizard) | Control flow complexity | Lower is better |
| Pylint convention issues | Style violations | Fewer is better |
| PMD error-prone patterns | Likely bugs | Fewer is better |
| SonarCloud maintainability | Long-term code health | Fewer issues is better |
| Lines of code | Verbosity | Concise is better (at 3B, multi-task produced 9.6% fewer LoC) |

**Example 3: Deciding between single-task and multi-task training**

User: "I have a 1.5B code model and need it for Java code generation and Java-to-C# translation. Should I train one model or two?"

Approach:
1. Assess model size against task count
2. Consider the transfer learning characteristics at this scale
3. Recommend based on empirical findings

Output:

At 1.5B parameters, the research shows mixed results for multi-task training:
- Java code generation at 1.5B saw a **25.5% drop** in pass@1 under multi-task vs single-task QLoRA
- Java→C# translation at 1.5B saw a modest **1.1% improvement** in CodeBLEU

**Recommendation for 1.5B:** Train two separate single-task QLoRA adapters. The adapter weights are small (~20-50MB each), so serving two adapters is practical. At 1.5B, the model lacks sufficient capacity to absorb cross-task knowledge without correctness regressions on generation.

If you can scale to 3B, multi-task becomes advantageous: Java generation improves by 7.3% pass@1 with comparable or better code quality.

## Best Practices

**Do:**
- Always evaluate code quality alongside correctness. Pass@1 alone hides maintainability regressions that static analysis catches (the paper found up to 70% increases in maintainability issues at small scale even when pass@1 held steady)
- Use a fixed random seed when shuffling the combined dataset to ensure reproducible task mixing
- Monitor per-task validation metrics separately during training to catch task-specific degradation early
- Prefer 3B+ parameter models when training multi-task—they consistently balance correctness and quality

**Avoid:**
- Do not apply complex task weighting or curriculum learning schedules; simple proportional mixing works well and adds no tuning burden
- Do not skip the `gate_proj`, `up_proj`, `down_proj` target modules—adapting only attention projections leaves significant capacity on the table
- Do not assume multi-task training always helps; at 0.5B–1.5B, certain task combinations (especially including translation) can cause negative transfer
- Do not evaluate only with similarity metrics (BLEU, CodeBLEU); execution-based metrics (pass@1) are essential for code generation since syntactically similar code can be functionally wrong

## Error Handling

- **Out of memory during training:** Enable gradient checkpointing, reduce batch size, or increase gradient accumulation steps. Double quantization should already be on. If still OOM, reduce sequence length or use paged optimizers.
- **Pass@1 drops significantly for one task after multi-task training:** This signals negative transfer. Check if one task dominates the dataset size. Try training with the problematic task excluded, or scale up the model. At 1.5B, Java generation showed -25.5% pass@1—the fix was moving to 3B.
- **High Pylint/PMD issues but good pass@1:** The model generates functionally correct but low-quality code. This is common at 0.5B–1.5B. Consider post-processing with a linter auto-fix pass, or accept the tradeoff if deployment is latency-sensitive.
- **CodeBLEU is high but execution fails:** CodeBLEU measures structural similarity, not runtime correctness. Always pair it with execution-based testing for generation and translation tasks.
- **Adapter merging conflicts:** If merging the QLoRA adapter back into the base model for deployment, verify that quantization artifacts don't corrupt weights. Test inference before and after merging on a held-out set.

## Limitations

- The research evaluated only Qwen2.5-Coder-Instruct at 0.5B/1.5B/3B. Results may not generalize directly to other architectures (CodeLlama, DeepSeek-Coder, StarCoder) or larger scales (7B+), though the directional findings about scale should hold.
- Only three tasks were tested (generation, translation, summarization). Adding more diverse tasks (bug detection, test generation, code review) may change the transfer dynamics.
- Language coverage was limited to Python, Java, and C#. Languages with less representation in pre-training data may behave differently.
- The rank=8 configuration was not ablated against other ranks. For significantly different model architectures, rank tuning may be necessary.
- Multi-task QLoRA still underperformed single-task on translation at most scales, suggesting that code-to-code translation may benefit less from cross-task transfer than generation or summarization.

## Reference

[Parameter-Efficient Multi-Task Fine-Tuning in Code-Related Tasks](https://arxiv.org/abs/2601.15094v1) — Haque, Afrin, Mastropaolo (2026). Look for: Table 3 (QLoRA config), Tables 4–9 (correctness results by task and model size), Tables 10–15 (static analysis quality results), and Section 4.1–4.3 for transfer learning analysis across tasks.