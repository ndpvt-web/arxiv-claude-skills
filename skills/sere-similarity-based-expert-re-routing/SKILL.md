---
name: "sere-similarity-based-expert-re-routing"
description: "Deploy SERE (Similarity-based Expert Re-routing) to accelerate MoE model batch decoding in vLLM by dynamically skipping redundant experts. Use when: 'speed up MoE inference', 'optimize Qwen MoE serving', 'reduce MoE expert activation overhead', 'SERE expert re-routing', 'batch decoding latency MoE', 'vLLM MoE throughput optimization'"
---

# SERE: Similarity-based Expert Re-routing for Efficient MoE Batch Decoding

This skill enables Claude to help users deploy and configure SERE, a method that accelerates Mixture-of-Experts (MoE) model inference by dynamically reducing active experts during batch decoding. SERE pre-computes similarity matrices between experts via calibration, then at inference time re-routes tokens from secondary experts to their most similar primary experts — skipping redundant computation without permanent pruning or retraining. It integrates into vLLM as a plugin with a custom CUDA kernel, achieving up to 2.0x decoding speedup with minimal quality loss.

## When to Use

- When a user wants to reduce latency or increase throughput of MoE model serving (Qwen1.5-MoE, DeepSeek-V2-Lite, Qwen3-30B-A3B)
- When deploying MoE models via vLLM and batch decoding is the bottleneck (memory-bound decoding stage)
- When the user asks about expert pruning or merging alternatives that don't require retraining
- When optimizing a production MoE inference pipeline for cost efficiency under latency SLAs
- When a user wants to trade a small accuracy margin for significant decoding speed gains on reasoning benchmarks
- When configuring `select_top_k` and `threshold` hyperparameters for a SERE deployment

## Key Technique

**The core problem:** In MoE models, each token activates only a few experts (sparse activation). But during batched inference, different tokens in a batch may route to different experts, causing the *union* of activated experts to grow large — negating the sparsity benefit and making the decoding stage memory-bound.

**SERE's solution operates in three stages per MoE layer:**

1. **Primary Expert Selection** — Across all tokens in the batch, identify the union of top-S experts (where S = `select_top_k`). These are the "primary" experts that will always execute. Every other expert routed to by any token is labeled "secondary."

2. **Similarity-based Re-routing** — For each secondary expert, look up its pre-computed similarity score against every primary expert. If the highest similarity exceeds a threshold `ρ`, re-route that secondary expert's tokens to the most similar primary expert. If no primary expert is sufficiently similar, the secondary expert is flagged as "critical" and preserved — it runs normally to prevent capability loss.

3. **Efficient Execution** — Only primary experts and preserved critical experts are activated. This reduces the total number of expert forward passes, directly cutting memory bandwidth and compute.

**Similarity matrices are computed once during an offline calibration step** using CKA (Centered Kernel Alignment), cosine similarity, or Frobenius norm on expert activations from a small calibration dataset. No model retraining or weight modification is needed. The custom CUDA kernel handles the re-routing logic efficiently within vLLM's execution pipeline.

## Step-by-Step Workflow

### 1. Verify Environment Prerequisites

Confirm the deployment environment meets requirements: Python 3.10+, PyTorch 2.6+, CUDA 12.4+, GCC/G++ 7.0+ with C++17, and vLLM 0.8.4. Set the v0 backend flag:

```bash
export VLLM_USE_V1=0
```

### 2. Clone SERE and Install the vLLM Plugin

```bash
git clone https://github.com/JL-Cheng/SERE.git && cd SERE
cd vllm && pip install .
```

This installs the SERE-augmented model architectures (e.g., `Qwen2MoeForCausalLMSERE`) and the custom CUDA kernel as a vLLM plugin.

### 3. Run Expert Similarity Calibration

Compute the pairwise similarity matrix between all experts in each MoE layer using a small calibration dataset:

```bash
cd calibration
python cal_expert_similarity.py \
  --model_type qwen2_moe \
  --model_path Qwen/Qwen1.5-MoE-A2.7B-Chat \
  --output_path ./output/qwen2_similarity \
  --similarity_method cka \
  --kernel linear \
  --batch_size 200 \
  --max_len 128
```

Choose the similarity method based on your accuracy/speed tradeoff:
- **CKA (linear kernel)**: Best quality preservation; recommended default
- **Cosine**: Faster calibration, slightly less robust
- **Frobenius**: Simplest norm-based comparison

The output directory will contain the model checkpoint, tokenizer, and serialized similarity matrices.

### 4. Select Hyperparameters (`select_top_k` and `threshold`)

