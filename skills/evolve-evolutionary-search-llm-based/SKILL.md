---
name: "evolve-evolutionary-search-llm-based"
description: "Evolutionary search framework for LLM-driven Verilog/RTL generation and PPA optimization. Uses MCTS for functional correctness and Idea-Guided Refinement for optimization, with structured testbench generation for rapid feedback. Triggers: 'generate Verilog module', 'optimize RTL design', 'evolutionary search for hardware', 'MCTS Verilog generation', 'reduce PPA for circuit', 'fix Verilog functional correctness'."
---

# EvolVE: Evolutionary Search for LLM-based Verilog Generation and Optimization

This skill enables Claude to apply the EvolVE framework — a dual-strategy evolutionary search system — to generate functionally correct Verilog/RTL code and optimize it for Power, Performance, and Area (PPA). Instead of single-shot code generation, Claude iteratively refines hardware designs using Monte Carlo Tree Search (MCTS) when chasing functional correctness, and Idea-Guided Refinement (IGR) when optimizing an already-correct design. A Structured Testbench Generation (STG) phase provides continuous scoring feedback to accelerate convergence.

## When to Use

- When the user asks Claude to generate a Verilog or SystemVerilog module from a natural-language specification
- When the user has a functionally incorrect RTL design and needs iterative debugging with structured feedback
- When the user wants to optimize an existing Verilog design for area, latency, or power (PPA reduction)
- When the user needs automated testbench generation for a Verilog module's port interface
- When the user is working on contest-style IC design problems (e.g., systolic arrays, convolution accelerators, Huffman encoders)
- When the user wants to explore multiple architectural variants of a hardware design systematically rather than committing to a single approach

## Key Technique

**Dual-strategy evolutionary search.** EvolVE recognizes that functional correctness and PPA optimization are fundamentally different objectives requiring different search strategies. For correctness, MCTS explores a tree of code variants where each node `N = (V, S, F)` holds Verilog code `V`, a continuous correctness score `S` (fraction of testbench vectors passed), and diagnostic feedback `F`. The UCT formula balances exploitation of high-scoring nodes against exploration of under-visited branches, with expansion constant `c = 1.4` and 3 child nodes per expansion. For optimization, Idea-Guided Refinement (IGR) first generates `k` high-level architectural ideas (e.g., "convert output-stationary to weight-output-stationary dataflow", "share multipliers across pipeline stages"), then each idea spawns a chain of `m` differential refinement steps. The best node across all `k x m` candidates wins.

**Structured Testbench Generation (STG)** eliminates the bottleneck of hand-written testbenches. STG parses the DUT port interface with regex to classify signals into Clock/Reset, Control, and Datapath groups. Control signals with width `w <= 8` get exhaustive `2^w` state coverage; wider signals use constrained-random sampling. Datapath inputs are seeded with corner cases (zero, max, alternating bits). The output is a continuous score `P_stg in [0,1]` — the pass rate across all test vectors — which provides much richer gradient signal than binary pass/fail. This scoring function drives both MCTS backpropagation and IGR candidate ranking.

**Scoring separates the two phases.** For generation (correctness): `S_gen = p_i / |T|` where `p_i` is passed test count. For optimization: `S_opt = -A * L / eta` (negative area-latency product, normalized) — but only if all tests pass; otherwise a heavy penalty `C_penalty = -10^5` forces the search back toward correctness. This constraint ensures optimization never sacrifices functionality.

## Step-by-Step Workflow

1. **Parse the hardware specification.** Extract the module name, port interface (inputs, outputs, widths), behavioral description, and any constraints (target clock period, area budget, latency requirement). If the spec is ambiguous, ask the user to clarify before proceeding.

2. **Generate Structured Testbench (STG).** Classify each port signal as Clock/Reset, Control, or Datapath using its name and width. For control signals with width <= 8 bits, enumerate all `2^w` input combinations. For wider datapath signals, generate corner-case vectors (all zeros, all ones, alternating `0xAA`/`0x55`, max unsigned value) plus 20-50 constrained-random vectors. Write a self-checking testbench that computes a continuous pass rate.

3. **Produce an initial Verilog implementation.** Generate a straightforward, correct-first design from the spec. Prioritize clarity and correctness over optimization. Use synchronous design patterns with explicit reset logic.

4. **Evaluate via simulation.** Run the design against the STG testbench using `iverilog` and `vvp`. Compute the continuous correctness score (pass rate). Capture any compiler errors or simulation mismatches as diagnostic feedback `F`.

5. **If correctness < 100%, apply MCTS-style search.** Treat the current code as a tree node. For each iteration: (a) select the most promising node via UCT balancing, (b) generate a child variant by applying targeted fixes guided by the diagnostic feedback, (c) simulate the child, (d) backpropagate the score to all ancestors. Repeat for up to N iterations (start with 10-20; the paper shows monotonic improvement up to 300 nodes). Use differential edits — fix specific bugs rather than regenerating the entire file.

