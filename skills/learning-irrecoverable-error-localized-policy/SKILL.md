---
name: "learning-irrecoverable-error-localized-policy"
description: "Debug multi-step tool-using agent pipelines by localizing the first irrecoverable error via binary-search rollback, then concentrating fixes on that critical step and its downstream suffix. Use when: 'find where my agent pipeline first breaks', 'debug this tool chain failure', 'locate the root cause in my multi-step workflow', 'which step killed this reasoning chain', 'binary search for the breaking step', 'error-localize my agent trajectory'."
---

# Error-Localized Debugging for Multi-Step Tool-Using Pipelines

This skill applies the core insight from Error-Localized Policy Optimization (ELPO): in any multi-step agent trajectory that interleaves reasoning with tool calls (APIs, code execution, search, calculators), a single **irrecoverable step** -- the earliest point after which no sequence of subsequent actions can recover a correct outcome -- dominates whether the pipeline succeeds or fails. Rather than treating all steps equally when diagnosing failures, this skill teaches Claude to **binary-search for that first irrecoverable step**, attribute blame hierarchically, and concentrate corrective effort on the critical step and everything downstream of it.

## When to Use

- When a multi-step agent pipeline (LangChain, CrewAI, AutoGen, custom orchestrator) produces a wrong final answer and the user needs to find which step is the root cause.
- When a tool-integrated reasoning chain fails -- e.g., a chain that interleaves LLM reasoning with code execution, API calls, or database queries -- and the user wants to know exactly where recovery became impossible.
- When debugging a failing agentic workflow that has 5+ steps and a naive "check each step" approach is too expensive or slow.
- When the user wants to build retry/fallback logic and needs to know the **minimal suffix** of the pipeline to re-execute after fixing the root cause.
- When designing test harnesses for multi-step LLM pipelines that need fine-grained step-level pass/fail attribution rather than end-to-end outcome checks.
- When refactoring a long reasoning chain and deciding which steps to make more robust vs. which are reliably correct.

## Key Technique

**The Irrecoverability Insight.** In a trajectory of N steps (interleaved reasoning + tool calls), errors fall into two classes: *recoverable* errors that later steps can compensate for (e.g., a rounding mistake that a calculator corrects), and *irrecoverable* errors after which no valid continuation can reach the correct outcome (e.g., querying the wrong database table so all downstream analysis uses wrong data). ELPO's core finding is that the **first irrecoverable step** is the single highest-leverage point for improvement. Every step before it is either correct or recoverable; every step after it inherits the damage.

**Binary-Search Rollout Localization.** To find this step efficiently, ELPO uses binary search over the trajectory. Given a failed N-step trace, probe the midpoint: re-execute the suffix from step N/2 with the original prefix held fixed. If re-execution can produce a correct outcome, the irrecoverable step is in the first half; otherwise it is in the second half (or at the midpoint). Recurse until the critical step is isolated. This costs O(log N) re-executions instead of O(N), which matters when each step involves expensive tool calls.

**Hierarchical Blame and Concentrated Fixes.** Once the irrecoverable step t* is found, attribute blame hierarchically: the step itself gets the strongest negative signal, its immediate downstream dependents get proportionally less, and steps before t* that are recoverable get no blame. Concentrate corrective effort -- prompt rewrites, tool-call parameter fixes, retry logic, guard-rails -- on step t* and its suffix. Do not waste effort hardening steps that are already reliable.

## Step-by-Step Workflow

1. **Capture the full trajectory.** Collect the complete step-by-step execution log of the failing pipeline: every reasoning step, every tool invocation (with inputs and outputs), and the final outcome. Label each step with an index (1..N) and its type (reasoning | tool-call).

2. **Confirm the trajectory actually fails.** Verify the final output is incorrect against the known expected outcome. If the pipeline sometimes succeeds, collect both passing and failing traces for the same input to make rollout comparison easier.

3. **Binary-search for the first irrecoverable step.** Set `lo=1, hi=N`. At each iteration, pick `mid = (lo+hi)//2`. Mentally or actually re-execute the suffix from step `mid` onward, assuming all steps 1..mid-1 executed correctly. If the suffix *can* produce a correct result, the irrecoverable step is in `[lo, mid-1]`; set `hi=mid-1`. If the suffix *cannot* recover, the irrecoverable step is in `[mid, hi]`; set `lo=mid`. Stop when `lo == hi`; that is step `t*`.

