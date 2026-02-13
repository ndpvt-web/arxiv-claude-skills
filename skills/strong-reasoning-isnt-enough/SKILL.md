---
name: "strong-reasoning-isnt-enough"
description: "Build interactive diagnostic agents that systematically elicit evidence before concluding, using the REFINE (Reasoning-Enhanced Feedback for INformation Elicitation) loop from EID-Benchmark research. Prevents premature diagnosis by measuring Information Coverage Rate (ICR) and closing evidence gaps through verification feedback. Trigger phrases: 'build a diagnostic chatbot', 'interactive troubleshooting agent', 'evidence gathering workflow', 'systematic question-asking agent', 'build an intake triage system', 'REFINE diagnostic loop'"
---

# Interactive Evidence Elicitation with the REFINE Loop

This skill teaches Claude to build **interactive diagnostic agents** that systematically gather evidence before reaching conclusions, based on the REFINE (Reasoning-Enhanced Feedback for INformation Elicitation) framework from the paper "Strong Reasoning Isn't Enough" (arXiv:2601.19773). The core insight: strong reasoning ability alone causes agents to jump to correct-looking conclusions from minimal cues, skipping the thorough evidence collection that real-world diagnosis demands. The REFINE loop fixes this by introducing a closed-loop verification cycle where a Diagnosis Verifier identifies evidence gaps and feeds them back to the Information Collector, achieving significantly higher Information Coverage Rates without sacrificing accuracy.

## When to Use

- When building a **medical intake or triage chatbot** that must ask systematic questions before suggesting possible conditions
- When creating an **IT troubleshooting agent** that diagnoses system failures through interactive questioning rather than guessing from the first symptom
- When designing a **customer support bot** that gathers complete problem context before proposing solutions
- When implementing a **debugging assistant** that elicits reproduction steps, environment details, and error context before suggesting fixes
- When building any **interactive agent that must gather information under uncertainty** before making a decision (hiring screening, insurance claims, root cause analysis)
- When evaluating an existing conversational agent's **evidence-gathering thoroughness** using ICR-style metrics

## Key Technique

### The Problem: Reasoning Shortcuts

High-capability LLMs achieve strong diagnostic accuracy on static benchmarks but perform poorly in interactive settings. The reason is counterintuitive: strong reasoning lets agents pattern-match to a conclusion from minimal evidence, so they stop asking questions prematurely. In experiments, models like GPT-5 achieved correct diagnoses with ICR scores below 50%--meaning they skipped gathering more than half the relevant evidence. This is dangerous in production: a correct diagnosis reached through incomplete evidence is fragile and unexplainable.

### The REFINE Architecture

REFINE decomposes the diagnostic agent into four specialized modules that form a closed loop:

1. **Information Collector** -- Conducts multi-turn interaction, decides at each turn whether to ask another question, request an examination, or stop collecting. It receives explicit gap-feedback from the Verifier telling it what uncertainties remain.
2. **Evidence Organizer** -- Structures all collected evidence into a canonical summary, grouping facts into patient-reported evidence (demographics, symptoms, history) and examination evidence (lab results, test findings).
3. **Diagnosis Reasoner** -- Generates diagnostic hypotheses from the organized evidence.
4. **Diagnosis Verifier** -- Evaluates whether collected evidence sufficiently supports the hypothesis. If gaps exist, it produces explicit feedback identifying unresolved uncertainties, which routes back to the Information Collector. The loop terminates when verification passes or a turn limit is reached.

### Atomic Evidence and ICR

Evidence is represented as **atomic facts**: minimal, self-contained units like "Patient reports sharp chest pain radiating to left arm" or "WBC count: 12,400/uL." The Information Coverage Rate measures completeness: `ICR = |collected_evidence ∩ required_evidence| / |required_evidence|`. This metric is orthogonal to diagnostic accuracy and reveals whether your agent is thorough or just lucky.

## Step-by-Step Workflow

### 1. Define Your Domain's Atomic Evidence Schema

