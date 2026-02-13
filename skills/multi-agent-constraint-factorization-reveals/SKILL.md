---
name: "multi-agent-constraint-factorization-reveals"
description: "Orchestrate multi-agent LLM pipelines using constraint factorization -- decomposing complex requirements into separate constraint-enforcement agents that iteratively project a shared solution toward feasibility. Use when: 'build a multi-agent pipeline', 'decompose this problem into agent constraints', 'set up constraint-based agent orchestration', 'factorize requirements across agents', 'iterative multi-agent refinement loop', 'cyclic projection agent workflow'."
---

# Multi-Agent Constraint Factorization

This skill enables Claude to design and implement multi-agent systems where each agent enforces a distinct constraint on a shared output, and their sequential (cyclic) composition converges to a solution satisfying all constraints simultaneously. The core insight from Scofield (2026) is that factorized constraint enforcement -- where agents take turns projecting a candidate solution onto their individual constraint sets -- reaches invariant solution regions that a single monolithic agent applying all constraints at once cannot dynamically access, even with identical information and capacity.

## When to Use

- When the user wants to build a multi-agent pipeline where each agent is responsible for a different quality dimension (e.g., factual accuracy, safety, tone, relevance).
- When a single LLM prompt struggles to satisfy many simultaneous requirements and the user asks for a multi-pass or iterative refinement approach.
- When the user asks to decompose a complex generation task into independent, verifiable constraint checks.
- When designing code-review or content-moderation pipelines where multiple validators each enforce a specific policy.
- When the user needs a principled convergence guarantee for an iterative multi-agent editing loop rather than ad-hoc chaining.
- When building agentic workflows that must handle conflicting soft constraints with tunable priority weights.

## Key Technique

**Constraint-enforcement operators as projections.** Each agent *i* is modeled as a projection operator P_i that takes a candidate solution x and returns the nearest point in its constraint set C_i. For an LLM agent, this means: given a draft output, the agent rewrites it to be the minimally-modified version that satisfies its specific constraint (factuality, safety, style, etc.). A single agent trying to enforce C_1 AND C_2 AND ... AND C_m simultaneously must solve the full intersection problem in one shot -- which is computationally hard and often leads to mode collapse or constraint neglect.

**Factorized cyclic composition converges where monolithic application fails.** The factorized approach applies agents in sequence: x_{k+1} = P_m(P_{m-1}(...P_1(x_k)...)). Under mild conditions (non-empty closed constraint sets with transverse intersection), this cyclic iteration converges to the intersection C* = C_1 AND C_2 AND ... AND C_m at a geometric rate. The key theoretical result is that this sequential factorization reaches invariant solutions that are *not dynamically accessible* to a single agent applying all constraints at once, because the single agent's optimization landscape has local minima that the factorized path avoids.

**Soft constraints via proximal operators.** When perfect satisfaction of all constraints is impossible (e.g., maximum helpfulness conflicts with maximum safety), each hard projection P_i is replaced with a proximal operator prox_{lambda*rho_i} that trades off constraint satisfaction against minimal modification, weighted by a priority parameter rho_i. Higher rho means stricter enforcement. This lets you express a priority hierarchy across agents without binary feasible/infeasible outcomes.

## Step-by-Step Workflow

1. **Identify the constraint dimensions.** List every distinct requirement the output must satisfy. Each requirement becomes one agent's constraint set. Good factorizations have constraints that are roughly orthogonal -- correcting for one does not inherently violate another. Examples: factual accuracy, policy compliance, stylistic consistency, relevance to query, format adherence, code correctness.

2. **Define each agent's projection operator.** For each constraint C_i, write a prompt (or tool-augmented agent) whose sole job is: "Given this draft, return the minimally-modified version that satisfies [constraint i]. Change nothing else." Minimal modification is critical -- it is the Euclidean projection principle. Agents that rewrite aggressively will oscillate instead of converging.

3. **Choose an iteration order.** Place the most permissive (broadest constraint set) agents first and the most restrictive last. This follows the principle that early projections make large corrections and later ones make fine adjustments. Safety/policy agents typically go last so their constraints are never overwritten.

