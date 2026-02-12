---
name: "ai-agent-for-reverseengineering"
description: >
  Reverse-engineer legacy numerical/scientific Fortran or C code and translate it into modern
  Python frameworks (Devito, NumPy, SciPy, FEniCS, etc.) using a multi-stage analysis pipeline
  with knowledge-graph-guided retrieval, structured code synthesis, and iterative validation.
  Trigger phrases: "convert this Fortran code to Python", "reverse engineer this finite difference code",
  "translate this legacy numerical solver to Devito", "modernize this scientific computing code",
  "what does this Fortran stencil do and how do I write it in Python",
  "migrate this CFD solver from Fortran to a modern framework"
---

# Reverse-Engineering Legacy Scientific Code and Translating to Modern Frameworks

This skill enables Claude to systematically reverse-engineer legacy finite-difference and numerical
simulation code (Fortran, C, or older scientific codebases) and translate it into modern Python-based
frameworks such as Devito, NumPy/SciPy, or FEniCS. The approach follows the multi-stage pipeline from
Hou & Yang (2026): static analysis to extract computational structure, knowledge-graph-organized
retrieval to map legacy patterns onto target framework APIs, Pydantic-style constrained code synthesis
to produce correct output, and multi-dimensional validation covering execution correctness, mathematical
consistency, and API compliance.

## When to Use

- When the user provides Fortran, C, or MATLAB finite-difference code and asks to rewrite it in Python/Devito
- When the user wants to understand what a legacy numerical stencil computes (reverse engineering)
- When translating a seismic wave simulation, CFD solver, or heat equation solver from a legacy language to a modern symbolic PDE framework
- When the user asks to "modernize" or "migrate" scientific computing code with explicit loop-based array operations to vectorized or symbolic equivalents
- When the user needs to verify that a translated numerical code preserves the original mathematical formulation
- When refactoring a large legacy Fortran codebase and needing to understand module-level dependencies before rewriting

## Key Technique

**Multi-stage reverse engineering with structured retrieval and constrained synthesis.** The core insight
from Hou & Yang is that naive LLM-based code translation fails on scientific code because finite-difference
stencils encode implicit mathematical relationships (PDEs, boundary conditions, stability constraints) that
are not apparent from syntax alone. The solution is a three-level analysis pipeline: (1) function-level
static analysis to identify computational kernels and stencil patterns, (2) module-level analysis to
capture data flow and organizational structure, and (3) codebase-level dependency mapping across files.