Decompose each case in your domain into atomic evidence units. Categorize them (e.g., `patient_reported` vs `examination_results` in medicine, or `user_reported` vs `system_telemetry` in IT). Each unit gets a unique index for tracking.

```python
evidence_schema = {
    "patient_reported": {
        "demographics": ["age", "sex", "occupation"],
        "chief_complaint": ["primary_symptom", "onset", "duration", "severity"],
        "history": ["past_conditions", "medications", "allergies", "family_history"],
        "review_of_systems": ["cardiovascular", "respiratory", "neurological"]
    },
    "examination": {
        "vitals": ["bp", "hr", "temp", "spo2", "rr"],
        "lab_results": ["cbc", "bmp", "urinalysis"],
        "imaging": ["xray", "ct", "mri"]
    }
}
```

### 2. Build the Simulated Information Source

Create a grounded responder that holds the complete evidence set for each case and reveals at most 2 evidence items per response. If asked about something not in its knowledge base, it responds with uncertainty rather than fabricating. Use temperature 0 and stateless (single-turn context) responses to prevent drift.

### 3. Implement the Information Collector Module

Build the agent that conducts multi-turn interaction. At each turn it must:
- Assess whether current evidence is sufficient to diagnose (the sufficiency check)
- If insufficient, formulate a targeted question aimed at a specific evidence category
- If sufficient (or max turns reached), signal termination and pass evidence to the Organizer

### 4. Implement the Evidence Organizer

After collection terminates, structure all gathered evidence into a canonical format. Group facts by category, deduplicate, and flag any contradictions. Output a structured summary (JSON or typed object).

```python
organized_evidence = {
    "patient_reported": [
        {"id": "P1", "category": "chief_complaint", "fact": "Intermittent 503 errors on /api/checkout"},
        {"id": "P2", "category": "chief_complaint", "fact": "Started 2 hours ago after deploy v3.2.1"},
        {"id": "P3", "category": "history", "fact": "Similar issue 3 months ago traced to connection pool"}
    ],
    "examination": [
        {"id": "E1", "category": "logs", "fact": "Connection pool exhaustion in db-primary at 14:32 UTC"},
        {"id": "E2", "category": "metrics", "fact": "p99 latency spike from 200ms to 4500ms"}
    ]
}
```

### 5. Implement the Diagnosis Reasoner

Generate one or more diagnostic hypotheses from the organized evidence. Each hypothesis should cite specific evidence IDs that support it.

### 6. Implement the Diagnosis Verifier (Critical Step)

This is where REFINE diverges from naive agents. The Verifier:
- Takes the hypothesis and the organized evidence
- Checks whether the evidence sufficiently supports the diagnosis
- Identifies specific gaps: "No information about X, which is needed to distinguish between hypothesis A and B"
- Returns either `VERIFIED` or a structured list of `unresolved_uncertainties`

### 7. Close the Loop

Route unresolved uncertainties back to the Information Collector as explicit prompts. The Collector resumes interaction, now targeting the specific gaps identified by the Verifier. Repeat steps 3-7 until verified or turn budget exhausted.

### 8. Compute ICR for Evaluation

Track which atomic evidence items were elicited during the interaction. Compute ICR against the ground-truth evidence set. Use this metric alongside diagnostic accuracy to evaluate and iterate on your agent.

```python
def compute_icr(collected: set, required: set) -> float:
    if not required:
        return 1.0
    return len(collected & required) / len(required)
```

### 9. Tune the Role Assignment

The paper found that pairing a weaker model as Information Collector with a stronger model as Diagnosis Verifier outperforms the reverse. Apply this insight: use your most capable model for verification/reasoning and a lighter model for the interactive collection phase.

### 10. Set Stopping Criteria

Define a maximum turn count (the paper uses domain-specific limits). The Verifier's "no gaps found" signal is the primary termination condition; the turn limit is the safety fallback.

## Concrete Examples