4. **Set convergence criteria.** Define a stopping condition: either a maximum iteration count (typically 2-4 cycles suffice) or a diff threshold where the output changes less than epsilon between consecutive full cycles.

5. **Implement the cyclic loop.** Pass the initial draft through each agent in order, then repeat the full cycle until convergence. Track the diff between iterations to monitor convergence rate.

6. **Tune soft-constraint weights (if needed).** When constraints conflict, replace hard projection with proximal weighting. Assign rho_i values reflecting priority. Start with equal weights and increase rho for the constraints that matter most. Monitor which constraints show residual violation and adjust.

7. **Add convergence diagnostics.** Log per-agent constraint violation scores at each iteration. A healthy run shows monotonically decreasing total violation. If violation oscillates, two agents are fighting -- their constraints may be nearly parallel or contradictory, requiring relaxation or re-factorization.

8. **Handle infeasibility gracefully.** If after max iterations the output still violates a constraint, report which constraints remain unsatisfied and by how much. Let the caller decide whether to accept the best-effort result or escalate.

9. **Validate the final output.** Run each agent's constraint check one final time as a read-only verification pass (no modification). If all pass, return the result. If any fail, flag the specific violation.

## Concrete Examples

**Example 1: Code review pipeline with factorized constraints**

User: "Build me a multi-agent code review system that checks for security vulnerabilities, style compliance, and correctness."

Approach:
1. Define three constraint agents:
   - `security_agent`: Checks for OWASP Top 10 vulnerabilities; rewrites only the vulnerable lines.
   - `style_agent`: Enforces project linting rules; reformats only non-compliant code.
   - `correctness_agent`: Runs type checks and unit tests; fixes only failing code paths.
2. Order: correctness first (broadest changes), then style (formatting), then security (most critical, applied last so it is never overwritten).
3. Cycle through agents up to 3 times or until the code diff between cycles is empty.

Implementation skeleton:
```python
def factorized_code_review(draft_code: str, max_cycles: int = 3) -> str:
    agents = [correctness_agent, style_agent, security_agent]
    current = draft_code
    for cycle in range(max_cycles):
        previous = current
        for agent in agents:
            current = agent.project(current)  # minimal modification
        if current == previous:
            break  # converged
    return current
```

Output: Code that passes all three checks, reached by iterative minimal corrections rather than a single monolithic rewrite.

**Example 2: Content generation with safety, accuracy, and tone constraints**

User: "I need a multi-agent pipeline that generates marketing copy which is factually accurate, brand-compliant, and engaging."

Approach:
1. Define three constraint agents:
   - `factuality_agent`: Cross-references claims against a product database; removes or corrects unsupported claims.
   - `brand_agent`: Checks tone, terminology, and messaging guidelines; adjusts only non-compliant phrases.
   - `engagement_agent`: Scores readability and hooks; strengthens weak openings and CTAs without changing factual content.
2. Order: engagement (broadest creative freedom), then factuality (narrows to truthful claims), then brand compliance (final polish, never overwritten).
3. Use soft constraints with proximal weighting: rho_brand = 1.0 (strict), rho_factuality = 1.0 (strict), rho_engagement = 0.6 (flexible -- some engagement can be sacrificed for accuracy).

```python
agents = [
    ProximalAgent(engagement_agent, rho=0.6),
    ProximalAgent(factuality_agent, rho=1.0),
    ProximalAgent(brand_agent,      rho=1.0),
]

copy = initial_draft
for cycle in range(4):
    for agent in agents:
        copy = agent.proximal_project(copy)
```

Output: Marketing copy that is factually grounded, brand-consistent, and as engaging as those two hard constraints allow.

**Example 3: Multi-agent structured data extraction**

User: "Extract structured records from messy PDFs. Each record must have valid dates, normalized addresses, and consistent entity names."

