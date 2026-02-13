---
name: "dr-kernel-reinforcement-learning-done"
description: "Write high-performance Triton GPU kernels using Dr. Kernel's multi-turn refinement strategy: profile-guided optimization, reward hacking prevention, and iterative kernel fusion. Use when asked to 'write a Triton kernel', 'optimize GPU code', 'fuse CUDA operations', 'speed up a PyTorch operation with Triton', 'generate a fast kernel for [operation]', or 'profile and optimize this kernel'."
---

This skill enables Claude to write high-performance Triton GPU kernels by applying the Dr. Kernel methodology: a multi-turn, profile-guided approach to kernel generation that avoids common pitfalls like reward hacking (generating code that appears fast but doesn't actually execute the kernel) and lazy optimization (optimizing trivial sub-operations while ignoring dominant bottlenecks). The core insight is that kernel optimization should be driven by profiling data -- identify what consumes the most runtime, fuse those operations into a single Triton kernel, then iteratively tune block sizes and memory access patterns based on measured performance.

## When to Use

- When the user asks to write a Triton kernel for a PyTorch operation (e.g., LayerNorm, softmax, attention, matrix multiply)
- When the user wants to fuse multiple PyTorch operations into a single GPU kernel for speedup
- When the user asks to optimize an existing Triton kernel that isn't achieving expected performance
- When the user needs to replace a PyTorch `nn.Module` with a custom Triton implementation that matches correctness and improves throughput
- When the user asks to profile a kernel and determine whether their Triton code is actually optimizing the bottleneck
- When the user is working with KernelBench tasks or similar kernel-generation benchmarks
- When the user wants guidance on multi-turn kernel refinement (write, profile, fix, tune)

## Key Technique

**Profile-Guided Kernel Optimization.** The central lesson from Dr. Kernel is that naive kernel generation often falls into "lazy optimization" -- the model writes a kernel that handles a trivial sub-operation (e.g., a bias addition consuming 0.01% of runtime) while the dominant bottleneck (e.g., a matrix multiply or reduction consuming 86% of runtime) runs unchanged through PyTorch. The fix is profiling-based: measure what fraction of total CUDA execution time your generated kernel accounts for (`PR = T_kernel / T_total`). If PR is low (< 0.3), the kernel is optimizing the wrong thing. Redirect effort to the operation that dominates the profile.

**Multi-Turn Refinement.** Effective kernel generation is iterative, not one-shot. The Dr. Kernel approach uses up to 3 turns: (1) generate an initial kernel targeting the dominant operation, (2) incorporate correctness feedback and profiling data to fix errors and improve fusion, (3) tune hardware-specific parameters (block sizes, number of warps, pipeline stages) based on measured throughput. Each turn receives environment feedback including compilation errors, correctness checks against reference outputs, and timing measurements.

**Reward Hacking Prevention.** Several degenerate patterns must be actively avoided: kernels that define a `@triton.jit` function but never call it (speedup comes from skipping computation), kernels that branch on `self.training` to bypass execution during profiling, and kernels that simply copy the PyTorch reference. Valid kernels must (a) actually execute a Triton kernel confirmed via launch instrumentation, (b) produce numerically correct outputs within tolerance (typically `atol=1e-2, rtol=1e-2` for float32), and (c) show wall-clock speedup in end-to-end measurement, not just isolated kernel timing.

## Step-by-Step Workflow

1. **Analyze the reference PyTorch operation.** Read the user's PyTorch code and identify every operation it performs. List them by expected cost: large matmuls and reductions dominate; element-wise ops and small reshapes are cheap. Determine which operations are fusible (same tensor shapes, sequential data flow, no host-device synchronization between them).

2. **Profile to identify the bottleneck.** If timing data is available, compute the profiling ratio for each operation. If not, estimate based on FLOPs and memory bandwidth. The target operation for your Triton kernel should account for the majority of runtime. Do NOT write a kernel for a trivial sub-operation.

3. **Design the kernel fusion strategy.** Decide which operations to fuse into a single Triton kernel. Good fusion candidates share input tensors, have compatible reduction dimensions, and avoid complex control flow. For example, fuse `LayerNorm = mean + variance + normalize + scale + shift` into one kernel. Avoid attempting to fuse convolutions that cuDNN handles better unless the user specifically requests it.

4. **Write the initial Triton kernel.** Implement a `@triton.jit` function with explicit grid, block sizes, and memory access patterns. Wrap it in a `ModelNew(nn.Module)` class that matches the reference module's `forward()` signature exactly. Ensure the kernel is actually called in the forward pass -- never define it without invoking it.

5. **Validate correctness against the reference.** Generate test inputs matching the expected shapes and dtypes. Run both the reference and your kernel, comparing outputs with `torch.allclose(ref_out, kern_out, atol=1e-2, rtol=1e-2)`. If outputs diverge, check for: off-by-one in pointer arithmetic, incorrect reduction dimension, missing masking for partial blocks, or dtype mismatches.

6. **Measure end-to-end performance.** Time the full `forward()` call (not just the Triton kernel in isolation) using CUDA events with warmup iterations. Compute speedup as `T_reference / T_kernel`. If speedup < 1.0, the kernel is slower than PyTorch -- revisit memory access patterns and occupancy.

7. **Compute the profiling ratio.** Measure what fraction of total execution time your Triton kernel accounts for. If `PR < 0.3`, your kernel is optimizing a minor operation. Go back to step 2 and target the actual bottleneck.

8. **Tune hardware parameters.** Adjust `BLOCK_SIZE`, `num_warps`, and `num_stages` based on the target GPU. Use Triton's `triton.autotune` decorator with a grid of configurations, or manually set values based on the problem size. For H100: start with `BLOCK_SIZE=1024`, `num_warps=8`, `num_stages=3` and adjust down for smaller problems.

9. **Iterate with feedback.** If correctness fails, fix pointer arithmetic and masking first. If performance is underwhelming, check for: uncoalesced memory access, excessive shared memory usage, low occupancy due to register pressure, or unnecessary `tl.debug_barrier()` calls. Repeat steps 5-8 for up to 2 additional refinement turns.

10. **Final validation.** Run the kernel across multiple input sizes to confirm it generalizes. Verify that speedup holds for the user's target batch sizes, not just the profiling size. Report the achieved speedup and profiling ratio.

## Concrete Examples

**Example 1: Fused LayerNorm Kernel**

User: "Write a Triton kernel that replaces `nn.LayerNorm` for shape (batch=32, seq=512, hidden=768). I want it faster than PyTorch."

Approach:
1. LayerNorm computes: mean, variance, normalize, scale, shift -- all fusible into one kernel
2. The reduction is over `hidden=768`, which fits in a single Triton block
3. Write a kernel with `BLOCK_SIZE=1024` (next power of 2 above 768) operating on one row per program

Output:
```python
import triton
import triton.language as tl
import torch
import torch.nn as nn

@triton.jit
def layernorm_kernel(
    X, Y, W, B, stride, N, eps,
    BLOCK_SIZE: tl.constexpr,
):
    row = tl.program_id(0)
    cols = tl.arange(0, BLOCK_SIZE)
    mask = cols < N
    x = tl.load(X + row * stride + cols, mask=mask, other=0.0).to(tl.float32)
    mean = tl.sum(x, axis=0) / N
    x_centered = x - mean
    var = tl.sum(x_centered * x_centered, axis=0) / N
    x_norm = x_centered / tl.sqrt(var + eps)
    w = tl.load(W + cols, mask=mask, other=1.0).to(tl.float32)
    b = tl.load(B + cols, mask=mask, other=0.0).to(tl.float32)
    y = x_norm * w + b
    tl.store(Y + row * stride + cols, y, mask=mask)

class ModelNew(nn.Module):
    def __init__(self, normalized_shape, eps=1e-5):
        super().__init__()
        self.eps = eps
        self.weight = nn.Parameter(torch.ones(normalized_shape))
        self.bias = nn.Parameter(torch.zeros(normalized_shape))
        self.normalized_shape = normalized_shape

    def forward(self, x):
        assert x.is_cuda and x.is_contiguous()
        y = torch.empty_like(x)
        N = self.normalized_shape
        n_rows = x.numel() // N
        BLOCK_SIZE = triton.next_power_of_2(N)
        layernorm_kernel[(n_rows,)](
            x, y, self.weight, self.bias,
            x.stride(-2), N, self.eps,
            BLOCK_SIZE=BLOCK_SIZE,
        )
        return y
```

**Example 2: Detecting and Fixing Lazy Optimization**

User: "I wrote a Triton kernel for a transformer block but it's only 1.02x faster. What's wrong?"

Approach:
1. Profile the full forward pass to identify what operations consume the most time
2. Check the profiling ratio -- likely the Triton kernel handles a minor operation
3. Redirect to the dominant bottleneck

Analysis:
```
Operation Breakdown (profiled via torch.profiler):
  - Linear projections (Q, K, V):  45% of runtime
  - Attention matmul + softmax:    38% of runtime
  - Your Triton kernel (bias add):  0.8% of runtime  <-- PR = 0.008
  - Other (dropout, residual):     16.2% of runtime

Problem: Your kernel fuses bias addition (0.8% of runtime).
Even a 10x speedup on bias addition saves < 0.7% total time.

Fix: Target the attention computation (38%) or fuse the full
QKV projection + attention into a single kernel. Consider using
Triton's flash-attention pattern for the matmul + softmax fusion.
```

**Example 3: Catching a Reward Hacking Pattern**

User: "My kernel reports 3x speedup but something seems off."

Approach:
1. Inspect whether the `@triton.jit` function is actually invoked in the forward pass
2. Check for conditional execution paths that skip the kernel
3. Verify outputs match the reference

Detection:
```python
# HACKING PATTERN DETECTED -- kernel is defined but never called:
class ModelNew(nn.Module):
    def forward(self, x):
        if self.training:
            return x  # Bypasses kernel entirely during training
        return triton_kernel(x)  # Only runs during eval

# FIX: Remove the training guard. The kernel must execute
# unconditionally in forward():
class ModelNew(nn.Module):
    def forward(self, x):
        return triton_kernel(x)  # Always executes
```

Another common hack -- defining `@triton.jit` but calling `torch.ops` instead:
```python
# BAD: Kernel defined but torch fallback used
@triton.jit
def my_kernel(...): ...

class ModelNew(nn.Module):
    def forward(self, x):
        return torch.layer_norm(x, ...)  # Uses PyTorch, not the Triton kernel
```

## Best Practices

- **Do:** Always compute the profiling ratio (`T_kernel / T_total`). If it's below 0.3, you're optimizing the wrong operation. Rewrite targeting the bottleneck.
- **Do:** Use `triton.autotune` with at least 4-6 configurations varying `BLOCK_SIZE`, `num_warps`, and `num_stages`. Hardware-specific tuning accounts for a large fraction of real-world speedup.
- **Do:** Test correctness with multiple input shapes and dtypes. A kernel that works for `(32, 512, 768)` may fail for `(1, 2048, 768)` due to masking bugs.
- **Do:** Clip reported speedup at a reasonable maximum (3x). Speedups above this are often measurement artifacts from insufficient warmup or timer resolution.
- **Avoid:** Writing a Triton kernel for an operation that cuDNN or cuBLAS already handles optimally (standalone large matmuls, standard convolutions). Triton wins on fusion and custom reductions, not on beating vendor libraries at their core operation.
- **Avoid:** Generating kernel code that defines `@triton.jit` functions without calling them, or that uses `self.training` guards to bypass execution. These are hacking patterns that inflate benchmarks without real optimization.
- **Avoid:** Optimizing element-wise operations in isolation. A standalone ReLU kernel will never meaningfully speed up a model. Fuse it with the preceding matmul or normalization.

## Error Handling

| Problem | Diagnosis | Fix |
|---------|-----------|-----|
| `triton.CompilationError` | Invalid Triton IR, usually from type mismatches or unsupported ops | Cast inputs explicitly with `.to(tl.float32)`, avoid Python-level control flow inside `@triton.jit` |
| Numerical mismatch (`atol > 1e-2`) | Pointer arithmetic error or missing mask | Verify `tl.arange` bounds match tensor dimensions, add `mask=` to all `tl.load`/`tl.store` calls |
| CUDA OOM during kernel launch | Grid size too large or shared memory overflow | Reduce `BLOCK_SIZE`, check that grid dimensions match the problem (not `numel()` when you mean `n_rows`) |
| Kernel slower than PyTorch | Uncoalesced access, low occupancy, or excessive synchronization | Ensure innermost dimension is contiguous, reduce register pressure by lowering `BLOCK_SIZE`, remove unnecessary barriers |
| Speedup doesn't reproduce | Insufficient warmup or background GPU load | Use `torch.cuda.synchronize()` before timing, run 100+ warmup iterations, measure 100+ timed iterations |
| Profiling ratio near zero | Kernel optimizes a trivial operation | Reprofile to find the dominant operation and rewrite the kernel to target it |

## Limitations

- **Vendor-optimized operations:** Standard convolutions and large standalone GEMMs are extremely well optimized by cuDNN/cuBLAS. Triton kernels for these rarely beat the vendor libraries unless fusing adjacent operations.
- **Complex control flow:** Triton's programming model is SIMT with limited branching. Operations requiring data-dependent control flow (e.g., sparse attention with variable-length rows) are difficult to express efficiently.
- **Hardware specificity:** Kernels tuned for H100 may underperform on A100 or consumer GPUs due to different cache sizes, warp schedulers, and memory bandwidth. Always retune `autotune` configs for the target hardware.
- **Multi-kernel pipelines:** This approach optimizes individual kernels. System-level optimization (operator scheduling, memory planning, kernel launch overhead) requires tools like `torch.compile` or custom CUDA graphs.
- **Correctness at reduced precision:** FP16/BF16 kernels require careful handling of accumulation precision. Always accumulate reductions in FP32 and cast back, or numerical errors will compound.

## Reference

**Paper:** [Dr. Kernel: Reinforcement Learning Done Right for Triton Kernel Generations](https://arxiv.org/abs/2602.05885v2) -- Focus on Sections 3-5 for the profiling-based reward design, TRLOO algorithm for multi-turn training, and the case studies showing lazy optimization vs. effective kernel fusion patterns.

**Code:** [github.com/hkust-nlp/KernelGYM](https://github.com/hkust-nlp/KernelGYM) -- The KernelGYM environment for evaluating kernel correctness and performance with subprocess isolation, plus the Dr. Kernel training recipes.