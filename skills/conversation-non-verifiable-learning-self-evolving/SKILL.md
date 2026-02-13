---
name: "conversation-non-verifiable-learning-self-evolving"
description: |
  Implements the CoNL (Conversation for Non-verifiable Learning) multi-agent
  self-play framework for iteratively improving outputs on tasks without clear
  right answers -- creative writing, code review, API design, UX copy, ethical
  reasoning, and architectural decisions. Uses structured propose-critique-revise
  conversations with diagnostic reward scoring to surface the highest-quality
  solutions.

  Trigger phrases:
  - "Improve this through self-critique" or "iteratively refine this"
  - "Use multi-agent critique to improve my code/writing/design"
  - "Run a propose-critique-revise loop on this"
  - "Help me evaluate and improve this where there's no single right answer"
  - "Self-play review this design decision"
  - "Meta-evaluate these critiques"
---

# CoNL: Multi-Agent Self-Play for Non-Verifiable Tasks

This skill enables Claude to apply the CoNL (Conversation for Non-verifiable Learning) framework from Sui & Hooi (2026) to iteratively improve outputs on **non-verifiable tasks** -- problems where there is no single correct answer and quality is inherently subjective. Instead of producing one answer and hoping it's good, Claude simulates multiple agent perspectives that propose solutions, critique them, and revise based on critiques. The key innovation: a critique is scored as valuable only if it demonstrably leads to an improved revision. This diagnostic reward signal eliminates empty criticism and rewards only actionable feedback that makes outputs better.

## When to Use This Skill

- When a user asks you to **improve creative writing** (prose style, dialogue, storytelling) and there is no ground-truth "correct" version
- When reviewing **API design or interface decisions** where multiple valid approaches exist and the user wants structured deliberation
- When asked to **refine UX copy, error messages, or documentation** where tone, clarity, and helpfulness are subjective
- When the user wants to **evaluate architectural trade-offs** (e.g., microservices vs monolith, REST vs GraphQL) with structured pros/cons and iterative refinement
- When improving **code review comments or PR descriptions** -- outputs where quality depends on helpfulness rather than correctness
- When a user says "make this better" for any artifact where better is not objectively measurable (naming conventions, commit messages, README structure)
- When asked to **resolve subjective disagreements** about code style, project structure, or design patterns by generating and evaluating multiple perspectives

## Key Technique: Diagnostic Reward via Structured Conversation

Standard LLM-as-Judge approaches hit a ceiling: the evaluator can only be as good as itself. If the judge has biases (favoring verbose responses, preferring certain styles), those biases propagate into the training signal. CoNL breaks this ceiling through **meta-evaluation** -- evaluating the evaluator -- using a simple but powerful insight: **a critique is good if and only if it helps someone produce a better solution**.

The framework operates through three roles sharing a single policy (all played by Claude):

1. **Proposer**: Generates an initial solution to the task.
2. **Critic**: Examines the solution and produces a structured critique identifying specific weaknesses and suggesting concrete improvements.
3. **Reviser**: Takes the original solution plus the critique and produces a revised version.

The **diagnostic reward** is computed by comparing the revised solution against the original. If the revision is better (as judged by pairwise comparison), the critique that led to it earns a positive reward. If the revision is worse or unchanged, the critique earns zero or negative reward. This creates a self-correcting loop: only critiques that produce measurable improvement survive, filtering out vague complaints, stylistic nitpicks, and hallucinated problems.

The process runs for multiple rounds. Each round generates preference pairs (better revision vs. original, or good critique vs. bad critique), which can be used to train the underlying model via Direct Preference Optimization (DPO). In Claude's inference-time application, we simulate this by running multiple propose-critique-revise cycles and selecting outputs where critiques demonstrably improved the result.

## Step-by-Step Workflow

1. **Classify the task as non-verifiable.** Confirm the task lacks an objectively correct answer. If the task IS verifiable (math, factual lookup, code that must pass tests), use standard approaches instead. Non-verifiable indicators: subjective quality, multiple valid solutions, stylistic preferences, trade-off decisions.

2. **Generate the initial proposal.** Produce a complete first-draft solution to the user's request. Make it genuine and complete -- not a strawman. Label this as `Proposal v0`.

3. **Produce a structured critique from a distinct perspective.** Switch to the Critic role. Evaluate the proposal across 3-5 specific dimensions relevant to the task (e.g., for writing: clarity, engagement, structure, accuracy, tone; for API design: consistency, discoverability, error handling, backwards compatibility). Each critique point must include: (a) what specifically is weak, (b) why it matters, (c) a concrete suggestion for improvement.

4. **Revise the proposal using the critique.** Switch to the Reviser role. Produce `Proposal v1` that addresses each critique point. For each change, note which critique point it addresses. Do NOT change aspects that weren't critiqued unless the critique reveals a deeper structural issue.

