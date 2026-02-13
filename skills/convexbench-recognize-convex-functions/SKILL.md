---
name: "convexbench-recognize-convex-functions"
description: "Determine the convexity of arbitrarily deep symbolic function compositions using AST decomposition and recursive DCP-rule reasoning. Use when the user asks to 'check if this function is convex', 'analyze convexity of a composed expression', 'verify DCP compliance', 'classify a nested objective as convex/concave/neither', 'decompose a composite function for convexity analysis', or 'apply disciplined convex programming rules'."
---

# Recognizing Convexity in Deeply Composed Functions via Divide-and-Conquer AST Reasoning

This skill enables Claude to determine whether a symbolic mathematical expression -- potentially involving dozens or hundreds of nested function compositions -- is convex, concave, affine, or neither. It applies the agentic divide-and-conquer framework from the ConvexBench paper: first parsing the expression into an abstract syntax tree (AST) to isolate each sub-expression, then reasoning recursively about convexity at each node using Disciplined Convex Programming (DCP) composition rules with focused dependency context. This approach eliminates the two failure modes that cause LLMs to fail on deep compositions: parsing failure (losing track of operator scope in long expressions) and lazy reasoning (applying shallow heuristics instead of systematic verification).

## When to Use

- When the user provides a symbolic mathematical expression and asks whether it is convex, concave, or neither
- When verifying that an optimization objective satisfies DCP rules before passing it to a solver like CVXPY, CVX, or Convex.jl
- When the user has a deeply nested function composition (e.g., `exp(log(sum(square(Ax - b))))`) and needs convexity classification
- When debugging why a convex optimization solver rejects an expression as non-DCP-compliant
- When the user asks to decompose a complex objective function into sub-expressions and trace convexity through the composition chain
- When building or validating a custom atom library for a disciplined convex programming framework

## Key Technique

**The core problem.** Determining convexity of a single function like `exp(x)` or `x^2` is straightforward. But real optimization objectives chain many operations together -- `log(1 + exp(-y * (w'x + b)))` -- and convexity of the whole depends on how each layer's convexity and monotonicity interact with the layers below it. LLMs fail at this for deep compositions because they lose track of parenthetical scope (parsing failure) and skip rigorous step-by-step verification (lazy reasoning).

**The divide-and-conquer solution.** Instead of asking for a single-shot convexity judgment, the framework (1) deterministically parses the expression into an AST, introducing named intermediate variables for each sub-expression, and (2) reasons about each sub-expression individually, carrying forward only the direct dependency context (the convexity class, monotonicity, and range of immediate parent sub-expressions). Each sub-expression `g_i` is classified using only `{(g_j, sigma_j) | j in Parents(i)}` -- not the entire expression. This focused-context approach prevents attention dilution and forces systematic rule application.

**DCP composition rules.** The classification at each node uses standard disciplined convex programming rules: a nondecreasing convex function of a convex argument is convex; a nonincreasing convex function of a concave argument is convex; a sum of convex functions is convex; an affine function composed with anything preserves the inner function's convexity class; and so on. When no rule applies, the composition is classified as "neither." These rules are sound (never produce false positives) though incomplete (some functions that are technically convex cannot be verified via DCP rules alone).

## Step-by-Step Workflow

1. **Receive and normalize the expression.** Take the user's symbolic expression and normalize notation: expand shorthand (`||x||^2` to `sum(square(x))`), clarify variable vs. parameter roles, and confirm the domain.

2. **Build the abstract syntax tree.** Parse the expression into a tree where each internal node is an operation (e.g., `exp`, `+`, `log`, `max`, scalar multiplication) and each leaf is a variable or constant. Assign a named intermediate variable to each internal node (e.g., `t1 = Ax - b`, `t2 = square(t1)`, `t3 = sum(t2)`).

3. **Flatten to a dependency-ordered sub-expression list.** Topologically sort the AST nodes so that every sub-expression appears after its dependencies. Produce a sequence `G = [g_1, g_2, ..., g_k]` where `g_1` involves only leaf variables and `g_k` is the root expression.

