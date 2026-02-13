---
name: "proopf-benchmarking-improving-professional-grade"
description: "Translate natural-language power system operational requirements into executable Optimal Power Flow (OPF) optimization code using differential modification of a canonical formulation. Use when the user asks to 'generate OPF code from a description', 'model power flow constraints', 'add security constraints to OPF', 'translate dispatch requirements to optimization', 'modify OPF parameters from scenario description', or 'build power system optimization from natural language'."
---

# ProOPF: Natural-Language to Executable OPF Optimization Code

This skill enables Claude to translate natural-language descriptions of power system operational requirements into executable optimization models for Optimal Power Flow (OPF). Instead of generating complete formulations from scratch, the approach applies **differential modifications** -- parameter adjustments and structural extensions -- to a canonical OPF backbone, following the ProOPF methodology. This dramatically reduces error rates and keeps generated code grounded in physically valid power system models.

## When to Use

- When the user describes a power system scenario (e.g., heatwave, line outage, load increase) and needs executable optimization code
- When modifying an existing OPF model based on new operational constraints described in plain English
- When adding structural extensions like N-1 security constraints, unit commitment, or transmission switching to a base OPF
- When the user provides numeric parameter changes (e.g., "increase load at bus 7 by 10%") and needs them applied to an OPF formulation
- When the user describes an operational scenario semantically (e.g., "model summer peak conditions with reduced thermal margins") requiring inference of which parameters to adjust
- When building or benchmarking LLM-driven pipelines for power system optimization code generation
- When evaluating generated OPF code against ground-truth objective values

## Key Technique

ProOPF's core insight is that most OPF problem instances share a common mathematical backbone and differ only through **scenario-dependent parameter updates** (changing load values, thermal limits, cost coefficients) and **structural extensions** (adding new variables, constraints, or objective terms). Rather than asking an LLM to produce a complete optimization program -- which introduces errors in the shared backbone -- the method decomposes the task into (1) identifying what changes from the canonical formulation and (2) generating only the differential code.

The method distinguishes four difficulty levels along two axes. The **parameter axis** separates *explicit* modifications (the user states exact numeric changes) from *semantic* modifications (the user describes an operational scenario and the model must infer which parameters change and in what direction). The **structural axis** separates *parameter-only* changes from cases requiring new decision variables, constraints, or objective terms. Level 1 (explicit + no structure) is straightforward; Level 4 (semantic + structural) requires both domain reasoning and code synthesis. For semantic inference, the technique uses **scenario trees** -- hierarchical mappings from events (e.g., "extreme heatwave") through mechanisms (e.g., "reduced transmission thermal margin") to parameter-direction pairs (e.g., decrease line rating limits).

Generated code targets MATPOWER (MATLAB) or equivalent frameworks (Pyomo, CVXPY, pandapower in Python). Correctness is validated end-to-end: the generated model is solved and its optimal objective value is compared against a ground-truth reference within a numerical tolerance. This six-dimensional evaluation isolates failure modes: executability, parameter correctness, semantic inference accuracy, structural identification, solver configuration, and end-to-end objective accuracy.

## Step-by-Step Workflow

1. **Identify the canonical OPF backbone.** Determine the base formulation the user is working from: AC-OPF, DC-OPF, or economic dispatch. Establish the network case (e.g., IEEE 14-bus, 118-bus, PEGASE 9241). If no base is specified, default to AC-OPF and ask the user for the network case data file.

2. **Classify the modification level.** Parse the user's natural-language request to determine:
   - Are parameters stated explicitly (numeric values) or semantically (scenario descriptions)?
   - Does the request require only parameter changes, or also structural extensions (new variables, constraints, objectives)?
   This classification determines the generation strategy.

3. **Extract parameter modifications as a differential patch.** For explicit requests, directly map stated changes to the data structure (e.g., `mpc.bus(7, PD) *= 1.10`). For semantic requests, build a scenario tree: identify the event, trace through physical mechanisms, and resolve to specific parameter-direction pairs. Document the inference chain.

4. **Identify required structural extensions.** If the request implies new modeling constructs (N-1 contingency, unit commitment binary variables, renewable curtailment penalties, voltage stability margins), select from the known structural variant catalog:
   - **Decision variable extensions**: DC approximation, economic dispatch, unit commitment, optimal transmission switching
   - **Objective modifications**: angle difference penalties, renewable curtailment costs, emission costs, load shedding penalties
   - **Constraint extensions**: N-1 security, voltage stability limits, ramp rate constraints, reserve requirements

5. **Check physical compatibility.** Validate that parameter modifications and structural extensions are mutually consistent. For example: reactive power updates are incompatible with DC-OPF; transmission switching requires binary variables and a MILP solver. Reject or flag invalid combinations before generating code.