6. **Once functionally correct, decide if optimization is needed.** If the user requested PPA optimization or the design has obvious inefficiencies (e.g., combinational multipliers that could be pipelined, redundant registers), proceed to IGR. Otherwise, deliver the correct design.

7. **Apply Idea-Guided Refinement (IGR) for optimization.** Generate 3-5 high-level optimization ideas appropriate to the design type. Examples: "pipeline the datapath to reduce critical path", "share ALU resources across time-multiplexed operations", "convert to systolic array dataflow", "use Gray coding to reduce switching activity". For each idea, produce an implementation and then refine it through 3-4 differential edit steps.

8. **Score optimization candidates.** For each variant that passes all testbench vectors, compute the PPA proxy: `area * latency`. If synthesis tools are available (`yosys`), run `synth -top <module>; stat` to get cell count and estimated area. Rank candidates by the optimization score.

9. **Select and deliver the best design.** Present the winning variant to the user with a summary of: (a) correctness status, (b) architectural approach chosen, (c) PPA metrics if available, (d) trade-offs compared to alternatives explored.

10. **Iterate if the user requests further refinement.** The IGR phase can be re-entered with new ideas informed by synthesis feedback. Each round should target a specific metric (area, timing, power) rather than all at once.

## Concrete Examples

**Example 1: Generating a FIFO buffer from spec**

User: "Write a synchronous FIFO with 8-entry depth, 32-bit data width, full/empty flags, and parameterizable depth."

Approach:
1. Parse spec: module `sync_fifo`, params `DEPTH=8, WIDTH=32`, ports `clk, rst_n, wr_en, rd_en, din[31:0], dout[31:0], full, empty`.
2. Generate STG testbench: enumerate control combos for `{wr_en, rd_en}` (4 states), seed `din` with `0x00000000`, `0xFFFFFFFF`, `0xAAAAAAAA`, `0x55555555`, plus random vectors. Test sequences: fill to full, read to empty, simultaneous read/write, overflow/underflow edge cases.
3. Generate initial Verilog using pointer-based FIFO (wr_ptr, rd_ptr, count register).
4. Simulate — suppose score is 0.85 (empty flag logic is wrong on simultaneous read/write when count=1).
5. MCTS iteration: feedback says `empty` asserted one cycle early during simultaneous R/W. Generate child node fixing the count update priority. Re-simulate — score 1.0.
6. Deliver correct FIFO.

Output:
```verilog
module sync_fifo #(
  parameter DEPTH = 8,
  parameter WIDTH = 32
)(
  input  wire             clk,
  input  wire             rst_n,
  input  wire             wr_en,
  input  wire             rd_en,
  input  wire [WIDTH-1:0] din,
  output reg  [WIDTH-1:0] dout,
  output wire             full,
  output wire             empty
);
  localparam PTR_W = $clog2(DEPTH);
  reg [WIDTH-1:0] mem [0:DEPTH-1];
  reg [PTR_W:0]   wr_ptr, rd_ptr;

  wire [PTR_W:0] count = wr_ptr - rd_ptr;
  assign full  = (count == DEPTH);
  assign empty = (count == 0);

  always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
      wr_ptr <= 0;
      rd_ptr <= 0;
    end else begin
      if (wr_en && !full) begin
        mem[wr_ptr[PTR_W-1:0]] <= din;
        wr_ptr <= wr_ptr + 1;
      end
      if (rd_en && !empty) begin
        dout <= mem[rd_ptr[PTR_W-1:0]];
        rd_ptr <= rd_ptr + 1;
      end
    end
  end
endmodule
```

**Example 2: Optimizing a combinational multiplier for area**

User: "I have a working 16x16 unsigned multiplier but it uses too much area. Optimize it."

Approach:
1. Read existing code — confirm it is a single-cycle `assign product = a * b;` style.
2. Generate STG: corner cases `{0x0000, 0xFFFF, 0x0001, 0x8000, 0x5555, 0xAAAA}` cross-product plus random pairs. Verify existing design passes 100%.
3. Enter IGR phase. Generate optimization ideas:
   - Idea A: "Sequential shift-and-add multiplier over 16 cycles (minimal area, higher latency)"
   - Idea B: "Booth-encoded radix-4 multiplier (halves partial products)"
   - Idea C: "Pipelined 4-stage array multiplier (balanced area/throughput)"
4. Implement each idea with 3-4 refinement steps. Verify each passes all testbench vectors.
5. If `yosys` available, synthesize each: `read_verilog mult.v; synth; stat`.
6. Present results:

```
| Variant               | Cells | Latency (cycles) | Area*Latency |
|-----------------------|-------|-------------------|--------------|
| Original (combinational) | 1847  | 1                 | 1847         |
| Shift-and-add (16 cyc)  | 312   | 16                | 4992         |
| Booth radix-4            | 923   | 1                 | 923          |
| Pipelined array (4 stg)  | 1105  | 4 (1 throughput)  | 1105*        |
```
Recommendation: Booth radix-4 cuts area by 50% with no latency penalty.