4. **Validate the localization.** Confirm that re-executing from step `t*` onward (with a corrected version of step `t*`) produces a correct final output, while re-executing from step `t*+1` onward (with the original broken step `t*`) cannot. This double-check prevents false localization.

5. **Classify the error type at t*.** Determine what went wrong: wrong tool selected, wrong parameters passed to correct tool, flawed reasoning that committed to an incorrect intermediate conclusion, hallucinated data, or a tool returning an unexpected result the reasoning failed to handle.

6. **Attribute downstream damage.** For each step after t*, identify whether it is *independently broken* or *broken only because it inherited bad state from t\**. Steps that are independently broken need their own fixes; steps that are merely propagating the t* error will self-heal once t* is fixed.

7. **Design the minimal fix.** Write the corrected version of step t* and, if needed, any independently broken downstream steps. Do not modify steps before t* unless they contain recoverable errors you want to harden opportunistically.

8. **Add a guard-rail at t\*.** If the error at t* is a recurring pattern (e.g., the LLM tends to pick the wrong API endpoint), add a validation check immediately after t* -- an assertion, a schema check, or a retry-with-different-parameters block -- so future executions catch the same class of error before it propagates.

9. **Re-run the full pipeline end-to-end.** Verify the fix produces the correct final outcome. If it fails again, re-apply the binary-search procedure to the new trace; the new irrecoverable step may be different.

10. **Document the localization result.** Record which step was irrecoverable, what the error class was, and what fix was applied. Over time, this builds a map of which pipeline stages are fragile, guiding architectural improvements.

## Concrete Examples

**Example 1: Debugging a math-solving agent that uses a code interpreter**

```
User: My math agent gets the wrong answer for "Find all primes p such that
p^2 + 2 is also prime." The agent has 6 steps: (1) parse problem,
(2) reason about structure, (3) write Python to brute-force check,
(4) execute code, (5) interpret results, (6) format answer. It outputs
"2, 3, 5" but the correct answer is just "3". Where does it break?

Approach:
1. Capture trajectory:
   Step 1: Parse -- correctly identifies the task.
   Step 2: Reason -- states "p must be odd or 2, check small primes."
   Step 3: Write code -- generates: `for p in range(2,100): if
           is_prime(p) and is_prime(p**2 + 2): print(p)`
   Step 4: Execute -- code prints "2, 3, 5".
   Step 5: Interpret -- "The primes are 2, 3, 5."
   Step 6: Format -- "Answer: 2, 3, 5."

2. Binary search: mid = 3. Re-execute from step 3 onward with correct
   prefix. The code itself is logically correct, so the output "2, 3, 5"
   is what brute force actually produces. Wait -- is the expected answer
   wrong? Check: 2^2+2=6 (not prime), 3^2+2=11 (prime), 5^2+2=27 (not
   prime). So the code has a bug.

3. Narrow: mid=3 is suspicious. Check is_prime function in the code.
   The agent defined is_prime(n) with `if n < 2: return False` and
   `for i in range(2, n): ...` -- correct but slow. The real bug:
   the agent's code actually printed the results correctly as "3" in
   execution, but step 5 hallucinated "2, 3, 5" from the reasoning
   in step 2 instead of reading the actual code output.

4. Localization: t* = Step 5 (interpret results). The code output was
   correct ("3"), but the LLM ignored it and relied on its own flawed
   reasoning from step 2.

Fix: Add a guard-rail after step 4 that injects the literal code output
into the prompt for step 5, with an instruction: "Base your answer
ONLY on the code output above, not on prior reasoning."
```

**Example 2: Debugging a retrieval-augmented QA pipeline**

```
User: My RAG pipeline answers "Who invented the transistor?" with
"Lee De Forest" instead of "Bardeen, Brattain, and Shockley." The
pipeline: (1) query rewriting, (2) vector search, (3) reranking,
(4) context assembly, (5) answer generation. Find the breaking step.

Approach:
1. Binary search: mid = 3 (reranking). Check if the correct documents
   about Bardeen/Brattain/Shockley are present in the reranker input.
   They are -- vector search retrieved them at positions 3 and 7.

2. Check reranker output: the reranker pushed those documents to
   positions 12 and 15, below the top-k=5 cutoff. The top results
   are about vacuum tubes and Lee De Forest (inventor of the triode).

3. Localization: t* = Step 3 (reranking). The reranker confused
   "transistor" with "triode" due to semantic similarity and promoted
   irrelevant documents.

4. Validate: manually placing the correct documents in the top-5 and
   re-running steps 4-5 produces "Bardeen, Brattain, and Shockley."

Fix: Adjust the reranker to use a cross-encoder that distinguishes
"transistor" from "triode," or add a keyword filter requiring the
exact term "transistor" in reranked results.
```

