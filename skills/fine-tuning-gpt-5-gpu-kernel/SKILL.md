---
name: "fine-tuning-gpt-5-gpu-kernel"
description: "Generate optimized GPU kernels in Triton from PyTorch reference code using the Makora RL-based iterative refinement workflow. Applies tool-augmented kernel generation with correctness validation and performance benchmarking. Use when: 'write a Triton kernel for this PyTorch op', 'optimize this GPU kernel', 'generate a fast Triton implementation of matrix multiply', 'convert this PyTorch module to a custom Triton kernel', 'speed up this CUDA operation with Triton', 'benchmark my kernel against TorchInductor'."
---

# GPU Kernel Generation with Makora-Style Iterative Refinement

This skill enables Claude to generate high-performance Triton GPU kernels from PyTorch reference implementations, following the Makora methodology from "Fine-Tuning GPT-5 for GPU Kernel Generation" (arXiv:2602.11000). Rather than producing a single kernel attempt, Claude applies an iterative tool-augmented workflow: generate a candidate kernel, validate correctness against reference outputs, measure performance against TorchInductor baselines, and refine using profiling feedback and structural patterns from successful prior kernels. This approach achieved 97.4% correctness and 2.12x geometric mean speedup over TorchInductor on KernelBench.

## When to Use

- When the user provides a PyTorch operation (nn.Module, functional op, or tensor expression) and asks for an equivalent Triton kernel
- When the user wants to replace a TorchInductor-compiled path with a hand-tuned Triton kernel for better performance
- When optimizing a specific computational pattern: matrix multiplication, convolution, reduction, softmax, attention, normalization, or element-wise fusion
- When the user asks to benchmark a custom kernel against PyTorch's default compiled output
- When debugging a Triton kernel that compiles but produces numerically incorrect results
- When the user needs to fuse multiple sequential PyTorch operations into a single Triton kernel to reduce memory bandwidth

## Key Technique: RL-Trained Tool-Augmented Kernel Generation

The Makora approach departs from supervised fine-tuning (which suffers from scarce high-quality GPU kernel training data and compiler biases in synthetic examples) by using Reinforcement Learning from Verifiable Rewards (RLVR). The reward function has two components: a binary correctness gate (zero reward for kernels that fail compilation or produce wrong outputs) and a logistic-normalized performance score calibrated so that matching TorchInductor yields ~0.5 reward and exceeding it approaches 1.0. This incentivizes the model to first get correctness right, then optimize.

The critical practical insight is the **iterative tool-augmented workflow**. During generation, the model has access to four tools: a kernel evaluator (compile, run, benchmark), a kernel search tool (retrieve high-performing prior candidates for the same problem pattern), a web search tool (look up optimization strategies), and a profiler (collect hardware utilization metrics). The model generates a candidate, evaluates it, reads the feedback, and refines — up to three iterations. This mirrors how expert kernel developers work: write, test, profile, fix.

Training problem curation is equally important. Problems are stratified into difficulty levels (L0 trivial through L5 expert), deduplicated using embedding similarity, filtered to 1-1000ms runtime ranges, and cluster-balanced via inverse-log weighting. This prevents the model from overfitting to easy patterns and ensures exposure to complex fusion, tiling, and synchronization challenges.

## Step-by-Step Workflow

1. **Extract the PyTorch reference implementation.** Identify the exact operation: input tensors (shapes, dtypes), computation performed, and expected output. If the user provides an `nn.Module`, isolate the `forward()` method. Write a standalone PyTorch function that takes explicit tensor inputs and returns tensor outputs.

2. **Classify the kernel pattern and difficulty.** Determine which category the operation falls into: element-wise (L0-L1), reduction (L2), matrix multiplication or convolution (L3), multi-stage compute like attention or normalization (L4), or complex fusion requiring multiple passes (L5). This guides optimization strategy selection.

3. **Generate the initial Triton kernel.** Write a `@triton.jit`-decorated kernel function with:
   - Explicit grid and block dimension calculations
   - `tl.program_id(axis=0)` for work distribution
   - Appropriate `tl.load` / `tl.store` with masking for boundary conditions
   - A Python wrapper function that computes the grid, allocates the output tensor, and launches the kernel

4. **Build a correctness test harness.** Create test inputs matching the problem specification (same shapes, dtypes, value ranges). Run both the PyTorch reference and the Triton kernel. Compare outputs with `torch.allclose(ref, out, atol=1e-3, rtol=1e-3)`. Test with at least 3 different input sizes to catch boundary bugs.

