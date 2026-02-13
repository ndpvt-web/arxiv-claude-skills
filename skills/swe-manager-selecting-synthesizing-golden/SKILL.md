---
name: "swe-manager-selecting-synthesizing-golden"
description: >
  Select and synthesize a golden proposal from multiple candidate fix strategies before writing code.
  Mirrors how technical managers deliberate on competing proposals by weighing scope, impact, risk,
  and each candidate's strengths and weaknesses. Use when: "compare these fix approaches", "which
  solution should we pick", "evaluate these proposals for the bug", "help me choose between these
  implementation strategies", "synthesize the best approach from these options", "review competing
  fixes before coding".
---

# SWE-Manager: Selecting and Synthesizing Golden Proposals Before Coding

This skill enables Claude to act as a technical manager who evaluates multiple candidate proposals for resolving a software issue, selects the strongest approach, and synthesizes a single "golden proposal" that combines the best elements before any code is written. Rather than jumping straight to implementation, this technique forces structured deliberation over competing strategies -- assessing scope, impact, urgency, regression risk, and each proposal's trade-offs -- producing a justified recommendation that makes downstream coding more reliable and focused.

## When to Use

- When the user presents a bug or feature request and asks "how should we fix this?" with multiple viable approaches
- When reviewing a GitHub issue that has several proposed solutions in the comments and the user needs to pick one
- When the user provides 2-5 different implementation strategies and wants a reasoned comparison
- When refactoring decisions have multiple valid paths (e.g., extract class vs. inline, different design patterns)
- When a pull request review surfaces alternative approaches and the team needs to converge on one
- When the user wants to reduce regression risk by deliberating before coding, not after
- When triaging a complex bug where the root cause is ambiguous and multiple hypotheses exist

## Key Technique

**Joint Selection and Synthesis.** Traditional approaches pick the single best proposal from a set. SWE-Manager goes further: it selects the strongest candidate *and* synthesizes a golden proposal that may incorporate elements from multiple candidates. This mirrors how experienced technical managers operate -- they don't just vote for option A or B; they extract the best reasoning from each and produce a refined directive. The golden proposal is not merely a winner; it is an improved version informed by the deliberation.

**Proposal Evaluation as Reasoning, Not Execution.** The core insight is that choosing the right fix strategy is a *reasoning* task, not a code-execution task. You don't need to run tests or compile code to evaluate whether "add a null check at the API boundary" is more robust than "refactor the caller to never pass null." The evaluation examines: (1) how well the proposal addresses the root cause, (2) the blast radius of the change, (3) regression risk to adjacent functionality, (4) alignment with existing codebase patterns, and (5) operational hazards like performance degradation or data loss.

**Structured Justification.** Every selection must be accompanied by an explicit justification that references concrete strengths and weaknesses of each candidate. This prevents gut-feel decisions and creates an auditable record of *why* a particular approach was chosen -- useful for code review, onboarding, and post-incident analysis.

## Step-by-Step Workflow

1. **Characterize the issue.** Read the bug report, feature request, or failing test. Identify the root cause (or top hypotheses if ambiguous), the scope of affected code, the severity/urgency, and any constraints (backward compatibility, performance budgets, deployment windows).

2. **Enumerate candidate proposals.** If the user provides proposals, catalog them. If not, generate 2-4 distinct candidate approaches yourself by exploring the codebase. Each proposal should be a concrete strategy (not just "fix the bug") with a description of *what changes where* and *why it works*.

3. **Analyze each proposal independently.** For every candidate, evaluate along five dimensions:
   - **Root-cause coverage:** Does it fix the actual cause or just a symptom?
   - **Blast radius:** How many files/modules/tests does it touch?
   - **Regression risk:** Could it break existing behavior? How likely?
   - **Pattern alignment:** Does it follow the codebase's existing conventions?
   - **Operational risk:** Performance impact, data migration needs, deployment complexity?

4. **Compare proposals head-to-head.** Build a comparison matrix. Identify where proposals agree (likely correct shared reasoning) and where they diverge (the decision points). Flag any proposal that introduces unnecessary complexity or addresses only a subset of the problem.

5. **Select the strongest candidate.** Choose the proposal with the best overall profile. State your selection clearly and justify it by referencing the comparison. Be explicit about what the runner-up got right and where the winner is superior.

6. **Synthesize the golden proposal.** Refine the selected proposal by incorporating any superior elements from the other candidates. For example, if proposal A has the best fix strategy but proposal B identified an additional edge case, the golden proposal should address both. Write the golden proposal as a clear, implementable directive.

7. **Define the implementation plan.** Translate the golden proposal into a concrete sequence of code changes: which files to modify, what to add/remove, what tests to write or update. This becomes the coding spec.

8. **Validate against constraints.** Re-check the golden proposal against the original issue's constraints (backward compatibility, performance, timeline). If any constraint is violated, iterate on the synthesis.

9. **Document the decision.** Produce a brief decision record: the issue, the candidates considered, the selection rationale, and the golden proposal. This serves as the commit message preamble or PR description.

## Concrete Examples

**Example 1: Choosing a Fix for a Race Condition**

