---
name: "report-nsf-workshop-ai"
description: "Apply AI techniques from the NSF AI-for-EDA workshop to hardware design tasks: RTL code generation from natural language, HLS pragma optimization via GNNs, logic synthesis optimization with RL, LLM-assisted hardware verification, and ML-augmented SAT solving. Trigger phrases: 'generate Verilog from description', 'optimize HLS pragmas', 'verify RTL design', 'AI-assisted EDA', 'logic synthesis optimization', 'hardware design automation'."
---

# AI for Electronic Design Automation (EDA)

This skill enables Claude to apply the AI-for-EDA techniques catalogued in the NSF Workshop report (arXiv:2601.14541v3) to real hardware design tasks. It covers four pillars: (1) generating RTL code from natural language specifications using LLM pipelines with formal verification feedback loops, (2) optimizing HLS designs by predicting performance from program graphs and inserting pragmas, (3) applying RL-based logic synthesis to minimize area/delay, and (4) augmenting hardware verification with LLM-generated assertions and ML-guided SAT solving. The workshop distills findings from leading EDA and ML researchers into concrete methods that bridge the gap between AI capabilities and production chip design workflows.

## When to Use

- When the user asks to generate Verilog or SystemVerilog modules from a natural language specification or high-level description
- When the user wants to optimize HLS C/C++ code with pragmas for FPGA targets (loop unrolling, pipelining, array partitioning)
- When the user needs to write or improve SystemVerilog Assertions (SVA) for design verification
- When the user is working on logic synthesis optimization and wants to explore AI-guided synthesis recipes
- When the user asks to set up ML pipelines for predicting timing, power, or area from RTL or gate-level netlists
- When the user wants to integrate LLM-based code generation into an EDA toolchain with correctness feedback
- When the user is building or evaluating GNN models over circuit graphs (AIGs, MIGs, control-dataflow graphs)
- When the user asks about AI-assisted DRC (design rule checking) code generation or DFM hotspot detection

## Key Technique

The workshop identifies a **neurosymbolic feedback loop** as the unifying pattern across successful AI-for-EDA applications. Rather than using AI models in isolation, the highest-impact methods pair a generative AI component (LLM for code, GNN for prediction, RL agent for optimization) with a symbolic verifier (formal equivalence checker, SAT/SMT solver, simulation testbench, or HLS tool). The symbolic component provides ground-truth feedback that corrects the AI's output, enabling iterative refinement without human intervention. For example, an LLM generates Verilog, a formal checker identifies mismatches against the spec, and the error trace is fed back to the LLM for correction. This pattern applies at every level: RTL generation, pragma tuning, logic synthesis, and post-silicon test.

For **HLS optimization**, the GNN-DSE approach represents C/C++ programs as control-dataflow graphs and trains graph neural networks to predict post-synthesis metrics (latency, resource usage, throughput) in milliseconds rather than the hours required by running the actual HLS tool. This enables rapid design-space exploration: thousands of pragma configurations can be scored by the GNN surrogate and only the top candidates need full HLS evaluation. Extensions like HARP add hierarchical modeling for long-range dependencies, and ProgSG fuses GNN and LLM representations via cross-modality attention for richer program understanding.

For **logic synthesis**, RL agents learn synthesis recipes (sequences of optimization passes like rewrite, refactor, balance, resub) that outperform hand-tuned defaults. DRiLLS uses Advantage Actor-Critic RL to achieve 13% average QoR improvement, while INVICTUS combines RL with search to reduce area-delay product by 30% with 6.3x runtime reduction. The key insight is that synthesis pass ordering is a sequential decision problem with delayed rewards, making it naturally suited to RL formulations over AND-Inverter Graphs (AIGs).

## Step-by-Step Workflow

### A. Natural Language to RTL Generation with Verification Feedback

1. **Parse the specification**: Extract the module interface (inputs, outputs, widths, protocols) from the user's natural language description. Identify functional requirements, timing constraints, and edge cases explicitly stated or implied.

2. **Generate initial RTL**: Produce a Verilog or SystemVerilog module implementing the specification. Follow synthesizable coding conventions: use `always_ff` for sequential logic, `always_comb` for combinational, explicit reset, and no latches unless specified.

3. **Generate a testbench**: Create a SystemVerilog testbench with directed test vectors covering normal operation, boundary conditions, and reset behavior. Include self-checking assertions using `assert` statements.