4. **Classify leaf atoms.** For each leaf-level sub-expression, assign its convexity class (convex, concave, affine, neither), monotonicity (nondecreasing, nonincreasing, non-monotone), and range (e.g., nonneg, pos, real). Variables are affine and nondecreasing. Constants are affine.

5. **Apply DCP rules bottom-up at each node.** For each sub-expression `g_i` in order, retrieve only the properties of its direct dependencies `{(g_j, sigma_j) | j in Parents(i)}` and apply the matching DCP composition rule:
   - `f(convex)` where `f` is nondecreasing and convex -> convex
   - `f(concave)` where `f` is nonincreasing and convex -> convex
   - `f(affine)` where `f` is convex -> convex (and analogously for concave)
   - `sum(convex, convex)` -> convex; `sum(concave, concave)` -> concave
   - `alpha * convex` -> convex if `alpha >= 0`, concave if `alpha < 0`
   - `max(convex, convex)` -> convex; `min(concave, concave)` -> concave
   - If no rule matches -> "neither"

6. **Propagate range information.** Track the output range of each sub-expression (nonneg, nonpos, pos, neg, real) since some rules require range constraints (e.g., `log` requires positive input, `sqrt` requires nonneg input).

7. **Report the root node classification.** The convexity of `g_k` (the root) is the convexity of the overall expression.

8. **Present the full reasoning trace.** Show each sub-expression, its dependencies, the rule applied, and the resulting classification. This lets the user verify every step and identify exactly where a non-DCP composition breaks the chain.

9. **If the result is "neither," attempt diagnosis.** Identify the specific sub-expression where DCP compliance breaks. Suggest equivalent reformulations that may be DCP-compliant (e.g., replacing `square(abs(x))` with `square(x)`, or using `quad_over_lin` instead of `square(x) / y`).

## Concrete Examples

**Example 1: Logistic regression loss**

User: "Is `log(1 + exp(-y * (w'x + b)))` convex in `(w, b)` where `y` and `x` are parameters?"

Approach:
1. Parse into sub-expressions:
   - `t1 = w'x + b` (affine in w, b)
   - `t2 = y * t1` (affine -- scalar * affine, y is a parameter)
   - `t3 = -t2` (affine -- negation of affine)
   - `t4 = exp(t3)` (exp is convex and nondecreasing; applied to affine -> convex)
   - `t5 = 1 + t4` (sum of constant and convex -> convex)
   - `t6 = log(t5)` (log is concave and nondecreasing; applied to convex -> ?)

2. At `t6`: `log` is concave nondecreasing, `t5` is convex. DCP rule: concave nondecreasing of convex does NOT match any standard DCP rule. But `log(1 + exp(u))` is the log-sum-exp of `(0, u)`, which is convex by the LSE rule.

3. Re-parse using `log_sum_exp` atom: `t6 = log_sum_exp(0, t3)`. LSE is convex nondecreasing in each argument; each argument is affine -> convex.

Output:
```
Expression: log(1 + exp(-y * (w'x + b)))

Decomposition:
  t1 = w'x + b         -> affine       (linear in w, b)
  t2 = -y * t1         -> affine       (scalar * affine)
  t3 = logsumexp(0, t2)-> convex       (LSE of affine args)

Result: CONVEX in (w, b).

Note: Direct composition log(1 + exp(u)) is not DCP-compliant
as written, but is equivalent to logsumexp(0, u) which is a
recognized convex atom. Solvers like CVXPY accept log_sum_exp.
```

**Example 2: Deep nested composition**

User: "Check convexity of `exp(square(abs(3*x + 2)))` over all real x."

Approach:
1. Decompose:
   - `t1 = 3*x + 2` (affine, nondecreasing, range: real)
   - `t2 = abs(t1)` (convex, non-monotone, range: nonneg)
   - `t3 = square(t2)` (square is convex; on nonneg domain it is nondecreasing; applied to convex nonneg -> convex, range: nonneg)
   - `t4 = exp(t3)` (exp is convex nondecreasing; applied to convex -> convex)

2. Apply rules at each step, carrying forward range info.

Output:
```
Decomposition:
  t1 = 3*x + 2     -> affine         [range: real]
  t2 = abs(t1)      -> convex         [abs is convex; range: nonneg]
  t3 = square(t2)   -> convex         [square is convex nondecreasing on nonneg; convex of convex nonneg -> convex; range: nonneg]
  t4 = exp(t3)      -> convex         [exp is convex nondecreasing; convex of convex -> convex]

Result: CONVEX.
```

