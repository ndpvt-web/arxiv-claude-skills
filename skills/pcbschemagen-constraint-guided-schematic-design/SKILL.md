---
name: "pcbschemagen-constraint-guided-schematic-design"
description: "Generate PCB schematics from natural language using constraint-guided LLM code generation with knowledge-graph verification. Use when the user says 'generate a PCB schematic', 'design a circuit board', 'create a KiCad schematic from description', 'convert circuit requirements to netlist', 'automate schematic design', or 'generate SKiDL code for a circuit'."
---

This skill enables Claude to generate complete PCB schematics from natural-language circuit descriptions by applying the PCBSchemaGen framework: a training-free, constraint-guided pipeline that produces executable SKiDL Python code, validates it against a knowledge graph encoding real IC datasheet constraints, and outputs KiCad-compatible netlist files. The technique replaces manual schematic entry with an LLM agent loop that reasons about pin-role semantics, topological correctness, and power/analog/digital signal domains in a single unified workflow.

## When to Use

- When the user asks to generate a PCB schematic or circuit design from a textual specification (e.g., "design a buck converter with LM5146")
- When the user needs to produce KiCad netlists or SKiDL Python code for a circuit
- When the user wants to automate schematic entry for circuits involving specific IC components with known pin configurations
- When the user asks to validate an existing circuit netlist against IC datasheet constraints
- When the user wants to build a knowledge graph from IC datasheets to encode pin-role semantics and design rules
- When the user needs to set up an iterative LLM-driven code generation pipeline with domain-specific verification feedback
- When the user asks to convert component datasheets into structured constraint representations for automated design

## Key Technique

PCBSchemaGen introduces a two-part architecture: (1) an LLM agent that generates executable SKiDL Python code from natural-language circuit specifications, and (2) a constraint-guided verification framework built on a knowledge graph (KG) derived from real IC datasheets. The KG is defined as a tuple `KG = (C, R, A, Phi, I)` where C is the set of component types, R is 36 pin-role types (power, ground, gate-drive, Kelvin-source, signal I/O, etc.), A maps components to their attributes, Phi encodes constraint functions, and I defines isolation boundaries. This compact representation compresses a typical 16k-token datasheet into ~300 tokens per component -- a 30-70x context reduction that makes RAG practical.

Verification works by constructing a bipartite graph `G = (V_C union V_N, E)` from the generated SKiDL code, where components and nets form disjoint vertex sets. Design rules are encoded as reference subgraph patterns. Checking correctness reduces to subgraph isomorphism via the VF2 algorithm: a design passes if and only if an injective mapping exists from every reference rule graph into the generated circuit graph. When verification fails, the system produces structured error feedback (error type, pin/net location, component values) that the LLM uses for self-correction in up to 3 retry iterations.

The prompting strategy combines in-context learning (ICL) with worked SKiDL examples, chain-of-thought (CoT) reasoning about circuit connectivity, and domain-specific prompt templates that inject KG constraints. Ablation studies show ICL is critical -- removing it collapses hard-task accuracy from 78% to 44%. The full pipeline achieves 86-88% Pass@1 across 23 benchmark tasks spanning digital, analog, and power domains, at ~37x the speed of human experts.

## Step-by-Step Workflow

1. **Parse the circuit specification** into structured requirements: identify all IC components by part number, classify signal domains (digital, analog, power), and list functional blocks (e.g., "buck converter stage", "gate driver", "feedback network").

2. **Build or load the knowledge graph** for each IC component. For each part, extract from its datasheet: pin names and numbers, pin-role classifications (from the 36-role taxonomy: VIN, PGND, SW, BOOT, FB, COMP, EN, SS, etc.), absolute maximum ratings, and typical application circuit topology. Encode as JSON:
   ```json
   {
     "component": "LM5146",
     "pins": [
       {"number": 1, "name": "VIN", "role": "power_input"},
       {"number": 2, "name": "PGND", "role": "power_ground"},
       {"number": 3, "name": "SW", "role": "switch_node"}
     ],
     "constraints": [
       {"rule": "decoupling_cap", "pins": ["VIN", "PGND"], "value": "4.7uF"},
       {"rule": "boot_cap", "pins": ["BOOT", "SW"], "value": "100nF"}
     ]
   }
   ```

3. **Construct the domain-specific prompt** by combining: (a) a system prompt defining SKiDL syntax and conventions, (b) 2-3 in-context examples of working SKiDL code for similar circuit types, (c) the KG-derived constraint summary for each IC, and (d) the user's natural-language specification. Keep total prompt under the context window by using the compressed KG representation, not raw datasheets.

4. **Generate SKiDL Python code** via the LLM. The code must import `skidl`, instantiate `Part` objects with the correct library and footprint, create `Net` objects for each electrical connection, and wire pins to nets. Structure the code in logical blocks: power input, IC instantiation, passive component placement, signal routing.

