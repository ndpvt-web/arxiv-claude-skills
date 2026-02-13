---
name: "unicomp-unified-evaluation-compression"
description: "Guide Claude through evaluating and recommending LLM compression strategies (pruning, quantization, distillation) using the UniComp framework. Triggers: 'compress my model', 'quantize this LLM', 'prune a language model', 'which compression method should I use', 'optimize model for deployment', 'reduce model size for inference'"
---

# UniComp: Unified Evaluation of LLM Compression via Pruning, Quantization, and Distillation

This skill enables Claude to act as an LLM compression advisor, helping users select, configure, and evaluate compression techniques for large language models. Drawing on the UniComp framework, Claude assesses trade-offs across three dimensions -- performance retention, reliability (safety/bias), and hardware-aware efficiency -- to recommend the right compression strategy for a given deployment scenario. Rather than defaulting to a single method, Claude applies the paper's key finding that compression choice depends heavily on which capabilities matter most: knowledge recall, reasoning, multilingual support, or instruction following.

## When to Use

- When the user wants to deploy an LLM to resource-constrained hardware (edge devices, consumer GPUs, mobile) and needs to choose between pruning, quantization, or distillation
- When the user asks which quantization format (GPTQ, AWQ, INT4, INT8) to use for a specific model and task
- When the user wants to prune an LLM and needs guidance on calibration data to preserve reasoning ability
- When the user is comparing compressed model variants and needs a structured evaluation framework across performance, safety, and efficiency
- When the user wants to understand how compression will affect specific capabilities (e.g., "Will quantizing my model hurt multilingual performance?")
- When the user is building a compression pipeline and needs to decide method ordering and configuration
- When the user asks about distillation cost-benefit analysis versus quantization for production serving

## Key Technique

UniComp's central contribution is demonstrating that LLM compression is not uniformly lossy -- it exhibits a **knowledge bias**. Knowledge-intensive tasks (factual QA, MMLU-style benchmarks) are relatively preserved under compression, while reasoning (GSM8K, GPQA), multilingual, and instruction-following capabilities degrade substantially. This means standard evaluations on knowledge-centric benchmarks systematically overestimate compressed model quality. Any serious compression evaluation must test across capability categories, not just aggregate accuracy.

The framework evaluates six techniques spanning three families: **pruning** (SparseGPT, Wanda), **quantization** (GPTQ, AWQ), and **knowledge distillation** (student-teacher transfer). Across 40+ benchmarks on modern LLMs, quantization consistently provides the best trade-off between retained performance and inference efficiency -- typical 2-4x latency reduction with under 5% accuracy loss when properly calibrated. Distillation yields the strongest runtime speedups but demands enormous computational cost upfront (full training runs). Pruning is cost-effective to apply but degrades more unpredictably without task-specific calibration.

The most actionable finding is that **task-specific calibration improves pruned model reasoning by up to 50%**. Instead of using generic calibration data (e.g., WikiText), selecting calibration samples that match the target task distribution (e.g., math word problems for a math assistant) dramatically recovers lost reasoning capability. This applies to both pruning and quantization but has the largest impact on structured/semi-structured pruning methods like SparseGPT.

## Step-by-Step Workflow

1. **Profile the deployment constraints.** Identify the target hardware (GPU type, VRAM, CPU-only), latency budget (tokens/second), memory ceiling, and batch size requirements. These constraints eliminate entire compression families early.

2. **Categorize the primary task capabilities.** Classify the model's workload into UniComp's capability buckets: knowledge recall, reasoning/math, multilingual, instruction following, code generation, or safety-critical. This determines which benchmarks matter and which compression risks are acceptable.

3. **Select candidate compression methods using the decision matrix:**
   - **Tight memory budget, moderate accuracy tolerance** -> Quantization (AWQ for 4-bit, GPTQ for flexible group sizes)
   - **Maximum throughput, large training budget available** -> Distillation to a smaller architecture
   - **Moderate compression with fine-grained control** -> Pruning (SparseGPT for structured, Wanda for unstructured)
   - **Reasoning-heavy workload with pruning** -> Must include task-specific calibration

4. **Prepare task-matched calibration data.** Collect 128-512 representative samples from the target task distribution. For math/reasoning tasks, use problem-solution pairs. For multilingual tasks, use balanced language samples. For instruction following, use diverse instruction-response pairs. Avoid defaulting to WikiText or C4 unless the task is general language modeling.

5. **Apply compression with appropriate configuration:**
   - Quantization: Start with 4-bit (INT4) group-size 128. If accuracy drops exceed tolerance, try 8-bit or mixed-precision (sensitive layers at higher precision).
   - Pruning: Start at 50% sparsity with SparseGPT. Use task-specific calibration data. Evaluate before increasing sparsity.
   - Distillation: Define student architecture (typically 40-60% of teacher parameters). Use intermediate layer matching plus output logit distillation.

