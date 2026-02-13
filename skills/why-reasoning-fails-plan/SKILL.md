---
name: "why-reasoning-fails-plan"
description: "Apply FLARE (Future-aware Lookahead with Reward Estimation) to long-horizon coding tasks. Replaces greedy step-by-step reasoning with explicit lookahead, value propagation, and limited commitment so early decisions account for downstream consequences. Use when: 'plan a complex refactor', 'help me sequence these migrations', 'break down this multi-step task', 'design a long pipeline', 'why does my agent keep failing at step 7', 'plan this feature without painting myself into a corner'."
---

# FLARE: Future-aware Planning for Long-Horizon Coding Tasks

This skill teaches Claude to apply the FLARE framework (Future-aware Lookahead with Reward Estimation) from Wang et al. (2026) when tackling multi-step coding tasks. Standard step-by-step reasoning picks the locally best action at each step — a greedy policy that works for short tasks but causes **myopic commitments** in long-horizon work. FLARE fixes this by simulating candidate paths forward, propagating outcome quality backward to inform early decisions, and committing to only one step at a time before replanning. The result: early choices reflect their downstream consequences instead of just looking good in isolation.

## When to Use

- When a user asks you to plan or sequence a multi-step refactor, migration, or feature implementation spanning 5+ interdependent steps
- When a coding task has **irreversible or costly early decisions** (e.g., choosing a database schema, picking an API contract, selecting an architectural pattern) that constrain everything downstream
- When you notice a prior plan failed because an early choice created cascading problems later (the classic "painted into a corner" scenario)
- When building a multi-stage pipeline (data ingestion -> transform -> validate -> store -> serve) where stage ordering and interface design matter
- When the user asks you to design an agent, workflow, or state machine with many branching paths
- When sequencing database migrations, infrastructure changes, or deployment steps where rollback is expensive

## Key Technique

**The core problem.** Standard chain-of-thought reasoning evaluates each step in isolation: "What's the best next action given where I am now?" This produces a step-wise greedy policy. For short tasks (2-3 steps), greedy works fine. For long-horizon tasks (6+ steps with dependencies), greedy reasoning makes early commitments that look locally optimal but are globally suboptimal. These myopic errors compound — each bad early choice constrains future options, and by step 5 or 6, no local fix can recover the plan. This is why an 8B-parameter model with proper planning structure can outperform GPT-4o with naive step-by-step reasoning.

**FLARE's three mechanisms.** (1) **Explicit Lookahead**: Before committing to any action, generate 2-4 candidate actions and mentally simulate each one forward by 2-3 steps. Ask: "If I take this action, what does the world look like 3 steps from now?" (2) **Value Propagation**: Score each simulated trajectory by its end-state quality (does it reach the goal? does it leave options open?), then propagate those scores backward to the candidate actions that started them. An action that looks mediocre now but leads to excellent outcomes in 3 steps should score higher than an action that looks great now but leads to a dead end. (3) **Limited Commitment**: Commit to only the single best-scoring action, execute it, then **replan from scratch**. Never lock in a full multi-step plan. This receding-horizon approach means later evidence can revise earlier value estimates and the plan self-corrects.

**Why this matters for coding.** Software engineering is full of long-horizon decisions with delayed consequences: choosing an abstraction early in a refactor, picking the order of migration steps, deciding which tests to write first. FLARE's discipline — simulate forward, score by outcomes, commit minimally — prevents the most common planning failure mode: making an early architectural choice that seems reasonable in isolation but forces painful workarounds five steps later.

## Step-by-Step Workflow

1. **Identify the decision horizon.** Count the number of distinct, sequential steps in the task. If it's 5 or fewer with no major dependencies, standard reasoning is fine. If it's 6+ steps or has irreversible early decisions, activate FLARE.

2. **Map the state space.** Write down the current state (what files exist, what interfaces are defined, what constraints apply) and the goal state (what the system should look like when done). Be explicit — vague goals produce vague plans.

3. **Enumerate candidate first actions (k=3-4).** For the first step, generate 3-4 genuinely different approaches. Not minor variations — structurally different paths. For a refactor, this might be: (a) extract interface first, (b) write tests first, (c) create new module alongside old one, (d) migrate consumers first.

4. **Simulate each candidate forward (depth H=3).** For each candidate first action, mentally execute the next 2-3 steps that would follow. Write out the intermediate states explicitly. What files change? What interfaces exist? What constraints are introduced?

5. **Score terminal states by goal proximity and optionality.** For each simulated 3-step trajectory, assign a quality score based on: (a) How close is the terminal state to the goal? (b) How many options remain open vs. how many are foreclosed? (c) Are there any dead-end states or irreversible mistakes?

6. **Propagate scores backward to candidate actions.** The first-step action that leads to the best average terminal state wins — even if it's not the most immediately productive action. An action that "sets up" future steps beats one that makes fast initial progress but constrains later work.

7. **Commit to exactly one action.** Execute only the winning first-step action. Do not execute the full simulated trajectory.

