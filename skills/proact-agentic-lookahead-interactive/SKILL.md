---
name: proact-agentic-lookahead-interactive
description: >
  Apply ProAct-style lookahead reasoning to multi-step coding and planning tasks.
  Compresses search-tree exploration into concise causal reasoning chains so the
  agent considers future consequences before committing to actions. Use when:
  "plan ahead before refactoring", "think through consequences of this change",
  "what happens if I restructure this module", "lookahead before migrating",
  "simulate outcomes of this architecture decision", "evaluate trade-offs for
  this design choice".
---

# ProAct: Agentic Lookahead for Interactive Decision-Making

This skill teaches Claude to apply **ProAct-style lookahead reasoning** — a structured method for exploring future consequences of actions before committing to them. Derived from the ProAct framework (Yu et al., 2026), the core idea is to mentally simulate multiple candidate actions, trace their downstream effects through a causal chain, compare outcomes including rejected alternatives, and only then select the best path. This replaces naive step-by-step execution with deliberate foresight grounded in actual codebase state.

## When to Use

- When the user asks to **refactor a module** and you need to evaluate which refactoring strategy avoids breaking downstream dependencies
- When **migrating** between frameworks, libraries, or API versions and each migration path has different cascading consequences
- When making an **architectural decision** (e.g., state management approach, database schema change) with multiple valid options and non-obvious trade-offs
- When **debugging a complex issue** where fixing one symptom could introduce regressions elsewhere
- When the user asks you to **plan a multi-file change** and wants to understand consequences before you start editing
- When resolving **merge conflicts or dependency tangles** where the resolution order affects the final outcome
- When evaluating whether a **performance optimization** (caching, parallelization, denormalization) will create correctness or maintenance problems downstream

## Key Technique

ProAct's insight is that effective planning requires **grounded lookahead** — not vague speculation, but reasoning anchored in the actual state of the system. The framework achieves this through two mechanisms that translate directly to agentic coding:

**Grounded LookAhead Distillation (GLAD):** Instead of exploring every possibility at decision time (expensive tree search), the agent learns to compress multi-branch exploration into a concise causal reasoning chain. For coding tasks, this means: read the actual code, mentally simulate what happens if you apply candidate change A vs. B vs. C, trace each through dependent files and tests, and articulate the comparison in natural language before choosing. The reasoning follows a strict **Observation-Analysis-Conclusion** structure with explicit counterfactual comparison ("Option A fixes the import cycle but forces a breaking change in the public API; Option B preserves the API but adds a layer of indirection").

**Monte-Carlo Critic (MC-Critic):** When the consequences of an action are uncertain, perform lightweight "rollouts" — quick, concrete checks rather than exhaustive analysis. In coding terms: instead of reasoning abstractly about whether a type change will break anything, actually grep for usages, run a targeted type-check, or trace one representative call path. These cheap probes calibrate your confidence in each candidate action, reducing the variance of your judgment.

The key departure from standard chain-of-thought is that ProAct reasoning is **contrastive and grounded**: every chosen action is accompanied by explicit reasoning about why alternatives were rejected, and every future-state prediction is anchored in observed code rather than hallucinated assumptions.

## Step-by-Step Workflow

1. **Observe the current state.** Read the files, dependencies, tests, and configuration involved. Build a concrete mental model of the system as it exists now — not as you assume it to be. Identify constraints (type contracts, public API surface, test expectations, performance requirements).

2. **Enumerate candidate actions.** List 2-4 distinct approaches to the task. Each candidate should be a concrete, implementable strategy — not a vague direction. For example: "Extract shared logic into a base class" vs. "Use a mixin/composition pattern" vs. "Duplicate with minor variation."

3. **Simulate each candidate forward (GLAD-style compression).** For each candidate, trace the causal chain through the codebase:
   - What files change directly?
   - What downstream files import or depend on changed interfaces?
   - Do any tests break? Which ones and why?
   - Does this introduce new coupling or remove existing coupling?
   - What does the diff look like in 3 months when the next feature is added?

4. **Perform lightweight rollout probes (MC-Critic checks).** For any prediction you're uncertain about, verify with a cheap concrete check:
   - Grep for all usages of a function you plan to rename or retype
   - Check the test suite for coverage of the path you plan to change
   - Read the build configuration to confirm your assumption about module boundaries
   - Look at git history to see how frequently the affected code changes

5. **Compare candidates with explicit counterfactual reasoning.** Write out why each rejected candidate was rejected. Use the format: "Candidate X achieves [benefit] but causes [specific problem traced in step 3], which is worse than Candidate Y's trade-off of [its cost] because [concrete reason]."

6. **Select the best action with stated justification.** Commit to one approach, summarizing the key reason it wins. The justification must reference specific observations from steps 3-4, not abstract preferences.

7. **Execute the first increment and verify.** Implement the smallest meaningful piece of the chosen approach. Run the relevant checks (type-checker, tests, linter) to confirm the predicted outcome matches reality.

