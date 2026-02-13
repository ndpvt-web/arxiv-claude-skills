---
name: "focus-dllms-know-tame"
description: "Deploy and optimize FOCUS inference for Diffusion Large Language Models (DLLMs). Configures attention-based token eviction to achieve up to 3.5x throughput gains. Use when: 'deploy FOCUS for DLLM inference', 'optimize diffusion language model serving', 'configure FOCUS token eviction', 'benchmark DLLM throughput', 'set up SDAR or LLaDA2 inference', 'tune FOCUS alpha and confidence parameters'."
---

This skill enables Claude to deploy, configure, and optimize the FOCUS inference system for Diffusion Large Language Models (DLLMs). FOCUS exploits the insight that only ~10% of block tokens are decodable at each diffusion step by computing attention-derived importance deltas between early transformer layers, dynamically evicting non-decodable tokens after Layer 1, and packing freed compute capacity into larger effective batches. This reduces computational redundancy by ~79% and achieves up to 3.52x throughput over production-grade baselines while preserving generation quality.

## When to Use

- When the user wants to deploy a Diffusion LLM (SDAR, LLaDA2) for production inference and needs throughput optimization
- When the user asks to set up the FOCUS inference engine from the sands-lab/FOCUS repository
- When tuning FOCUS hyperparameters (alpha, confidence threshold, block size) for a specific workload
- When benchmarking DLLM throughput against baseline LMDeploy or other engines
- When debugging token eviction behavior, KV-cache stability, or generation quality regressions in FOCUS
- When integrating FOCUS into an existing serving pipeline with continuous batching
- When evaluating whether FOCUS is appropriate for a given model architecture (dense vs. MoE)

## Key Technique

Standard DLLM decoding parallelizes computation across an entire block of B tokens at each diffusion step, but only a small fraction (~10%) are actually decodable at any given step. The remaining ~90% of compute is wasted on tokens that cannot yet transition from masked to decoded state. FOCUS identifies which tokens are likely decodable by computing an **importance delta** between the first two transformer layers: `delta_j = I_j(Layer1) - I_j(Layer0)`, where token importance `I_j` aggregates softmax-pooled attention scores across all heads and query positions. Layer 0 captures static positional priors; Layer 1 introduces semantic signal. The delta isolates the semantic divergence that correlates with decodability, acting as a common-mode rejection filter.

After computing importance deltas from Layer 1's Q/K projections, FOCUS determines a **retention budget** K using the formula `K = min(B, max(ceil(alpha * N_decoded_avg), N_sigma))`, where `alpha` controls expansion aggressiveness, `N_decoded_avg` is the historical average of decoded tokens per step, and `N_sigma` counts tokens whose deltas exceed one standard deviation. The top-K tokens (plus structural constraints for AR-context preservation and placeholder integrity) are gathered into a reduced tensor; evicted tokens retain their KV references but skip Layers 2+ entirely. This reduces per-step FLOPs proportionally while adding only ~1% overhead for the importance computation. A **neighbor-aware stability criterion** delays KV-cache commits until both a token and its right neighbor decode successfully, preventing error propagation.

The freed compute from eviction is reinvested through **dynamic batch packing**: because each sequence now processes fewer active tokens, more sequences fit into the same GPU memory budget. This transforms a compute-bound workload into one that better saturates hardware, yielding wall-clock throughput gains that scale with block size (larger blocks = more eviction opportunity = greater speedup).

## Step-by-Step Workflow

1. **Clone and install FOCUS** from `github.com/sands-lab/FOCUS`. Create a Python environment, install `requirements/runtime_cuda.txt`, then run `DISABLE_TURBOMIND=1 pip install -e .` to build the LMDeploy-based engine with FOCUS kernels (requires CUDA 12+).

2. **Select the target model and block size.** FOCUS supports SDAR variants (b16, b32, b64 block sizes on 8B-Chat) and LLaDA2.0-mini. Choose block size based on workload: b32 is the default; b64 yields peak speedup (3.52x) but requires more memory; b16 suits latency-sensitive deployments.

