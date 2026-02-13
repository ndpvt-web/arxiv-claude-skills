---
name: "grounding-generative-planners-verifiable"
description: >
  Build neuro-symbolic safety verification pipelines using the VIRF (Verifiable Iterative Refinement Framework)
  pattern: a Logic Tutor provides formal, causal feedback to an LLM planner, enabling intelligent plan repair
  instead of mere rejection. Use this skill when:
  - "Add safety verification to my LLM agent pipeline"
  - "Build a plan validator with formal logic feedback"
  - "Create a tutor-apprentice loop for safe AI planning"
  - "Implement iterative plan refinement with ontology checks"
  - "Add OWL-based safety constraints to my agent"
  - "Build a verifiable planning system with correction feedback"
---

# Grounding Generative Planners in Verifiable Logic (VIRF Pattern)

This skill teaches Claude to implement the **Verifiable Iterative Refinement Framework (VIRF)** — a neuro-symbolic architecture where a deterministic Logic Tutor, grounded in a formal safety ontology (OWL 2 / Description Logics), provides causal, pedagogical feedback to an LLM planner. Instead of simply rejecting unsafe plans, the tutor explains *why* a plan is unsafe via a structured diagnostic report (Action -> Axiom -> Violation), enabling the LLM to repair plans intelligently. The original paper achieves 0% hazardous action rate with only 1.1 correction iterations on average.

## When to Use

- When building an LLM-based agent that generates action plans and those plans must satisfy formal safety or business constraints before execution
- When the user wants to add a verification layer to an existing AI planner that provides *corrective feedback* rather than binary accept/reject
- When implementing a robotics, home automation, or embodied AI pipeline that needs provable safety guarantees
- When the user asks to encode domain rules (safety, compliance, policy) as an ontology and verify LLM outputs against it
- When building any system where LLM outputs must be iteratively refined against deterministic rules — e.g., code generation with formal spec checks, workflow orchestration with policy constraints
- When the user needs a knowledge acquisition pipeline to extract and formalize safety rules from regulatory documents

## Key Technique

VIRF instantiates **Dual Process Theory**: the LLM acts as System 1 (fast, creative, error-prone) while the Logic Tutor acts as System 2 (slow, deliberate, logically rigorous). The tutor uses an OWL 2 ontology with a TBox (abstract safety principles) and ABox (concrete world-state assertions), verified by a DL reasoner (e.g., Pellet via `owlready2`). When the LLM proposes a plan, each step is checked in parallel against the ontology. If unsafe, the tutor generates a pedagogical report exposing the causal chain of failure, which is fed back to the LLM for repair.

The critical insight is that **feedback quality determines repair quality**. A simple "unsafe" label forces the LLM to guess; a structured diagnostic like "Step 3 (place metal bowl in microwave) violates axiom `MetalInMicrowave -> ArcingHazard` because the bowl `isMadeOfMetal` and the microwave `emitsMicrowaves`" lets the LLM surgically fix the problem. This is why VIRF needs only ~1.1 iterations on average — the feedback is precise enough for single-shot repair.

The **Traceable Axiom Synthesis (TAS)** pipeline is the second key contribution: a three-stage process (RAG indexing of authority documents, semantic vocabulary alignment with a base ontology like BFO, interactive LLM-assisted axiom extraction with mandatory source citation) that produces a permanent, verified TBox from real-world regulatory documents. This is a design-time process, not runtime RAG — the axioms are logically validated once and reused.

## Step-by-Step Workflow

1. **Define the domain ontology skeleton.** Create an OWL 2 ontology file (`.owl`) with class hierarchies for your domain. Use Basic Formal Ontology (BFO) as the upper ontology if applicable. Separate the TBox (rules, constraints, class definitions) from the ABox (instance assertions about the current state). Use `owlready2` in Python for programmatic access.

2. **Encode safety/constraint axioms in the TBox.** For each domain rule, write an OWL 2 axiom. Structure each as: "If an action has property X and the target object has property Y, then the resulting state is classified as UnsafeState." Keep axioms traceable — annotate each with a source reference (regulation, policy document, or domain expert justification).

3. **Build the ABox from perception/context.** Before plan verification, populate the ABox with the current world state. For embodied AI, use a VLM cascade (detect objects -> refine safety-critical attributes -> extract spatial relations). For software tasks, parse the current system state into ontology instances. Key attributes: object classes, states, materials, spatial relations.

4. **Implement the LLM planner (Apprentice).** Prompt the LLM to generate a step-by-step plan given the user command and ABox context. Each step should be a structured action with: action type, target object(s), preconditions assumed, and expected postconditions. Use JSON or a typed schema for plan steps.

