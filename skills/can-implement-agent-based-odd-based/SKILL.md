---
name: "can-implement-agent-based-odd-based"
description: "Translate ODD protocol specifications into validated, executable agent-based model (ABM) code in Python. Use when the user says 'implement this ABM', 'convert ODD to code', 'build an agent-based model from this specification', 'replicate this NetLogo model in Python', 'translate this model description into a simulation', or 'create a predator-prey / ecological / social ABM'."
---

# ODD-to-Code: Implementing Agent-Based Models from Standardized Specifications

This skill enables Claude to translate agent-based model (ABM) descriptions written in the ODD (Overview, Design concepts, Details) protocol into correct, validated Python implementations. Based on a rigorous replication study (Fachada et al., 2026) that tested 17 LLMs on ODD-to-code translation, this skill encodes the patterns that produce statistically faithful implementations and avoids the failure modes that cause behaviorally incorrect simulations -- even when the code runs without errors.

## When to Use

- When a user provides an ODD protocol description and asks for a working Python simulation
- When replicating an existing ABM (e.g., from NetLogo, Mesa, or Repast) in Python from its ODD documentation
- When building a new agent-based model and the user describes entities, scheduling, and state variables in structured prose
- When the user wants to verify that a generated ABM produces statistically equivalent output to a reference implementation
- When converting ecological, social, or economic simulation specifications into executable code
- When the user asks to "implement PPHPC" or any named ABM with known ODD documentation

## Key Technique

The ODD protocol (Grimm et al., 2006, 2010) is a standardized framework for describing agent-based models. It has three blocks: **Overview** (purpose, entities, state variables, process overview, scheduling), **Design concepts** (emergence, adaptation, sensing, interaction, stochasticity, observation), and **Details** (initialization, input data, submodels). The critical insight from the paper is that **executability is insufficient for scientific validity** -- code that runs and produces output can still be behaviorally wrong in subtle ways that only surface under statistical comparison.

The validated approach uses a **staged evaluation pipeline** with six levels: (1) code presence, (2) syntax correctness, (3) runtime success, (4) output format compliance, (5) statistical comparison against a reference baseline, and (6) distributional equivalence under multiple parameter regimes. Stages 1-4 are necessary but not sufficient; only implementations passing stage 6 are scientifically usable. The statistical comparison uses PCA-based dimensionality reduction on time-series outputs followed by Energy tests (nonparametric multivariate distributional comparison), with Benjamini-Hochberg correction for multiple testing.

The paper identifies specific implementation patterns that separate success from failure: correct asynchronous agent scheduling (random order within each tick), proper energy accounting across all operations, countdown-based state management for environmental processes, toroidal boundary handling, and post-iteration output collection. Models that synchronize agent actions, misapply reproduction probabilities, or skip environmental regeneration cycles produce code that *appears* correct but diverges statistically from the reference under load.

## Step-by-Step Workflow

1. **Parse the ODD specification into structured sections.** Extract: purpose, entities and their state variables, spatial structure (grid dimensions, topology), process overview with explicit scheduling order, design concepts (especially stochasticity and interaction rules), initialization parameters, and submodel equations. If the user provides prose rather than formal ODD, restructure it into ODD sections before proceeding.

2. **Define the function signature with explicit parameters.** Create a single entry-point function (e.g., `run_model()`) that accepts all model parameters as typed arguments. Use only standard scientific Python (numpy, pandas). List every parameter from the ODD's initialization and submodel sections -- do not hardcode values.

3. **Implement entity classes or data structures for each agent type and environmental cell.** Each entity needs: all state variables from the ODD, methods for each submodel action (move, eat, reproduce, die), and energy tracking. Use numpy arrays for grid state if performance matters; use simple classes if clarity matters.

4. **Implement the scheduling loop in exact ODD order.** This is the most failure-prone step. Follow the process overview *literally*: if the ODD says "agents act in random order," shuffle the agent list each tick. If it says "movement, then feeding, then reproduction," execute those phases sequentially within each tick. Do not batch or parallelize phases unless the ODD explicitly allows it.

5. **Implement each submodel as a separate function matching the ODD's Details section.** For movement: respect the specified neighborhood (von Neumann vs. Moore) and topology (toroidal wrapping). For feeding: apply energy gains/losses exactly. For reproduction: check energy thresholds *before* applying probability, then split energy between parent and offspring. For death: trigger when energy reaches zero, removing the agent from the active list.

6. **Implement environmental dynamics as cell-level state machines.** Food regeneration, resource depletion, or other grid processes need countdown timers or state flags per cell. Process these at the correct point in the scheduling loop, not as agent actions.

7. **Collect output metrics at the correct phase of each iteration.** Record population counts, mean energy values, and environmental state *after* all agent actions for that tick are complete. Return results as a pandas DataFrame with one row per iteration and named columns matching the ODD's observation section.

8. **Run a smoke test with minimal parameters (3-5 iterations) to verify executability and output format.** Check: function returns a DataFrame, column names match specification, no runtime errors, values are in plausible ranges.

