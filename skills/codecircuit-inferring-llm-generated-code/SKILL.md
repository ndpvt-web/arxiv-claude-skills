---
name: "codecircuit-inferring-llm-generated-code"
description: "Assess LLM-generated code correctness using attribution graph analysis inspired by mechanistic interpretability. Apply structural reasoning diagnostics to identify buggy logic, predict failure modes, and suggest targeted fixes. Use when: 'analyze this code for correctness', 'why might this generated code be wrong', 'find structural bugs in this function', 'verify this algorithm logic', 'diagnose why this code fails', 'check this code for logical errors without running it'."
---

# CodeCircuit: Inferring LLM-Generated Code Correctness via Attribution Graph Analysis

This skill enables Claude to assess LLM-generated code correctness by applying the attribution graph reasoning framework from the CodeCircuit paper. Instead of relying solely on test execution or surface-level pattern matching, you simulate the mechanistic interpretability approach: decompose code into a line-level dependency graph, extract topological features (density, centrality, clustering), and use structural signatures to identify where reasoning breaks down. This technique is particularly powerful for catching "structural near misses"--code that looks almost right but has subtle logical flaws in control flow, boundary conditions, or state transitions.

## When to Use

- When a user asks you to review LLM-generated code (from ChatGPT, Copilot, or any model) for correctness without running it
- When debugging a function that passes some tests but fails edge cases, and the user wants to understand *why* the logic is wrong
- When a user asks "is this algorithm implementation correct?" for classic algorithms (binary search, sorting, graph traversal, dynamic programming)
- When verifying code translated between languages (Python to C++, Java to Python) where subtle semantic differences cause bugs
- When a user wants a deeper analysis than linting--structural reasoning about whether the code's logic actually solves the stated problem
- When identifying which specific line or expression in a function is the root cause of incorrect behavior

## Key Technique: Line-Level Attribution Graph Analysis

CodeCircuit's core insight is that correct code and incorrect code have measurably different *internal structure* when you trace how each line's output depends on prior computations. The paper constructs attribution graphs where nodes are code lines (or sub-expressions) and directed edges represent data/control flow dependencies weighted by their influence on the final output. Correct implementations form dense, well-connected subgraphs with balanced centrality--meaning no single line dominates the reasoning flow unnaturally. Incorrect code shows characteristic anomalies: fragmented components, abnormally high betweenness centrality on wrong operations (e.g., a greedy heuristic dominating where careful state tracking should), and high error-to-feature influence ratios where the model "shortcuts" past necessary logic.

The practical takeaway for code review is a structured diagnostic: build the dependency graph of a function, compute topological features, and look for specific failure signatures. High graph fragmentation suggests missing logic connections. Centrality spikes on simple expressions (like `high = mid` instead of `high = mid - 1`) indicate the code is taking a shortcut that skips a necessary boundary adjustment. Low clustering around loop update logic signals that iteration state is not being properly maintained across iterations.

The paper validates this across Python, C++, and Java, showing these structural patterns are language-agnostic. Topological features predict correctness at 79.9% AUROC for Python and improve with code complexity (80-92% AUROC for 10-30 line functions), precisely where surface heuristics fail.

## Step-by-Step Workflow

1. **Parse the code into a line-level dependency graph.** For each line, identify what variables it reads (inputs) and what it writes (outputs). Draw directed edges from each line that produces a value to every line that consumes it. Include control flow edges from conditionals and loop headers to the lines they govern.

2. **Annotate edge weights by influence strength.** Assign higher weights to edges where a variable is directly used in a computation (e.g., `result += arr[i]`) versus where it only affects control flow indirectly. Lines that contribute to the return value or final state get the strongest backward attribution.

3. **Identify the critical path.** Trace the highest-weight path from input parameters through to the return statement. This is the "algorithmic spine"--the sequence of operations that most directly determines the output. Verify that this path implements the stated algorithm correctly.

4. **Compute topological diagnostics on the graph:**
   - **Graph density**: ratio of actual edges to possible edges. Unusually low density for the code's complexity suggests missing logic.
   - **Betweenness centrality**: find which lines sit on the most shortest paths. If a trivial line (like a simple assignment) has disproportionately high centrality, it may be a shortcutting error.
   - **Clustering coefficient**: measure how interconnected each line's neighbors are. Low clustering around loop bodies or recursive calls signals broken state propagation.
   - **Connected components**: if the graph fragments into disconnected parts, there is likely dead code or a missing link between initialization and usage.