**Example 3: Localizing a failure in an API-orchestration agent**

```
User: My travel-booking agent fails to book a hotel. Steps:
(1) parse user request, (2) search flights, (3) book flight,
(4) extract destination city from booking confirmation,
(5) search hotels in that city, (6) book hotel. Step 6 returns
"no hotels found." Debug this.

Approach:
1. Binary search: mid = 3. If we re-execute from step 3 onward with
   correct prefix, does it work? Step 3 books the flight successfully.
   Step 4 extracts the city. Check step 4 output: it extracted
   "LAX" (airport code) instead of "Los Angeles" (city name).

2. Step 5 searches hotels in "LAX" -- the hotel API expects city
   names, not airport codes, so it returns empty results.

3. Localization: t* = Step 4. The extraction step produced an airport
   code instead of a city name. Steps 5-6 are not independently broken;
   they will work correctly given the right city name.

Fix: Update the extraction prompt at step 4 to specify "Extract the
destination CITY NAME (not airport code)" or add a post-processing
step that maps airport codes to city names.
```

## Best Practices

- **Do:** Always binary-search rather than linearly scanning. A 10-step pipeline needs at most 4 probes instead of 10.
- **Do:** Distinguish recoverable from irrecoverable errors. A typo in a search query that the reranker compensates for is recoverable -- don't waste effort fixing it.
- **Do:** Validate your localization with a double-check: confirm the suffix fails with the broken step and succeeds with the fixed step.
- **Do:** Record error patterns over many failing traces. If step 4 is irrecoverable in 80% of failures, that step needs architectural hardening, not just prompt tweaks.
- **Avoid:** Blaming all steps equally for a failure. This leads to over-engineering early steps that are already correct.
- **Avoid:** Fixing only the final step because it produced the wrong output. The final step is often a victim, not the cause.
- **Avoid:** Re-running the entire pipeline from scratch when only the suffix from t* onward needs re-execution. This wastes tool-call budget.

## Error Handling

- **Binary search gives ambiguous results (suffix sometimes passes, sometimes fails).** The step at the boundary is *partially* irrecoverable -- it fails under some rollout continuations but not others. Run multiple rollouts at that step to estimate the recovery probability. If recovery rate is below 50%, treat it as irrecoverable.
- **Multiple irrecoverable steps exist.** The binary search finds the *first* one. After fixing it, re-run the procedure to find the next. Iterate until the pipeline passes.
- **Tool calls are non-deterministic or have side effects.** If a tool call mutates external state (e.g., writes to a database), you cannot naively re-execute the suffix. Use mocked/cached tool responses for the rollout probes, or run against a staging environment.
- **The trajectory is too short (2-3 steps).** Binary search degenerates to linear scan, which is fine -- just check each step directly.

## Limitations

- This technique assumes you can re-execute (or reason about re-executing) suffixes of the trajectory. If tool calls are irreversible and cannot be mocked, localization requires manual inspection.
- It finds the first irrecoverable step, not all bugs. A trajectory may contain multiple recoverable errors that degrade quality without being individually irrecoverable.
- For stochastic pipelines where the same input produces different traces each run, you need multiple rollouts per binary-search probe to get reliable localization, increasing cost.
- The method is designed for sequential pipelines. For DAG-structured workflows with parallel branches, you need to binary-search each branch independently and then check cross-branch interactions.
- It does not help when the failure is in the input specification itself (garbage in, garbage out) rather than in the pipeline's execution.

## Reference

- **Paper:** [Learning from the Irrecoverable: Error-Localized Policy Optimization for Tool-Integrated LLM Reasoning](https://arxiv.org/abs/2602.09598v1) (Liang et al., 2026). Key sections: binary-search rollout tree construction (Sec. 3.1), hierarchical advantage attribution (Sec. 3.2), and error-localized adaptive clipping (Sec. 3.3). Look for the ablation study showing that localizing the first irrecoverable step captures 70-90% of the improvement signal compared to full trajectory-level feedback.