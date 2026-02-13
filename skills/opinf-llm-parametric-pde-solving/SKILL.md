---
name: opinf-llm-parametric-pde-solving
description: >
  Solve parametric PDEs using operator inference with reduced-order models. Builds POD-based
  reduced dynamics from a small set of simulation snapshots, then predicts solutions at unseen
  parameters and boundary conditions via polynomial regression on learned operators.
  Trigger phrases: "solve this PDE for different parameters", "build a reduced-order model",
  "operator inference for PDE", "parametric PDE solver", "predict PDE solution at new viscosity",
  "ROM for heat/Burgers/Navier-Stokes equation"
---

# OpInf-LLM: Parametric PDE Solving via Operator Inference

This skill enables Claude to build and deploy reduced-order models (ROMs) for parametric partial differential equations using the Operator Inference (OpInf) framework. Given a small number of full-order simulation snapshots at a few parameter values, Claude constructs a shared POD basis, learns reduced quadratic operators via regularized least-squares, and then predicts solutions at arbitrary unseen parameters and boundary conditions through polynomial regression on those operators. The approach achieves high numerical accuracy with minimal computational cost and generalizes across heterogeneous PDE settings without re-running expensive solvers.

## When to Use

- When the user wants to solve a PDE (heat, diffusion, advection, Burgers, Navier-Stokes) across a range of physical parameters (viscosity, diffusivity, Reynolds number) without running a full solver for each
- When the user asks to build a reduced-order model or surrogate from existing simulation data
- When the user has FEM/FDM/spectral simulation snapshots and wants fast predictions at new parameter values
- When the user needs to predict PDE solutions under new initial or boundary conditions not seen during training
- When the user wants to extrapolate a PDE solution beyond the original time horizon
- When the user asks for a lightweight parametric solver that avoids retraining neural networks for each new configuration

## Key Technique

**Operator Inference (OpInf)** is a non-intrusive reduced-order modeling method. It works in two stages. In the **offline stage**, you collect solution snapshots from a full-order solver at a small set of training parameter values (e.g., 3-5 viscosity values). You concatenate all snapshots and compute a Proper Orthogonal Decomposition (POD) basis via truncated SVD -- keeping the first `r` left-singular vectors that capture ~99% of the snapshot energy. You then project the snapshots and their time derivatives onto this basis to get reduced modal coefficients. For each training parameter, you solve a regularized least-squares problem to identify reduced operators (linear `A`, quadratic `H`, input `B`, constant `c`) that best fit the reduced dynamics: `da/dt = A*a + H*(a kron a) + B*u + c`.

In the **online stage**, to predict at an unseen parameter value, you fit low-degree polynomials through the learned operator entries as a function of the parameter (e.g., `A(nu) ~ p0 + p1*nu + p2*nu^2`). This polynomial regression interpolates/extrapolates the operators cheaply. You then compute initial modal coefficients from the new initial condition, integrate the reduced ODE using a standard solver (BDF or RK45), and reconstruct the full-order solution via `y(x,t) = Phi @ a(t)`.

The critical advantage over direct LLM code generation or neural operator approaches is the **decoupling of accuracy from LLM reliability**. The LLM handles parsing, orchestration, and tool calling, while the numerical heavy lifting is done by well-conditioned linear algebra (SVD, least-squares, ODE integration). This yields both high execution success rates (~99%) and low prediction errors, even for unseen parameters.

## Step-by-Step Workflow

1. **Parse the PDE specification.** Extract the governing equation type (heat, Burgers, Navier-Stokes), spatial domain, time horizon, parameter name and range (e.g., viscosity `nu` in [0.01, 2.0]), initial conditions, and boundary conditions from the user's description.

2. **Generate or load training snapshots.** If the user has simulation data, load it as arrays of shape `[n_trajectories, n_timesteps, n_spatial_points]`. If not, write a full-order solver (finite differences, Chebyshev collocation, or FEniCS) for 3-5 training parameter values, each with 2-3 trajectories using varied ICs/BCs. Store snapshots and their time derivatives.

3. **Compute the shared POD basis.** Concatenate all training snapshots into a single matrix `S` of shape `[n_spatial_points, total_snapshots]`. Run truncated SVD: `U, sigma, Vt = np.linalg.svd(S, full_matrices=False)`. Select `r` modes where cumulative energy `sum(sigma[:r]^2) / sum(sigma^2) >= 0.99`. Set `Phi = U[:, :r]`.

4. **Project snapshots onto the POD basis.** For each training parameter `xi_i`, compute reduced coefficients `a(t) = Phi.T @ y(t)` and their time derivatives `da/dt = Phi.T @ dy/dt`. Assemble the Kron product matrix `a kron a` for quadratic terms.

5. **Solve the OpInf least-squares problem per parameter.** For each `xi_i`, set up the data matrix `D = [a.T, (a kron a).T, u.T, 1]` and target `R = (da/dt).T`. Solve `min ||D @ O - R||_F^2 + lambda * ||O||_F^2` using `np.linalg.lstsq` or Tikhonov-regularized normal equations. Extract operators `A(xi_i)`, `H(xi_i)`, `B(xi_i)`, `c(xi_i)`.