4. **Run simulation and collect feedback**: If the user has a simulator available (Icarus Verilog, Verilator, etc.), run the testbench. Parse any assertion failures or mismatches into structured error descriptions.

5. **Iterate on failures**: For each failing test vector, identify the root cause in the RTL, fix the logic, and re-run. Limit to 3 refinement iterations before presenting the user with the remaining issues and asking for clarification.

6. **Generate SVA coverage properties**: After functional correctness, produce SystemVerilog Assertions for key protocol invariants (e.g., `assert property (@(posedge clk) req |-> ##[1:3] ack)`) to catch regressions.

### B. HLS Pragma Optimization

1. **Ingest the HLS C/C++ source**: Read the user's kernel code and identify loop nests, array declarations, function boundaries, and data dependencies.

2. **Enumerate the pragma design space**: For each loop, list candidate pragmas: `PIPELINE II=N`, `UNROLL factor=N`, `LOOP_FLATTEN`. For each array, list: `ARRAY_PARTITION type={cyclic,block,complete} factor=N`. Compute the combinatorial space size.

3. **Build a program graph representation**: Construct a control-dataflow graph from the source. Nodes represent operations and control structures; edges represent data dependencies and control flow. Annotate nodes with operation types and bitwidths.

4. **Apply GNN-based surrogate prediction**: If a trained model is available, use it to score pragma configurations. Otherwise, recommend a heuristic search: start with full pipelining of innermost loops, then progressively unroll and partition arrays to resolve resource bottlenecks.

5. **Select top-K configurations**: Rank configurations by predicted latency or throughput. Present the top 3-5 to the user with estimated resource usage trade-offs.

6. **Validate with HLS tool**: Run Vitis HLS, Intel HLS, or Bambu on the selected configurations. Compare actual vs. predicted metrics and flag any configurations that fail timing or exceed resource limits.

### C. RL-Guided Logic Synthesis

1. **Accept the design in AIG or Verilog format**: Parse the input into an And-Inverter Graph using ABC or Yosys (`read_verilog`, `synth -top`, `abc -script`).

2. **Define the optimization objective**: Clarify whether the user is optimizing for area (node count), delay (critical path depth), or area-delay product. Set up the reward function accordingly.

3. **Generate a synthesis recipe sequence**: Apply optimization passes in sequence: `rewrite`, `rewrite -z`, `refactor`, `refactor -z`, `balance`, `resub`, `resub -z`. Use an RL-inspired greedy search: at each step, try all candidate passes and keep the one yielding the best immediate improvement with ties broken by historical performance.

4. **Evaluate after each pass**: Record AIG node count and depth. Stop when no pass yields improvement for 2 consecutive rounds or a maximum of 20 passes is reached.

5. **Map to technology library**: Run `abc -c "map"` or equivalent technology mapping to produce gate-level metrics. Report final area, delay, and power estimates.

6. **Compare against baseline**: Run the default `resyn2` script and report relative improvement in area, delay, and area-delay product.

## Concrete Examples

**Example 1: Generate a UART Transmitter from Natural Language**

User: "Write a Verilog module for a UART transmitter. 8 data bits, 1 stop bit, no parity. Baud rate configurable via a parameter. Include a busy signal."

Approach:
1. Define module interface: `uart_tx #(parameter CLKS_PER_BIT = 868)` with inputs `clk`, `rst_n`, `tx_start`, `tx_data[7:0]` and outputs `tx_out`, `tx_busy`.
2. Implement as a state machine: IDLE -> START_BIT -> DATA_BITS -> STOP_BIT.
3. Use a clock counter to hold each bit for `CLKS_PER_BIT` cycles.
4. Generate testbench transmitting bytes 0x00, 0x55, 0xAA, 0xFF and checking `tx_out` timing.

