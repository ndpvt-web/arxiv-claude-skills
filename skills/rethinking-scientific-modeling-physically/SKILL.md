---
name: "rethinking-scientific-modeling-physically"
description: "Generate physics-consistent, simulation-executable structural engineering code using constraint-oriented alignment and verification-driven evaluation. Use when: 'generate an OpenSees model', 'create a structural analysis script', 'build a seismic-resistant frame model', 'write simulation-ready building code', 'check my structural model for physical consistency', 'debug my OpenSeesPy script'."
---

# Physics-Consistent Simulation-Executable Code Generation

This skill enables Claude to generate structurally sound, simulation-executable code for computational engineering — specifically reinforced concrete frame models in OpenSeesPy. It applies the AutoBM framework's three-pillar approach: domain knowledge construction (formalizing engineering constraints before generation), constraint-oriented alignment (enforcing API compliance and physical laws during generation), and verification-driven evaluation (closed-loop validation of executability, modal period accuracy, and code compliance after generation). The core insight is that physics-constrained generation with multi-granularity verification vastly outperforms unconstrained code generation, even from frontier models.

## When to Use

- When the user asks to generate a structural analysis model (e.g., "create an OpenSeesPy model for a 5-story office building")
- When the user provides building specifications (function, dimensions, seismic zone) and needs executable simulation code
- When the user has an OpenSeesPy script with runtime errors or physically implausible results and needs debugging
- When the user needs to verify that generated structural code satisfies seismic design codes (e.g., GB50011, ASCE 7, Eurocode 8)
- When the user asks for a complete structural analysis pipeline: geometry, materials, loads, modal analysis, and compliance checks
- When the user wants to parameterize structural models across building types, story counts, or seismic intensities

## Key Technique

**The Problem:** Frontier LLMs generate structurally plausible-looking code that fails in execution. Benchmarks show Claude, GPT, and Gemini achieve only 18-20% strict compliance (simultaneous executability + period accuracy + code compliance) on structural modeling tasks. The dominant failure mode (87.5% of errors) is syntax/API violations; the rest split between compliance violations (excessive drift) and modal period extraction errors.

**The Solution — Three-Stage Pipeline:**

1. **Domain Knowledge Construction (CivilInstruct):** Rather than treating structural code as generic programming, the approach formalizes four knowledge layers: (a) API documentation with line-by-line annotations, (b) parameterized full-workflow code (~600 lines covering modeling through verification), (c) bug-fix chain-of-thought pairs capturing root causes and resolutions, and (d) physics-informed expert examples with ground-truth structural periods. This layered knowledge forces the generator to internalize not just syntax but engineering semantics.

2. **Constraint-Oriented Generation with Multi-Granularity Hybrid Reward (MGHR):** Generation is guided by three reward signals weighted by importance: format compliance via regex (5%), logical completeness via hierarchical AST analysis of three API tiers — topology/T1, boundary-load/T2, analysis-solver/T3 (25%), and sandbox-executed physical consistency including structural period error (70%). The heavy weighting toward execution-verified physical consistency is what distinguishes this from standard code generation — the model learns that syntactically valid but physically wrong code scores near zero.

3. **Verification-Driven Evaluation (MBEval):** Three pass@k metrics provide closed-loop validation: `pass@k_period` (fundamental period within 30% of ground truth), `pass@k_compliance` (design code adherence verified by keyword analysis), and `pass@k_strict` (all three conditions simultaneously). Code is executed in a sandboxed environment with time limits, and partial credit is awarded for runtime failures based on execution progress (lines executed / total lines).

## Step-by-Step Workflow

1. **Parse the building specification into structured parameters.** Extract: building function (office, residential, hospital, etc.), number of stories (typically 3-7), story height (3.0-4.0m), plan dimensions (length x width in meters), seismic intensity (e.g., 7, 7.5, 8, 8.5, 9 on Chinese scale or equivalent MMI/PGA), and peak ground acceleration (0.10g-0.40g). If any parameter is missing, ask the user or select code-compliant defaults.

