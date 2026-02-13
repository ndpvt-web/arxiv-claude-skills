---
name: "haif-human-ai-integration-framework"
description: "Apply the HAIF protocol to organize hybrid human-AI team workflows with tiered autonomy, delegation governance, and validation checklists. Use when: 'set up HAIF for our team', 'classify this task for AI delegation', 'generate a delegation registry', 'create AI validation checklists', 'add HAIF tier metadata to our sprint board', 'assess autonomy tier for this task'."
---

This skill enables Claude to apply the Human-AI Integration Framework (HAIF) from Bara (2026) to real software teams. HAIF is a protocol-based operational system for teams where AI agents perform substantive work alongside humans. It provides four core principles (named ownership, governed delegation, proportional validation, active competence maintenance), a formal delegation decision matrix mapping task properties to autonomy tiers, and concrete integration points for Agile/Kanban workflows. Claude uses this skill to classify tasks, generate delegation registries, produce domain-specific validation checklists, and structure sprint ceremonies around hybrid human-AI delivery.

## When to Use

- When a user wants to introduce structured AI delegation into their team's workflow (e.g., "how should we govern AI-generated code in our sprints?")
- When classifying whether a task should be AI-assisted, AI-supervised, autonomously monitored, or human-only
- When generating validation checklists for AI-produced code, documents, or data pipelines
- When setting up a delegation registry to track what AI does, at what autonomy level, and who owns it
- When adapting sprint planning, daily standups, reviews, or retrospectives for hybrid human-AI teams
- When a team needs tier transition criteria (when to promote or demote AI autonomy on a task type)
- When assessing whether AI delegation is appropriate given task structure, verifiability, consequence of error, and demonstrated AI capability

## Key Technique

HAIF addresses an operational gap: existing frameworks (Agile, DevOps, MLOps, AI governance) each cover adjacent concerns but none models the hybrid human-AI team as a coherent delivery unit. HAIF fills this by treating AI delegation as a first-class workflow concern with four principles: (1) every AI output has a named human owner accountable for it, (2) delegation decisions are explicit, governed by assigned tiers, and reversible without stigma, (3) validation effort is proportional to autonomy tier and budgeted as planned capacity, and (4) human competence is actively maintained through periodic AI-restricted work blocks.

The framework's core mechanism is a **delegation decision matrix** that maps four assessments — Structuredness (S), Verifiability (V), Consequence of Error (C), and AI Demonstrated Capability (D) — to one of four autonomy tiers. Tier 1 (Assisted) means AI supports human execution. Tier 2 (Supervised) means AI produces output that a human reviews before delivery. Tier 3 (Autonomously Monitored) means AI produces output with post-hoc sampling at a defined rate. Tier 4 (Autonomously Bounded) means AI operates independently within defined parameters. Tier promotion is slow and evidence-based (3-8 cycles minimum); demotion is immediate and frictionless. This asymmetry is deliberate — it embodies the adoption paradox: the more capable AI appears, the more oversight matters.

Integration with existing workflows is lightweight. In Scrum, every backlog item receives a tier tag and human owner during sprint planning, validation time is explicitly budgeted, and retrospectives add hybrid-specific questions. In Kanban, tier classification happens at the commitment point, and items cannot move to "Done" without completing tier-specific validation. For solo practitioners or pairs, the registry is a personal log; for departments, it becomes a formal system with a dedicated validation lead.

## Step-by-Step Workflow

1. **Assess the four delegation inputs for the task.** Evaluate Structuredness (are inputs/outputs well-defined?), Verifiability (can the output be objectively checked?), Consequence of Error (internal-correctable vs. legal/safety/reputational), and AI Demonstrated Capability (Unproven / Emerging / Established / Mature on *this specific task type*). Rate each Low/Medium/High (or the D scale).

2. **Consult the delegation decision matrix to determine the default tier.** Apply these rules:
   - S:High, V:High, D:Mature, C:Low → Tier 4; C:Medium → Tier 3; C:High → Tier 2
   - S:High, V:High, D:Established, C:Low → Tier 3; C:Medium/High → Tier 2
   - S:High, V:Medium, D:Established → Tier 3 (C:Low), Tier 2 (C:Med), Tier 1 (C:High)
   - S:Medium, V:Medium, D:Established → Tier 2 (C:Low/Med), Tier 1 (C:High)
   - S:Medium, V:Medium, D:Emerging → Tier 2 (C:Low), Tier 1 (C:Med), AI-restricted (C:High)
   - S:Low or V:Low (any C) → Tier 1 or AI-restricted
   - D:Unproven (any combination) → Tier 1 pilot or AI-restricted

3. **Assign a named human owner.** Record the person who accepts accountability for the AI output regardless of tier. No AI output crosses a team boundary without this assignment.

