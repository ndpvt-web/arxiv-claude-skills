---
name: "lhaw-controllable-underspecification-long-horizon"
description: "Detect and handle ambiguity in long-horizon agent tasks using the LHAW framework. Systematically identify underspecified Goals, Constraints, Inputs, and Context in task prompts, classify ambiguity severity, and decide when to clarify vs. proceed. Use when: 'analyze this task for ambiguity', 'what's missing from this spec', 'generate underspecified variants', 'test agent robustness to vague instructions', 'find what could go wrong with this prompt', 'stress-test this workflow for unclear requirements'."
---

# LHAW: Controllable Underspecification for Long-Horizon Tasks

This skill enables Claude to systematically detect, classify, and handle ambiguity in multi-step task specifications using the LHAW (Long-Horizon Augmented Workflows) framework. Rather than guessing what an ambiguous instruction means or blindly proceeding, Claude applies a four-dimensional analysis (Goals, Constraints, Inputs, Context) to identify exactly which information is missing, assess whether that missing information will cause task failure or merely produce variant outcomes, and make cost-sensitive decisions about when to ask for clarification versus proceeding with reasonable defaults.

## When to Use

- When a user provides a multi-step task and you suspect key details are missing or ambiguous
- When building or reviewing agent prompts and you need to stress-test them for robustness
- When a user asks you to generate harder or more ambiguous variants of a task for benchmarking
- When triaging user requests to decide which clarifying questions are worth asking (cost-sensitive clarification)
- When auditing a workflow specification for completeness before handing it to an automated pipeline
- When a user says "what could go wrong if I give this prompt to an agent?" or "what assumptions am I making here?"
- When designing evaluation datasets that test agent behavior under ambiguity

## Key Technique

LHAW treats ambiguity not as a binary property but as a structured, measurable phenomenon across four orthogonal dimensions. **Goals** cover what the task's deliverable or success criteria are — vague action verbs ("process the data") or missing completion metrics create Goal ambiguity. **Constraints** cover execution rules like budgets, precision thresholds, format requirements, or exclusion criteria. **Inputs** cover data sources, file locations, tool selections, or system references — phrases like "the latest dataset" or "the file" are Input ambiguity. **Context** covers domain knowledge, business logic, jargon, or implicit assumptions the agent needs but the prompt doesn't state.

The framework scores each identifiable information segment along two axes: **criticality** (1.0 = essential, 0.5 = important, 0.0 = cosmetic) and **guessability** (0.0 = impossible to infer, 0.5 = sometimes inferable, 1.0 = trivially guessable). The priority of a missing segment is `criticality * (1 - guessability)`. This lets you rank which missing pieces matter most and focus clarification effort there. Removal is performed via three strategies — **Delete** (remove entirely), **Vaguify** (replace with ambiguous references like "the output"), or **Genericize** (substitute with generic phrases like "an appropriate format").

Crucially, LHAW does not trust linguistic intuition about what is ambiguous. It classifies variants empirically through observed outcomes: **Outcome-critical** (agent always fails without the information), **Divergent** (agent sometimes succeeds but reaches inconsistent terminal states), or **Benign** (agent reliably infers the missing information). This three-way taxonomy drives practical decisions: outcome-critical gaps demand clarification, divergent gaps warrant clarification if cost allows, and benign gaps can be safely ignored.

## Step-by-Step Workflow

1. **Extract atomic information segments from the task prompt.** Parse the user's task description into discrete, removable units of information. Each segment should map to exactly one fact: a file path, a constraint threshold, a success criterion, a tool name, a domain term, etc.

2. **Classify each segment into one of the four dimensions (Goal, Constraint, Input, Context).** Tag each segment: Is it defining *what* to produce (Goal), *how* to produce it (Constraint), *where/what* to use as source material (Input), or *why/under what assumptions* (Context)?

3. **Score each segment for criticality and guessability.** Assign criticality (1.0/0.5/0.0) based on whether task success depends on this information. Assign guessability (1.0/0.5/0.0) based on whether an agent could reasonably infer it from the remaining prompt, common conventions, or environment. Compute priority = `criticality * (1 - guessability)`.

4. **Rank segments by priority and identify the top-N gaps.** Focus on segments with priority >= 0.5. These are the information pieces whose absence will most likely cause failure or divergent behavior.