3. **Set the alpha expansion factor.** Start with `alpha = 1.5` (the paper's default). Higher alpha retains more tokens per step (safer but less speedup); lower alpha is more aggressive. For complex reasoning tasks (e.g., MATH benchmarks), keep alpha >= 1.5. For straightforward chat/generation, alpha = 1.2-1.5 works well.

4. **Set the confidence threshold.** Default is `Conf = 0.8` for robustness across workloads. Increase to 0.9 for higher quality on demanding benchmarks; decrease to 0.7 for maximum throughput on tolerant workloads.

5. **Run the FOCUS throughput benchmark** to establish baseline performance: `benchmark/run_focus_throughput_evaluation.sh <dataset_id> <model_id> [alpha]`. Compare against the baseline: `benchmark/run_baseline_throughput_evaluation.sh <dataset_id> <model_id>`. Results land in `./results/`.

6. **Analyze the eviction statistics** from FocusState logs. Check the ratio of retained-to-total tokens per step. A healthy deployment evicts 65-80% of tokens. If retention is too high (>50%), alpha may be too large or the workload has unusually high decodability rates.

7. **Validate generation quality** using OpenCompass benchmarks (bundled in `opencompass-0.5.1.post1/`). Run evaluations on target benchmarks (GSM8K, MATH, HumanEval, MBPP, MT-Bench) comparing FOCUS output against baseline. Quality should be preserved or slightly improved due to the implicit focus on high-importance tokens.

8. **Tune block size for your hardware.** Run `benchmark/run_block_size_comparison.sh <dataset_id>` to sweep b16/b32/b64. On memory-constrained GPUs, b32 balances throughput and memory. On A100/H100 with ample memory, b64 maximizes FOCUS benefit.

9. **Integrate into production serving.** FOCUS supports continuous batching and ragged PagedAttention natively. Configure the scheduler's max batch size upward (FOCUS's token eviction frees capacity for more concurrent sequences). Monitor the delayed cache state coordination to ensure KV-cache consistency under load.

10. **Monitor and iterate.** Track per-step decoded token counts, eviction ratios, and end-to-end latency under production traffic. Adjust alpha and confidence threshold based on observed quality-throughput tradeoffs for your specific traffic pattern.

## Concrete Examples

**Example 1: Deploy FOCUS for an SDAR-8B chat model**

User: "Set up FOCUS inference for SDAR-8B-Chat with block size 32 on my A100 GPU."

Approach:
1. Clone repository and install dependencies:
```bash
git clone https://github.com/sands-lab/FOCUS.git
cd FOCUS
python -m venv focus_env && source focus_env/bin/activate
pip install -r requirements/runtime_cuda.txt
DISABLE_TURBOMIND=1 pip install -e .
```

2. Run throughput benchmark with default alpha:
```bash
# FOCUS with alpha=1.5 (default)
bash benchmark/run_focus_throughput_evaluation.sh \
  "dataset/sharegpt_processed.jsonl" \
  "sdar-b32-8b-chat" \
  1.5

# Baseline comparison
bash benchmark/run_baseline_throughput_evaluation.sh \
  "dataset/sharegpt_processed.jsonl" \
  "sdar-b32-8b-chat"
```

3. Compare results in `./results/` -- expect ~2-3x throughput improvement at batch sizes 64-256.

Output:
```
Baseline (LMDeploy):  1,247 tokens/sec @ batch=128
FOCUS (alpha=1.5):    3,412 tokens/sec @ batch=128  (2.74x)
Eviction ratio:       72.3% tokens evicted per step
Quality (MT-Bench):   7.82 (baseline: 7.79) -- preserved
```

---

**Example 2: Tune alpha for a math reasoning workload**

User: "FOCUS is dropping quality on GSM8K. How do I fix it?"

Approach:
1. Check current alpha and confidence values. Math reasoning requires more conservative eviction because multi-step reasoning tokens have subtler importance signals.
2. Increase alpha from 1.5 to 2.0 to retain more tokens per step:
```bash
bash benchmark/run_focus_throughput_evaluation.sh \
  "dataset/gsm8k.jsonl" \
  "sdar-b32-8b-chat" \
  2.0
```
3. If quality is still degraded, increase confidence threshold to 0.9 in the model config.
4. Run OpenCompass evaluation to verify:
```bash
cd opencompass-0.5.1.post1
# Configure eval with FOCUS backend, alpha=2.0, conf=0.9
python run.py configs/eval_gsm8k_focus.py
```

Output:
```
alpha=1.5, conf=0.8:  GSM8K accuracy 74.2% (baseline 76.1%)  -- degraded
alpha=2.0, conf=0.9:  GSM8K accuracy 76.4% (baseline 76.1%)  -- restored
Throughput tradeoff:  2.74x -> 2.11x (still substantial gain)
```

---

**Example 3: Evaluate block size tradeoffs**

User: "Should I use b32 or b64 for my high-throughput batch inference pipeline?"

