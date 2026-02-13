---
name: canonical-intermediate-representation-llm-based
description: >
  Translate natural language optimization problems into executable solver code using
  a Canonical Intermediate Representation (CIR) schema and multi-agent R2C pipeline.
  Decomposes operational rules into constraint archetypes and modeling paradigms before
  generating code. Triggers: "formulate this optimization problem", "write a solver for
  this scheduling problem", "convert these business rules to constraints", "model this
  linear program from the description", "generate Gurobi/PuLP code for this OR problem",
  "help me formulate these operational constraints mathematically".
---

# Canonical Intermediate Representation for Optimization Problem Formulation

This skill enables Claude to translate natural language descriptions of optimization
problems into executable mathematical programming code by using a structured intermediate
representation. Instead of jumping directly from English to solver code (which causes
LLMs to botch composite constraints and choose wrong modeling paradigms), Claude first
generates a **Canonical Intermediate Representation (CIR)** that explicitly maps each
operational rule to a constraint archetype and modeling paradigm, then instantiates that
CIR into solver code. This decouples *what a rule means* from *how it's encoded
mathematically*, dramatically improving correctness on complex problems.

## When to Use

- When a user provides a natural language description of a scheduling, routing, allocation, or resource planning problem and wants executable solver code
- When a user has business rules (e.g., "no nurse works more than 5 consecutive days", "each truck must return to the depot") that need mathematical constraint formulation
- When the user asks to formulate a mixed-integer program, linear program, or combinatorial optimization model from a word problem
- When converting operational requirements documents into optimization models using PuLP, Gurobi, OR-Tools, or CPLEX
- When the user has a partially formulated model and needs help adding complex conditional or composite constraints
- When debugging an optimization model where constraints don't correctly capture the intended business rule

## Key Technique

The core insight of this approach is that LLMs fail on optimization formulation not because they lack mathematical knowledge, but because they try to do too much in one step: simultaneously understanding the operational rule, choosing a modeling paradigm, and writing code. **CIR breaks this into explicit, auditable stages.** Each operational rule is first classified into a *constraint archetype* (assignment, capacity, flow conservation, temporal sequencing, cardinality, indicator/conditional, symmetry-breaking, linking) and paired with a *candidate modeling paradigm* (binary decision variables, big-M linearization, indicator constraints, piecewise-linear, flow-balance equations, set-covering). This classification is recorded in a structured schema before any code is written.

The **R2C (Rule-to-Constraint) framework** then operates as a multi-agent pipeline with four stages: (1) a **Parser Agent** extracts sets, parameters, decision variables, objectives, and operational rules from the natural language description; (2) a **CIR Synthesizer Agent** maps each extracted rule to constraint archetypes and modeling paradigms by retrieving from a knowledge base of known (archetype, paradigm, code-template) triples; (3) a **Code Generator Agent** instantiates the full optimization model in executable solver code; and (4) a **Reflection Agent** validates the generated code by checking constraint coverage, dimensional consistency, and solver feasibility, feeding corrections back if needed.

The knowledge base is the backbone: it stores canonical pairings of constraint archetypes with modeling paradigms alongside reusable code templates. When a new rule is encountered, the synthesizer retrieves the closest matching entries and adapts them. This retrieval-augmented approach means the system doesn't reinvent formulations from scratch — it composes proven patterns.

## Step-by-Step Workflow

1. **Extract problem components from natural language.** Parse the user's description to identify: (a) sets/indices (e.g., nurses, shifts, days), (b) parameters (e.g., demand per shift, max hours), (c) decision variables with their domains (binary, integer, continuous), (d) the objective function (minimize cost, maximize coverage), and (e) each distinct operational rule stated in the problem.

2. **Enumerate operational rules explicitly.** List every constraint-bearing sentence as a separate rule. For example, "Each nurse works at most 5 consecutive days and must have at least 2 weekends off per month" becomes two rules: R1 (consecutive day limit) and R2 (weekend-off minimum).