4. **Budget validation effort explicitly.** For Tier 1, validation is inherent in the human process. For Tier 2, budget 30-60% of original human effort for full review plus domain checklist. For Tier 3, budget monitoring + sampling at a defined rate (start at 20% if no historical data). For Tier 4, budget boundary maintenance + periodic audit + exception handling.

5. **Select and adapt the domain validation checklist.** Use the appropriate checklist (code generation, document generation, data analysis, or a custom one) and tailor it to the team's stack and domain. Record what was checked, findings, and acceptance decisions.

6. **Record the delegation in the registry.** Create or update a registry entry with: Task Type, Current Tier, Human Owner, S/V/C/D assessment, Evidence basis, Last Reviewed date, and performance data (error rates, review time, false acceptance rates).

7. **Execute with tier-appropriate oversight.** Tier 1: human drives, AI assists. Tier 2: AI produces, human reviews every output pre-delivery. Tier 3: AI produces, human samples at p% + automated checks. Tier 4: AI operates within parameters, human handles exceptions and audits.

8. **Capture feedback at cycle close.** Record validation findings (error rates, error types, first-pass acceptance rate). Flag process violations. Update registry with performance data.

9. **Evaluate tier transitions in retrospective.** Promotion requires minimum cycles at current tier (3 for T1→T2, 5 for T2→T3, 8 for T3→T4), error rate below threshold, and zero critical errors. Demotion is immediate upon any critical error, consecutive threshold breaches, or insufficient validation capacity.

10. **Schedule competence maintenance.** For Tier 2+ delegations, schedule weekly "AI-restricted" work blocks where team members execute the delegated task type manually to preserve independent capability.

## Concrete Examples

**Example 1: Classifying a task for AI delegation**

User: "We want to use AI to generate unit tests for our payment service. How should we govern this?"

