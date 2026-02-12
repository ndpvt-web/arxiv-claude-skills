---
name: "veri-sure-contract-aware-multi-agent-framework"
description: "Generate functionally correct RTL/Verilog code using a contract-aware multi-agent workflow with formal verification. Triggers: 'generate Verilog module', 'write RTL with verification', 'design hardware with formal proof', 'create verified Verilog', 'contract-driven RTL generation', 'fix Verilog with dependency slicing'"
---

# Veri-Sure: Contract-Aware Multi-Agent Framework for Verified RTL Code Generation

This skill enables Claude to generate functionally correct Register-Transfer Level (RTL) Verilog/SystemVerilog code by applying the Veri-Sure methodology: a structured workflow where a **design contract** anchors the specification, a **multi-branch verification pipeline** (simulation traces, assertions, equivalence checking) validates correctness beyond basic testbenches, and a **static dependency slicing** mechanism performs surgical repairs when verification fails. The result is RTL code that converges toward silicon-grade correctness rather than code that merely passes a handful of test cases.

## When to Use

- When the user asks to generate a Verilog or SystemVerilog module from a natural-language specification (e.g., "write a FIFO controller in Verilog")
- When the user has existing RTL code that fails simulation or formal checks and needs targeted, regression-free repairs
- When the user wants to add SystemVerilog Assertions (SVA) or formal properties to an existing design
- When the user needs to verify that a modified RTL module is functionally equivalent to a reference implementation
- When the user is debugging a hardware design and wants localized fixes that do not break previously-passing behavior
- When the user asks for a multi-agent or iterative approach to hardware code generation with verification in the loop

## Key Technique

**Design Contracts as the Single Source of Truth.** Traditional LLM-based RTL generation suffers from semantic drift: each iteration of code generation or repair subtly reinterprets the original intent, accumulating mismatches. Veri-Sure addresses this by extracting a **design contract** from the specification before any code is written. The contract captures input/output port declarations with explicit bit widths, behavioral invariants (e.g., "output must be zero within one cycle of reset assertion"), timing constraints, and interface protocols. Every subsequent step -- generation, verification, and patching -- references this immutable contract, preventing agents (or iteration rounds) from silently redefining requirements.

**Multi-Branch Verification Beyond Simulation.** Rather than relying solely on testbench simulation, Veri-Sure runs three complementary verification strategies in parallel: (1) **trace-driven temporal analysis** that replays stimulus sequences and checks signal transitions against expected state machines, (2) **assertion-based checking** using SVA properties derived from the contract to catch violations at specific cycle points, and (3) **boolean equivalence proofs** that use SAT/SMT reasoning to confirm the implementation matches a reference model under all input combinations. A design is considered correct only when all three branches pass.

**Static Dependency Slicing for Surgical Patches.** When verification fails, the framework does not regenerate the entire module. Instead, it performs backward slicing from the failing signal to identify root-cause assignments, then forward slicing to determine dependent logic. Only the code within this slice is modified, preserving all previously verified behavior. This reduces patch scope by roughly 70% compared to full regeneration and prevents the "fix one bug, introduce two" pattern that plagues iterative LLM debugging.

## Step-by-Step Workflow

1. **Extract the design contract.** Parse the user's natural-language specification and produce a structured contract containing: module name, port list with directions and widths, clock/reset conventions, behavioral invariants as English-to-formal property mappings, and any protocol or timing constraints. Present this contract to the user for confirmation before proceeding.

2. **Generate the initial RTL implementation.** Write a Verilog/SystemVerilog module that satisfies the contract. Follow standard synthesizable coding patterns: use `always_ff` for sequential logic, `always_comb` for combinational logic, explicit reset handling, and clear signal naming that maps back to contract ports.

3. **Derive verification artifacts from the contract.** For each behavioral invariant in the contract, produce: (a) a SystemVerilog Assertion (`assert property`) encoding the temporal property, (b) at least two directed test vectors exercising the property's positive and negative cases, and (c) where a reference model exists or can be constructed, a combinational equivalence check specification.

4. **Run trace-driven temporal analysis.** Write a testbench that applies the directed vectors, captures output traces cycle-by-cycle, and compares them against expected waveforms derived from the contract. Flag any cycle where an output diverges from the specification.