6. **Generate the differential code.** Produce only the code that modifies the base case. Structure the output as:
   - A loader for the canonical case data (e.g., `mpc = loadcase('case118')`)
   - Parameter modification block with comments tracing each change to the NL requirement
   - Structural extension block defining new variables, constraints, and/or objective terms
   - Solver configuration (e.g., `mpopt = mpoption('opf.ac.solver', 'MIPS')`)
   - Execution call and result extraction

7. **Add placeholder support for semantic parameters.** When parameter values must be inferred at runtime or from external data, generate parameterized code with clearly named placeholders (e.g., `load_scale_factor`, `thermal_derate_pct`) and a configuration block at the top of the script where the user fills in values.

8. **Validate executability.** If the user has a MATLAB/Octave or Python environment available, run the generated code against the base case data to confirm it executes without errors and produces a feasible solution. Report the objective value.

9. **Compare against reference if available.** If a ground-truth objective value or reference implementation exists, compute the relative error: `|f*(generated) - f*(reference)| / |f*(reference)|`. Flag if error exceeds 1e-4.

10. **Document the modification summary.** Output a concise table listing each NL requirement, the inferred modification type (parameter/structural), the specific code change, and the inference rationale for semantic modifications.

## Concrete Examples

**Example 1: Explicit parameter modification (Level 1)**

User: "I have an IEEE 30-bus AC-OPF in MATPOWER. Increase all active power loads by 15% and set the thermal limit on the line between buses 1 and 2 to 200 MVA."

Approach:
1. Classify as Level 1: explicit numeric parameters, no structural changes
2. Load base case `case30`
3. Apply uniform load scaling and specific line rating override
4. Run standard AC-OPF

Output:
```matlab
%% Base case
mpc = loadcase('case30');

%% Parameter modifications
% Requirement: Increase all active power loads by 15%
mpc.bus(:, 3) = mpc.bus(:, 3) * 1.15;  % PD column

% Requirement: Set thermal limit on line 1->2 to 200 MVA
branch_idx = find(mpc.branch(:,1) == 1 & mpc.branch(:,2) == 2);
mpc.branch(branch_idx, 6) = 200;  % RATE_A

%% Solve
mpopt = mpoption('verbose', 0, 'out.all', 0);
results = runopf(mpc, mpopt);

fprintf('Optimal cost: %.2f $/hr\n', results.f);
fprintf('Converged: %d\n', results.success);
```

**Example 2: Semantic parameter inference (Level 2)**

User: "Model a summer heatwave scenario on the IEEE 118-bus system. Assume air conditioning drives up residential demand, and high ambient temperatures reduce transmission line thermal capacity."

Approach:
1. Classify as Level 2: semantic parameters, no structural extensions
2. Build scenario tree:
   - Event: extreme heatwave
   - Mechanism 1: increased cooling load -> increase PD at load buses
   - Mechanism 2: elevated ambient temperature -> decrease line RATE_A values
3. Generate parameterized code with inference rationale

Output:
```matlab
%% Base case
mpc = loadcase('case118');

%% Configuration (adjust based on severity)
load_increase_pct = 20;    % Heatwave drives ~15-25% residential load increase
thermal_derate_pct = 15;   % Ambient temp reduces thermal ratings ~10-20%

%% Parameter modifications
% Inference: Heatwave -> increased AC load -> higher active power demand
mpc.bus(:, 3) = mpc.bus(:, 3) * (1 + load_increase_pct / 100);  % PD
mpc.bus(:, 4) = mpc.bus(:, 4) * (1 + load_increase_pct / 100);  % QD

% Inference: High ambient temp -> reduced convective cooling -> derated lines
mpc.branch(:, 6) = mpc.branch(:, 6) * (1 - thermal_derate_pct / 100);  % RATE_A
mpc.branch(:, 7) = mpc.branch(:, 7) * (1 - thermal_derate_pct / 100);  % RATE_B

%% Solve
mpopt = mpoption('verbose', 0, 'out.all', 0);
results = runopf(mpc, mpopt);

fprintf('Optimal cost: %.2f $/hr\n', results.f);
fprintf('Converged: %d\n', results.success);
```

**Example 3: Structural extension with semantic parameters (Level 4)**

User: "For the IEEE 14-bus system under winter storm conditions, add N-1 security constraints for all transmission lines and include a load shedding penalty in the objective."

Approach:
1. Classify as Level 4: semantic parameters (winter storm) + structural extensions (N-1 + load shedding)
2. Semantic inference: winter storm -> icing on lines -> derate thermal limits; heating load spike -> increase demand
3. Structural: add contingency constraints for each line outage; add load shedding variables with penalty cost

