---
name: "funprm-function-as-step-process-reward"
description: "Generate high-quality code by decomposing solutions into modular functions (Chain-of-Function style), then self-evaluating each function as a discrete reasoning step to select the best candidate. Triggers: 'solve this coding problem', 'generate modular code', 'write clean functions for this algorithm', 'best-of-N code generation', 'help me with this competitive programming problem', 'generate multiple solution candidates'"
---

This skill teaches Claude to apply the FunPRM (Function-as-Step Process Reward Model) technique from research by Zhang et al. (2026). Instead of writing monolithic code blocks, Claude decomposes every solution into well-named helper functions, treats each function as a scoreable reasoning step, generates multiple candidate solutions, and selects the best one by evaluating per-function correctness. This produces code that is more likely correct, more readable, and more reusable -- particularly on hard algorithmic and competitive programming problems.

## When to Use

- When the user asks you to solve a non-trivial coding problem (LeetCode, competitive programming, algorithmic challenges)
- When the user requests multiple solution candidates and wants you to pick the best one
- When the user explicitly asks for modular, clean, well-structured code for a complex task
- When implementing an algorithm that naturally decomposes into sub-problems (graph traversal + scoring + selection, parsing + transformation + validation)
- When a first attempt at a coding problem failed and the user wants a more systematic retry
- When the user asks for production-quality code that is easy to test, debug, and extend

## Key Technique: Chain-of-Function Decomposition with Per-Function Evaluation

**Why functions as steps?** Traditional process reward models for math treat each line of reasoning as a step. This fails for code because individual lines are not semantically meaningful units. FunPRM's insight is that *functions* are the natural reasoning steps in code: each one encapsulates a distinct sub-problem, has a clear interface (inputs/outputs), and can be evaluated for correctness independently. A solution decomposed into 3-6 well-named functions is far easier to score than a 50-line monolithic block.

**The Chain-of-Function prompting pattern** instructs the LLM to: (1) place the high-level solving logic in a main entry-point function, (2) extract repeated or complex logic into dedicated helper functions at the top level (no nesting), and (3) add docstrings to every function explaining its purpose and algorithmic approach. This mirrors how experienced developers naturally structure code -- and it gives the self-evaluation pass clear units to reason about.

**Self-evaluation via per-function scoring:** After generating N candidate solutions (typically 3-8), evaluate each by mentally tracing through every function: Does it handle edge cases? Is its logic consistent with its docstring? Does the main function compose helpers correctly? Aggregate per-function confidence into a solution-level score and select the highest-scoring candidate. This is the core Best-of-N selection mechanism, adapted for a single-model workflow without a separate trained reward model.

## Step-by-Step Workflow

1. **Analyze the problem requirements.** Read the problem statement carefully. Identify inputs, outputs, constraints, and edge cases. Note the expected time/space complexity if specified.

2. **Decompose into sub-problems.** Before writing any code, list 2-5 distinct sub-tasks the solution requires. For example, a graph problem might decompose into: build adjacency list, run BFS/DFS, extract result from traversal state. Each sub-task becomes a function.

3. **Design the function hierarchy.** Define the main entry-point function signature first. Then define helper function signatures with clear parameter names and return types. Order them: main function first, helpers below. No nested function definitions -- all functions at the top level.

4. **Write each function with a docstring.** Every function gets a 1-2 line docstring explaining: (a) what it computes, (b) the algorithmic approach it uses. The main function's docstring describes the overall strategy. Helper docstrings describe the specific sub-logic.

5. **Generate N candidate solutions (N=3-5).** Produce multiple distinct implementations. Vary the algorithmic approach where possible (e.g., iterative vs recursive, different data structures). If variation is limited, vary the decomposition boundaries or edge-case handling. Each candidate must follow the Chain-of-Function structure.

6. **Score each candidate per-function.** For each candidate, evaluate every function independently:
   - Does the function correctly implement what its docstring claims?
   - Does it handle boundary/edge inputs (empty arrays, zero, negative, max-size)?
   - Is the return type consistent with how the caller uses it?
   - Are there off-by-one errors, missing base cases, or incorrect loop bounds?
   Assign a confidence score (high/medium/low) to each function.

7. **Aggregate and select.** The solution-level score is the average of per-function scores. Select the candidate with the highest aggregate score. If two candidates tie, prefer the one with fewer functions (simpler decomposition) or better edge-case coverage.

8. **Verify with mental dry-run.** Trace the selected solution through 2-3 test cases including at least one edge case. If it fails, either fix it or fall back to the next-best candidate.

9. **Present the solution.** Output the selected code with its function structure intact. Include the docstrings. If the user asked for a single solution, show only the winner. If they asked for candidates, show all with your scoring rationale.

