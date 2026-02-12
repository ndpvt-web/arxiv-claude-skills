---
name: "davinci-agency-unlocking-long-horizon-agency"
description: "Decompose complex, long-horizon coding tasks into PR-like chains of verifiable subtasks with cross-stage dependency tracking and iterative refinement. Use when: 'break this feature into PRs', 'plan a multi-step implementation', 'decompose this project into verifiable stages', 'help me build this feature incrementally with tests at each step', 'create a chain of PRs for this task', 'plan an implementation with bug-fix iterations'."
---

# daVinci-Agency: PR-Grounded Long-Horizon Task Decomposition

This skill enables Claude to tackle complex, multi-step software engineering tasks by structuring work as **chains of Pull Requests**--mirroring how experienced developers naturally decompose large objectives into verifiable, causally-linked submission units. Derived from the daVinci-Agency framework (Jiang et al., 2026), the core insight is that PR sequences from real software evolution encode three properties essential for long-horizon success: progressive decomposition through commits, consistency enforcement through unified functional objectives, and verifiable refinement through bug-fix trajectories. Claude applies this structure to plan and execute multi-stage implementations where each stage is independently testable yet contributes to a coherent whole.

## When to Use

- When the user asks to implement a feature that spans multiple files, modules, or architectural layers (e.g., "Add user authentication with OAuth, session management, and role-based access control")
- When a task requires iterative refinement--building a foundation, then extending it across several rounds of changes
- When the user wants a structured implementation plan broken into mergeable, reviewable units
- When debugging or refactoring requires tracing dependencies across a long chain of changes
- When building a feature where later steps depend on earlier ones being correct (e.g., database schema -> API -> frontend)
- When the user explicitly asks to "break this into PRs" or "plan an incremental implementation"
- When a project-level task would produce 500+ lines of changes and benefits from staged delivery

## Key Technique

**Chain-of-PRs as Supervision Structure.** The daVinci-Agency paper demonstrates that real Pull Request sequences are a natural source of long-horizon supervision signals. Unlike synthetic step-by-step plans that treat each action independently, PR chains preserve **causal dependencies** (PR-3 fixes a bug introduced by PR-2's interaction with PR-1's schema), **iterative refinement** (bug-fix commits within a PR demonstrate diagnosis-hypothesis-fix-validate loops), and **functional coherence** (all PRs in the chain serve a unified objective). The key insight: decomposition should produce units that are *individually verifiable* yet *collectively coherent*.

**Three Interlocking Mechanisms.** The framework operates through: (1) **Progressive task decomposition** -- breaking the objective into a sequence of commits/PRs where each builds on the last, analogous to how a developer submits incremental work. (2) **Long-term consistency enforcement** -- maintaining a unified functional objective across all stages so that local decisions align with the global goal. (3) **Verifiable refinement** -- explicitly modeling the bug-fix cycle where test failures after a stage trigger targeted corrections before proceeding, rather than hoping each stage is perfect on the first attempt.

**Data-Efficient Execution.** The paper shows that even 239 well-structured trajectories (averaging 85k tokens and 116 tool calls each) can yield a 47% relative gain on complex benchmarks. For Claude, this translates to a practical principle: invest heavily in the *structure* of the plan rather than the volume of output. A well-decomposed chain of 5 PRs with clear verification at each stage outperforms a monolithic implementation attempt.

## Step-by-Step Workflow

1. **Extract the unified functional objective.** Before any decomposition, state the single overarching goal in one sentence. This is the "north star" that every subsequent PR must serve. Example: "Enable users to authenticate via GitHub OAuth and access role-gated API endpoints."

2. **Identify the dependency graph.** Map out which components depend on which: database schema before ORM models, models before API routes, routes before frontend integration. Sketch this as a DAG (directed acyclic graph) of work units.

3. **Decompose into a chain of PRs.** Convert the DAG into a linear (or minimally-branching) sequence of PRs, each representing the smallest unit of work that is independently *testable* and *mergeable*. Each PR should have: a title, a 1-2 sentence description of what it adds, files touched, and an explicit verification criterion (test, assertion, or observable behavior).

