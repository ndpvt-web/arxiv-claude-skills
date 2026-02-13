---
name: "chipbench-next-step-benchmark-evaluating"
description: "Evaluate and improve LLM-generated hardware designs using ChipBench methodology: structured Verilog generation with hierarchical decomposition, systematic RTL debugging across four bug categories, and cross-language reference model generation. Use when: 'generate Verilog for this module', 'debug this RTL code', 'create a Python reference model for this hardware design', 'verify my Verilog against a reference implementation', 'benchmark my chip design workflow', 'translate this hardware spec to SystemC'."
---

# ChipBench: Rigorous Evaluation and Improvement of LLM-Aided Chip Design

This skill enables Claude to apply the ChipBench evaluation methodology to real hardware design tasks. ChipBench (arXiv:2601.21448v2) exposes critical gaps in LLM chip design capabilities by evaluating across three axes: Verilog generation (with hierarchical complexity tiers), systematic RTL debugging (across four bug categories), and heterogeneous reference model generation (Python, SystemC, CXXRTL). The key insight is that structuring hardware design tasks by complexity tier and applying cross-language verification catches errors that single-language testing misses entirely.

## When to Use

- When the user asks to generate Verilog or SystemVerilog for a hardware module from a natural-language specification
- When the user provides buggy RTL code and asks for debugging help (timing, assignment, arithmetic, or state machine bugs)
- When the user needs a Python, SystemC, or CXXRTL reference model for an existing hardware design
- When the user wants to verify hardware code correctness across multiple implementation languages
- When the user is building testbenches or verification infrastructure for digital designs
- When the user asks to evaluate or benchmark LLM-generated hardware code quality
- When the user needs to decompose a complex hierarchical hardware design into manageable sub-modules

## Key Technique

**Tiered Complexity Decomposition.** ChipBench categorizes hardware design tasks into three difficulty tiers that mirror real industrial workflows: (1) self-contained modules — single-module designs with no sub-module dependencies, averaging ~48 lines; (2) hierarchical modules — designs that instantiate sub-modules, where the LLM receives sub-module implementations and must generate only the top-level integration; (3) CPU/IP-level components — real modules from open-source CPU projects (ALUs, branch predictors, register files) where even state-of-the-art models achieve under 22% accuracy. This tiering is critical because LLMs that score 95%+ on flat benchmarks collapse when faced with hierarchical structure. When generating Verilog, always identify which tier a design falls into and adjust the approach accordingly.

**Systematic Bug Taxonomy for Debugging.** Rather than treating debugging as a monolithic task, ChipBench classifies RTL bugs into four actionable categories: arithmetic bugs (operator misuse, e.g., `*` instead of `+`), assignment bugs (incorrect constant values to registers/wires), timing bugs (wrong clock cycle assignments, blocking vs. non-blocking confusion), and state machine bugs (flawed FSM transition logic). This taxonomy drives targeted debugging strategies — timing bugs require clock-cycle-aware analysis while arithmetic bugs need expression-level scrutiny.

**Heterogeneous Cross-Language Verification.** The Heterogeneous Test Engine (HTE) validates hardware designs by generating reference models in multiple languages, compiling golden Verilog to C++ via Verilator, and running 1,000+ random stimulus iterations comparing outputs. The critical detail: reset is toggled once and outputs are compared only during normal operation phase, matching real-world verification practice. This catches bugs that same-language testing misses — a Python model and a SystemC model failing differently on the same stimulus reveals specification ambiguity.

## Step-by-Step Workflow

### For Verilog Generation

1. **Classify the design tier.** Determine if the target module is self-contained (no sub-module dependencies), hierarchical (instantiates other modules), or CPU/IP-level. This dictates prompt structure and expected difficulty.

2. **Extract the I/O interface.** Parse the specification to identify all input/output ports, their widths, clock/reset signals, and any parameterizable values. Write the module declaration skeleton first.

3. **For hierarchical designs, enumerate sub-modules.** List every sub-module the top-level design must instantiate. Provide or request their interface definitions. Generate the top-level wiring and instantiation before filling in glue logic.

4. **Generate the RTL body using specification-to-RTL mapping.** Translate each behavioral requirement into combinational or sequential logic. For FSMs, explicitly enumerate states and draw the transition table before writing `case` blocks.

5. **Apply syntax validation.** Check the generated Verilog compiles with iverilog or a similar tool. Fix syntax errors before functional verification.

6. **Run functional verification against testbenches.** Execute the design against both corner-case tests and random stimulus (1,000+ vectors). Compare outputs against golden reference or specification.

### For RTL Debugging

7. **Categorize the bug type.** Analyze the failing behavior to classify it: arithmetic (wrong computation results), assignment (incorrect constant/initial values), timing (output appears on wrong cycle, blocking/non-blocking mismatch), or state machine (stuck states, wrong transitions).