3. **Classify each rule into a constraint archetype.** Assign each rule to one of these archetypes:
   - **Assignment**: one-to-one or one-to-many allocation (e.g., assign exactly one nurse per shift)
   - **Capacity**: upper/lower bounds on resource usage (e.g., max hours per week)
   - **Flow conservation**: balance equations for network problems (e.g., inflow = outflow at each node)
   - **Temporal/sequencing**: ordering or adjacency constraints (e.g., no consecutive night shifts)
   - **Cardinality**: count-based limits (e.g., at most 3 products per warehouse)
   - **Indicator/conditional**: if-then logic (e.g., if facility is open, then capacity applies)
   - **Linking**: coupling between variable groups (e.g., production must equal shipments)
   - **Symmetry-breaking**: eliminate equivalent solutions (e.g., order identical machines)

4. **Select a modeling paradigm for each archetype.** Choose the mathematical encoding:
   - Binary decision variables for assignment/selection
   - Big-M linearization for conditional/indicator constraints
   - Native indicator constraints (if solver supports them, e.g., Gurobi `addGenConstrIndicator`)
   - Piecewise-linear approximation for nonlinear objectives
   - Flow-balance equalities for network conservation
   - Set-covering or set-partitioning formulations for coverage requirements
   - Auxiliary variables + linking constraints for composite rules

5. **Construct the CIR table.** For each rule, record a structured entry:
   ```
   Rule ID | Natural Language Rule | Archetype | Paradigm | Variables Involved | Constraint Sketch
   ```
   The constraint sketch is a semi-formal mathematical expression (not yet code) showing the constraint structure.

6. **Validate the CIR for completeness and consistency.** Check that: every rule from step 2 has a CIR entry, variable domains are consistent across constraints, the objective references only defined variables, and no rule is modeled with a paradigm that contradicts another (e.g., two constraints requiring contradictory variable domains).

7. **Generate executable solver code from the CIR.** Translate each CIR entry into solver API calls. Use PuLP by default (most accessible), Gurobi for performance-critical problems, or OR-Tools if the user specifies. Structure the code as: (a) import solver, (b) define model, (c) create variables, (d) add constraints grouped by CIR rule ID, (e) set objective, (f) solve, (g) extract and print results.

8. **Add constraint annotations.** Comment each constraint block with its Rule ID, natural language rule, and archetype so the code is auditable and the user can trace each line back to the original problem statement.

9. **Run the reflection check.** Verify the generated code by mentally executing: Are all indices bounded? Do big-M values have justified bounds (not arbitrary 1e6)? Are binary variables correctly typed? Does the objective direction match the problem (min vs max)? Are there off-by-one errors in temporal constraints?

10. **Present the CIR table and code together.** Always show the user the intermediate CIR table before or alongside the code, so they can validate the modeling decisions independently of the code.

## Concrete Examples

**Example 1: Nurse Scheduling Problem**

User: "I need to schedule 20 nurses across 3 shifts (morning, evening, night) over 7 days. Each shift needs exactly 5 nurses. No nurse can work more than 5 days. No nurse should work night shift followed by morning shift the next day. Minimize the total number of night shifts assigned."

Approach:

1. Extract: Sets = {nurses 1..20, shifts {M,E,N}, days 1..7}; Variables = x[n,s,d] binary; Objective = minimize sum of x[n,N,d]
2. Rules: R1 (demand: exactly 5 per shift/day), R2 (max 5 days per nurse), R3 (no night-then-morning)
3. CIR Table:

```
R1 | "Each shift needs exactly 5 nurses"  | Assignment   | Set-partitioning  | x[n,s,d] | sum_n x[n,s,d] = 5, forall s,d
R2 | "No nurse works more than 5 days"    | Capacity     | Binary sum bound   | x[n,s,d] | sum_{s,d} x[n,s,d] <= 5, forall n  (note: sum over s first gives at most 1 per day)
R3 | "No night then morning next day"     | Temporal      | Binary linking     | x[n,s,d] | x[n,N,d] + x[n,M,d+1] <= 1, forall n, d<7
```

4. Generated code (PuLP):