Output:
```verilog
module uart_tx #(
  parameter CLKS_PER_BIT = 868  // 100MHz / 115200 baud
)(
  input  logic       clk,
  input  logic       rst_n,
  input  logic       tx_start,
  input  logic [7:0] tx_data,
  output logic       tx_out,
  output logic       tx_busy
);

  typedef enum logic [1:0] {
    IDLE      = 2'b00,
    START_BIT = 2'b01,
    DATA_BITS = 2'b10,
    STOP_BIT  = 2'b11
  } state_t;

  state_t state;
  logic [$clog2(CLKS_PER_BIT)-1:0] clk_cnt;
  logic [2:0] bit_idx;
  logic [7:0] tx_shift;

  assign tx_busy = (state != IDLE);

  always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      state   <= IDLE;
      tx_out  <= 1'b1;
      clk_cnt <= '0;
      bit_idx <= '0;
    end else begin
      case (state)
        IDLE: begin
          tx_out <= 1'b1;
          if (tx_start) begin
            tx_shift <= tx_data;
            state    <= START_BIT;
            clk_cnt  <= '0;
          end
        end
        START_BIT: begin
          tx_out <= 1'b0;
          if (clk_cnt == CLKS_PER_BIT - 1) begin
            clk_cnt <= '0;
            bit_idx <= '0;
            state   <= DATA_BITS;
          end else clk_cnt <= clk_cnt + 1;
        end
        DATA_BITS: begin
          tx_out <= tx_shift[bit_idx];
          if (clk_cnt == CLKS_PER_BIT - 1) begin
            clk_cnt <= '0;
            if (bit_idx == 7) state <= STOP_BIT;
            else bit_idx <= bit_idx + 1;
          end else clk_cnt <= clk_cnt + 1;
        end
        STOP_BIT: begin
          tx_out <= 1'b1;
          if (clk_cnt == CLKS_PER_BIT - 1) begin
            state   <= IDLE;
            clk_cnt <= '0;
          end else clk_cnt <= clk_cnt + 1;
        end
      endcase
    end
  end
endmodule
```

**Example 2: Optimize HLS Pragmas for a Matrix Multiply Kernel**

User: "I have this HLS C++ matrix multiply targeting a Xilinx FPGA. Help me choose pragmas to minimize latency."

```cpp
void matmul(int A[32][32], int B[32][32], int C[32][32]) {
  for (int i = 0; i < 32; i++)
    for (int j = 0; j < 32; j++) {
      int sum = 0;
      for (int k = 0; k < 32; k++)
        sum += A[i][k] * B[k][j];
      C[i][j] = sum;
    }
}
```

Approach:
1. Identify the innermost loop (`k`) as the pipeline target.
2. Partition arrays to resolve memory port bottlenecks: `A` needs row access (partition dim=2), `B` needs column access (partition dim=1).
3. Present three configurations ranked by latency vs. resource trade-off.

Output:
```cpp
void matmul(int A[32][32], int B[32][32], int C[32][32]) {
  // Config 1: Pipeline k-loop, partition both arrays completely on accessed dim
  // Estimated: ~1024 cycles, high BRAM usage
  #pragma HLS ARRAY_PARTITION variable=A complete dim=2
  #pragma HLS ARRAY_PARTITION variable=B complete dim=1

  for (int i = 0; i < 32; i++)
    for (int j = 0; j < 32; j++) {
      #pragma HLS PIPELINE II=1
      int sum = 0;
      for (int k = 0; k < 32; k++)
        #pragma HLS UNROLL
        sum += A[i][k] * B[k][j];
      C[i][j] = sum;
    }
}
```

Trade-off summary:
| Config | Pipeline | Array Partition | Est. Latency | DSP Usage | BRAM |
|--------|----------|-----------------|-------------|-----------|------|
| 1      | j-loop, II=1 | A dim=2 complete, B dim=1 complete | ~1,024 cycles | 32 DSPs | High |
| 2      | k-loop, II=1 | A dim=2 factor=8, B dim=1 factor=8 | ~4,096 cycles | 8 DSPs | Medium |
| 3      | k-loop, II=1 | No partition | ~32,768 cycles | 1 DSP | Low |

**Example 3: Generate SVA Properties for an AXI-Lite Interface**

User: "Write SystemVerilog Assertions for my AXI-Lite slave to catch protocol violations."

Approach:
1. Identify key AXI-Lite protocol rules from the AMBA spec.
2. Generate assertions for handshake stability, response validity, and no simultaneous read/write constraints.

