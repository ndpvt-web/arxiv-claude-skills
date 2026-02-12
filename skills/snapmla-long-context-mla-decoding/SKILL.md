---
name: "snapmla-long-context-mla-decoding"
description: "Deploy and optimize FP8-quantized Multi-head Latent Attention (MLA) decoding for long-context LLM inference on Hopper GPUs. Implements SnapMLA's RoPE-aware KV cache quantization, PV pipeline reconstruction, and end-to-end dataflow optimization for up to 1.91x throughput gains. Use when: 'optimize MLA decoding for long context', 'add FP8 quantization to DeepSeek MLA', 'speed up long-context inference on H800', 'implement RoPE-aware KV cache quantization', 'reduce MLA KV cache memory with FP8', 'deploy SnapMLA with SGLang'."
---

# SnapMLA: Efficient Long-Context MLA Decoding via FP8 Quantized Pipelining

This skill enables Claude to help users implement, deploy, and optimize SnapMLA — a hardware-aware FP8 quantization framework for DeepSeek's Multi-head Latent Attention (MLA) architecture during autoregressive decoding. SnapMLA addresses three specific challenges that arise when applying FP8 to MLA: the numerical heterogeneity between RoPE and content components in the KV cache, the quantization scale misalignment in PV GEMM caused by MLA's shared KV structure, and the need for end-to-end kernel-level dataflow optimization on Hopper GPUs. The technique delivers up to 1.91x throughput improvement with negligible accuracy loss on long-context tasks including math reasoning and code generation.

## When to Use

- When the user wants to deploy DeepSeek-V3, DeepSeek-R1, or any MLA-based model with FP8 quantization for long-context inference (16k-128k tokens)
- When the user is building or modifying CUDA attention kernels for MLA decoding on NVIDIA Hopper (H800/H100/H20) GPUs
- When the user needs to reduce KV cache memory consumption for MLA models while preserving accuracy on reasoning tasks
- When the user asks about quantizing attention mechanisms that have decoupled positional embeddings (RoPE separate from content)
- When the user is configuring SGLang-FluentLLM or similar serving frameworks for MLA model deployment
- When the user encounters accuracy degradation after naively applying FP8 quantization to MLA attention
- When the user is writing or debugging FP8 GEMM kernels that need scale-aware accumulation for attention PV computation

## Key Technique

**The MLA Quantization Problem.** MLA compresses key-value representations into a shared low-dimensional latent vector `c_KV` (dimension `d_c`, much smaller than full `d_h * n_h`). Keys are formed by concatenating a content component `k_C = W_UK * c_KV` with a decoupled RoPE component `k_R` that carries positional information. The critical insight is that these two components have radically different numerical distributions: RoPE values span a dynamic range of approximately +/-10^3 with outlier tails, while content components concentrate near zero within +/-10^1. Naively applying uniform FP8 (E4M3) quantization to the concatenated KV cache increases RoPE reconstruction error by an order of magnitude, destroying positional information that is essential for long-context tasks.

**SnapMLA's Three-Part Solution.** First, RoPE-Aware Per-Token KV Quantization keeps the RoPE component in BF16 while quantizing the content `c_KV` in FP8, with per-token granularity that aligns naturally with autoregressive decoding (each new token gets its own scale factor, eliminating complex "tail buffer" management that per-block schemes require). Second, Quantized PV Pipeline Reconstruction solves the scale misalignment problem: because V is derived from the same shared `c_KV`, it inherits per-token quantization scales that align with GEMM's reduction dimension rather than its output dimension — breaking standard post-GEMM dequantization. SnapMLA fuses V's quantization scale into the attention probability matrix P before PV GEMM, then applies block-wise dynamic quantization (block size 64) to the scaled P, integrating dequantization directly into the online softmax computation. Third, End-to-End Dataflow Optimization uses three fused kernels (Fused-Q-Quant, Fused-K-Append, Fused-Fetch-Dequant) plus memory subsystem tuning (128-wide tiling aligned to L2 cache lines, Swizzle-128B shared memory layout, register-file V-tile transposition overlapped with QK GEMM).

**Pre-Scaled Domain Alignment** is a key formula: `Q_R = Q_R / S_Qc` and `K_R = K_R / S_Kc`, which projects the BF16 RoPE query/key components into the FP8 content quantization domain. This allows the RoPE and content attention score terms to be accumulated together without explicit precision conversion during the QK dot product.

