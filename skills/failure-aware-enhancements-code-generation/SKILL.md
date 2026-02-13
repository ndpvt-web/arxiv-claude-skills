---
name: "failure-aware-enhancements-code-generation"
description: >
  Diagnose why generated code fails and apply the right fix strategy (self-critique, RAG, multi-model, or progressive prompting) based on a data-driven decision framework from empirical research on 25 GitHub projects.
  Trigger phrases: "my generated code doesn't work", "fix this code generation failure",
  "why does this code keep failing", "help me debug LLM-generated code",
  "improve code generation quality", "the AI-generated code is wrong"
---

# Failure-Aware Enhancements for Code Generation

This skill equips Claude to systematically diagnose *why* generated code fails and select the most effective repair strategy based on failure type rather than trial-and-error. Drawing from an empirical study of 25 GitHub projects (Shen, Peng & Owen, 2026), the approach classifies code generation failures into distinct categories -- logic errors, missing edge cases, integration issues, and specification gaps -- then maps each category to the enhancement method with the highest empirical success rate: self-critique for logic errors, RAG for implementation pattern gaps, multi-model reasoning for low-confidence outputs, and progressive prompting for unclear specifications.

## When to Use

- When code you generated fails tests and you need to determine the root cause before retrying
- When a user says "this code doesn't handle X correctly" and you need to decide between reviewing the logic vs. looking up documentation
- When initial code generation misses requirements and you need a structured approach to fill the gaps
- When integrating with external services (APIs, databases, third-party libraries) where self-critique alone has 0% success rate
- When a user asks you to generate a complex feature and you want to proactively avoid common failure modes
- When repeated generation attempts keep producing the same errors, signaling you need to change strategy rather than retry

## Key Technique