6. **Fit polynomial regressors on operators.** For each entry of each operator matrix, fit a polynomial of degree `p` (typically 1-3) across the training parameter values using `np.polyfit`. This yields functions `A(xi)`, `H(xi)`, `B(xi)`, `c(xi)` for arbitrary `xi`.

7. **Predict at unseen parameters.** Given a new parameter `xi_test`: evaluate the polynomial regressors to get `A(xi_test)`, `H(xi_test)`, etc. Compute initial reduced state `a0 = Phi.T @ y0`. Define the reduced ODE `da/dt = A*a + H*(a kron a) + B*u + c`. Integrate with `scipy.integrate.solve_ivp` (method `'BDF'` for stiff, `'RK45'` otherwise).

8. **Reconstruct the full-order solution.** Compute `y(t) = Phi @ a(t)` for all time steps. Reshape to the spatial grid for visualization or further analysis.

9. **Validate and report accuracy.** Compute relative L2 error: `||y_pred - y_ref||_2 / ||y_ref||_2`. If error exceeds threshold, suggest increasing `r`, adding training parameters, or adjusting regularization `lambda`.

10. **Visualize results.** Plot the spatiotemporal solution as a heatmap or animation. Overlay predicted vs. reference if available. Show the singular value decay to justify the chosen `r`.

## Concrete Examples

**Example 1: Parametric 1D Heat Equation**

User: "I have simulation data for the 1D heat equation with diffusivity values 0.1, 0.5, and 2.0. Build a reduced-order model and predict the solution at diffusivity 0.8 with a sinusoidal initial condition."

Approach:
1. Load the snapshot matrices for each diffusivity value
2. Concatenate and compute POD basis with r modes capturing 99% energy
3. Learn reduced operators A(nu), c(nu) at nu = 0.1, 0.5, 2.0 (heat equation is linear, so H=0, B=0)
4. Fit quadratic polynomial through A entries as function of nu
5. Evaluate A(0.8), set a0 = Phi.T @ sin(pi*x), integrate reduced ODE
6. Reconstruct y = Phi @ a(t)

Output:
```python
import numpy as np
from scipy.integrate import solve_ivp

# --- Offline stage ---
# snapshots: dict {nu: array of shape [N_x, N_t]} from solver
all_snaps = np.hstack([snapshots[nu] for nu in [0.1, 0.5, 2.0]])
U, sigma, _ = np.linalg.svd(all_snaps, full_matrices=False)
energy = np.cumsum(sigma**2) / np.sum(sigma**2)
r = np.searchsorted(energy, 0.99) + 1
Phi = U[:, :r]

operators = {}
for nu in [0.1, 0.5, 2.0]:
    a = Phi.T @ snapshots[nu]          # [r, N_t]
    dadt = Phi.T @ time_derivs[nu]     # [r, N_t]
    # Solve: A @ a = dadt  =>  A = dadt @ a.T @ inv(a @ a.T + lam*I)
    lam = 1e-6
    G = a @ a.T + lam * np.eye(r)
    A_nu = dadt @ a.T @ np.linalg.inv(G)
    operators[nu] = A_nu

# Polynomial fit on each entry of A
nus_train = np.array([0.1, 0.5, 2.0])
A_entries = np.array([operators[nu] for nu in nus_train])  # [3, r, r]
poly_coeffs = np.zeros((r, r, 3))  # degree-2 polynomial
for i in range(r):
    for j in range(r):
        poly_coeffs[i, j] = np.polyfit(nus_train, A_entries[:, i, j], deg=2)

# --- Online stage ---
nu_test = 0.8
A_test = np.zeros((r, r))
for i in range(r):
    for j in range(r):
        A_test[i, j] = np.polyval(poly_coeffs[i, j], nu_test)

x = np.linspace(0, 1, snapshots[0.1].shape[0])
y0 = np.sin(np.pi * x)
a0 = Phi.T @ y0

sol = solve_ivp(lambda t, a: A_test @ a, [0, 1], a0, method='BDF', dense_output=True)
t_eval = np.linspace(0, 1, 201)
a_sol = sol.sol(t_eval)
y_pred = Phi @ a_sol  # [N_x, 201] full solution field
```

**Example 2: Viscous Burgers' Equation at Unseen Viscosity**

User: "Set up an OpInf pipeline for the 1D Burgers equation. Train at nu = 0.01, 0.05, 0.1 and predict at nu = 0.03 with u(x,0) = sin(2*pi*x)."

Approach:
1. Generate training data using Chebyshev collocation with RK45 at each training nu
2. Compute shared POD basis (Burgers is nonlinear, so include quadratic term H)
3. For each nu, solve OpInf with operators A, H, c (no external input B for this setup)
4. Polynomial regression on A, H, c entries
5. Predict at nu = 0.03

