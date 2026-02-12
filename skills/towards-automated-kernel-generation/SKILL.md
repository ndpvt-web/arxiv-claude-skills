---
name: "towards-automated-kernel-generation"
description: "Automate GPU kernel generation and optimization using LLM-driven agentic workflows with profiling feedback loops. Use when user asks to 'write a CUDA kernel', 'optimize a Triton kernel', 'generate a GPU kernel for this operation', 'speed up this PyTorch operator with a custom kernel', 'convert this PyTorch op to Triton', or 'profile and optimize my kernel'."
---

# LLM-Driven Automated Kernel Generation

This skill enables Claude to generate, optimize, and iteratively refine GPU compute kernels (CUDA, Triton, and other DSLs) by applying the structured taxonomy and agentic optimization workflows from the survey "Towards Automated Kernel Generation in the Era of LLMs" (Yu et al., 2026). Rather than one-shot code generation, this skill implements a profiling-guided iterative refinement loop where kernel code is generated, compiled, profiled, and refined based on hardware performance feedback — mirroring the Plan-Code-Debug multi-agent pattern that the survey identifies as the most robust approach.

## When to Use

- When the user asks to write a CUDA or Triton kernel for a specific operation (matmul, attention, reduction, elementwise, etc.)
- When the user wants to convert a PyTorch operator into a custom high-performance kernel
- When the user has an existing kernel that runs correctly but needs performance optimization
- When the user asks to optimize memory access patterns, tiling strategies, or thread-block configurations in GPU code
- When the user needs a kernel that targets a specific GPU architecture (e.g., H100, A100, RTX 4090) and wants hardware-aware optimization
- When the user wants to understand why a kernel is slow and get profiling-guided improvement suggestions

## Key Technique

The survey categorizes LLM-driven kernel generation into two paradigms: **LLM4Kernel** (direct single-pass generation) and **Agent4Kernel** (iterative agentic optimization). The critical insight is that kernel generation differs fundamentally from general code generation — functional correctness is necessary but insufficient; kernels must also satisfy strict efficiency constraints tied to hardware execution characteristics. This makes kernel development closer to compiler optimization than software engineering.

The most effective approach identified is the **agentic optimization loop** with four structural dimensions: (1) a search strategy (iterative refinement, population-based evolution, or test-time scaling), (2) external memory of high-quality kernel patterns and hardware-specific knowledge, (3) hardware profiling integration that converts low-level metrics into natural-language optimization suggestions, and (4) multi-agent role decomposition where separate planning, coding, and debugging phases operate in sequence. Systems like PRAGMA parse profiling output into interpretable diagnostics; PEAK uses stepwise modular refinement; and KernelFalcon coordinates manager/worker agents for complex kernels.

The practical workflow this enables: generate a kernel from a high-level specification, compile and test it for correctness, profile it against hardware metrics, translate profiling bottlenecks into specific code-level suggestions (e.g., "L2 cache hit rate is 34% — apply swizzled memory access patterns"), then generate a refined version. This loop repeats until performance targets are met or diminishing returns are observed.

## Step-by-Step Workflow

1. **Extract the kernel specification** from the user's request. Identify: the mathematical operation (e.g., batched matrix multiply, fused attention, layernorm), input/output tensor shapes and dtypes, the target hardware (GPU model, compute capability), and the target DSL (Triton preferred for portability, CUDA for maximum control).

2. **Establish a correctness baseline.** If the user provides a PyTorch reference implementation, use it directly. If not, write a PyTorch reference that computes the expected output. This reference serves as the ground truth for all correctness checks.

3. **Generate the initial kernel** using structured prompting. Include in the prompt: (a) the operation semantics, (b) tensor layout and memory access patterns, (c) target hardware specs (warp size, shared memory size, L2 cache size, SM count), and (d) known optimization patterns relevant to the operation (tiling for matmul, online softmax for attention, vectorized loads for elementwise ops).

4. **Write a test harness** that compiles the kernel, runs it on representative inputs, checks numerical correctness against the reference (using appropriate tolerances for float16/bfloat16), and measures wall-clock time via CUDA events or `triton.testing.do_bench`.

5. **Run the kernel and collect diagnostics.** Capture: compilation success/failure and error messages, numerical correctness results, runtime in milliseconds, and if available, profiling metrics (achieved bandwidth, compute utilization, L2 cache hit rate, occupancy). Parse errors into structured categories: syntax error, type mismatch, out-of-bounds access, numerical divergence, or performance regression.

