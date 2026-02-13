---
name: "state-transition-framework-reasoning"
description: |
  Applies a state-transition reasoning framework that models multi-step reasoning as an evolving state,
  compressing historical reasoning into a compact state representation and correcting noisy intermediate
  steps via momentum-based smoothing. Improves efficiency and accuracy on long reasoning chains.
  Trigger phrases: "reason step by step with state tracking", "use state-transition reasoning",
  "solve this with efficient chain-of-thought", "break this into reasoning states",
  "apply state-based reasoning", "reason efficiently without overthinking"
---

# State-Transition Framework for Efficient LLM Reasoning

This skill teaches Claude to structure complex multi-step reasoning as a **state-transition process** based on the framework from Zhang et al. (ICLR 2026). Instead of generating an unconstrained chain-of-thought where every token attends to every prior token, the approach segments reasoning into discrete steps, maintains a compressed **reasoning state** that accumulates conclusions from prior steps, and applies **momentum-based correction** to detect and dampen noisy or redundant reasoning. The result is reasoning that stays focused, avoids overthinking, and produces clearer intermediate conclusions.

## When to Use

- When a user asks Claude to solve a complex multi-step math, logic, or coding problem that requires long reasoning chains (e.g., competitive programming, algorithm design, debugging multi-layered issues)
- When Claude's initial reasoning attempt produces verbose, repetitive, or circular thinking that wastes tokens without progress
- When the user requests "efficient reasoning" or wants Claude to avoid overthinking on a problem
- When solving problems that require tracking many intermediate results across steps (e.g., dynamic programming derivations, proof construction, multi-file refactoring plans)
- When the user explicitly asks for state-tracked or structured chain-of-thought reasoning
- When debugging a problem where prior reasoning steps have introduced confusion or contradictory conclusions

## Key Technique

### Reasoning as State Transitions

Standard chain-of-thought reasoning treats the entire generated sequence as a flat token stream. Every new token must (implicitly) attend to all prior tokens, which becomes costly and error-prone as reasoning grows long. The state-transition framework instead **segments** the reasoning into discrete steps, where each step produces a **conclusion** that gets folded into a compact **reasoning state** `S_t`. At step `t`, the reasoner only needs the current query and the accumulated state `S_t` -- not the raw text of all prior steps. This is analogous to how a programmer tracks variable values through a debugger rather than re-reading the entire execution log.

### Momentum-Based Noise Correction

Not every reasoning step moves the solution forward. Some steps introduce noise -- wrong assumptions, unnecessary tangents, or redundant restatements. The framework detects this by computing the **reasoning direction** at each step: `delta_t = S_t - S_{t-1}`. It then maintains a running average of all prior directions `delta_bar_{t-1}`. Each step's direction is corrected via momentum blending: `delta_hat_t = (1 - alpha) * delta_t + alpha * delta_bar_{t-1}`, where `alpha` increases as reasoning progresses (starting at 0, capping at ~0.4). Early steps are trusted more; later steps are increasingly regularized toward the established reasoning trajectory. This prevents late-stage overthinking and circular reasoning.

### Practical Application for Claude

Since Claude cannot modify its own attention mechanism at inference time, this skill operationalizes the framework as a **structured reasoning protocol**: explicitly maintain a state summary after each reasoning step, evaluate whether each new step is productive or noisy relative to the accumulated direction, and apply self-correction when reasoning drifts.

## Step-by-Step Workflow

1. **Parse the problem and identify the reasoning type.** Determine whether the task is mathematical derivation, code debugging, logical deduction, planning, or another multi-step domain. Identify what a "step" means in this context (e.g., one equation transformation, one function analysis, one logical inference).

2. **Initialize the reasoning state S_0.** Write a concise summary of what is known from the problem statement: the givens, constraints, and goal. This is the initial state before any reasoning begins. Format it as a labeled block:
   ```
   STATE S_0: [Known: ..., Goal: ..., Constraints: ...]
   ```

3. **Execute reasoning step t.** Perform one logical unit of reasoning -- a single derivation, deduction, or analysis. Keep the step focused on one sub-goal. Do not attempt multiple leaps in a single step.

4. **Extract the conclusion and update state S_t.** After each step, write the new state by appending the step's conclusion to the prior state. Drop the working details; keep only the result. Format:
   ```
   STATE S_t: [S_{t-1} conclusions + new conclusion from step t]
   ```