## Concrete Examples

**Example 1: Binary search on answer**

User: "Find the minimum capacity needed to ship packages within D days, given a list of package weights."

Approach:
1. Decompose: (a) binary search over capacity range, (b) simulate shipping with a given capacity, (c) check if simulation fits within D days
2. Generate Chain-of-Function solution:

```python
def ship_within_days(weights: list[int], days: int) -> int:
    """Find minimum ship capacity to deliver all packages within given days.
    Approach: Binary search on capacity between max(weights) and sum(weights)."""
    lo, hi = max(weights), sum(weights)
    while lo < hi:
        mid = (lo + hi) // 2
        if can_ship_in_time(weights, mid, days):
            hi = mid
        else:
            lo = mid + 1
    return lo

def can_ship_in_time(weights: list[int], capacity: int, days: int) -> bool:
    """Check whether all packages can be shipped within the day limit at given capacity.
    Approach: Greedily fill each day up to capacity, count days used."""
    days_needed = 1
    current_load = 0
    for w in weights:
        if current_load + w > capacity:
            days_needed += 1
            current_load = 0
        current_load += w
    return days_needed <= days
```

Per-function evaluation:
- `ship_within_days`: Binary search bounds correct (lo=max single package, hi=all in one day). Loop termination correct (lo < hi with hi = mid). **High confidence.**
- `can_ship_in_time`: Greedy simulation correctly resets load on overflow, starts at days_needed=1. Handles single-element case. **High confidence.**
- Aggregate: High. Selected.

**Example 2: Complex string parsing with multiple candidates**

User: "Parse a nested mathematical expression like `2*(3+(4*5))` and evaluate it."

Approach:
1. Decompose: (a) tokenize the string, (b) parse tokens into an AST, (c) evaluate the AST
2. Generate 3 candidates varying the parsing approach:

Candidate A -- Recursive descent with explicit tokenizer:
```python
def evaluate_expression(expr: str) -> int:
    """Evaluate a nested math expression string.
    Approach: Tokenize, then recursive descent parse and evaluate."""
    tokens = tokenize(expr)
    result, _ = parse_expr(tokens, 0)
    return result

def tokenize(expr: str) -> list[str]:
    """Split expression into number and operator tokens.
    Approach: Scan characters, accumulate digits into number tokens."""
    tokens = []
    i = 0
    while i < len(expr):
        if expr[i].isdigit():
            j = i
            while j < len(expr) and expr[j].isdigit():
                j += 1
            tokens.append(expr[i:j])
            i = j
        elif expr[i] in '+-*()':
            tokens.append(expr[i])
            i += 1
        else:
            i += 1  # skip whitespace
    return tokens

def parse_expr(tokens: list[str], pos: int) -> tuple[int, int]:
    """Parse additive expression (handles + and -).
    Approach: Parse left term, then consume +/- operators greedily."""
    left, pos = parse_term(tokens, pos)
    while pos < len(tokens) and tokens[pos] in '+-':
        op = tokens[pos]
        right, pos = parse_term(tokens, pos + 1)
        left = left + right if op == '+' else left - right
    return left, pos

def parse_term(tokens: list[str], pos: int) -> tuple[int, int]:
    """Parse multiplicative expression (handles *).
    Approach: Parse left factor, then consume * operators greedily."""
    left, pos = parse_factor(tokens, pos)
    while pos < len(tokens) and tokens[pos] == '*':
        right, pos = parse_factor(tokens, pos + 1)
        left *= right
    return left, pos

def parse_factor(tokens: list[str], pos: int) -> tuple[int, int]:
    """Parse a number or parenthesized sub-expression.
    Approach: If '(' consume until matching ')', else parse number."""
    if tokens[pos] == '(':
        val, pos = parse_expr(tokens, pos + 1)
        pos += 1  # skip ')'
        return val, pos
    return int(tokens[pos]), pos + 1
```

Candidate B -- Stack-based evaluation (single pass, no AST):
```python
def evaluate_expression(expr: str) -> int:
    """Evaluate nested math expression using operator-precedence stack.
    Approach: Two stacks (values, operators), process by precedence."""
    # ... (implementation with precedence handling)
```

Per-function evaluation of Candidate A:
- `tokenize`: Handles multi-digit numbers, skips whitespace, captures all operators. **High.**
- `parse_expr`: Correctly delegates to `parse_term` for precedence. Left-associative. **High.**
- `parse_term`: Mirrors `parse_expr` for multiplication. **High.**
- `parse_factor`: Handles parens and numbers. Correctly advances past closing paren. **High.**
- `evaluate_expression`: Thin wrapper, correct. **High.**

