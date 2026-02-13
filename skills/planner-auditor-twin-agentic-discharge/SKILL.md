---
name: "planner-auditor-twin-agentic-discharge"
description: "Implement a Planner-Auditor twin architecture that decouples LLM generation from deterministic validation with self-improvement loops. Use when: 'build a planner-auditor pipeline', 'add deterministic auditing to my LLM agent', 'implement self-improving generation with confidence calibration', 'create a two-tier feedback loop for LLM outputs', 'build a FHIR discharge planning system', 'add discrepancy buffering and replay to my agent'"
---

# Planner-Auditor Twin: Agentic Generation with Deterministic Validation and Self-Improvement

This skill teaches Claude to build **Planner-Auditor** architectures — a pattern where an LLM generator (Planner) produces structured outputs with explicit confidence scores, and a rule-based validator (Auditor) performs deterministic coverage checks, calibration tracking (Brier score, ECE), and drift monitoring. The system self-improves through two tiers: within-episode regeneration when the Auditor flags gaps, and cross-episode discrepancy buffering that replays high-confidence failures. This architecture raised task coverage from 32% to 86% (and 100% with replay) while drastically reducing calibration error — without any model retraining.

## When to Use

- When the user wants to build an LLM agent that **generates structured plans or action items** and needs deterministic quality gates before surfacing results.
- When building a pipeline where **LLM confidence must be calibrated** — the model says "I'm 90% sure" and you need to verify that claim against actual correctness.
- When the user asks to add **self-improvement or feedback loops** to an LLM workflow without fine-tuning or retraining.
- When implementing **FHIR-based clinical pipelines** for discharge planning, care coordination, or patient summarization.
- When the user needs a **triage routing system** (green/yellow/red lanes) based on auditor pass/fail and confidence levels.
- When building any agent that must cover **mandatory task categories** and you need to detect and correct omissions systematically.
- When the user asks about **discrepancy buffering** or **targeted replay** to fix persistent high-confidence errors across batch runs.

## Key Technique

The core insight is **separation of concerns**: the LLM Planner generates structured outputs (action plans, recommendations, reports) with an explicit confidence estimate per output. The Auditor is a purely deterministic module — no LLM involved — that checks three things: (1) **coverage** — are all mandatory categories present? (2) **calibration** — does the model's stated confidence match its actual accuracy, measured via Brier score and Expected Calibration Error (ECE)? (3) **drift** — has the distribution of output types shifted beyond an L1 threshold compared to baseline?

Self-improvement happens at two tiers without retraining. **Tier 1 (within-episode)**: when the Auditor flags a coverage violation, the system regenerates the plan in the same request, feeding the prior draft plus the Auditor's specific complaints back to the LLM as context. **Tier 2 (cross-episode)**: cases where confidence was high (>= 0.8) but coverage failed are logged to a discrepancy buffer. These "hard cases" are batched and replayed later, giving the LLM a second chance with enriched context. This two-tier approach drove coverage from 32% baseline to 86% with Tier 1 alone and 100% with Tier 2 replay.

The triage routing turns Auditor results into actionable lanes: **Green** (Auditor passes — surface to user), **Yellow** (Auditor fails, low confidence — flag for manual review), **Red** (Auditor fails, high confidence — buffer for replay, the model is confidently wrong and needs correction).

## Step-by-Step Workflow

1. **Define mandatory output categories.** Enumerate the task categories the Planner must cover. For discharge planning these are: follow-up appointments, medication reconciliation, patient education, symptom monitoring. For other domains, define your own mandatory set as an explicit checklist.

2. **Build the Planner module.** Create a function that takes structured input context (e.g., patient data, FHIR resources, user request) and calls an LLM to produce a typed JSON output. The schema must include: an array of actions (each with `type`, `details`, `deadline` or priority), and a top-level `confidence` float (0.0–1.0). Use a system prompt that instructs the model to self-assess its confidence.

   ```python
   class Action(BaseModel):
       type: str          # Must match one of the mandatory categories
       details: str
       deadline_hours: int | None = None

   class ActionPlan(BaseModel):
       actions: list[Action]
       confidence: float  # 0.0 to 1.0, model's self-assessed confidence
       reasoning: str
   ```

3. **Build the Auditor module — coverage check.** Write a deterministic function that takes the ActionPlan and checks whether every mandatory category appears at least once. Return a `coverage_all` boolean and per-category indicators.

   ```python
   def audit_coverage(plan: ActionPlan, required: list[str]) -> dict:
       found = {a.type for a in plan.actions}
       per_cat = {cat: cat in found for cat in required}
       return {"coverage_all": all(per_cat.values()), "per_category": per_cat,
               "missing": [c for c, v in per_cat.items() if not v]}
   ```