- **`select_top_k`** (integer): Number of primary experts retained per layer. Lower values = more aggressive skipping = higher speedup but more quality risk. Start with `select_top_k = floor(original_top_k / 2)` (e.g., 2 for a model that normally routes to 4 experts).
- **`threshold`** (float, 0.0–1.0): Minimum similarity for re-routing. Lower threshold = more aggressive re-routing. Start with `0.1` and increase if quality degrades.

### 5. Initialize vLLM with SERE Configuration

```python
from vllm import LLM, SamplingParams

llm = LLM(
    model="./calibration/output/qwen2_similarity",
    tensor_parallel_size=1,
    gpu_memory_utilization=0.9,
    trust_remote_code=True,
    hf_overrides={
        "architectures": ["Qwen2MoeForCausalLMSERE"],
        "select_top_k": 2,
        "threshold": 0.1
    }
)
```

The architecture name in `hf_overrides` must match the model family:
- Qwen1.5-MoE / Qwen2-MoE: `Qwen2MoeForCausalLMSERE`
- DeepSeek-V2-Lite: use the corresponding SERE architecture name from the plugin
- Qwen3-30B-A3B: use the Qwen3 SERE architecture name

### 6. Run Batch Inference

```python
prompts = ["Explain quantum entanglement.", "Solve: 2x + 3 = 15"]
sampling_params = SamplingParams(max_tokens=512, temperature=0.0)
outputs = llm.generate(prompts, sampling_params)
for output in outputs:
    print(output.outputs[0].text)
```

### 7. Deploy as an OpenAI-Compatible API Server

```bash
export VLLM_USE_V1=0
vllm serve ./calibration/output/qwen2_similarity \
  --trust-remote-code \
  --hf-overrides '{"architectures": ["Qwen2MoeForCausalLMSERE"], "select_top_k": 2, "threshold": 0.1}' \
  --port 8000
```

Clients can hit `http://localhost:8000/v1/completions` using the standard OpenAI API format.

### 8. Evaluate Quality on Target Benchmarks

Run OpenCompass benchmarks to verify quality retention:

```bash
cd experiments/opencompass
opencompass eval_qwen1_5.py --work-dir ./results/ --mode all
```

Key benchmarks: MATH, GSM8K, HumanEval, MBPP (reasoning/code), CMMLU, BoolQ, BBH (general knowledge). Compare against baseline (no SERE) to quantify quality delta.

### 9. Tune Hyperparameters Based on Results

If quality loss exceeds tolerance: increase `select_top_k` by 1, or raise `threshold` by 0.05. If speedup is insufficient: decrease `select_top_k` or lower `threshold`. Iterate until the speedup-vs-quality tradeoff meets your SLA.

## Concrete Examples

**Example 1: Accelerating Qwen1.5-MoE-A2.7B for a Chat Service**

User: "I'm serving Qwen1.5-MoE-A2.7B-Chat on a single A100 with vLLM and decoding is too slow at batch size 32. How can I speed it up without switching models?"

Approach:
1. Clone SERE, install the vLLM plugin (`cd SERE/vllm && pip install .`)
2. Run calibration with CKA on the Chat model: `python cal_expert_similarity.py --model_type qwen2_moe --model_path Qwen/Qwen1.5-MoE-A2.7B-Chat --output_path ./calibrated_qwen --similarity_method cka --kernel linear`
3. Serve with SERE enabled: set `hf_overrides={"architectures": ["Qwen2MoeForCausalLMSERE"], "select_top_k": 2, "threshold": 0.1}`
4. The model normally activates 4 out of 60 experts per token. With `select_top_k=2`, SERE collapses redundant secondary experts into primaries, reducing active experts per batch and yielding ~1.5-2.0x decoding speedup.

Output: vLLM server runs at the same endpoint, same API — clients require zero changes. Decoding throughput roughly doubles at batch size 32.

**Example 2: Deploying Qwen3-30B-A3B with Latency Constraints**

User: "We need Qwen3-30B-A3B to respond within 2 seconds for our reasoning pipeline. Current p95 latency is 3.5s. Can SERE help?"

Approach:
1. Calibrate with: `python cal_expert_similarity.py --model_type qwen3_moe --model_path Qwen/Qwen3-30B-A3B-Instruct --output_path ./calibrated_qwen3 --similarity_method cka`
2. Qwen3-30B-A3B routes to 8 out of 128 experts. Start with `select_top_k=4` and `threshold=0.1`
3. Deploy via `vllm serve` with SERE overrides
4. Benchmark latency. If p95 is still above 2s, try `select_top_k=3`. If quality drops on MATH/GSM8K, raise threshold to 0.15
5. Validate on your specific reasoning tasks before production rollout