5. **Fix compilation and correctness errors first.** If the kernel fails to compile, read the Triton error message carefully — common issues are mismatched tensor dimensions in `tl.dot`, incorrect `constexpr` usage, or missing `tl.trans()`. If it compiles but produces wrong results, check: (a) pointer arithmetic and stride calculations, (b) reduction accumulator initialization, (c) mask boundaries, (d) data type promotions (fp16 accumulation losing precision — use `tl.float32` accumulators).

6. **Benchmark against TorchInductor baseline.** Compile the PyTorch reference with `torch.compile()` and measure median runtime over 100 iterations (with 3 warmup runs). Measure the Triton kernel the same way. Report `speedup = t_torch / t_triton`.

7. **Profile and identify bottlenecks.** If speedup < 1.0, use `triton.testing.do_bench` with `return_mode='median'` and inspect: Are you memory-bound (increase tile size, ensure coalesced access)? Compute-bound (use `tl.dot` for matrix ops, exploit tensor cores)? Launch-overhead-bound (fuse multiple operations into one kernel)?

8. **Apply targeted optimizations based on the kernel pattern:**
   - **Element-wise ops**: Maximize `BLOCK_SIZE` (1024-4096), ensure contiguous memory access, fuse chains of element-wise ops into a single kernel
   - **Reductions**: Use two-phase reduction (per-block partial sums, then final reduction), leverage `tl.sum` / `tl.max` intrinsics
   - **Matrix multiply**: Tile with `BLOCK_M`, `BLOCK_N`, `BLOCK_K` (start with 64/64/32), use `tl.dot` for tensor core utilization, implement software pipelining with `tl.load(..., eviction_policy='evict_first')`
   - **Attention**: Fuse Q@K^T, scaling, masking, softmax, and V multiplication into a single kernel (Flash Attention pattern), use online softmax to avoid materializing the full attention matrix
   - **Normalization**: Fuse mean/variance computation with the normalization and optional affine transform

9. **Re-evaluate after each optimization.** Run correctness tests again (optimizations frequently introduce bugs). Measure speedup. If correctness breaks, revert the last change and try an alternative approach. Iterate up to 3 refinement cycles.

10. **Deliver the final kernel with documentation.** Provide: the Triton kernel code, the Python wrapper, a benchmark script, and a brief explanation of which optimizations were applied and why. Include the measured speedup over TorchInductor.

## Concrete Examples

**Example 1: Fused SiLU + Multiply (Element-wise Fusion)**

User: "Write a Triton kernel that fuses SiLU activation with element-wise multiplication. In PyTorch it's `x * torch.sigmoid(x) * gate`."

Approach:
1. Identify pattern: element-wise fusion (L1), two operations that can share a single memory pass
2. Generate kernel with single load of `x` and `gate`, compute SiLU in registers, store result
3. Test against PyTorch reference

Output:
```python
import triton
import triton.language as tl
import torch

@triton.jit
def fused_silu_mul_kernel(
    x_ptr, gate_ptr, out_ptr,
    n_elements,
    BLOCK_SIZE: tl.constexpr,
):
    pid = tl.program_id(0)
    offsets = pid * BLOCK_SIZE + tl.arange(0, BLOCK_SIZE)
    mask = offsets < n_elements

    x = tl.load(x_ptr + offsets, mask=mask)
    gate = tl.load(gate_ptr + offsets, mask=mask)

    # SiLU(x) * gate = x * sigmoid(x) * gate
    silu = x * tl.sigmoid(x.to(tl.float32)).to(x.dtype)
    result = silu * gate

    tl.store(out_ptr + offsets, result, mask=mask)

def fused_silu_mul(x: torch.Tensor, gate: torch.Tensor) -> torch.Tensor:
    assert x.shape == gate.shape and x.is_contiguous() and gate.is_contiguous()
    out = torch.empty_like(x)
    n = x.numel()
    grid = lambda meta: (triton.cdiv(n, meta['BLOCK_SIZE']),)
    fused_silu_mul_kernel[grid](x, gate, out, n, BLOCK_SIZE=1024)
    return out

# Correctness test
x = torch.randn(4096, 4096, device='cuda', dtype=torch.float16)
gate = torch.randn_like(x)
ref = x * torch.sigmoid(x.float()).half() * gate
out = fused_silu_mul(x, gate)
assert torch.allclose(ref, out, atol=1e-2, rtol=1e-2), "Correctness check failed"
```

**Example 2: Tiled Matrix Multiplication (Compute-Bound)**