Output:
```python
import numpy as np
from scipy.integrate import solve_ivp

def opinf_solve(Phi, a_data, dadt_data, lam=1e-4):
    """Solve OpInf least-squares for A, H, c (quadratic model)."""
    r = Phi.shape[1]
    n_snaps = a_data.shape[1]
    # Build data matrix: [a; a kron a; 1]
    kron_data = np.array([np.kron(a_data[:, k], a_data[:, k]) for k in range(n_snaps)]).T
    D = np.vstack([a_data, kron_data, np.ones((1, n_snaps))]).T  # [n_snaps, r + r^2 + 1]
    R = dadt_data.T  # [n_snaps, r]
    # Tikhonov solve
    O, _, _, _ = np.linalg.lstsq(D.T @ D + lam * np.eye(D.shape[1]), D.T @ R, rcond=None)
    A = O[:r, :].T
    H = O[r:r+r**2, :].T
    c = O[-1, :]
    return A, H, c

# Train at multiple nu values, then polynomial-regress operators for nu=0.03
# Integrate: da/dt = A*a + H*(a kron a) + c
def reduced_rhs(t, a, A, H, c):
    return A @ a + H @ np.kron(a, a) + c

sol = solve_ivp(lambda t, a: reduced_rhs(t, a, A_test, H_test, c_test),
                [0, 1], a0, method='RK45', max_step=1e-3)
y_pred = Phi @ sol.y  # reconstruct full solution
```

**Example 3: 2D Lid-Driven Cavity at New Reynolds Number**

User: "I have vorticity snapshots from a lid-driven cavity simulation at Re = 100, 400, 1000. Predict the flow at Re = 250."

Approach:
1. Reshape 2D vorticity fields to column vectors, concatenate across Re values
2. Compute POD basis (may need r = 15-30 modes for 2D flow)
3. Learn operators at each Re. The vorticity equation has quadratic nonlinearity (advection), so include H
4. Polynomial regression on operators parameterized by Re (or 1/Re for better conditioning)
5. Evaluate at Re = 250, integrate, reshape solution back to 2D grid
6. Plot vorticity contours at selected time snapshots

## Best Practices

- **Do:** Choose the regularization parameter `lambda` by validating on a held-out trajectory or parameter value. Typical range is 1e-8 to 1e-2.
- **Do:** Parameterize operators by `1/Re` or `1/nu` instead of `Re` or `nu` directly when the PDE coefficients scale inversely with the parameter. This yields smoother polynomial fits.
- **Do:** Include multiple trajectories per parameter value with different ICs/BCs to ensure persistent excitation of all POD modes.
- **Do:** Check the singular value decay before choosing `r`. A sharp drop indicates a low-dimensional structure amenable to ROM.
- **Avoid:** Using too many POD modes. Overfitting the reduced model to noise in the snapshots degrades generalization. Start with the fewest modes that capture 99% energy.
- **Avoid:** Extrapolating far outside the training parameter range. Polynomial regression on operators is reliable for interpolation and mild extrapolation, but degrades for distant parameter values.
- **Avoid:** Applying this to PDEs with strong shocks or discontinuities without enrichment. POD bases from smooth solutions will not resolve sharp fronts well.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Reduced ODE blows up | `solve_ivp` returns inf/nan | Increase regularization `lambda`, reduce `r`, or use a stiffer solver (`'BDF'`) |
| High reconstruction error | Relative L2 error > 0.1 | Increase `r`, add more training snapshots, or check that time derivatives are computed accurately |
| Polynomial regression oscillates | Predicted operators have unreasonable magnitudes | Reduce polynomial degree, add more training parameter values, or switch to spline interpolation |
| Ill-conditioned least-squares | Warning from `lstsq` about rank deficiency | Increase regularization, reduce `r`, or add more snapshot data |
| Poor generalization to new BCs | Large error only for new boundary conditions | Ensure training trajectories include diverse BCs; the POD basis must span the relevant solution manifold |

## Limitations

- Assumes the PDE has **at most quadratic nonlinearity** in the state. PDEs with exponential, trigonometric, or other non-polynomial nonlinearities require lifting transformations or are not directly supported.
- The POD basis is **fixed after offline training**. If the solution structure changes dramatically at unseen parameters (e.g., bifurcations, new flow regimes), the basis may be inadequate.
- **1D and 2D problems** with moderate spatial resolution work well. Very high-dimensional 3D problems may require additional techniques (randomized SVD, incremental POD) for the offline stage.
- Polynomial regression on operators works for **smooth parameter dependence**. Discontinuous or highly nonlinear parameter dependence needs more training points or adaptive regression.
- The framework **does not replace** a full-order solver for generating training data. You need at least 3 parameter values with 2-3 trajectories each.

## Reference

**Paper:** [OpInf-LLM: Parametric PDE Solving with LLMs via Operator Inference](https://arxiv.org/abs/2602.01493) -- Wang et al., 2026. Focus on Section 3 (OpInf formulation), Section 4 (LLM integration architecture), and Tables 1-2 (accuracy and success rate benchmarks across heat, Burgers, and cavity flow problems).