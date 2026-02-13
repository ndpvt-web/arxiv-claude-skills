---
name: "chehab-rl-learning-optimize"
description: "Optimize Fully Homomorphic Encryption code using RL-guided rewriting rules for automatic vectorization, latency reduction, and noise minimization. Use when: 'optimize my FHE program', 'vectorize this homomorphic encryption code', 'reduce noise in my BFV/CKKS computation', 'rewrite FHE scalar ops to SIMD', 'compile encrypted computation efficiently', 'minimize rotation count in FHE circuit'."
---

# CHEHAB RL: RL-Guided Optimization of Fully Homomorphic Encryption Code

This skill enables Claude to apply the CHEHAB RL methodology — a reinforcement-learning-driven rewriting framework — to optimize Fully Homomorphic Encryption (FHE) programs. The core technique transforms scalar FHE operations into vectorized (SIMD-packed) equivalents by applying sequences of algebraic rewriting rules in a learned order, minimizing instruction latency, rotation count, and multiplicative noise growth. Claude uses this to advise on FHE code restructuring, select optimal rewriting sequences, design cost-aware transformation pipelines, and build RL training environments for FHE compilation.

## When to Use

- When the user asks to **vectorize scalar FHE code** into SIMD-packed slot operations for BFV or CKKS schemes.
- When the user needs to **reduce noise growth** (multiplicative depth) in an FHE computation pipeline.
- When the user wants to **minimize rotation operations** in packed ciphertext programs (rotations are expensive key-switching operations).
- When the user is **building or extending an FHE compiler** and needs a learned optimization pass instead of hand-written heuristics.
- When the user wants to **design an RL environment** for program transformation / compiler optimization tasks.
- When the user asks to **analyze the cost** of an FHE circuit and suggest rewriting strategies.
- When the user is **generating diverse FHE training programs** via LLM-based synthesis for compiler benchmarking.

## Key Technique

**Problem:** FHE allows computation on encrypted data, but operations are orders of magnitude slower than plaintext. BFV/CKKS schemes support SIMD batching — packing multiple values into ciphertext slots — but deciding *which* scalar operations to group, *how* to pack them, and *what order* to apply algebraic rewrites is combinatorially explosive. Heuristic compilers like Coyote use integer linear programming (ILP) which is slow and often suboptimal.

**CHEHAB RL's Insight:** Model the rewriting process as a Markov Decision Process. The *state* is the current FHE program (tokenized via Identifier and Constant Invariant canonicalization, then embedded by a 4-layer Transformer encoder). The *action space* consists of 84 rewriting rules (algebraic simplifications, commutativity swaps, common-factor extraction, isomorphic/non-isomorphic vectorization) plus an END action, each parameterized by the sub-expression location where the rule applies. A PPO agent learns to select which rule to apply where, guided by a cost function weighting operation latency (vector add=1, rotation=50, multiply=100, scalar op=250), circuit depth, and multiplicative depth. The reward decomposes into per-step improvement `(C_t - C_{t+1})/C_t` and a terminal bonus for total optimization achieved.

**Why it works better:** The learned policy generalizes across program structures (trained on 15,855 LLM-synthesized expressions), avoids the exponential blowup of ILP/combinatorial search, and discovers rewriting sequences that enable further rewrites — e.g., applying commutativity *before* common-factor extraction unlocks vectorization opportunities that the reverse order misses. This yields 5.3x faster execution, 2.54x less noise, and 27.9x faster compilation versus the state-of-the-art.

## Step-by-Step Workflow

1. **Represent the FHE program as a dataflow DAG.** Parse the input computation into an expression tree where leaf nodes are encrypted variables/constants and internal nodes are operations (add, multiply, rotate, negate). Track slot indices for each ciphertext.

2. **Canonicalize the expression using ICI tokenization.** Replace variable names with positional identifiers (`v0, v1, ...`) and constants with canonical tokens (`c0, c1, ...`, preserving literal 0 and 1). This makes the representation invariant to naming, enabling policy transfer across programs.

3. **Compute the baseline cost.** Evaluate the unoptimized program using the three-component cost model:
   ```
   Cost(e) = w_ops * C_ops(e) + w_depth * D_circuit(e) + w_mult * D_mult(e)
   ```
   Where `C_ops` sums weighted operation costs (scalar ops=250, multiply=100, rotation=50, add/sub=1), `D_circuit` is total circuit depth, and `D_mult` is multiplicative depth (determines noise budget).

4. **Identify applicable rewriting rules at each sub-expression.** Scan the DAG for pattern matches against the 84-rule library. Key rule categories:
   - **Algebraic:** `a*b + a*c => a*(b+c)` (common factor), `a*b => b*a` (commutativity)
   - **Vectorization:** `f(a,b) op f(c,d) => f_vec([a,c], [b,d])` for isomorphic sub-trees
   - **Non-isomorphic vectorization:** Pack structurally different sub-expressions with masking
   - **Rotation optimization:** Merge or eliminate redundant slot rotations
   - **Simplification:** Constant folding, identity elimination, dead code removal

