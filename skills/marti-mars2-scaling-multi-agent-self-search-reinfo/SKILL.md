---
name: "marti-mars2-scaling-multi-agent-self-search-reinfo"
description: "Multi-agent tree-search code generation using heterogeneous agent collaboration with error-feedback refinement. Spawns multiple agents with distinct strategies to explore solution trees, iteratively refine code via structured error diagnostics, and select the best solution through diversity-aware ranking. Use when: 'solve this hard coding problem with multiple agents', 'use multi-agent search to generate code', 'use MARTI-MARS2 to solve this', 'spawn a team to tackle this algorithm problem', 'use tree search to find the best solution', 'generate code with agent collaboration'."
---

# MARTI-MARS2: Multi-Agent Self-Search for Code Generation

This skill enables Claude to apply the MARTI-MARS2 framework -- a multi-agent collaborative tree-search strategy for solving difficult code generation tasks. Instead of a single agent generating one solution, this approach spawns multiple agents with deliberately different problem-solving strategies (heterogeneous policies), organizes their work as a search tree with iterative error-feedback refinement, and selects the best solution through structured evaluation. The core insight from the paper: **policy diversity across agents is the critical factor for scaling code generation quality**, more impactful than simply sampling more solutions from a single agent.

## When to Use

- When the user asks to solve a competitive programming problem (LeetCode Hard, Codeforces, USACO, CodeContests) where a single attempt is unlikely to succeed
- When a coding task has failed multiple times and the user wants a systematic multi-attempt strategy with error correction
- When the user explicitly requests multi-agent collaboration for code generation
- When tackling algorithmic problems with multiple valid solution strategies (DP, greedy, divide-and-conquer) and the best approach is unclear
- When the user wants to maximize probability of a correct solution on a problem with test cases available for validation
- When generating code that must pass a rigorous test suite and brute-force sampling is insufficient

## Key Technique

**Multi-Agent Tree Search with Heterogeneous Strategies.** MARTI-MARS2 frames code generation as a tree-search problem where multiple agents collaboratively expand a solution tree. Each node in the tree represents a candidate solution or partial refinement. Unlike standard best-of-N sampling (generating N independent solutions from one agent), agents here are *heterogeneous* -- they approach the problem with distinct strategies, algorithmic paradigms, and implementation styles. This forces exploration of a broader solution space and prevents the convergence trap where repeated sampling yields minor variations of the same flawed approach.

**Error-Feedback Refinement at Each Node.** When a candidate solution fails tests, the framework constructs structured error diagnostics: execution summary, error type and traceback for runtime errors, or specific failed input-output pairs for wrong-answer cases. These diagnostics are fed back to agents as context for generating refined child nodes. A depth-aware exploration strategy (inspired by Thompson sampling) balances broad exploration at shallow levels with focused refinement at deeper levels -- early on, agents try diverse approaches; later, they converge on fixing the most promising candidates.

**Diversity-Aware Solution Selection.** Among all candidate solutions that pass available tests, the framework ranks by estimated correctness on unseen inputs rather than simply taking the first passing solution. In practice, this means preferring solutions that use distinct algorithmic strategies from other passing candidates, as diversity in passing solutions correlates with robustness. The paper demonstrates this yields 77.7% on LiveCodeBench with two 32B models, outperforming single-agent approaches by 10+ percentage points.

## Step-by-Step Workflow

1. **Analyze the problem and identify 2-4 distinct solution strategies.** Read the problem statement, constraints, and examples. Determine fundamentally different algorithmic approaches (e.g., for a shortest-path problem: BFS, Dijkstra, dynamic programming, A*). These become the initial heterogeneous "policies" for your agents.

2. **Spawn heterogeneous agents with distinct system prompts.** Launch 2-4 agents using the Task tool, each with an explicit instruction to pursue a *specific* algorithmic strategy. Include the problem statement and test cases in each prompt. Critically, instruct each agent differently: one favors brute-force-then-optimize, another targets the theoretically optimal approach, a third tries an unconventional angle.

3. **Collect initial candidate solutions (tree depth 0).** Each agent produces its first solution attempt. Gather all candidates as the root-level nodes of the search tree.

4. **Validate each candidate against available test cases.** Run every solution against the provided examples and any additional test cases. Record pass/fail status and capture structured error diagnostics for failures: the specific test case that failed, expected vs. actual output, or the runtime error traceback.

5. **Construct error-feedback refinement prompts for failed candidates.** For each failed solution, build a structured diagnostic containing: (a) the original code, (b) which tests passed/failed, (c) for wrong answers: the failing input, expected output, and actual output, (d) for runtime errors: error type, message, and traceback. This becomes the context for the next tree level.