## Step-by-Step Workflow

1. **Identify the MLA model and target hardware.** Confirm the model uses MLA (DeepSeek-V3, DeepSeek-R1, LongCat-Flash, or derivatives). Confirm Hopper-class GPU (H100/H800/H20) with FP8 Tensor Core support. SnapMLA's kernel design depends on Hopper's `k-major` layout constraint and TMA (Tensor Memory Accelerator) descriptors.

2. **Set up the SGLang-FluentLLM inference stack.** Clone `meituan-longcat/SGLang-FluentLLM`, install with `pip3 install -e "./python[cuda_sm90]"`, and run `sh clean_setup.sh sm90`. This builds the custom FlashMLA FP8 kernels and DeepGEMM variants. Requires Python 3.11, uv 0.7.2, and Rust toolchain.

3. **Choose the model checkpoint with FP8 weights.** Use FP8-quantized checkpoints (e.g., `meituan-longcat/LongCat-Flash-Lite-FP8`) for maximum throughput, or BF16 checkpoints (e.g., `meituan-longcat/LongCat-Flash-Lite`) when accuracy is paramount and you rely on runtime KV cache quantization only.

4. **Configure the attention backend and parallelism.** Launch with `--attention-backend flashmla --enable-flashinfer-mla`. Choose parallelism based on context length: use `--dp-size 8 --attn-tp-size 1` (DP8/TP1) for maximum throughput on shorter contexts, or `--dp-size 1 --attn-tp-size 8` (DP1/TP8) for very long contexts (128k+) that need KV cache distributed across GPUs.

5. **Configure the KV cache quantization strategy.** Ensure per-token quantization is active for the `c_KV` content component (FP8 E4M3) while RoPE keys `k_R` remain in BF16. This is the default in SnapMLA kernels. Do NOT apply blanket FP8 quantization to the full concatenated KV — this will destroy positional information.

6. **Tune memory and batch parameters.** Set `--mem-fraction-static 0.85` to allocate GPU memory for KV cache. For latency-sensitive workloads, add `--low-latency-max-num-tokens-per-gpu 2048`. Set `--max-running-requests 32` as the tested batch size that saturates the kernel pipeline.

7. **Validate accuracy on target tasks.** Run evaluation on representative long-context benchmarks. Expected accuracy deltas: <0.1% on MMLU-Pro, <0.5% on MATH-500, <3% on AIME-25 (the hardest reasoning tasks show the most sensitivity). If accuracy drops exceed these thresholds, check that RoPE components are not being quantized.

8. **Profile kernel performance.** Use `nsys profile` or `ncu` to verify the SnapMLA kernel is achieving near the effective theoretical peak of ~279.6 TFLOPS (computed as 148 TFLOPS BF16 base * 17/9 FP8 ratio). Kernel utilization should saturate at ~85% peak with >= 64 attention heads. If throughput is lower, check tiling alignment (should be 128 along content dimension) and TMA descriptor coalescing.

9. **Implement custom kernels (advanced).** If writing custom MLA FP8 kernels, follow the SnapMLA pipeline: (a) Fused-Q-Quant kernel for per-token scale computation + mixed-precision conversion + domain alignment, (b) Fused-K-Append for quantization + PagedAttention-style cache write, (c) main attention kernel with dual-warp-group pipelining where the first warp group handles QK GEMM while the second handles scale-fused PV GEMM with register-file transposition overlapped with the QK stage.

10. **Enable speculative decoding for additional throughput.** Combine SnapMLA with speculative decoding: `--speculative-algorithm NEXTN --speculative-num-steps 3 --speculative-num-draft-tokens 4`. This compounds throughput gains since SnapMLA reduces per-step decode latency.

## Concrete Examples

**Example 1: Deploy DeepSeek-V3 with SnapMLA for 128k-context serving**

User: "I need to serve DeepSeek-V3 for long-context code generation tasks (up to 128k tokens) on 8x H800 GPUs. How do I get maximum throughput with FP8?"

Approach:
1. Clone and build SGLang-FluentLLM with sm90 support
2. Download the FP8 checkpoint
3. Launch with DP1/TP8 for 128k context distribution across GPUs
4. Enable SnapMLA attention backend