6. **Evaluate across all three UniComp dimensions, not just accuracy:**
   - **Performance**: Run benchmarks from each capability category (at minimum: one knowledge task like MMLU, one reasoning task like GSM8K, one instruction-following task like IFEval or AlpacaEval).
   - **Reliability**: Test for safety regression using TruthfulQA, BBQ (bias), and toxicity benchmarks. Compression can amplify existing biases.
   - **Efficiency**: Measure actual inference latency, peak memory, and throughput on target hardware -- not just theoretical compression ratio.

7. **Analyze results for knowledge bias.** Compare per-category accuracy drops. If reasoning drops exceed knowledge drops by more than 2x, the compression is exhibiting knowledge bias. Consider: increasing calibration data for weak categories, reducing compression aggressiveness, or switching methods.

8. **Iterate on calibration if reasoning degrades.** If pruning or quantization shows disproportionate reasoning loss, rebuild calibration data with 2-3x more reasoning-oriented samples and re-compress. The paper shows this single change can recover up to 50% of lost reasoning performance.

9. **Run end-to-end integration tests.** Validate the compressed model in the actual serving pipeline (not just offline benchmarks). Measure real latency including tokenization, KV-cache overhead, and batch scheduling.

10. **Document the compression configuration and trade-offs.** Record the method, bit-width/sparsity, calibration data source, and per-category benchmark results so the decision can be revisited when models or requirements change.

## Concrete Examples

**Example 1: Choosing compression for a coding assistant on consumer GPU**

```
User: I want to deploy Llama 3 70B as a coding assistant on a single RTX 4090 (24GB VRAM).
Which compression method should I use?

Approach:
1. Profile constraints: 24GB VRAM, need interactive latency (<100ms/token),
   single-GPU, primary task is code generation (reasoning-heavy).
2. 70B at FP16 = ~140GB -> need at least 6x memory reduction.
3. Distillation is out (no training budget mentioned, need the 70B's capability).
4. Pruning alone unlikely to achieve 6x memory reduction reliably.
5. Quantization to 4-bit: 70B * 0.5 bytes = ~35GB -- still too large.
   Quantization to 3-bit or GGUF Q4_K_M with offloading: ~20-25GB, fits.
6. Code generation is reasoning-heavy -> knowledge bias risk is HIGH.
   Must evaluate on HumanEval/MBPP, not just MMLU.

Recommendation:
- Use AWQ 4-bit quantization with group-size 128 as baseline.
- If 4-bit doesn't fit, use GGUF Q4_K_M format with partial CPU offloading.
- Calibrate with code-heavy data (e.g., StarCoder training samples), NOT WikiText.
- Benchmark on HumanEval pass@1 and GSM8K, not just perplexity.
- Expected: <5% drop on code tasks with AWQ-4bit if calibration is code-focused.
- If reasoning drops >10%, try mixed-precision: keep attention layers at 8-bit,
  FFN layers at 4-bit.
```

**Example 2: Evaluating whether compression broke safety alignment**

```
User: I quantized our customer-facing chatbot to INT4 and it passes our accuracy
benchmarks. Is it safe to deploy?

Approach:
1. Accuracy benchmarks alone are insufficient -- UniComp shows compression can
   degrade safety/reliability even when knowledge benchmarks hold steady.
2. Run reliability evaluation:
   - TruthfulQA: Check if hallucination rate increased post-compression
   - BBQ (Bias Benchmark for QA): Test for amplified demographic biases
   - Toxicity generation: Prompt with edge cases, measure toxic output rate
   - Instruction refusal: Verify the model still refuses harmful requests
3. Compare pre/post compression on each safety metric.
4. Check instruction-following fidelity: compression often degrades the model's
   ability to follow system prompts and constraints.

Output (example evaluation report):
| Metric              | FP16 Baseline | INT4 Quantized | Delta  |
|---------------------|---------------|----------------|--------|
| MMLU (knowledge)    | 78.2%         | 76.8%          | -1.4%  |
| GSM8K (reasoning)   | 72.1%         | 64.3%          | -7.8%  |
| TruthfulQA          | 61.0%         | 55.2%          | -5.8%  |
| BBQ bias score      | 0.12          | 0.19           | +58%   |
| Instruction refusal | 94%           | 87%            | -7.4%  |

Verdict: Knowledge looks fine (-1.4%), but reasoning (-7.8%), truthfulness
(-5.8%), and bias (+58%) all show significant degradation. This model is NOT
safe to deploy without mitigation. Recalibrate with safety-oriented data or
use 8-bit quantization instead.
```

