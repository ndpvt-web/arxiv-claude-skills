---
name: "agyn-multi-agent-system-team-based"
description: "Organize multi-agent teams with specialized roles (manager, researcher, engineer, reviewer) to solve complex software engineering issues through structured methodology: research, task specification, implementation, and iterative review. Use when: 'set up a team to fix this bug', 'use agents to resolve this issue', 'multi-agent software engineering', 'team-based issue resolution', 'organize agents to implement this feature', 'coordinate agents with roles to solve this'."
---

This skill enables Claude to orchestrate multi-agent teams that mirror real engineering organizations for autonomous software engineering. Based on the Agyn system (Benkovich & Valkov, 2026), it decomposes issue resolution into four specialized roles -- manager, researcher, engineer, and reviewer -- coordinated through a manager-centric hub with structured methodology phases. The key insight: replicating team structure, methodology, and communication outperforms single-agent approaches by 7%+ on standard benchmarks, because role separation lets each agent optimize for its specific task (exploration vs. implementation vs. critique) rather than context-switching within a monolithic prompt.

## When to Use

- When the user asks to resolve a GitHub issue or bug that requires codebase exploration, implementation, and validation
- When implementing a non-trivial feature that benefits from separation of research, coding, and review
- When the user explicitly requests a multi-agent or team-based approach to a software task
- When tackling a complex codebase change where understanding the problem and fixing it are distinct cognitive tasks
- When the user wants iterative code review built into the implementation process
- When working on production codebases where a single-pass approach risks missing edge cases

## Key Technique

**Manager-Centric Multi-Agent Coordination**: Unlike pipeline systems where agents run in a fixed sequence, Agyn uses a manager agent as the single coordination point. The manager dynamically decides when to invoke the researcher (for more exploration), the engineer (for implementation), or the reviewer (for quality checks). All inter-agent communication flows through the manager -- agents never talk directly to each other. This simplifies control flow and makes the entire process traceable.

**Methodology Over Steps**: The system follows a development methodology (research -> specification -> implementation -> review) but does not prescribe how many steps each phase takes. The manager can loop back to research if the engineer's implementation reveals new complexity, or request additional review rounds. The reviewer uses an explicit approve-or-request-changes mechanism: the loop continues until the reviewer approves, providing a concrete acceptance signal rather than a heuristic stopping condition.

**Isolated Sandboxes With Full Tooling**: Each agent operates in its own environment with shell access, git, and package management. Engineers can freely explore the codebase, test partial solutions, and discard failed attempts without polluting other agents' state. This mirrors how real developers work on feature branches -- experimentation is cheap and reversible.

## Step-by-Step Workflow

1. **Spawn a team with four named agents**: Create a team using TeamCreate, then spawn agents for each role: `manager` (general-purpose, coordinates), `researcher` (Explore-type, investigates codebase), `engineer` (general-purpose, writes code), and `reviewer` (general-purpose, reviews changes). The manager is the team lead.

2. **Manager analyzes the issue**: The manager reads the issue description, identifies the repository, and formulates the high-level problem. It decides what the researcher needs to investigate first -- root cause analysis, affected files, relevant tests, and API contracts.

3. **Researcher explores the codebase**: The researcher agent searches for relevant code paths, reads related files, identifies the root cause or the integration points for a new feature. It produces a structured research summary: affected files, root cause hypothesis, relevant tests, and dependencies.

4. **Manager synthesizes a task specification**: The manager takes the researcher's output and writes a concrete, unambiguous task specification: what files to modify, what behavior to change, what tests should pass, and what constraints to respect. This specification is the contract between research and implementation.

5. **Engineer implements the solution**: The engineer receives the task specification and works in its sandbox -- modifying files, running tests, and iterating until the implementation matches the spec. It creates a branch and opens a pull request with a descriptive title including the task identifier.