**Knowledge-graph-guided retrieval for target framework mapping.** Rather than relying on the LLM's
parametric knowledge of the target framework, the approach builds a structured knowledge graph of the
target API (e.g., Devito's `Function`, `TimeFunction`, `Eq`, `Operator` classes) organized into semantic
communities via Leiden clustering. When translating a specific stencil, the system retrieves the most
relevant API patterns from the correct community (e.g., seismic simulation vs. CFD vs. performance
tuning), then expands the query with related concepts to capture edge cases like boundary handling or
subdomain specifications.

**Iterative validation with feedback-driven refinement.** Generated code is validated across four
dimensions: execution correctness (does it run?), structural soundness (does it follow framework idioms?),
mathematical consistency (does it implement the same PDE discretization?), and API compliance (does it
use the target framework correctly?). When validation fails, the specific failure dimension feeds back
into retrieval weighting, causing the system to pull more context from the relevant knowledge community
on the next iteration. This transforms static translation into an adaptive refinement loop.

## Step-by-Step Workflow

1. **Parse the legacy source code statically.** Identify all subroutines/functions, their call graph, array declarations with dimensions, loop nests, and index arithmetic. For Fortran, pay special attention to COMMON blocks, IMPLICIT typing, and column-major array ordering. Produce a structured inventory: `{function_name, arguments, array_accesses, loop_bounds, stencil_offsets}`.

2. **Extract the computational stencil from each kernel.** For every nested loop that updates an array, determine the stencil shape by collecting all relative index offsets (e.g., `u(i-1,j)`, `u(i+1,j)`, `u(i,j-1)`, `u(i,j+1)` indicates a 5-point Laplacian). Record the coefficients multiplying each offset to reconstruct the finite-difference weights.

3. **Identify the governing PDE and discretization scheme.** From the stencil weights, spatial dimensions, and time-stepping structure, determine which PDE is being solved (wave equation, heat equation, Navier-Stokes, etc.) and the discretization order (second-order central differences, fourth-order, upwind, etc.). Document the CFL condition or stability constraints if present.

4. **Map boundary conditions and initial conditions.** Analyze code outside the main stencil loops for boundary handling: Dirichlet (fixed values), Neumann (gradient conditions), absorbing boundaries (PML, sponge layers), or periodic wrapping. Record these as structured constraints.

5. **Build a target-framework knowledge map.** For the target framework (Devito, NumPy, FEniCS, etc.), organize the relevant API into categories: grid/mesh construction, field variable declaration, equation specification, operator compilation, boundary condition application, and time-stepping control. If using Devito, map: `Grid` for domain, `TimeFunction`/`Function` for fields, `Eq` for stencil equations, `Operator` for compiled kernels.

6. **Translate each component using constrained synthesis.** Generate the target code component by component, enforcing structural constraints: (a) grid dimensions must match the original, (b) stencil order must match extracted weights, (c) boundary conditions must be explicitly applied, (d) time-stepping loop structure must preserve the original update sequence. Use Pydantic-style validation schemas to verify each component before assembly.

7. **Assemble the full translated program.** Combine grid setup, field initialization, equation definitions, boundary conditions, and the time-stepping driver into a complete runnable script. Preserve the original code's I/O structure (reading input parameters, writing output snapshots) adapted to Python conventions.

8. **Validate across four dimensions.** Run the translated code and check: (a) *Execution*: does it run without errors? (b) *Structure*: does it follow target framework idioms (no raw NumPy loops where symbolic operators should be used)? (c) *Mathematics*: do the stencil coefficients match the original discretization order? (d) *API compliance*: are framework-specific objects used correctly (e.g., Devito `Operator` vs. manual loops)?

9. **Iterate on failures with targeted retrieval.** If validation fails on a specific dimension, refine that component. For mathematical failures, re-examine the stencil extraction. For API failures, retrieve more framework documentation for the specific construct that failed. For execution failures, check array shapes, index ordering (column-major vs. row-major), and off-by-one errors in loop bounds.

10. **Document the translation mapping.** Produce a summary table mapping each original Fortran subroutine to its Python equivalent, noting any semantic changes (e.g., 1-based to 0-based indexing, column-major to row-major array layout, explicit loops to symbolic operators).

## Concrete Examples

**Example 1: Fortran 2D acoustic wave equation to Devito**

User: "Convert this Fortran code to Devito:"
```fortran
do it = 1, nt
  do j = 2, ny-1
    do i = 2, nx-1
      u_new(i,j) = 2*u(i,j) - u_old(i,j) + &
        (dt**2 * v(i,j)**2) * ( &
          (u(i+1,j) - 2*u(i,j) + u(i-1,j))/dx**2 + &
          (u(i,j+1) - 2*u(i,j) + u(i,j-1))/dy**2 )
    end do
  end do
  u_old = u
  u = u_new
end do
```

Approach:
1. Extract stencil: 5-point Laplacian with offsets `[-1,0], [+1,0], [0,-1], [0,+1]`, coefficients `[1,-2,1]/dx^2` in x and `[1,-2,1]/dy^2` in y. This is the 2nd-order central difference for the 2D acoustic wave equation `d2u/dt2 = v^2 * (d2u/dx2 + d2u/dy2)`.
2. Identify time discretization: 2nd-order explicit leapfrog (`2*u - u_old + dt^2 * ...`).
3. Boundary conditions: implicit zero-Dirichlet (loops start at index 2, end at n-1).
4. Translate to Devito symbolic API.

Output:
```python
from devito import Grid, TimeFunction, Function, Eq, Operator

grid = Grid(shape=(nx, ny), extent=(nx*dx, ny*dy))
u = TimeFunction(name='u', grid=grid, time_order=2, space_order=2)
v = Function(name='v', grid=grid)
v.data[:] = velocity_model  # load from original input

pde = u.dt2 - v**2 * (u.dx2 + u.dy2)
stencil = Eq(u.forward, solve(pde, u.forward))

op = Operator([stencil])
op.apply(time_M=nt-1, dt=dt)
```

**Example 2: Understanding and documenting a legacy Fortran stencil**

User: "What does this Fortran code compute?"
```fortran
do j = 2, ny-1
  do i = 2, nx-1
    phi_new(i,j) = 0.25*(phi(i+1,j)+phi(i-1,j)+phi(i,j+1)+phi(i,j-1))
  end do
end do
```

Approach:
1. Extract stencil: 4 neighbors with equal weight 0.25, no self-term.
2. Recognize pattern: Jacobi iteration for the 2D Laplace equation `d2phi/dx2 + d2phi/dy2 = 0`.
3. Confirm: averaging the 4 neighbors is the standard Jacobi relaxation update.

Output:
```
This is a single Jacobi iteration step for solving the 2D Laplace equation
(nabla^2 phi = 0) on a uniform grid. Each interior point is replaced by the
average of its 4 neighbors. The loop excludes boundaries (indices 2 to n-1
in Fortran 1-based indexing), implying Dirichlet boundary conditions held
fixed at the array edges.

To converge, this must be wrapped in an outer iteration loop checking
a residual tolerance (e.g., max|phi_new - phi| < epsilon).
```

**Example 3: Translating a 1D advection scheme with upwind differencing**

User: "Translate this C finite-difference code to NumPy:"
```c
for (int n = 0; n < nt; n++) {
    for (int i = 1; i < nx; i++) {
        u_new[i] = u[i] - c * dt / dx * (u[i] - u[i-1]);
    }
    memcpy(u, u_new, nx * sizeof(double));
}
```

Approach:
1. Extract stencil: backward difference `(u[i] - u[i-1])/dx`, first-order upwind for advection equation `du/dt + c*du/dx = 0`.
2. Note: loop starts at `i=1`, so `u[0]` is a fixed inflow boundary.
3. Translate using NumPy vectorized slicing instead of explicit loops.

Output:
```python
import numpy as np

u = u_initial.copy()
courant = c * dt / dx  # CFL number, must be <= 1 for stability

for n in range(nt):
    u[1:] = u[1:] - courant * (u[1:] - u[:-1])
    # u[0] remains fixed (inflow Dirichlet BC)
```

## Best Practices

- **Do:** Always extract and verify stencil coefficients before translating. A single wrong sign or missing factor of 2 changes the PDE being solved.
- **Do:** Account for indexing convention differences. Fortran is 1-based and column-major; Python/C is 0-based and row-major. Transpose array layouts when the original code uses column-major access patterns for performance.
- **Do:** Preserve the time-stepping order. If the original uses leapfrog (3-level), don't silently convert to forward Euler (2-level) -- the stability properties change entirely.
- **Do:** Validate the CFL number constraint. If the original code had `dt <= dx / v_max`, the translated code must enforce the same condition.
- **Avoid:** Blindly converting explicit Fortran loops into Python loops. Use vectorized NumPy operations or symbolic framework operators (Devito `Operator`, FEniCS `solve`) for performance.
- **Avoid:** Ignoring COMMON blocks, module-level state, or IMPLICIT NONE declarations in Fortran. These define variable types and sharing patterns that affect correctness.
- **Avoid:** Assuming boundary conditions are "just zeros." Analyze all code paths that modify boundary array elements, including absorbing boundary layers (PML/sponge) which require additional auxiliary fields.

## Error Handling

- **Stencil extraction ambiguity:** When array indexing uses computed offsets (e.g., `u(idx(i)+1, j)`) rather than literal constants, trace the index computation to resolve the actual offsets. If unresolvable, flag to the user and ask for clarification.
- **Mixed precision issues:** Fortran's `REAL*4` vs `REAL*8` affects numerical stability. Default to `float64` in Python and note where the original used single precision.
- **Missing boundary code:** If the legacy code relies on array initialization to zero for boundary conditions (common in Fortran), make this explicit in the translation with a comment.
- **Framework API mismatches:** If the target framework lacks a direct equivalent (e.g., Devito doesn't support a specific boundary type natively), implement it as a manual NumPy operation applied between operator calls, and document the workaround.
- **Convergence differences:** Iterative solvers (Jacobi, Gauss-Seidel) may converge at different rates due to floating-point ordering differences. Validate against the original's output, not just against the analytical solution.

## Limitations

- This approach works best for structured-grid finite-difference codes. Unstructured mesh codes (finite element, finite volume on irregular grids) require different analysis techniques.
- Legacy codes with heavy preprocessor macros (C `#ifdef` forests, Fortran `cpp` directives) need macro expansion before stencil extraction is reliable.
- Codes that interleave I/O, MPI communication, and computation in the same loop nest require separating concerns before translation; the stencil extraction assumes clean computational kernels.
- Implicit solvers (solving linear systems at each timestep) are harder to translate than explicit time-stepping codes, since the solver choice affects the API mapping significantly.
- Performance equivalence is not guaranteed. The translated code may be slower or faster depending on framework overhead, JIT compilation, and memory layout differences.

## Reference

Hou, Y. & Yang, Z. (2026). *AI Agent for Reverse-Engineering Legacy Finite-Difference Code and Translating to Devito.* arXiv:2601.18381v1. Key takeaway: the three-level static analysis (function/module/codebase) combined with knowledge-graph-organized retrieval and four-dimensional validation produces reliable translations where single-pass LLM translation fails.