Output:
```bash
# Install
git clone git@github.com:meituan-longcat/SGLang-FluentLLM.git
cd SGLang-FluentLLM
pip3 install uv==0.7.2
uv venv fluentllmenv --python 3.11 --seed
source fluentllmenv/bin/activate
pip3 install -e "./python[cuda_sm90]" --no-cache-dir
sh clean_setup.sh sm90

# Launch — DP1/TP8 for long context, flashmla backend for SnapMLA FP8 decoding
python3 -m sglang.launch_server \
  --model-path meituan-longcat/LongCat-Flash-Lite-FP8 \
  --trust-remote-code \
  --mem-fraction-static 0.85 \
  --attention-backend flashmla \
  --enable-flashinfer-mla \
  --dp-size 1 \
  --attn-tp-size 8 \
  --nprocs-per-node 8 \
  --max-running-requests 32 \
  --port 10000 \
  --host 0.0.0.0
```
Expected result: ~1.9x throughput vs BF16 baseline at 128k context length.

**Example 2: Diagnose accuracy loss after FP8 quantization of MLA model**

User: "I applied FP8 quantization to my MLA model's KV cache and accuracy on AIME dropped by 15%. What went wrong?"

Approach:
1. Check whether RoPE components are being quantized — this is the most common cause
2. Verify quantization granularity is per-token, not per-tensor
3. Apply SnapMLA's RoPE-aware strategy

Analysis:
```
Root cause: Uniform FP8 quantization of the full KV cache [c_KV; k_R] destroys
positional information because:
- Content c_KV has dynamic range ~[-10, +10] → fits E4M3 well
- RoPE k_R has dynamic range ~[-1000, +1000] with outlier tails → E4M3 loses
  critical precision, increasing MSE by ~10x

Fix: Split the KV cache quantization:
- c_KV (content): FP8 E4M3 with per-token scale factors
- k_R (RoPE):     Keep in BF16

Additionally, apply pre-scaled domain alignment for the QK dot product:
  Q_R = Q_R / S_Qc    (divide BF16 RoPE query by content's FP8 scale)
  K_R = K_R / S_Kc    (divide BF16 RoPE key by content's FP8 scale)

This projects RoPE into the FP8 domain so content and positional terms
can be accumulated together without explicit precision conversion.
```

Expected result: Accuracy restored to within 0.1-3% of BF16 baseline depending on task difficulty.

**Example 3: Write a custom FP8 PV GEMM kernel for MLA**

User: "I'm writing a custom CUDA kernel for MLA PV computation with FP8. My dequantization is producing wrong results."

Approach:
1. Identify the scale misalignment problem specific to MLA's shared KV
2. Implement scale-fusion pipeline reconstruction
3. Use block-wise dynamic quantization on the scale-fused P matrix

Analysis and code pattern:
```
Problem: In MLA, V = W_UV * c_KV. Both K content and V derive from the same
c_KV, so V inherits per-token quantization scales S_V[t] along the sequence
dimension. In Hopper's k-major PV GEMM layout, this means scales align with
the reduction dimension — standard post-GEMM dequantization CANNOT work.

Solution — Scale-Fusion Pipeline:
  Step 1: Fuse V scales into attention probs:  P'[h,t] = P[h,t] * S_V[t]
  Step 2: Block-quantize P' with block_size=64: P'_fp8, S_P = quantize(P')
  Step 3: PV GEMM in FP8:  O_block = P'_fp8 @ V_fp8
  Step 4: Dequantize with P's block scale only:  O = O_block * S_P

// Pseudocode for scale fusion in softmax
for each block k:
    m_new = max(m_old, max(S_Kc[k] * QK_block[k]))  // incorporate K scale
    L_new = L_old * exp(m_old - m_new) + sum(exp(S_Kc[k] * QK[k] - m_new))
    // Fuse S_V into the softmax output before PV GEMM
    P_scaled[k] = exp(S_Kc[k] * QK[k] - m_new) * S_V[k]
    P_fp8[k], S_P[k] = dynamic_quantize(P_scaled[k], block_size=64)
    O_block[k] = fp8_gemm(P_fp8[k], V_fp8[k])  // both in E4M3
    O = rescale(O, O_block[k], S_P[k], L_old, L_new, m_old, m_new)
```

## Best Practices

