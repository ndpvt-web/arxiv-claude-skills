---
name: "evaluating-achieving-controllable-code"
description: "Instruction-guided code completion that follows user constraints on algorithm choice, data structures, control flow, and code scope. Use when: 'complete this function using a deque-based BFS', 'finish this code with exactly 3 lines', 'implement the sort using quicksort not mergesort', 'complete using recursion instead of iteration', 'fill in this block with a single for loop', 'generate the rest using dynamic programming'."
---

# Controllable Code Completion

This skill enables Claude to perform **instruction-guided code completion** — generating code that not only works correctly but strictly follows user-specified constraints on implementation approach, algorithm choice, data structures, control flow patterns, and code scope. Based on the C3-Bench framework (arXiv:2601.15879), this technique separates two orthogonal concerns: *functional correctness* (does the code pass tests?) and *instruction adherence* (does it follow the user's stated constraints?). Most code completion today optimizes only for correctness; this skill ensures Claude also respects how the user wants something implemented.

## When to Use

- When a user asks to complete a function **using a specific algorithm** (e.g., "finish this with Dijkstra's, not Bellman-Ford")
- When a user wants code completed **within a precise scope** (e.g., "add exactly one if-else block", "complete in 3 lines")
- When the user specifies a **data structure constraint** during completion (e.g., "use a heap here, not a sorted list")
- When the user requests a **control flow pattern** (e.g., "use iteration, not recursion" or "implement with a while loop")
- When completing code where the user has a **parameter or variable naming requirement** (e.g., "use `left`/`right` pointers, not `i`/`j`")
- When a user provides partial code and asks to fill in a gap with **specific algorithmic constraints** (e.g., "complete the middle using two-pointer technique")

## Key Technique

The C3-Bench paper identifies two independent dimensions of controllable code completion:

**Implementation-Control Completion (ICC):** The user constrains *how* code is implemented. This covers four sub-categories: (1) **Structural Specifications** — requiring specific data structures or class hierarchies, (2) **Algorithmic Implementation** — mandating a particular algorithm or approach, (3) **Control Flow** — specifying execution patterns like recursion vs. iteration or loop types, and (4) **Critical Parameters** — constraining variable names, default values, or configuration choices. The key insight is that for any given function signature and test suite, there are often multiple *functionally equivalent* implementations that differ in approach. The user's instruction selects which one to produce.

**Scale-Control Completion (SCC):** The user constrains *how much* code is generated. This covers three scopes: (1) **Line Span** — completing a partial line, (2) **Multi-line** — generating exactly N complete lines, and (3) **Statement Block** — producing exactly one control structure (a single for-loop, a single if-else, etc.). This prevents the model from over-generating or under-generating relative to the user's intent.

The actionable insight is: **treat every code completion as a two-objective problem**. First, decompose the user's request into functional requirements (what must the code do?) and control requirements (how must it do it, and how much should be generated?). Then generate code that satisfies both, verifying each independently.

## Step-by-Step Workflow

1. **Parse the completion context.** Read the surrounding code — the prefix (code before the cursor), the suffix (code after the cursor if available), the function signature, imports, and any docstrings. Identify what functionality is expected from the structural context.

2. **Extract the user's control instructions.** Separate the request into two categories:
   - *Implementation controls*: algorithm, data structure, control flow, or parameter constraints
   - *Scale controls*: line count, block count, or scope limitations
   If the user only specifies one category, leave the other unconstrained.

3. **Identify the functionally equivalent alternatives.** Before generating, mentally enumerate 2-3 valid approaches that would satisfy the functional requirement. This ensures you understand the *choice space* the user is constraining. For example, if the task is shortest path, alternatives include Dijkstra's, Bellman-Ford, SPFA, or Floyd-Warshall.

4. **Select the approach matching the user's instruction.** From the alternatives identified in step 3, choose the one that matches the user's constraint. If the instruction is ambiguous, ask for clarification rather than guessing.

5. **Draft the completion within scope constraints.** If the user specified a line count or block type, generate exactly that amount. Do not add extra helper lines, comments, or blank lines that violate the scope. If the user said "one for loop," produce exactly one for loop — not a for loop plus a conditional.

6. **Verify functional correctness.** Mentally trace or actually run the generated code against the expected behavior. Ensure it handles edge cases the same way the surrounding code implies.

7. **Verify instruction adherence independently.** After confirming correctness, re-read the user's control instructions and check each one:
   - Does it use the specified algorithm? (not a different one that happens to work)
   - Does it use the specified data structure? (not an equivalent substitute)
   - Does it match the scope constraint? (count the lines / blocks)
   - Does it follow naming or parameter requirements?

8. **If either check fails, regenerate.** Do not ship code that is correct but violates the instruction, or code that follows the instruction but is buggy. Both objectives must be met.

9. **Present the completion with a brief explanation** of how it satisfies both the functional and control requirements, so the user can verify.

## Concrete Examples

**Example 1: Algorithm-Constrained Completion**

User provides this partial code and asks: "Complete the `shortest_path` function using the SPFA algorithm with Small Label First (SLF) optimization using a deque."

```python
# Prefix:
from collections import deque

def shortest_path(graph, source, n):
    dist = [float('inf')] * n
    dist[source] = 0
    in_queue = [False] * n
    # CURSOR — complete from here

# Suffix:
    return dist
```

Approach:
1. Recognize the functional requirement: single-source shortest path.
2. Identify the control instruction: SPFA + SLF optimization + deque.
3. Alternatives considered: standard SPFA with list, Dijkstra's with heap, Bellman-Ford with relaxation loop — all rejected per instruction.
4. Generate using deque with SLF appendleft/append logic.

Output:
```python
    q = deque([source])
    in_queue[source] = True
    while q:
        u = q.popleft()
        in_queue[u] = False
        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                if not in_queue[v]:
                    in_queue[v] = True
                    # SLF: if new dist is small, push to front
                    if q and dist[v] < dist[q[0]]:
                        q.appendleft(v)
                    else:
                        q.append(v)
```

Verification: Uses deque (not heap or list). Implements SLF via the `appendleft` conditional. SPFA relaxation pattern, not Dijkstra's greedy extraction.

---

**Example 2: Scope-Constrained Completion**

User: "Complete the missing part with exactly one for-loop block. No additional statements outside the loop."

```python
def flatten(nested_list):
    result = []
    # CURSOR — complete with exactly one for-loop
    return result
```

Approach:
1. Functional requirement: flatten a nested list into `result`.
2. Scale control: exactly one for-loop block, nothing else.
3. A recursive approach would require additional statements — rejected.
4. A list comprehension avoids a for-loop — rejected.
5. Use a single for-loop with extend to stay within scope.

Output:
```python
    for item in nested_list:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
```

Verification: Exactly one for-loop block. The if/else is nested inside the for-loop, not an additional top-level statement. No extra lines outside the loop.

---

**Example 3: Data Structure and Control Flow Constraints**

User: "Complete this using a min-heap (heapq), not a sorted list. Use a while loop to process, not a for loop."

```python
import heapq

def k_smallest_pairs(nums1, nums2, k):
    """Return k pairs with smallest sums from nums1 and nums2."""
    if not nums1 or not nums2:
        return []
    # CURSOR
```

Approach:
1. Functional requirement: k smallest sum pairs from two arrays.
2. Implementation controls: heapq (not sorted list), while loop (not for loop).
3. Generate heap-based approach with while-loop extraction.

Output:
```python
    heap = []
    for i in range(min(k, len(nums1))):
        heapq.heappush(heap, (nums1[i] + nums2[0], i, 0))
    result = []
    while heap and len(result) < k:
        _, i, j = heapq.heappop(heap)
        result.append([nums1[i], nums2[j]])
        if j + 1 < len(nums2):
            heapq.heappush(heap, (nums1[i] + nums2[j + 1], i, j + 1))
    return result
```

Verification: Uses `heapq` (not a sorted list). Main processing uses `while heap` (not a for loop). The initial for-loop is for seeding the heap, not for processing — the user's constraint applies to the processing loop.

## Best Practices

- **Do:** Explicitly acknowledge the user's constraint before generating. Restate it (e.g., "Using BFS with a deque as requested") so both you and the user confirm alignment.
- **Do:** When multiple valid interpretations of an instruction exist, ask the user to clarify rather than picking one silently. "Use a stack" could mean `list` used as a stack or `collections.deque`.
- **Do:** Count lines and blocks literally when scope constraints are given. "Exactly 3 lines" means 3 lines of code, not 3 logical statements that happen to span 5 lines.
- **Do:** Verify instruction adherence *after* verifying correctness — both must pass independently.
- **Avoid:** Substituting an "equivalent" data structure or algorithm that violates the instruction, even if it's more idiomatic or efficient. If the user says "use a list as a stack," don't switch to `collections.deque` for performance.
- **Avoid:** Over-generating beyond the requested scope. If asked for "one if-block," do not add an else branch, a logging statement, or a comment block.
- **Avoid:** Ignoring control instructions when they seem suboptimal. The user may have pedagogical, compatibility, or codebase-consistency reasons for their choice.

## Error Handling

- **Contradictory instructions:** If the user's functional requirement conflicts with their implementation constraint (e.g., "sort this in O(n) using comparison-based sorting"), explain the impossibility and suggest the closest feasible alternative.
- **Ambiguous scope:** If "complete in 2 lines" is unclear (2 logical statements? 2 physical lines?), default to physical lines and note the assumption.
- **Insufficient context:** If the prefix/suffix doesn't provide enough information to determine what the code should do, ask for the expected behavior or test cases before generating.
- **Constraint impossible in language:** If the user requests a construct that doesn't exist in the target language (e.g., "use a do-while loop" in Python), explain the limitation and offer the idiomatic equivalent with user approval.

## Limitations

- This approach works best for **single-function or single-block completions** where the scope is clearly defined. For large multi-file generation tasks, instruction adherence becomes harder to verify exhaustively.
- **Subjective instructions** like "write clean code" or "make it efficient" are not controllable in the C3-Bench sense — this skill targets objective, verifiable constraints (specific algorithms, exact line counts, named data structures).
- Scale-control (exact line counts) can conflict with code quality. Forcing exactly N lines may produce awkward formatting or overly compressed logic. When this happens, flag the tension to the user.
- The technique assumes the user understands the implications of their constraint. If they request an O(n^2) algorithm for a large-n problem, follow the instruction but note the performance consideration.

## Reference

- **Paper:** [Evaluating and Achieving Controllable Code Completion in Code LLM](https://arxiv.org/abs/2601.15879v1) — Zhang et al., 2026. Focus on Section 3 (C3-Bench construction and ICC/SCC taxonomy) and Section 5 (data synthesis pipeline) for the core methodology.