5. **Compute the diagnostic reward.** Compare v0 and v1 side by side. For each critique point, assess: did addressing it actually improve the output? Score each critique point as `+1` (improved), `0` (no change), or `-1` (made worse). Drop any critique point that scored 0 or -1 from future consideration.

6. **Run a second critique-revise cycle (meta-evaluation round).** Critique `Proposal v1`, but this time informed by which critique strategies worked in step 5. Focus on dimensions where the first round's critiques were effective. Produce `Proposal v2`.

7. **Perform pairwise selection.** Compare all versions (v0, v1, v2) and select the best one. Justify the selection by citing specific improvements that traced back to specific critique points. If v0 is still the best, acknowledge that the critiques were not helpful and return the original.

8. **Synthesize actionable meta-insights.** Report back to the user: which critique dimensions led to real improvements, which were dead ends, and what general principles emerged. This is the meta-evaluation output -- knowledge about what kinds of feedback actually help for this type of task.

9. **Present the final output with a change log.** Deliver the selected version alongside a concise summary of what changed from the original and why, so the user can verify the improvements align with their intent.

## Concrete Examples

**Example 1: Improving an API error message**

```
User: "Improve this error message for our REST API: 'Error 422: Invalid input'"

Step 1 - Classify: Non-verifiable. "Good" error messages are subjective
(developer experience, clarity, actionability).

Step 2 - Proposal v0:
  "Validation failed: The request body contains invalid fields.
   Please check the API documentation for required formats."

Step 3 - Critique (dimensions: specificity, actionability, tone, consistency):
  - Specificity (-): Does not say WHICH fields are invalid or WHY.
    Suggestion: Include the field name and validation rule that failed.
  - Actionability (-): "Check the docs" is vague. Link to the specific
    endpoint docs or show the expected format inline.
  - Tone (+): Professional and neutral. No change needed.
  - Consistency (neutral): Cannot evaluate without seeing other API errors.

Step 4 - Proposal v1:
  {
    "error": "validation_failed",
    "message": "2 fields failed validation",
    "details": [
      { "field": "email", "rule": "Must be a valid email address",
        "received": "not-an-email" },
      { "field": "age", "rule": "Must be an integer between 0 and 150",
        "received": "-5" }
    ],
    "docs": "https://api.example.com/docs/users#create"
  }

Step 5 - Diagnostic reward:
  - Specificity critique → +1 (clearly improved: fields and rules shown)
  - Actionability critique → +1 (docs link and inline format shown)
  - Tone critique → 0 (no change, was already fine)

Step 6 - Second critique on v1:
  - Security: Echoing back the received value could leak PII in logs.
    Suggestion: Truncate or mask received values.
  - Developer UX: Add a request_id for correlation with server logs.

Step 7 - Proposal v2 (selected):
  {
    "error": "validation_failed",
    "message": "2 fields failed validation",
    "request_id": "req_abc123",
    "details": [
      { "field": "email", "rule": "Must be a valid email address" },
      { "field": "age", "rule": "Must be an integer between 0 and 150" }
    ],
    "docs": "https://api.example.com/docs/users#create"
  }

Meta-insight: For error messages, specificity and actionability critiques
consistently improve output. Security review in round 2 caught a real issue
(PII leakage) that round 1 missed because it focused on user-facing quality.
```

**Example 2: Refining a README introduction**

```
User: "Make this README intro better:
'This is a tool for parsing logs. It supports JSON and CSV.'"

Step 2 - Proposal v0:
  "LogParse is a fast, lightweight log parser that reads JSON and CSV
   log files and outputs structured, queryable data. Built for DevOps
   teams who need to search production logs without spinning up a full
   ELK stack."

Step 3 - Critique (dimensions: clarity, audience fit, completeness, hook):
  - Hook (-): Opens with product name, not the problem it solves.
    Suggestion: Lead with the pain point.
  - Completeness (-): Doesn't mention install, key differentiator, or
    a quick example. Suggestion: Add a one-liner install + usage.
  - Audience fit (+): "DevOps teams" and "ELK stack" reference are good
    audience signals.
  - Clarity (+): Concise and scannable. No change.

Step 4 - Proposal v1:
  "Searching production logs shouldn't require a 3-node Elasticsearch
   cluster. LogParse reads JSON and CSV log files from disk and lets you
   query them with SQL-like syntax -- no infrastructure needed.

   pip install logparse
   logparse query 'SELECT * FROM app.log WHERE level = "ERROR"' --last 1h"

Step 5 - Diagnostic reward:
  - Hook critique → +1 (pain-point lead is stronger)
  - Completeness critique → +1 (install + example added)

Step 7 - Selected: v1 (clear improvement on both scored dimensions).

Meta-insight: For README intros, leading with the problem (not the product
name) and including a runnable example are the two highest-impact changes.
```

**Example 3: Evaluating an architectural decision**

