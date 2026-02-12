---
name: "rethinking-scientific-modeling-physically"
description: "Generate physics-consistent, simulation-executable structural engineering code using domain-constrained LLM workflows. Applies the AutoBM framework: domain knowledge injection, constraint-oriented alignment, and closed-loop verification. Trigger phrases: 'generate OpenSees model', 'create structural simulation code', 'physics-consistent building model', 'structural engineering code generation', 'finite element model from description', 'simulation-ready structural code'."
---

# Physics-Consistent Structural Modeling Code Generation

This skill enables Claude to generate simulation-executable structural engineering code (primarily OpenSeesPy) from natural language building descriptions. It applies the AutoBM framework's three-pillar approach: (1) injecting formalized domain knowledge and engineering design codes as constraints, (2) aligning generated code to physical laws and API specifications through structured reasoning, and (3) validating outputs through closed-loop execution and physical metric extraction. The result is code that not only runs without errors but produces structurally meaningful results -- periods, drift ratios, and compliance checks that match real engineering expectations.

## When to Use

- When the user asks to generate a finite element model for a reinforced concrete frame, steel structure, or other building type
- When the user needs OpenSeesPy (or similar FEA) code that must actually execute and produce valid simulation results
- When translating a structural engineering specification (story count, heights, seismic zone, building function) into simulation code
- When debugging structural modeling code that runs but produces physically implausible results (e.g., fundamental period wildly off)
- When the user wants automated seismic design verification -- checking interstory drift limits, demand-to-capacity ratios, or code compliance
- When generating parameterized structural models from a description like "6-story residential RC frame in seismic intensity zone VII"

## Key Technique

**The Grammatical-Physical Divide.** Standard LLMs can generate syntactically valid structural modeling code, but the code frequently violates physical constraints -- producing buildings with impossible natural periods, missing gravity loads, or analysis pipelines that skip critical steps. The AutoBM paper measured this: Claude Sonnet 4.5 achieves 79% executability but only 19% strict physical correctness. The gap between "code that runs" and "code that means something" is the core problem.

**Three-Pillar Constraint Framework.** The solution decomposes constraints into three categories that must all be satisfied: (1) *Physical constraints (DP)* -- structural mechanics principles, seismic design codes (e.g., GB50011-2010), capacity design rules, gravity load limits, interstory drift bounds; (2) *API constraints (DA)* -- correct OpenSeesPy function signatures, required modeling workflow sequences (nodes before elements, materials before sections), analysis pipeline ordering; (3) *Specification constraints (DT)* -- building geometry matching the user's request, correct seismic intensity parameters, appropriate building function classification. Code must satisfy all three simultaneously.

**Closed-Loop Verification.** Instead of treating code generation as a one-shot task, the framework validates outputs through execution in a sandbox, extracting physical metrics (fundamental period T1), checking compliance keywords, and comparing against ground truth. Failures feed back into iterative repair. This verification uses three tiers: Tier 1 (topology -- nodes/elements defined), Tier 2 (boundary/loads -- constraints and forces applied), Tier 3 (analysis/solver -- eigenvalue analysis and response spectrum configured). All three tiers must be present for a logically complete model.

## Step-by-Step Workflow

1. **Parse the structural specification from the user's request.** Extract: building function (residential, office, hospital), number of stories, story heights, span dimensions, seismic design intensity, soil class, and any material specifications (concrete grade, rebar grade). If details are missing, use standard engineering defaults and state them explicitly.

2. **Inject domain knowledge as constraint preamble.** Before generating code, assemble the applicable design code constraints: seismic intensity maps to design basic acceleration, building function maps to importance factor, structure type determines behavior factor. Format these as structured parameters, not prose. Example: `seismic_group=1, intensity=VII, site_class=II, Tg=0.35s, alpha_max=0.08`.

3. **Generate the node and element topology (Tier 1).** Define all structural nodes with coordinates matching the specified geometry. Create frame elements (beamColumn) connecting nodes. Verify: node count = (stories + 1) * bays_per_direction, element connectivity forms a valid frame without orphan nodes.