8. **Re-evaluate after each increment.** After each step of execution, briefly re-assess: did the outcome match your lookahead prediction? If not, return to step 2 with updated state. This catch-and-correct loop prevents compounding errors.

## Concrete Examples

**Example 1: Choosing a state management refactor strategy**

```
User: Our React app has prop-drilling through 6 levels. Refactor to fix this.

Approach (ProAct Lookahead):
1. OBSERVE: Read the component tree. Identify the 4 props being drilled
   (user, theme, locale, permissions). Note that `user` and `permissions`
   change together on auth events; `theme` and `locale` change independently
   via settings.

2. ENUMERATE CANDIDATES:
   A) Single React Context for all four props
   B) Two Contexts: AuthContext (user + permissions) and PrefsContext (theme + locale)
   C) Zustand store for everything
   D) React Context for auth, CSS variables for theme, i18n library for locale

3. SIMULATE FORWARD:
   - Candidate A: Simple, but any auth change re-renders every consumer
     including pure UI components that only need theme. 47 components
     consume at least one prop — all would re-render on login.
   - Candidate B: Auth changes only re-render auth consumers (12 components).
     Prefs changes only re-render prefs consumers (31 components). Clean
     separation matching actual change patterns.
   - Candidate C: Adds a dependency for a problem React Context solves natively.
     Zustand selectors would fix re-render issue but team has no Zustand
     experience — maintenance cost.
   - Candidate D: Most granular, but theme-as-CSS-variables breaks dynamic
     theme objects used in 3 charting components that need JS theme values.

4. ROLLOUT PROBES:
   - Grep for theme usage: confirmed 3 chart components use theme.colors
     in JS, not just CSS — rules out Candidate D.
   - Check test coverage: auth flow has 89% coverage, prefs has 34% —
     Candidate B's separation lets us refactor auth confidently.

5. COMPARE:
   Reject A (unnecessary re-renders at scale), reject C (added dependency
   without proportional benefit), reject D (breaks chart components).
   Select B: two-context split matches actual data change patterns and
   minimizes re-render blast radius.

6. EXECUTE: Create AuthContext first (higher test coverage = safer),
   migrate the 12 auth consumers, verify tests pass, then create
   PrefsContext for remaining consumers.
```

**Example 2: Resolving a circular dependency**

```
User: I have a circular import between services/auth.ts and services/user.ts.
      Fix it.

Approach (ProAct Lookahead):
1. OBSERVE: auth.ts imports getUserPermissions from user.ts. user.ts imports
   validateToken from auth.ts. Both are used in 14 other files.

2. ENUMERATE CANDIDATES:
   A) Extract shared types/functions into services/shared.ts
   B) Merge into a single services/auth-user.ts module
   C) Use dependency injection — auth accepts a permission-checker function
      instead of importing user.ts directly
   D) Lazy import: auth.ts uses dynamic import() for getUserPermissions

3. SIMULATE FORWARD:
   - Candidate A: shared.ts would contain validateToken and getUserPermissions.
     But these are not "shared utilities" — they have real business logic and
     their own dependencies. Moving them orphans their tests and creates a
     grab-bag module that will accumulate unrelated code over time.
   - Candidate B: 800-line merged file. Violates single-responsibility.
     Git blame becomes useless for the merge commit. 14 import sites need
     updating.
   - Candidate C: auth.ts exports a factory `createAuthService(deps: {
     getPermissions: ... })`. Breaks the cycle cleanly. user.ts continues
     importing from auth.ts (no cycle — auth no longer imports user).
     Only auth.ts and its initialization site change.
   - Candidate D: Works but hides the dependency, makes it async where it
     was sync, and dynamic imports are harder to tree-shake.

4. ROLLOUT PROBES:
   - Check how auth.ts uses getUserPermissions: called in 2 functions,
     both already async — injection is straightforward.
   - Check initialization: app.ts already creates both services — natural
     place to wire the dependency.

5. COMPARE:
   Reject A (creates a junk-drawer module), reject B (too coarse),
   reject D (unnecessary async conversion and bundler implications).
   Select C: dependency injection breaks the cycle at the right seam,
   changes are minimal (auth.ts + app.ts), and no other files need updating.

6. EXECUTE: Refactor auth.ts to accept getPermissions as a parameter,
   update app.ts to inject it, verify the 14 downstream consumers still
   work unchanged.
```

**Example 3: Database schema migration with lookahead**