5. **Execute syntax validation** by running the generated Python code. Catch import errors, SKiDL pin-resolution failures, and Python exceptions. If syntax fails, feed the traceback back to the LLM with the original code for correction.

6. **Build the bipartite connection graph** from the successfully-executed SKiDL netlist. Extract all component-to-net edges. Construct the reduced net-role topology graph by mapping each pin to its KG-defined role, collapsing vendor-specific naming into the standardized 36-role taxonomy.

7. **Run subgraph isomorphism verification** using VF2. For each design rule in the KG (e.g., "VIN must connect to decoupling capacitor to PGND"), check that the reference subgraph pattern maps injectively into the generated circuit graph. Collect all violations with specific error locations.

8. **Generate structured feedback** for any failures. Categorize errors as: connection errors (missing/wrong pin wiring), topological errors (subgraph mismatch), or constraint violations (wrong component values). Include the specific pin, net, and expected-vs-actual values.

9. **Iterate with LLM self-correction** up to 3 times. Feed the error feedback into the LLM along with the previous code, asking it to fix only the identified issues. Re-run verification after each correction.

10. **Export the validated design** to KiCad format. Compile the passing SKiDL code to `.kicad_sch` schematic files and optionally generate SVG visual representations for review.

## Concrete Examples

**Example 1: Buck Converter Schematic Generation**

User: "Generate a PCB schematic for a 12V-to-3.3V buck converter using the TPS54360B with input/output capacitors and a feedback resistor divider."

Approach:
1. Parse requirements: TPS54360B IC, 12V input, 3.3V output, passive components needed.
2. Load/build KG for TPS54360B: VIN (power_input), GND (power_ground), SW (switch_node), BOOT (bootstrap), EN (enable), FB (feedback), COMP (compensation), SS (soft_start).
3. Generate SKiDL code:

```python
from skidl import *

# IC
U1 = Part("TPS54360B", "TPS54360BDDA", footprint="SOIC-8")

# Passive components
C_in = Part("Device", "C", value="10uF", footprint="C_0805")
C_out = Part("Device", "C", value="22uF", footprint="C_0805")
C_boot = Part("Device", "C", value="100nF", footprint="C_0402")
L1 = Part("Device", "L", value="10uH", footprint="L_1210")
R_top = Part("Device", "R", value="49.9k", footprint="R_0402")
R_bot = Part("Device", "R", value="16.5k", footprint="R_0402")

# Nets
vin, vout, gnd = Net("VIN"), Net("VOUT"), Net("GND")
sw_node, fb_node = Net("SW"), Net("FB_DIV")

# Power input
vin += U1["VIN"], C_in[1]
gnd += U1["GND"], C_in[2], C_out[2], R_bot[2]

# Switch node and output
sw_node += U1["SW"], L1[1]
vout += L1[2], C_out[1], R_top[1]

# Bootstrap
Net("BOOT") += U1["BOOT"], C_boot[1]
sw_node += C_boot[2]

# Feedback divider
fb_node += U1["FB"], R_top[2], R_bot[1]
```

4. Verify: bipartite graph confirms VIN-decoupling, BOOT-capacitor, and FB-divider subgraph patterns all match.
5. Export to `.kicad_sch`.

Output: A validated KiCad schematic file with correct pin assignments, proper decoupling, and a feedback network producing 3.3V output.

---

**Example 2: Knowledge Graph Construction from Datasheet**

User: "Build a component knowledge graph entry for the INA228 power monitor IC from its datasheet."

Approach:
1. Extract pin table from INA228 datasheet (16-pin TSSOP).
2. Classify each pin into the 36-role taxonomy:

```json
{
  "component": "INA228",
  "package": "TSSOP-16",
  "pins": [
    {"number": 1, "name": "VS+", "role": "sense_positive"},
    {"number": 2, "name": "VS-", "role": "sense_negative"},
    {"number": 3, "name": "VBS", "role": "bus_voltage_sense"},
    {"number": 5, "name": "SCL", "role": "digital_clock"},
    {"number": 6, "name": "SDA", "role": "digital_data"},
    {"number": 7, "name": "ALERT", "role": "digital_output_od"},
    {"number": 10, "name": "GND", "role": "power_ground"},
    {"number": 11, "name": "VDD", "role": "power_input"},
    {"number": 12, "name": "A0", "role": "address_select"},
    {"number": 13, "name": "A1", "role": "address_select"}
  ],
  "constraints": [
    {"rule": "decoupling_cap", "pins": ["VDD", "GND"], "value": "100nF"},
    {"rule": "shunt_resistor", "pins": ["VS+", "VS-"], "value": "10mR"},
    {"rule": "i2c_pullup", "pins": ["SCL", "SDA"], "target_net": "VDD"}
  ]
}
```

3. Validate completeness: all pins classified, mandatory constraints (decoupling, shunt) present.

Output: A structured KG entry ready for use in constraint-guided schematic generation.

---

**Example 3: Verifying an Existing Circuit Design**

