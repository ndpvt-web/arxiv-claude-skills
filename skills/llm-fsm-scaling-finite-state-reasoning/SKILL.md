---
name: "llm-fsm-scaling-finite-state-reasoning"
description: "Generate correct RTL (Verilog/SystemVerilog) implementations of finite-state machines from natural-language specifications using a structured YAML intermediate representation. Use when the user asks to: 'generate Verilog for this state machine', 'convert this FSM spec to RTL', 'write a hardware controller from this description', 'implement this protocol as a state machine in Verilog', 'create an FSM module from this specification', 'translate this state diagram to synthesizable code'."
---

# LLM-FSM: Finite-State Reasoning for RTL Code Generation

This skill enables Claude to translate natural-language hardware specifications into correct register transfer-level (RTL) implementations by decomposing the problem through a structured YAML intermediate representation of the underlying finite-state machine (FSM). Rather than attempting direct spec-to-Verilog generation (which degrades sharply as FSM complexity grows), this approach first extracts the FSM topology -- states, transitions, inputs, outputs, and conditions -- into a machine-readable YAML format, then compiles that representation into synthesizable Verilog. This two-stage pipeline (Spec -> YAML -> RTL) significantly reduces structural errors like missing transitions and incorrect timing semantics, based on findings from the LLM-FSM benchmark of 1,000 FSM problems across varying complexity tiers.

## When to Use

- When the user provides a natural-language description of a hardware controller, protocol handler, or sequencer and wants Verilog/SystemVerilog output
- When the user has a state diagram or state table and needs it translated to synthesizable RTL
- When the user asks to implement an FSM-based module (e.g., UART controller, SPI master, bus arbiter, traffic light controller)
- When the user provides a specification document describing state-dependent behavior and asks for hardware code
- When debugging an existing FSM implementation -- extract the intended FSM structure from the spec before comparing to the code
- When the user wants to verify that an RTL implementation matches a natural-language specification by reconstructing the FSM

## Key Technique

**The core insight from the LLM-FSM paper is that direct specification-to-RTL generation fails at scale.** Even the strongest LLMs drop from ~90% accuracy on simple FSMs (4-14 states) to ~66% on complex ones (27-59 states). The critical failure modes are: (1) missing or extra state transitions, (2) incorrect cycle-level timing semantics, (3) malformed output logic, and (4) syntax errors in the generated code. These errors compound as FSM complexity grows because the LLM must simultaneously reason about graph structure, signal assignments, and HDL syntax.

**The two-stage pipeline mitigates this by separating concerns.** Stage 1 (Spec -> YAML) focuses purely on FSM reasoning: identifying states, enumerating all transitions with their conditions, and mapping inputs to outputs per state. Stage 2 (YAML -> RTL) is a deterministic or near-deterministic compilation step that maps the structured YAML into a standard Verilog FSM template. This decomposition exploits the fact that LLMs are better at structured extraction than simultaneous reasoning-and-coding, and the YAML intermediate representation serves as a verifiable checkpoint.

**Test-time scaling through multi-trace sampling further improves reliability.** Generating multiple candidate solutions (k=8-16) and selecting the best one via syntax checking and functional verification consistently outperforms single-shot generation. For critical hardware, generate several candidates and verify each against the specification.

## Step-by-Step Workflow

1. **Parse the specification for FSM indicators.** Read the natural-language spec and identify: reset behavior, named states or operational modes, conditional transitions ("when X occurs, move to state Y"), input signals, output signals, and timing requirements (synchronous/asynchronous, clock edge sensitivity).

2. **Extract the FSM into structured YAML.** Create a YAML document with these required fields:
   - `module_name`: the Verilog module name
   - `clock` and `reset`: clock/reset signal names and polarity (active-high/low, posedge/negedge)
   - `inputs`: list of input signals with bit widths
   - `outputs`: list of output signals with bit widths and default values
   - `states`: enumeration of all states with a designated reset state
   - `transitions`: for each state, a list of `{condition, next_state, outputs}` tuples ordered by priority
   - `output_type`: Moore (outputs depend only on state) vs Mealy (outputs depend on state + inputs)