```
User: "Should we use WebSockets or SSE for our real-time dashboard notifications?"

Step 2 - Proposal v0:
  "Use Server-Sent Events (SSE). Your dashboard only needs server-to-client
   push for notifications. SSE is simpler, works over HTTP/2, auto-reconnects,
   and doesn't require a separate WebSocket server."

Step 3 - Critique (dimensions: trade-off coverage, assumptions, specificity):
  - Assumptions (-): Assumes notifications are unidirectional. If users need
    to acknowledge/dismiss notifications from the dashboard, that requires
    client-to-server messaging. Suggestion: Ask whether user interaction
    flows back to the server in real-time.
  - Trade-off coverage (-): Doesn't mention browser support for SSE with
    HTTP/2, connection limits in HTTP/1.1 (6 per domain), or load balancer
    configuration differences. Suggestion: Add a comparison table.
  - Specificity (+): Correctly identifies the key advantage (simplicity).

Step 4 - Proposal v1:
  "Recommendation: SSE for notifications, with an escape hatch to WebSockets.

   | Factor             | SSE                        | WebSocket              |
   |--------------------|----------------------------|------------------------|
   | Direction          | Server → Client only       | Bidirectional          |
   | Protocol           | HTTP (works with proxies)  | Upgrade to ws://       |
   | Reconnection       | Built-in auto-reconnect    | Manual implementation  |
   | Connection limits  | 6/domain (HTTP/1.1)        | No HTTP limit          |
   | Load balancer      | Standard HTTP routing      | Sticky sessions needed |

   If the dashboard is read-only (notifications appear, user clicks to
   navigate), SSE is the right choice. If users need to send real-time
   actions back (e.g., acknowledge, snooze, reply), use WebSockets.

   Suggested clarification question for the user: Do notification
   interactions need to be real-time, or can they use standard REST calls?"

Step 5 - Diagnostic reward:
  - Assumptions critique → +1 (surfaced a real decision point)
  - Trade-off coverage critique → +1 (table makes trade-offs concrete)

Step 7 - Selected: v1 (addresses both valid critique points, gives the
user a decision framework rather than a flat recommendation).
```

## Best Practices

- **Do:** Make each critique point specific and actionable -- name the exact weakness, explain why it matters, and suggest a concrete fix. Vague critiques ("could be better") score zero on diagnostic reward and waste a revision cycle.
- **Do:** Track which critique dimensions consistently lead to improvements across rounds. Reuse effective dimensions; drop ineffective ones. This is the meta-evaluation payoff.
- **Do:** Keep the original proposal genuine. A weak strawman proposal makes the critique seem effective when it's actually just correcting artificial flaws.
- **Do:** Present the user with the version comparison so they can override the selection. The diagnostic reward is a heuristic, not ground truth.
- **Avoid:** Running more than 2-3 critique-revise cycles. Diminishing returns set in quickly, and later rounds tend to introduce stylistic churn rather than substantive improvement.
- **Avoid:** Critiquing everything at once. Focus each round on 3-5 dimensions. Attempting to improve all aspects simultaneously dilutes the signal and makes it hard to attribute which critiques caused which improvements.

## Error Handling

- **Critique finds no real weaknesses:** This happens when the initial proposal is already strong. Return v0 as the final output and explicitly tell the user: "The structured critique process did not surface improvements. The original output is the recommended version." Do not fabricate weaknesses to justify the process.
- **Revision is worse than original:** Score the responsible critique as -1, revert to the previous version, and note which critique direction was counterproductive. This is valuable meta-evaluation signal.
- **User disagrees with the selected version:** The user's preference overrides the diagnostic reward. Ask which version they prefer and why -- their reasoning becomes a new critique dimension for future rounds.
- **Task turns out to be verifiable:** If during critique you discover the task has an objective answer (e.g., the "style" question is actually about correctness), switch to direct problem-solving. CoNL adds overhead without value when ground truth exists.

## Limitations

- **Not for verifiable tasks.** If the output can be checked against tests, specifications, or factual sources, standard verification is faster and more reliable than multi-round critique.
- **Inference-time only.** This skill simulates CoNL's self-play at inference time. It does not perform actual DPO training or update model weights. The improvements are per-session, not persistent.
- **Single-model ceiling.** All agent roles (proposer, critic, reviser) are played by the same model. Blind spots shared across all roles cannot be caught. For high-stakes decisions, pair this with human review.
- **Diminishing returns.** Most improvement happens in round 1. Round 2 catches secondary issues. Round 3+ rarely adds value and risks introducing preference cycling (A→B→A).
- **Cost multiplier.** Each propose-critique-revise cycle roughly triples the token usage compared to a single generation. Use this for outputs where quality matters enough to justify the cost.

## Reference

Sui, Y. & Hooi, B. (2026). *Conversation for Non-verifiable Learning: Self-Evolving LLMs through Meta-Evaluation.* arXiv:2601.21464. Key insight to look for: the diagnostic reward mechanism that scores critiques based on whether they produce measurable improvement in revisions, enabling joint optimization of generation and evaluation without external judges.