User: "Generate a Triton matmul kernel that beats torch.compile for (2048, 2048) @ (2048, 2048) in float16."

Approach:
1. Classify as L3 (matrix multiply), requires tiling and tensor core utilization
2. Implement blocked matmul with BLOCK_M=128, BLOCK_N=128, BLOCK_K=32
3. Use `tl.dot` for tensor core acceleration, accumulate in float32
4. Benchmark against `torch.compile(torch.matmul)`

Output:
```python
import triton
import triton.language as tl
import torch

@triton.jit
def matmul_kernel(
    a_ptr, b_ptr, c_ptr,
    M, N, K,
    stride_am, stride_ak,
    stride_bk, stride_bn,
    stride_cm, stride_cn,
    BLOCK_M: tl.constexpr, BLOCK_N: tl.constexpr, BLOCK_K: tl.constexpr,
):
    pid_m = tl.program_id(0)
    pid_n = tl.program_id(1)

    offs_m = pid_m * BLOCK_M + tl.arange(0, BLOCK_M)
    offs_n = pid_n * BLOCK_N + tl.arange(0, BLOCK_N)
    offs_k = tl.arange(0, BLOCK_K)

    a_ptrs = a_ptr + offs_m[:, None] * stride_am + offs_k[None, :] * stride_ak
    b_ptrs = b_ptr + offs_k[:, None] * stride_bk + offs_n[None, :] * stride_bn

    acc = tl.zeros((BLOCK_M, BLOCK_N), dtype=tl.float32)
    for k in range(0, K, BLOCK_K):
        a = tl.load(a_ptrs, mask=(offs_m[:, None] < M) & (offs_k[None, :] < K))
        b = tl.load(b_ptrs, mask=(offs_k[:, None] < K) & (offs_n[None, :] < N))
        acc += tl.dot(a, b)
        a_ptrs += BLOCK_K * stride_ak
        b_ptrs += BLOCK_K * stride_bk
        offs_k += BLOCK_K

    c = acc.to(tl.float16)
    c_ptrs = c_ptr + offs_m[:, None] * stride_cm + offs_n[None, :] * stride_cn
    mask = (offs_m[:, None] < M) & (offs_n[None, :] < N)
    tl.store(c_ptrs, c, mask=mask)

def triton_matmul(a: torch.Tensor, b: torch.Tensor) -> torch.Tensor:
    M, K = a.shape
    K2, N = b.shape
    assert K == K2
    c = torch.empty((M, N), device=a.device, dtype=a.dtype)
    grid = lambda meta: (triton.cdiv(M, meta['BLOCK_M']), triton.cdiv(N, meta['BLOCK_N']))
    matmul_kernel[grid](
        a, b, c, M, N, K,
        a.stride(0), a.stride(1), b.stride(0), b.stride(1), c.stride(0), c.stride(1),
        BLOCK_M=128, BLOCK_N=128, BLOCK_K=32,
    )
    return c

# Benchmark
a = torch.randn(2048, 2048, device='cuda', dtype=torch.float16)
b = torch.randn(2048, 2048, device='cuda', dtype=torch.float16)
ref = torch.matmul(a, b)
out = triton_matmul(a, b)
assert torch.allclose(ref, out, atol=1e-1, rtol=1e-1), "Correctness failed"

# Measure speedup
t_torch = triton.testing.do_bench(lambda: torch.matmul(a, b))
t_triton = triton.testing.do_bench(lambda: triton_matmul(a, b))
print(f"Speedup: {t_torch / t_triton:.2f}x")
```

**Example 3: Fused LayerNorm (Multi-Stage Reduction)**

User: "My model spends a lot of time on LayerNorm. Write a Triton kernel that fuses the mean, variance, normalization, and affine transform."

Approach:
1. Classify as L4 (multi-stage compute: two reductions + normalization + affine)
2. Compute mean and variance in a single pass using Welford's algorithm
3. Fuse normalization and affine transform into the same kernel to avoid extra memory reads
4. Each program instance handles one row of the input