5. **Build the Logic Tutor verifier.** For each plan step, simulate the state transition and run the DL reasoner (Pellet) to check if any individual is classified under an UnsafeState concept. Verify all N steps in parallel for efficiency. Return one of three verdicts: SAFE, UNSAFE (with diagnostic), or UNKNOWN (information gap).

6. **Generate the pedagogical feedback report.** When UNSAFE, construct a structured diagnostic: identify the violating action, the triggered axiom, the causal chain (which object properties led to the violation), and a natural-language explanation. Format: `{action, axiom_id, causal_chain, explanation, suggested_direction}`.

7. **Implement the iterative refinement loop.** Feed the diagnostic report back to the LLM with the original command and conversation history. Cap iterations at N_max (typically 3-5). If the LLM returns a refusal or cannot repair after N_max attempts, return TASK_REJECTED with the reason.

8. **Handle UNKNOWN verdicts.** When the reasoner cannot determine safety (missing information), prompt the user or an oracle to provide the missing fact. Update the ABox accordingly and re-verify. This implements the Open World Assumption — absence of information is not assumed safe.

9. **Build the knowledge acquisition pipeline (TAS).** For new domains: (a) index authority documents with a vector store, (b) vectorize the base ontology vocabulary, (c) use an LLM to draft candidate OWL axioms from retrieved evidence with mandatory source citations, (d) have a domain expert validate each axiom for semantic correctness and logical consistency.

10. **Test with adversarial scenarios.** Measure four metrics: Hazardous Action Rate (HAR — must be 0%), Goal-Condition Rate (GCR — task completion), False Positive Rate (FPR — over-rejection), and average correction iterations. Stress-test with edge cases where safety violations are subtle.

## Concrete Examples

**Example 1: Safe Home Task Planner**

User: "Build a safety-verified planner for a home robot that can cook, clean, and organize."

Approach:
1. Define ontology with classes: `Appliance`, `Container`, `Food`, `Chemical`, `HeatSource`, `UnsafeState`
2. Encode axioms like:
   ```python
   # Using owlready2
   with onto:
       class MetalObject(Thing): pass
       class Microwave(Appliance): pass
       class ArcingHazard(UnsafeState): pass

       # Axiom: Metal in active microwave -> arcing hazard
       class causes_arcing(MetalObject >> ArcingHazard):
           equivalent_to = [MetalObject & is_inside.some(Microwave & has_state.value("active"))]
   ```
3. LLM generates plan: `["pick_up(metal_bowl)", "place_in(metal_bowl, microwave)", "turn_on(microwave)"]`
4. Tutor detects: Step 2 + Step 3 triggers `ArcingHazard`
5. Feedback to LLM:
   ```json
   {
     "verdict": "UNSAFE",
     "violating_step": 2,
     "axiom": "MetalInActiveMicrowave -> ArcingHazard",
     "causal_chain": "metal_bowl.isMadeOfMetal AND microwave.emitsMicrowaves -> ArcingHazard",
     "explanation": "Placing a metal object in a microwave causes electrical arcing. Use a microwave-safe container instead.",
     "suggestion": "Replace metal_bowl with a ceramic or glass container."
   }
   ```
6. LLM repairs: `["pick_up(glass_bowl)", "transfer(food, glass_bowl)", "place_in(glass_bowl, microwave)", "turn_on(microwave)"]`
7. Tutor re-verifies: SAFE. Execute plan.

**Example 2: Compliance-Verified Workflow Automation**

User: "Add safety verification to our data processing pipeline that handles PII."

Approach:
1. Define ontology: `DataField`, `PIIField(DataField)`, `StorageTarget`, `UnencryptedStore(StorageTarget)`, `ComplianceViolation(UnsafeState)`
2. Encode GDPR/CCPA axioms:
   ```python
   # Axiom: PII written to unencrypted storage -> compliance violation
   class pii_exposure(ComplianceViolation):
       equivalent_to = [has_action.value("write") &
                        has_subject.some(PIIField) &
                        has_target.some(UnencryptedStore)]
   ```
3. LLM generates ETL plan: `["read(users_table)", "transform(extract_emails)", "write(emails, s3_public_bucket)"]`
4. Tutor detects: `emails` is `PIIField`, `s3_public_bucket` is `UnencryptedStore`
5. Feedback:
   ```json
   {
     "verdict": "UNSAFE",
     "violating_step": 3,
     "axiom": "PIIToUnencryptedStore -> ComplianceViolation",
     "causal_chain": "emails.isPII AND s3_public_bucket.isUnencrypted -> GDPR_Article_32_Violation",
     "explanation": "Writing PII (email addresses) to an unencrypted public S3 bucket violates data protection requirements.",
     "suggestion": "Use an encrypted, access-controlled storage target. Apply pseudonymization if the downstream task permits."
   }
   ```