User: "Check if this SKiDL netlist for a half-bridge gate driver is correctly wired against the UCC21520 datasheet."

Approach:
1. Load the user's SKiDL code and execute it to build the netlist.
2. Load the KG entry for UCC21520 (pin roles: VCC, GND, INA+, INA-, INB+, INB-, OUTA, OUTB, VDDA, VSSA, VDDB, VSSB, DT).
3. Construct the bipartite connection graph from the netlist.
4. Define reference subgraph patterns:
   - Bootstrap capacitor: VDDA--cap--OUTA (high-side supply)
   - Decoupling: VCC--cap--GND (logic supply)
   - Output isolation: VSSA separate from VSSB (independent return paths)
5. Run VF2 subgraph isomorphism for each rule.
6. Report results:

```
Verification Results:
[PASS] VCC decoupling capacitor: 100nF between VCC(pin 1) and GND(pin 2)
[PASS] High-side bootstrap: 100nF between VDDA(pin 11) and OUTA(pin 12)
[FAIL] Low-side decoupling missing: No capacitor found between VDDB(pin 7) and VSSB(pin 8)
[PASS] Dead-time pin: DT(pin 5) connected to resistor to GND

Action required: Add 100nF capacitor between VDDB and VSSB for low-side gate driver supply.
```

## Best Practices

- **Do:** Compress IC datasheet information into the KG tuple format (~300 tokens per component) rather than injecting raw datasheet text into prompts. This 30-70x context reduction is what makes multi-component designs feasible within LLM context windows.
- **Do:** Always include 2-3 in-context SKiDL code examples in the prompt that match the circuit domain (power, analog, or digital). ICL is critical -- without it, complex task accuracy drops by ~35 percentage points.
- **Do:** Use the 36 pin-role taxonomy to normalize vendor-specific pin names. This decouples verification logic from naming conventions (e.g., "PGND" vs "GND" vs "AGND" all map to appropriate ground roles).
- **Do:** Provide full structured feedback (error type + pin/net location + expected values) when the LLM needs to self-correct. Full feedback significantly outperforms binary pass/fail on medium and hard tasks.
- **Avoid:** Generating schematics without verification. Even high-quality LLMs produce pin-wiring errors in ~15% of first attempts. Always run the subgraph isomorphism check before exporting.
- **Avoid:** Skipping the bipartite graph construction step by trying to validate connectivity through string matching on net names. Graph-based verification catches topological errors that text-based checks miss.

## Error Handling

| Error Type | Cause | Recovery Strategy |
|---|---|---|
| SKiDL pin resolution failure | Part number or pin name mismatch against library | Check KiCad library availability; fall back to generic parts with explicit pin numbers |
| Subgraph isomorphism failure | Missing or incorrect connections | Feed specific violation (pin, net, expected pattern) back to LLM for targeted correction |
| Component value constraint violation | Wrong passive value (e.g., boot cap too large) | Include the KG constraint range in feedback; LLM adjusts value |
| Isolation boundary violation | Analog/digital ground mixing or cross-domain coupling | Flag the specific nets that violate isolation rules; restructure ground topology |
| Retry exhaustion (3 attempts failed) | Fundamental misunderstanding of circuit topology | Escalate to user with the best-attempt code and remaining violations; suggest manual review of the specific failing subgraph |

When verification produces multiple simultaneous errors, prioritize fixes in this order: power connections first (safety-critical), then signal connectivity, then passive component values. This ordering prevents cascading failures during iterative correction.

## Limitations

- **Component library coverage:** The approach requires KiCad symbol libraries for every IC used. Components not in the library need manual symbol creation before SKiDL can reference them.
- **Knowledge graph maintenance:** KG entries must be manually constructed or extracted from datasheets for each new IC. There is no fully automated datasheet-to-KG pipeline yet.
- **Analog precision circuits:** While the framework handles typical analog blocks (op-amp feedback, voltage dividers), it does not verify analog performance metrics like gain, bandwidth, or stability. It checks topology, not simulation.
- **Layout constraints:** The system generates schematics (netlists), not PCB layouts. Physical constraints like trace impedance, thermal relief, and component placement are out of scope.
- **Single-board scope:** The framework targets individual PCB schematics. Multi-board systems with inter-board connectors and harness definitions are not supported.
- **LLM dependency:** Generation quality varies across LLM providers. The benchmark shows significant performance differences between models, with Gemini 3 Pro/Flash at 86-88% Pass@1 vs. lower-tier models around 60-70%.

## Reference

[PCBSchemaGen: Constraint-Guided Schematic Design via LLM for Printed Circuit Boards](https://arxiv.org/abs/2602.00510v1) -- Focus on Section 3 (Knowledge Graph formalization), Section 4 (Subgraph Isomorphism verification algorithm), and Section 5 (ablation studies showing the impact of ICL, CoT, and feedback granularity on generation accuracy). Code: [github.com/HZou9/PCBSchemaGen](https://github.com/HZou9/PCBSchemaGen).