Approach:
1. Define three constraint agents:
   - `date_agent`: Parses and normalizes all date fields to ISO 8601; fixes ambiguous formats (01/02/03).
   - `address_agent`: Validates and normalizes addresses against a postal database; corrects typos.
   - `entity_agent`: Resolves entity name variants to canonical forms using a reference dictionary.
2. Order: entity resolution first (changes names that date/address agents reference), then dates, then addresses.
3. Convergence: typically 1-2 cycles since these constraints are nearly orthogonal (modifying a date rarely breaks an address).

```python
record = raw_extraction
for cycle in range(2):
    record = entity_agent.project(record)
    record = date_agent.project(record)
    record = address_agent.project(record)
```

Output: Clean structured records where every field independently passes validation.

## Best Practices

- **Do:** Design each agent's prompt to make *minimal* modifications. The projection principle requires returning the closest feasible point, not a full rewrite. Instruct agents: "Change only what is necessary to satisfy your constraint. Preserve everything else verbatim."
- **Do:** Place the most critical constraint last in the cycle order so it is never overwritten by a subsequent agent in the same cycle.
- **Do:** Monitor per-iteration diffs. Convergence should show shrinking diffs. If diffs grow or oscillate, your constraints are fighting.
- **Do:** Use soft proximal weighting (rho parameters) when you know constraints will conflict, rather than discovering infeasibility at runtime.
- **Avoid:** Giving a single agent multiple unrelated constraints. The entire point of factorization is that each agent owns exactly one constraint dimension.
- **Avoid:** Running too many cycles without a convergence check. In practice, 2-4 cycles suffice for well-factorized constraints. If you need more, the factorization is likely wrong.

## Error Handling

| Symptom | Cause | Fix |
|---------|-------|-----|
| Output oscillates between two versions across cycles | Two agents have nearly contradictory constraints | Relax one constraint to soft (lower its rho), or merge both into a single agent that balances them |
| Output degrades in quality after the first cycle | An agent is rewriting too aggressively instead of projecting minimally | Tighten the agent's prompt to emphasize minimal modification; add examples of good vs. bad projections |
| Convergence takes many cycles (>5) | Constraints are nearly parallel (overlapping concern areas) | Re-factorize: merge overlapping agents or redefine constraint boundaries to be more orthogonal |
| Final output still violates a constraint after max cycles | Constraint intersection is empty (infeasible) | Switch to proximal mode with priority weights; report the residual violation to the user |
| One agent's changes are always undone by the next | Ordering is wrong; a less-critical agent runs after a more-critical one | Reorder agents so the most critical constraint is enforced last |

## Limitations

- **Constraint orthogonality assumption.** The framework works best when constraints are roughly independent. If two constraints are deeply entangled (e.g., "be concise" and "include all details"), factorization provides little benefit over a single agent.
- **Projection fidelity of LLMs.** Real LLM agents are imperfect projectors -- they may overshoot (rewrite too much) or undershoot (miss violations). The convergence guarantees assume ideal projection, so practical systems need the diagnostic checks described above.
- **Latency scales linearly.** Each cycle requires m sequential agent calls. For latency-sensitive applications, limit to 1-2 cycles or use the parallel (averaging) variant where agents project independently and a coordinator averages their outputs.
- **Not a substitute for a single strong model on simple tasks.** If one prompt can reliably satisfy all constraints, the overhead of multi-agent orchestration is not justified. This technique shines when constraint count or complexity exceeds what a single pass handles well.
- **Soft-constraint tuning is empirical.** There is no closed-form way to set optimal rho values for LLM agents. Tuning requires experimentation on representative inputs.

## Reference

Scofield, C. (2026). *Multi-Agent Constraint Factorization Reveals Latent Invariant Solution Structure.* arXiv:2601.15077v1. [https://arxiv.org/abs/2601.15077v1](https://arxiv.org/abs/2601.15077v1)

Key takeaway: Section 3 (factorized composition theorem) and Section 5 (dialog system application) provide the formal proof that cyclic multi-agent projection reaches solution regions inaccessible to monolithic single-agent constraint enforcement.