Output: Expect ~1.5-1.8x speedup at `select_top_k=4`, potentially reaching 2.0x at `select_top_k=3` — likely bringing p95 under the 2s target.

**Example 3: Writing a SERE Calibration Script for a Custom Dataset**

User: "I want to calibrate SERE on my domain-specific data instead of the default dataset. How?"

Approach:
1. Prepare calibration data as a Parquet file with a `text` column containing representative samples from your domain (200+ samples recommended)
2. Modify the calibration call to point to your data:
   ```bash
   python cal_expert_similarity.py \
     --model_type qwen2_moe \
     --model_path Qwen/Qwen1.5-MoE-A2.7B-Chat \
     --output_path ./calibrated_custom \
     --similarity_method cka \
     --kernel linear \
     --batch_size 100 \
     --max_len 256
   ```
3. The script tokenizes and filters sequences (keeping those > 64 tokens), runs forward passes to capture expert activations, then computes pairwise similarity matrices per MoE layer
4. Use the output path as the model path in vLLM initialization

Output: Similarity matrices calibrated to your domain's expert activation patterns, potentially yielding better quality preservation than generic calibration on domain-specific tasks.

## Best Practices

- **Do:** Always run calibration before deploying SERE. The pre-computed similarity matrices are essential — without them, the re-routing decisions have no basis.
- **Do:** Start conservative (`select_top_k` = half the original top-k, `threshold` = 0.1) and tune down only after verifying quality on your target tasks.
- **Do:** Use CKA with a linear kernel as the default similarity method — it is the most robust at capturing representational equivalence between experts.
- **Do:** Evaluate on reasoning-heavy benchmarks (MATH, GSM8K, HumanEval) since these are most sensitive to expert skipping.
- **Avoid:** Setting `select_top_k=1` unless you have verified quality is acceptable — overly aggressive reduction can degrade complex reasoning.
- **Avoid:** Using SERE with batch size 1 — the method's benefit comes from collapsing the expert union across batch tokens. Single-token batches gain nothing.
- **Avoid:** Skipping the `VLLM_USE_V1=0` environment variable — SERE requires the v0 vLLM backend.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| `KeyError: 'Qwen2MoeForCausalLMSERE'` | vLLM plugin not installed | Run `cd SERE/vllm && pip install .` |
| Calibration OOM | Batch size too large or max_len too high | Reduce `--batch_size` to 50 and `--max_len` to 64 |
| Quality collapse on benchmarks | `select_top_k` too low or `threshold` too low | Increase `select_top_k` by 1 or raise `threshold` by 0.05 |
| No speedup observed | Batch size too small (e.g., 1-2) | SERE benefits scale with batch size; test at batch >= 8 |
| CUDA compilation errors | GCC < 7.0 or missing C++17 support | Upgrade GCC/G++ to 7.0+ and verify `nvcc` availability |
| `VLLM_USE_V1` errors | v1 backend does not support SERE plugin | Set `export VLLM_USE_V1=0` before any vLLM command |

## Limitations

- **Model coverage:** Currently supports Qwen1.5-MoE, DeepSeek-V2-Lite, and Qwen3-30B-A3B. Other MoE architectures (Mixtral, DBRX, etc.) require writing adapted modeling files and CUDA kernel integration.
- **vLLM version lock:** Requires vLLM 0.8.4 with the v0 backend. Newer vLLM versions may break compatibility until the plugin is updated.
- **Prefill stage:** SERE targets the *decoding* stage specifically. The prefill (prompt processing) stage is compute-bound, not memory-bound, so SERE does not accelerate it.
- **Single-request latency:** For batch size 1, SERE provides no benefit because there is no cross-token expert redundancy to exploit. The technique is designed for batched serving.
- **Calibration is offline:** Similarity matrices are static after calibration. If the model is fine-tuned, recalibration is needed.
- **No dynamic threshold tuning:** The `threshold` is fixed at inference time. Adaptive per-layer thresholds could improve quality but are not yet implemented.

## Reference

**Paper:** [SERE: Similarity-based Expert Re-routing for Efficient Batch Decoding in MoE Models](https://arxiv.org/abs/2602.07616v1) (ICLR 2026)
**Code:** [https://github.com/JL-Cheng/SERE](https://github.com/JL-Cheng/SERE)
**Key insight:** Instead of permanently pruning or merging MoE experts, dynamically skip secondary experts at inference time by re-routing their tokens to similar primary experts — a batch-level optimization that preserves model weights and enables plug-and-play deployment via vLLM.