4. **Define verification gates for each PR.** For every PR in the chain, specify what "done" looks like: a passing test suite, a curl command that returns expected JSON, a UI element that renders correctly, or a type-check that passes. These gates are non-negotiable checkpoints.

5. **Implement PR-1: the foundation.** Start with the PR that has zero dependencies. Write the code, then immediately run verification. Do not proceed to PR-2 until PR-1's gate passes.

6. **Run the bug-fix refinement loop.** If verification fails after implementing a PR, enter the refinement cycle: diagnose the failure, form a hypothesis about the root cause, implement a targeted fix, and re-verify. This mirrors the authentic bug-fix trajectories that make daVinci-Agency's training data effective. Track each fix as an explicit commit within the current PR.

7. **Enforce cross-stage consistency before advancing.** Before starting the next PR, review the unified objective and confirm that the current state of the codebase still aligns with it. Check for unintended side effects: does the new code break assumptions that later PRs depend on?

8. **Implement subsequent PRs sequentially.** For each remaining PR in the chain, repeat steps 5-7. Carry forward context from prior PRs -- explicitly reference which files/functions from earlier stages are being extended or consumed.

9. **Run integration verification after the final PR.** Once all PRs are implemented, run a full integration check that exercises the entire chain end-to-end. This validates that the cross-stage dependencies hold and the unified objective is met.

10. **Document the chain.** Produce a summary listing each PR in order, what it accomplished, and any refinement iterations that occurred. This serves as both a review artifact and a reusable template for similar tasks.

## Concrete Examples

**Example 1: Multi-stage API feature**
```
User: "Build a notification system that supports email and Slack,
       with user preferences and rate limiting."

Approach (Chain-of-PRs):

PR-1: Notification preferences schema and model
  - Add `notification_preferences` table with columns: user_id, channel
    (email/slack), enabled, frequency_limit
  - Migration file + model definition
  - Verification: migration runs, model CRUD tests pass

PR-2: Core notification dispatcher
  - NotificationService class with send(user, event) method
  - Channel routing based on preferences from PR-1
  - Verification: unit tests with mocked channels pass

PR-3: Email channel implementation
  - EmailAdapter implementing ChannelInterface from PR-2
  - Template rendering for notification body
  - Verification: integration test sends email via test SMTP

PR-4: Slack channel implementation
  - SlackAdapter implementing ChannelInterface from PR-2
  - Webhook configuration and message formatting
  - Verification: integration test posts to Slack test channel

PR-5: Rate limiting middleware
  - RateLimiter wrapping NotificationService from PR-2
  - Uses frequency_limit from preferences (PR-1)
  - Verification: test that >N sends within window are throttled

PR-6: API endpoints and integration
  - REST endpoints for preference CRUD + manual notification trigger
  - End-to-end test: set preferences, trigger notification, verify delivery
  - Verification: full API test suite passes
```

**Example 2: Refactoring with preserved behavior**
```
User: "Refactor our monolithic request handler into middleware layers
       without breaking existing API contracts."

Approach (Chain-of-PRs):

PR-1: Extract authentication into middleware
  - Move auth logic from handler to AuthMiddleware class
  - Wire into request pipeline at same position
  - Verification: existing auth test suite passes unchanged

PR-2: Extract validation into middleware
  - Move input validation to ValidationMiddleware
  - Depends on PR-1 (auth runs before validation)
  - Verification: all validation edge-case tests pass

PR-3: Extract rate limiting into middleware
  - Move rate limiting to RateLimitMiddleware
  - Verification: rate limit tests pass, load test shows same thresholds

PR-4: Slim down the core handler
  - Remove extracted logic, handler now only does business logic
  - Verification: full integration test suite passes, no behavior change

Bug-fix refinement example (within PR-2):
  - Initial implementation breaks when auth middleware rejects --
    validation middleware receives null user context
  - Fix: add early-return guard in validation middleware for
    unauthenticated requests
  - Re-run tests: all pass
```