```
User: We need to add multi-tenancy to our PostgreSQL schema. Every table
      needs a tenant_id column.

Approach (ProAct Lookahead):
1. OBSERVE: 23 tables, 47 foreign key relationships, 12 unique constraints,
   8 database queries use JOINs across tenant-unaware tables. ORM is Prisma.

2. ENUMERATE CANDIDATES:
   A) Add tenant_id to every table, update every query manually
   B) Use PostgreSQL Row-Level Security (RLS) with a session variable
   C) Schema-per-tenant (separate PostgreSQL schemas)

3. SIMULATE FORWARD:
   - Candidate A: 23 migrations, every unique constraint needs tenant_id
     added to become a compound key, all 8 JOIN queries need WHERE
     tenant_id clauses. Risk: one missed query leaks data cross-tenant.
   - Candidate B: Add tenant_id column + RLS policies. Queries don't change —
     RLS filters automatically. But: Prisma doesn't support SET
     session variables natively; need raw SQL middleware. Risk: if middleware
     fails to set tenant, RLS defaults to showing nothing (safe) or
     everything (catastrophic) — must verify default policy.
   - Candidate C: Complete data isolation, no query changes needed. But:
     schema count grows linearly with tenants, migrations must run N times,
     connection pooling becomes complex at >100 tenants.

4. ROLLOUT PROBES:
   - Check Prisma docs: confirmed no native RLS support, but $executeRaw
     works for SET commands in middleware.
   - Check expected tenant count: README says "targeting enterprise, 10-50
     tenants" — schema-per-tenant is viable at this scale.
   - Check RLS default: PostgreSQL RLS default is DENY (restrictive) —
     safe failure mode.

5. COMPARE:
   Candidate A has highest data-leak risk (manual query updates).
   Candidate C is viable at 10-50 tenants but creates migration complexity.
   Select B: RLS provides automatic enforcement, safe default-deny,
   and requires no query changes. The Prisma middleware is the only
   new code needed.

6. EXECUTE: Write migration adding tenant_id + RLS policies for one
   table first. Test with Prisma middleware. Verify default-deny behavior.
   Then apply pattern to remaining 22 tables.
```

## Best Practices

- **Do:** Ground every prediction in observed code. Read the actual file before simulating what happens if you change it. Hallucinated dependencies are the primary failure mode.
- **Do:** Explicitly state why rejected alternatives were rejected. This contrastive reasoning is what distinguishes ProAct lookahead from ordinary planning — it forces you to actually compare, not just rationalize the first idea.
- **Do:** Use cheap verification probes (grep, type-check, test run) whenever a prediction is uncertain. A 5-second grep is always cheaper than a wrong refactor.
- **Do:** Re-evaluate after each execution step. Lookahead predictions degrade over time as the codebase state diverges from what you simulated. Reground after each change.
- **Avoid:** Analyzing more than 3-4 candidates. Diminishing returns set in fast. Pick the most distinct options that span the solution space.
- **Avoid:** Spending lookahead effort on trivially reversible changes. If a change is a one-line edit with no dependencies, just make it. Reserve structured lookahead for changes with cascading or hard-to-reverse consequences.

## Error Handling

| Failure Mode | Symptom | Recovery |
|---|---|---|
| Hallucinated dependency | You predict file X imports Y, but it doesn't | Always grep to verify import relationships before simulating cascading effects |
| Stale mental model | Your lookahead was correct at step 1 but wrong by step 4 because earlier edits changed the landscape | Re-read affected files after each edit increment; don't rely on cached assumptions |
| Analysis paralysis | 30 minutes of lookahead reasoning for a 5-minute change | Apply a proportionality check: if the change touches <3 files and has tests, reduce lookahead to a single paragraph |
| Overconfident rollout | You skipped verification probes and your "certain" prediction was wrong | Default to verifying. Run the type-checker or test after the first increment rather than batching all changes |
| Anchoring on first candidate | You generate candidates but always pick the first one | Write the rejection reasons for candidate 1 first to counter anchoring bias |

## Limitations

- **Lookahead depth is bounded by codebase familiarity.** In an unfamiliar codebase, your simulations are only as good as what you've read. The technique works best after an initial exploration phase.
- **Not suitable for trivial tasks.** Adding a CSS class, fixing a typo, or updating a version number does not warrant multi-candidate lookahead. Use judgment on when the overhead is justified.
- **Cannot replace runtime verification.** Lookahead reasoning predicts consequences but does not prove them. Always verify critical predictions with actual tool invocations (tests, type-checkers, grep).
- **Stochastic environments are harder.** When outcomes depend on external factors (network calls, user behavior, race conditions), lookahead predictions carry inherently higher uncertainty. Acknowledge this and keep rollout probes focused on what you can observe deterministically.
- **Diminishing returns beyond 2-3 steps.** Simulating 5+ steps into the future compounds uncertainty rapidly. Focus lookahead energy on the first 2-3 consequences, then re-evaluate after executing.

## Reference

**Paper:** [ProAct: Agentic Lookahead in Interactive Environments](https://arxiv.org/abs/2602.05327v1) (Yu et al., 2026)
**Key takeaway:** Compressing environment search trees into concise Observation-Analysis-Conclusion reasoning chains (GLAD) lets an agent internalize accurate foresight without expensive inference-time search. The contrastive format — explaining why alternatives were rejected — is critical to learning quality.
**Code:** [github.com/GreatX3/ProAct](https://github.com/GreatX3/ProAct)