5. **Apply rules in cost-reducing order (the RL policy).** At each step, select the (rule, location) pair that the policy network scores highest. After applying the rule, recompute matched locations since new patterns may have been exposed. Continue until the END action is selected or no cost-reducing rules remain.

6. **Verify noise budget feasibility.** After the rewriting sequence, compute the multiplicative depth of the optimized circuit. Confirm it fits within the encryption parameters' noise budget (`log2(q/t)` for BFV). If not, relax vectorization decisions that increased multiplicative depth.

7. **Apply classical compiler passes.** Run common subexpression elimination (CSE), constant folding, and dead code elimination on the optimized DAG. These are cheap and always beneficial post-rewriting.

8. **Generate rotation keys.** Identify all unique rotation offsets in the final program. Each distinct rotation requires a rotation key (expensive to generate and store). Minimize the set by algebraic rotation merging.

9. **Emit target code.** Lower the optimized DAG to the target FHE library API (e.g., Microsoft SEAL, OpenFHE, or TFHE-rs), generating slot packing, rotation, masking, and key-generation calls.

10. **Benchmark and validate.** Run the optimized program on test vectors to verify functional equivalence with the original scalar version. Measure execution time, noise consumption, and key material size.

## Concrete Examples

**Example 1: Vectorizing a scalar polynomial evaluation**

User: "I have this FHE computation on 4 encrypted inputs: `result = (a*b + c*d) * (e*f + g*h)`. How can I optimize it for BFV with 4096 slots?"

Approach:
1. Parse expression tree: two additions of products, then a final multiply.
2. Identify isomorphic sub-trees: `a*b` and `c*d` are structurally identical, as are `e*f` and `g*h`.
3. Apply vectorization rule to pack: `[a,c] * [b,d] => vec_mul_1`, `[e,g] * [f,h] => vec_mul_2`.
4. Sum within each vector: use rotation + add to reduce `vec_mul_1` into a single slot, same for `vec_mul_2`.
5. Final scalar multiply on the two reduced results.

Output:
```
// Before: 4 scalar multiplies + 2 scalar adds + 1 scalar multiply = 7 ops
// After: 2 vector multiplies + 2 rotations + 2 vector adds + 1 multiply = 7 ops
// BUT vector ops are ~250x cheaper than scalar, and multiplicative depth stays at 2

vec_ab_cd = seal.multiply(pack([a,c]), pack([b,d]))   // 1 vector mul
sum_1 = seal.add(vec_ab_cd, seal.rotate(vec_ab_cd, 1)) // 1 rotate + 1 add
// mask slot 0 to get (a*b + c*d)

vec_ef_gh = seal.multiply(pack([e,g]), pack([f,h]))    // 1 vector mul
sum_2 = seal.add(vec_ef_gh, seal.rotate(vec_ef_gh, 1)) // 1 rotate + 1 add

result = seal.multiply(sum_1, sum_2)                    // 1 mul (depth 2)
```

**Example 2: Designing an RL environment for FHE rewriting**

User: "I want to build a Gymnasium environment that trains a PPO agent to optimize FHE expression trees, following the CHEHAB RL approach."

Approach:
1. Define the observation space as a tokenized expression (ICI-canonicalized), embedded via a Transformer encoder (4 layers, 8 heads, dim=256, CLS token as state vector).
2. Define the action space as `Tuple(Discrete(85), Discrete(max_locations))` — 84 rules + END, times matched locations.
3. Implement the 84 rewriting rules as pattern-match-and-replace on the expression AST.
4. Define step reward as `(cost_before - cost_after) / cost_before` with terminal bonus.
5. Episode ends when END is selected or step limit reached.

Output:
```python
import gymnasium as gym
import numpy as np

class FHERewriteEnv(gym.Env):
    def __init__(self, expressions, max_steps=200):
        super().__init__()
        self.expressions = expressions
        self.max_steps = max_steps
        self.num_rules = 85  # 84 rules + END
        self.max_locations = 64
        self.action_space = gym.spaces.MultiDiscrete([self.num_rules, self.max_locations])
        self.observation_space = gym.spaces.Box(-np.inf, np.inf, shape=(256,))

    def reset(self, seed=None, **kwargs):
        self.expr = self._sample_expression()
        self.initial_cost = self._compute_cost(self.expr)
        self.current_cost = self.initial_cost
        self.step_count = 0
        return self._encode(self.expr), {}

    def step(self, action):
        rule_id, loc_id = action
        if rule_id == 84:  # END action
            terminal_reward = (self.initial_cost - self.current_cost) / self.initial_cost * 100
            return self._encode(self.expr), terminal_reward, True, False, {}

        matches = self._find_matches(self.expr, rule_id)
        if loc_id < len(matches):
            new_expr = self._apply_rule(self.expr, rule_id, matches[loc_id])
            new_cost = self._compute_cost(new_expr)
            reward = (self.current_cost - new_cost) / self.current_cost
            self.expr = new_expr
            self.current_cost = new_cost
        else:
            reward = -0.01  # invalid location penalty

        self.step_count += 1
        done = self.step_count >= self.max_steps
        return self._encode(self.expr), reward, done, False, {}

    def _compute_cost(self, expr):
        """Cost = w_ops*C_ops + w_depth*D_circuit + w_mult*D_mult"""
        ops_cost = sum(
            {'add': 1, 'sub': 1, 'mul': 100, 'rot': 50, 'scalar': 250}[op.type]
            for op in expr.operations
        )
        return ops_cost + 10 * expr.circuit_depth + 50 * expr.mult_depth
```