4. **Build the Auditor module — calibration tracking.** Accumulate (confidence, correctness) pairs across episodes. Compute Brier score as `mean((p_i - y_i)^2)` and ECE by binning predictions into 10 bins and computing `sum((n_b/N) * |acc_b - conf_b|)`. Track these over time to detect calibration degradation.

   ```python
   def compute_brier(predictions: list[tuple[float, int]]) -> float:
       return sum((p - y) ** 2 for p, y in predictions) / len(predictions)

   def compute_ece(predictions: list[tuple[float, int]], bins: int = 10) -> float:
       # Bin predictions, compute |accuracy - confidence| per bin, weight by bin size
       ...
   ```

5. **Build the Auditor module — drift monitoring.** Maintain a baseline distribution of action types (e.g., from your first N episodes). For each new batch, compute the L1 distance between the current action-type distribution and baseline. Flag a drift warning if L1 > 0.4.

6. **Implement Tier 1: within-episode regeneration.** When the Auditor returns `coverage_all=False`, feed the original plan plus the Auditor's `missing` categories back into the Planner with an augmented prompt: "Your previous plan was missing: {missing}. Regenerate a complete plan covering all required categories." Cap regeneration attempts (e.g., max 2 retries).

7. **Implement Tier 2: discrepancy buffer and replay.** Log every case where `confidence >= 0.8` AND `coverage_all=False` to a persistent buffer (JSON file or database). Periodically run a batch replay job that re-processes buffered cases with enriched context (e.g., additional guidelines, the prior failure details).

   ```python
   def maybe_buffer(plan, audit_result, buffer_path):
       if plan.confidence >= 0.8 and not audit_result["coverage_all"]:
           entry = {"input": ..., "plan": plan.dict(), "audit": audit_result}
           append_json(buffer_path, entry)
   ```

8. **Implement triage routing.** Route each completed plan to one of three lanes:
   - **Green**: `coverage_all=True` — surface to user/clinician.
   - **Yellow**: `coverage_all=False` AND `confidence < 0.8` — flag for manual review.
   - **Red**: `coverage_all=False` AND `confidence >= 0.8` — the model is confidently wrong; buffer and replay.

9. **Wire up the end-to-end pipeline.** Orchestrate: gather input context -> Planner generates plan -> Auditor validates -> Tier 1 retry if needed -> triage route -> Tier 2 buffer if Red -> periodic replay batch. Emit structured logs at each step for observability.

10. **Track metrics across runs.** Maintain a running dashboard or log of: coverage rate, per-category hit rates, Brier score, ECE, drift L1, buffer size, high-confidence miss rate. Use these to decide when the system needs prompt tuning or context enrichment.

## Concrete Examples

**Example 1: Clinical Discharge Planning Pipeline**

User: "Build a discharge planning agent that generates action plans from FHIR patient data and validates completeness."

Approach:
1. Create a `FHIRClient` that queries a FHIR server for Patient, Condition, MedicationRequest, Observation, and Procedure resources for a given encounter.
2. Build a deterministic `SummaryGenerator` that flattens FHIR bundles into a `PatientSnapshot` (text summary + structured JSON) — no LLM involved in summarization to avoid hallucinating clinical data.
3. Implement the Planner: feed the PatientSnapshot into an LLM prompt that outputs an `ActionPlan` with actions in four categories (follow-up, medications, education, monitoring) plus a confidence score.
4. Implement the Auditor: check all four categories present, compute Brier/ECE from accumulated runs, check drift.
5. Wire Tier 1 regeneration and Tier 2 buffering.

Output structure:
```json
{
  "patient_id": "P-12345",
  "plan": {
    "actions": [
      {"type": "follow_up", "details": "Cardiology visit within 7 days", "deadline_hours": 168},
      {"type": "medication", "details": "Continue metoprolol 50mg BID, discontinue IV heparin", "deadline_hours": 24},
      {"type": "education", "details": "Heart failure self-monitoring: daily weights, fluid restriction", "deadline_hours": 48},
      {"type": "monitoring", "details": "Watch for dyspnea, weight gain >2lbs/day, chest pain", "deadline_hours": 72}
    ],
    "confidence": 0.85,
    "reasoning": "Clear CHF exacerbation with standard discharge protocol"
  },
  "audit": {"coverage_all": true, "brier": 0.12, "drift_l1": 0.08, "triage": "green"}
}
```

**Example 2: Code Review Agent with Auditor Validation**

User: "I want an LLM code review agent that checks for security, performance, style, and test coverage — and self-corrects when it misses categories."

Approach:
1. Define four mandatory review categories: `security`, `performance`, `style`, `test_coverage`.
2. Planner: LLM reviews the diff and outputs structured findings per category with a confidence score.
3. Auditor: deterministic check that all four categories have at least one finding or an explicit "no issues found" entry. Track calibration of the confidence score against human reviewer agreement over time.
4. Tier 1: if the Auditor finds missing categories, re-prompt: "Your review did not address: {missing}. Please review the diff again focusing on those areas."
5. Tier 2: buffer cases where the model was 80%+ confident but missed categories; replay weekly with additional context from style guides and security checklists.

