---
name: "adaptive-confidence-gating-multi-agent"
description: "Multi-agent code generation using structured debate with adaptive confidence gating. Three specialized agents (User/Product, Technical, QA) debate before generating code, with a 95% confidence threshold to skip debate on simple tasks and a reviewer-guided debugging loop for post-generation refinement. Use when: 'generate code with multi-agent debate', 'use DebateCoder approach', 'code generation with confidence gating', 'multi-agent collaborative coding', 'debate-driven code synthesis', 'adaptive confidence code generation'."
---

# Adaptive Confidence Gating Multi-Agent Code Generation

This skill implements the DebateCoder framework for code generation: a three-agent collaborative protocol where a User Agent (product owner), Technical Agent (architect), and QA Agent (quality engineer) engage in structured pre-generation debate to produce high-quality code. An Adaptive Confidence Gating mechanism with a 95% threshold lets simple problems bypass debate entirely, while complex problems get up to 3 rounds of deliberation. After code generation, a reviewer-guided debugging loop analyzes failures and directs targeted fixes rather than blind rewrites. This approach improves code quality by 11+ percentage points over direct generation while reducing unnecessary computation on easy tasks by ~35%.

## When to Use

- When the user asks you to generate a non-trivial function or algorithm and you want to reason through it from multiple perspectives before writing code
- When a coding problem involves competing concerns (functionality vs. performance vs. robustness) that benefit from structured deliberation
- When the user asks for production-quality code with thorough edge case handling
- When you've generated code that fails tests and need a systematic debugging approach rather than blind trial-and-error
- When solving competitive programming or benchmark-style problems (HumanEval, MBPP, LeetCode) where correctness matters more than speed of response
- When the user explicitly requests a multi-agent or debate-based approach to code generation

## Key Technique

**Pre-Generation Debate with Confidence Gating:** Before writing any code, simulate three agent perspectives on the problem. Each agent independently produces a candidate plan and a confidence score (0-100) reflecting perceived task solvability. Compute the collective confidence as the mean: `Gamma = (1/|A|) * sum(c_i)`. If Gamma >= 95, the problem is simple enough to skip debate and proceed directly to code synthesis. Otherwise, run up to 3 rounds of deliberation where each agent reviews the others' plans, identifies weaknesses, and refines its own approach. This gating mechanism avoids wasting reasoning effort on straightforward problems like "return the sum of a list" while investing deeply in problems with tricky edge cases or algorithmic complexity.

**Orthogonal Pre/Post Refinement:** The framework separates concerns into two phases. Pre-generation debate resolves design disagreements (algorithm choice, edge case strategy, data structure selection) before any code is written. Post-generation refinement uses a reviewer-guided debugging loop: when code fails tests, a "Code Reviewer" role analyzes the failure triplet (problem prompt + code + error/test log) to produce a root cause analysis and fix plan—critically, it does NOT rewrite the code itself. A separate "Debugging Agent" then applies the targeted fix. This separation prevents the common failure pattern where a model blindly rewrites code and introduces new bugs.

**Role Specialization:** The three agents have distinct evaluation priorities. The User Agent focuses on functionality completeness and usability (does the code do what was asked?). The Technical Agent prioritizes technical feasibility and performance efficiency (is the algorithm correct and fast?). The QA Agent emphasizes robustness and reliability (what about edge cases, invalid inputs, boundary conditions?). This structured disagreement surfaces problems that a single-perspective approach misses.

## Step-by-Step Workflow

1. **Parse the problem statement.** Extract the function signature, input/output types, constraints, and any provided test cases. Identify ambiguities or underspecified requirements.

2. **Generate three independent plans with confidence scores.** For each agent role, produce a candidate approach and rate your confidence (0-100):
   - **User Agent plan:** Focus on whether the approach handles all stated requirements and typical use patterns. Score based on functional completeness.
   - **Technical Agent plan:** Focus on algorithmic correctness, time/space complexity, and implementation feasibility. Score based on technical soundness.
   - **QA Agent plan:** Focus on edge cases (empty input, overflow, duplicates, off-by-one), error handling, and boundary conditions. Score based on robustness.