Candidate B stack-based: More error-prone with precedence edge cases. **Medium.**

Selected: Candidate A (higher aggregate, clearer structure).

**Example 3: Graph problem with edge-case focus**

User: "Find if a path exists between two nodes in an undirected graph given as an edge list."

```python
def valid_path(n: int, edges: list[list[int]], source: int, destination: int) -> bool:
    """Determine if a path exists between source and destination in an undirected graph.
    Approach: Build adjacency list, then BFS from source."""
    if source == destination:
        return True
    graph = build_adjacency_list(n, edges)
    return bfs_path_exists(graph, source, destination)

def build_adjacency_list(n: int, edges: list[list[int]]) -> dict[int, list[int]]:
    """Convert edge list to adjacency list representation.
    Approach: Iterate edges, add both directions for undirected graph."""
    adj = {i: [] for i in range(n)}
    for u, v in edges:
        adj[u].append(v)
        adj[v].append(u)
    return adj

def bfs_path_exists(graph: dict[int, list[int]], source: int, target: int) -> bool:
    """Check reachability from source to target using BFS.
    Approach: Standard BFS with visited set, return True if target found."""
    from collections import deque
    visited = {source}
    queue = deque([source])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor == target:
                return True
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return False
```

Per-function evaluation:
- `valid_path`: Early return for source==destination (edge case). **High.**
- `build_adjacency_list`: Initializes all n nodes, adds both directions. **High.**
- `bfs_path_exists`: Uses visited set (no infinite loops), deque for O(1) pops. **High.**

## Best Practices

**Do:**
- Always decompose before coding. Even 30 seconds of sub-problem identification prevents monolithic code. Aim for 3-6 functions per solution.
- Write docstrings *before* the function body. This forces you to articulate the approach and catches design errors early.
- Keep helpers pure (no side effects, no global state) wherever possible. Pure functions are easier to evaluate independently.
- Generate at least 2 candidates for hard problems. Even if one feels "obviously right," a second perspective catches blind spots.

**Avoid:**
- Do not create trivially thin wrappers that just call one other function. Each function should encapsulate real logic (at least 3-5 meaningful lines).
- Do not nest function definitions inside other functions. All functions at the top level for independent evaluation.
- Do not skip the per-function evaluation step. Monolithic "does the whole thing look right?" checking misses the bugs that hide at function boundaries.
- Do not over-decompose simple problems. A two-line solution should not be split into four functions. Apply Chain-of-Function when the problem has genuine sub-structure.

## Error Handling

- **Functions with mismatched interfaces:** If during evaluation you find that a helper returns a different type than the caller expects, this is the most common inter-function bug. Fix the return type or add explicit conversion.
- **Edge cases missed at function boundaries:** Each function should be evaluated with minimal inputs (empty list, single element, zero, None). Bugs often appear when one function handles the edge case but passes a problematic value to the next.
- **All candidates score low:** If every candidate has medium or low confidence across functions, do not pick the "least bad" one. Instead, re-analyze the problem decomposition -- the sub-problem split may be wrong. Redesign the function hierarchy and generate new candidates.
- **Candidate explosion on open-ended problems:** For problems with many valid approaches, limit yourself to 3-5 candidates maximum. Evaluate breadth at the decomposition level, not at the implementation level.

## Limitations

- **Simple problems do not benefit.** If the solution is naturally 5-10 lines with no sub-structure (e.g., "reverse a string"), forcing function decomposition adds ceremony without value. Use this technique for problems with genuine algorithmic complexity.
- **This is a heuristic, not a trained reward model.** The paper trains an actual PRM with meta-learning on reward signals. Claude's self-evaluation is a practical approximation -- it catches many errors but is not as rigorous as unit-test-verified Monte Carlo scores.
- **Candidate generation is limited by context.** Generating 8 full solutions for a complex problem consumes significant context. In practice, 2-4 candidates is the useful range for an interactive coding session.
- **Domain-specific code may resist decomposition.** UI rendering, DSL construction, and heavily framework-dependent code may not decompose cleanly into pure helper functions. The technique works best for algorithmic and data-processing code.

## Reference

Zhang, R., Qin, P., Cao, Q., Xue, E., & Xie, P. (2026). *FunPRM: Function-as-Step Process Reward Model with Meta Reward Correction for Code Generation.* arXiv:2601.22249v1. https://arxiv.org/abs/2601.22249v1

Key insight: Functions are the natural unit of reasoning in code. Decomposing solutions into modular functions and evaluating each function independently produces better selection among candidates than evaluating whole solutions at once. The meta-learning reward correction mechanism further improves accuracy by using clean final-solution signals to denoise per-step estimates.