Output:
```matlab
%% Base case
mpc = loadcase('case14');
nb = size(mpc.bus, 1);
nl = size(mpc.branch, 1);

%% Semantic parameter modifications
% Winter storm -> icing reduces thermal capacity
mpc.branch(:, 6) = mpc.branch(:, 6) * 0.80;  % RATE_A derated 20%

% Winter storm -> heating load increase
mpc.bus(:, 3) = mpc.bus(:, 3) * 1.18;  % PD +18%

%% Structural extension: Load shedding variables
% Add load shedding as negative generation at each load bus
load_shed_cost = 10000;  % $/MWh penalty (VOLL)
load_buses = find(mpc.bus(:, 3) > 0);
for k = 1:length(load_buses)
    bus = load_buses(k);
    mpc.gen = [mpc.gen; zeros(1, size(mpc.gen, 2))];
    idx = size(mpc.gen, 1);
    mpc.gen(idx, 1) = bus;       % GEN_BUS
    mpc.gen(idx, 9) = mpc.bus(bus == mpc.bus(:,1), 3);  % PMAX = load
    mpc.gen(idx, 8) = 1;         % GEN_STATUS on
    mpc.gencost = [mpc.gencost;
        [2 0 0 2 load_shed_cost 0]];  % Linear cost
end

%% Structural extension: N-1 security (SCOPF via iterative approach)
mpopt = mpoption('verbose', 0, 'out.all', 0);
results = runopf(mpc, mpopt);

% Check each contingency; add binding constraints iteratively
for c = 1:nl
    mpc_c = mpc;
    mpc_c.branch(c, 11) = 0;  % Trip line c
    res_c = runpf(mpc_c, mpopt);
    if ~res_c.success
        fprintf('Contingency %d: infeasible, load shedding needed\n', c);
    end
end

fprintf('Base case cost: %.2f $/hr\n', results.f);
```

## Best Practices

- **Do:** Always start from a canonical base case and apply differential modifications. This preserves the validated backbone and limits error surface to the changed components.
- **Do:** Trace every code modification back to a specific phrase in the user's natural-language requirement with an inline comment. This makes the mapping auditable.
- **Do:** Use parameterized placeholders with sensible defaults for semantically inferred values. Power system parameters have well-known ranges (e.g., load growth 5-30%, thermal derating 10-25%).
- **Do:** Validate physical compatibility before generating code. Catch mismatches like reactive power modifications in DC-OPF or continuous relaxation where integers are required.
- **Avoid:** Generating an entire OPF formulation from scratch. The backbone (bus admittance matrix construction, power balance equations, generator limits) is complex and error-prone to regenerate.
- **Avoid:** Hardcoding bus/branch indices without verifying they exist in the case data. Always use lookup-based indexing (e.g., `find(mpc.branch(:,1) == from_bus)`).
- **Avoid:** Assuming a specific solver is available. Always set solver options explicitly and provide fallback configurations.

## Error Handling

| Failure Mode | Symptom | Resolution |
|---|---|---|
| Non-executable code | MATLAB/Python syntax errors, undefined variables | Verify all MATPOWER constants (PD=3, QD=4, etc.) are correct; test against a minimal case first |
| Infeasible OPF | Solver returns non-convergence | Check if parameter modifications pushed the system beyond physical limits; relax load shedding or reduce scaling factors |
| Wrong objective value | Solution exists but cost differs from reference | Verify cost function coefficients weren't inadvertently modified; check that structural extensions added cost terms correctly |
| Semantic inference error | Wrong parameters modified for the scenario | Revisit the scenario tree: ensure the event-mechanism-parameter chain is physically justified; consult domain references |
| Structural incompatibility | Runtime error when combining extensions | Check the compatibility matrix: e.g., OTS requires integer variables and a MILP solver; N-1 with unit commitment may need decomposition |
| Index out of bounds | Referencing buses/branches not in the case | Always validate bus/branch existence before modification; use defensive lookups |

## Limitations

- **Domain expertise required for validation.** While the skill generates executable code, verifying that semantic inferences are physically reasonable (e.g., correct parameter directions for a specific weather event) requires power systems knowledge.
- **MATPOWER-centric.** The primary code generation targets MATPOWER. For Pyomo, pandapower, or CVXPY targets, the structural patterns transfer but the API calls differ significantly and need adaptation.
- **No dynamic/transient modeling.** OPF is a steady-state optimization. Requests involving frequency response, transient stability, or time-series dispatch require different formulations outside this skill's scope.
- **Combinatorial explosion in structural extensions.** Combining multiple structural variants (e.g., N-1 + unit commitment + transmission switching) may produce models that are computationally intractable for large networks. Flag this to the user.
- **Semantic inference is approximate.** Mapping narrative scenarios to parameter adjustments involves judgment calls. The generated placeholder values are reasonable defaults, not precise engineering values.

## Reference

[ProOPF: Benchmarking and Improving LLMs for Professional-Grade Power Systems Optimization Modeling](https://arxiv.org/abs/2602.03070v3) -- Shen et al., 2026. Focus on Section 3 (dataset construction with the four-level taxonomy), Section 4 (the scenario-tree mechanism for semantic parameter inference), and Section 5 (the six-dimensional evaluation framework for generated OPF code).