3. **Compute collective confidence and gate.** Average the three scores. If the mean >= 95, the problem is straightforward—skip to step 6 (direct synthesis). Otherwise, proceed to deliberation.

4. **Run multi-turn deliberation (up to 3 rounds).** In each round:
   - Each agent reviews the other two agents' plans from the previous round.
   - Each agent identifies specific weaknesses in peer plans (e.g., "Technical Agent's O(n^2) approach will TLE on n=10^5", "QA Agent missed the empty-string edge case").
   - Each agent refines its own plan by incorporating the strongest elements from peers.
   - Re-evaluate confidence. If all agents converge (mean confidence >= 95 or plans are substantively identical), terminate early.

5. **Resolve remaining disagreements.** If agents still disagree after 3 rounds, prioritize by: correctness (Technical Agent) > robustness (QA Agent) > usability (User Agent). Choose the approach that satisfies the most critical concerns.

6. **Synthesize the final plan into code.** Merge the terminal plans, resolving any contradictions by keeping the most robust and functionally complete components. Write the code implementing the agreed-upon approach.

7. **Validate against test cases.** Run the generated code against all available test cases (provided examples and inferred edge cases).

8. **If tests fail, invoke the reviewer-guided debugging loop:**
   - **Reviewer step:** Analyze the triplet (problem prompt, current code, failure log). Identify the root cause. Write a specific fix plan (e.g., "Line 12: the loop should use `<=` not `<` because the range is inclusive"). Do NOT rewrite the code.
   - **Debugger step:** Apply only the targeted fix described by the reviewer. Do not refactor or restructure surrounding code.
   - Re-run tests. Repeat up to 3 cycles.

9. **Return the final code** with a brief summary of the debate outcome: which perspectives influenced the final solution and what edge cases were caught during deliberation.

## Concrete Examples

**Example 1: Simple problem (confidence gating triggers early exit)**

```
User: Write a function that returns the reverse of a string.

Agent Plans:
- User Agent (confidence: 98): Use slicing s[::-1]. Handles all string types.
- Technical Agent (confidence: 99): s[::-1] is O(n) time, O(n) space. Optimal.
- QA Agent (confidence: 96): Edge cases: empty string returns "", single char returns itself. Slicing handles both.

Collective confidence: 97.7 >= 95 → SKIP DEBATE

Synthesized output:
def reverse_string(s: str) -> str:
    return s[::-1]
```

**Example 2: Complex problem (full debate cycle)**

```
User: Write a function that finds the longest substring without repeating characters.

Round 1 Plans:
- User Agent (confidence: 70): Brute force check all substrings. Simple but complete.
- Technical Agent (confidence: 75): Sliding window with a set. O(n) time.
- QA Agent (confidence: 65): What about empty strings? Unicode? Must track window boundaries carefully.

Collective confidence: 70 < 95 → ENTER DEBATE

Round 2:
- User Agent: "Technical Agent's sliding window is better than brute force. Adopting it.
  But the window shrink logic needs care—must move left pointer past the duplicate, not just by one."
- Technical Agent: "QA Agent raises valid point about empty input. Adding explicit empty check.
  Using a dict to store last-seen index for O(1) left-pointer jumps."
- QA Agent: "Technical Agent's dict approach handles the shrink correctly.
  Adding test cases: empty string, all-same chars ('aaaa'), full-unique ('abcd'), duplicate at end."

Collective confidence: 92 → continue to Round 3

Round 3:
- All agents converge on sliding window with dict. Confidence: 96 → STOP

Synthesized output:
def length_of_longest_substring(s: str) -> int:
    seen = {}
    start = 0
    max_len = 0
    for end, char in enumerate(s):
        if char in seen and seen[char] >= start:
            start = seen[char] + 1
        seen[char] = end
        max_len = max(max_len, end - start + 1)
    return max_len
```

**Example 3: Debugging loop after test failure**