**Example 3: LLM-based FHE training data synthesis**

User: "How do I generate diverse FHE expressions to train my rewriting agent?"

Approach:
1. Design a prompt specifying the CHEHAB IR syntax (scalar ops: `+, *, -`; vector ops: `vadd, vmul, rot`; variables: `v0..vN`; constants: `c0..cN, 0, 1`).
2. Include 3-4 real-world kernel examples (matrix multiply, squared difference, polynomial eval) as few-shot demonstrations.
3. Ask the LLM to generate varied expressions of different sizes (5-50 operations) and structures.
4. Post-process: parse-validate, ICI-canonicalize, deduplicate, and exclude any benchmark programs.

Output prompt template:
```
Generate 20 diverse arithmetic expressions over encrypted variables v0-v9
using operations: +, *, - (scalar) and constants 0, 1.

Syntax: binary ops are prefix: *(v0, v1), +(*(v0,v1), *(v2,v3))

Examples of real FHE workloads:
- Union cardinality: +(v0, +(v1, -(0, *(v0, v1))))
- Squared difference: *(+(v0, -(0, v1)), +(v0, -(0, v1)))
- Dot product 2D: +(*(v0,v2), *(v1,v3))

Requirements:
- Vary expression depth from 3 to 8
- Mix additions, multiplications, and subtractions
- Some expressions should have repeated sub-expressions (CSE opportunities)
- Some should have isomorphic sub-trees (vectorization opportunities)
```

## Best Practices

- **Do:** Order rewrites so that enabling rules (commutativity, associativity) come before consuming rules (vectorization, common-factor). The sequence `commutativity -> common_factor -> vectorize` is often productive; the reverse blocks opportunities.
- **Do:** Weight multiplicative depth heavily in the cost function. In FHE, each multiplication roughly squares the noise, so reducing mult-depth by 1 can halve the required ciphertext modulus size.
- **Do:** Use ICI canonicalization before any comparison or deduplication of expressions. Without it, `a*b + c*d` and `x*y + z*w` appear different but are structurally identical.
- **Do:** Generate training data that includes both "easy" (already partially vectorized) and "hard" (deeply nested scalar) programs to avoid policy overfitting.
- **Avoid:** Applying vectorization rules that increase multiplicative depth. Packing two sub-expressions of different depths forces the shallower one to "wait," wasting noise budget.
- **Avoid:** Over-rotating. Each distinct rotation offset requires a rotation key (~32MB for BFV-128). Prefer packing layouts that minimize unique rotation distances.

## Error Handling

- **Rule application fails (no match):** If a selected rule has no valid match at the chosen location, return a small negative reward and skip the step. Do not terminate the episode — the agent learns to avoid invalid selections.
- **Noise budget exceeded:** After optimization, if the multiplicative depth exceeds the parameter set's capacity, backtrack the last vectorization rule that increased depth, or suggest the user increase the polynomial modulus degree.
- **Expression too large for Transformer encoder:** Truncate to the maximum token length (typically 512-1024 tokens). For very large programs, decompose into sub-DAGs, optimize each independently, then compose.
- **No cost improvement after full episode:** The expression may already be near-optimal. Report the baseline cost and confirm no further rewrites are beneficial. Consider that the training distribution may not cover this program structure — flag for inclusion in future training data.
- **Functionally incorrect rewrite:** Always verify equivalence by evaluating the original and rewritten expressions on random plaintext inputs before accepting the transformation.

## Limitations

- The 84-rule library targets **BFV-style batching** (integer slot packing via CRT). CKKS (approximate arithmetic) and TFHE (bitwise bootstrapping) have different cost profiles and may need different rule sets.
- The learned policy is trained on expressions up to moderate size (~50 operations). For very large FHE programs (hundreds of operations), the Transformer encoder may lose fine-grained structural information. Hierarchical decomposition is needed.
- **Rotation cost depends on hardware.** The relative weights (rotation=50, multiply=100) are empirical for CPU-based SEAL. GPU-accelerated backends (e.g., cuFHE) shift these ratios significantly.
- The approach assumes **fully combinational circuits** (no control flow, no bootstrapping decisions). Programs requiring conditional bootstrapping or branching need a different formulation.
- Training requires ~43 hours on a single CPU for 2M timesteps. Retraining for a new FHE scheme or target library is non-trivial, though fine-tuning from an existing policy is faster.

## Reference

**Paper:** [CHEHAB RL: Learning to Optimize Fully Homomorphic Encryption Computations](https://arxiv.org/abs/2601.19367v1) — Sefsaf et al., 2026. Focus on Section 3 (RL formulation and rewriting rules), Section 4 (LLM-based data synthesis), and Table 1 (benchmark results vs. Coyote) for the core actionable insights.