8. **Apply category-specific debugging.** For timing bugs: trace clock edges and check blocking (`=`) vs. non-blocking (`<=`) usage. For arithmetic: isolate each operator and verify against the spec. For assignment: check all constant values and reset conditions. For FSMs: enumerate all state transitions and compare against the specification.

9. **Verify the fix against the original testbench** and confirm the corrected module produces expected outputs across all stimulus patterns.

### For Reference Model Generation

10. **Select the target language based on design complexity.** Use Python for simple combinational/sequential modules (fastest iteration). Use SystemC or CXXRTL for complex designs requiring cycle-accurate simulation or when cross-language verification is needed.

11. **Implement the reference model preserving cycle-level behavior.** Match the hardware's clock-by-clock semantics — don't abstract away timing. For Python models, use explicit `step()` functions that advance one clock cycle. Reset the model once, then compare only during normal operation.

12. **Cross-verify against the Verilog implementation.** Run identical stimulus through both the reference model and the RTL, comparing outputs at every cycle boundary. Mismatches indicate bugs in either the reference model or the RTL.

## Concrete Examples

**Example 1: Self-Contained Verilog Module Generation**

User: "Generate a Verilog module for a 4-bit priority encoder with valid output."

Approach:
1. Classify as Tier 1 (self-contained, no sub-modules).
2. Define interface: `input [3:0] in`, `output reg [1:0] pos`, `output reg valid`.
3. Generate combinational logic using `casez` or priority `if-else` chain.
4. Validate edge cases: all-zero input (valid=0), multiple bits set (highest wins).

Output:
```systemverilog
module priority_encoder (
    input  [3:0] in,
    output reg [1:0] pos,
    output reg       valid
);
    always @(*) begin
        valid = 1'b1;
        casez (in)
            4'b1???: pos = 2'd3;
            4'b01??: pos = 2'd2;
            4'b001?: pos = 2'd1;
            4'b0001: pos = 2'd0;
            default: begin
                pos   = 2'd0;
                valid = 1'b0;
            end
        endcase
    end
endmodule
```

**Example 2: Debugging an FSM Timing Bug**

User: "My UART transmitter sends data one cycle too early. Here's the RTL..."

Approach:
1. Categorize as **timing bug** — output appears on wrong clock cycle.
2. Trace the FSM: check when `tx_data` is loaded vs. when `tx_start` is asserted.
3. Look for blocking (`=`) vs. non-blocking (`<=`) misuse in sequential always blocks.
4. Check if the baud rate counter initializes correctly or is off-by-one.

Diagnosis:
```
// BUG: Using blocking assignment in sequential block
always @(posedge clk) begin
    tx_reg = shift_reg[0];  // <-- Should be <=
    shift_reg = shift_reg >> 1;  // <-- Should be <=
end

// FIX: Non-blocking assignments for sequential logic
always @(posedge clk) begin
    tx_reg <= shift_reg[0];
    shift_reg <= shift_reg >> 1;
end
```
The blocking assignment causes `tx_reg` to update immediately within the same simulation delta, making downstream logic see the new value one cycle too early.

**Example 3: Python Reference Model for a FIFO**

User: "Create a Python reference model for my synchronous FIFO so I can cross-verify."

Approach:
1. Select Python (moderate complexity, cycle-accurate verification needed).
2. Model the FIFO with explicit clock-step semantics, matching RTL behavior.
3. Implement reset, write, read, full/empty flags with cycle-level accuracy.

Output:
```python
class SyncFifoModel:
    def __init__(self, depth=8, width=8):
        self.depth = depth
        self.mem = [0] * depth
        self.wr_ptr = 0
        self.rd_ptr = 0
        self.count = 0

    def reset(self):
        self.wr_ptr = 0
        self.rd_ptr = 0
        self.count = 0

    def step(self, wr_en, rd_en, wr_data):
        """Advance one clock cycle. Returns (rd_data, full, empty)."""
        rd_data = self.mem[self.rd_ptr] if (rd_en and self.count > 0) else 0
        # Update state (matches non-blocking RTL semantics)
        next_count = self.count
        if wr_en and self.count < self.depth:
            self.mem[self.wr_ptr] = wr_data
            self.wr_ptr = (self.wr_ptr + 1) % self.depth
            next_count += 1
        if rd_en and self.count > 0:
            self.rd_ptr = (self.rd_ptr + 1) % self.depth
            next_count -= 1
        self.count = next_count
        return rd_data, (self.count == self.depth), (self.count == 0)
```
Run 1,000+ random stimulus vectors through both this model and the Verilog FIFO, comparing `rd_data`, `full`, and `empty` at each cycle. Toggle reset once at the start, then compare only during normal operation.