Output:
```json
{
  "review": {
    "findings": [
      {"category": "security", "severity": "high", "detail": "SQL injection via unsanitized user input at line 42"},
      {"category": "performance", "detail": "N+1 query in user listing endpoint"},
      {"category": "style", "detail": "Inconsistent naming: mix of camelCase and snake_case"},
      {"category": "test_coverage", "detail": "No tests for the new /export endpoint"}
    ],
    "confidence": 0.78
  },
  "audit": {"coverage_all": true, "triage": "green"}
}
```

**Example 3: Adding Planner-Auditor to an Existing Agent**

User: "I have an existing LLM agent that generates project plans. Add a Planner-Auditor layer to catch omissions."

Approach:
1. Identify the mandatory sections the project plan must cover (e.g., `scope`, `timeline`, `risks`, `resources`, `milestones`).
2. Wrap the existing agent's output in an `ActionPlan` schema that adds a `confidence` field.
3. Build an Auditor that checks section presence and tracks calibration across historical plans.
4. Add Tier 1 retry logic: if sections are missing, re-invoke the agent with "Please add the missing sections: {missing}."
5. Add Tier 2 buffering for confidently incomplete plans to be replayed with additional project context.
6. Implement triage routing so green-lane plans go straight to stakeholders, yellow-lane plans get flagged, and red-lane plans are auto-retried.

## Best Practices

- **Do:** Keep the Auditor entirely deterministic. The power of this pattern comes from separating stochastic generation from rule-based validation. If your Auditor uses an LLM, you lose the reliability guarantee.
- **Do:** Require the Planner to output explicit confidence scores. This enables calibration tracking and is essential for the Red lane triage (catching confidently wrong outputs).
- **Do:** Cap Tier 1 regeneration attempts (2-3 max). Unbounded retries waste tokens and can cause the model to hallucinate to satisfy the checklist.
- **Do:** Persist discrepancy buffer entries with full context (input, plan, audit result) so replay has everything it needs.
- **Avoid:** Using the Auditor to modify plans directly. It should only observe and report — the Planner is the only component that generates or edits content.
- **Avoid:** Setting the drift threshold too low. L1 > 0.4 is a reasonable starting point; tighter thresholds cause false alarms in early runs with small sample sizes.
- **Avoid:** Treating Brier score and ECE as interchangeable. Brier measures overall accuracy of probability estimates; ECE measures binned calibration. Track both — a model can have decent Brier but poor ECE if errors are concentrated in specific confidence ranges.

## Error Handling

- **Planner returns malformed JSON:** Validate against the ActionPlan schema before passing to the Auditor. If parsing fails, retry once with a stricter prompt requesting valid JSON. If it fails again, route to Yellow lane for manual handling.
- **Auditor flags drift but coverage is fine:** This is an early warning, not an immediate failure. Log it, continue processing, and investigate if drift persists across multiple batches. It may indicate the input distribution has changed (new patient population, new code patterns).
- **Tier 1 regeneration still fails after max retries:** Route to Yellow lane. Do not force the model to produce a plan — a flagged incomplete plan is safer than a hallucinated complete one.
- **Discrepancy buffer grows unboundedly:** Set a buffer size limit or TTL. If cases aren't being resolved by replay, the issue is likely in the prompt or context, not the retry mechanism. Escalate for prompt engineering review.
- **Confidence scores cluster at extremes (all 0.95 or all 0.50):** The model isn't calibrating well. Add few-shot examples showing varied confidence levels with justification, or switch to a model with better inherent calibration.

## Limitations

- The Auditor can only check **structural completeness** (are categories present?) — it cannot assess **clinical or semantic correctness** of the content within each category. A plan that says "follow up with cardiology" for a dermatology patient will pass the coverage check.
- Confidence calibration requires **accumulated data** (50+ episodes minimum) to produce meaningful Brier/ECE metrics. Early runs will have noisy calibration numbers.
- Tier 2 replay assumes that a **second attempt with the same or enriched context** will succeed. For fundamentally ambiguous inputs, replay may not help.
- The mandatory-category model works best when the output space is **well-defined and enumerable**. For open-ended creative tasks with no fixed checklist, this pattern adds overhead without proportional benefit.
- Drift monitoring via L1 distance is a **coarse signal**. It detects large distributional shifts but won't catch subtle quality degradation within categories.

## Reference

**Paper:** Wu, K., Nagori, A., & Kamaleswaran, R. (2026). "Planner-Auditor Twin: Agentic Discharge Planning with FHIR-Based LLM Planning, Guideline Recall, Optional Caching and Self-Improvement." [arXiv:2601.21113v1](https://arxiv.org/abs/2601.21113v1)

Look for: Table 1 ablation results showing the impact of each component (baseline 32% -> self-improve 86% -> replay 100% coverage), the formal definitions of Brier/ECE/drift metrics, and the Green/Yellow/Red triage routing logic.