5. **Check for known failure signatures:**
   - **Off-by-one in binary search**: the update lines (`low = mid + 1`, `high = mid - 1`) should have balanced centrality. If one dominates, the boundary is likely wrong.
   - **Missing base case in recursion**: the base case should be reachable from the recursive call's dependency chain. A disconnected base case means it may never trigger.
   - **State mutation in loops**: every variable modified inside a loop should have an edge back to the loop condition or a subsequent iteration's input. Missing back-edges mean state is lost between iterations.

6. **Perform targeted causal analysis on suspicious lines.** For each flagged line, ask: "If I change this expression, does the dependency structure become more balanced?" Try the most common fix patterns (adjusting `<` vs `<=`, `mid` vs `mid-1`, `i` vs `i+1`) and check if the graph anomaly resolves.

7. **Report findings with structural evidence.** For each identified issue, explain: (a) which line is problematic, (b) what topological anomaly flagged it, (c) what the fix is, and (d) why the fix restores correct structure.

8. **Validate the fix against the full graph.** After proposing changes, re-trace the dependency graph to confirm no new anomalies are introduced and the critical path now correctly implements the algorithm.

## Concrete Examples

**Example 1: Binary Search Off-by-One Bug**

User: "Is this binary search correct?"
```python
def binary_search(arr, target):
    low, high = 0, len(arr) - 1
    while low <= high:
        mid = (low + high) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            low = mid + 1
        else:
            high = mid  # Bug: should be mid - 1
    return -1
```

Approach:
1. Build dependency graph: `low`/`high` -> `mid` -> `arr[mid]` -> comparison -> update `low`/`high` -> loop condition
2. Compute centrality: `high = mid` (line 9) has abnormally high betweenness because it creates a dependency cycle where `high` never advances past `mid`, making this line the bottleneck of all future iterations
3. Check failure signature: the `low = mid + 1` path shrinks the search space, but `high = mid` does not--asymmetric narrowing is the structural anomaly
4. Causal intervention: replacing `mid` with `mid - 1` balances the centrality between both update branches

Output:
```
Bug found on line 9: `high = mid` should be `high = mid - 1`.

Structural evidence: The dependency graph shows asymmetric search space
narrowing. The `low` update (mid + 1) advances past mid, but `high = mid`
does not exclude mid from the next iteration. This creates an infinite
loop when `low == high == mid` and `arr[mid] != target`.

The attribution graph confirms: line 9 has 3.2x higher betweenness
centrality than line 8, indicating it acts as a bottleneck rather than
a symmetric counterpart. Balanced binary search requires both updates
to exclude mid.

Fix: Change line 9 to `high = mid - 1`.
```

**Example 2: Dynamic Programming Missing Dependency**

User: "Check this longest common subsequence implementation."
```python
def lcs(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

Approach:
1. Build dependency graph: `dp[i][j]` depends on `dp[i-1][j-1]`, `dp[i-1][j]`, `dp[i][j-1]` depending on the branch
2. Verify critical path: return value `dp[m][n]` traces back through the recurrence to `dp[0][*]` and `dp[*][0]` (base cases, initialized to 0)
3. Check topological features: graph density is consistent, clustering coefficient around the inner loop is high (each cell properly references its three neighbors), no disconnected components
4. Centrality analysis: both branches of the conditional have balanced influence on the final result

Output:
```
No structural anomalies detected. The dependency graph shows:
- Symmetric recurrence: dp[i][j] correctly depends on dp[i-1][j-1] (match
  case) and max(dp[i-1][j], dp[i][j-1]) (mismatch case)
- Complete connectivity: every cell traces back to the base case row/column
- Balanced centrality: no single line dominates the computation path
- Proper initialization: dp[0][*] and dp[*][0] are implicitly 0

This implementation is structurally correct for LCS.
```

**Example 3: Graph Traversal with Subtle State Bug**

User: "Does this BFS find the shortest path correctly?"
```python
def shortest_path(graph, start, end):
    from collections import deque
    queue = deque([(start, [start])])
    while queue:
        node, path = queue.popleft()
        if node == end:
            return path
        for neighbor in graph[node]:
            queue.append((neighbor, path + [neighbor]))
    return None