**Example 4: Hierarchical Module — ALU Top-Level Integration**

User: "I have adder, multiplier, and shifter sub-modules. Generate the top-level ALU that selects between them."

Approach:
1. Classify as Tier 2 (hierarchical — instantiates sub-modules).
2. Collect sub-module interfaces (ports, widths).
3. Generate top-level with instantiation and multiplexing logic only.

Output:
```systemverilog
module alu_top (
    input         clk,
    input  [31:0] a, b,
    input  [2:0]  op,
    output reg [31:0] result
);
    wire [31:0] add_out, mul_out, shift_out;

    adder    u_add (.a(a), .b(b), .sum(add_out));
    multiplier u_mul (.a(a), .b(b), .product(mul_out));
    shifter  u_shift (.a(a), .amt(b[4:0]), .out(shift_out));

    always @(*) begin
        case (op)
            3'b000: result = add_out;
            3'b001: result = mul_out;
            3'b010: result = shift_out;
            default: result = 32'b0;
        endcase
    end
endmodule
```
The key is generating only the glue logic and instantiation — the sub-modules are provided, not generated.

## Best Practices

- **Do:** Classify every hardware task by complexity tier before starting. Self-contained modules can be generated directly; hierarchical designs require sub-module interfaces first; CPU/IP designs need careful decomposition.
- **Do:** Use non-blocking assignments (`<=`) in sequential `always @(posedge clk)` blocks and blocking assignments (`=`) in combinational `always @(*)` blocks. Mixing these is the #1 source of timing bugs.
- **Do:** Generate testbenches with both targeted corner cases and random stimulus (1,000+ vectors). Corner cases alone miss interaction bugs; random-only misses boundary conditions.
- **Do:** When creating reference models, use explicit `step()` or `eval()` functions that advance exactly one clock cycle. Never abstract away cycle boundaries.
- **Avoid:** Generating entire hierarchical designs monolithically. Break them into sub-modules and integrate at the top level, providing sub-module code as context.
- **Avoid:** Relying on single-language verification. Cross-language comparison (e.g., Python model vs. Verilog via Verilator) catches specification ambiguity that same-language tests miss.
- **Avoid:** Debugging RTL without first categorizing the bug type. Blind inspection wastes effort — classify as arithmetic, assignment, timing, or state machine first, then apply targeted analysis.

## Error Handling

- **Syntax failures in generated Verilog:** Run iverilog or Verilator compilation first. Parse error messages to identify the failing line and fix before attempting functional verification. Common causes: missing `end`, mismatched port widths, undeclared signals.
- **Simulation mismatches during cross-language verification:** Check reset sequencing first — both the reference model and RTL must see identical reset behavior. Compare outputs only after reset is deasserted. If mismatches persist, narrow down to the first divergent cycle and trace signal values.
- **Sub-module interface mismatches in hierarchical designs:** Verify port names, widths, and directions match exactly between the instantiation and the sub-module definition. This is the most common failure mode for Tier 2 designs.
- **FSM gets stuck in unexpected state:** Check if all state transitions are covered, especially the default case. Missing transitions cause the FSM to remain in the current state or jump to an undefined value.
- **Reference model diverges on complex designs but works on simple ones:** Python models lose accuracy on designs with complex timing dependencies. Switch to SystemC or CXXRTL for cycle-accurate C++ simulation when Python reference models fail on hierarchical or CPU-level designs.

## Limitations

- LLMs currently achieve under 31% pass@1 on realistic Verilog generation and under 14% on Python reference model generation for complex modules. Always verify generated hardware code — never trust it without testbench validation.
- Hierarchical designs with deep instantiation trees (3+ levels) remain extremely challenging. Expect to decompose and generate one level at a time.
- Waveform-based debugging (using VCD data as context) is still unreliable — ChipBench found that providing waveform data actually degraded performance for most models. Prefer specification-based debugging over waveform-based approaches.
- Reference model generation for designs requiring precise multi-clock-domain behavior or asynchronous interfaces is beyond current LLM capabilities. These require manual implementation.
- The benchmark focuses on functional correctness, not synthesis quality (area, timing, power). Generated RTL may be functionally correct but suboptimal for physical implementation.

## Reference

**Paper:** [ChipBench: A Next-Step Benchmark for Evaluating LLM Performance in AI-Aided Chip Design](https://arxiv.org/abs/2601.21448v2) — Look for the tiered complexity taxonomy (Section 3), the four-category bug classification (Section 4), the Heterogeneous Test Engine cross-language verification methodology (Section 5), and the automated training data toolbox (Section 6). Code: https://github.com/zhongkaiyu/ChipBench