4. **Define materials, sections, and boundary conditions (Tier 2).** Specify concrete and steel material models using `uniaxialMaterial` commands. Define fiber sections for beams and columns with appropriate reinforcement ratios. Apply fixity at base nodes. Add gravity loads: dead load + live load per story, applied as nodal loads or distributed element loads. Verify: strong-column-weak-beam capacity ratios are satisfied.

5. **Configure the analysis pipeline (Tier 3).** Implement in this exact order: (a) gravity analysis with load-controlled static integrator, (b) modal/eigenvalue analysis to extract fundamental periods, (c) response spectrum analysis or equivalent lateral force procedure. Each step must include convergence checks and `analyze()` calls.

6. **Add physical verification checks in the generated code.** After modal analysis, compute and print the fundamental period T1. After response spectrum analysis, compute interstory drift ratios for each story and compare against code limits (typically 1/550 for RC frames). Print explicit compliance/non-compliance statements.

7. **Validate API call ordering and signatures.** Audit the generated code for correct OpenSeesPy workflow: `ops.wipe()` -> `ops.model()` -> materials -> nodes -> elements -> boundary -> loads -> analysis. Check that every function call uses valid argument types and counts per the OpenSeesPy documentation.

8. **Execute the code in a sandboxed environment.** Run the generated script with a timeout (60s recommended). Capture stdout/stderr. If execution fails, classify the error: syntax error (fix string literals, parentheses), undefined variable (trace data flow), API misuse (check function signatures), or convergence failure (adjust solver parameters).

9. **Validate physical plausibility of results.** Check extracted period T1 against empirical formulas: for RC frames, T1 ~ 0.1 * N_stories (rough check). If the period deviates by more than 30% from expected range, investigate: likely causes are missing mass, incorrect section properties, or wrong units. Iterate on the code until physical metrics are plausible.

10. **Deliver the final code with an engineering summary.** Output the complete, verified script along with: computed fundamental period, interstory drift ratios per story, compliance status against the applicable design code, and any assumptions made. Flag any constraints that are close to their limits.

## Concrete Examples

**Example 1: RC Frame from Natural Language Description**

User: "Generate an OpenSeesPy model for a 5-story reinforced concrete office building. Each story is 3.6m high, 3 bays at 6m span in each direction. Seismic zone VII, site class II. Concrete C30, rebar HRB400."

Approach:
1. Parse: 5 stories, h=3.6m, 3x3 bays, span=6m, intensity VII, site II
2. Derive constraints: alpha_max=0.08, Tg=0.35s, characteristic period group 1
3. Compute: 24 nodes per floor (4x4 grid), 120 total nodes, column sections 500x500mm (initial estimate), beam sections 300x600mm

Output (abbreviated):
```python
import openseespy.opensees as ops

ops.wipe()
ops.model('basic', '-ndm', 3, '-ndf', 6)

# --- Material Definitions ---
fc = 20.1e6  # C30 design compressive strength (Pa)
fy = 360e6   # HRB400 yield strength (Pa)
ops.uniaxialMaterial('Concrete01', 1, -fc, -0.002, -0.85*fc, -0.005)
ops.uniaxialMaterial('Steel01', 2, fy, 2.0e11, 0.01)

# --- Node Generation ---
story_heights = [0.0, 3.6, 7.2, 10.8, 14.4, 18.0]
bay_coords_x = [0.0, 6.0, 12.0, 18.0]
bay_coords_y = [0.0, 6.0, 12.0, 18.0]
node_tag = 1
for sz in story_heights:
    for bx in bay_coords_x:
        for by in bay_coords_y:
            ops.node(node_tag, bx, by, sz)
            if sz == 0.0:
                ops.fix(node_tag, 1, 1, 1, 1, 1, 1)  # Fixed base
            node_tag += 1

# --- Fiber Sections (columns: 500x500, beams: 300x600) ---
# [section definitions with cover concrete, core concrete, rebar layers]

# --- Elements (forceBeamColumn with 5 integration points) ---
# [column and beam element definitions connecting nodes]

# --- Gravity Loads ---
ops.timeSeries('Linear', 1)
ops.pattern('Plain', 1, 1)
# Dead: 5.0 kN/m2, Live: 2.0 kN/m2 (office)
# Applied as nodal loads at each floor node

# --- Gravity Analysis ---
ops.constraints('Transformation')
ops.numberer('RCM')
ops.system('BandGeneral')
ops.test('NormDispIncr', 1.0e-6, 100)
ops.algorithm('Newton')
ops.integrator('LoadControl', 0.1)
ops.analysis('Static')
ops.analyze(10)
ops.loadConst('-time', 0.0)

# --- Modal Analysis ---
num_modes = 3
eigenvalues = ops.eigen(num_modes)
periods = [2 * 3.14159 / (ev**0.5) for ev in eigenvalues]
print(f"Fundamental period T1 = {periods[0]:.3f} s")
# Expected range for 5-story RC frame: ~0.4-0.7s

# --- Interstory Drift Check ---
# [extract node displacements, compute drift ratios per story]
# Limit: 1/550 = 0.00182 for RC frame per GB50011-2010
```