```

Approach:
1. Build graph: `queue` -> `node, path` -> `neighbor` loop -> `queue.append` (cycle back)
2. Identify missing component: there is no `visited` set. The dependency graph shows the `neighbor` iteration has no filtering edge--every neighbor is enqueued unconditionally
3. Structural anomaly: the graph has no convergence mechanism. The clustering coefficient around the neighbor loop is abnormally low because there is no back-edge from "already processed nodes" to "skip this neighbor"
4. This causes infinite loops on cyclic graphs and exponential memory on dense graphs

Output:
```
Bug found: Missing visited-node tracking causes infinite loops on cyclic
graphs and redundant exploration on all graphs.

Structural evidence: The dependency graph around lines 8-9 shows an
unconditional feedback loop--every neighbor is enqueued with no filtering
edge. A correct BFS requires a "visited" guard that creates a dependency
from the set of processed nodes back to the neighbor enumeration,
pruning already-seen nodes.

Fix: Add a visited set and check membership before enqueuing:
    visited = {start}
    ...
    for neighbor in graph[node]:
        if neighbor not in visited:
            visited.add(neighbor)
            queue.append((neighbor, path + [neighbor]))
```

## Best Practices

- **Do:** Always build the full dependency graph before making any correctness judgment. Premature conclusions from reading code linearly miss structural issues that only appear in the graph topology.
- **Do:** Pay special attention to loop update variables and their back-edges. The paper shows that iteration state management is where most structural bugs hide--the centrality and clustering around loop bodies are the strongest predictive features.
- **Do:** Check for asymmetry in symmetric algorithms. Binary search, two-pointer, merge operations, and balanced tree updates should show symmetric centrality across their branches. Asymmetry is a reliable bug signal.
- **Do:** Use cross-language structural reasoning. The paper confirms that topological bug signatures transfer across Python, C++, and Java. The same density/centrality patterns flag the same logical errors regardless of syntax.
- **Avoid:** Relying on surface-level pattern matching alone (e.g., "this looks like standard binary search so it's probably fine"). The paper shows structural near-misses--code that looks nearly identical to correct implementations but has critical one-token differences--are exactly what surface heuristics miss.
- **Avoid:** Reporting graph metrics without explaining their implications. Raw numbers like "betweenness centrality = 0.45" mean nothing to the user. Always translate topological findings into concrete code-level explanations.

## Error Handling

- **Ambiguous control flow (complex nesting, early returns, exceptions):** When code has many conditional branches, the dependency graph becomes dense. Focus analysis on the critical path to the return value and flag any branch where a variable needed downstream is not guaranteed to be initialized.
- **Higher-order functions and callbacks:** Lambda functions and callbacks create implicit edges in the dependency graph. Expand them inline for analysis. If the callback modifies external state, flag this as a potential side-effect disconnection.
- **Code too short for structural analysis (< 5 lines):** Very small functions may not have enough structure for meaningful topological features. Fall back to direct logical reasoning for trivial functions.
- **Multiple return points:** Each return statement creates a separate critical path. Analyze each independently and verify they all produce correct results for their respective conditions.

## Limitations

- This technique works best on algorithmic code (10-30 lines) with clear input-output contracts. It is less effective on glue code, configuration, or I/O-heavy code where correctness depends on external system behavior rather than internal logic.
- The structural analysis simulates what CodeCircuit does with actual neural activations. Without access to the generating model's internals, we approximate attribution graphs using static data/control flow analysis. This is effective for common bug patterns but cannot catch bugs that arise from the model's specific learned biases.
- Performance-related bugs (time complexity issues, unnecessary copies) are not well captured by correctness-focused attribution graphs. The graph shows *what* the code computes, not *how efficiently*.
- Concurrency bugs (race conditions, deadlocks) require temporal analysis that static dependency graphs do not capture.

## Reference

**Paper:** [CodeCircuit: Toward Inferring LLM-Generated Code Correctness via Attribution Graphs](https://arxiv.org/abs/2602.07080v1) (He et al., 2026). Key sections: Section 3 for attribution graph construction via per-layer transcoders, Section 4 for topological feature extraction (30+ features across mechanical composition, global structure, and centrality categories), and Section 5 for the causal intervention case study on binary search. Code: [github.com/bruno686/CodeCircuit](https://github.com/bruno686/CodeCircuit).