Approach:
1. Run the block size comparison sweep:
```bash
bash benchmark/run_block_size_comparison.sh "dataset/sharegpt_processed.jsonl"
```
2. Analyze the results for your target batch size range.
3. Check GPU memory utilization -- b64 processes 2x more tokens per block but FOCUS evicts proportionally more, so net memory growth is sublinear.

Output:
```
Block Size | Baseline tok/s | FOCUS tok/s | Speedup | GPU Mem
b16        | 2,103          | 4,879       | 2.32x   | 28 GB
b32        | 1,247          | 3,412       | 2.74x   | 34 GB
b64        |   891          | 3,136       | 3.52x   | 42 GB
```

Recommendation: Use b64 if your A100/H100 has sufficient memory headroom. The 3.52x speedup at b64 comes from FOCUS evicting ~80% of the larger token block, turning the worst-case block size into the best-case for FOCUS.

## Best Practices

**Do:**
- Start with the paper's defaults (`alpha=1.5`, `Conf=0.8`, `b32`) before tuning -- they are robust across workloads
- Monitor eviction ratios per step; 65-80% eviction is the healthy operating range
- Use the neighbor-aware stability criterion (enabled by default) to prevent KV-cache corruption from premature commits
- Increase max batch size in the scheduler configuration to exploit freed compute from token eviction
- Run quality benchmarks (OpenCompass) whenever changing alpha or confidence, not just throughput tests

**Avoid:**
- Setting alpha below 1.2 -- overly aggressive eviction causes quality degradation, especially on reasoning tasks
- Disabling the delayed cache state coordination for latency reasons -- it prevents cascading decode errors that are hard to diagnose
- Using FOCUS with block size b16 when throughput is the primary goal -- the eviction overhead (1% of step latency) has diminishing returns on small blocks
- Assuming MoE models (LLaDA2.0-mini) will see the same gains as dense models -- MoE's inherent sparsity reduces FOCUS's relative benefit

## Error Handling

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Quality drops on reasoning benchmarks | Alpha too low or confidence too low | Increase alpha to 2.0+, confidence to 0.9 |
| Throughput gain < 1.5x | Block size too small or batch size too low | Use b32+ and batch size >= 64 |
| CUDA OOM on b64 | Insufficient GPU memory for large blocks | Drop to b32 or reduce max batch size |
| Inconsistent outputs across runs | KV-cache instability from disabled delayed cache | Re-enable neighbor-aware stability criterion |
| Build fails on `pip install -e .` | Missing CUDA toolkit or wrong version | Ensure CUDA 12+ is installed; check `nvcc --version` |
| Eviction ratio below 50% | Workload has unusually high per-step decodability | This is fine -- FOCUS gracefully degrades to near-baseline when most tokens are decodable |
| Scheduler overhead at high batch sizes | Multi-loop optimization conflicts with large batches | On MATH-type workloads at batch 64+, profile scheduler vs. compute time and consider disabling multi-loop |

## Limitations

- **DLLM-specific**: FOCUS only applies to Diffusion Large Language Models (SDAR, LLaDA2). It does not benefit autoregressive models (LLaMA, GPT, etc.) which decode one token at a time.
- **Model support**: Currently limited to SDAR (b16/b32/b64 variants of 8B-Chat) and LLaDA2.0-mini. Other DLLMs require porting the importance delta computation and eviction kernels.
- **GPU required**: The Triton kernels in `lmdeploy/pytorch/kernels/cuda/focus.py` require NVIDIA GPUs with CUDA 12+. No CPU or AMD GPU support.
- **MoE models see reduced gains**: Models with Mixture-of-Experts routing already have inherent sparsity, so FOCUS's additional token-level sparsity provides less marginal benefit.
- **Not a quality improvement tool**: FOCUS preserves quality but is not designed to improve it. On some benchmarks it shows slight quality gains (likely from implicit attention to important tokens), but this is a side effect, not a guarantee.
- **Single hyperparameter sensitivity**: While alpha is the only tunable, it interacts with block size and confidence threshold in workload-dependent ways. No single setting is optimal across all tasks.

## Reference

**Paper**: [FOCUS: DLLMs Know How to Tame Their Compute Bound](https://arxiv.org/abs/2601.23278v1) (Liang et al., 2026)
**Code**: [github.com/sands-lab/FOCUS](https://github.com/sands-lab/FOCUS)
**Key insight**: Section 3's analysis of the attention importance delta (Equations 2-5) and the empirical correlation between Layer 0-1 delta and token decodability probability. This is the theoretical foundation that makes training-free token eviction possible.