8. **Observe the actual outcome and replan.** After executing the one committed action, reassess the current state. The actual outcome may differ from the simulation. Return to step 3 with the new state and repeat.

9. **Cache trajectory evaluations.** If a new state is very similar to a previously simulated state (same files modified, same interfaces in play), reuse the prior trajectory scores instead of re-simulating. This prevents redundant analysis on each replanning cycle.

10. **Terminate when the remaining horizon is short.** Once only 2-3 steps remain and the path is clear, switch back to standard sequential execution. FLARE's overhead is only justified when the horizon is long enough for myopic errors to compound.

## Concrete Examples

**Example 1: Sequencing a database migration with application changes**

```
User: I need to rename the `users.username` column to `users.handle` across
the app. It's referenced in 12 files, 3 API endpoints, and 2 database views.
Plan the migration steps.

FLARE Analysis:

State: Column is `username` everywhere. Goal: Column is `handle` everywhere,
zero downtime.

Candidate first actions:
  A) Rename column in migration first, then update app code
  B) Add `handle` as alias/computed column, then migrate consumers, then drop old
  C) Update all app code to use `handle` first (with alias), then rename column
  D) Create new column `handle`, dual-write, migrate reads, drop old

Simulate forward (3 steps each):
  A) Step 1: rename column -> Step 2: app breaks immediately -> Step 3: emergency rollback
     Terminal score: 1/10 (downtime, cascading failures)

  B) Step 1: ALTER TABLE add computed column `handle` -> Step 2: migrate 12 files
     to read `handle` -> Step 3: update API contracts
     Terminal score: 7/10 (safe but computed columns add complexity)

  C) Step 1: add column alias at ORM level -> Step 2: update all 12 files ->
     Step 3: run migration to rename actual column
     Terminal score: 8/10 (clean, reversible at each step)

  D) Step 1: add `handle` column + dual-write trigger -> Step 2: backfill +
     migrate reads -> Step 3: drop `username` column + trigger
     Terminal score: 6/10 (safe but heavy machinery for a rename)

Propagated scores: C wins (8/10). Commit to step 1 of path C only.

Action: Add ORM-level alias mapping `handle` -> `username` so both names
work in application code. Verify tests pass with alias in place.

Then replan from the new state.
```

**Example 2: Planning a multi-file refactor to extract a service layer**

```
User: Our Express app has business logic mixed into route handlers across
8 files. Extract it into a service layer.

FLARE Analysis:

State: 8 route files with inline business logic. Goal: Route files delegate
to service modules.

Candidate first actions:
  A) Create all service files first (empty stubs), then move logic one by one
  B) Pick the simplest route file, extract its logic as the pattern, then repeat
  C) Write integration tests for all 8 endpoints first, then refactor safely
  D) Extract shared utilities first, then build services on top of them

Simulate forward (3 steps each):
  A) Step 1: create 8 stub files -> Step 2: start moving logic into first stub ->
     Step 3: realize stubs have wrong interfaces, rewrite
     Terminal score: 4/10 (premature interface design without seeing patterns)

  B) Step 1: extract simplest route (e.g., GET /health) -> Step 2: extract
     second route using same pattern -> Step 3: pattern is established,
     remaining 6 follow mechanically
     Terminal score: 9/10 (pattern emerges from concrete case, low risk)

  C) Step 1: write 8 integration tests -> Step 2: discover 3 endpoints have
     no clear contract -> Step 3: still haven't started refactoring
     Terminal score: 5/10 (useful but delays actual extraction)

  D) Step 1: extract shared DB/validation utilities -> Step 2: realize services
     need different utility shapes per route -> Step 3: refactor utilities again
     Terminal score: 4/10 (bottom-up design without knowing what services need)

Propagated scores: B wins (9/10). Commit to extracting the simplest route only.

Action: Identify the simplest route handler. Extract its business logic into
a service module. Ensure the route handler only does req/res handling.
Verify tests pass. Use this as the template for remaining 7.

Then replan — the pattern from this first extraction informs the interface
design for all subsequent services.
```

**Example 3: Designing an agent workflow with branching logic**