5. **Evaluate reasoning direction.** Compare S_t to S_{t-1}. Ask: "Did this step produce new information that moves toward the goal?" If the delta is near-zero (restated something already known), tangential (explored an irrelevant path), or contradictory (conflicts with established conclusions), flag it as noisy.

6. **Apply momentum correction for noisy steps.** If a step is flagged noisy, do not simply discard it. Instead, blend: keep the portion that aligns with the overall reasoning trajectory and discard the divergent portion. Rewrite the state update to reflect only the productive component. Explicitly note the correction:
   ```
   CORRECTION: Step t was partially noisy. Retaining [useful part], discarding [tangent/repetition].
   ```

7. **Increase skepticism as reasoning lengthens.** For early steps (steps 1-3), trust each step's output with high confidence. As step count increases beyond 5-6, apply greater scrutiny to each new step -- the equivalent of increasing alpha. Ask: "Is this step genuinely needed, or am I overthinking?"

8. **Check for termination.** After each state update, evaluate whether S_t contains sufficient information to answer the original question. If yes, proceed to formulate the answer. Do not continue reasoning past the point of sufficiency.

9. **Synthesize the final answer from the terminal state.** Construct the answer directly from the final state S_T, not by re-reading all intermediate steps. The state should contain everything needed.

10. **Present the answer with optional reasoning trace.** Give the user the final answer first, then optionally show the state progression as a compact trace for transparency.

## Concrete Examples

**Example 1: Multi-step mathematical reasoning**

User: "What is the remainder when 2^100 is divided by 7? Use state-transition reasoning."

```
STATE S_0: [Known: compute 2^100 mod 7, Goal: find remainder]

Step 1: Compute small powers of 2 mod 7 to find a cycle.
  2^1 mod 7 = 2, 2^2 mod 7 = 4, 2^3 mod 7 = 1

STATE S_1: [2^k mod 7 cycles with period 3: {2, 4, 1, 2, 4, 1, ...}]

Step 2: Express 100 in terms of the cycle length.
  100 = 3 * 33 + 1, so 2^100 mod 7 = 2^1 mod 7 = 2

STATE S_2: [100 mod 3 = 1, therefore 2^100 mod 7 = 2]

Termination check: S_2 contains the answer.

ANSWER: The remainder is 2.
```

**Example 2: Debugging a multi-layered code issue**

User: "My API returns 500 errors intermittently. The endpoint calls a database, then a cache layer, then formats the response. Help me reason through this."

```
STATE S_0: [Known: intermittent 500s on API endpoint with 3 stages:
  DB query -> cache layer -> response formatting. Goal: identify root cause]

Step 1: Intermittent failures rule out static bugs (syntax, logic).
  Points to: resource exhaustion, timeouts, race conditions, or transient
  external failures.

STATE S_1: [S_0 + Root cause category: transient/resource issue, not static bug]

Step 2: Analyze each stage for transient failure modes.
  - DB: connection pool exhaustion, query timeout, deadlocks
  - Cache: connection refused, eviction during read, serialization error
  - Formatting: unlikely to be intermittent unless input-dependent

STATE S_2: [S_1 + Primary suspects: DB connection pool or timeout;
  Cache connection or serialization; Formatting ruled low-probability]

Step 3: Intermittent pattern narrows further. If failures correlate with
  load, connection pool exhaustion is most likely. If random, timeout or
  deadlock.

STATE S_3: [S_2 + Key diagnostic: check if failure rate correlates with
  request volume. If yes -> pool exhaustion. If no -> timeout/deadlock]

CORRECTION: Step 3 partially restated Step 2's suspects.
  Retaining: the load-correlation diagnostic.
  Discarding: repeated enumeration of failure modes.

Step 4: Recommend concrete investigation actions.
  1. Check DB connection pool config (max connections, timeout settings)
  2. Add structured logging at each stage boundary with latency
  3. Check cache client retry/timeout configuration
  4. Look for correlation between error timestamps and traffic spikes

STATE S_4: [S_3 + Action plan: 4 diagnostic steps prioritized by likelihood]

Termination: S_4 contains actionable diagnosis.

ANSWER:
Most likely cause: DB connection pool exhaustion under load.
Diagnostic steps:
1. Check your DB connection pool max size and wait timeout
2. Add timing logs between each stage (DB -> cache -> format)
3. Review cache client timeout and retry config
4. Correlate 500 error timestamps with traffic volume
```