```
User: Write a function to merge two sorted lists into one sorted list.

[After debate, code generated:]
def merge_sorted(a, b):
    result = []
    i = j = 0
    while i < len(a) and j < len(b):
        if a[i] < b[j]:
            result.append(a[i])
            i += 1
        else:
            result.append(b[j])
            j += 1
    return result

Test failure: merge_sorted([1,3,5], [2,4,6]) returns [1,2,3,4,5] — missing 6.

Reviewer analysis:
  Root cause: After the while loop exits, remaining elements in the non-exhausted
  list are never appended. The loop exits when either pointer reaches the end,
  but leftover elements are discarded.
  Fix plan: After the while loop, append a[i:] and b[j:] to result.

Debugger applies fix:
def merge_sorted(a, b):
    result = []
    i = j = 0
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:      # also fixed: use <= for stability
            result.append(a[i])
            i += 1
        else:
            result.append(b[j])
            j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result

Re-test: All cases pass.
```

## Best Practices

- **Do:** Assign genuinely distinct priorities to each agent role. The User Agent should push back if the Technical Agent over-engineers. The QA Agent should raise edge cases the others overlook. Meaningful disagreement is the point.
- **Do:** Use the confidence gating honestly. If a problem is trivially simple (reverse a string, find max of a list), score it 95+ and skip debate. The gating exists to save effort, not to be ignored.
- **Do:** Keep the reviewer and debugger roles strictly separated during the debugging loop. The reviewer diagnoses; the debugger fixes. Mixing them leads to the same blind-rewrite failure pattern this technique is designed to prevent.
- **Do:** Re-evaluate confidence after each debate round and terminate early when agents converge. Three rounds is a maximum, not a target.
- **Avoid:** Letting agents agree too quickly on Round 1. The value comes from genuine critique. If all three agents propose identical approaches with 95+ confidence on a complex problem, at least one agent is not doing its job.
- **Avoid:** Using the debugging loop for more than 3 cycles. If the code still fails after 3 reviewer-debugger iterations, the approach itself is likely wrong—return to the debate phase and reconsider the algorithm.

## Error Handling

- **Agents cannot reach consensus after 3 rounds:** Prioritize correctness over elegance. Take the Technical Agent's approach as the base, layer on the QA Agent's edge case handling, and verify it satisfies the User Agent's functional requirements.
- **Confidence scores are wildly divergent (e.g., 30 vs 95):** This signals a fundamental disagreement about problem complexity. The low-confidence agent likely sees an issue the others missed. Explicitly address that agent's concerns before proceeding.
- **Debugging loop fixes introduce new failures:** Roll back to the pre-fix version and have the reviewer re-analyze with the additional failure information. Do not stack fixes on top of broken fixes.
- **Problem is ambiguous or underspecified:** The User Agent should flag this in Round 1. Ask the user for clarification rather than debating interpretations of unclear requirements.

## Limitations

- **Overhead on trivial problems:** Even with confidence gating, generating three plans for `return a + b` is wasteful. For truly trivial one-liners, skip this framework entirely.
- **Not a substitute for real testing:** The debate improves reasoning about edge cases but does not replace running actual tests. Always validate generated code against concrete test cases.
- **Diminishing returns beyond 3 agents:** The paper uses exactly 3 roles. Adding more agents (e.g., Security Agent, Performance Agent) increases deliberation cost without proportional quality gains. Stick to three.
- **Best suited for self-contained functions:** This technique works well for algorithmic problems with clear inputs/outputs. It is less effective for tasks requiring deep codebase context, multi-file refactoring, or UI work where "correctness" is subjective.
- **Debate quality scales with problem understanding:** If the underlying model cannot reason about the algorithm at all, debate among three equally confused agents will not produce good results. The technique amplifies existing reasoning ability rather than creating it from nothing.

## Reference

[Adaptive Confidence Gating in Multi-Agent Collaboration for Efficient and Optimized Code Generation](https://arxiv.org/abs/2601.21469v1) — Zhang et al. (2026). Look for: Section 3 (DebateCoder framework architecture), the confidence formula `Gamma = mean(c_i)` with tau=95 threshold, Table 1 (Pass@1 results showing 70.12% on HumanEval), and Table 3 (ablation study showing +4.87% from debate alone, +7.92% from the debugging loop).