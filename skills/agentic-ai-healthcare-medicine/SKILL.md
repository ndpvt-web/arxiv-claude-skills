---
name: "agentic-ai-healthcare-medicine"
description: "Design, evaluate, and improve LLM-based agentic systems for healthcare using a seven-dimensional taxonomy with 29 sub-dimensions. Triggers: 'build a healthcare AI agent', 'evaluate my medical agent', 'healthcare agent architecture review', 'audit agent capabilities for clinical use', 'design a multi-agent medical system', 'gap analysis for healthcare LLM agent'."
---

This skill enables Claude to architect, evaluate, and systematically improve LLM-based agentic systems for healthcare and medicine using a rigorous seven-dimensional taxonomy derived from an empirical review of 49 studies (Vatsal, Dubey & Singh, 2026). Rather than ad-hoc agent design, this approach maps every agent capability to one of 29 operational sub-dimensions across Cognitive Capabilities, Knowledge Management, Interaction Patterns, Adaptation & Learning, Safety & Ethics, Framework Typology, and Core Tasks & Subtasks — then uses quantitative benchmarks of capability prevalence to identify gaps, prioritize development, and avoid known architectural pitfalls.

## When to Use

- When a user asks to design or scaffold a multi-agent system for clinical workflows (EHR analysis, diagnosis, treatment planning)
- When evaluating an existing healthcare AI agent against a structured maturity rubric
- When performing a gap analysis to find missing capabilities in a medical LLM agent
- When choosing between single-agent vs. multi-agent architectures for healthcare tasks
- When adding safety guardrails, human-in-the-loop controls, or regulatory compliance to a medical agent
- When building RAG pipelines, memory modules, or knowledge management for clinical data
- When reviewing code for a healthcare agent and needing to flag absent sub-dimensions (drift detection, event-triggered activation, error recovery)

## Key Technique: Seven-Dimensional Taxonomy

The taxonomy organizes healthcare agent capabilities into seven dimensions with 29 sub-dimensions. Each sub-dimension has a three-level rubric: **Fully Implemented (✓)**, **Partially Implemented (Δ)**, and **Not Implemented (✗)**, with precise criteria distinguishing each level. The empirical finding across 49 studies reveals stark asymmetries: retrieval-grounded capabilities dominate (External Knowledge Integration at 76% ✓, Multi-Agent Design at 82% ✓) while adaptation, safety, and action-oriented capabilities lag severely (Drift Detection at 96% ✗, Event-Triggered Activation at 92% ✗, Regulatory Compliance at 82% ✗).

The actionable insight is that most healthcare agents cluster in a "retrieval-advising" archetype — strong at ingesting knowledge and answering questions, weak at acting on decisions, adapting to distributional shifts, and satisfying regulatory requirements. A well-designed agent must consciously address the neglected dimensions. The taxonomy provides a checklist: if your agent scores ✗ on Treatment Planning, Safety Guardrails, or Human-in-the-Loop, those are not optional features — they are empirically-identified gaps that separate prototype from production-grade systems.

The co-occurrence analysis adds further guidance: Multi-Agent Design pairs naturally with Conversational Mode; External Knowledge Integration pairs with Medical QA but rarely with Dynamic Updates (meaning RAG pipelines are typically static). These patterns reveal where architectural choices create downstream constraints.

## The 29 Sub-Dimensions (Reference Card)

| # | Dimension | Sub-Dimension | Benchmark (✓ / Δ / ✗) |
|---|-----------|---------------|------------------------|
| 1 | Cognitive Capabilities | Planning | 43% / 39% / 18% |
| 2 | | Perception (Input Processing) | 49% / 47% / 4% |
| 3 | | Action (Output & Execution) | 43% / 20% / 37% |
| 4 | | Meta-Capabilities | 33% / 37% / 30% |
| 5 | | Consistency & Conflict Resolution | 35% / 27% / 38% |
| 6 | Knowledge Management | External Knowledge Integration | 76% / 8% / 16% |
| 7 | | Memory Module | 45% / 49% / 6% |
| 8 | | Dynamic Updates & Forgetting | 2% / 51% / 47% |
| 9 | Interaction Patterns | Conversational Mode | 45% / 12% / 43% |
| 10 | | Event-Triggered Activation | 4% / 4% / 92% |
| 11 | | Human-in-the-Loop | 20% / 8% / 72% |
| 12 | | Error Recovery | 14% / 47% / 39% |
| 13 | Adaptation & Learning | Drift Detection & Mitigation | 0% / 4% / 96% |
| 14 | | Reinforcement-Based Adaptation | 24% / 6% / 70% |
| 15 | | Meta-Learning & Few-Shot | 35% / 2% / 63% |
| 16 | Safety & Ethics | Safety Guardrails & Adversarial Robustness | 10% / 37% / 53% |
| 17 | | Bias & Fairness | 16% / 39% / 45% |
| 18 | | Privacy-Preserving Mechanism | 18% / 29% / 53% |
| 19 | | Regulatory & Compliance Constraints | 12% / 6% / 82% |
| 20 | Framework Typology | Multi-Agent Design | 82% / 6% / 12% |
| 21 | | Centralized Orchestration | 45% / 39% / 16% |
| 22 | Core Tasks | Clinical Documentation & EHR Analysis | 47% / 29% / 24% |
| 23 | | Medical QA & Decision Support | 65% / 20% / 15% |
| 24 | | Triage & Differential Diagnosis | 39% / 31% / 30% |
| 25 | | Diagnostic Reasoning | 41% / 27% / 32% |
| 26 | | Treatment Planning & Prescription | 12% / 29% / 59% |
| 27 | | Drug Discovery & Clinical Trial Design | 18% / 10% / 72% |
| 28 | | Patient Interaction & Monitoring | 10% / 8% / 82% |
| 29 | | Benchmarking & Simulation | 12% / 6% / 82% |

