---
name: "system-12-synergy-dynamic"
description: |
  Dynamically blend fast (System 1) and deep-reasoning (System 2) approaches per query using the DAMI framework's interpolation strategy. Instead of always defaulting to full chain-of-thought or always giving quick answers, this skill estimates per-query reasoning intensity and configures cognitive depth accordingly.
  Trigger phrases:
  - "Solve this efficiently — only think hard if needed"
  - "Use adaptive reasoning depth for this batch of problems"
  - "Route these questions by difficulty"
  - "Blend fast and slow thinking for this task"
  - "Optimize reasoning effort across these queries"
  - "Don't overthink simple questions but reason deeply on hard ones"
---

# System 1 & 2 Synergy via Dynamic Model Interpolation (DAMI)

This skill enables Claude to dynamically calibrate its reasoning depth per query by applying the DAMI framework's core insight: rather than uniformly applying maximum chain-of-thought reasoning or uniformly giving terse answers, estimate each query's **Reasoning Intensity** and adapt cognitive effort accordingly. Simple queries get fast, direct answers (System 1); ambiguous or complex queries receive structured deliberation (System 2); and intermediate queries get proportional effort. The result is higher aggregate accuracy than always-deep-thinking, with dramatically lower total token cost.

## When to Use

- When the user submits a **batch of mixed-difficulty tasks** (e.g., code reviews spanning trivial lint fixes and deep architectural issues)
- When asked to **triage and solve** a set of problems where some are straightforward and others require multi-step reasoning
- When building **agent pipelines** that need to decide how much reasoning budget to allocate per sub-task
- When the user explicitly asks to **optimize reasoning cost** while maintaining accuracy
- When implementing **routing logic** that dispatches queries to different processing depths
- When designing **prompt chains** that should short-circuit on easy inputs and elaborate on hard ones
- When evaluating whether a coding problem needs quick pattern-matching or careful step-by-step analysis

## Key Technique

**The core insight:** The DAMI paper (Yang et al., 2026) demonstrates that linearly interpolating between an Instruct model (fast, System 1) and a Thinking model (deliberative, System 2) using `theta_lambda = (1 - lambda) * theta_instruct + lambda * theta_thinking` produces a **convex, monotonic Pareto frontier** — meaning every interpolation point is an optimal accuracy-efficiency tradeoff. This works because the two models share representation continuity (their internal feature spaces are smoothly connected) and structural connectivity (parameter neighborhoods map to behavior neighborhoods).

**For Claude's application:** We cannot interpolate model weights at inference time, but we can operationalize the same principle by treating reasoning depth as a continuous dial. The key variable is **lambda(q)** — the per-query Reasoning Intensity — which we estimate using two complementary methods: (1) **Confidence-based estimation**: run a fast initial pass; if confidence is high, emit the answer directly (low lambda); if confidence is low or the fast answer is ambiguous, escalate to deep reasoning (high lambda). (2) **Difficulty heuristics**: classify the query's structural complexity (token count, nested dependencies, domain-specificity, ambiguity markers) to pre-assign a lambda before any generation.

**Why this beats uniform strategies:** Always thinking deeply wastes tokens on easy queries and can introduce overthinking errors. Always answering quickly misses hard problems. DAMI's dynamic approach achieved higher accuracy than the Thinking-only model on math benchmarks while using significantly fewer reasoning tokens overall, because it concentrated reasoning budget where it mattered.

## Step-by-Step Workflow

1. **Receive the query or batch of queries.** If a single query, proceed to step 2. If a batch, process each query independently through steps 2-7, then aggregate results.

2. **Run a fast System 1 assessment.** For each query, generate a brief internal assessment: What is the likely answer? How confident am I? Capture this as a confidence signal (high / medium / low) and a tentative answer.

3. **Estimate Reasoning Intensity lambda(q).** Score the query on these dimensions (each 0-1, then average):
   - **Confidence gap**: If the fast answer feels uncertain or could plausibly be wrong, score high (0.7-1.0). If confident, score low (0.0-0.3).
   - **Structural complexity**: Count nested logic steps, conditional branches, or multi-part requirements. Single-step = 0.1, multi-step = 0.5, deeply nested = 0.9.
   - **Domain specificity**: Generic coding patterns = 0.1, specialized algorithms or tricky edge cases = 0.7+.
   - **Ambiguity**: Clear specification = 0.1, underspecified or conflicting requirements = 0.8+.