**The core insight**: not all code generation failures respond to the same fix. The study found that progressive prompting raises average task completion from 80.5% to 96.9% (Cohen's d=1.63, p<0.001), but the remaining failures require targeted interventions. Self-critique works well for code-reviewable logic errors but achieves 0% improvement on external service integration failures. RAG achieves the highest completion rate across all failure types with superior efficiency. The wrong enhancement wastes tokens and time.

**The decision framework** maps failure patterns to methods:
1. *Logic errors* (wrong algorithm, off-by-one, incorrect conditionals) → **Self-critique**: re-read the code against the specification, identify the discrepancy, generate a targeted fix.
2. *Specification gaps* (ambiguous requirements, missing details) → **Progressive prompting**: break the problem into smaller, sequential prompts that clarify each requirement incrementally.
3. *Implementation pattern gaps* (unfamiliar API usage, library idioms, framework conventions) → **RAG**: retrieve documentation, examples, or prior solutions to ground the generation in correct patterns.
4. *Low-confidence or multi-faceted failures* (unclear root cause, multiple interacting bugs) → **Multi-model reasoning**: generate alternative implementations, compare outputs, and synthesize the strongest solution.

**Why this matters**: developers typically default to "just retry with a better prompt," which the study shows is suboptimal. Matching the enhancement to the failure type reduces wasted iterations and produces higher-quality code on fewer attempts.

## Step-by-Step Workflow

1. **Generate initial code** using progressive prompting: decompose the requirement into ordered sub-tasks (data model → core logic → edge cases → integration → output formatting) and generate code for each sequentially, feeding prior outputs as context.

2. **Run validation** against available tests, type checks, or manual inspection. Collect all errors, warnings, and unmet requirements into a failure list.

3. **Classify each failure** into one of four categories:
   - **Logic error**: The code runs but produces wrong results. Symptoms: test assertions fail on correctness, off-by-one bugs, wrong conditional branches.
   - **Specification gap**: The code doesn't address part of the requirement. Symptoms: missing functionality, incomplete handling, ambiguous behavior.
   - **Pattern gap**: The code misuses an API, library, or framework convention. Symptoms: runtime errors from incorrect method signatures, deprecated usage, wrong configuration format.
   - **Low-confidence / compound**: Multiple interacting issues or unclear root cause. Symptoms: cascading errors, inconsistent behavior across inputs.

4. **Apply self-critique for logic errors**: Re-read the generated code line by line against the requirement. State explicitly what the code does vs. what it should do. Generate a minimal, targeted diff that fixes only the identified discrepancy. Do not rewrite surrounding code.

5. **Apply progressive prompting for specification gaps**: Identify the missing requirement. Formulate a focused follow-up prompt that asks specifically about the unaddressed aspect. Generate the additional code and integrate it into the existing solution.

6. **Apply RAG for pattern gaps**: Search documentation, codebases, or known examples for the correct usage pattern. Use the retrieved context to rewrite only the misused API calls or library interactions. Verify against official docs if accessible.

7. **Apply multi-model reasoning for compound failures**: Generate 2-3 alternative implementations of the failing section using different approaches. Compare outputs to identify which aspects each gets right. Synthesize the strongest elements into a single solution.

8. **Re-validate after each fix**: Run the same validation from step 2. If new failures appear, classify and address them. Track which enhancement method resolved which failure to build a feedback loop.

9. **Document the failure-fix mapping**: For each resolved failure, note the category and the method that worked. This becomes a project-specific decision guide for future iterations.

10. **Finalize with integration testing**: Once all individual failures are resolved, run end-to-end validation to catch interaction effects between the fixes.

## Concrete Examples

**Example 1: Logic error in sorting algorithm**

```
User: "Write a function that returns the top K most frequent elements from an array."

Initial generation:
def top_k_frequent(nums, k):
    freq = {}
    for n in nums:
        freq[n] = freq.get(n, 0) + 1
    sorted_freq = sorted(freq.items(), key=lambda x: x[1])
    return [x[0] for x in sorted_freq[:k]]

Failure: Returns LEAST frequent elements (sorted ascending, takes first k).

Classification: Logic error -- code runs but produces wrong results.

Enhancement: Self-critique.

Reasoning: The sort order is ascending but we need descending.
The slice takes [:k] which grabs the lowest values.
Fix: Either reverse the sort or take [-k:].

Fixed code:
def top_k_frequent(nums, k):
    freq = {}
    for n in nums:
        freq[n] = freq.get(n, 0) + 1
    sorted_freq = sorted(freq.items(), key=lambda x: x[1], reverse=True)
    return [x[0] for x in sorted_freq[:k]]
```

**Example 2: External API integration failure**

```
User: "Add Stripe payment processing to my checkout endpoint."

Initial generation uses stripe.Charge.create() -- deprecated since 2022.

Failure: stripe.error.InvalidRequestError -- Charges API no longer
recommended, PaymentIntents required.

Classification: Pattern gap -- incorrect API usage pattern.

Enhancement: RAG (self-critique would fail here; the study shows
0% improvement from self-critique on external service integration).

Action:
1. Retrieve current Stripe docs for PaymentIntents API
2. Identify correct method: stripe.PaymentIntent.create()
3. Note required parameters: amount, currency, payment_method, confirm
4. Rewrite only the payment processing section:

Fixed code:
intent = stripe.PaymentIntent.create(
    amount=amount_cents,
    currency="usd",
    payment_method=payment_method_id,
    confirm=True,
    automatic_payment_methods={"enabled": True, "allow_redirects": "never"},
)
```

**Example 3: Underspecified requirements with compound failures**

```
User: "Build a caching layer for my database queries."

Initial generation: Simple dict-based cache with no expiration.

Failures (multiple):
- F1: No TTL / expiration (specification gap)
- F2: No thread safety (specification gap)
- F3: Unbounded memory growth (logic error)
- F4: Cache key doesn't account for query parameters (logic error)

Approach -- address each by category:

Step 1 (Progressive prompting for F1, F2):
  "The cache needs TTL-based expiration. What should the default TTL be?"
  "This runs in a multi-threaded web server. Add thread-safe access."

Step 2 (Self-critique for F3, F4):
  F3: "The cache grows without bound. Add an LRU eviction policy
       with a configurable max size."
  F4: "Cache key is just the query string. It must include
       parameterized values: key = hash(query + str(params))."

Step 3: Validate the combined solution handles all four failures.

Output: Thread-safe LRU cache with TTL expiration and
parameter-aware cache keys.
```

## Best Practices

**Do:**
- Classify failures before attempting fixes. The single most impactful step is correct diagnosis.
- Use progressive prompting as your default first-pass strategy -- it handles 96.9% of cases.
- Prefer RAG over self-critique for any failure involving external libraries, APIs, or services. Self-critique has 0% success on integration issues.
- Apply fixes incrementally and re-validate after each one. Batching fixes obscures which changes helped.

**Avoid:**
- Do not retry the same prompt hoping for a different result. If generation failed, the failure type determines the fix method, not more attempts.
- Do not apply self-critique to pattern gaps. Reviewing code against a specification cannot surface information about correct API usage that wasn't in the original context.
- Do not use multi-model reasoning as a first resort. It is the most expensive strategy and should be reserved for genuinely ambiguous, compound failures.
- Do not skip the classification step. Applying the wrong enhancement wastes tokens and may introduce new errors.

## Error Handling

| Situation | Response |
|-----------|----------|
| Self-critique identifies no discrepancy but tests still fail | Reclassify as pattern gap or compound failure; switch to RAG or multi-model |
| RAG returns no relevant documentation | Fall back to multi-model reasoning; generate alternatives and test empirically |
| Progressive prompting produces contradictory sub-task outputs | Consolidate requirements into a single coherent specification before regenerating |
| Fix introduces new failures | Classify the new failures independently; do not assume they share the original category |
| All enhancement methods fail | The requirement may exceed single-generation capability; recommend decomposing into separate modules with clear interfaces |

## Limitations

- **Failure classification requires judgment**: The four categories are not always cleanly separable. A wrong API call might look like a logic error until you realize the method signature changed. When uncertain, default to RAG.
- **RAG depends on retrieval quality**: If documentation is unavailable, outdated, or the codebase is proprietary, RAG effectiveness drops. In closed-source contexts, multi-model reasoning may be the only viable fallback.
- **The study used 25 GitHub projects**: While statistically significant, the findings may not generalize to all domains (embedded systems, GPU kernels, highly domain-specific DSLs).
- **Progressive prompting adds latency**: The sequential sub-task approach is slower than direct prompting. For simple, well-specified tasks, direct generation is sufficient.
- **Self-critique cannot surface unknown unknowns**: It only catches errors visible by comparing code to stated requirements. Unstated assumptions or implicit requirements will not be found.

## Reference

Shen, J., Peng, Z., & Owen, L. (2026). *Failure-Aware Enhancements for Large Language Model (LLM) Code Generation: An Empirical Study on Decision Framework*. SANER 2026. [arXiv:2602.02896](https://arxiv.org/abs/2602.02896v1) -- Read for: the failure taxonomy (Section 3), decision framework mapping failures to enhancement methods (Section 4), and empirical results showing self-critique's 0% success rate on integration failures vs. RAG's cross-category effectiveness (Section 5).