5. **Run assertion-based checking.** Embed the SVA properties into the testbench or a bind file. Simulate (or formally check with an available tool) and report any assertion violations with the failing property name, the triggering cycle, and the signal values at failure.

6. **Run equivalence checking (when applicable).** If a reference model or golden netlist exists, compare the generated module against it. Report any input combination that produces differing outputs as a counterexample.

7. **If all checks pass, deliver the verified module.** Return the RTL code, the contract, and the verification artifacts (assertions + testbench) as a complete deliverable.

8. **If any check fails, perform dependency slicing.** Identify the failing signal from the verification report. Trace backward through assignments to find root-cause statements. Trace forward to find dependent logic. Mark the union of these statements as the **patch region**.

9. **Generate a targeted patch.** Rewrite only the statements within the patch region to fix the violation, referencing the original contract invariant that was broken. Do not modify code outside the slice. Annotate the patch with the contract clause it addresses.

10. **Re-run all verification branches.** Repeat steps 4-6 on the patched code. If new failures appear, return to step 8. Limit iteration to 3 rounds; if convergence is not reached, report remaining failures with full diagnostic context to the user.

## Concrete Examples

**Example 1: 8-Bit Register with Async Reset**

User: "Write a Verilog module for an 8-bit register with active-high asynchronous reset, write enable, and synchronous load."

Approach:
1. Extract the design contract:
```
Module: reg8
Ports: clk (input), rst (input, async active-high), wen (input), din[7:0] (input), dout[7:0] (output reg)
Invariant I1: When rst is asserted, dout = 0 within the same cycle (async)
Invariant I2: When rst is low and wen is high, dout captures din on the next rising edge of clk
Invariant I3: When rst is low and wen is low, dout retains its previous value
```

2. Generate RTL:
```verilog
module reg8 (
    input  wire       clk,
    input  wire       rst,
    input  wire       wen,
    input  wire [7:0] din,
    output reg  [7:0] dout
);
    always @(posedge clk or posedge rst) begin
        if (rst)
            dout <= 8'b0;
        else if (wen)
            dout <= din;
    end
endmodule
```

3. Derive assertions:
```systemverilog
// I1: Async reset clears output
property p_async_reset;
    @(posedge rst) 1'b1 |-> ##0 (dout == 8'b0);
endproperty
assert property (p_async_reset);

// I2: Write-enable captures data
property p_write;
    @(posedge clk) disable iff (rst)
    wen |=> (dout == $past(din));
endproperty
assert property (p_write);

// I3: Hold when not writing
property p_hold;
    @(posedge clk) disable iff (rst)
    !wen |=> (dout == $past(dout));
endproperty
assert property (p_hold);
```

All assertions pass; deliver module with contract and testbench.

---

**Example 2: Repairing a Broken FSM via Dependency Slicing**

User: "My 3-state FSM for a traffic light controller fails — the yellow state is never reached."

Approach:
1. Establish contract from spec: states are `GREEN -> YELLOW -> RED -> GREEN`, each held for a timer count.
2. Read existing code. Identify the failing signal: `state` never equals `YELLOW`.
3. Backward-slice from the `YELLOW` assignment:
```
Slice target: state <= YELLOW
Root cause found: transition condition `(state == GREEN && timer_done)`
assigns `state <= RED` instead of `state <= YELLOW`
```
4. The patch region is the single case branch for `GREEN`:
```verilog
// BEFORE (patch region only):
GREEN: if (timer_done) state <= RED;

// AFTER:
GREEN: if (timer_done) state <= YELLOW;
```
5. Re-run verification: trace analysis now shows GREEN->YELLOW->RED->GREEN cycle. Assertions on state sequencing pass. Deliver patched code.

---

**Example 3: FIFO with Formal Equivalence**

User: "Generate a synchronous FIFO (depth 16, width 8) and prove it matches my golden reference."

