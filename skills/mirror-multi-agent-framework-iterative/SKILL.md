---
name: "mirror-multi-agent-framework-iterative"
description: "Translate natural language optimization problems into mathematical models and solver code using MIRROR's multi-agent pipeline with iterative error correction and hierarchical retrieval. Use when: 'solve this optimization problem', 'write a linear program for', 'model this scheduling/routing/allocation problem', 'convert this OR problem to Gurobi code', 'formulate constraints for this optimization', 'help me model this integer program'."
---

# MIRROR: Multi-Agent Iterative Revision for Operations Research Modeling

This skill enables Claude to translate natural language descriptions of optimization problems into formal mathematical models and executable solver code (Gurobi/PuLP/OR-Tools). It applies the MIRROR framework's four-phase generation pipeline -- parameter extraction, domain advising, mathematical modeling, and code generation -- followed by execution-driven iterative revision that diagnoses and corrects both formulation errors and implementation bugs. The approach systematically decomposes the expert-level task of OR modeling into specialized sub-tasks, each handled with domain-aware reasoning, producing reliable optimization solutions for non-expert users.

## When to Use

- When the user describes a real-world optimization problem in natural language and wants a mathematical formulation (LP, ILP, MIP, QP, etc.)
- When the user asks to generate solver code (Gurobi, PuLP, OR-Tools, CPLEX) from a problem description
- When the user has an existing optimization model that produces wrong answers or solver errors and needs systematic debugging
- When the user needs to model scheduling, routing, assignment, knapsack, facility location, network flow, or resource allocation problems
- When the user provides tabular data and asks to formulate an optimization over it
- When the user asks to convert a word problem or business scenario into constraints and an objective function

## Key Technique

MIRROR decomposes optimization modeling into a sequential pipeline of four specialized agent roles during **generation**, then activates two revision agents during **correction**. The generation agents are: (1) a Parameter Extraction agent that identifies all decision variables, constants, and data from the problem statement and structures them as typed JSON; (2) a Modeling Advisor that interprets domain-specific terminology and clarifies ambiguous problem characteristics; (3) a Mathematical Modeling agent that produces the formal optimization model (variables, constraints, objective); and (4) a Code Generation agent that translates the model into executable solver code.

The critical innovation is **Iterative Adaptive Revision (IAR)**. After code is generated and executed, failures trigger a feedback loop. A Modeling Revision agent diagnoses logical or formulation errors (wrong constraints, missing edge cases, incorrect objective), while a Code Revision agent handles implementation failures (syntax errors, API misuse, data type mismatches). Both agents have access to a local memory of previous attempts and revision tips for the current problem, preventing repeated mistakes. The loop continues until the solver returns a valid solution or a retry limit is reached.

The second innovation is **Hierarchical Retrieval-Augmented Generation (HRAG)**: a two-stage retrieval process that first performs coarse semantic filtering over an exemplar library of verified (problem, model, code) triples, then fine-grained reranking by problem category alignment. This injects precise domain knowledge -- not generic examples, but structurally similar optimization patterns -- into the modeling and coding steps. When no sufficiently similar exemplar exists, the system signals empty rather than injecting noise.

## Step-by-Step Workflow

1. **Extract parameters**: Read the problem description carefully. Identify every quantitative element: decision variables (what we choose), parameters/constants (given data), indices/sets (dimensions of the problem). Structure them as typed entries with symbols, data types, and semantic meanings.

2. **Advise on domain semantics**: Determine the problem category (LP, ILP, MIP, network flow, scheduling, etc.). Clarify any domain-specific terms (e.g., "throughput" means units per hour, "makespan" means total completion time). Identify implicit constraints the user may not have stated (e.g., non-negativity, integrality, capacity limits).

3. **Retrieve similar exemplars**: Search your knowledge for structurally similar optimization problems. If the problem is a variant of a classic type (bin packing, TSP, assignment, flow), recall the standard formulation pattern. Adapt it to the specifics rather than building from scratch.

4. **Formulate the mathematical model**: Define decision variables with explicit domains. Write each constraint as a mathematical inequality or equality with clear indexing. State the objective function (minimize or maximize) precisely. Present the complete model in standard mathematical notation.

