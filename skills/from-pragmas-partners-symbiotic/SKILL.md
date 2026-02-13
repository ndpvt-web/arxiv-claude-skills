---
name: "from-pragmas-partners-symbiotic"
description: "Agentic High-Level Synthesis (HLS) optimization: autonomously analyze, insert, and tune C/C++ HLS pragmas (pipeline, unroll, array_partition, dataflow) through closed-loop feedback with synthesis tools. Use when: 'optimize this HLS kernel', 'add pragmas to this C++ for FPGA', 'explore the design space for this Vitis HLS project', 'tune my hardware accelerator', 'analyze this HLS synthesis report', 'debug why my HLS design has poor throughput'."
---

# Agentic High-Level Synthesis: From Pragmas to Partners

This skill enables Claude to act as an agentic HLS optimization partner that analyzes C/C++ hardware kernels, systematically inserts and tunes synthesis pragmas, interprets HLS tool reports, and iteratively refines designs through a closed-loop feedback workflow. Based on the six-level autonomy taxonomy from Zhang et al. (2026), this skill operates at L1-L3: explaining synthesis reports with source attribution, suggesting pragma edits, running closed-loop autotuning over defined search spaces, and performing multi-step tool-guided optimization with verification.

## When to Use

- When the user asks to optimize a C/C++ kernel for FPGA synthesis (Vitis HLS, Vivado HLS, Catapult HLS, or similar)
- When the user wants to add or tune HLS pragmas (`#pragma HLS pipeline`, `unroll`, `array_partition`, `dataflow`, `inline`, etc.)
- When the user shares an HLS synthesis report and wants to understand bottlenecks or improve latency/throughput/resource usage
- When the user wants to explore the design space of an HLS kernel — varying pragma configurations to find Pareto-optimal points
- When the user asks to debug a failing or underperforming HLS design (e.g., "why does my loop have II > 1?")
- When the user wants to convert a software-style C++ function into an efficient hardware accelerator description
- When the user asks about HLS portability or wants to restructure code to be more "permutable" across pragma configurations

## Key Technique

**HLS as an agentic abstraction layer.** High-Level Synthesis compiles C/C++ into RTL (Verilog/VHDL) for FPGAs. The key optimization lever is *pragmas* — compiler directives that control parallelism, pipelining, memory layout, and dataflow. The design space is combinatorial: a kernel with 5 loops and 3 arrays can have thousands of valid pragma configurations. Manual exploration is slow; agents excel here because HLS code is *highly permutable* — one pragma configuration can be swapped for another without changing functional semantics.

**Mixed-fidelity feedback loop.** Rather than running full synthesis (which takes minutes to hours) for every candidate, agents should use a three-tier evaluation strategy: (1) *low-fidelity* — static analysis of pragma compatibility and resource estimates, (2) *medium-fidelity* — C-simulation and lightweight scheduling analysis from HLS tool reports, and (3) *high-fidelity* — full synthesis and place-and-route for the most promising candidates. This reduces iteration cost dramatically.

**Golden reference validation.** Every optimization must be verified against a functional golden reference — the original un-optimized or known-correct design. Agents run C-simulation or co-simulation to confirm that pragma insertions preserve correctness before evaluating performance. This prevents the common failure mode where aggressive pragmas produce designs that synthesize but compute incorrect results.

## Step-by-Step Workflow

1. **Parse the HLS kernel.** Read the C/C++ source and identify all synthesizable functions, loops (with trip counts), arrays (with dimensions and access patterns), and existing pragmas. Build a mental model of the compute and memory structure.

2. **Establish the golden reference.** If a testbench exists, run C-simulation (`vitis_hls -f run_csim.tcl` or equivalent) to capture expected outputs. If no testbench exists, create one with representative inputs and record outputs. This is the correctness baseline.

3. **Profile the current design.** Run HLS synthesis on the unoptimized design and extract key metrics from the report: total latency (cycles), initiation interval (II) per loop, resource utilization (LUT, FF, BRAM, DSP), and any warnings about unresolved dependencies or failed scheduling.

4. **Identify bottlenecks.** Map report metrics back to source code locations. Common bottlenecks: loops with II > 1 (caused by loop-carried dependencies or memory port conflicts), large latency from non-pipelined loops, excessive BRAM usage from non-partitioned arrays, and missing dataflow between sequential functions.

5. **Generate pragma candidates.** For each bottleneck, propose targeted pragmas:
   - **II > 1 on a loop:** `#pragma HLS pipeline II=1` (and resolve the dependency causing II inflation)
   - **Sequential loop execution:** `#pragma HLS unroll factor=N` (choose N based on resource budget)
   - **Memory port conflicts:** `#pragma HLS array_partition variable=arr type=cyclic factor=N`
   - **Sequential function calls:** `#pragma HLS dataflow` at the function level
   - **Large functions preventing inlining:** `#pragma HLS inline`