**Example 3: Non-convex expression**

User: "Is `sin(x^2)` convex?"

Approach:
1. Decompose:
   - `t1 = x^2` (convex, range: nonneg)
   - `t2 = sin(t1)` (sin is neither convex nor concave; no DCP rule applies)

Output:
```
Decomposition:
  t1 = x^2      -> convex   [range: nonneg]
  t2 = sin(t1)  -> NEITHER   [sin is not convex/concave/affine; no DCP rule applies]

Result: NEITHER (not DCP-verifiable).

Diagnosis: sin() is not a recognized DCP atom. The composition
sin(convex) cannot be classified by DCP rules. Numerically,
sin(x^2) is indeed non-convex (e.g., compare midpoint values
around x=1 and x=2).
```

## Best Practices

- **Do:** Always decompose into explicit named sub-expressions before reasoning. Never attempt to classify a multi-layer composition in one mental step.
- **Do:** Track monotonicity AND range at every node -- range constraints are critical for rules like "square is nondecreasing on nonneg" vs. "square is non-monotone on reals."
- **Do:** When an expression fails DCP verification, suggest equivalent reformulations. Many practical objectives have DCP-compliant rewritings (e.g., `norm(x)^2` -> `sum_squares(x)`, `x*x/y` -> `quad_over_lin(x, y)`).
- **Do:** Verify that the variable/parameter distinction is correct before analysis. `x'Ax` is convex in `x` if `A` is PSD but is generally neither convex nor concave if `A` is also a variable.
- **Avoid:** Classifying the entire expression in one shot. This is the "lazy reasoning" failure mode identified in the paper -- it works for depth 2 but collapses at depth 5+.
- **Avoid:** Carrying the full expression text into each reasoning step. Use only the direct dependency context `{(g_j, sigma_j) | j in Parents(i)}` to prevent attention dilution on irrelevant sub-expressions.

## Error Handling

- **Ambiguous variable roles:** If it is unclear which symbols are optimization variables vs. fixed parameters, ask the user before proceeding. Convexity is always relative to the variables being optimized.
- **Unknown atoms:** If the expression contains a function not in the standard atom library (e.g., a user-defined function), ask the user for its convexity class, monotonicity, and domain. Do not guess.
- **Domain violations:** If a sub-expression's range does not satisfy the domain requirement of the outer function (e.g., `log` applied to something that can be negative), flag this as a domain error rather than a convexity classification.
- **False "neither" results:** DCP rules are sufficient but not necessary. If the analysis yields "neither," note that the function might still be convex -- it just cannot be verified using DCP composition rules. Suggest the user verify numerically or analytically if needed.

## Limitations

- **DCP incompleteness:** Some convex functions cannot be expressed in DCP form. For example, `log(sum(exp(x_i)))` is convex but naive decomposition into `sum` then `log` breaks DCP. This requires recognizing `log_sum_exp` as a single atom. The approach depends on the atom library being sufficiently rich.
- **Matrix-valued expressions:** The DCP rules described here apply to scalar or elementwise operations. Semidefinite constraints, matrix norms, and other matrix convexity properties require extended DCP (e.g., SDP-representable atoms) which is beyond this workflow.
- **Non-symbolic inputs:** This technique applies only to closed-form symbolic expressions. It cannot classify convexity of functions defined by iterative procedures, simulations, or black-box oracles.
- **Multivariate composition nuance:** For vector-valued intermediate expressions composed with non-separable outer functions, the standard scalar DCP rules must be extended to vector composition rules (convex + nondecreasing in each argument + each argument convex). The workflow handles this but the user should verify vector composition steps carefully.

## Reference

**ConvexBench: Can LLMs Recognize Convex Functions?** -- Liu, Huang, Wang, Liang, Bu (2026). [arXiv:2602.01075v2](https://arxiv.org/abs/2602.01075v2). Focus on Section 4 (agentic divide-and-conquer framework), Table 2 (DCP composition rules), and Algorithm 1 (recursive reasoning with focused context).