4. **Map lambda(q) to a reasoning strategy:**
   - **lambda < 0.3 (System 1):** Give a direct, concise answer. No chain-of-thought. Emit code or answer immediately.
   - **0.3 <= lambda < 0.6 (Blended):** Provide a brief explanation of the approach, then the answer. Show key reasoning steps but skip obvious ones.
   - **lambda >= 0.6 (System 2):** Full deliberative reasoning. Break down the problem, consider edge cases, verify the solution, and show work.

5. **Execute the chosen strategy.** Generate the response at the selected depth. For System 1 responses, prioritize speed and directness. For System 2 responses, prioritize correctness verification and explicit reasoning.

6. **Post-hoc confidence check.** After generating the response, briefly verify: Does the answer hold up under a quick sanity check? If a System 1 answer fails the sanity check, escalate to System 2 and regenerate.

7. **Emit the final answer with a reasoning-depth tag.** Optionally annotate the response with `[S1]`, `[S1+S2]`, or `[S2]` so the user can see which cognitive mode was applied and calibrate their trust accordingly.

8. **For batch processing, report summary statistics.** Show how many queries were routed to each depth level and flag any that were escalated from System 1 to System 2 during the confidence check.

## Concrete Examples

**Example 1: Mixed-difficulty code review batch**

```
User: Review these three functions for bugs:

def add(a, b): return a + b

def binary_search(arr, target):
    lo, hi = 0, len(arr)
    while lo < hi:
        mid = (lo + hi) // 2
        if arr[mid] < target: lo = mid + 1
        elif arr[mid] > target: hi = mid
        else: return mid
    return -1

def merge_intervals(intervals):
    intervals.sort()
    merged = [intervals[0]]
    for s, e in intervals[1:]:
        if s <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], e)
        else:
            merged.append([s, e])
    return merged
```

Approach:
1. Fast assessment of `add`: trivially correct. lambda = 0.05 -> System 1.
2. Fast assessment of `binary_search`: standard pattern but off-by-one errors are common.
   lambda = 0.55 -> Blended.
3. Fast assessment of `merge_intervals`: mutation of tuples vs lists, empty-input edge case.
   lambda = 0.65 -> System 2.

Output:
- `add` [S1]: No issues. Correct as written.
- `binary_search` [S1+S2]: Implementation is correct for a sorted array.
  Returns -1 when not found, uses half-open interval `[lo, hi)` properly.
  One note: `lo + hi` can overflow in languages with fixed-width integers,
  but this is Python, so no issue here. Correct.
- `merge_intervals` [S2]: Bug found. If `intervals` is empty, line
  `merged = [intervals[0]]` raises IndexError. Also, `.sort()` works on
  tuples, but `merged[-1][1] = max(...)` requires lists, not tuples.
  If the input is a list of tuples, this will raise TypeError.
  Fix: add an empty check and convert to lists.