6. **Evaluate candidates at low fidelity.** Before synthesis, check for pragma conflicts: unrolling a loop by N requires N memory ports, so array partition factors must match. Verify that pipeline and dataflow pragmas are not applied to the same scope (they are mutually exclusive within a region). Estimate resource growth from unroll factors.

7. **Synthesize promising configurations.** Run HLS synthesis on the top 2-4 configurations. Compare latency, II, and resource utilization against the baseline. Record results in a structured format for comparison.

8. **Verify correctness.** For any configuration that improves performance, re-run C-simulation against the golden reference. If outputs diverge, discard the configuration and investigate whether the pragma exposed a latent bug or changed semantics.

9. **Iterate or compose.** If targets are not met, combine successful pragmas (e.g., pipeline + array partition), try alternative strategies (e.g., loop tiling instead of full unroll), or restructure the code (e.g., split a function to enable dataflow). Each iteration follows the same synthesize-verify cycle.

10. **Report the Pareto frontier.** Present the user with a table of configurations showing latency, throughput, and resource utilization. Recommend the configuration that best matches their constraints (e.g., "must fit within 80% of available BRAMs" or "minimize latency regardless of area").

## Concrete Examples

**Example 1: Optimizing a matrix multiplication kernel**

User: "Optimize this HLS kernel for Vitis HLS — I need better throughput."

```c
// matmul.cpp
void matmul(int A[64][64], int B[64][64], int C[64][64]) {
    for (int i = 0; i < 64; i++)
        for (int j = 0; j < 64; j++) {
            int sum = 0;
            for (int k = 0; k < 64; k++)
                sum += A[i][k] * B[k][j];
            C[i][j] = sum;
        }
}
```

Approach:
1. Identify three nested loops with no existing pragmas. The innermost `k` loop accumulates into a scalar — good candidate for pipelining.
2. Run baseline synthesis: expect latency ~262144 cycles (64^3), II=1 on inner loop but no parallelism.
3. Apply pragmas incrementally:

```c
void matmul(int A[64][64], int B[64][64], int C[64][64]) {
#pragma HLS array_partition variable=A type=complete dim=2
#pragma HLS array_partition variable=B type=complete dim=1
    for (int i = 0; i < 64; i++)
        for (int j = 0; j < 64; j++) {
#pragma HLS pipeline II=1
            int sum = 0;
            for (int k = 0; k < 64; k++)
                sum += A[i][k] * B[k][j];
            C[i][j] = sum;
        }
}
```

4. The `pipeline` on the `j` loop flattens the inner `k` loop. Array partitioning on `dim=2` of A and `dim=1` of B provides enough ports for parallel reads.
5. Expected result: latency drops to ~4096 cycles (64x64), at the cost of significantly more DSPs and LUTs.

Output:
```
| Config     | Latency (cycles) | II  | BRAM | DSP | LUT   |
|------------|-------------------|-----|------|-----|-------|
| Baseline   | 262,144           | 1   | 6    | 3   | 1,200 |
| Optimized  | 4,160             | 1   | 6    | 64  | 8,400 |
```

**Example 2: Debugging II > 1 in a FIR filter**

User: "My FIR filter has II=3 but I set pipeline II=1. Why?"

```c
void fir(int input, int output[1], int coeffs[16], int shift_reg[16]) {
#pragma HLS pipeline II=1
    shift_reg[0] = input;
    for (int i = 15; i > 0; i--)
        shift_reg[i] = shift_reg[i-1];
    int acc = 0;
    for (int i = 0; i < 16; i++)
        acc += shift_reg[i] * coeffs[i];
    output[0] = acc;
}
```

Approach:
1. Identify the problem: `shift_reg` is accessed in a read-after-write pattern across the shift loop, and it defaults to a dual-port BRAM — only 2 read/write ports per cycle.
2. The shift loop writes `shift_reg[i] = shift_reg[i-1]` which creates a sequential dependency chain, and BRAM port limits cause II inflation.
3. Fix: partition `shift_reg` completely so each element is a register.

```c
#pragma HLS array_partition variable=shift_reg type=complete
```

4. With 16 registers instead of a BRAM, all shifts happen in one cycle. II=1 achieved.

Output: "The II=3 is caused by BRAM port contention on `shift_reg`. The shift loop requires reading and writing multiple elements per cycle, but a BRAM only has 2 ports. Adding `#pragma HLS array_partition variable=shift_reg type=complete` converts it to registers, resolving the conflict. This costs 16 FFs instead of 1 BRAM block."