Approach:
1. Contract: 16-deep, 8-wide FIFO with `wr_en`, `rd_en`, `full`, `empty` flags, pointer-based implementation.
2. Generate RTL using binary pointers with an extra MSB for full/empty disambiguation.
3. Derive assertions: `full` when write pointer wraps to equal read pointer, `empty` when pointers are exactly equal, no read when empty, no write when full.
4. Run equivalence check against golden reference module:
```
Counterexample found: when wr_en and rd_en asserted simultaneously at full,
generated module asserts both full and empty. Golden module keeps full=1, empty=0.
```
5. Dependency slice: backward from `empty` flag logic, root cause is the empty condition not excluding the simultaneous read-write case.
6. Patch the `empty` assignment:
```verilog
// BEFORE:
assign empty = (wr_ptr == rd_ptr);
// AFTER:
assign empty = (wr_ptr == rd_ptr) && !(wr_en && !rd_en);
```
Wait -- re-examine contract. Correct approach: empty is `wr_ptr[ADDR_W] == rd_ptr[ADDR_W] && wr_ptr[ADDR_W-1:0] == rd_ptr[ADDR_W-1:0]`. Root cause was a missing MSB comparison. Patch accordingly and re-verify. All branches pass.

## Best Practices

- **Do:** Always write the design contract first and get user confirmation before generating any RTL. The contract is the anchor that prevents drift across iterations.
- **Do:** Generate SVA properties for every behavioral invariant in the contract. Assertions are cheap to write and catch entire classes of bugs that directed tests miss.
- **Do:** Keep patches minimal. Use dependency slicing to identify the exact statements that need to change. Never regenerate an entire module to fix a localized bug.
- **Do:** Run all three verification branches (traces, assertions, equivalence) even if one passes. Different strategies catch different bug classes; a design that passes simulation can still fail formal checks.
- **Avoid:** Regenerating the full module after a verification failure. This discards correct logic and introduces new regressions.
- **Avoid:** Writing assertions that are trivially true or that restate the implementation rather than the specification. Assertions must be derived from the contract, not from the code.
- **Avoid:** Iterating more than 3 patch rounds without escalating to the user. If convergence is not reached, the contract or specification likely has an ambiguity that requires human judgment.

## Error Handling

- **Ambiguous specification:** If the natural-language spec is underspecified (e.g., no reset behavior mentioned), ask the user to clarify before finalizing the contract. Do not guess timing or protocol behavior.
- **Contradictory invariants:** If two contract invariants conflict (e.g., "output must be zero on reset" vs. "output must hold previous value on reset"), flag the contradiction and request resolution.
- **Tool unavailability:** If no formal verification tool (Yosys, SymbiYosys, JasperGold) is available in the user's environment, fall back to assertion-based simulation using Icarus Verilog or Verilator. Note the reduced coverage in the deliverable.
- **Patch divergence:** If a patch fixes one assertion but breaks another, widen the dependency slice to include the newly failing signal's cone of logic and generate a joint patch.
- **Unsynthesizable constructs:** If the generated code uses non-synthesizable SystemVerilog (e.g., `$display`, `initial` blocks with delays), flag it and rewrite using synthesizable alternatives.

## Limitations

- **Formal verification requires tooling.** Full equivalence proofs need SAT/SMT-based tools (e.g., SymbiYosys, Yosys-SMTBMC). Without them, the workflow degrades to simulation-only, losing the "beyond simulation" guarantee.
- **Scalability of equivalence checking.** Boolean equivalence proofs are computationally expensive for large designs (>10K gates). For complex modules, assertion-based checking is more practical than full equivalence.
- **Contract quality depends on specification quality.** If the user's description is vague or incorrect, the contract will inherit those flaws. The framework catches implementation bugs, not specification bugs.
- **LLM-generated patches are not formally proven correct.** The verification loop catches bad patches, but convergence is not guaranteed. Complex bugs in deeply pipelined or multi-clock-domain designs may exceed the patch mechanism's reach.
- **No support for analog or mixed-signal designs.** This workflow applies only to digital RTL (Verilog/SystemVerilog), not SPICE-level or AMS design.

## Reference

**Paper:** [Veri-Sure: A Contract-Aware Multi-Agent Framework with Temporal Tracing and Formal Verification for Correct RTL Code Generation](https://arxiv.org/abs/2601.19747v1) (Liu, Zhou, Jiang, 2026). Look for Section III (framework architecture and contract formalization), Section IV (static dependency slicing algorithm), and Section V (multi-branch verification pipeline details).