5. **For each high-priority gap, determine the likely impact class.** Predict whether removing this information would be outcome-critical (task impossible without it), divergent (task completable but results vary unpredictably), or benign (agent can safely infer a correct default). Use concrete reasoning, not gut feel — consider whether the task environment provides the missing information implicitly.

6. **Generate clarifying questions for outcome-critical and divergent gaps only.** Formulate specific, non-compound questions. Each question should target exactly one missing segment. Avoid asking about benign gaps — this wastes the user's time and signals low competence.

7. **Order clarifying questions by expected information gain.** Ask about outcome-critical gaps first, then divergent gaps. If the user's tolerance for questions is low (e.g., they said "just do it"), proceed with explicit assumptions for divergent gaps and only ask about outcome-critical ones.

8. **Document assumptions explicitly for any gap you choose not to clarify.** When proceeding without clarification on a divergent or benign gap, state the assumption clearly: "I'm assuming X because Y. If this is wrong, the output will differ in Z way."

9. **If generating underspecified variants for testing, apply removal strategies systematically.** Use Delete for maximum difficulty, Vaguify for moderate difficulty, and Genericize for realistic-but-subtle ambiguity. Control the number of simultaneous removals (1 for targeted testing, 2+ for stress testing).

10. **Validate classification through execution when possible.** If you can run the task multiple times with different interpretations of the ambiguous prompt, compare terminal states to empirically classify the variant as outcome-critical, divergent, or benign.

## Concrete Examples

**Example 1: Analyzing a user's task for underspecification**

User: "Set up a CI pipeline for this repo that runs tests and deploys to staging."

Approach:
1. Extract segments: [CI platform], [test command], [staging environment target], [deploy trigger], [branch policy], [failure behavior], [notification config]
2. Classify and score:

```
| Segment              | Dimension  | Criticality | Guessability | Priority |
|----------------------|------------|-------------|--------------|----------|
| CI platform          | Input      | 1.0         | 0.5          | 0.5      |
| Test command         | Input      | 1.0         | 0.5          | 0.5      |
| Staging target       | Input      | 1.0         | 0.0          | 1.0      |
| Deploy trigger       | Constraint | 0.5         | 0.5          | 0.25     |
| Branch policy        | Constraint | 0.5         | 0.5          | 0.25     |
| Failure behavior     | Goal       | 0.5         | 1.0          | 0.0      |
| Notification config  | Context    | 0.0         | 0.5          | 0.0      |
```

3. Outcome classification:
   - **Staging target** (priority 1.0): Outcome-critical — cannot deploy without knowing where
   - **CI platform** (priority 0.5): Divergent — could pick GitHub Actions or CircleCI, results differ
   - **Test command** (priority 0.5): Divergent — might guess from package.json but could be wrong

Output:
```
I need to clarify a few things before proceeding:

1. **Where is your staging environment?** (e.g., AWS, Heroku, a specific
   server URL, Kubernetes namespace) — I cannot deploy without this.
2. **Which CI platform do you use?** I see a .github/ directory, so I'll
   default to GitHub Actions unless you prefer something else.

Proceeding with these assumptions:
- Test command: `npm test` (based on your package.json scripts)
- Deploy trigger: on push to main branch
- Failure behavior: block deploy if tests fail
```

**Example 2: Generating underspecified variants for agent testing**

User: "Take this task spec and generate harder versions for benchmarking: 'Download the Q3 2025 sales report from the internal dashboard at dashboard.corp.com/reports, convert it to CSV format, filter rows where revenue exceeds $10,000, and email the result to finance@corp.com with subject line Q3 Summary.'"

Approach:
1. Extract 7 segments: [data source URL], [report identifier: Q3 2025], [output format: CSV], [filter threshold: $10,000], [filter column: revenue], [recipient: finance@corp.com], [subject line: Q3 Summary]
2. Apply removal strategies at increasing difficulty:

Output:
```
**Variant A (Single removal, Vaguify, Input dimension):**
"Download the latest sales report from the internal dashboard, convert
it to CSV format, filter rows where revenue exceeds $10,000, and email
the result to finance@corp.com with subject line Q3 Summary."
→ Removed: specific URL and report period
→ Expected class: Divergent (agent might pick wrong quarter)

**Variant B (Single removal, Delete, Constraint dimension):**
"Download the Q3 2025 sales report from dashboard.corp.com/reports,
convert it to CSV, filter for high-revenue rows, and email the result
to finance@corp.com with subject line Q3 Summary."
→ Removed: revenue threshold ($10,000)
→ Expected class: Outcome-critical (no way to guess the cutoff)

**Variant C (Double removal, Genericize, Goal + Input):**
"Get the sales data from the company dashboard, process it into a
suitable format, filter rows where revenue exceeds $10,000, and send
the result to the finance team."
→ Removed: specific URL, output format, recipient email, subject line
→ Expected class: Outcome-critical (too many unknowns)
```

**Example 3: Cost-sensitive clarification during task execution**

User: "Refactor the authentication module to use the new pattern."

Approach:
1. Extract segments: [which auth module], [which new pattern], [scope of refactor], [backward compatibility requirements]
2. Score: "new pattern" is Goal dimension, criticality 1.0, guessability 0.0 → priority 1.0. "Which auth module" is Input dimension, criticality 1.0, guessability 0.5 (may be only one).

Output:
```
Before I start, one critical question:

1. **What is "the new pattern"?** (e.g., a specific PR, a design doc,
   OAuth2 PKCE, session tokens → JWTs?) — I can't proceed without this.

I'll check the codebase for auth modules and assume I should refactor
all of them unless you narrow the scope. I'll maintain backward
compatibility with existing API contracts unless told otherwise.
```

## Best Practices

- **Do:** Ask compound-free questions — one question per missing segment. Compound questions ("What's the target and what format and should I notify anyone?") reduce answer quality and signal poor analysis.
- **Do:** Rank your clarifications by priority and lead with outcome-critical gaps. If you can only ask one question, it should be about the highest-priority missing segment.
- **Do:** State your assumptions explicitly when proceeding without clarification. "I'm assuming X" is always better than silently choosing X.
- **Do:** Re-evaluate guessability as you learn more about the environment. A missing file path has guessability 0.0 in isolation, but if `ls` shows exactly one candidate file, guessability rises to 1.0.
- **Avoid:** Over-clarifying benign gaps. Asking "Should I use UTF-8 encoding?" when every file in the repo is UTF-8 wastes trust and patience. The LHAW finding is that over-clarification (asking about guessable information) is as costly as under-clarification.
- **Avoid:** Treating all ambiguity as equal. A missing deploy target (outcome-critical) and a missing log verbosity level (benign) deserve completely different responses.

## Error Handling

- **Segment extraction misses a critical piece:** If you proceed and the task fails, backtrack by re-analyzing the original prompt with the failure mode as a hint. The failure itself reveals which segment was outcome-critical.
- **Guessability scored too high:** If you assumed something was inferable but the environment doesn't support that inference, escalate to clarification immediately rather than guessing again. One wrong guess is recoverable; two compounds the error.
- **User refuses to clarify:** Document the ambiguity, proceed with the most conservative interpretation (the one least likely to cause irreversible damage), and flag the output as provisional.
- **Too many high-priority gaps (>3):** The task spec is fundamentally incomplete. Rather than asking 5+ questions, summarize the gaps and ask the user to provide a more complete specification. This is more efficient than iterative Q&A.

## Limitations

- The four-dimension taxonomy (Goal, Constraint, Input, Context) covers most software engineering ambiguity but can miss cross-dimensional interactions where two individually benign gaps combine into an outcome-critical situation.
- Empirical validation (running the task multiple times to classify variants) is only feasible for tasks with fast, deterministic execution. Long-running or expensive tasks must rely on analytical classification.
- Guessability scoring is itself subjective — what one agent can infer from context, another cannot. Calibrate scores to the specific agent's capabilities and environment access.
- The framework assumes the original well-specified task is actually well-specified. If the "complete" spec is itself ambiguous, the analysis will have a flawed baseline.
- This technique focuses on information *absence*. It does not address information *conflict* (contradictory requirements) or information *overload* (too many constraints to satisfy simultaneously).

## Reference

[LHAW: Controllable Underspecification for Long-Horizon Tasks](https://arxiv.org/abs/2602.10525v1) — See Sections 3-4 for the segment extraction pipeline, criticality/guessability scoring, and the three removal strategies (Delete, Vaguify, Genericize). Section 5 covers the empirical classification taxonomy and agent trial methodology. Section 6 contains findings on clarification efficiency across frontier models.