## Step-by-Step Workflow

### For Designing a New Healthcare Agent

1. **Define the target Core Tasks** — Select which of the 8 Core Task sub-dimensions the agent must address (e.g., Diagnostic Reasoning + Treatment Planning). Use the benchmark table above to understand baseline difficulty: Treatment Planning at 59% ✗ means expect significant engineering effort.

2. **Select Framework Typology** — Decide between multi-agent (specialized roles for retrieval, reasoning, safety checking) vs. monolithic agent. Multi-agent is dominant (82% ✓) for good reason: it enables modularity and redundancy. Design an explicit orchestration layer if choosing multi-agent — 39% of systems only partially implement orchestration.

3. **Design the Cognitive Pipeline** — For each agent, implement: (a) Planning — decompose clinical tasks into sub-goals with strategy comparison, not fixed workflows; (b) Perception — build encoders/parsers for each input modality (text, imaging, structured EHR); (c) Action — implement verified tool execution with precondition/postcondition checks, not just text generation; (d) Meta-Capabilities — add self-critique loops where the agent evaluates its own reasoning and flags uncertainty.

4. **Build Knowledge Management** — Implement a RAG pipeline with domain-specific medical knowledge bases (clinical guidelines, drug databases, ICD ontologies). Add a persistent memory module (episodic for patient history, semantic for domain knowledge). Critically, add dynamic update mechanisms — 47% of systems lack this, creating stale knowledge risk.

5. **Wire Interaction Patterns** — Implement conversational mode with session-scoped context. Add human-in-the-loop confirmation gates for high-stakes decisions (prescriptions, diagnoses). Build error recovery with transactional rollbacks and bounded retries. Consider event-triggered activation for monitoring use cases (92% absence means competitive advantage).

6. **Layer Safety & Ethics** — Implement multi-stage guardrails: input validation → reasoning audit → output filtering. Run stratified bias audits across demographic groups. Add privacy controls (role-based access, data retention schedules). Map to specific regulatory frameworks (HIPAA, GDPR) with documented consent flows.

7. **Add Adaptation Mechanisms** — Implement few-shot or in-context learning for new clinical scenarios. Add drift detection on incoming data distributions with automatic alerts. Consider RLHF or reward-based refinement from clinician feedback.

8. **Score the system against all 29 sub-dimensions** using the ✓/Δ/✗ rubric. Target ✓ on all sub-dimensions relevant to the deployment context. Flag any ✗ on Safety & Ethics sub-dimensions as blockers.

### For Evaluating an Existing Agent

1. **Collect implementation evidence** — For each of the 29 sub-dimensions, gather concrete evidence from code, documentation, and test results.

2. **Apply the rubric** — Rate each sub-dimension as ✓ (end-to-end with demonstrated evidence), Δ (mechanism present but incomplete), or ✗ (absent or asserted without evidence). Be conservative: default to Δ when claims are implicit or simulation-only.

3. **Generate the gap report** — Compare ratings against the benchmark prevalence table. Highlight sub-dimensions rated ✗ where the benchmark shows >30% ✓ (the agent is behind the field). Flag sub-dimensions rated ✗ in Safety & Ethics regardless of benchmark.

4. **Prioritize remediation** — Rank gaps by clinical risk (Safety first), then by benchmark prevalence (catch up to field), then by co-occurrence dependencies (e.g., fixing Error Recovery unblocks Treatment Planning).

## Concrete Examples

**Example 1: Scaffold a Multi-Agent Diagnostic System**