**Example 3: Greenfield project bootstrapping**
```
User: "Set up a new CLI tool that fetches data from an API,
       caches it locally, and generates reports."

Approach (Chain-of-PRs):

PR-1: Project scaffolding and CLI argument parsing
  - Initialize project, configure CLI framework (e.g., argparse/click)
  - Verification: `tool --help` prints usage, `tool --version` works

PR-2: API client with error handling
  - HTTP client wrapping the target API, retry logic, auth
  - Verification: unit tests with mocked responses cover success,
    timeout, and auth-failure cases

PR-3: Local caching layer
  - SQLite-backed cache keyed by query parameters
  - Cache invalidation by TTL
  - Verification: test cache hit/miss/expiry behavior

PR-4: Report generation engine
  - Transform cached data into formatted output (CSV, JSON, table)
  - Verification: snapshot tests comparing generated reports to fixtures

PR-5: End-to-end integration
  - Wire CLI args -> API client -> cache -> report generator
  - Verification: `tool fetch --format csv` produces expected output
    against a local mock server
```

## Best Practices

- **Do:** Make each PR's verification gate concrete and automated. "Tests pass" is acceptable; "looks right" is not. The power of this method comes from verifiable intermediate states.
- **Do:** Track the unified functional objective explicitly. Write it at the top of your plan and reference it when making decisions within each PR. Drift is the primary failure mode of long-horizon work.
- **Do:** Model the bug-fix loop explicitly. When something breaks, treat the diagnosis-fix-verify cycle as a first-class part of the process, not an interruption. Log what broke and why -- this is the "verifiable refinement" mechanism.
- **Do:** Keep PRs small enough to reason about in isolation. If a single PR touches more than 3-4 files or introduces more than one conceptual change, split it further.
- **Avoid:** Implementing multiple PRs simultaneously or skipping verification between stages. The sequential, verified chain is what maintains causal correctness.
- **Avoid:** Treating the decomposition as purely top-down. Real PR chains involve feedback -- a bug discovered in PR-3 may require amending PR-1's schema. Build in explicit checkpoints to reassess earlier decisions.
- **Avoid:** Over-decomposing trivial tasks. If the entire feature is 50 lines across 2 files, a single PR suffices. Reserve chain-of-PRs for tasks where long-horizon dependencies genuinely exist.

## Error Handling

- **Verification gate failure:** Do not skip to the next PR. Enter the refinement loop: read the error output, identify the root cause, apply a minimal fix, and re-run verification. If three refinement attempts fail, reassess whether the PR's scope is correct -- it may need to be split or reordered.
- **Cross-stage regression:** If PR-N breaks something that passed in PR-(N-1), the dependency graph was incomplete. Roll back PR-N's changes, update the dependency map, and determine whether an intermediate PR is needed to bridge the gap.
- **Scope creep within a PR:** If implementing a PR reveals that it requires significant unplanned work, stop. Create a new PR in the chain for the unexpected work, slot it into the correct position in the dependency order, and proceed with the revised plan.
- **Context window pressure:** For very long chains (7+ PRs), summarize completed PRs into a compact state description rather than carrying full diffs forward. Focus context on the current PR and its immediate dependencies.

## Limitations

- **Not suitable for exploratory/research tasks.** Chain-of-PRs assumes a reasonably clear objective. If the user is still figuring out *what* to build, use a different approach (spikes, prototypes) before applying this decomposition.
- **Overhead for small tasks.** Tasks under ~100 lines of total change rarely benefit from multi-PR decomposition. The planning overhead exceeds the coordination benefit.
- **Linear chains struggle with parallel work.** If multiple components can be developed independently, a strict PR chain may impose unnecessary serialization. In such cases, use parallel branches that merge at a defined integration PR.
- **Depends on testability.** The verification gates require that each stage can be tested in isolation. Codebases without testing infrastructure may need a "PR-0" to set up basic test scaffolding first.

## Reference

- **Paper:** [daVinci-Agency: Unlocking Long-Horizon Agency Data-Efficiently](https://arxiv.org/abs/2602.02619v2) (Jiang et al., 2026). Look for Section 3 on the three interlocking mechanisms and the chain-of-PRs formalization, and Section 4 on how 239 structured trajectories yielded a 47% gain on Toolathlon through PR-grounded causal structure.