**Example 3: Debugging a failing FSM**

User: "My UART transmitter FSM fails simulation — the stop bit is never sent."

Approach:
1. Read the user's Verilog. Identify states: IDLE, START, DATA, STOP.
2. Generate STG targeting the FSM control signals: `tx_start`, `data_in[7:0]`. Enumerate all FSM transitions with data corner cases.
3. Simulate — score 0.75. Diagnostic: `tx_out` stays low during STOP state; bit counter never reaches 8 because reset condition uses `<` instead of `<=`.
4. MCTS node 1: fix `bit_cnt < 8` to `bit_cnt <= 7` (equivalent but clarifies intent). Re-simulate — score 0.75 (same issue, wrong fix).
5. MCTS node 2: trace deeper — the counter increments in DATA state but transition to STOP checks `bit_cnt == 8` while counter only reaches 7. Fix transition condition to `bit_cnt == 7`. Re-simulate — score 1.0.
6. Deliver fix with explanation of the off-by-one error.

## Best Practices

- **Do:** Generate the testbench BEFORE writing the design. STG provides the scoring function that drives the entire evolutionary loop. Without it, you are flying blind.
- **Do:** Use differential edits during refinement — modify only the broken or suboptimal section rather than regenerating the full module. This preserves working code and reduces the chance of introducing new bugs (the paper shows >60% token reduction).
- **Do:** Separate the correctness and optimization phases cleanly. Never optimize a design that does not pass 100% of test vectors. The penalty function (`-10^5`) exists for a reason.
- **Do:** Generate diverse optimization ideas in IGR. Architectural diversity (e.g., dataflow changes, resource sharing, pipelining) discovers better optima than incremental tweaks to the same structure.
- **Avoid:** Regenerating entire files on each iteration. This is rejection sampling, not evolutionary search, and converges far slower.
- **Avoid:** Using binary pass/fail as your only feedback signal. The continuous pass-rate score from STG is critical for guiding MCTS toward partially-correct nodes that are close to a full solution.
- **Avoid:** Optimizing without synthesis feedback. Area/timing estimates from `yosys` (even rough ones) are far more reliable than guessing from code structure alone.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| `iverilog` compilation error | Syntax error in generated Verilog | Feed the exact error message as diagnostic feedback `F` into the next MCTS expansion. Common fixes: missing `end`, wrong port widths, undeclared wires. |
| Testbench hangs (no `$finish`) | Design deadlocks or testbench timeout missing | Add `initial begin #(MAX_TIME) $finish; end` to STG testbench. If design deadlocks, check FSM for unreachable states. |
| All MCTS children score 0 | Spec misunderstood or fundamentally wrong approach | Backtrack to root. Re-read the specification. Generate a completely different architectural approach rather than refining the broken one. |
| Optimization variant breaks correctness | Aggressive transformation introduced bug | Score falls to `C_penalty`. Discard this branch. The IGR chain should continue from the last correct variant, not the broken one. |
| `yosys` synthesis fails | Unsupported constructs (e.g., `real`, system tasks) | Remove simulation-only constructs from the synthesizable module. Keep testbench and DUT cleanly separated. |
| Scores plateau across iterations | Search exhausted local optima | Inject new ideas into IGR (try a fundamentally different architecture). For MCTS, increase exploration constant `c` to encourage visiting unexplored branches. |

## Limitations

- **No replacement for formal verification.** STG provides simulation-based testing with good coverage, but cannot prove correctness for all input combinations on wide datapaths. For safety-critical designs, supplement with formal tools (e.g., SymbiYosys).
- **Synthesis accuracy depends on tools available.** Without `yosys` or a commercial synthesis flow, PPA estimates are heuristic at best. Cell counts from `yosys` do not map directly to silicon area without a technology library.
- **LLM Verilog knowledge has gaps.** Claude's training data contains far less Verilog than software languages. Complex protocols (AXI, PCIe) and advanced constructs (generate blocks, PLI) may require more iterations or human guidance.
- **Scalability ceiling.** The approach works well for module-level design (hundreds to low thousands of lines). Full SoC integration, multi-clock-domain designs, and analog/mixed-signal are outside scope.
- **IGR idea quality depends on problem familiarity.** For well-known structures (FIFOs, multipliers, FSMs), the idea pool is rich. For novel or highly specialized designs, the generated ideas may be less creative than an expert hardware engineer's.

## Reference

**Paper:** [EvolVE: Evolutionary Search for LLM-based Verilog Generation and Optimization](https://arxiv.org/abs/2601.18067) — Hsin et al., 2026. Look for Algorithm 1 (unified MCTS/IGR framework), the STG signal classification scheme (Section III-C), and the IC-RTL benchmark results (Table 5) showing up to 66% PPA reduction over human contest submissions.

**Benchmark code:** [github.com/weiber2002/ICRTL](https://github.com/weiber2002/ICRTL) — Six industry-scale RTL problems with testbenches, reference solutions, and open-source simulation scripts (`iverilog` + `yosys`).