3. **Validate the FSM graph for completeness.** Check that: every state is reachable from the reset state, every state has at least one outgoing transition (including self-loops for hold conditions), there are no orphan states, and a default/else transition exists for each state to prevent latches.

4. **Select the RTL coding style.** Use a three-always-block pattern for synchronous FSMs: one block for state register update, one for next-state combinational logic, one for output combinational logic. This is the most portable and synthesis-friendly pattern.

5. **Generate the Verilog module.** Compile the YAML into Verilog using the three-always-block template: declare state encoding (one-hot or binary based on state count), implement the state register with synchronous reset, write the next-state logic as a case statement with priority-ordered conditions, and write the output logic.

6. **Add default assignments to prevent latches.** At the top of every combinational always block, assign default values to all outputs and next_state before the case statement. This is the single most common source of synthesis bugs in FSM code.

7. **Verify signal completeness.** Confirm that every input signal mentioned in the spec appears in at least one transition condition, every output signal is assigned in every state (either explicitly or via defaults), and bit widths match the specification.

8. **Generate a basic testbench.** Create a testbench that exercises: the reset sequence, at least one path through every state, boundary conditions on input signals, and the longest path through the FSM graph.

9. **If the FSM has >20 states, decompose hierarchically.** Split into sub-FSMs by identifying phases (initialization, operation, error handling, shutdown). Each phase becomes a sub-module with its own local FSM, and a top-level FSM manages phase transitions. This keeps each individual FSM within the complexity range where LLM generation is reliable.

10. **Review for common RTL pitfalls.** Check for: blocking vs non-blocking assignment correctness (`<=` in sequential, `=` in combinational), complete sensitivity lists (use `always @(*)` for combinational), no multiple drivers on the same signal, and correct reset initialization of all state-dependent registers.

## Concrete Examples

**Example 1: SPI Master Controller**

User: "Write a Verilog module for an SPI master that transmits 8 bits MSB-first. It should have an idle state waiting for a start signal, then clock out 8 bits on MOSI synchronized to SCK, then return to idle with a done signal."

Approach:
1. Extract FSM structure from the description
2. Build the YAML intermediate representation
3. Compile to three-always-block Verilog

YAML intermediate:
```yaml
module_name: spi_master
clock: {name: clk, edge: posedge}
reset: {name: rst_n, active: low}
inputs:
  - {name: start, width: 1}
  - {name: data_in, width: 8}
outputs:
  - {name: mosi, width: 1, default: 0}
  - {name: sck, width: 1, default: 0}
  - {name: done, width: 1, default: 0}
states: [IDLE, LOAD, SHIFT, DONE_ST]
reset_state: IDLE
output_type: mealy
transitions:
  IDLE:
    - {condition: "start", next: LOAD, outputs: {}}
    - {condition: "default", next: IDLE, outputs: {}}
  LOAD:
    - {condition: "default", next: SHIFT, outputs: {mosi: "shift_reg[7]"}}
  SHIFT:
    - {condition: "bit_cnt == 7", next: DONE_ST, outputs: {sck: "~sck_r", mosi: "shift_reg[7]"}}
    - {condition: "default", next: SHIFT, outputs: {sck: "~sck_r", mosi: "shift_reg[7]"}}
  DONE_ST:
    - {condition: "default", next: IDLE, outputs: {done: 1}}
```