User: "Build me a multi-agent system that takes patient symptoms and lab results, generates differential diagnoses, and recommends next steps."

Approach:
1. Map to taxonomy: Core Tasks = Triage & Differential Diagnosis (39% ✓) + Diagnostic Reasoning (41% ✓). Framework = Multi-Agent Design.
2. Design three agents: (a) Intake Agent — parses symptoms and labs (Perception ✓), queries clinical guidelines via RAG (External Knowledge Integration ✓); (b) Reasoning Agent — generates ranked differential list with explicit reasoning chains (Planning ✓, Meta-Capabilities ✓); (c) Safety Agent — checks for contraindications, flags high-acuity cases for human review (Safety Guardrails ✓, Human-in-the-Loop ✓).
3. Implement orchestrator that routes data between agents with approval gates before final output (Centralized Orchestration ✓).
4. Add error recovery: if RAG retrieval returns empty, fallback to parametric knowledge with reduced confidence flag.

Output structure:
```python
# agent_config.py
AGENTS = {
    "intake": {
        "role": "Parse patient data and retrieve relevant guidelines",
        "tools": ["ehr_parser", "rag_retriever", "lab_normalizer"],
        "perception": "multimodal",  # structured EHR + free text
    },
    "reasoner": {
        "role": "Generate ranked differential diagnoses with reasoning",
        "tools": ["knowledge_graph", "reasoning_chain"],
        "meta_capabilities": {
            "self_critique": True,
            "confidence_calibration": True,
            "uncertainty_threshold": 0.7,  # flag for human review
        },
    },
    "safety_checker": {
        "role": "Validate outputs against safety constraints",
        "tools": ["drug_interaction_db", "guideline_validator"],
        "guardrails": ["contraindication_check", "demographic_bias_audit"],
        "human_in_the_loop": {
            "trigger": "high_acuity_or_low_confidence",
            "gate_type": "blocking",
        },
    },
}

ORCHESTRATOR = {
    "routing": "sequential_with_gates",
    "error_recovery": {
        "rag_fallback": "parametric_with_confidence_penalty",
        "max_retries": 2,
    },
}
```

**Example 2: Audit an Existing Medical QA Chatbot**

User: "I have a medical chatbot that answers patient questions using RAG over clinical guidelines. Can you evaluate it against the taxonomy?"

Approach:
1. Examine the codebase for evidence of each sub-dimension.
2. Apply the rubric conservatively.

Output — Gap Report:
```
TAXONOMY AUDIT REPORT
=====================

COGNITIVE CAPABILITIES
  Planning:                        Δ  (task breakdown exists, no strategy comparison)
  Perception:                      Δ  (text only, no multimodal)
  Action:                          ✗  (text-only output, no tool execution)
  Meta-Capabilities:               ✗  (no self-critique or confidence calibration)
  Consistency & Conflict:          ✗  (no conflict detection across sources)

KNOWLEDGE MANAGEMENT
  External Knowledge Integration:  ✓  (RAG pipeline with clinical guidelines)
  Memory Module:                   Δ  (conversation history only, no persistent memory)
  Dynamic Updates & Forgetting:    ✗  (static index, no temporal decay)

INTERACTION PATTERNS
  Conversational Mode:             ✓  (multi-turn with context)
  Event-Triggered Activation:      ✗  (user-initiated only)
  Human-in-the-Loop:               ✗  (no clinician review gates)
  Error Recovery:                  Δ  (basic retry, no fallback strategy)

ADAPTATION & LEARNING
  Drift Detection:                 ✗  CRITICAL — no monitoring for guideline changes
  Reinforcement-Based:             ✗
  Meta-Learning & Few-Shot:        ✗

SAFETY & ETHICS
  Safety Guardrails:               ✗  BLOCKER — no input/output filtering
  Bias & Fairness:                 ✗  BLOCKER — no demographic audit
  Privacy-Preserving:              Δ  (basic deidentification only)
  Regulatory Compliance:           ✗  BLOCKER — no HIPAA/GDPR mapping

FRAMEWORK TYPOLOGY
  Multi-Agent Design:              ✗  (monolithic)
  Centralized Orchestration:       ✗  (N/A for single agent)

PRIORITY REMEDIATION:
  1. [BLOCKER] Add safety guardrails — input sanitization + output filtering
  2. [BLOCKER] Implement bias audit across age/gender/race subgroups
  3. [BLOCKER] Map data handling to HIPAA requirements
  4. [HIGH]    Add human-in-the-loop gates for clinical recommendations
  5. [HIGH]    Implement confidence calibration and self-critique
  6. [MEDIUM]  Add persistent memory for longitudinal patient context
  7. [MEDIUM]  Implement dynamic knowledge base updates
```