2. **Select material constitutive models and compute section properties.** For reinforced concrete frames, define concrete (e.g., Concrete01 or Concrete02 in OpenSeesPy with compressive strength, strain at peak, crushing strain) and steel (e.g., Steel02 with yield strength, elastic modulus, hardening ratio). Compute beam and column cross-sections based on story count and seismic demand — larger sections for lower stories and higher seismic zones.

3. **Generate the structural geometry: nodes and elements.** Define nodes at every beam-column intersection with coordinates. Create nonlinearBeamColumn or forceBeamColumn elements for beams and columns. Assign fiber sections to elements with concrete core, concrete cover, and steel reinforcement layers. Ensure node numbering is consistent and all elements connect to valid nodes.

4. **Apply boundary conditions and loads.** Fix base nodes (typically all DOFs for fixed supports). Apply gravity loads: dead load + live load as distributed beam loads converted to nodal forces using tributary area. Define mass at each floor level consistent with representative gravity loads (typically 1.0D + 0.5L for seismic mass).

5. **Configure the analysis pipeline in three API tiers.**
   - **Tier 1 (Topology):** `model`, `node`, `element`, `section`, `uniaxialMaterial` — the structural skeleton.
   - **Tier 2 (Boundary/Load):** `fix`, `timeSeries`, `pattern`, `load`, `mass` — the loading environment.
   - **Tier 3 (Analysis/Solver):** `constraints`, `numberer`, `system`, `test`, `algorithm`, `integrator`, `analysis`, `eigen`, `recorder` — the solution engine.
   Verify that every tier is complete before proceeding. Missing any tier causes runtime failure.

6. **Implement eigenvalue analysis for the fundamental period.** Call `eigen(numModes)` after gravity analysis. Extract eigenvalues, compute periods as T = 2*pi/sqrt(eigenvalue). Print the fundamental period T1 explicitly — this is the primary physical verification target. Compare T1 against empirical formulas (e.g., T1 ≈ 0.1*N for RC frames where N is number of stories) as a sanity check.

7. **Add seismic demand analysis.** Apply lateral forces via response spectrum analysis or equivalent lateral force method per the applicable design code. Compute interstory drift ratios at each level. The critical check: drift must not exceed code limits (e.g., 1/550 for RC frames under frequent earthquakes per GB50011).

8. **Generate compliance verification output.** Print explicit pass/fail conclusions for: (a) fundamental period within expected range, (b) interstory drift within code limits at every story, (c) demand-to-capacity ratios below 1.0 for critical members. Use keywords like "pass", "satisfy", "compliant" for automated verification parsing.