6. **Translate profiling feedback into optimization directives.** Map hardware metrics to specific code changes:
   - Low memory bandwidth utilization → coalesce global memory accesses, use vectorized loads (`tl.load` with `mask` and contiguous pointers in Triton)
   - Low compute utilization → increase tile size to expose more parallelism, check for unnecessary synchronization barriers
   - Low L2 cache hit rate → apply swizzled access patterns, reorder loop nesting
   - Low occupancy → reduce shared memory per block or reduce register pressure
   - High instruction replay → fix bank conflicts in shared memory

7. **Generate a refined kernel** incorporating the specific optimization directives from step 6. Preserve all correctness-critical logic; only modify performance-related code paths. Include comments explaining each optimization choice.

8. **Iterate steps 5-7** up to 3-5 rounds. Track the speedup achieved at each iteration. Stop when: (a) the target speedup is reached, (b) two consecutive iterations yield less than 5% improvement, or (c) a correctness regression is detected (revert to the last correct version).

9. **Report results** to the user with a summary: final kernel code, speedup over the PyTorch baseline, key optimizations applied, and any remaining bottlenecks that would require lower-level tuning (inline PTX, warp-level primitives, etc.).

10. **Document hardware assumptions.** Note which optimizations are architecture-specific (e.g., TMA for Hopper, async copy for Ampere) so the user understands portability implications.

## Concrete Examples

**Example 1: Convert a PyTorch softmax to a Triton kernel**

User: "Write a Triton kernel for softmax along the last dimension. Input shape is (batch, 4096) in float16 on an A100."

Approach:
1. Write PyTorch reference: `torch.softmax(x, dim=-1)` with shape `(batch, 4096)`, dtype `float16`.
2. Generate initial Triton kernel using the online softmax algorithm (two-pass: max reduction + normalize, fused into tiles):
```python
@triton.jit
def softmax_kernel(output_ptr, input_ptr, n_cols, BLOCK_SIZE: tl.constexpr):
    row_idx = tl.program_id(0)
    col_offsets = tl.arange(0, BLOCK_SIZE)
    mask = col_offsets < n_cols
    row_start = row_idx * n_cols
    # Load row into SRAM
    x = tl.load(input_ptr + row_start + col_offsets, mask=mask, other=-float('inf'))
    # Online softmax: subtract max for numerical stability
    x_max = tl.max(x, axis=0)
    x = x - x_max
    numerator = tl.exp(x)
    denominator = tl.sum(numerator, axis=0)
    result = numerator / denominator
    tl.store(output_ptr + row_start + col_offsets, result, mask=mask)
```
3. Set `BLOCK_SIZE = 4096` (power-of-2 >= n_cols) so entire row fits in one block.
4. Benchmark: compare against `torch.softmax` and report throughput in GB/s.
5. If bandwidth utilization is low, try `tl.load` with `eviction_policy='evict_last'` and ensure the grid covers all rows.

Output: Complete Triton kernel with launch wrapper, benchmark script, and expected ~1.2-1.5x speedup over PyTorch on A100 for this shape.

**Example 2: Optimize an existing slow CUDA matmul kernel**

User: "My CUDA matmul kernel computes C = A @ B for (2048, 2048) float32 matrices but only achieves 8 TFLOPS on an H100. Can you help?"

Approach:
1. Read the user's kernel code. Identify the current tiling strategy, shared memory usage, and thread mapping.
2. Establish baseline: 8 TFLOPS vs. H100 peak of ~60 TFLOPS (FP32 TF32) = ~13% utilization.
3. Diagnose likely bottlenecks from code inspection:
   - No shared memory tiling → global memory bandwidth bound
   - Small thread block → low occupancy
   - No vectorized loads → wasted memory transactions
4. Generate optimized version with:
   - 128x128 output tile per thread block, 32x32 per warp
   - Double-buffered shared memory to overlap compute and loads
   - `float4` vectorized global memory loads
   - Swizzled shared memory layout to avoid bank conflicts
5. Test correctness against `torch.matmul`, then benchmark.
6. If still below target, apply thread-level tiling (each thread computes an 8x8 output fragment) and register blocking.

Output: Optimized CUDA kernel, profiling comparison table showing before/after TFLOPS, and explanation of each optimization.

**Example 3: Generate a fused attention kernel from scratch**

User: "I need a fused multi-head attention kernel in Triton for sequence length 2048, head dim 64, float16 on A100."

Approach:
1. Reference: `torch.nn.functional.scaled_dot_product_attention`.
2. Implement FlashAttention-style tiling in Triton:
   - Outer loop over query blocks (BLOCK_M = 128)
   - Inner loop over key/value blocks (BLOCK_N = 64)
   - Maintain running max and sum-of-exponentials for online softmax
   - Accumulate output in float32 registers, cast to float16 on store
