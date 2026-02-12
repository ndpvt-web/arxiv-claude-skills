---
name: "agyn-multi-agent-system-team-based"
description: "Orchestrate multi-agent teams for autonomous software engineering using the Agyn methodology: coordinator, researcher, implementer, and reviewer agents with structured communication, isolated sandboxes, and iterative review loops. Use when: 'set up a multi-agent team to fix this bug', 'use agent swarm to implement this feature', 'resolve this GitHub issue with a team of agents', 'coordinate agents to refactor this module', 'spin up an engineering team to tackle this task', 'use Agyn-style agents to solve this problem'."
---

This skill enables Claude to orchestrate autonomous software engineering teams modeled after the Agyn architecture (Benkovich & Valkov, 2026). Instead of treating issue resolution as a single monolithic prompt, Claude decomposes work across four specialized agent roles — Coordinator, Researcher, Implementer, and Reviewer — each operating in isolated sandboxes with structured inter-agent communication. The agents follow a defined development methodology: analysis, task specification, implementation, and iterative review, mirroring how real engineering teams operate. This approach resolves 72.2% of SWE-bench 500 tasks, outperforming comparable single-agent baselines.

## When to Use

- When the user asks to resolve a GitHub issue or bug that requires codebase exploration, implementation, and validation
- When the user wants a multi-agent swarm to implement a feature end-to-end
- When the user needs to refactor a module and wants structured research before changes are made
- When the user has a complex task that benefits from separating investigation from implementation from review
- When the user explicitly asks for an "Agyn-style" or "team-based" agent workflow
- When the user wants parallel agents to collaborate on a pull request with iterative review
- When a task involves unfamiliar code where research must precede implementation

## Key Technique

The Agyn system's core insight is that **organizational design matters as much as model capability**. Rather than feeding a single agent an issue description and hoping it produces a correct patch, Agyn replicates the structure of a real engineering team. Four agents — Coordinator, Researcher, Implementer, and Reviewer — communicate through structured messages, each with a narrow mandate and isolated execution environment. The Coordinator decomposes problems and routes work; the Researcher analyzes the codebase to produce structured findings; the Implementer writes code guided by those findings; and the Reviewer validates against the original requirements, triggering re-research when needed.

The iterative review loop is what distinguishes Agyn from pipeline-based multi-agent systems. When the Reviewer identifies failures — test regressions, unmet requirements, logical errors — it produces specific feedback that flows back through the Coordinator to the Researcher, who re-analyzes the relevant code sections. The Implementer then creates targeted fixes rather than rewriting from scratch. This loop continues until the Reviewer passes the changes or an iteration limit is reached. This progressive refinement mimics the PR review cycle in real teams and prevents the compounding errors that plague single-pass approaches.

Sandbox isolation is the third pillar. Each agent operates within bounded environments with defined resource constraints and repository access limits. This prevents agents from making unintended modifications, enables safe experimentation (e.g., the Researcher can explore code paths without side effects), and ensures that only the Implementer's final, reviewed changes are applied. Structured communication — messages containing task parameters, file references, and execution constraints — replaces the lossy context passing of monolithic prompts.

## Step-by-Step Workflow

1. **Receive and analyze the issue**: Parse the user's request into a concrete issue specification with success criteria, target repository paths, and constraints. If the user provides a GitHub issue URL, fetch its details. Identify whether this is a bug fix, feature implementation, or refactoring task.

2. **Spawn the Coordinator agent**: Create a team using `TeamCreate` and launch a general-purpose agent as the Coordinator. The Coordinator owns the task list, decomposes the problem, and manages all inter-agent routing. It does NOT write code — it orchestrates.

3. **Launch the Researcher agent in an Explore role**: Spawn an Explore-type agent (read-only) tasked with codebase analysis. The Researcher's mandate is: identify relevant files, trace execution paths, map dependencies, understand existing patterns, and produce a structured research report. Give it the issue description and any known entry points.

4. **Collect research findings into a task specification**: The Coordinator receives the Researcher's report — a list of relevant files, root cause analysis (for bugs), or design context (for features) — and produces a precise task specification. This spec includes: files to modify, the expected behavior, constraints to respect, and test commands to validate.

5. **Launch the Implementer agent**: Spawn a general-purpose agent (with write access) that receives the task specification and research findings. The Implementer writes code changes, runs tests in its sandbox, and produces a candidate patch. It should NOT explore the codebase broadly — that was the Researcher's job.

6. **Launch the Reviewer agent**: Spawn a general-purpose agent to review the Implementer's changes. The Reviewer reads the diff, runs the test suite, checks alignment with the original issue requirements, and produces a structured review: PASS (with rationale) or FAIL (with specific issues and file:line references).