Output:
```systemverilog
// AWVALID must remain asserted until AWREADY handshake
aw_stable: assert property (@(posedge ACLK) disable iff (!ARESETn)
  AWVALID && !AWREADY |=> AWVALID
) else $error("AWVALID deasserted before AWREADY");

// WVALID must remain asserted until WREADY handshake
w_stable: assert property (@(posedge ACLK) disable iff (!ARESETn)
  WVALID && !WREADY |=> WVALID
) else $error("WVALID deasserted before WREADY");

// Write response must be OKAY (2'b00) or SLVERR (2'b10)
bresp_valid: assert property (@(posedge ACLK) disable iff (!ARESETn)
  BVALID |-> (BRESP == 2'b00 || BRESP == 2'b10)
) else $error("Invalid BRESP value");

// ARVALID stability
ar_stable: assert property (@(posedge ACLK) disable iff (!ARESETn)
  ARVALID && !ARREADY |=> ARVALID
) else $error("ARVALID deasserted before ARREADY");

// RVALID stability
r_stable: assert property (@(posedge ACLK) disable iff (!ARESETn)
  RVALID && !RREADY |=> RVALID
) else $error("RVALID deasserted before RREADY");

// No response without prior request
no_orphan_bresp: assert property (@(posedge ACLK) disable iff (!ARESETn)
  BVALID |-> $past(AWVALID && AWREADY)
    or $past(AWVALID && AWREADY, 2)
    or $past(AWVALID && AWREADY, 3)
) else $warning("BRESP without recent AW handshake");
```

## Best Practices

- **Do:** Always pair LLM-generated RTL with a testbench or formal properties. The neurosymbolic feedback loop is the core insight of the workshop -- generation without verification is unreliable.
- **Do:** Use synthesizable SystemVerilog constructs (`always_ff`, `always_comb`, `logic`) rather than behavioral Verilog (`always @*`, `reg`, `wire` mixing) when generating RTL.
- **Do:** When optimizing HLS pragmas, start with the innermost loop and work outward. Pipeline the innermost loop first, then address memory bottlenecks with array partitioning.
- **Do:** Present multiple pragma configurations with resource/performance trade-offs rather than a single "optimal" solution -- the right choice depends on the user's FPGA resource budget.
- **Avoid:** Generating RTL with inferred latches (incomplete case statements, missing else branches in combinational blocks). Always include default cases.
- **Avoid:** Over-unrolling in HLS -- complete unrolling of large loops exhausts DSP and LUT resources. Use `factor=N` for partial unrolling when the loop bound exceeds 16.
- **Avoid:** Applying logic synthesis RL recipes trained on small benchmarks (e.g., ISCAS-85) directly to large industrial designs without re-evaluation. Domain shift is a known failure mode.

## Error Handling

- **Simulation fails to compile**: Check for missing module declarations, undeclared signals, or width mismatches. Icarus Verilog requires explicit `timescale` directives; Verilator requires `--sv` flag for SystemVerilog.
- **HLS tool rejects pragmas**: The most common issue is partitioning arrays that are function arguments (tool-specific restrictions). Move arrays to local scope or use `#pragma HLS INTERFACE` directives first.
- **Synthesis recipe produces worse results**: RL-guided synthesis can get stuck in local optima. Reset to the best-known AIG and try a different pass ordering. If the design has >100K nodes, restrict to `resyn2; resyn2` as a baseline before exploring.
- **SVA assertions fire spuriously after reset**: Ensure all assertions use `disable iff (!reset_n)` to mask the reset period. Also check that the reset polarity matches the design.
- **GNN surrogate predictions are inaccurate**: Surrogate models trained on one HLS toolchain (e.g., Vivado 2020.2) may not transfer to another version. Always validate the top-K predicted configurations with the actual tool.

## Limitations

- LLM-generated RTL is most reliable for well-known module types (UART, SPI, FIFO, simple AXI peripherals). Novel or highly custom architectures require significantly more human review and iterative refinement.
- HLS pragma optimization via surrogate models requires a pre-trained GNN model specific to the target HLS tool and FPGA family. Without one, the skill falls back to heuristic-guided search, which is less effective.
- RL-based logic synthesis has demonstrated gains on academic benchmarks but real-world industrial netlists with millions of gates remain challenging due to training time and generalization gaps.
- The workshop identifies data scarcity as a fundamental bottleneck: unlike software, hardware design datasets are proprietary and small. Open benchmarks (HLSyn, OpenLS-DGF, MG-Verilog) help but do not cover all design domains.
- AI-generated designs carry security risks: adversarial inputs or training data poisoning could introduce hardware trojans. All AI-generated RTL should undergo independent formal verification before tapeout.

## Reference

**Paper:** "Report for NSF Workshop on AI for Electronic Design Automation" (arXiv:2601.14541v3, IEEE Circuits and Systems Magazine 2026). Key sections: Section 2 for GNN-DSE and HLS pragma methods, Section 3 for NL2RTL generation pipelines and logic synthesis RL, Section 4 for neurosymbolic verification approaches and ML-augmented SAT solving. The workshop website with additional materials: https://ai4eda-workshop.github.io/