```python
import pulp

nurses = range(20)
shifts = ['M', 'E', 'N']
days = range(7)

model = pulp.LpProblem("NurseScheduling", pulp.LpMinimize)

# Decision variables
x = pulp.LpVariable.dicts("x", (nurses, shifts, days), cat='Binary')

# Objective: minimize total night shifts
model += pulp.lpSum(x[n]['N'][d] for n in nurses for d in days)

# R1 - Assignment: exactly 5 nurses per shift per day
for s in shifts:
    for d in days:
        model += pulp.lpSum(x[n][s][d] for n in nurses) == 5

# R2 - Capacity: each nurse works at most 5 days
for n in nurses:
    model += pulp.lpSum(x[n][s][d] for s in shifts for d in days) <= 5

# R2 auxiliary - at most one shift per nurse per day
for n in nurses:
    for d in days:
        model += pulp.lpSum(x[n][s][d] for s in shifts) <= 1

# R3 - Temporal: no night shift followed by morning shift next day
for n in nurses:
    for d in range(6):
        model += x[n]['N'][d] + x[n]['M'][d+1] <= 1

model.solve()
for n in nurses:
    schedule = [s for d in days for s in shifts if x[n][s][d].varValue > 0.5]
    print(f"Nurse {n}: {schedule}")
```

**Example 2: Facility Location with Conditional Capacity**

User: "I have 10 candidate warehouse locations and 50 customers. Each warehouse has a fixed opening cost and a capacity. Each customer must be served by exactly one open warehouse. If a warehouse is open, it can serve at most its capacity. Minimize total fixed cost plus transportation cost."

Approach:

1. Extract: Sets = {warehouses 1..10, customers 1..50}; Params = fixed_cost[w], capacity[w], transport_cost[w,c], demand[c]; Variables = y[w] binary (open), x[w,c] binary (assignment); Objective = min sum(fixed_cost[w]*y[w]) + sum(transport_cost[w,c]*x[w,c])
2. Rules: R1 (each customer served by exactly one warehouse), R2 (only open warehouses serve), R3 (capacity limit on open warehouses)
3. CIR Table:

```
R1 | "Each customer served by exactly one" | Assignment  | Set-partitioning | x[w,c]     | sum_w x[w,c] = 1, forall c
R2 | "Only open warehouses can serve"      | Indicator   | Binary linking   | x[w,c],y[w]| x[w,c] <= y[w], forall w,c
R3 | "Capacity limit if open"              | Capacity    | Binary linking   | x[w,c],y[w]| sum_c demand[c]*x[w,c] <= capacity[w]*y[w], forall w
```

Note: R3 subsumes R2 (if y[w]=0, no customer can be assigned), but keeping R2 explicitly tightens the LP relaxation, which is good practice.

```python
import pulp

warehouses = range(10)
customers = range(50)
# Assume these are loaded from data:
# fixed_cost[w], capacity[w], transport_cost[w][c], demand[c]

model = pulp.LpProblem("FacilityLocation", pulp.LpMinimize)

y = pulp.LpVariable.dicts("open", warehouses, cat='Binary')
x = pulp.LpVariable.dicts("assign", (warehouses, customers), cat='Binary')

# Objective
model += (pulp.lpSum(fixed_cost[w] * y[w] for w in warehouses) +
          pulp.lpSum(transport_cost[w][c] * x[w][c] for w in warehouses for c in customers))

# R1 - Assignment: each customer to exactly one warehouse
for c in customers:
    model += pulp.lpSum(x[w][c] for w in warehouses) == 1

# R2 - Indicator: assignment only if open
for w in warehouses:
    for c in customers:
        model += x[w][c] <= y[w]

# R3 - Capacity: respect warehouse capacity
for w in warehouses:
    model += pulp.lpSum(demand[c] * x[w][c] for c in customers) <= capacity[w] * y[w]

model.solve()
```

**Example 3: Vehicle Routing with Time Windows (Partial)**

User: "Each delivery truck starts and ends at the depot. Every customer must be visited exactly once. Each truck has capacity 100. Customers have time windows [a_i, b_i]. Minimize total travel distance."

CIR Table (key constraints only):