**Example 3: Algorithm design with self-correction**

User: "Design an algorithm to find the longest increasing subsequence in O(n log n)."

```
STATE S_0: [Known: LIS problem, Goal: O(n log n) algorithm]

Step 1: The naive DP approach is O(n^2) -- for each element, scan all
  prior elements. The bottleneck is finding the best predecessor.

STATE S_1: [S_0 + Naive DP is O(n^2), bottleneck: finding best predecessor
  for each element]

Step 2: Replace linear scan with binary search. Maintain an array `tails`
  where tails[i] = smallest tail element of all increasing subsequences
  of length i+1.

STATE S_2: [S_1 + Key insight: maintain `tails` array, use binary search
  to find insertion point. tails is always sorted.]

Step 3: For each element x in the input:
  - If x > tails[-1]: append x (extends longest subsequence)
  - Else: binary search for leftmost tails[j] >= x, set tails[j] = x

STATE S_3: [S_2 + Complete algorithm: iterate + binary search on tails.
  O(n) iterations * O(log n) binary search = O(n log n)]

Termination: S_3 contains the complete algorithm and complexity proof.

ANSWER:
Maintain a `tails` array where tails[i] holds the smallest possible
tail element for an increasing subsequence of length i+1. For each
element, binary search `tails` to find where it fits. This gives
O(n log n) time, O(n) space.
```

## Best Practices

**Do:**
- Keep each reasoning step atomic -- one inference, one derivation, one sub-conclusion per step
- Write state updates as cumulative summaries, not append-only logs; drop superseded information
- Flag and correct noisy steps explicitly rather than silently continuing with confusion
- Increase self-scrutiny as step count grows; if you're past step 6-7, strongly question whether each new step is necessary
- Terminate as soon as the state contains enough to answer the question

**Avoid:**
- Do not carry forward the full text of prior steps; the state summary replaces them
- Do not skip the state update after a step -- an unrecorded conclusion is a lost conclusion
- Do not continue reasoning after the answer is derivable from the current state (this is the overthinking the framework corrects)
- Do not use this framework for simple one-step questions; it adds overhead that is only justified for 3+ step reasoning chains
- Do not treat every step as equally trustworthy; later steps in long chains deserve more skepticism

## Error Handling

- **Circular reasoning detected:** If S_t is essentially identical to S_{t-2} or earlier, halt and reframe the sub-problem. The reasoning has entered a loop. Explicitly state the loop and try an alternative approach.
- **Contradictory state:** If a new step's conclusion contradicts an established conclusion in S_{t-1}, do not silently overwrite. Flag the contradiction, determine which conclusion has stronger support, and update the state with a resolution note.
- **State explosion:** If the state summary grows beyond 5-6 key facts, compress by identifying which conclusions subsume earlier ones. Keep the state lean.
- **Premature termination:** If the state appears complete but the answer derived from it is wrong or incomplete, add a verification step that cross-checks the answer against the original constraints in S_0.

## Limitations

- This framework adds structural overhead that is counterproductive for simple questions answerable in 1-2 reasoning steps
- The momentum correction heuristic (increasing alpha) is a rough approximation; it may over-dampen genuinely novel late-stage insights in some problems
- Claude cannot literally implement linear attention at inference time; this skill operationalizes the paper's insights as a structured reasoning protocol, which captures the benefits of state compression and noise correction but not the raw computational speedup
- Problems requiring heavy backtracking (e.g., search problems with many dead ends) may need the full reasoning trace rather than a compressed state
- The framework works best for sequential reasoning; problems requiring parallel exploration of multiple hypotheses need adaptation (maintain multiple state branches)

## Reference

**Paper:** [A State-Transition Framework for Efficient LLM Reasoning](https://arxiv.org/abs/2602.01198v1) -- Zhang et al., ICLR 2026. Key sections: Section 3 (Mixed Attention Module and state update formulas), Section 4 (state-based reasoning strategy with momentum correction), Table 1 (benchmark results showing 16-18% efficiency gains with maintained or improved accuracy).