**Example 3: Design space exploration for a convolution kernel**

User: "Explore the design space for this 2D convolution — I have a budget of 200 DSPs and 100 BRAMs."

Approach:
1. Identify tunable parameters: unroll factor on output-channel loop (1, 2, 4, 8, 16), array partition factor on weight array, pipeline target (inner loop vs. outer loop).
2. Eliminate infeasible points at low fidelity: unroll=16 requires 16x DSPs (likely over budget).
3. Synthesize 4-5 candidates, record results.
4. Present Pareto frontier filtered by the resource budget.

Output:
```
| Unroll | Partition | Latency  | DSP | BRAM | Feasible? |
|--------|-----------|----------|-----|------|-----------|
| 1      | none      | 1,048K   | 5   | 12   | Yes       |
| 2      | cyclic=2  | 524K     | 10  | 16   | Yes       |
| 4      | cyclic=4  | 262K     | 20  | 24   | Yes       |
| 8      | cyclic=8  | 131K     | 40  | 40   | Yes       |
| 16     | cyclic=16 | 65K      | 80  | 72   | Yes       |
Recommended: unroll=8 (best latency within 200 DSP / 100 BRAM budget).
```

## Best Practices

- **Do:** Always establish a golden reference via C-simulation before optimizing. Pragma changes that break correctness are worse than no optimization.
- **Do:** Start with `pipeline` on the innermost loop, then work outward. Pipelining is the highest-impact single pragma in most designs.
- **Do:** Match `array_partition` factors to the parallelism implied by `unroll` or `pipeline`. If you unroll by 4, you need at least 4 memory ports on each accessed array.
- **Do:** Check pragma compatibility before synthesis. `dataflow` and `pipeline` in the same scope conflict. Nested `dataflow` regions are not supported by most tools.
- **Avoid:** Applying `unroll` with no factor (defaults to complete unroll) on large loops — this explodes resource usage and synthesis time.
- **Avoid:** Ignoring the synthesis warnings section. Messages about "unable to schedule" or "dependency prevents II=1" contain the exact source-level cause of performance loss.

## Error Handling

- **Synthesis fails with "cannot schedule operation":** The pragma configuration creates an impossible schedule. Most common cause: too-aggressive pipeline II combined with insufficient array ports. Relax the II target or add array partitioning.
- **C-simulation passes but co-simulation fails:** Interface mismatch. Check that `#pragma HLS interface` directives match the testbench's port protocol (ap_ctrl_hs vs. ap_ctrl_chain, axis vs. m_axi).
- **Resource utilization exceeds 100%:** Over-parallelization. Reduce unroll factors or switch from complete to cyclic/block array partitioning with smaller factors.
- **Synthesis report shows "?" for latency:** The loop bound is not statically determinable. Add `#pragma HLS loop_tripcount min=N max=M` to provide estimates, or restructure code to use compile-time constants.
- **II target not met (achieved II > requested II):** Read the dependency violation details in the schedule viewer. Common causes: loop-carried dependencies on arrays (fix with partitioning or rewriting the accumulation), or external memory access latency.

## Limitations

- This skill applies to C/C++ HLS workflows (Vitis HLS, Vivado HLS, Intel HLS, Catapult HLS). It does not apply to RTL-level (Verilog/VHDL) optimization or purely software performance tuning.
- Pragma optimization cannot fix algorithmic inefficiency. If the algorithm is O(n^3) and the target requires O(n^2), code restructuring — not pragmas — is needed.
- Without access to the actual HLS synthesis tool, Claude can suggest pragmas and predict their effects but cannot guarantee exact latency or resource numbers. Always verify with synthesis.
- Design space exploration at L2-L3 autonomy requires iterative tool invocation. In a single-shot interaction, Claude operates at L1 (copilot) — providing analysis and recommendations rather than closed-loop optimization.
- FPGA-specific constraints (clock frequency, available resources on a specific chip) affect which configurations are feasible. Always specify the target device when asking for optimization.

## Reference

Zhang, N., Kim, S., Srinath, S., & Zhang, Z. (2026). *From Pragmas to Partners: A Symbiotic Evolution of Agentic High-Level Synthesis.* arXiv:2602.01401v3. [https://arxiv.org/abs/2602.01401v3](https://arxiv.org/abs/2602.01401v3)

Key insight: HLS code is *highly permutable* — pragma configurations can be swapped without changing functional semantics — making it an ideal substrate for agentic optimization through mixed-fidelity feedback loops and automated design space exploration.