```
User: Build a document processing pipeline: ingest PDF/DOCX/HTML, extract
text, classify document type, route to specialized processors, validate
output, store results.

FLARE Analysis:

State: No pipeline exists. Goal: End-to-end pipeline handling 3 input formats.

Candidate first actions:
  A) Build the full pipeline for PDF first (vertical slice), then add formats
  B) Build the ingestion layer for all 3 formats first (horizontal slice)
  C) Define the inter-stage data contract/schema first, then implement stages
  D) Start with classification + routing since it's the decision point

Simulate forward (3 steps each):
  A) Step 1: PDF ingestion+extraction -> Step 2: classify+route PDF ->
     Step 3: realize the data contract between stages is PDF-specific, must
     redesign for DOCX/HTML
     Terminal score: 5/10 (works fast but creates format-coupled interfaces)

  B) Step 1: all 3 ingestors -> Step 2: need a common output format but
     haven't designed it -> Step 3: retrofit ingestors to common schema
     Terminal score: 4/10 (same problem, just hit later)

  C) Step 1: define DocumentPayload schema (text, metadata, source_format) ->
     Step 2: implement ingestors targeting the schema -> Step 3: implement
     classifier consuming the schema
     Terminal score: 9/10 (contract-first prevents rework, all stages decouple)

  D) Step 1: build classifier -> Step 2: no input data to test it ->
     Step 3: build mock ingestor to test classifier
     Terminal score: 3/10 (building in wrong dependency order)

Propagated scores: C wins (9/10). Commit to defining the inter-stage schema.

Action: Define the DocumentPayload interface/schema that flows between
pipeline stages. Include: raw_text, metadata dict, source_format enum,
extracted_sections list. Validate that this schema can represent output
from all 3 input formats.

Then replan with schema in hand.
```

## Best Practices

- **Do:** Generate structurally different candidates, not minor variations. "Add column then update code" vs. "Update code then rename column" are meaningfully different paths. "Rename with sed" vs. "Rename with find-replace" are not.
- **Do:** Score terminal states by **optionality** (how many future paths remain open), not just immediate progress. A step that closes off options is more dangerous than one that's slower but keeps doors open.
- **Do:** Write out simulated trajectories explicitly. Mental simulation without written traces falls back into intuitive greedy reasoning. The act of writing forces rigor.
- **Do:** Commit to one step and replan. Resist the urge to execute a full 8-step plan you simulated. Reality diverges from simulation, and replanning catches that divergence.
- **Avoid:** Using FLARE for simple, linear tasks (fix a typo, add a log statement, update a dependency version). The overhead isn't justified when there's only one reasonable path.
- **Avoid:** Simulating more than 3-4 steps ahead. Beyond that, simulation accuracy degrades and the analysis becomes speculative. Depth H=3 is the paper's default for good reason.
- **Avoid:** Scoring candidates by how "elegant" or "clean" they feel. Score by concrete end-state properties: does it compile? Do tests pass? Are interfaces stable? Is rollback possible?

## Error Handling

**Simulation produces tied scores.** When two candidates have near-identical propagated scores, pick the one with higher optionality (more future paths remain viable). If still tied, pick the more reversible option.

**Actual outcome diverges from simulation.** This is expected and is precisely why FLARE commits to only one step. When the real outcome after execution differs from what was simulated, the replanning cycle at step 8 catches the divergence. Do not try to "get back on track" to the original simulation — replan from actual state.

**Too many candidate actions.** If there are more than 4 reasonable first actions, prune to the 3-4 most structurally distinct ones before simulating. Simulating 8 candidates wastes analysis budget on minor variations.

**User wants to see the full plan upfront.** Present the current best trajectory as a tentative plan, but explicitly mark steps 2+ as provisional. Explain that each step will be re-evaluated after the previous one executes. Frame this as a feature, not a limitation — the plan adapts to reality.

**Replanning discovers a better path mid-execution.** This is a success, not a failure. Switching to a newly discovered better path after step 3 is exactly what FLARE enables. Do not treat plan changes as instability — treat them as the system working correctly.

## Limitations

- **Short-horizon tasks (1-4 steps):** FLARE adds analysis overhead that isn't justified. Standard step-by-step reasoning is sufficient when the horizon is short and consequences are immediate.
- **Highly uncertain environments:** FLARE simulates deterministic outcomes. When the result of an action is genuinely unpredictable (e.g., "will this third-party API change its behavior?"), simulation accuracy drops. In these cases, prefer reversible actions over simulated-best actions.
- **Creative/generative tasks:** Writing prose, designing UIs, naming things — these don't have the structured state transitions that FLARE exploits. The technique is designed for tasks with explicit states, actions, and evaluable outcomes.
- **Time-critical situations:** When a production system is down, the overhead of simulating 3-4 candidate recovery paths isn't worth it. Act on the most likely fix immediately and iterate.
- **Single-model simulation fidelity:** The LLM simulating future states is the same LLM that might make planning errors. Simulation quality is bounded by the model's own understanding, so trajectory scores are estimates, not ground truth.

## Reference

Wang, Z., Wu, F., Wang, H., Tang, X., & Li, B. (2026). *Why Reasoning Fails to Plan: A Planning-Centric Analysis of Long-Horizon Decision Making in LLM Agents.* arXiv:2601.22311v1. [https://arxiv.org/abs/2601.22311v1](https://arxiv.org/abs/2601.22311v1)

Key insight to take from the paper: step-wise reasoning creates a greedy policy that makes locally optimal but globally suboptimal decisions. The fix is three mechanisms — lookahead (simulate forward), value propagation (score by outcomes, not by immediate appeal), and limited commitment (execute one step, then replan). This minimal structure lets a small model with planning discipline outperform a much larger model using only reasoning.