Approach:
1. Assess inputs — S: High (unit tests have clear inputs/outputs and patterns), V: High (tests either pass or fail, coverage is measurable), C: Medium (bad tests could miss payment bugs reaching production, but CI catches regressions), D: Emerging (AI generates tests but team hasn't systematically measured quality yet).
2. Consult matrix — S:High, V:High, D:Emerging, C:Medium → Tier 1 (pilot).
3. Recommend starting at Tier 1: human writes tests with AI assistance, tracking AI suggestion acceptance rate.

Output:
```
## Delegation Assessment: AI-Generated Unit Tests (Payment Service)

| Input           | Rating   | Rationale                                          |
|-----------------|----------|----------------------------------------------------|
| Structuredness  | High     | Unit tests have well-defined inputs/outputs         |
| Verifiability   | High     | Tests pass/fail; coverage is measurable             |
| Consequence     | Medium   | Missed bugs could reach production; CI is a safety net |
| AI Capability   | Emerging | Not yet measured on this codebase                   |

**Assigned Tier: 1 (Assisted) — Pilot phase**
**Human Owner: [Senior backend engineer]**

Promotion path to Tier 2:
- Run 3 sprint cycles at Tier 1
- Track: AI suggestion acceptance rate, test quality (mutation testing scores)
- Zero critical missed bugs attributable to AI-suggested test gaps
- Then: AI generates full test suites, human reviews before merge
```

**Example 2: Generating a validation checklist for AI-produced code**

User: "Create a review checklist for code our AI agent generates for the API layer."

Output:
```
## HAIF Validation Checklist: AI-Generated API Code

### Functional Correctness
- [ ] Output matches specification (endpoint contracts, status codes, response shapes)
- [ ] Edge cases handled (empty inputs, max-length strings, concurrent requests)
- [ ] Tests cover happy path AND failure paths

### Business Logic
- [ ] Correct interpretation of business rules (not just technically valid)
- [ ] Domain invariants preserved (e.g., balance cannot go negative)

### Security
- [ ] Input validation present on all user-facing parameters
- [ ] Authentication/authorization checks in place
- [ ] No injection vulnerabilities (SQL, command, SSRF)
- [ ] No secrets or credentials hardcoded
- [ ] Dependencies are real and not hallucinated packages

### Error Handling
- [ ] All failure paths produce informative error responses
- [ ] Exceptions caught at appropriate boundaries
- [ ] No silent failures or swallowed errors

### Performance
- [ ] No N+1 query patterns
- [ ] Appropriate pagination for list endpoints
- [ ] No unbounded loops or allocations

### Integration
- [ ] Follows existing codebase patterns and conventions
- [ ] All called functions/APIs exist and are correctly invoked
- [ ] Database migrations included if schema changes

### Review Metadata
- Reviewer: _______________
- First-pass acceptance: Yes / No
- Errors found: _______________
- Decision: Accept / Revise / Reject
```

**Example 3: Setting up HAIF for a small team's sprint**

User: "We're a team of 4 devs starting to use AI coding agents. Help us set up HAIF for our next sprint."

Approach:
1. Assign HWO function to tech lead (absorbed role, no new hire).
2. Agree on tier definitions with team.
3. Create lightweight registry as board metadata.
4. Adapt sprint ceremonies.

Output:
```
## HAIF Quick-Start: 4-Person Dev Team

### 1. Before Sprint (1 hour)
- Tech Lead absorbs Hybrid Work Oversight (HWO) function
- Team reviews and agrees on 4 tier definitions (post on team wiki)
- Create delegation registry as a shared spreadsheet:
  | Task Type | Tier | Owner | S | V | C | D | Error Rate | Last Reviewed |
- Select code generation checklist; adapt to your stack

### 2. Sprint Planning (+30 min added)
For each backlog item:
- Classify: Tier 1 / Tier 2 / AI-restricted (start conservative; no Tier 3-4 yet)
- Assign human owner by name
- Budget validation: Tier 2 items get +40% effort estimate for review
- Record in registry

### 3. During Sprint
- AI-produced code reviewed using checklist before merge (Tier 2)
- Record: was output accepted first-pass? Errors found? Categories?
- Any dev can demote a task's tier immediately — no approval needed

### 4. Sprint Retrospective (+15 min added)
Answer these four questions:
1. Were tier assignments accurate this sprint?
2. Was validation effort correctly estimated? (Adjust next sprint)
3. Any tier promotions earned? (Need 3+ cycles of clean data)
4. Is anyone losing the ability to do [task] without AI? Schedule human-only block.

### 5. Competence Maintenance
- Each dev does one AI-restricted task per week in their delegation area
- Rotate who reviews AI output to spread verification skill
```

## Best Practices

- **Do:** Start conservative — default to Tier 1 for any task type where AI capability is unproven on your specific codebase. Promotion requires evidence accumulated at lower tiers.
- **Do:** Budget validation time explicitly in sprint planning. Treat it as first-class planned capacity, not overhead discovered mid-sprint.
- **Do:** Make demotion frictionless. Any team member can immediately downgrade a task's tier without formal approval. This is routine adjustment, not punishment.
- **Do:** Record validation outcomes (first-pass acceptance, error type, error location) — this data drives tier transition decisions and reveals patterns.
- **Avoid:** Skipping named human ownership. Even at Tier 4, a specific person must be accountable. "The team" is not a valid owner.
- **Avoid:** Promoting tiers based on AI marketing claims or general benchmarks. The "D" (Demonstrated Capability) input requires observed performance on *this specific task type in this codebase*.
- **Avoid:** Treating validation as reading — the paper warns of the "plausibility trap" where polished AI output receives less scrutiny. Verify claims against sources; check for omissions (missing edge cases, risks, error paths).

## Error Handling

- **AI produces output at wrong tier:** If AI-generated code is merged without the required tier-level review, flag the process violation in the retrospective. Do not assign blame — treat it as a workflow gap. Add a CI check or PR template enforcement to prevent recurrence.
- **Tier transition data is missing:** If the team hasn't been tracking validation outcomes, they cannot justify tier promotion. Default to current tier; begin tracking immediately. Start Tier 3 sampling at 20% when no baseline data exists.
- **Validation capacity overwhelmed:** If the team cannot sustain the validation effort for current tier assignments, immediately demote affected tasks to a lower tier. Validation capacity is a hard constraint — never skip validation to meet a deadline.
- **Continuous co-production doesn't fit discrete delegation:** For sustained human-AI pair-programming sessions, apply three interim practices: (1) re-grounding checkpoints every 25-30 minutes where the human articulates current decisions without AI, (2) provenance logging when AI suggests direction changes, and (3) adversarial self-checks identifying three counterarguments independently.
- **Multi-agent orchestration ambiguity:** HAIF requires accountability to resolve to a natural person. If multiple AI agents are involved, the human owner is accountable for the aggregate output, not individual agent behavior.

## Limitations

- HAIF's discrete delegation model does not fully address **continuous co-production** (e.g., extended pair-programming sessions where human and AI contributions blur). The interim practices help but are acknowledged as incomplete.
- The framework is **empirically unvalidated** — threshold parameters (cycle counts, sampling rates, error rate thresholds) are reasoned estimates from design science research, not measured in controlled studies.
- HAIF assumes a functioning team context with **psychological safety**, working retrospectives, and basic quality discipline. It cannot compensate for dysfunctional team dynamics.
- The framework is **tool-agnostic by design**, which means it does not prescribe specific CI/CD integrations, board configurations, or automation. Teams must implement the protocols in their own tooling.
- For organizations with **no existing Agile or Kanban practice**, HAIF adds a layer that may be premature — establishing basic iterative workflow discipline is a prerequisite.

## Reference

Bara, M. (2026). *HAIF: A Human-AI Integration Framework for Hybrid Team Operations.* arXiv:2602.07641v1. [https://arxiv.org/abs/2602.07641v1](https://arxiv.org/abs/2602.07641v1) — Read for the complete delegation decision matrix (Table 2), autonomy tier specifications (Table 3), domain-specific validation checklists (Section 7), and the scaling model for team sizes (Table 4).