- **Do:** Always split KV cache quantization into content (FP8) and RoPE (BF16) components. The numerical heterogeneity between these two is fundamental to MLA and cannot be resolved by finer quantization granularity alone.
- **Do:** Use per-token quantization granularity for autoregressive decoding. It aligns naturally with token-by-token generation and avoids the "tail buffer" complexity of per-block quantization in PagedAttention.
- **Do:** Apply pre-scaled domain alignment (`Q_R / S_Qc`, `K_R / S_Kc`) so that RoPE and content attention terms can be accumulated in the same precision domain without explicit type conversion in the inner loop.
- **Do:** Tile the content dimension at 128 (not 64) to align with Hopper's 128-byte L2 cache lines and Swizzle-128B shared memory layout. This directly impacts memory bandwidth utilization.
- **Avoid:** Applying uniform FP8 quantization to the concatenated `[c_KV; k_R]` KV cache. This is the single most common mistake and will cause severe accuracy degradation on long-context and reasoning tasks.
- **Avoid:** Using per-tensor quantization for the KV cache in decoding mode. Per-tensor granularity computed over the full sequence conflates tokens with different value distributions, especially as context grows to 64k+ tokens.
- **Avoid:** Standard post-GEMM dequantization for PV computation in MLA. The shared `c_KV` structure means V's per-token scales align with the GEMM reduction dimension, not the output dimension, requiring scale-fusion into P before the GEMM.

## Error Handling

| Issue | Symptom | Fix |
|-------|---------|-----|
| RoPE quantized in FP8 | >5% accuracy drop on reasoning, especially positional-sensitive tasks (needle-in-haystack, long-context QA) | Ensure k_R stays in BF16; check kernel is using the split quantization path |
| PV scale misalignment | NaN or wildly incorrect attention outputs; loss spikes | Implement scale-fusion pipeline: fuse S_V into P before PV GEMM, then use block-wise dynamic quantization on the fused P |
| Memory OOM at 128k context | CUDA out-of-memory during decoding | Switch to DP1/TP8 to distribute KV cache; reduce `--mem-fraction-static` to 0.80; reduce `--max-running-requests` |
| Low kernel utilization (<50%) | Throughput much lower than 279.6 TFLOPS effective peak | Check number of attention heads (need >=64 to saturate); verify tiling is 128 along content dim; check TMA descriptors are coalesced |
| Build failure on clean_setup.sh | Compilation errors for FlashMLA kernels | Verify CUDA toolkit >= 12.3, sm_90 target, Rust installed; ensure submodules are initialized |
| Accuracy slightly lower on AIME | 2-3% delta on hardest reasoning benchmarks | This is expected behavior within SnapMLA's accuracy envelope; if unacceptable, keep full BF16 for these workloads |

## Limitations

- **Hopper-only.** SnapMLA's kernel design depends on Hopper architecture features: FP8 Tensor Cores with E4M3 format, TMA descriptors, Swizzle-128B shared memory layout, and k-major WGMMA constraints. It will not work on Ampere (A100) or older GPUs.
- **MLA-specific.** The techniques address quantization challenges unique to MLA's decoupled RoPE and shared KV structure. Standard multi-head attention (MHA) or grouped-query attention (GQA) can use simpler FP8 quantization without the RoPE-aware splitting or PV scale fusion.
- **Decoding phase only.** SnapMLA targets the autoregressive decoding phase where per-token quantization is natural. Prefill uses different quantization strategies (per-block is more suitable there).
- **Hardest reasoning tasks show measurable delta.** While most benchmarks show <0.5% accuracy change, competition-level math (AIME-25) can show up to ~2.5% degradation. Applications requiring absolute peak reasoning accuracy on the hardest problems may need to evaluate this tradeoff.
- **Requires SGLang-FluentLLM fork.** The optimized kernels are currently available in the Meituan fork, not upstream SGLang. Integration with other serving frameworks requires porting the custom CUDA kernels.

## Reference

**Paper:** [SnapMLA: Efficient Long-Context MLA Decoding via Hardware-Aware FP8 Quantized Pipelining](https://arxiv.org/abs/2602.10718v1) (Zhang et al., 2026). Key sections: Section 3.1 for RoPE sensitivity analysis and per-token quantization rationale, Section 3.2 for the PV scale-fusion pipeline reconstruction algorithm, Algorithm 1 for the complete dual-warp-group pipelined kernel, and Appendix B for online softmax fusion with scale integration.

**Code:** [github.com/meituan-longcat/SGLang-FluentLLM](https://github.com/meituan-longcat/SGLang-FluentLLM) — SGLang fork with SnapMLA kernels, FlashMLA FP8 variants, and DeepGEMM integration.