**Example 2: Debugging Physically Implausible Output**

User: "My OpenSeesPy model runs fine but gives a fundamental period of 15.2 seconds for a 3-story building. That can't be right."

Approach:
1. A 3-story RC frame should have T1 ~ 0.3-0.5s. A period of 15.2s indicates the model is ~30-50x too flexible.
2. Check common causes using the three-constraint taxonomy:
   - **Physical (DP)**: Are masses assigned? Missing mass produces near-zero eigenvalues -> huge periods. Are gravity loads applied before modal analysis?
   - **API (DA)**: Is `ops.loadConst('-time', 0.0)` called after gravity analysis? Without it, the eigenvalue solver includes the load pattern as stiffness modification. Is the mass defined via `ops.mass()` or via element density?
   - **Specification (DT)**: Are units consistent? Mixing meters/millimeters or N/kN is the #1 cause of wrong-magnitude results.

Output:
```
Diagnosis: Period of 15.2s for 3-story building indicates one of:

1. MISSING NODAL MASS (most likely)
   - Check: search for ops.mass() calls. If absent, eigenvalue analysis
     has no mass matrix and returns garbage.
   - Fix: Add ops.mass(nodeTag, mx, my, mz, 0, 0, 0) at each floor node
     where mx = tributary_area * (dead_load + 0.5*live_load) / g

2. UNIT INCONSISTENCY
   - If you defined E_concrete = 30e9 (Pa) but lengths in mm,
     stiffness is 1e6x too low -> period is 1000x too high.
   - Fix: Use consistent unit system. Recommended: N, m, kg, s.

3. MISSING loadConst AFTER GRAVITY
   - If gravity loads remain as pseudo-time, modal analysis
     is corrupted.
   - Fix: Add ops.loadConst('-time', 0.0) after gravity analyze().
```

**Example 3: Parameterized Model Generation**

User: "I need to generate 20 variations of a 4-story RC frame model, varying column sizes from 400mm to 700mm in 50mm increments, and extract the fundamental period for each."

Approach:
1. Create a parameterized function that takes column dimension as input
2. Wrap in a loop with proper `ops.wipe()` between runs
3. Include all three constraint tiers in each iteration
4. Collect and report results as a table

Output:
```python
import openseespy.opensees as ops
import numpy as np

results = []
col_sizes = np.arange(400, 750, 50)  # mm -> convert to m

for col_dim_mm in col_sizes:
    col_dim = col_dim_mm / 1000.0  # Convert to meters
    ops.wipe()
    ops.model('basic', '-ndm', 2, '-ndf', 3)

    # [Full model generation with col_dim as parameter]
    # Nodes, materials, sections (fiber with col_dim x col_dim),
    # elements, loads, gravity analysis, modal analysis

    eigenvalues = ops.eigen(1)
    T1 = 2 * 3.14159 / (eigenvalues[0]**0.5)
    results.append((col_dim_mm, T1))
    print(f"Column {col_dim_mm}mm: T1 = {T1:.4f} s")

# Verify physical trend: larger columns -> shorter period (stiffer)
for i in range(1, len(results)):
    assert results[i][1] <= results[i-1][1], \
        f"Non-monotonic period at {results[i][0]}mm -- check model"
```

