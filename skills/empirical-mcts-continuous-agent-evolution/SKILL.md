---
name: "empirical-mcts-continuous-agent-evolution"
description: "Applies Empirical-MCTS dual-loop reasoning: structured tree search with persistent memory that accumulates experience across problems. Use when asked to 'solve this step by step with learning', 'use MCTS reasoning', 'try multiple approaches and remember what works', 'evolve your strategy as you go', 'search for the best solution systematically', or 'use tree search with memory'."
---

# Empirical-MCTS: Continuous Agent Evolution via Dual-Experience Search

This skill enables Claude to apply the Empirical-MCTS framework from Lu et al. (2026) to complex reasoning and coding tasks. Instead of treating each problem attempt as stateless, Claude maintains a **dual-loop architecture**: an inner loop that explores solution candidates via tree search with dynamically evolving evaluation criteria, and an outer loop that distills successful reasoning patterns into a persistent memory store. This transforms brute-force retry strategies into a disciplined search that gets smarter with each iteration, accumulating empirical wisdom across related problems.

## When to Use

- When the user asks to solve a hard algorithmic, mathematical, or logic problem where a single attempt is unlikely to succeed
- When tackling a series of related coding challenges (e.g., a test suite with multiple failing tests) where lessons from one fix should inform the next
- When the user wants Claude to "try multiple approaches" and systematically compare them rather than picking one heuristically
- When debugging a complex issue where the root cause is unclear and structured exploration of hypotheses is needed
- When solving puzzle-like tasks (e.g., ARC-style pattern recognition, constraint satisfaction) that benefit from iterative refinement
- When the user explicitly asks for "MCTS reasoning", "tree search", or "evolve your approach"

## Key Technique

**Empirical-MCTS** replaces stateless LLM inference with a dual-loop search process. The **inner loop** is a Monte Carlo Tree Search where each node represents a candidate solution or reasoning state. Nodes are selected via UCB (Upper Confidence Bound), expanded by generating new candidate solutions, and scored using **Pairwise-Experience-Evolutionary Meta-Prompting (PE-EMP)**. PE-EMP does not simply rate a solution in isolation -- it compares two candidates head-to-head, dynamically generates evaluation criteria specific to the problem context, assigns weighted scores across those criteria, and distills the comparison into actionable insights and an evolved meta-prompt. This means the search criteria themselves adapt as the search progresses, unlike fixed heuristics.

The **outer loop** maintains a **global experience memory** -- a structured repository of distilled insights tagged by task type. After each problem (or search iteration), a Memory Optimization Agent performs atomic operations on this store: **Add** new high-quality insights, **Modify** existing ones with richer understanding, **Merge** fragmented observations into cohesive principles, and **Delete** obsolete or low-quality entries. Retrieved experiences serve as a "policy prior" for subsequent problems, biasing the search toward strategies that have worked before while preserving the ability to explore novel approaches.

The critical insight is that neither loop alone is sufficient. Stateless MCTS discards hard-won reasoning patterns. Pure experience replay without structured search drifts toward confirmation bias. The coupling of disciplined exploration with empirical accumulation is what produces the 10-20 percentage point gains observed on benchmarks like AIME25 (73.3% vs 63.3% for LLaMA-Berry) and MathArena Apex.

## Step-by-Step Workflow

### 1. Initialize the search tree and experience memory
Create a root node containing the problem statement and an initial system prompt (generic or domain-appropriate). Initialize an empty experience memory (or load prior experiences if solving a series of related problems). Set search parameters: rollout budget (T=4-8 for most tasks), retrieval count (k=3-5 prior experiences), and decay factor (gamma=0.7 for backpropagation).

### 2. Retrieve relevant prior experiences
Classify the problem by task type (e.g., "graph algorithm", "string parsing", "geometry proof"). Query the experience memory for the top-k most semantically relevant insights. These become the `experience_prior` that biases the current search.

### 3. Select the most promising node via UCB
Apply Upper Confidence Bound to balance exploitation (high-scoring nodes) with exploration (under-visited nodes). For the first iteration, this is trivially the root.

### 4. Expand by generating a candidate solution
Using the current node's meta-prompt, the retrieved experiences, and the problem statement, generate a new candidate solution. This is a full attempt at solving the problem -- not a partial step.

### 5. Score via PE-EMP pairwise comparison
Compare the new candidate against the current best (or baseline) using the 7-stage PE-EMP process:
  - **Generate criteria**: Formulate 3-5 evaluation dimensions specific to this problem (e.g., "correctness of edge case handling", "time complexity", "code readability")
  - **Comparative analysis**: Chain-of-thought comparison of candidate vs. baseline on each criterion
  - **Weight and score**: Assign percentage weights to criteria, compute weighted scores on [0, 10]
  - **Synthesize insights**: Merge comparison feedback with prior experiences into new empirical insight
  - **Evolve the meta-prompt**: Extract critical success factors and failure lessons to produce a refined system prompt for the next expansion