```
User: We have a race condition in our order processing service. When two
requests hit the same order simultaneously, we get duplicate charges.
Here are three proposals from the team:

A) Add a database-level unique constraint on (order_id, charge_id)
B) Use Redis distributed locks around the charge operation
C) Switch to an optimistic concurrency control pattern with version columns

Help me pick the best approach.
```

Approach:
1. Characterize: Race condition causing duplicate charges. High severity (financial).
   Constraint: Must not add significant latency to the happy path.

2. Analyze each proposal:
   - **A (DB constraint):** Fixes symptom (prevents duplicate rows) but doesn't prevent
     the duplicate API call to the payment provider. Low blast radius (one migration).
     Risk: the second request still hits the payment API and fails at DB insert, requiring
     careful error handling to avoid charging but not recording.
   - **B (Redis lock):** Prevents the race entirely by serializing access. Adds Redis as
     a runtime dependency. Risk: lock expiry edge cases, Redis availability becomes critical
     path. Adds ~2-5ms latency.
   - **C (Optimistic concurrency):** Detects the conflict at write time via version column.
     No external dependency. Risk: requires retry logic on conflict, slightly more complex
     to implement correctly.

3. Compare: A is incomplete (doesn't prevent payment API double-call). B and C both
   prevent the race but B adds an infrastructure dependency. C is self-contained.

4. Select: **C (Optimistic concurrency)** -- self-contained, no new infra, proven pattern.

5. Synthesize golden proposal: Use optimistic concurrency with version columns on the
   orders table, AND add proposal A's unique constraint as a safety net. This gives
   defense in depth: the version check prevents the race, the constraint catches any
   edge case that slips through.

Output:
```
## Golden Proposal: Optimistic Concurrency + DB Safety Net

**Primary mechanism:** Add a `version` column to the `orders` table. Before
charging, read the current version. After charging, update the order with
`WHERE version = :read_version`. If 0 rows affected, the order was modified
concurrently -- abort and return 409 Conflict.

**Safety net:** Add a UNIQUE constraint on `(order_id, charge_id)` to catch
any edge case the optimistic lock misses.

**Implementation plan:**
1. Migration: add `version INTEGER DEFAULT 0` to orders, add unique constraint
2. Modify `OrderService.processCharge()` to read version before charge
3. Modify `OrderRepository.recordCharge()` to use versioned UPDATE
4. Add retry/conflict handling in the API controller
5. Add integration test simulating concurrent charge attempts

**Why not the others:**
- Pure DB constraint (A) doesn't prevent duplicate payment API calls
- Redis lock (B) adds infrastructure dependency and lock-expiry edge cases
```

**Example 2: Evaluating Approaches to a Memory Leak**

```
User: Our Node.js API server's memory grows linearly over 24 hours until
it OOMs. I've narrowed it to the WebSocket connection handler. Two ideas:

A) Add explicit cleanup in the 'close' event handler -- null out references
   to the message buffer and user session cache for each connection
B) Refactor to use WeakRef for the session cache so GC can collect
   disconnected sessions automatically
```

Approach:
1. Characterize: Memory leak in WebSocket handler. Severity: causes production crashes
   every ~24h. Scope: isolated to connection lifecycle management.

2. Analyze:
   - **A (Explicit cleanup):** Direct, easy to understand and debug. Addresses the
     immediate leak. Risk: if any code path skips the close handler (crash, timeout
     without clean disconnect), the leak persists for those connections.
   - **B (WeakRef):** Elegant, GC handles cleanup automatically. Risk: WeakRef behavior
     is non-deterministic (GC timing), harder to reason about in debugging, and the
     session cache may need strong references for active operations (WeakRef could cause
     premature collection if the only reference is through the cache).

3. Select: **A (Explicit cleanup)** -- deterministic, debuggable, directly addresses
   the known leak path.

4. Synthesize golden proposal: Use explicit cleanup (A), but also add a periodic sweep
   (inspired by B's GC thinking) that runs every 5 minutes to catch connections that
   missed the close handler. Log when the sweep finds orphaned references to detect
   the root cause of missed close events.

Output:
```
## Golden Proposal: Explicit Cleanup + Periodic Sweep

1. In `ws.on('close')`: null out `connection.messageBuffer` and
   `sessionCache.delete(connection.userId)`
2. Add a 5-minute interval that iterates `sessionCache`, checks each
   entry's `ws.readyState`, and cleans up any in CLOSED state
3. Add a metric counter for sweep-cleaned entries (if > 0, investigate
   why close handler missed them)
4. Add unit test: simulate abrupt disconnect, verify cleanup runs

**Why not WeakRef:** Non-deterministic GC timing makes memory behavior
hard to predict and debug. Active sessions need strong references.
```

**Example 3: Feature Implementation Strategy**

```
User: We need to add full-text search to our product catalog. Options:

A) Use PostgreSQL's built-in tsvector/tsquery
B) Add Elasticsearch as a dedicated search service
C) Use a hosted solution like Algolia

We have ~50K products, 2 developers, and need this in production soon.
```

Approach:
1. Characterize: Feature request, not a bug. 50K products (small dataset). Team of 2.
   Implicit constraint: minimize operational burden and time to ship.

2. Analyze:
   - **A (PostgreSQL tsvector):** No new infrastructure. Built into existing DB. Handles
     50K products easily. Limitations: less sophisticated relevance ranking, no typo
     tolerance out of the box. Blast radius: schema migration + query changes only.
   - **B (Elasticsearch):** Powerful search features. But: new infrastructure to deploy,
     monitor, and keep in sync. Overkill for 50K products. Significant operational burden
     for a 2-person team.
   - **C (Algolia):** Excellent search UX with minimal code. But: ongoing cost, external
     dependency, data leaves your infrastructure, vendor lock-in.

3. Select: **A (PostgreSQL tsvector)** -- zero new infrastructure, sufficient for scale,
   keeps the system simple.

4. Synthesize: Use PostgreSQL full-text search with a GIN index on a generated tsvector
   column. Add `pg_trgm` extension for typo tolerance (trigram similarity), borrowing the
   fuzzy-matching strength that would otherwise require Elasticsearch.

Output:
```
## Golden Proposal: PostgreSQL FTS + pg_trgm

1. Add `search_vector tsvector` generated column to `products` table
2. Create GIN index on `search_vector`
3. Enable `pg_trgm` extension, add trigram index on `name` for fuzzy matching
4. Implement search endpoint: tsquery for exact matches, fall back to
   trigram similarity for typo tolerance
5. Add search result ranking using `ts_rank_cd()`

**Why not the others:**
- Elasticsearch (B): operational overhead unjustified for 50K records and 2-person team
- Algolia (C): adds cost and vendor dependency for a tractable in-house problem

**Upgrade path:** If product count reaches 1M+ or search requirements grow
(faceting, geo, personalization), revisit Elasticsearch at that point.
```

## Best Practices

- **Do:** Generate at least 3 candidate proposals before selecting. With fewer than 3, you haven't explored the solution space enough and may miss the best approach.
- **Do:** Evaluate proposals against the *actual codebase* -- read the relevant files, understand existing patterns, check what dependencies are already available. Abstract reasoning without codebase context leads to impractical proposals.
- **Do:** Always produce a written justification that names specific strengths and weaknesses. "Proposal A is better" without explanation is not a golden proposal selection.
- **Do:** Synthesize across proposals. The best answer is often not any single candidate but a combination. Look for complementary strengths.
- **Avoid:** Jumping to implementation before deliberation. The whole point is that 10 minutes of structured comparison saves hours of rework from picking the wrong approach.
- **Avoid:** Evaluating proposals by running code or tests. This is a reasoning task -- assess feasibility, risk, and alignment from the code structure and domain knowledge. Execution-based validation comes *after* selection, during implementation.
- **Avoid:** Letting sunk-cost bias influence selection. If the user has already started on proposal B, that's not a reason to select it if proposal A is clearly superior. State the trade-off honestly.

## Error Handling

- **Insufficient context:** If the issue description is vague or the proposals lack detail, ask the user clarifying questions before evaluating. A garbage-in evaluation produces a garbage golden proposal.
- **All proposals are weak:** If none of the candidates adequately address the root cause, say so explicitly. Generate a new proposal rather than selecting the "least bad" option. The golden proposal can be entirely new.
- **Tied candidates:** When two proposals are genuinely equivalent, the synthesis step becomes critical. Combine their strengths into one golden proposal. If they're identical in substance, pick the one with lower blast radius.
- **Conflicting constraints:** If the issue has constraints that no single proposal can satisfy (e.g., "zero downtime AND complete schema rewrite"), flag the conflict. Propose a phased approach or ask the user to prioritize constraints.
- **User disagrees with selection:** Present the comparison matrix and let the user override. The goal is informed decision-making, not forcing a choice. If overridden, document why.

## Limitations

- **Requires domain understanding.** The quality of proposal evaluation depends on understanding the codebase and domain. For highly specialized domains (e.g., cryptography, distributed consensus), the evaluation may miss subtle correctness issues that only domain experts would catch.
- **Not a substitute for testing.** The golden proposal is a *plan*, not a verified fix. It still needs implementation, testing, and review. The technique reduces the chance of picking the wrong approach but doesn't guarantee the selected approach is bug-free.
- **Diminishing returns beyond 5 proposals.** The comparison becomes unwieldy with too many candidates. If more than 5 proposals exist, cluster similar ones and evaluate the clusters first, then compare within the strongest cluster.
- **Biased toward conservative choices.** The risk-analysis framing naturally favors lower-risk options. For greenfield projects or prototypes where speed matters more than safety, this deliberative approach may be overly cautious.
- **Cannot evaluate performance-sensitive trade-offs without profiling.** If the decision hinges on "is approach A 10ms faster than B," reasoning alone won't resolve it. Flag these cases for benchmarking.

## Reference

**Paper:** [SWE-Manager: Selecting and Synthesizing Golden Proposals Before Coding](https://arxiv.org/abs/2601.22956) -- Tan et al., 2026. Focus on Section 3 (methodology for joint selection-synthesis) and Section 4 (the five-dimension evaluation framework for comparing proposals against real-world maintainer rationales).