5. **Generate solver code**: Translate the mathematical model into executable Python code using the user's preferred solver (default to PuLP for accessibility, Gurobi if requested). Include data setup, model construction, constraint addition, objective setting, solve call, and result extraction. Add status checking after the solve.

6. **Execute and validate**: Run the code. Check for three categories of failure: (a) syntax/runtime errors, (b) infeasible/unbounded model status, (c) solution exists but may be incorrect (sanity-check against problem constraints).

7. **Diagnose errors with modeling revision**: If the model is infeasible or produces clearly wrong results, revisit the formulation. Check: Are constraints too tight? Is the objective direction correct? Are variable domains correct (continuous vs. integer vs. binary)? Are index sets complete? Generate specific revision tips.

8. **Diagnose errors with code revision**: If there are runtime errors or API issues, fix implementation bugs: incorrect solver API calls, data type mismatches, missing variable bounds, wrong constraint sense operators. Preserve the mathematical intent while fixing the code.

9. **Iterate with memory**: Re-execute the revised code. On each iteration, carry forward the history of previous errors and fixes to avoid repeating failed approaches. Continue for up to 3-5 revision cycles.

10. **Present the solution**: Return the final mathematical model, the working code, the optimal objective value, and the decision variable values. Explain what the solution means in the context of the original problem.

## Concrete Examples

**Example 1: Production Planning Problem**

User: "A factory makes two products A and B. Product A needs 2 hours of machining and 1 hour of assembly. Product B needs 1 hour of machining and 3 hours of assembly. There are 100 machining hours and 90 assembly hours available. Product A profits $40, product B profits $60. How many of each should they make?"

Approach:
1. Extract parameters: x_A, x_B (decision variables, integer >= 0), machining coefficients [2, 1], assembly coefficients [1, 3], capacities [100, 90], profits [40, 60]
2. Classify as a simple linear program (LP), two-variable resource allocation
3. Formulate:
   - Maximize: 40*x_A + 60*x_B
   - Subject to: 2*x_A + x_B <= 100, x_A + 3*x_B <= 90, x_A >= 0, x_B >= 0

Output:
```python
from pulp import *

prob = LpProblem("production_planning", LpMaximize)
x_A = LpVariable("x_A", lowBound=0)
x_B = LpVariable("x_B", lowBound=0)

prob += 40 * x_A + 60 * x_B, "Total_Profit"
prob += 2 * x_A + x_B <= 100, "Machining_Capacity"
prob += x_A + 3 * x_B <= 90, "Assembly_Capacity"

prob.solve(PULP_CBC_CMD(msg=0))
print(f"Status: {LpStatus[prob.status]}")
print(f"x_A = {x_A.varValue}, x_B = {x_B.varValue}")
print(f"Max Profit = ${value(prob.objective)}")
```
Result: x_A = 42.0, x_B = 16.0, Max Profit = $2640.0

**Example 2: Job Scheduling with Debugging**

User: "I have 5 jobs that each need to be assigned to one of 3 machines. Each job has a different processing time on each machine. Minimize the makespan (time when the last job finishes). Here are the processing times: Job 1: [3,5,2], Job 2: [4,2,6], Job 3: [5,4,3], Job 4: [2,3,4], Job 5: [6,5,2]."

Approach:
1. Extract: binary variables x[i,j] (job i on machine j), continuous variable makespan, processing time matrix P[i][j]
2. Classify as a minimum-makespan assignment problem (ILP)
3. Formulate:
   - Minimize: makespan
   - Subject to: sum_j(x[i,j]) = 1 for each job i (each job assigned exactly once)
   - makespan >= sum_i(P[i][j] * x[i,j]) for each machine j (makespan >= load on every machine)
   - x[i,j] in {0, 1}
4. Generate code, execute, verify solution

Output:
```python
from pulp import *

jobs = range(5)
machines = range(3)
P = [[3,5,2],[4,2,6],[5,4,3],[2,3,4],[6,5,2]]

prob = LpProblem("min_makespan", LpMinimize)
x = LpVariable.dicts("x", (jobs, machines), cat="Binary")
makespan = LpVariable("makespan", lowBound=0)

prob += makespan

for i in jobs:
    prob += lpSum(x[i][j] for j in machines) == 1, f"assign_job_{i}"

for j in machines:
    prob += lpSum(P[i][j] * x[i][j] for i in jobs) <= makespan, f"makespan_machine_{j}"

prob.solve(PULP_CBC_CMD(msg=0))
print(f"Optimal makespan: {value(makespan)}")
for i in jobs:
    for j in machines:
        if x[i][j].varValue > 0.5:
            print(f"  Job {i+1} -> Machine {j+1} (time: {P[i][j]})")
```