6. **Expand the tree: agents refine promising failed candidates.** Send each failed-but-promising candidate (those that passed some tests or had minor errors) to an agent for refinement. Assign different agents to refine different candidates to maintain diversity. Apply depth-aware focus: at depth 1-2, allow agents to substantially restructure; at depth 3+, constrain to targeted bug fixes.

7. **Prune hopeless branches, double down on promising ones.** If a candidate fails all tests with fundamental logic errors after one refinement, abandon that branch. If a candidate passes most tests with one edge case failing, allocate more refinement attempts to it.

8. **Repeat steps 4-7 for 2-4 iterations (tree depth 2-4).** Each iteration narrows the search. Track the full tree of attempts to avoid revisiting failed approaches.

9. **Select the best solution using diversity-aware ranking.** Among all solutions passing all available tests, prefer the one using the most robust algorithmic approach (best time complexity, fewest edge-case assumptions). If multiple pass, select the one whose strategy is most different from the others -- diverse correctness signals higher confidence.

10. **Return the solution with a brief explanation of the search process.** Report which strategies were tried, which failed and why, and why the selected solution is the strongest candidate.

## Concrete Examples

**Example 1: Competitive Programming -- Longest Increasing Subsequence Variant**

User: "Solve this: Given an array of N integers, find the length of the longest strictly increasing subsequence where no two adjacent selected elements in the original array are more than K positions apart. N up to 10^5, K up to 100."

Approach:
1. Identify distinct strategies:
   - Agent A: Standard DP with O(N*K) transition, segment tree optimization
   - Agent B: Patience sorting variant with K-constraint pruning
   - Agent C: Binary search approach adapting classic LIS with sliding window
2. Spawn 3 agents, each with the problem and their assigned strategy
3. Agent A produces a segment tree DP -- passes examples but TLE on large inputs
4. Agent B produces patience sorting -- fails on edge case with K=1
5. Agent C produces binary search with deque -- passes all examples
6. Refine Agent A's solution: feed back "TLE on N=10^5" diagnostic, agent optimizes to use BIT instead of segment tree, now passes
7. Refine Agent B's solution: feed back failing test case, agent fixes boundary condition, now passes
8. Three passing solutions. Select Agent A's BIT-based DP as final answer (optimal O(N*K*logN) complexity, clearest correctness argument)

Output:
```python
import sys
from sortedcontainers import SortedList

def solve():
    input_data = sys.stdin.buffer.read().split()
    idx = 0
    n = int(input_data[idx]); idx += 1
    k = int(input_data[idx]); idx += 1
    a = [int(input_data[idx + i]) for i in range(n)]; idx += n

    # dp[i] = length of LIS ending at index i with K-gap constraint
    dp = [1] * n
    # BIT indexed by value for range-max query over positions [i-k, i-1]
    # Coordinate compress values
    sorted_vals = sorted(set(a))
    rank = {v: i + 1 for i, v in enumerate(sorted_vals)}
    size = len(sorted_vals) + 1
    bit = [0] * size

    def update(pos, val):
        while pos < size:
            bit[pos] = max(bit[pos], val)
            pos += pos & (-pos)

    def query(pos):
        res = 0
        while pos > 0:
            res = max(res, bit[pos])
            pos -= pos & (-pos)
        return res

    # Process with sliding window of size K
    from collections import deque
    pending = deque()
    ans = 0
    for i in range(n):
        # Activate elements from window [i-k, i-1]
        while pending and pending[0][0] <= i - 1:
            idx_p, val_p, dp_p = pending.popleft()
            # This is already activated
        # Query BIT for max dp among indices in [i-k, i-1] with value < a[i]
        r = rank[a[i]]
        best = query(r - 1)
        dp[i] = best + 1
        ans = max(ans, dp[i])
        # Schedule activation after K steps
        pending.append((i + k, r, dp[i]))
        # Activate if within window
        if i >= 0:
            update(r, dp[i])

    print(ans)

solve()
```

**Example 2: Debugging a Failing Solution with Multi-Agent Refinement**

User: "My solution to this graph problem keeps getting Wrong Answer on test 3. Here's my code and the problem statement. Can you fix it using multi-agent search?"

Approach:
1. Analyze the failing code and problem. Identify potential bug categories:
   - Agent A: Focus on graph construction errors (off-by-one, directed vs undirected)
   - Agent B: Focus on algorithm correctness (BFS vs DFS choice, visited tracking)
   - Agent C: Focus on edge cases (disconnected graphs, self-loops, integer overflow)
2. Each agent independently diagnoses and patches the code
3. Run all patches against provided test cases including test 3
4. Agent A found the bug: adjacency list was treating undirected edges as directed
5. Agent C found an additional overflow issue that would fail on larger tests
6. Merge both fixes, validate against all tests
7. Return fixed solution with explanation of both bugs found