```
R1 | "Every customer visited once"     | Assignment         | Set-partitioning | x[i,j,k] | sum_{j,k} x[i,j,k] = 1, forall customer i
R2 | "Truck starts/ends at depot"      | Flow conservation  | Flow-balance     | x[i,j,k] | sum_j x[0,j,k] = 1, sum_j x[j,0,k] = 1, forall k
R3 | "Flow conservation at customers"  | Flow conservation  | Flow-balance     | x[i,j,k] | sum_j x[j,i,k] = sum_j x[i,j,k], forall i,k
R4 | "Truck capacity"                  | Capacity           | Binary sum bound | x[i,j,k] | sum_i demand[i]*sum_j x[i,j,k] <= 100, forall k
R5 | "Time windows"                    | Temporal           | Big-M linking    | t[i],x    | t[i] + travel[i,j] - M*(1-x[i,j,k]) <= t[j], a[i] <= t[i] <= b[i]
```

The Big-M for R5 should be set to `max(b[i] + travel[i,j] - a[j])` for each (i,j) pair — never an arbitrary large number.

## Best Practices

- **Do:** Always produce the CIR table before writing code. This is the core differentiator — skipping it defeats the purpose and leads to the same errors as direct NL-to-code generation.
- **Do:** Choose tight Big-M values derived from problem data. For time-window constraints, compute M[i,j] = b[i] + travel_time[i,j] - a[j]. Loose Big-M values (like 1e6) cause numerical instability and slow solving.
- **Do:** Add redundant constraints that tighten the LP relaxation when they're cheap (like R2 in the facility location example). This improves solver performance significantly.
- **Do:** Comment every constraint block with its Rule ID and natural language origin. This makes the model auditable and debuggable.
- **Avoid:** Jumping directly from natural language to code without the CIR step. This is the failure mode the paper identifies — composite rules get partially modeled or use wrong paradigms.
- **Avoid:** Using a single paradigm for all constraints. Different rules need different paradigms — forcing everything into big-M or everything into indicator constraints produces incorrect or inefficient models.
- **Avoid:** Treating the CIR as optional documentation. It is the **reasoning artifact** — if the CIR is wrong, the code will be wrong. Validate the CIR with the user before generating code.

## Error Handling

- **Missing constraints:** After constructing the CIR, count the rules and count the constraint blocks in code. If they don't match, a rule was dropped. Re-read the problem statement to find it.
- **Infeasible model:** When the solver returns infeasible, use the CIR table to systematically relax one constraint at a time (by commenting it out) to identify which rule combination causes infeasibility. Report the conflicting rules to the user.
- **Wrong variable domains:** If continuous variables are used where binary is needed (common with indicator constraints), the model will produce fractional solutions. Check each CIR entry's paradigm requires the correct variable type.
- **Big-M too loose:** If solve time is excessive on MIP problems, check Big-M values. Replace arbitrary constants with data-derived bounds from the CIR constraint sketch.
- **Archetype misclassification:** If a temporal constraint is classified as capacity (or vice versa), the chosen paradigm will be wrong. When in doubt, re-read the rule and ask: does it bound a *quantity* (capacity), an *ordering* (temporal), a *selection* (assignment), or a *conditional activation* (indicator)?

## Limitations

- This approach is designed for **mathematical programming** problems (LP, MIP, MILP). It does not apply to constraint programming, SAT solving, or metaheuristic optimization without adaptation.
- The CIR schema assumes constraints can be cleanly decomposed into independent rules. Problems with deeply interleaved constraints (where one rule's meaning depends on another) may require iterative CIR refinement.
- The approach works best for structured operational problems (scheduling, routing, facility location, production planning). Highly novel or research-frontier formulations may not have matching archetypes in the knowledge base.
- Code generation targets Python solver APIs (PuLP, Gurobi, OR-Tools). For other languages or solvers (AMPL, GAMS, Julia/JuMP), the CIR is still valid but code templates need adaptation.
- The paper reports 47.2% accuracy on their hardest benchmark — complex multi-rule problems remain genuinely difficult. Always have the user validate the CIR before treating the code as correct.

## Reference

**Paper:** Lyu et al., "Canonical Intermediate Representation for LLM-based optimization problem formulation and code generation" (arXiv:2602.02029, 2026). Look for: the CIR schema definition (Section 3), constraint archetype taxonomy (Table 2), modeling paradigm catalog (Table 3), and the R2C pipeline architecture (Figure 2).