**Example 3: Task-specific calibration to recover reasoning in pruned model**

```
User: I pruned Mistral 7B to 50% sparsity with SparseGPT using WikiText
calibration. Math performance dropped from 65% to 38% on GSM8K. Can I fix this?

Approach:
1. This is the classic knowledge bias problem: WikiText calibration preserves
   language modeling (knowledge) but sacrifices reasoning structure.
2. UniComp shows task-specific calibration recovers up to 50% of lost reasoning.
   Lost performance = 65% - 38% = 27 points. Potential recovery: ~13.5 points.
3. Rebuild calibration dataset:
   - Collect 256 GSM8K-style math word problems with chain-of-thought solutions
   - Mix in 128 general reasoning samples (e.g., from ARC-Challenge)
   - Keep 128 WikiText samples to maintain general capability
   - Total: 512 calibration samples, 75% task-specific
4. Re-run SparseGPT pruning at 50% sparsity with new calibration data.
5. Re-evaluate on GSM8K AND MMLU (ensure knowledge didn't regress).

Expected outcome:
- GSM8K: ~38% -> ~50-52% (recovering ~50% of the 27-point drop)
- MMLU: Should remain within 1-2% of WikiText-calibrated version
- If still insufficient, reduce sparsity to 40% with same calibration strategy
```

## Best Practices

- **Do:** Always evaluate compressed models on reasoning benchmarks (GSM8K, GPQA, HumanEval), not just knowledge benchmarks (MMLU, TriviaQA). Knowledge scores mask real degradation.
- **Do:** Use task-specific calibration data that matches your deployment workload. This is the single highest-impact intervention for pruning and quantization quality.
- **Do:** Measure actual hardware efficiency (latency, memory, throughput) on target devices. Theoretical compression ratios (e.g., "4x smaller") do not translate linearly to speedups due to memory bandwidth, kernel support, and batch scheduling.
- **Do:** Test safety and bias metrics after compression. Models can become less aligned even when accuracy looks stable.
- **Avoid:** Using WikiText/C4 as default calibration data for task-specific models. This is the most common source of unnecessary quality loss.
- **Avoid:** Comparing compression methods using only perplexity or a single aggregate score. UniComp shows that per-category analysis reveals critical differences hidden by averages.
- **Avoid:** Assuming distillation is always better than quantization. Distillation requires full training runs (days/weeks on many GPUs) while quantization is a post-training step (minutes/hours on one GPU) with competitive quality retention.

## Error Handling

- **Compressed model produces garbage output**: Likely over-aggressive compression. Reduce bit-width (try 8-bit instead of 4-bit) or sparsity (try 30% instead of 50%). Check that the quantization library version matches the model architecture.
- **Accuracy looks fine but users report quality issues**: You are probably evaluating only knowledge-centric benchmarks. Add reasoning, instruction-following, and open-ended generation tests. The knowledge bias means standard benchmarks miss real degradation.
- **Calibration data doesn't help**: Ensure calibration samples have the same tokenization and format as inference inputs. Mismatched prompt templates between calibration and deployment nullify calibration benefits.
- **Model fits in memory but inference is slow**: Quantized models need kernel support for actual speedup. Verify the serving framework (vLLM, TGI, llama.cpp) has optimized kernels for your chosen format. INT4 without GPU kernel support may be slower than FP16.
- **Mixed results across categories after compression**: This is expected -- it's the knowledge bias at work. Prioritize the capability category that matters most for your use case and optimize calibration for it, accepting graceful degradation elsewhere.

## Limitations

- UniComp's benchmarks focus on English-centric and standard academic tasks. Compression behavior on domain-specific tasks (medical, legal, financial) may differ and requires separate validation.
- The 50% reasoning recovery from task-specific calibration is an upper bound observed in controlled experiments. Real-world recovery depends on calibration data quality and task alignment.
- The framework evaluates models up to ~70B parameters. Compression dynamics for 100B+ models (e.g., Llama 3 405B) may exhibit different trade-off curves.
- Hardware-aware efficiency measurements are GPU-specific. CPU, mobile, and custom accelerator deployments may show different relative advantages between methods.
- UniComp does not cover combination strategies (e.g., quantization + pruning together, or distillation followed by quantization) which are increasingly common in practice.

## Reference

**UniComp: A Unified Evaluation of Large Language Model Compression via Pruning, Quantization and Distillation**
Jonathan von Rad, Yong Cao, Andreas Geiger (2026). arXiv: [2602.09130v2](https://arxiv.org/abs/2602.09130v2)
Key takeaway: Look at Tables 2-5 for per-category accuracy breakdowns showing the knowledge bias effect, and Section 5.3 for the calibration experiments that recover reasoning performance in pruned models.