Summary: 1 S1, 1 Blended, 1 S2. One escalation-worthy bug found.
```

**Example 2: Routing reasoning depth in an agent pipeline**

```
User: I'm building an agent that processes user support tickets. Some are
simple ("reset my password") and some are complex ("my API returns 500
intermittently when load exceeds 1k RPS"). Help me implement adaptive
reasoning depth.

Approach:
1. Classify the request as a pipeline design problem. lambda = 0.7 -> System 2.
2. Design a two-stage routing architecture using DAMI principles.

Output:
Implement a TicketRouter with these components:

Stage 1 — Fast Classification (System 1):
  - Run the ticket through a lightweight classifier (keyword match + embeddings)
  - Assign initial lambda based on:
    * Known-pattern tickets (password reset, billing inquiry): lambda = 0.1
    * Tickets mentioning errors, intermittent issues, performance: lambda = 0.6
    * Tickets with code snippets, stack traces, multi-system: lambda = 0.9

Stage 2 — Confidence-Gated Escalation:
  - For lambda < 0.3: Generate templated response, verify with regex match
  - For 0.3 <= lambda < 0.7: Generate response with brief reasoning,
    run confidence check against known solutions DB
  - For lambda >= 0.7: Full diagnostic reasoning chain — reproduce scenario,
    check logs, cross-reference documentation, propose root cause

Code skeleton:

    def route_ticket(ticket: str) -> Response:
        # Stage 1: Fast assessment
        fast_answer, confidence = system1_classify(ticket)
        lam = estimate_lambda(confidence, complexity_score(ticket))

        if lam < 0.3 and confidence > 0.9:
            return emit_response(fast_answer, depth="S1")

        if lam < 0.7:
            answer = reason_briefly(ticket, fast_answer)
            if sanity_check(answer):
                return emit_response(answer, depth="S1+S2")
            lam = 0.8  # escalate

        return emit_response(deep_reason(ticket), depth="S2")
```

**Example 3: Optimizing a batch of math/logic problems**

```
User: Solve these efficiently — don't overthink the easy ones:
1. What is 17 * 23?
2. Find all primes below 50.
3. Prove that sqrt(2) is irrational.
4. What is the derivative of x^3?

Approach:
1. Query 1: arithmetic. lambda = 0.05 -> S1.
2. Query 2: enumeration, well-known. lambda = 0.15 -> S1.
3. Query 3: proof, requires structured argument. lambda = 0.85 -> S2.
4. Query 4: basic calculus rule. lambda = 0.05 -> S1.

Output:
1. [S1] 391
2. [S1] 2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47
3. [S2] Proof by contradiction: Assume sqrt(2) = p/q in lowest terms.
   Then 2q^2 = p^2, so p^2 is even, so p is even. Write p = 2k.
   Then 2q^2 = 4k^2, so q^2 = 2k^2, so q is even. Contradiction:
   p and q share factor 2, violating lowest-terms assumption. QED.
4. [S1] 3x^2

Batch summary: 3x S1 (direct), 1x S2 (full proof). Total reasoning
budget concentrated on query 3.
```

## Best Practices

- **Do:** Run a genuine fast assessment first. The confidence signal from the initial pass is the most valuable routing input — don't skip it to save time.
- **Do:** Escalate aggressively when the sanity check fails. A wrong fast answer costs more than a slow correct one. The post-hoc confidence check in step 6 is non-optional.
- **Do:** Annotate outputs with depth markers (`[S1]`, `[S1+S2]`, `[S2]`) so users can see where reasoning effort was spent and can request re-analysis at a different depth.
- **Do:** When processing batches, sort by estimated lambda and handle easy items first — this builds context and catches patterns that might help with harder items.
- **Avoid:** Treating lambda as binary (always 0 or 1). The blended middle zone (0.3-0.6) is where the most value lies — brief reasoning catches many errors that pure System 1 misses.
- **Avoid:** Over-indexing on query length as a complexity proxy. Short queries can be deeply ambiguous ("Is this code safe?") and long queries can be straightforward (verbose but clear specifications).

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| System 1 gives wrong answer confidently | Sanity check in step 6 fails | Escalate to System 2; flag the discrepancy to the user |
| Lambda estimation is miscalibrated | User feedback indicates easy things got S2 or hard things got S1 | Adjust dimension weights in step 3; increase weight on the dimension that was underestimated |
| Batch contains adversarial/trick questions | Fast pass produces suspiciously clean answers for all items | Apply a blanket lambda floor of 0.4 for the batch and re-evaluate |
| Deep reasoning loops without converging | Token budget exceeded in System 2 mode | Cap reasoning at a fixed step count; emit best-so-far answer with a confidence caveat |

## Limitations

- **No actual weight interpolation:** Claude cannot blend model parameters at runtime. This skill approximates DAMI's effect through behavioral strategy selection, which is a coarser control than true parameter interpolation.
- **Calibration is heuristic:** The four-dimension lambda estimation (confidence, complexity, domain, ambiguity) is an approximation. True DAMI uses a trained estimator or inter-model discrepancy, neither of which is directly available.
- **Not suited for all-hard workloads:** If every query genuinely requires deep reasoning, the routing overhead adds cost with no benefit. This skill shines on mixed-difficulty workloads.
- **Single-model constraint:** The paper's method interpolates between two separate model checkpoints. Operating within a single model, we can only vary reasoning *strategy*, not underlying capability.
- **Confidence estimation is imperfect:** System 1 can be confidently wrong, especially on problems that pattern-match to familiar templates but have subtle twists.

## Reference

**Paper:** [System 1&2 Synergy via Dynamic Model Interpolation](https://arxiv.org/abs/2601.21414v1) — Yang et al., 2026. Key finding: linear interpolation between Instruct and Thinking model checkpoints produces a convex Pareto frontier of accuracy vs. efficiency, and a per-query Reasoning Intensity estimator can dynamically navigate this frontier to beat both endpoints.