7. **Handle review feedback in an iterative loop**: If the Reviewer returns FAIL, the Coordinator routes the specific feedback back. For understanding-related failures, re-engage the Researcher to gather additional context. For implementation-related failures, send targeted fix instructions to the Implementer. Each iteration should be narrowly scoped — fix the specific failure, don't rewrite.

8. **Cap iterations and converge**: Set a maximum of 3 review iterations. If the patch passes, apply it. If the iteration limit is reached with a still-failing review, present the best attempt to the user with a summary of remaining issues and the Reviewer's final feedback.

9. **Aggregate results and clean up**: Collect the final diff, test results, and a summary of the methodology (research findings, implementation decisions, review outcomes). Shut down all agents gracefully using `SendMessage` with `shutdown_request`. Clean up the team with `TeamDelete`.

10. **Present the solution**: Show the user the final changes with context: what was found during research, what approach was chosen and why, what the Reviewer validated, and any caveats.

## Concrete Examples

**Example 1: Fixing a bug reported in a GitHub issue**

```
User: "Resolve this issue — users report that the /api/export endpoint returns
a 500 error when the dataset has null values in the 'timestamp' column."

Approach:
1. Coordinator creates tasks: research the export endpoint, identify null handling,
   implement fix, validate with tests.

2. Researcher (Explore agent) traces the /api/export route:
   - Finds route handler in src/routes/export.ts:45
   - Follows data flow to src/services/exporter.ts:112 where timestamps are
     formatted without null checks
   - Identifies that formatTimestamp() in src/utils/date.ts:23 throws on null input
   - Notes existing test file: tests/export.test.ts (no null-value test case)

   Research report:
   - Root cause: formatTimestamp() at src/utils/date.ts:23 calls .toISOString()
     on null
   - Fix location: src/utils/date.ts:23 or src/services/exporter.ts:112
   - Related pattern: other formatters in date.ts handle null (see formatDate:31)
   - Test gap: no null-column test in tests/export.test.ts

3. Coordinator produces task spec:
   - Add null guard in formatTimestamp() following existing pattern from formatDate()
   - Add test case for null timestamp column in export test suite
   - Run: npm test -- --grep "export"

4. Implementer adds null check and test case, runs tests — all pass.

5. Reviewer validates:
   - Diff is minimal and follows existing patterns: PASS
   - New test covers the exact failure scenario: PASS
   - No regressions in test suite: PASS
   - Review: APPROVED

Output: Two-file patch (src/utils/date.ts, tests/export.test.ts) with explanation.
```

**Example 2: Implementing a new feature with unfamiliar codebase**

```
User: "Add rate limiting to all public API endpoints. Use a sliding window
algorithm. I want it configurable per-route."

Approach:
1. Coordinator identifies subtasks: research API routing layer, research
   existing middleware patterns, implement rate limiter, implement per-route
   config, review.

2. Researcher analyzes:
   - Middleware chain in src/middleware/index.ts — finds auth, logging, cors
   - Route registration in src/routes/index.ts — finds registerPublicRoutes()
   - Config pattern in src/config/index.ts — env-based with defaults
   - Existing Redis connection in src/services/redis.ts (available for state)

   Research report:
   - Middleware pattern: export function, register in chain at src/middleware/index.ts:8
   - Config pattern: add to src/config/index.ts with RATE_LIMIT_ prefix
   - Redis client available at src/services/redis.ts:getClient()
   - Public routes identifiable via registerPublicRoutes() in src/routes/index.ts:34
   - Per-route config: recommend decorator/metadata pattern matching existing
     route options at src/routes/types.ts:12

3. Implementer creates:
   - src/middleware/rateLimit.ts — sliding window using Redis sorted sets
   - Adds route-level config option to src/routes/types.ts
   - Registers middleware in src/middleware/index.ts
   - Adds config defaults in src/config/index.ts
   - Writes tests in tests/middleware/rateLimit.test.ts

4. Reviewer flags: "Rate limit key uses IP only — behind a load balancer
   this will limit all users together. Should use X-Forwarded-For with fallback."

5. Iteration 2: Implementer fixes key extraction to use X-Forwarded-For header
   with configurable trust proxy setting.

6. Reviewer: APPROVED — all tests pass, pattern matches existing middleware.

Output: Multi-file patch with rate limiting middleware, per-route configuration,
and test suite.
```

**Example 3: Coordinating a refactoring task**