**Example 3: Iterative Revision on Infeasible Model**

User: "A delivery company has 3 trucks (capacity 10 tons each) and 8 packages weighing [3, 4, 5, 2, 6, 3, 4, 7] tons. Each package must be delivered. Minimize the number of trucks used."

Approach (showing revision cycle):
1. First attempt formulates bin packing. Solver returns infeasible.
2. **Modeling revision**: Total weight = 34 tons, capacity = 30 tons. Three trucks cannot carry all packages. Diagnose: the problem as stated is infeasible with the given fleet.
3. **Revision tip**: Report infeasibility to user with explanation. Ask if truck capacity or count should be adjusted.
4. If user says "use up to 5 trucks": revise model with y[j] binary (truck j used), j in 0..4, re-solve. Now feasible with 4 trucks.

This demonstrates the IAR loop: rather than silently failing, the system diagnoses the root cause (insufficient capacity) and communicates it.

## Best Practices

- **Do:** Always start by extracting and listing all parameters before writing any math. Missing a parameter is the most common source of wrong models.
- **Do:** Explicitly state variable domains (continuous, integer, binary). A common error is treating integer variables as continuous, producing fractional solutions that are meaningless.
- **Do:** Check model feasibility before interpreting results. An infeasible or unbounded status means the formulation has errors, not that the problem has no solution.
- **Do:** Sanity-check solutions against the original problem. If a scheduling model says to produce negative quantities, the formulation is wrong.
- **Avoid:** Jumping directly to code without first writing the mathematical formulation. The model is the source of truth; code is just a translation.
- **Avoid:** Adding unnecessary complexity. If the problem is a simple LP, do not introduce binary variables. Match the model complexity to the problem.
- **Avoid:** Silently ignoring solver warnings or non-optimal statuses. Always check `prob.status` and report it.

## Error Handling

| Error Type | Diagnosis | Fix |
|---|---|---|
| **Model infeasible** | Constraints are contradictory or data makes feasibility impossible | Relax constraints, check data consistency, add slack variables to identify which constraints conflict |
| **Model unbounded** | Missing a constraint that bounds the objective | Check for missing capacity/budget constraints, ensure all variables have appropriate bounds |
| **Wrong numerical answer** | Formulation error (wrong constraint direction, missing constraint, wrong objective) | Compare model against problem statement line by line; verify constraint sense (<=, >=, ==) |
| **Solver timeout** | Problem too large or poorly formulated for branch-and-bound | Add tighter bounds, provide warm-start solution, consider relaxation or decomposition |
| **Runtime/syntax error** | API misuse or data type mismatch | Check solver documentation for correct method signatures; ensure data arrays match index dimensions |
| **Fractional solution for integer problem** | Variables declared as continuous instead of integer/binary | Change `cat="Continuous"` to `cat="Integer"` or `cat="Binary"` |

## Limitations

- This approach works best for problems expressible as LP, ILP, MIP, or QP. It does not handle constraint programming, metaheuristics, or simulation-based optimization natively.
- Very large-scale problems (millions of variables) may require decomposition techniques (Benders, Dantzig-Wolfe, column generation) that go beyond direct formulation.
- Nonlinear, non-convex objectives or constraints require specialized solvers (BARON, SCIP) and are harder to formulate correctly from natural language alone.
- The quality of the model depends on the completeness of the problem description. Ambiguous or incomplete specifications will produce models that solve the wrong problem.
- Domain-specific constraints (regulatory, physical, business rules) that are implied but not stated in the problem description will be missed unless the user provides them.

## Reference

**Paper:** [MIRROR: A Multi-Agent Framework with Iterative Adaptive Revision and Hierarchical Retrieval for Optimization Modeling in Operations Research](https://arxiv.org/abs/2602.03318v2) (Shi et al., 2026). Key takeaway: decomposing OR modeling into parameter extraction, domain advising, mathematical formulation, and code generation -- with execution-driven revision loops and exemplar retrieval -- substantially outperforms monolithic approaches on both standard and industrial benchmarks.