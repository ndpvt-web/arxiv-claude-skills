---
name: "llamea-sage-guiding-automated-algorithm"
description: "Guide LLM-driven algorithm generation using AST structural feedback and explainable AI. Extracts graph-theoretic and complexity features from generated code's abstract syntax trees, trains a surrogate model, then uses SHAP explanations to produce natural-language mutation instructions that steer code generation toward higher-performing solutions. Trigger phrases: 'generate optimization algorithm', 'evolve code with structural feedback', 'automated algorithm design with AST analysis', 'improve generated code using SHAP guidance', 'LLM-driven algorithm evolution', 'code-feature-guided mutation'"
---

# LLaMEA-SAGE: Structural AST Feedback for Guided Algorithm Generation

This skill enables Claude to apply the LLaMEA-SAGE methodology — using structural analysis of generated code (via abstract syntax trees) combined with explainable AI to iteratively improve LLM-generated algorithms. Instead of relying solely on fitness/performance scores to guide code generation, Claude extracts measurable code features (cyclomatic complexity, AST depth, node counts, parameter counts), builds a surrogate model mapping features to performance, identifies which features matter most via SHAP, and translates those insights into concrete natural-language instructions for the next generation of code. This closes the loop between code structure and algorithm quality.

## When to Use

- When the user asks Claude to **generate or evolve optimization algorithms** (metaheuristics, search heuristics, black-box optimizers) and wants iterative improvement beyond simple prompt-and-retry
- When the user has a **benchmark function or fitness metric** and wants to automatically explore the space of possible algorithm implementations
- When the user wants to **understand why one generated algorithm outperforms another** by analyzing structural code differences
- When the user asks to **improve generated code using code-level features** rather than just output correctness
- When building an **automated algorithm design pipeline** that uses LLMs as the code generator and needs a principled feedback mechanism
- When the user wants to apply **explainable AI (SHAP) to code generation** — connecting code structure to performance predictions

## Key Technique

**The core insight:** LLM-generated code contains structural signals — measurable in its abstract syntax tree — that correlate with runtime performance. By extracting these features, learning a surrogate model over an archive of evaluated solutions, and using SHAP to identify which features drive performance, we can construct targeted natural-language instructions ("increase cyclomatic complexity", "reduce nesting depth") that guide subsequent code generation. This is more informative than fitness-only feedback because it tells the LLM *what to change structurally*, not just *whether the result was good or bad*.

**How it works:** The method maintains an archive of all generated algorithms paired with their performance scores and AST-derived feature vectors. An XGBoost regression model is trained on this archive to predict performance from code features. SHAP attribution values are computed for each feature — the feature with the largest absolute SHAP value is selected, and its sign determines the direction (increase or decrease). This becomes a mutation instruction injected into the LLM prompt: "Based on archive analysis, try to `<<increase/decrease>>` the `<<feature_name>>` of the solution."

**Why it works:** The approach does not restrict the LLM's expressivity — it adds a soft structural bias. The evolutionary (mu+lambda) selection strategy handles exploitation, while the SHAP-derived instructions provide informed exploration directions. The surrogate model improves as the archive grows, making guidance increasingly precise over generations.

## Step-by-Step Workflow

1. **Define the algorithm template and evaluation function.** Specify the interface the generated algorithm must implement (e.g., `__init__(self, budget, dim)` and `__call__(self, func)`), the search space bounds, and the fitness function that scores each algorithm's output quality (e.g., best objective value found, convergence speed, AOCC metric).

2. **Generate an initial population using the LLM.** Prompt Claude (or another LLM) to produce `mu` distinct algorithm implementations from a base prompt. Each should be a complete, runnable optimization algorithm. Use `mu=4` to `mu=8` as a starting population size.

3. **Evaluate each algorithm and extract AST features.** Run each generated algorithm against the benchmark. Then parse each algorithm's source code into an AST and extract the feature vector:
   - **Graph-theoretic features:** node count, edge count, degree mean/variance/entropy, tree depth (min/mean/max), clustering coefficient, assortativity, diameter, average shortest path length
   - **Complexity features:** cyclomatic complexity (per function and total), token count, parameter count (per function and total)

4. **Store results in the archive.** For each algorithm `s`, store the tuple `(s, fitness, feature_vector)` in a persistent archive `A`. This archive grows monotonically across all generations.

5. **Train the surrogate model.** Once the archive reaches a minimum size (e.g., `>= mu` entries), train an XGBoost gradient-boosted regression tree with squared-error loss to predict fitness from feature vectors: `f_hat(features) ≈ fitness`.

6. **Compute SHAP explanations.** Apply SHAP (TreeExplainer for XGBoost) to the surrogate model. For the current best solution's feature vector, compute attribution values for each feature. Identify feature `k` with the largest absolute SHAP value `|phi_k|`.