9. **Validate executability before delivering.** Mentally trace the code for: unterminated strings (the #1 failure mode at 95.7% of syntax errors), undefined variables, incorrect API signatures, mismatched parentheses, and missing analysis steps. Verify that `analyze()` is called after all analysis objects are configured. Check that `wipe()` or `wipeAnalysis()` is called between gravity and lateral analyses if reusing the model.

10. **Present the complete code with inline engineering annotations.** Structure the output as a single executable Python script with clear section headers: `# --- Material Definitions ---`, `# --- Node Geometry ---`, `# --- Element Connectivity ---`, `# --- Boundary Conditions ---`, `# --- Gravity Analysis ---`, `# --- Modal Analysis ---`, `# --- Seismic Analysis ---`, `# --- Compliance Check ---`. Include the parameter summary at the top as comments.

## Concrete Examples

**Example 1: Generate a complete RC frame model from a specification**

User: "Create an OpenSeesPy model for a 5-story office building, 60m x 20m plan, 3.6m story height, seismic intensity 8 with PGA 0.20g."

Approach:
1. Parse parameters: function=office, stories=5, height=3.6m, plan=60x20m, intensity=8, PGA=0.20g
2. Select materials: C30 concrete (fc=20.1 MPa), HRB400 steel (fy=400 MPa)
3. Design sections: columns 600x600mm (lower 3 stories), 500x500mm (upper 2); beams 300x600mm
4. Generate a 2D frame with 4 bays @ 5m (representative of the 60m direction)

Output (abbreviated — full code is ~600 lines):
```python
import openseespy.opensees as ops
import math

ops.wipe()
ops.model('basic', '-ndm', 2, '-ndf', 3)

# --- Parameters ---
nStory = 5
storyH = 3.6  # m
bayW = 5.0    # m
nBay = 4
PGA = 0.20    # g

# --- Materials ---
ops.uniaxialMaterial('Concrete01', 1, -20.1e3, -0.002, -4.02e3, -0.005)  # Core
ops.uniaxialMaterial('Concrete01', 2, -20.1e3, -0.002, 0.0, -0.004)       # Cover
ops.uniaxialMaterial('Steel02', 3, 400.0e3, 2.0e8, 0.01)                  # Rebar

# --- Nodes ---
nodeTag = 1
for i in range(nStory + 1):
    for j in range(nBay + 1):
        ops.node(nodeTag, j * bayW, i * storyH)
        if i == 0:
            ops.fix(nodeTag, 1, 1, 1)  # Fixed base
        nodeTag += 1

# --- Sections (fiber) ---
# [Column and beam fiber section definitions...]

# --- Elements ---
# [forceBeamColumn elements for columns and beams...]

# --- Gravity Analysis ---
ops.timeSeries('Linear', 1)
ops.pattern('Plain', 1, 1)
# [Nodal loads from tributary gravity...]
ops.constraints('Plain')
ops.numberer('RCM')
ops.system('BandGeneral')
ops.test('NormDispIncr', 1.0e-6, 100)
ops.algorithm('Newton')
ops.integrator('LoadControl', 0.1)
ops.analysis('Static')
ops.analyze(10)
ops.loadConst('-time', 0.0)

# --- Modal Analysis ---
eigenValues = ops.eigen(3)
T1 = 2 * math.pi / math.sqrt(eigenValues[0])
print(f"Fundamental Period T1 = {T1:.4f} s")
# Sanity check: T1 ~ 0.1 * nStory = 0.5s expected range

# --- Seismic Demand & Compliance ---
# [Response spectrum or ELF analysis...]
# [Interstory drift check: must be < 1/550 = 0.00182]
# [Print: "Structure satisfies seismic drift requirements"]
```

**Example 2: Debug a failing OpenSeesPy structural model**

User: "My OpenSeesPy script crashes with `SyntaxError: unterminated string literal` somewhere around the material definitions. Can you fix it?"

Approach:
1. This is the #1 failure mode (95.7% of syntax errors). Scan all string arguments in `uniaxialMaterial`, `section`, and `element` calls for unclosed quotes.
2. Check for multiline strings that break across lines without proper continuation.
3. Apply the Bug-CoT pattern: identify the error manifestation, trace root cause, apply fix, add preventive annotation.

Output:
```
Root cause: Line 47 has ops.uniaxialMaterial('Concrete01', 1, -20.1e3, -0.002, -4.02e3, -0.005)
with a curly quote character (') instead of a straight quote ('). This is common when code is
copied from documentation PDFs.

Fix: Replace all curly/smart quotes with straight ASCII quotes.
Prevention: Always define material tags as integer variables at the top of the script and
reference them by variable name to avoid string-related errors entirely.
```

**Example 3: Verify physical consistency of an existing model**

User: "I generated this 3-story residential building model. The fundamental period comes out as T1=2.8s. Does that make sense?"

Approach:
1. Apply the empirical sanity check: T1 ~ 0.1 * N = 0.1 * 3 = 0.3s for a typical 3-story RC frame.
2. T1=2.8s is ~9x the expected value — the structure is far too flexible.
3. Check common causes: mass units inconsistent with force units, missing or incorrect section stiffness, gravity loads not applied before eigen analysis, wrong number of integration points.

Output:
```
T1 = 2.8s is physically implausible for a 3-story RC frame (expected ~0.2-0.5s).

Most likely causes (in order of probability):
1. Unit mismatch: If you defined E in Pa but lengths in mm, stiffness is off by 1e6.
   Fix: Ensure consistent units — recommend kN, m, s throughout.
2. Sections have near-zero stiffness: Check that fiber sections have nonzero area and
   correct material assignment. Print ops.eleResponse(elemTag, 'stiffness') to verify.
3. Mass not assigned: If ops.mass() was not called or gravity loads weren't applied
   before eigen(), the mass matrix may be empty/wrong.

Run eigenvalue analysis after gravity load step with ops.loadConst('-time', 0.0) to ensure
masses are correctly established.
```

## Best Practices

- **Do:** Always structure generated code in the three-tier hierarchy (Topology → Boundary/Load → Analysis/Solver). Missing any tier causes runtime failure. Verify completeness of each tier before generating the next.
- **Do:** Weight physical consistency far above syntactic correctness. A model that runs but gives T1=50s for a 5-story building is worse than a model that crashes with a clear error — the crash is diagnosable, the wrong answer is dangerous.
- **Do:** Include explicit sanity checks in generated code (empirical period formulas, drift limit comparisons, equilibrium checks). Print verification results with clear pass/fail keywords.
- **Do:** Use `ops.loadConst('-time', 0.0)` between gravity and lateral analyses and call `ops.wipeAnalysis()` before reconfiguring solvers. Omitting these causes silent state corruption.
- **Avoid:** Generating structural code without first confirming the unit system. Unit mismatches between material properties (Pa vs kPa vs MPa), geometry (m vs mm), and loads (N vs kN) are the second most common source of physically implausible results.
- **Avoid:** Using placeholder or approximate section properties. Every beam and column must have explicitly computed fiber sections with realistic reinforcement ratios (typically 0.8%-4% for columns, 0.2%-2.5% for beams per code limits).

## Error Handling

| Error Type | Frequency | Root Cause | Resolution |
|---|---|---|---|
| `SyntaxError: unterminated string literal` | 95.7% of syntax errors | Smart quotes from PDF copy, unclosed strings | Replace all quotes with ASCII; use integer tags |
| `WARNING: eigenvalue not found` | Common | Mass matrix empty or singular | Ensure mass is assigned and gravity analysis converges before eigen call |
| Implausible fundamental period | 5.2% of physics errors | Unit mismatch or missing section stiffness | Verify unit consistency; print section properties |
| Excessive interstory drift | 7.3% of compliance errors | Undersized members for seismic demand | Increase column dimensions or add shear walls |
| `analyze() returns -1` | Common | Non-convergent solution | Reduce load increment, switch to `NewtonLineSearch`, increase iteration limit |

When execution fails partway, check how far the script progressed (which tier completed) to narrow the root cause. Tier 1 failures indicate geometry/material errors; Tier 2 failures indicate load/boundary errors; Tier 3 failures indicate solver configuration issues.

## Limitations

- **OpenSeesPy-specific.** The workflow and API tier hierarchy are tailored to OpenSeesPy. Adapting to ETABS, SAP2000, ABAQUS, or ANSYS requires mapping the constraint taxonomy to those platforms' APIs.
- **2D frame models only.** The current approach generates 2D planar frame models. 3D models with floor diaphragms, torsional effects, and biaxial bending require significant extensions.
- **Chinese seismic codes as default.** The compliance checks reference GB50011-2010 and related Chinese standards. For ASCE 7, Eurocode 8, or other codes, drift limits, load combinations, and capacity design rules must be substituted.
- **Reinforced concrete frames only.** Steel structures, timber, masonry, or composite systems require different material models, section types, and design checks.
- **Period tolerance of 30%.** The MBEval benchmark considers T1 within ±30% of ground truth as passing — this is appropriate for preliminary design but insufficient for final engineering validation.
- **No nonlinear dynamic analysis.** The workflow covers modal and response spectrum analysis. Nonlinear time-history analysis with ground motion records requires additional complexity (ground motion selection, hysteretic material models, convergence management).

## Reference

**Paper:** Jiang et al., "Rethinking Scientific Modeling: Toward Physically Consistent and Simulation-Executable Programmatic Generation" (arXiv:2602.07083, 2026). Key takeaway: physics-constrained reinforcement learning with multi-granularity hybrid reward (70% execution-verified physical consistency, 25% AST completeness, 5% format) transforms a 7B parameter model into one that outperforms frontier models on structural code generation by 3-4x on strict compliance metrics.

**Code:** https://github.com/Jovanqing/AutoBM