Output:
```python
@triton.jit
def layernorm_kernel(
    x_ptr, w_ptr, b_ptr, out_ptr,
    N,  # number of columns (normalized dimension)
    eps,
    stride,
    BLOCK_SIZE: tl.constexpr,
):
    row = tl.program_id(0)
    x_row_ptr = x_ptr + row * stride
    out_row_ptr = out_ptr + row * stride

    # Load row and compute mean
    offs = tl.arange(0, BLOCK_SIZE)
    mask = offs < N
    x = tl.load(x_row_ptr + offs, mask=mask, other=0.0).to(tl.float32)

    mean = tl.sum(x, axis=0) / N
    centered = x - mean
    var = tl.sum(centered * centered, axis=0) / N
    rstd = 1.0 / tl.sqrt(var + eps)

    # Normalize and apply affine
    w = tl.load(w_ptr + offs, mask=mask)
    b = tl.load(b_ptr + offs, mask=mask)
    normed = centered * rstd
    out = normed * w + b

    tl.store(out_row_ptr + offs, out, mask=mask)
```

## Best Practices

- **Do:** Always validate correctness before optimizing performance. A fast wrong kernel is useless. Use `atol=1e-3, rtol=1e-3` for float32 and `atol=1e-2, rtol=1e-2` for float16.
- **Do:** Accumulate in float32 even when inputs are float16. Precision loss in reduction accumulators is the most common source of numerical bugs in Triton kernels.
- **Do:** Start with conservative tile sizes (BLOCK_SIZE=1024 for 1D, 64x64 for 2D) and tune only after correctness is confirmed. Autotuning with `@triton.autotune` over a grid of configs is preferred over manual tuning.
- **Do:** Fuse sequential operations into a single kernel whenever they share the same iteration pattern. Each kernel launch has overhead; each extra global memory round-trip costs bandwidth.
- **Avoid:** Generating "baseline kernels" that just call `torch.xxx` under the hood — this is a known reward-hacking pattern. The kernel must contain actual Triton computation.
- **Avoid:** Assuming hardware features like Tensor Memory Accelerators exist on all GPUs. Write kernels that are correct across GPU generations and use `tl.dot` (which maps to tensor cores when available) rather than hardware-specific intrinsics.
- **Avoid:** Over-tiling small problems. If the total work is smaller than a single block, a complex tiling scheme adds overhead. Match the parallelism to the problem size.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `triton.CompilationError` | Invalid Triton IR (wrong types in `tl.dot`, non-constexpr block sizes) | Ensure block dims are `tl.constexpr`, operands to `tl.dot` are 2D with matching inner dim |
| Correct shape, wrong values | Accumulator overflow in fp16, incorrect stride math, or missing mask | Switch accumulators to fp32, print pointer offsets for first block, check `mask` covers boundaries |
| `CUDA out of memory` | Tile size too large or output buffer allocated incorrectly | Reduce BLOCK_SIZE, verify output tensor shape matches expected dimensions |
| Kernel runs but slower than PyTorch | Uncoalesced memory access, excessive register pressure, or launch overhead dominates | Ensure innermost dimension is contiguous in memory, reduce tile size to lower register use, fuse more work per kernel |
| `Correctness passes on small inputs, fails on large` | Off-by-one in grid calculation or missing mask on final block | Use `triton.cdiv(N, BLOCK_SIZE)` for grid, always mask loads/stores with `offsets < N` |

## Limitations

- Triton kernels are NVIDIA-focused. AMD ROCm support via Triton exists but is less mature; performance characteristics differ significantly.
- The Makora paper's 2.12x speedup was measured on specific hardware (likely A100/H100). Speedup ratios do not transfer directly across GPU architectures — always benchmark on the target device.
- Some operations (sparse computation, irregular memory access patterns, dynamic shapes changing per-call) are poorly suited for Triton's block-structured programming model. CUDA C++ may be necessary for these.
- Triton's `tl.dot` requires tile dimensions to be multiples of 16 for tensor core utilization. Non-aligned dimensions require padding or fallback to scalar math.
- This workflow produces single-kernel optimizations. System-level gains (operator fusion across an entire model graph, memory planning, overlap of compute and communication) require compiler-level tooling like TorchInductor or custom graph passes.
- The iterative refinement approach assumes a working CUDA environment with Triton installed. It cannot generate or validate kernels without GPU access.

## Reference

[Fine-Tuning GPT-5 for GPU Kernel Generation](https://arxiv.org/abs/2602.11000v1) — Tehrani et al., 2026. Focus on Section 3 (Makora environment and tools), Section 4 (reward function design with correctness gating and logistic-normalized performance), Section 5.2 (reward hacking taxonomy — six failure modes to avoid), and Table 1 (KernelBench results showing the iterative agent achieves 97.4% correctness with 2.12x speedup). The training problem curation in Section 3.3 (difficulty stratification L0-L5) is useful for understanding which kernel patterns are hardest for LLMs.