6. LLM repairs to use encrypted private bucket with SSE-KMS.

**Example 3: Knowledge Acquisition from Regulatory Docs**

User: "Extract safety rules from our lab chemical handling manual into formal axioms."

Approach:
1. Index the manual PDF with a vector store (e.g., FAISS + `bge-m3` embeddings)
2. Vectorize BFO + domain ontology vocabulary with `all-MiniLM-L6-v2`
3. For each safety topic, retrieve relevant passages and prompt the LLM:
   ```
   Given this passage from the chemical handling manual:
   "[Acids must never be stored in the same cabinet as bases...]"

   Draft an OWL 2 axiom in Manchester Syntax. You MUST cite the exact source sentence.
   Use only classes from the provided ontology vocabulary.
   ```
4. LLM outputs:
   ```
   Class: UnsafeChemicalStorage
     EquivalentTo: Storage and (contains some Acid) and (contains some Base)
     Source: "Acids must never be stored in the same cabinet as bases" (Section 4.2, p.12)
   ```
5. Domain expert validates logical correctness, adds to TBox.
6. Repeat until coverage is sufficient (target: high acceptance rate, ~92 axioms per 2-day sprint).

## Best Practices

- **Do:** Keep axioms atomic and traceable. Each axiom should encode exactly one safety rule with a citation. This makes debugging and updating straightforward.
- **Do:** Verify plan steps in parallel. Each step's state simulation is independent once the predecessor state is computed — batch reasoner calls to minimize latency.
- **Do:** Use the Open World Assumption. When information is missing (UNKNOWN verdict), ask rather than assume safe. This is the principled default for safety-critical systems.
- **Do:** Structure feedback as `(Action, Axiom, CausalChain, Explanation, Suggestion)`. The richer the feedback, the fewer correction iterations needed.
- **Avoid:** Using the LLM itself to perform safety verification. The entire point of VIRF is that verification is deterministic and provable — mixing in stochastic checks defeats the guarantee.
- **Avoid:** Encoding too many rules as soft constraints or heuristics. If a rule is safety-critical, it must be a formal axiom with DL reasoning, not a prompt instruction.
- **Avoid:** Skipping the ABox grounding step. The ontology is only as good as the world-state it reasons over — invest in accurate perception/context extraction.

## Error Handling

- **Reasoner timeout:** Pellet can be CPU-intensive on large ontologies. Set a timeout (e.g., 30s per step) and treat timeouts as UNKNOWN — do not default to SAFE. Consider ontology modularization for large domains.
- **LLM refuses to repair:** After N_max iterations, the plan may be fundamentally incompatible with safety constraints. Return TASK_REJECTED with the full diagnostic history so the user understands why.
- **Ontology inconsistency:** If the reasoner reports the ontology itself is inconsistent (unsatisfiable classes), halt verification and flag the TBox for review. Never verify against an inconsistent ontology.
- **ABox perception errors:** False negatives in object detection are more dangerous than false positives (missing a hazard vs. over-caution). Tune perception toward recall over precision for safety-critical attributes.
- **Axiom gaps:** When the system encounters scenarios with no applicable axioms, log them as coverage gaps. Feed these back into the TAS pipeline for the next axiom synthesis cycle.

## Limitations

- **Ontology engineering cost:** Building the initial TBox requires domain expertise and is labor-intensive. TAS reduces but does not eliminate this cost. Small teams should start with a narrow domain scope.
- **Expressiveness ceiling:** OWL 2 DL is decidable but limited — some constraints (e.g., counting, complex temporal sequences) require extensions or approximations. Not all safety rules map cleanly to DL axioms.
- **Perception bottleneck:** In embodied settings, the system is only as safe as its perception pipeline. If the VLM misclassifies a metal bowl as ceramic, the ontology cannot catch the downstream error.
- **Scalability of reasoning:** Pellet performance degrades with very large ABoxes (thousands of instances). For high-cardinality domains, consider OWL 2 EL or RL profiles for polynomial-time reasoning.
- **Not a substitute for runtime monitoring:** VIRF verifies plans before execution. It does not handle dynamic environment changes during execution — pair with runtime safety monitors for full coverage.

## Reference

**Paper:** [Grounding Generative Planners in Verifiable Logic: A Hybrid Architecture for Trustworthy Embodied AI](https://arxiv.org/abs/2602.08373v1) (Wu et al., ICLR 2026)
**Key takeaway:** Look at Algorithm 1 (the refinement loop), the pedagogical feedback report structure, and the TAS pipeline (Figure 2) for implementable patterns. The OWL 2 + Pellet + owlready2 stack is production-ready for Python deployments.