### 6. Backpropagate scores through the tree
Update Q-values from the scored child node back to the root using decay: `Q(parent) = (1 - gamma) * Q(parent) + gamma * Q(child)`. This lets strong insights propagate efficiently.

### 7. Repeat for T rollouts
Loop steps 3-6 for the remaining rollout budget. Each iteration uses the evolved meta-prompt from the previous PE-EMP cycle, so the search narrows toward high-quality solutions while the criteria themselves sharpen.

### 8. Select the best solution
After all rollouts, select the node with the highest Q-value as the final answer.

### 9. Update global experience memory
Feed the best insights from this problem's search into the Memory Optimization Agent. Execute atomic operations:
  - **Add**: Novel insights not already in memory (e.g., "For problems with overlapping subproblems, check if memoization keys include all state dimensions")
  - **Modify**: Refine existing insights with new evidence (e.g., update a heuristic's applicability scope)
  - **Merge**: Combine redundant or fragmented insights into a single principle
  - **Delete**: Remove insights that led to poor outcomes in this search

### 10. Return the solution and persist memory
Present the best solution to the user. Store the updated experience memory for use in subsequent problems.

## Concrete Examples

**Example 1: Debugging a series of failing tests**

```
User: I have 5 failing tests in my payment processing module. Help me fix them systematically.

Approach:
1. Initialize memory store (empty). Classify task: "unit test debugging / payment logic."
2. For Test 1, generate an initial fix attempt (Rollout 1).
3. PE-EMP: Compare fix against the failing state. Criteria generated:
   - "Correctness of currency rounding" (weight: 40%)
   - "Handling of null/zero amounts" (weight: 35%)
   - "Consistency with existing payment model" (weight: 25%)
4. Rollout 2: Evolved meta-prompt now includes "verify rounding to 2 decimal
   places before comparing amounts." Generate improved fix. Score higher.
5. After 4 rollouts, best fix passes Test 1. Distill insight to memory:
   ADD: "Payment amount comparisons require epsilon-based floating point
   comparison or integer cent conversion. Direct equality fails."
6. For Test 2, retrieve the insight from Test 1 as experience_prior.
   First rollout already avoids the rounding trap. Search converges faster.
7. Continue through Tests 3-5, accumulating insights. By Test 5, the memory
   contains 4 distilled principles that make the fix nearly immediate.

Output:
- 5 targeted code fixes, each informed by lessons from previous fixes
- A distilled set of payment-module heuristics for future debugging
```

**Example 2: Solving a complex algorithmic problem**

```
User: Find the minimum number of operations to transform string A into string B,
where allowed operations are: insert, delete, substitute, and transpose adjacent.

Approach:
1. Classify: "string algorithm / edit distance variant (Damerau-Levenshtein)."
   No prior experiences. Initialize with generic meta-prompt.
2. Rollout 1: Generate a standard edit distance DP solution.
3. PE-EMP scores against baseline (brute force). Criteria:
   - "Handles transposition correctly" (45%) -- candidate scores 3/10
     (missing transpose case)
   - "Time complexity" (30%) -- scores 8/10
   - "Edge cases (empty strings, single char)" (25%) -- scores 7/10
4. Evolved meta-prompt: "Ensure the recurrence relation includes a
   d[i-2][j-2] + cost term for adjacent transpositions when s[i]==t[j-1]
   and s[i-1]==t[j]."
5. Rollout 2: Generates correct Damerau-Levenshtein with transpose handling.
   PE-EMP scores: transpose 9/10, complexity 8/10, edge cases 6/10.
   Insight: "Off-by-one in boundary check for i<2 or j<2."
6. Rollout 3: Fixes boundary. All criteria score 9+/10.
7. Memory update:
   ADD: "Damerau-Levenshtein requires a separate check for i>=2 and j>=2
   before accessing d[i-2][j-2]. Initialize boundary with INF sentinel."

Output:
- Correct Damerau-Levenshtein implementation with O(nm) time and space
- Documented reasoning trace showing how each rollout improved on the last
```

**Example 3: Iterative prompt engineering for a code generation task**

```
User: Write a function that parses arbitrary nested JSON-like config files
with comments, environment variable interpolation, and include directives.

Approach:
1. Classify: "parser implementation / recursive descent with extensions."
2. Rollout 1: Generate a basic recursive JSON parser.
3. PE-EMP comparison against requirements. Criteria:
   - "Comment stripping (// and /* */)" (25%) -- 0/10, not implemented
   - "Env var interpolation ${VAR}" (25%) -- 0/10, not implemented
   - "Include directive handling" (25%) -- 0/10, not implemented
   - "Core JSON correctness" (25%) -- 7/10
4. Evolved meta-prompt: "Build a two-pass parser: first pass resolves
   includes and strips comments, second pass handles interpolation and
   JSON parsing. Use a tokenizer to avoid breaking strings."
5. Rollout 2: Implements two-pass approach. Scores improve to 7/10 across
   all criteria. Insight: "Nested includes can cause circular references."
6. Rollout 3: Adds include cycle detection via visited-path set.
   Evolved meta-prompt adds: "Track interpolation depth to prevent
   infinite ${VAR} -> ${VAR} recursion."
7. Rollout 4: Final solution handles all cases. Memory distilled:
   ADD: "Config parsers need cycle detection for both includes and
   variable interpolation. Maintain a visited set for each."
   ADD: "Two-pass architecture (structural then semantic) prevents
   comment/string ambiguity in config formats."

Output:
- Production-quality config parser with all requested features
- Clear separation of concerns across parsing phases
```

## Best Practices

**Do:**
- Start with a concrete, testable baseline before launching the search. The first rollout should be a genuine attempt, not a placeholder.
- Generate evaluation criteria dynamically for each problem. Generic criteria like "correctness" are too coarse -- prefer "handles the empty-input edge case" or "avoids quadratic blowup on sorted input."
- Keep the experience memory concise and actionable. Each insight should be a specific, reusable principle, not a vague observation. "Use sentinel values for DP boundary conditions" beats "be careful with boundaries."
- Tag experiences by task taxonomy so retrieval is precise. A geometry insight should not pollute a string-parsing search.
- Limit rollouts to 4-8 for most tasks. Diminishing returns set in quickly; the value comes from evolved meta-prompts, not exhaustive search.

**Avoid:**
- Do not skip the pairwise comparison step. Rating a solution in isolation produces poorly calibrated scores. Always compare against a baseline or the current best.
- Do not let the experience memory grow unbounded. Aggressively merge and delete. A memory of 20 sharp principles outperforms 200 fuzzy observations.
- Do not apply this framework to trivial tasks. If the problem has an obvious single-step solution, MCTS overhead adds no value. Reserve it for tasks where exploration genuinely helps.
- Do not treat the evolved meta-prompt as a fixed instruction. It should change every rollout -- if it stabilizes too early, the criteria may be too narrow.

## Error Handling

- **Search stagnation (all rollouts score similarly):** The criteria may be too coarse. Force PE-EMP to generate at least one new criterion targeting the weakest aspect of the current best solution. Alternatively, increase exploration weight in UCB.
- **Memory poisoning (bad insight propagated):** If a retrieved experience leads to worse solutions, flag it with a negative signal and run a Delete operation. Check if the task classification was incorrect (e.g., a graph problem misclassified as string processing).
- **Rollout budget exhaustion without convergence:** Return the best solution found so far with an explicit note about remaining weaknesses identified by PE-EMP. Suggest the user provide additional constraints or examples to narrow the search.
- **Circular meta-prompt evolution:** If the evolved prompt oscillates between two strategies, this signals a genuine trade-off. Surface both alternatives to the user with their respective PE-EMP scores and let them choose.

## Limitations

- **Token cost:** Each rollout involves generating a full solution plus a multi-step PE-EMP evaluation. A 6-rollout search on a complex problem may use 10-20x the tokens of a single attempt. Use this framework selectively.
- **Not suited for factual recall:** MCTS explores solution strategies, not knowledge. If the bottleneck is missing information (e.g., an API signature), search won't help -- retrieval will.
- **Single-session memory:** In a standard Claude conversation, the experience memory exists only within the current session. For cross-session persistence, the user must externalize the memory (e.g., store insights in a file that gets loaded in future sessions).
- **Subjective evaluation:** PE-EMP relies on LLM self-evaluation, which can have systematic blind spots. For tasks with objective test cases (unit tests, type checking), prefer running actual tests as the scoring signal rather than relying purely on pairwise LLM judgment.
- **Diminishing returns on well-understood problems:** If the first attempt is already near-optimal (e.g., standard library usage), additional rollouts add cost without benefit.

## Reference

Lu, H., Huang, H., Zhou, Y., Li, C., & Zhu, N. (2026). *Empirical-MCTS: Continuous Agent Evolution via Dual-Experience Monte Carlo Tree Search.* arXiv:2602.04248v1. [https://arxiv.org/abs/2602.04248v1](https://arxiv.org/abs/2602.04248v1)

Key sections to study: Algorithm 1 (full search loop), Section on PE-EMP's 7-stage cognitive process, and the Memory Optimization Agent's atomic operations (Add/Modify/Merge/Delete). The ablation study confirms both components are essential -- removing either PE-EMP or memory drops AIME25 performance from 76.7% to 56.7%.