## Best Practices

- **Do:** Always define units explicitly at the top of every generated script (e.g., `# Units: N, m, kg, s`). Unit confusion causes the majority of physically implausible results.
- **Do:** Include the full analysis pipeline -- gravity, modal, lateral -- in every model. Skipping gravity analysis before modal analysis produces incorrect periods because the stiffness matrix lacks P-delta effects.
- **Do:** Add self-checking assertions in generated code: verify node counts match expected geometry, check that periods fall in physically reasonable ranges, confirm drift ratios are below code limits.
- **Do:** Follow the strict OpenSeesPy call ordering: wipe -> model -> materials -> nodes -> fixities -> elements -> timeSeries -> pattern -> loads -> analysis. Out-of-order calls produce silent failures.
- **Avoid:** Generating partial models that require manual completion. Every output must be a complete, runnable script from `ops.wipe()` through result extraction.
- **Avoid:** Using placeholder values for critical structural parameters (section dimensions, material strengths, seismic coefficients). Either derive them from the specification or state the engineering assumption and source code provision.
- **Avoid:** Treating code executability as sufficient validation. A model that runs but predicts T1=50s for a 3-story building is worse than a model that crashes -- it gives false confidence.

## Error Handling

| Error Type | Symptom | Resolution |
|---|---|---|
| **Syntax (unterminated string)** | `SyntaxError` on execution | Check all string literals, especially multiline comments. This accounts for 95% of executability failures. |
| **Undefined variable** | `NameError` at runtime | Trace data flow: ensure node tags, material tags, and section tags are defined before reference. |
| **Convergence failure** | `analyze()` returns negative | Reduce load step size, switch algorithm (`Newton` -> `NewtonLineSearch`), or increase iteration limit. |
| **Singular stiffness matrix** | `WARNING: singular` | Check for unconnected nodes, missing boundary conditions, or zero-stiffness elements. |
| **Implausible period** | T1 off by order of magnitude | Check unit consistency, verify mass is assigned, confirm `loadConst` is called after gravity. |
| **Missing compliance output** | No drift/capacity checks | Add post-analysis verification block: extract displacements, compute drift ratios, compare to code limits, print pass/fail. |

## Limitations

- **Domain specificity.** This workflow is validated for reinforced concrete and steel frame structures under seismic loading. It does not cover specialized structures (bridges, dams, offshore platforms) without significant adaptation of the constraint set.
- **Design code coverage.** The constraint knowledge is primarily based on Chinese design codes (GB50011, GB50010, JGJ3). For Eurocode, ASCE 7, or other standards, the seismic parameters, drift limits, and capacity design rules must be substituted accordingly.
- **2D vs 3D limitations.** Many examples use 2D frame simplifications. Full 3D models with floor diaphragms, torsional irregularity checks, and bidirectional seismic input require additional constraint layers not fully covered here.
- **Nonlinear analysis.** The verification metrics focus on linear modal and response-spectrum analysis. Pushover analysis, time-history analysis, and material nonlinearity introduce additional failure modes that need extended validation.
- **Model scale.** For buildings beyond ~20 stories or with complex irregular geometries, the generated code may need manual optimization for computational efficiency (e.g., substructuring, reduced-order models).

## Reference

**Paper:** Jiang et al., "Rethinking Scientific Modeling: Toward Physically Consistent and Simulation-Executable Programmatic Generation" (arXiv:2602.07083, 2026). Look for: the three-constraint taxonomy (DP/DA/DT), the tiered API completeness check, the multi-granularity reward structure, and Table 1 showing the grammatical-physical divide across LLMs. **Code:** [github.com/Jovanqing/AutoBM](https://github.com/Jovanqing/AutoBM)