Output: The fixed code with inline comments marking the two bugs found, plus a summary:
```
Search tree explored 3 strategies across 2 refinement depths.
Bug 1 (Agent A, depth 0): Edges added in one direction only -- fixed by adding reverse edge.
Bug 2 (Agent C, depth 1): Integer sum overflowed int32 on test 7 -- fixed by using long long.
All 10 test cases now pass.
```

**Example 3: Multi-Strategy API Implementation**

User: "Implement a rate limiter that handles both fixed-window and sliding-window semantics, with Redis backend. It needs to handle 10K+ requests/sec."

Approach:
1. Distinct strategies:
   - Agent A: Lua script approach (atomic operations, single round-trip)
   - Agent B: Redis pipeline with MULTI/EXEC transactions
   - Agent C: Token bucket via sorted sets with ZRANGEBYSCORE
2. Spawn agents, collect implementations
3. Validate each against a test harness simulating concurrent requests
4. Agent A's Lua script passes all correctness tests and benchmarks at 15K req/sec
5. Agent B's pipeline has a race condition under concurrency -- refine with error feedback showing the specific interleaving that fails
6. After refinement, Agent B passes but benchmarks at 8K req/sec
7. Select Agent A's Lua script approach (correct, performant, atomic)

## Best Practices

- **Do: Assign genuinely different algorithmic strategies to each agent.** The power of this approach comes from diversity. "Try the same approach but with different variable names" is useless. Each agent should pursue a fundamentally different algorithm or design pattern.

- **Do: Construct rich, structured error feedback.** When a solution fails, don't just say "wrong answer." Include the specific failing input, expected output, actual output, and any observable execution details. This is the mechanism that makes tree-search refinement effective.

- **Do: Limit tree depth to 3-4 levels.** Diminishing returns set in quickly. If a solution hasn't converged after 3 rounds of refinement, the underlying approach is likely flawed -- prune that branch rather than continuing to patch.

- **Do: Prefer 2-3 heterogeneous agents over 5+ homogeneous ones.** Two agents with genuinely different strategies outperform five agents with minor prompt variations. Quality of diversity beats quantity.

- **Avoid: Running all agents on trivial problems.** If the problem is straightforward (easy/medium difficulty), a single well-crafted solution attempt is sufficient. Reserve multi-agent search for problems where the correct approach is genuinely uncertain.

- **Avoid: Letting agents see each other's solutions during initial generation.** Cross-contamination reduces diversity. Each agent should produce its first attempt independently. Only share information during the refinement phase via structured error feedback, not by showing other agents' code.

## Error Handling

- **All agents produce incorrect solutions:** Fall back to a meta-analysis phase. Collect all failure modes, identify common pitfalls, and spawn a new agent with explicit instructions to avoid those specific errors. This is equivalent to expanding the tree with a "learned" prior.

- **Test cases are insufficient for validation:** When only example test cases are available, generate additional edge-case tests (empty input, maximum constraints, boundary values) before running the tree search. Without adequate tests, the refinement loop cannot function.

- **Agents converge on the same flawed approach:** This indicates the strategy prompts weren't sufficiently distinct. Re-launch with more explicit algorithmic constraints (e.g., "You MUST use dynamic programming, not greedy" vs. "You MUST use a greedy approach with proof of correctness").

- **Timeout during tree search:** If the problem is time-sensitive, cap total agents at 2 and tree depth at 2. Prioritize breadth (distinct strategies) over depth (iterative refinement) when time is constrained.

## Limitations

- **Requires executable test cases.** The error-feedback refinement loop depends on running code against tests. For problems without test cases (e.g., open-ended system design), this approach degrades to basic multi-agent brainstorming without the tree-search benefit.

- **Overhead scales with agent count.** Each agent consumes a full context window and inference budget. For simple problems, this overhead is wasteful. Use this approach selectively on hard problems where single attempts have a low success rate.

- **Diversity is hard to guarantee.** Even with distinct prompts, agents may converge on similar solutions if the problem strongly suggests one canonical approach. The technique works best when multiple valid solution paradigms exist.

- **Not a substitute for domain expertise.** If the problem requires niche knowledge (e.g., advanced number theory, specific API semantics), multiple agents without that knowledge won't compensate. Ensure each agent's prompt includes relevant domain context.

- **Diminishing returns beyond 3-4 agents.** The paper's scaling law shows the largest gains from 1 to 2 agents, with progressively smaller improvements as agent count increases. The sweet spot for practical use is 2-3 agents.

## Reference

**Paper:** [MARTI-MARS2: Scaling Multi-Agent Self-Search via Reinforcement Learning for Code Generation](https://arxiv.org/abs/2602.07848v1) -- Look for Section 3 (framework design with GRPO-based tree search), Section 4.2 (heterogeneous vs. homogeneous training comparison), and Section 4.4 (MARS2-T+ inference strategy with error-feedback integration and depth-guided exploration).