Output:
```verilog
module spi_master (
    input  wire       clk,
    input  wire       rst_n,
    input  wire       start,
    input  wire [7:0] data_in,
    output reg        mosi,
    output reg        sck,
    output reg        done
);

    localparam IDLE    = 2'd0,
               LOAD    = 2'd1,
               SHIFT   = 2'd2,
               DONE_ST = 2'd3;

    reg [1:0] state, next_state;
    reg [7:0] shift_reg;
    reg [2:0] bit_cnt;

    // State register
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state     <= IDLE;
            shift_reg <= 8'd0;
            bit_cnt   <= 3'd0;
        end else begin
            state <= next_state;
            case (next_state)
                LOAD:    shift_reg <= data_in;
                SHIFT:   if (sck) shift_reg <= {shift_reg[6:0], 1'b0};
                default: shift_reg <= shift_reg;
            endcase
            if (state == SHIFT)
                bit_cnt <= bit_cnt + 3'd1;
            else
                bit_cnt <= 3'd0;
        end
    end

    // Next-state logic
    always @(*) begin
        next_state = state;
        case (state)
            IDLE:    next_state = start ? LOAD : IDLE;
            LOAD:    next_state = SHIFT;
            SHIFT:   next_state = (bit_cnt == 3'd7) ? DONE_ST : SHIFT;
            DONE_ST: next_state = IDLE;
        endcase
    end

    // Output logic
    always @(*) begin
        mosi = 1'b0;
        sck  = 1'b0;
        done = 1'b0;
        case (state)
            SHIFT: begin
                mosi = shift_reg[7];
                sck  = bit_cnt[0]; // Toggle SCK
            end
            DONE_ST: done = 1'b1;
        endcase
    end

endmodule
```

**Example 2: Traffic Light Controller**

User: "Implement a traffic light controller. Green for 30 cycles, yellow for 5, red for 30. Emergency input forces immediate red. After emergency clears, return to green."

Approach:
1. Identify 4 states: GREEN, YELLOW, RED, EMERGENCY
2. Build YAML with timer-based transitions
3. Generate Verilog with counter logic

YAML intermediate:
```yaml
module_name: traffic_light
clock: {name: clk, edge: posedge}
reset: {name: rst, active: high}
inputs:
  - {name: emergency, width: 1}
outputs:
  - {name: light, width: 2, default: 0}  # 00=off, 01=green, 10=yellow, 11=red
states: [GREEN, YELLOW, RED, EMERG]
reset_state: RED
output_type: moore
transitions:
  GREEN:
    - {condition: "emergency", next: EMERG, outputs: {light: "2'b11"}}
    - {condition: "timer == 29", next: YELLOW, outputs: {light: "2'b01"}}
    - {condition: "default", next: GREEN, outputs: {light: "2'b01"}}
  YELLOW:
    - {condition: "emergency", next: EMERG, outputs: {light: "2'b11"}}
    - {condition: "timer == 4", next: RED, outputs: {light: "2'b10"}}
    - {condition: "default", next: YELLOW, outputs: {light: "2'b10"}}
  RED:
    - {condition: "emergency", next: EMERG, outputs: {light: "2'b11"}}
    - {condition: "timer == 29", next: GREEN, outputs: {light: "2'b11"}}
    - {condition: "default", next: RED, outputs: {light: "2'b11"}}
  EMERG:
    - {condition: "!emergency", next: GREEN, outputs: {light: "2'b11"}}
    - {condition: "default", next: EMERG, outputs: {light: "2'b11"}}
```

Output: A three-always-block Verilog module with a 5-bit timer that resets on state transitions, priority checking for the emergency input in every state's next-state logic, and Moore-style output assignment based solely on current state.

**Example 3: Hierarchical Decomposition for Complex Protocol**

User: "Implement an I2C master controller with support for start, address, write data, read data with ACK/NACK, repeated start, and stop conditions."

Approach:
1. Recognize this exceeds 20 states -- decompose hierarchically
2. Identify phases: IDLE, START, ADDRESS, WRITE, READ, STOP
3. Create a top-level FSM managing phase transitions
4. Implement each phase as a sub-FSM module
5. Generate YAML and Verilog for each sub-module independently