6. **Reviewer inspects the pull request**: The reviewer reads the diff, checks for correctness against the task specification, verifies test coverage, and looks for edge cases. It either approves or leaves inline comments requesting specific changes.

7. **Iterate until approval**: If the reviewer requests changes, the manager routes the feedback to the engineer, who addresses each comment and pushes updates. The reviewer re-inspects. This loop continues until the reviewer explicitly approves.

8. **Manager validates and finalizes**: Once the reviewer approves, the manager performs a final check -- confirms tests pass, the PR is clean, and signals completion. The manager uses the `finish` action only when the entire methodology has been satisfied.

9. **Handle escalation**: If the engineer is stuck after multiple iterations, the manager routes the problem back to the researcher for additional investigation rather than letting the engineer thrash. This prevents wasted cycles on incorrect assumptions.

10. **Shut down the team**: Once the task is complete and verified, the manager sends shutdown requests to all agents and cleans up the team resources.

## Concrete Examples

**Example 1: Resolving a Bug in a Django Application**

```
User: "Use a team of agents to fix issue #247 -- the user profile API
returns 500 when the bio field is null"

Manager's analysis:
- Issue: NullPointerError in profile serialization
- Need: Find serializer, identify null handling gap, fix, test

Step 1 - Manager spawns team, assigns researcher to investigate:
  "Find the profile serializer, the API view that serves /api/profile/,
   and any existing tests for null bio fields."

Step 2 - Researcher reports back:
  "Root cause: ProfileSerializer in api/serializers.py:45 calls
   bio.strip() without null check. View is in api/views.py:112.
   Test file: tests/test_profile_api.py has no null-bio test case."

Step 3 - Manager writes task spec:
  - Modify api/serializers.py: add null guard before bio.strip()
  - Add test in tests/test_profile_api.py: test_profile_null_bio
  - Ensure existing tests still pass

Step 4 - Engineer implements fix, opens PR, runs tests (all pass)

Step 5 - Reviewer checks PR:
  "The fix handles None but not empty string. Add a test for
   bio='' to confirm strip() works on empty strings too."

Step 6 - Engineer adds empty-string test, pushes update

Step 7 - Reviewer approves. Manager finalizes.
```

**Example 2: Implementing a New Feature Across Multiple Files**

```
User: "Set up a multi-agent team to add CSV export to the reports module"

Manager's analysis:
- Feature: Add CSV download endpoint for existing report data
- Need: Understand report data model, existing export patterns, API conventions

Step 1 - Researcher investigates:
  "Reports are generated in reports/generator.py, served via
   reports/views.py. Existing PDF export uses reports/exporters/pdf.py
   pattern. Data models in reports/models.py. Tests in tests/test_exports.py."

Step 2 - Manager writes task spec:
  - Create reports/exporters/csv.py following pdf.py's interface pattern
  - Add GET /api/reports/{id}/export/csv endpoint in reports/views.py
  - Add URL route in reports/urls.py
  - Add tests: test_csv_export_success, test_csv_export_empty_report,
    test_csv_export_special_characters
  - Use Python's csv module, stream response with appropriate headers

Step 3 - Engineer implements across 4 files, runs tests

Step 4 - Reviewer flags: "CSV export doesn't handle Unicode BOM for
  Excel compatibility. Add utf-8-sig encoding."

Step 5 - Engineer fixes encoding, reviewer approves

Step 6 - Manager confirms all tests pass, signals completion
```

**Example 3: Refactoring with Safety Net**