**Example 1: Medical Triage Chatbot**

User: Build me an intake chatbot that gathers symptoms before suggesting what kind of specialist to see.

Approach:
1. Define atomic evidence schema: chief complaint (symptom, location, onset, duration, severity, triggers, relieving factors), medical history (conditions, surgeries, medications, allergies), review of systems (10 standard categories), demographics.
2. Implement Information Collector as a multi-turn chat that asks one focused question per turn, tracks which evidence categories remain uncovered.
3. After the Collector signals sufficiency, the Evidence Organizer produces a structured intake summary.
4. The Diagnosis Reasoner suggests 1-3 specialist referral options with supporting evidence.
5. The Diagnosis Verifier checks: "Do we know enough to distinguish cardiology vs pulmonology vs GI?" If not, it feeds back: "Need to clarify whether pain is exertional or postprandial."
6. Collector re-engages the patient with the targeted question.

Output:
```
Agent: What brings you in today?
Patient: I've been having chest pain.
Agent: Can you describe where exactly the pain is and what it feels like?
Patient: It's in the center of my chest, feels like pressure.
Agent: When did it start, and how often does it happen?
Patient: Started about a week ago, happens a few times a day.
Agent: Does anything trigger the pain--physical activity, eating, stress?
Patient: Usually after meals, especially large ones.
Agent: Does anything make it better--resting, antacids, changing position?
Patient: Antacids help sometimes.
[Verifier feedback: Missing family cardiac history, no vitals context]
Agent: Do you have any family history of heart disease?
Patient: No, none.
Agent: Are you currently taking any medications?
Patient: Just over-the-counter antacids occasionally.

--- Intake Summary ---
Evidence collected: 10/12 atomic items (ICR: 83%)
Suggested referral: Gastroenterology (primary), Cardiology (rule-out)
Supporting evidence: postprandial onset, antacid relief, no cardiac family hx
```

**Example 2: IT Incident Diagnostic Agent**

User: Create a Slack bot that diagnoses production incidents by asking engineers systematic questions before suggesting root causes.

Approach:
1. Define evidence schema: symptoms (error type, affected endpoints, frequency, user impact), timeline (onset, recent changes, deploy history), environment (infrastructure, dependencies, load), telemetry (logs, metrics, traces).
2. Collector asks targeted questions in Slack thread. Simulated Reporter module queries monitoring APIs (Datadog, PagerDuty) for examination-type evidence.
3. Organizer produces structured incident summary.
4. Reasoner hypothesizes: "Connection pool exhaustion due to leaked connections in new ORM migration."
5. Verifier identifies gap: "No evidence about connection pool metrics or recent ORM changes."
6. Collector asks engineer: "Were there any database-related changes in the last deploy?" and queries connection pool dashboards.

Output:
```
Bot: Incident reported on #api-checkout. What errors are users seeing?
Eng: Intermittent 503s on the checkout endpoint.
Bot: When did this start? Any recent deployments?
Eng: About 2 hours ago. We deployed v3.2.1 at 14:00 UTC.
Bot: Pulling metrics... [queries Datadog API]
     Found: p99 latency spike to 4500ms at 14:32, connection pool at 98% capacity.
Bot: What changed in v3.2.1 related to database access?
Eng: We migrated the checkout service to the new ORM.
[Verifier: VERIFIED - sufficient evidence]

--- Incident Analysis ---
ICR: 9/11 (82%)
Root cause hypothesis: New ORM in v3.2.1 leaks connections under concurrent load
Evidence: deploy timing correlation, connection pool exhaustion, ORM migration in changeset
Recommended action: Roll back v3.2.1, add connection pool monitoring, review ORM connection lifecycle
```

**Example 3: Evaluating an Existing Agent's Thoroughness**

User: I have a diagnostic chatbot already. How do I measure whether it asks enough questions?