**Example 3: Add Safety Layer to Treatment Planning Agent**

User: "My treatment planning agent generates prescription recommendations but has no safety checks. How do I add them?"

Approach:
1. Treatment Planning is 59% ✗ in the field — safety gaps here are the norm, not the exception. This must be addressed before deployment.
2. Map required sub-dimensions: Safety Guardrails, Human-in-the-Loop, Error Recovery, Regulatory Compliance, Bias & Fairness.
3. Implement a multi-stage verification pipeline.

Output:
```python
class TreatmentSafetyPipeline:
    """Multi-stage safety verification for prescription recommendations.
    Maps to taxonomy: Safety Guardrails (✓), Human-in-the-Loop (✓),
    Error Recovery (✓), Bias & Fairness (Δ)."""

    def verify(self, recommendation, patient_context):
        # Stage 1: Drug interaction check (Safety Guardrails)
        interactions = self.drug_db.check_interactions(
            proposed=recommendation.medications,
            current=patient_context.current_medications,
        )
        if interactions.severity == "critical":
            return self.block_with_explanation(interactions)

        # Stage 2: Dosage validation against weight/age/renal function
        dosage_ok = self.dosage_validator.check(
            recommendation, patient_context.demographics
        )

        # Stage 3: Bias audit — check if recommendation differs by
        # demographic group for same clinical presentation
        bias_flag = self.bias_auditor.check_demographic_parity(
            recommendation, patient_context
        )

        # Stage 4: Confidence gate — route to human review if uncertain
        if recommendation.confidence < 0.8 or bias_flag or not dosage_ok:
            return self.route_to_clinician(
                recommendation,
                flags={"bias": bias_flag, "dosage": not dosage_ok},
            )

        # Stage 5: Regulatory logging (HIPAA audit trail)
        self.audit_logger.log_decision(
            recommendation, patient_context, rationale=recommendation.chain_of_thought
        )
        return recommendation
```

## Best Practices

- **Do** score your agent against all 29 sub-dimensions before any deployment discussion. Treat ✗ on any Safety & Ethics sub-dimension as a deployment blocker.
- **Do** implement explicit orchestration when using multi-agent designs — 39% of systems leave this partially implemented, leading to inconsistent agent coordination.
- **Do** add dynamic update mechanisms to RAG pipelines. Clinical guidelines change; a static index becomes a liability.
- **Do** build error recovery with bounded retries and fallback strategies, not just retry loops.
- **Avoid** treating the taxonomy as a checklist to "get all ✓." Prioritize sub-dimensions relevant to your deployment context — a drug discovery agent doesn't need Patient Interaction.
- **Avoid** skipping Human-in-the-Loop for any agent that produces clinical recommendations. The 72% ✗ rate in the field is a warning, not a precedent.

## Error Handling

- **Rubric disagreement**: When evidence is ambiguous, default to Δ (partially implemented). Record the specific evidence and rationale for the rating so it can be revisited.
- **Missing sub-dimensions in codebase**: If a sub-dimension has no corresponding code or documentation, rate it ✗. Do not infer implementation from adjacent capabilities.
- **Co-occurrence conflicts**: If Multi-Agent Design is ✓ but Centralized Orchestration is ✗, flag this as an architectural risk — distributed agents without coordination lead to inconsistent outputs.
- **Scope creep during evaluation**: Evaluate only against sub-dimensions relevant to the agent's stated purpose. Not every agent needs Drug Discovery support.

## Limitations

- The taxonomy is derived from 49 studies published October 2023–June 2025. Emerging capabilities (e.g., agentic tool use patterns post-2025) may not be fully captured.
- The ✓/Δ/✗ rubric is categorical, not continuous. Two systems rated ✓ on the same sub-dimension may differ substantially in implementation quality.
- The benchmark percentages describe the research literature, not production deployments. Production systems may have different capability distributions.
- The taxonomy evaluates capability presence, not clinical effectiveness. A ✓ on Diagnostic Reasoning does not guarantee diagnostic accuracy.
- Safety & Ethics sub-dimensions are evaluated for technical implementation, not for regulatory approval. Passing the taxonomy audit is necessary but not sufficient for clinical deployment.

## Reference

**Paper**: Vatsal, Dubey & Singh (2026). "Agentic AI in Healthcare & Medicine: A Seven-Dimensional Taxonomy for Empirical Evaluation of LLM-based Agents." arXiv:2602.04813v1. https://arxiv.org/abs/2602.04813v1

**What to look for**: The full 29 sub-dimension definitions with ✓/Δ/✗ criteria (Section III), the per-study evaluation matrices (Tables I–VIII), and the co-occurrence analysis showing which capabilities cluster together and which are systematically absent.