Top-level YAML:
```yaml
module_name: i2c_master_top
states: [IDLE, START, ADDR, WRITE, READ, REP_START, STOP]
reset_state: IDLE
transitions:
  IDLE:
    - {condition: "go", next: START}
  START:
    - {condition: "start_done", next: ADDR}
  ADDR:
    - {condition: "addr_done && rw == 0", next: WRITE}
    - {condition: "addr_done && rw == 1", next: READ}
  # ... each phase sub-FSM handles its own internal states
```

Each sub-module (e.g., `i2c_start_gen`, `i2c_addr_phase`) is a self-contained FSM with <10 states, keeping every module within the reliable generation range.

## Best Practices

- **Do:** Always produce the YAML intermediate before writing Verilog. This catches missing transitions and ambiguous spec language before they become RTL bugs.
- **Do:** Use the three-always-block pattern (state register, next-state combinational, output combinational) as the default FSM coding style. It is universally supported by synthesis tools and easiest to verify.
- **Do:** Add default assignments at the top of every combinational block before the case statement. This single practice eliminates the most common class of FSM synthesis bugs (inferred latches).
- **Do:** Decompose any FSM with more than ~20 states into hierarchical sub-FSMs. LLM accuracy on FSMs with 27+ states drops by 15-25 percentage points.
- **Avoid:** Generating Verilog directly from a complex natural-language spec without the YAML extraction step. Direct generation has significantly higher error rates for FSMs beyond trivial complexity.
- **Avoid:** Using `always @(posedge clk)` for combinational next-state or output logic. This introduces unintended pipeline stages and breaks the FSM timing model.
- **Avoid:** Omitting self-loop transitions for states that should hold. Every state must have a defined behavior for every possible input combination, or synthesis will infer latches.

## Error Handling

| Error Type | Symptom | Fix |
|---|---|---|
| **Missing transitions** | Simulation hangs in a state or synthesis warns about latches | Add explicit default/else transitions for every state in the YAML before generating RTL |
| **Timing mismatch** | Outputs appear one cycle early/late | Verify Moore vs Mealy classification; Moore outputs are registered (one cycle delay), Mealy are combinational (same cycle) |
| **Unreachable states** | Dead code warnings from synthesis | Trace the state graph from reset_state; remove any state with no incoming path |
| **Multiple drivers** | Synthesis error on signal assignment | Ensure each output is assigned in exactly one always block; do not split output logic across blocks |
| **Reset incompleteness** | Simulation starts in unknown state | Initialize all registers (state, counters, shift registers) in the reset branch of the sequential always block |
| **Syntax errors** | Compilation failures in generated Verilog | Validate parentheses, semicolons, and `begin/end` blocks; use a linting pass before delivery |

## Limitations

- **Datapath-heavy designs**: This technique optimizes for control-dominant FSMs. Designs where the complexity is in arithmetic datapath operations (ALUs, DSP pipelines) rather than state transitions will not benefit from FSM decomposition.
- **Asynchronous FSMs**: The approach assumes synchronous, single-clock-domain FSMs. Asynchronous or multi-clock designs require additional considerations (clock domain crossing, metastability) not covered here.
- **Analog/mixed-signal**: FSM extraction does not apply to analog behavioral descriptions.
- **Ambiguous specifications**: If the natural-language spec is genuinely ambiguous about state behavior, the YAML extraction step will surface the ambiguity but cannot resolve it -- ask the user for clarification.
- **Very large FSMs (>50 states)**: Even with hierarchical decomposition, extremely large FSMs may require iterative refinement. Generate, simulate, identify failing paths, and fix incrementally.

## Reference

[LLM-FSM: Scaling Large Language Models for Finite-State Reasoning in RTL Code Generation](https://arxiv.org/abs/2602.07032v1) -- Wu et al., 2026. Key takeaway: the Spec -> YAML -> RTL two-stage pipeline outperforms direct generation, and FSM complexity scaling reveals that structured intermediate representations are essential for reliable hardware code generation by LLMs.