9. **Run full validation: execute 30 independent replications under at least two parameter regimes.** Compare output distributions against reference data using PCA + Energy test (or Mann-Whitney U per metric if reference distributions are available). Statistical equivalence at alpha=0.01 with Benjamini-Hochberg correction is the target.

10. **Measure code quality: compute SLOC, cyclomatic complexity, and lint/type-check density.** Target: maintainability index above 50, type error density below 5 per 100 SLOC, minimal ruff/pylint warnings. Refactor only if quality metrics are poor -- do not over-engineer a passing implementation.

## Concrete Examples

**Example 1: PPHPC Predator-Prey Model**

User: "Implement the PPHPC predator-prey model. Grid is 100x100 toroidal. Prey eat food on cells, predators eat prey. Both reproduce probabilistically above energy thresholds. Food regenerates after a countdown."

Approach:
1. Parse entities: Prey (energy, position), Predator (energy, position), Cell (food present, restart counter)
2. Define `run_pphpc(xdim, ydim, n_prey, n_pred, n_iters, prey_gain, pred_gain, prey_loss, pred_loss, prey_repro_thresh, pred_repro_thresh, prey_repro_prob, pred_repro_prob, food_restart)` returning a DataFrame
3. Initialize grid with food on all cells, scatter agents randomly
4. Each iteration in order: (a) shuffle all agents, (b) each agent moves to random von Neumann neighbor losing energy, (c) regenerate food on cells whose countdown expired, (d) shuffle agents again, (e) each prey on a food cell eats (gains energy, cell starts countdown), (f) each predator on a cell with prey eats one (gains energy, prey dies), (g) each agent with energy above threshold reproduces with given probability (energy split), (h) remove dead agents, (i) collect metrics

Output:
```python
import numpy as np
import pandas as pd

def run_pphpc(xdim=100, ydim=100, n_prey=400, n_pred=200, n_iters=4000,
              prey_gain=4, pred_gain=20, prey_loss=1, pred_loss=1,
              prey_repro_thresh=4, pred_repro_thresh=20,
              prey_repro_prob=0.25, pred_repro_prob=0.05,
              food_restart=10):
    # Grid: food_present (bool), restart_counter (int)
    food = np.ones((xdim, ydim), dtype=bool)
    counters = np.zeros((xdim, ydim), dtype=int)

    # Agent lists: [x, y, energy]
    prey = [[np.random.randint(xdim), np.random.randint(ydim),
             np.random.randint(1, 2 * prey_gain)] for _ in range(n_prey)]
    predators = [[np.random.randint(xdim), np.random.randint(ydim),
                  np.random.randint(1, 2 * pred_gain)] for _ in range(n_pred)]

    records = []
    offsets = [(-1, 0), (1, 0), (0, -1), (0, 1)]  # von Neumann

    for t in range(n_iters):
        # Movement (all agents, random order)
        all_agents = [(a, 'prey') for a in prey] + [(a, 'pred') for a in predators]
        np.random.shuffle(all_agents)  # ... (movement logic with toroidal wrap)

        # Food regeneration
        regen_mask = (~food) & (counters <= 0)
        # ... (countdown decrement, food restore)

        # Prey feeding, predator hunting, reproduction, death removal
        # ... (full submodel implementations)

        # Collect output
        records.append({
            'total_prey': len(prey), 'total_predators': len(predators),
            'total_food': int(food.sum()),
            'mean_energy_prey': np.mean([a[2] for a in prey]) if prey else 0,
            'mean_energy_predators': np.mean([a[2] for a in predators]) if predators else 0,
            'mean_c': counters[~food].mean() if (~food).any() else 0,
        })

    return pd.DataFrame(records)
```

**Example 2: Schelling Segregation Model from ODD Description**

User: "Here is the ODD for a Schelling segregation model. Grid 50x50, two agent types (40% each, 20% empty). Agents are unhappy if fewer than 30% of neighbors are same type. Unhappy agents move to random empty cell. Run until stable or 500 ticks."

Approach:
1. Extract ODD sections: entities (agents with type, position, happiness), grid (50x50, non-toroidal or toroidal per spec), scheduling (each tick: compute happiness for all, then unhappy agents move in random order)
2. Define `run_schelling(dim=50, frac_a=0.4, frac_b=0.4, tolerance=0.3, max_iters=500)`
3. Key scheduling detail: compute happiness status for ALL agents before any moves (synchronous evaluation, asynchronous movement) -- or as specified by the ODD
4. Collect: segregation index, fraction happy, count of moves per tick

Output:
```python
def run_schelling(dim=50, frac_a=0.4, frac_b=0.4, tolerance=0.3, max_iters=500):
    grid = np.zeros((dim, dim), dtype=int)  # 0=empty, 1=type_a, 2=type_b
    # Initialize populations...
    records = []
    for t in range(max_iters):
        happiness = compute_happiness(grid, tolerance)  # all at once
        unhappy = list(zip(*np.where(~happiness & (grid > 0))))
        np.random.shuffle(unhappy)
        empty = list(zip(*np.where(grid == 0)))
        moves = 0
        for (r, c) in unhappy:
            if empty:
                nr, nc = empty.pop(np.random.randint(len(empty)))
                grid[nr, nc] = grid[r, c]
                grid[r, c] = 0
                empty.append((r, c))
                moves += 1
        records.append({'tick': t, 'frac_happy': happiness[grid > 0].mean(),
                        'moves': moves})
        if moves == 0:
            break
    return pd.DataFrame(records)
```