Approach:
1. Curate 50-100 test cases with ground-truth atomic evidence sets (the complete list of facts the agent should elicit for each case).
2. Run each case through the chatbot, logging every evidence item revealed during the conversation.
3. Compute ICR per case: `len(revealed & required) / len(required)`.
4. Plot ICR vs diagnostic accuracy. Look for the "strong reasoning" failure mode: high accuracy but low ICR (agent guesses right from minimal evidence).
5. Identify systematic blind spots: which evidence categories consistently have low coverage?

Output:
```
Evaluation Results (n=100 cases):
  Diagnostic Accuracy: 78%
  Mean ICR: 41%
  Median turns per case: 3.2

Blind spots:
  - Medication history: covered in 12% of cases
  - Family history: covered in 8% of cases
  - Review of systems: covered in 22% of cases

Recommendation: Agent reaches conclusions too quickly.
Implement REFINE verification loop to close evidence gaps
before diagnosis. Target ICR > 70%.
```

## Best Practices

- **Do:** Decompose evidence into atomic, self-contained facts at the finest useful granularity. Coarse evidence units (e.g., "medical history") make ICR meaningless.
- **Do:** Use the Verifier to produce *specific* gap descriptions ("need to know if pain is positional") rather than generic ones ("need more information"). Specific feedback drives targeted questions.
- **Do:** Enforce stateless, deterministic responses from your simulated patient/information source (temperature 0, no conversation history carry-over) to ensure reproducible evaluation.
- **Do:** Cap evidence items revealed per response at 2 to simulate realistic information flow and prevent shortcutting.
- **Avoid:** Using diagnostic accuracy as your only evaluation metric. An agent that guesses right 80% of the time but collects only 30% of evidence is unreliable in production.
- **Avoid:** Letting the Information Collector see the Verifier's internal reasoning about the diagnosis. The Collector should only receive the list of evidence gaps, not the hypothesized diagnosis, to prevent anchoring bias.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Agent asks the same question repeatedly | Collector lacks memory of already-gathered evidence | Maintain an explicit evidence ledger that the Collector checks before each question |
| ICR is high but accuracy is low | Evidence is gathered but poorly synthesized | Check the Evidence Organizer for deduplication failures or miscategorization |
| Verifier never signals "verified" | Gap criteria too strict or evidence schema too granular | Add a confidence threshold: verify if ICR > 70% and no critical-category gaps remain |
| Agent exhausts turn limit with low ICR | Questions are too broad or off-target | Analyze which evidence categories are consistently missed; add category-aware prompting to the Collector |
| Simulated patient gives inconsistent answers | Stateful context window causes drift | Enforce stateless (context window = 1) and temperature 0 for the information source |

## Limitations

- **Domain expertise required for evidence schemas:** Defining the atomic evidence set for a domain requires subject-matter expertise. Poor schemas produce meaningless ICR scores.
- **Turn budget tension:** More thorough evidence gathering requires more turns, which increases latency and cost. The paper does not solve the optimal stopping problem--you must tune turn limits per domain.
- **ICR assumes known ground truth:** You need a curated set of "required evidence" per case to compute ICR. This limits automated evaluation to cases with expert-annotated evidence.
- **Not suited for open-ended domains:** REFINE works best when the space of relevant evidence is bounded and enumerable (medical diagnosis, IT troubleshooting, structured intake). It is less applicable to creative or exploratory tasks.
- **Simulated patients differ from real ones:** Real users give messy, contradictory, emotionally-charged responses. The 2-evidence-per-turn constraint and stateless design simplify evaluation but don't capture real interaction complexity.

## Reference

**Paper:** "Strong Reasoning Isn't Enough: Evaluating Evidence Elicitation in Interactive Diagnosis" by Zhuohan Long, Zhijie Bao, Zhongyu Wei (arXiv:2601.19773, 2026).
**Key takeaway:** Look at Table 3 and Figure 3 for the disconnect between diagnostic accuracy and ICR across models, and Section 4.3 for the REFINE architecture details showing how verification-guided feedback improves evidence coverage by 10-15% absolute.