```
User: "Refactor the authentication module to use JWT instead of session cookies.
Don't break existing endpoints."

Approach:
1. Coordinator creates phased plan: research current auth flow, research all
   consumers, implement JWT auth, implement backward compatibility, review.

2. Researcher maps the full auth surface:
   - Session creation in src/auth/session.ts
   - 14 endpoints that call req.session.user
   - 3 middleware functions checking session validity
   - Frontend cookie handling in src/client/api.ts
   - Test fixtures using session mocking in tests/helpers/auth.ts

3. Coordinator creates a task spec prioritizing backward compatibility:
   - Support both JWT (Authorization header) and session cookies during migration
   - New JWT middleware that falls back to session check

4. Implementer creates JWT utilities, dual-mode auth middleware, and updates
   test helpers. Runs full test suite.

5. Reviewer catches: "The refresh token endpoint still creates a session object
   — this will leak sessions even for JWT-authenticated users."

6. Iteration 2: Researcher confirms the refresh endpoint at src/auth/refresh.ts:28
   creates sessions unconditionally. Implementer adds a conditional check.

7. Reviewer: APPROVED after verifying all 14 endpoints work with both auth methods.

Output: Comprehensive refactoring patch with dual-mode authentication,
no breaking changes, and full test coverage.
```

## Best Practices

**Do:**
- Keep agent mandates narrow — the Researcher should ONLY investigate and report, never modify files. The Implementer should follow the task spec, not explore broadly.
- Pass structured data between agents — file paths with line numbers, specific function names, concrete error messages. Vague summaries lose information across the handoff boundary.
- Use the Reviewer's feedback verbatim when routing back to the Implementer. Don't summarize or reinterpret review comments — specificity prevents drift.
- Set iteration caps (3 iterations is the sweet spot). Unbounded loops waste resources; too few iterations miss fixable issues.

**Avoid:**
- Don't skip the research phase for "simple" issues. The Researcher frequently uncovers context (related patterns, test gaps, hidden dependencies) that prevents implementation mistakes.
- Don't let the Implementer agent also do research. Role separation is what makes this approach outperform single-agent baselines. Combining roles reintroduces the monolithic-agent failure mode.
- Don't broadcast messages to all agents when only one needs the information. Use targeted `SendMessage` to the specific agent that needs to act.
- Don't re-create agents between iterations. Resume existing agents so they retain context from prior rounds.

## Error Handling

- **Researcher finds no relevant files**: Widen the search scope. If the codebase structure is unusual, have the Researcher start from entry points (main files, route registrations, config) and trace outward. If still blocked, ask the user for hints about file locations.
- **Implementer's changes break existing tests**: This is a Reviewer catch. Route the specific failing test names and error output back through the Coordinator. The Researcher re-analyzes the test expectations, and the Implementer adjusts.
- **Review loop hits iteration limit**: Present the current best patch with a clear summary: what works, what still fails, and the Reviewer's last feedback. Let the user decide whether to accept, manually fix, or provide additional guidance.
- **Agent communication failure or timeout**: If a spawned agent fails, the Coordinator should retry once with the same prompt. If it fails again, fall back to single-agent mode for that role and note the degradation to the user.
- **Conflicting research findings**: When the Researcher reports ambiguous or contradictory information (e.g., two different patterns used for the same concern), escalate to the Coordinator to ask the user which pattern to follow.

## Limitations

- **Overhead for simple tasks**: A four-agent team is overkill for single-line fixes, typo corrections, or config changes. Use this approach only when the task genuinely benefits from separated research, implementation, and review phases. A good heuristic: if you can confidently identify the fix location and change without codebase exploration, skip the team.
- **Context window pressure**: Each agent gets its own context, but inter-agent messages must carry enough information to be self-contained. Very large codebases with deeply entangled dependencies can exceed what structured messages can convey. In these cases, the Researcher should produce focused, prioritized findings rather than exhaustive reports.
- **No shared memory between agents**: Agents communicate only through explicit messages. Implicit knowledge (e.g., "I noticed something odd in file X but it wasn't relevant to the current task") is lost. Encourage the Researcher to note peripheral findings in its report.
- **Iteration limits vs. complex bugs**: Some bugs require more than 3 research-implement-review cycles. The hard cap prevents infinite loops but may terminate before convergence on genuinely difficult issues. The user can re-invoke with refined guidance.
- **Model capability remains the floor**: Team structure amplifies model capabilities but cannot compensate for fundamental model limitations in code understanding or generation. If the underlying model cannot reason about the code, adding more agents won't help.

## Reference

[Agyn: A Multi-Agent System for Team-Based Autonomous Software Engineering](https://arxiv.org/abs/2602.01465v2) — Benkovich & Valkov, 2026. Focus on Section 3 (System Architecture) for agent role definitions and communication protocols, and Section 4 (Development Methodology) for the iterative review loop that drives the 72.2% SWE-bench resolution rate.