```
User: "Organize agents to refactor the payment processing from
synchronous to async"

Manager's analysis:
- High-risk refactor affecting critical path
- Need thorough research on all call sites before any code changes

Step 1 - Researcher maps all synchronous call sites:
  "Found 12 call sites across 6 files. 3 are in request handlers
   (must become async), 2 are in management commands (can stay sync
   with async_to_sync wrapper), 7 are in Celery tasks (already async
   context)."

Step 2 - Manager writes phased task spec:
  Phase A: Convert core payment function to async
  Phase B: Update 3 request handlers to async views
  Phase C: Add async_to_sync wrappers for management commands
  Phase D: Verify Celery tasks work with async function

Step 3 - Engineer implements phase by phase, running tests after each

Step 4 - Reviewer catches: "The webhook handler in phase B lost its
  @csrf_exempt decorator during conversion."

Step 5 - Engineer restores decorator, reviewer approves

Step 6 - Manager runs full test suite one final time, confirms green
```

## Best Practices

- **Do**: Have the manager write an explicit task specification before the engineer starts coding. Ambiguous handoffs are the primary failure mode in multi-agent systems.
- **Do**: Use the researcher for codebase exploration rather than having the engineer search while implementing. Separating exploration from implementation reduces context pollution.
- **Do**: Let the review loop run until explicit approval. Premature termination (e.g., "good enough after 2 rounds") undermines the quality guarantee the reviewer provides.
- **Do**: Route the engineer's failures back to the researcher. When implementation reveals that the initial analysis was wrong, more research is cheaper than more coding attempts.
- **Avoid**: Having agents communicate directly with each other. All coordination must flow through the manager to maintain a single source of truth about task state.
- **Avoid**: Prescribing a fixed number of iterations. The methodology defines phases, not step counts -- the manager decides dynamically when each phase is complete.
- **Avoid**: Giving every agent the same broad prompt. Each role should have a focused system prompt that constrains it to its specialty (research, implementation, or review).

## Error Handling

- **Researcher finds no clear root cause**: The manager should ask the researcher to broaden scope -- check logs, related modules, or upstream dependencies. If still unclear, the manager formulates a hypothesis-driven task spec with explicit "verify assumption X" steps for the engineer.
- **Engineer's tests fail after implementation**: Route the test output back through the manager to decide whether this is a implementation bug (send back to engineer) or a misunderstanding of the problem (send back to researcher).
- **Reviewer and engineer disagree**: The manager arbitrates by re-reading the task specification. If the spec is ambiguous, the manager rewrites it with more precision rather than letting agents argue.
- **Context window pressure in long tasks**: Summarize completed phases into concise status updates before starting new phases. Each agent should receive only the context relevant to its current task, not the entire conversation history.
- **Agent stuck in a loop**: If an engineer-reviewer cycle exceeds 4 iterations on the same issue, the manager should intervene -- either simplify the task spec, request fresh research, or decompose the problem into smaller sub-tasks.

## Limitations

- **Overhead on simple tasks**: For straightforward single-file fixes (typos, config changes, obvious one-liners), the multi-agent ceremony adds coordination cost without proportional quality gain. Use a single agent for trivial tasks.
- **Model cost scales with agents**: Each agent consumes its own token budget. A 4-agent team uses roughly 3-4x the tokens of a single agent. The quality improvement justifies this for complex tasks but not for simple ones.
- **Communication bottleneck**: Since all coordination flows through the manager, the manager's ability to synthesize and route information is the system's ceiling. A confused manager propagates confusion to all agents.
- **No true parallelism in review**: The researcher and engineer cannot meaningfully work in parallel on the same issue because the engineer depends on the researcher's output. Parallelism is only useful when handling multiple independent sub-tasks.
- **Repository-specific knowledge**: The researcher's effectiveness depends on the codebase being navigable (reasonable file names, some documentation, test coverage). Heavily obfuscated or undocumented codebases degrade research quality.

## Reference

[Agyn: A Multi-Agent System for Team-Based Autonomous Software Engineering](https://arxiv.org/abs/2602.01465v2) (Benkovich & Valkov, 2026). Key sections: the manager-centric coordination architecture, the development methodology phases, and Table 1 comparing multi-agent vs. single-agent performance on SWE-bench 500 (72.2% vs. 65-71.8% for single-agent baselines).