3. Key optimizations for A100:
   - Use `tl.dot` for tensor core utilization (requires aligned shapes)
   - Causal masking via `tl.where` with block-level early exit
   - Epilogue: rescale accumulated output by final softmax normalization
4. Verify numerical correctness within `atol=1e-2` for float16 (expected due to reduced precision in online accumulation).
5. Benchmark against PyTorch SDPA and report runtime at sequence lengths [512, 1024, 2048, 4096].

Output: Complete fused attention kernel, test script, benchmark results, and notes on where it falls short of FlashAttention-2 (likely in warp-level pipelining and TMA usage).

## Best Practices

- **Do:** Always generate a correctness test alongside the kernel. A fast but wrong kernel is useless — verify numerics before reporting any speedup.
- **Do:** Include hardware specifications in the generation prompt. The same operation requires different tiling, block sizes, and memory strategies on A100 vs. H100 vs. consumer GPUs.
- **Do:** Start with the simplest correct kernel, then optimize incrementally. Each optimization step should be independently verifiable.
- **Do:** Use Triton as the default target unless the user specifically requests CUDA. Triton is more portable, composable, and amenable to LLM generation due to its higher abstraction level.
- **Avoid:** Generating kernels with hardcoded shapes. Use `tl.constexpr` parameters and runtime grid calculations so the kernel generalizes across input sizes.
- **Avoid:** Applying optimizations blindly. A shared-memory tiling strategy that helps matmul may hurt an elementwise kernel that is already bandwidth-bound. Match the optimization to the bottleneck.
- **Avoid:** Claiming specific speedup numbers without measurement. Always frame expected performance as estimates pending profiling.

## Error Handling

| Error Category | Symptoms | Resolution |
|---|---|---|
| **Compilation failure** | Triton/CUDA compiler error, undefined symbols | Check for syntax errors, mismatched types, invalid `tl.constexpr` usage. Provide the exact compiler error to the refinement step. |
| **Numerical incorrectness** | Output diverges from reference beyond tolerance | Common causes: race conditions in reductions, incorrect masking at tile boundaries, float16 overflow. Switch accumulation to float32, verify mask logic. |
| **Performance regression** | New version slower than previous | Revert to last known-good version. Profile to identify whether the regression is from reduced occupancy, increased register pressure, or memory bank conflicts. |
| **Out-of-memory** | CUDA OOM during kernel launch | Reduce shared memory allocation per block, decrease tile size, or reduce grid dimensions. Check for accidental allocation inside the kernel. |
| **Hardware mismatch** | Kernel uses features unavailable on target GPU | Check compute capability. TMA requires sm_90+, async copy requires sm_80+. Fall back to synchronous alternatives for older architectures. |

## Limitations

- **No real execution environment.** Claude cannot compile or run kernels. All profiling feedback must come from the user or external tooling. The iterative refinement loop requires the user to run benchmarks and report results back.
- **Architecture-specific tuning.** Kernels optimized for one GPU family (e.g., Hopper) may not transfer to another (e.g., Ampere). Claude can reason about architectural differences but cannot predict exact performance without profiling data.
- **Complex memory hierarchies.** Low-level optimizations like warp shuffle, inline PTX, or TMA descriptor setup require precise register-level reasoning that is error-prone for LLMs. Flag these for human review.
- **Non-NVIDIA hardware.** This skill primarily targets NVIDIA GPUs via CUDA/Triton. AMD (ROCm/HIP) and Intel (SYCL/oneAPI) kernels follow similar principles but have different intrinsics and memory models that may require adaptation.
- **Kernel fusion boundaries.** Deciding which operations to fuse into a single kernel (vs. keeping separate) is an architectural decision that depends on the full computation graph, not just individual operations.

## Reference

**Paper:** Yu et al., "Towards Automated Kernel Generation in the Era of LLMs" (arXiv:2601.15727v2, 2026). Read for: the four-dimensional taxonomy of agentic kernel optimization (search strategy, external memory, hardware profiling, multi-agent orchestration) and the survey of benchmarks (KernelBench, TRITONBENCH, MultiKernelBench) for evaluating generated kernels.

**Repository:** [github.com/flagos-ai/awesome-LLM-driven-kernel-generation](https://github.com/flagos-ai/awesome-LLM-driven-kernel-generation) — Maintained catalog of systems, datasets, and benchmarks in this space.