**Example 3: Validating a Generated ABM Against Reference Data**

User: "I have 30 CSV files from a NetLogo reference run and 30 from my Python implementation. Are they statistically equivalent?"

Approach:
1. Load all runs, extract the same time-series columns from each
2. Standardize each column (zero mean, unit variance) across all runs combined
3. Concatenate standardized columns into feature vectors per run
4. Apply PCA, retain components explaining >= 80% of variance
5. Run Energy test (scipy or custom) on PCA-projected reference vs. generated data
6. Apply Benjamini-Hochberg correction across all tested metrics
7. Report: equivalent if no corrected p-value falls below 0.01

Output:
```python
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler
from scipy.spatial.distance import cdist

def validate_abm(ref_runs, gen_runs, alpha=0.01):
    """ref_runs, gen_runs: lists of DataFrames with identical columns."""
    # Stack and standardize
    all_data = pd.concat(ref_runs + gen_runs)
    scaler = StandardScaler().fit(all_data)
    ref_scaled = [scaler.transform(r) for r in ref_runs]
    gen_scaled = [scaler.transform(g) for g in gen_runs]
    # PCA
    pca = PCA(n_components=0.8).fit(np.vstack(ref_scaled))
    ref_pca = [pca.transform(r) for r in ref_scaled]
    gen_pca = [pca.transform(g) for g in gen_scaled]
    # Energy test per time step or aggregated
    # ... (compute energy statistic and permutation p-value)
    # Benjamini-Hochberg correction
    # ... (adjust p-values, compare to alpha)
    return results_df  # columns: metric, p_value, adjusted_p, equivalent
```

## Best Practices

- **Do:** Follow the ODD scheduling order exactly. The most common failure mode is reordering phases (e.g., moving agents before regenerating food) or processing agents synchronously when the spec requires random-order asynchronous updates.
- **Do:** Apply reproduction probability only after checking the energy threshold -- these are sequential conditions, not independent. Split energy between parent and offspring at the moment of reproduction.
- **Do:** Use toroidal wrapping (modulo arithmetic) for grid boundaries when the ODD specifies a torus. Off-by-one boundary errors cause subtle behavioral drift.
- **Do:** Validate under at least two parameter regimes. A model that passes under low-density conditions may fail under high-density stress due to interaction-order bugs.
- **Avoid:** Assuming that running code is correct code. The paper's central finding is that executability (stages 1-4) does not imply behavioral validity (stages 5-6). Always statistically validate.
- **Avoid:** Using global random state without independent seeding per replication. Each of the 30 validation runs must be stochastically independent.

## Error Handling

- **Runtime timeout:** ABMs with population explosions can hang. Set a per-iteration agent cap or a wall-clock timeout. If prey/predator counts grow unboundedly, check reproduction probability application -- a missing threshold check is the usual cause.
- **Empty population crash:** Mean energy of an extinct species causes division by zero. Guard with `if len(agents) > 0` before computing means; record 0 or NaN for extinct populations.
- **Statistical test failure under one parameter set:** This usually indicates a scheduling or interaction bug that only manifests under higher agent density. Instrument per-phase agent counts to locate the divergence point.
- **Output format mismatch:** Verify DataFrame column names and dtypes match the specification exactly. Downstream validation pipelines will reject misnamed columns silently.
- **Food/resource state desynchronization:** If environmental metrics drift, check that cell state updates happen at the correct scheduling phase, not inside agent action loops.

## Limitations

- This approach requires a complete ODD specification. Incomplete or ambiguous model descriptions will produce implementations that may be internally consistent but not faithful to the intended model.
- Statistical validation requires reference data from a trusted implementation (e.g., a published NetLogo model). Without reference data, you can check executability and format but cannot confirm behavioral correctness.
- The Energy test with PCA requires at least 20-30 independent replications per implementation to have sufficient statistical power. Single-run comparisons are unreliable.
- Complex ABMs with heterogeneous agent types, network structures, or continuous space may need adaptations beyond the grid-based patterns described here.
- The paper found that even top-performing LLMs achieve ~67-100% success rates, meaning generated code should always be treated as a draft requiring validation, not a finished product.

## Reference

Fachada, N., Fernandes, D., Fernandes, C. M., & Matos-Carvalho, J. P. (2026). *Can Large Language Models Implement Agent-Based Models? An ODD-based Replication Study.* arXiv:2602.10140v1. Focus on: the six-stage evaluation pipeline (Table 2), the PPHPC ODD specification (Section 3), common failure modes by stage (Section 5), and the PCA + Energy test validation methodology (Section 4.3).