7. **Construct the mutation instruction.** Determine the direction from the sign of `phi_k`: positive SHAP means the feature positively correlates with fitness (action = "increase"), negative means "decrease". Format the instruction: `"Based on archive analysis, try to <<increase/decrease>> the <<feature_name>> of the solution."`

8. **Generate offspring with guided prompts.** For each of `lambda` offspring: select a parent from the current population, append the mutation instruction to the standard mutation prompt, and call the LLM to generate a modified algorithm. The LLM receives the parent code, the base task description, and the structural guidance.

9. **Evaluate offspring, update archive and population.** Run each offspring against the benchmark, extract features, add to archive. Apply (mu+lambda) elitist selection — keep the best `mu` individuals from the combined parent + offspring pool.

10. **Iterate until budget exhausted.** Repeat steps 5-9 for each generation until the total evaluation budget is consumed (e.g., 200 algorithm evaluations). Return the best algorithm found in the archive.

## Concrete Examples

**Example 1: Evolving a Black-Box Optimizer**

User: "Generate a Python optimization algorithm for minimizing black-box functions in [-5, 5]^d, then iteratively improve it using structural code feedback."

Approach:
1. Generate initial population of 4 optimizers with the template:
```python
class Algorithm:
    def __init__(self, budget: int, dim: int):
        self.budget = budget
        self.dim = dim
        # ... initialization

    def __call__(self, func) -> tuple:
        # ... optimization loop using self.budget evaluations
        # func(x) evaluates the black-box at point x
        return best_x, best_f
```

2. Evaluate each on a test suite (e.g., Sphere, Rastrigin, Ackley). Compute AOCC scores.

3. Extract AST features for each. Example feature vector:
```
{
  "node_count": 87, "edge_count": 86, "depth_max": 12,
  "depth_mean": 5.3, "degree_mean": 1.97, "degree_entropy": 2.1,
  "cyclomatic_complexity_total": 8, "token_count_total": 234,
  "parameter_count_total": 5, "clustering_coefficient": 0.0
}
```

4. Train XGBoost surrogate on the 4 samples. SHAP identifies `cyclomatic_complexity_total` (phi = +0.15) as most impactful.

5. Mutation instruction appended to prompt:
```
Based on archive analysis, try to increase the total cyclomatic complexity of the solution.
```

6. LLM generates offspring with more conditional branches (adaptive step sizes, multiple search strategies). Evaluate, repeat.

Output after 50 iterations: An algorithm combining CMA-ES-like covariance adaptation with differential evolution crossover, discovered through guided structural evolution.

---

**Example 2: Analyzing Why One Algorithm Outperforms Another**

User: "I have two generated sorting heuristics. Why does version B perform better than version A?"

Approach:
1. Parse both versions into ASTs and extract feature vectors:
```
Version A: cyclomatic_complexity=3, depth_max=4, node_count=45, param_count=2
Version B: cyclomatic_complexity=7, depth_max=8, node_count=112, param_count=4
```

2. If an archive exists, query the surrogate model for SHAP attributions on both:
```
Version A SHAP: cyclomatic_complexity → -0.08 (hurting), depth_max → -0.05
Version B SHAP: cyclomatic_complexity → +0.12 (helping), param_count → +0.06
```

3. Report: "Version B's higher cyclomatic complexity (7 vs 3) is the strongest structural predictor of its better performance — it has more conditional branches enabling adaptive behavior. Its deeper AST (8 vs 4) and additional parameters also contribute positively."

---

**Example 3: Building a Feature Extraction Pipeline**

User: "Write a Python function that extracts LLaMEA-SAGE AST features from source code."

Output:
```python
import ast
import networkx as nx
from collections import Counter
import math

def extract_ast_features(source_code: str) -> dict:
    tree = ast.parse(source_code)

    # Build directed graph from AST
    G = nx.DiGraph()
    node_id = 0
    stack = [(tree, node_id)]
    G.add_node(node_id, type=type(tree).__name__)

    while stack:
        parent_node, parent_id = stack.pop()
        for child in ast.iter_child_nodes(parent_node):
            node_id += 1
            G.add_node(node_id, type=type(child).__name__)
            G.add_edge(parent_id, node_id)
            stack.append((child, node_id))

    U = G.to_undirected()

    # Graph-theoretic features
    degrees = [d for _, d in G.degree()]
    depths = dict(nx.single_source_shortest_path_length(G, 0))
    depth_values = list(depths.values())

    def entropy(values):
        counts = Counter(values)
        total = sum(counts.values())
        return -sum((c/total) * math.log2(c/total) for c in counts.values() if c > 0)

    features = {
        "node_count": G.number_of_nodes(),
        "edge_count": G.number_of_edges(),
        "degree_mean": sum(degrees) / len(degrees) if degrees else 0,
        "degree_variance": (sum((d - sum(degrees)/len(degrees))**2
                           for d in degrees) / len(degrees)) if degrees else 0,
        "degree_entropy": entropy(degrees),
        "depth_min": min(depth_values),
        "depth_mean": sum(depth_values) / len(depth_values),
        "depth_max": max(depth_values),
        "clustering_coefficient": nx.average_clustering(U),
        "diameter": nx.diameter(U) if nx.is_connected(U) else -1,
        "avg_shortest_path": (nx.average_shortest_path_length(U)
                              if nx.is_connected(U) else -1),
    }

    # Complexity features via AST walk
    functions = [n for n in ast.walk(tree) if isinstance(n, (ast.FunctionDef, ast.AsyncFunctionDef))]
    total_cc = 0
    total_params = 0
    for func in functions:
        cc = 1
        for node in ast.walk(func):
            if isinstance(node, (ast.If, ast.While, ast.For, ast.ExceptHandler,
                                 ast.With, ast.Assert, ast.BoolOp)):
                cc += 1
            if isinstance(node, ast.BoolOp):
                cc += len(node.values) - 1
        total_cc += cc
        total_params += len(func.args.args)

    features["cyclomatic_complexity_total"] = total_cc
    features["token_count_total"] = len(list(ast.walk(tree)))
    features["parameter_count_total"] = total_params
    features["function_count"] = len(functions)

    return features
```

## Best Practices

- **Do:** Start with a small initial population (4-8) to establish the archive before relying on surrogate guidance. The surrogate model is unreliable with fewer samples than features.
- **Do:** Use the mutation instruction as a *soft suggestion*, not a hard constraint. Append it to the prompt but let the LLM decide how to interpret "increase cyclomatic complexity" — it might add conditionals, loops, or strategy switches.
- **Do:** Re-train the surrogate model every generation as the archive grows. Early guidance may be noisy; it improves as data accumulates.
- **Do:** Track which features are repeatedly selected by SHAP across generations — persistent features reveal genuine structural drivers of performance for your problem domain.
- **Avoid:** Selecting multiple features simultaneously for mutation instructions. The paper uses only the single highest-impact feature to keep guidance focused and interpretable.
- **Avoid:** Applying this to problems where code structure has no meaningful relationship to output quality (e.g., purely data-driven tasks where the algorithm is trivial but the data matters).
- **Avoid:** Skipping the evaluation step — the surrogate is an approximation. Always evaluate generated algorithms on the actual fitness function; the surrogate only guides which structural direction to explore.

## Error Handling

- **AST parsing failure:** If generated code has syntax errors, catch `SyntaxError` during `ast.parse()`. Assign a penalty fitness score and exclude from the surrogate training set. Do not add malformed code to the archive's feature set.
- **Surrogate model underfitting:** If the archive is too small (< 8 samples) or features are highly collinear, SHAP values may be unreliable. Fall back to unguided LLM mutation until the archive reaches sufficient size.
- **Feature extraction on non-Python code:** The AST approach described here is Python-specific (`ast` module). For other languages, use tree-sitter or language-specific parsers to build equivalent AST graphs, then compute the same graph-theoretic metrics via NetworkX.
- **SHAP direction always the same:** If SHAP consistently recommends "increase" for one feature (common for cyclomatic complexity in optimization tasks), this is expected — it reflects a genuine structural bias for the problem class. Do not force alternation.
- **XGBoost import or training errors:** Ensure `xgboost` and `shap` are installed. Use `shap.TreeExplainer(model)` for XGBoost models specifically, as it provides exact SHAP values rather than approximations.

## Limitations

- **Python-centric:** The AST feature extraction pipeline uses Python's `ast` module. Extending to other languages requires a different parser but the same graph-theoretic approach.
- **Surrogate cold start:** The first few generations operate with little or no structural guidance because the surrogate needs a minimum archive size. Expect the first ~2 generations to behave identically to unguided evolution.
- **Single-feature guidance:** The method selects only one feature per mutation instruction. Complex performance landscapes may benefit from multi-feature guidance, but this is unexplored in the paper.
- **Computational overhead:** AST extraction and SHAP computation add per-generation cost. For problems where each algorithm evaluation is expensive (minutes+), this overhead is negligible. For cheap evaluations, it may be noticeable.
- **Effect size:** The paper reports medium effect size (Cliff's delta = 0.36) over vanilla LLaMEA with confidence intervals that include zero at n=5 runs. The improvement is consistent but modest — this is a refinement technique, not a paradigm shift.
- **Feature-performance correlation assumption:** The method assumes AST-level structural features meaningfully predict algorithmic performance. This holds for optimization heuristics but may not generalize to all code generation domains.

## Reference

**Paper:** [LLaMEA-SAGE: Guiding Automated Algorithm Design with Structural Feedback from Explainable AI](https://arxiv.org/abs/2601.21511v1) — Niki van Stein, Anna V. Kononova, Lars Kotthoff, Thomas Back (GECCO 2026). Focus on Section 3 (method description), Algorithm 1 (pseudocode), and Figures 5-7 (SHAP analysis and convergence curves).