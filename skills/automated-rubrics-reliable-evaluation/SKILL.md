---
name: "automated-rubrics-reliable-evaluation"
description: "Generate fine-grained evaluation rubrics for medical dialogue systems using a retrieval-augmented multi-agent pipeline. Decomposes medical evidence into atomic facts, synthesizes them with interaction constraints, and produces weighted, auditable rubrics. Use when: 'evaluate medical chatbot responses', 'generate rubrics for clinical QA', 'build a medical LLM evaluation pipeline', 'score health dialogue quality', 'create automated clinical evaluation criteria', 'refine medical AI responses with rubric feedback'."
---

# Automated Rubrics for Reliable Evaluation of Medical Dialogue Systems

This skill enables Claude to build retrieval-augmented multi-agent pipelines that automatically generate instance-specific evaluation rubrics for medical dialogue systems. Rather than relying on generic metrics or expensive expert annotation, the technique retrieves authoritative medical evidence, decomposes it into atomic facts (positive assertions, contraindications, safety red flags), extracts interaction intent constraints from the user query, and synthesizes both tracks into weighted, verifiable evaluation criteria. The resulting rubrics can score LLM responses, discriminate between subtly different answer qualities, and guide targeted response refinement.

## When to Use

- When the user needs to **evaluate medical chatbot or clinical QA system outputs** against evidence-based criteria rather than surface-level metrics like BLEU or ROUGE.
- When building an **automated evaluation pipeline for HealthBench-style medical benchmarks** where physician-authored rubrics are unavailable or too costly.
- When the user wants to **detect subtle clinical errors** (wrong dosage, missing contraindication, omitted red flag) that generic evaluators miss.
- When implementing a **rubric-guided response refinement loop** where an LLM's medical answer is iteratively improved against structured criteria.
- When the user asks to **score and rank multiple medical LLM responses** with discriminative sensitivity (near-miss detection between answers differing by one clinical fact).
- When designing a **multi-agent framework** where specialized agents handle retrieval routing, fact decomposition, intent extraction, rubric synthesis, and auditing.

## Key Technique

The core insight is **dual-track constraint construction**: medical evidence and user interaction intent are processed in parallel by specialized agents, then merged into a single rubric. The objective track retrieves content from authoritative sources (CDC, WHO, PubMed, Mayo Clinic, drug databases), synthesizes overlapping snippets, and decomposes them into three categories of atomic facts: positive facts (declarative assertions, dosage ranges, conditional logic), negative constraints (explicit prohibitions, contraindications), and safety red flags (emergency warnings). The subjective track extracts explicit instructions and implicit communication cues from the user query, identifying medically necessary but missing contextual variables (e.g., patient age, comorbidities not stated).

These two tracks feed into a Rubric Synthesis Agent that maps each atomic fact and interaction constraint to a structured criterion tuple `(criterion_text, evaluation_axis, clinical_weight)`. Each criterion is assigned to one of five axes: **accuracy** (factual correctness, safety violations), **completeness** (topic coverage), **context_awareness** (clarifying questions for missing info), **communication_quality** (tone, empathy, clarity), and **instruction_following** (formatting constraints). Clinical weights range from -10 to +10 following severity tiers: safety-critical items get extreme weights (|8-10|), completeness items get moderate weights (|4-7|), and minor details get low weights (|1-3|). Negative weights penalize harmful content.

An Auditing Agent then runs three-phase gap analysis: (1) scan source evidence for uncovered facts and generate missing criteria, (2) filter hallucinated or irrelevant criteria and validate that negative constraints are present, (3) merge fragmented criteria while enforcing a cap of 20 criteria per rubric and preserving all safety red flags. This audit loop is what prevents both hallucinated evaluation criteria and dangerous omissions.

## Step-by-Step Workflow

1. **Route the medical query to search**: Implement a Routing Agent that transforms the user's medical query into 3-5 optimized search queries. Use a high-capacity model for intent identification and query generation, and a lightweight model for reranking retrieved results by source authority (prioritize CDC, WHO, PubMed over generic web results).

2. **Retrieve and synthesize evidence**: Fetch the top-5 results per search query from medical knowledge sources. Use an Evidence Synthesis Agent to cross-check overlapping claims, de-duplicate, and consolidate into a single evidence block per query. Extract text with a tool like Trafilatura to handle diverse HTML layouts.

3. **Decompose evidence into atomic facts (Reference Board)**: Run a Medical Fact Agent that breaks the synthesized evidence into three categories:
   - `positive_facts`: Declarative assertions, quantitative data (dosages, thresholds), conditional logic ("if X then Y").
   - `negative_constraints`: Explicit prohibitions ("do not prescribe X with Y"), contraindications.
   - `safety_red_flags`: Emergency indicators ("seek immediate care if..."), critical alerts.
   Apply query-aware filtering to retain only relevant positive facts, but **always preserve all safety red flags and negative constraints regardless of query specificity**.

4. **Extract interaction intent constraints**: Run an Interaction Intent Agent in parallel with step 3. Parse the user query to extract explicit instructions (e.g., "explain in simple terms"), implicit communication cues (e.g., expressed anxiety suggesting empathetic tone), and identify medically necessary but missing contextual variables (e.g., unstated age, allergies, current medications) that the response should ask about.

5. **Synthesize the initial rubric**: Feed both the Reference Board (atomic facts) and interaction constraints into a Rubric Synthesis Agent. For each fact or constraint, generate a criterion tuple:
   ```json
   {
     "criterion": "Response must warn against combining ibuprofen with blood thinners",
     "axis": "accuracy",
     "weight": -9
   }
   ```
   Assign weights using clinical severity tiers: |8-10| for safety/accuracy-critical, |4-7| for completeness, |1-3| for minor communication or formatting items. Use negative weights for criteria that penalize harmful content.

6. **Audit and refine the rubric**: Run a three-phase Auditing Agent:
   - Phase 1: Scan source facts for any uncovered by existing criteria; generate new criteria for gaps.
   - Phase 2: Remove hallucinated criteria not grounded in retrieved evidence; verify negative constraint criteria exist; check all five evaluation axes are represented.
   - Phase 3: Merge overlapping criteria; enforce a maximum of 20 criteria; ensure safety red flags are never dropped during merging.

7. **Score responses against the rubric**: For each LLM response under evaluation, check every criterion. Use a structured judging protocol: run N=3 trials with order swapping (if comparing two responses) for 6 total runs, then decide by majority vote. Calculate a weighted sum of satisfied/violated criteria as the final score.

8. **Compute discriminative metrics**: For paired evaluation (reference vs. candidate), calculate:
   - **Win Rate**: Proportion of pairs where the better response scores higher.
   - **Mean Score Delta**: Average score difference across all pairs (larger = better discrimination).
   - **AUROC**: Probability that the rubric correctly ranks the reference above the candidate.

9. **Generate refinement feedback (optional)**: Convert rubric violations into a structured Edit Plan:
   ```json
   {
     "actions": [
       {"type": "ADD", "priority": 1, "detail": "Include warning about renal impairment risk with NSAIDs in elderly patients"},
       {"type": "REMOVE", "priority": 2, "detail": "Remove unsupported claim about herbal remedy efficacy"}
     ]
   }
   ```
   Feed this plan to a Refinement Agent that edits the response while strictly prohibited from introducing claims not in the evidence.

10. **Validate and report**: Calculate Clinical Intent Alignment (CIA) against any available gold-standard rubrics: `CIA = (matched keypoints / total keypoints)`. Report per-axis coverage breakdown and flag any axes with zero criteria as potential blind spots.

## Concrete Examples

**Example 1: Evaluating a medication interaction response**

User: "Build an evaluation rubric for a chatbot answering: 'Can I take ibuprofen with my blood pressure medication lisinopril?'"

Approach:
1. Route query to search: generate queries like "ibuprofen lisinopril interaction", "NSAID ACE inhibitor contraindication", "ibuprofen blood pressure effects".
2. Retrieve evidence from Drugs.com, PubMed, Mayo Clinic. Synthesize into a single evidence block.
3. Decompose into atomic facts:
   - Positive: "Ibuprofen can reduce the antihypertensive effect of ACE inhibitors including lisinopril."
   - Positive: "Occasional low-dose ibuprofen may be acceptable with monitoring."
   - Negative: "Chronic concurrent use increases risk of renal impairment."
   - Red flag: "Seek immediate care if experiencing signs of kidney failure: reduced urine output, swelling, fatigue."
4. Extract intent: User wants a direct yes/no answer with safety guidance. Missing context: patient age, kidney function, duration of use.

Output rubric (abbreviated):
```
Criterion                                                    | Axis              | Weight
-------------------------------------------------------------|-------------------|-------
States ibuprofen can reduce lisinopril's BP-lowering effect  | accuracy          | +8
Warns about increased renal impairment risk                  | accuracy          | -9
Mentions occasional low-dose may be acceptable with MD consult| completeness     | +6
Asks about patient's kidney function status                  | context_awareness | +5
Includes emergency signs requiring immediate care            | accuracy          | -8
Does NOT recommend chronic concurrent use as safe            | accuracy          | -10
Uses accessible language appropriate for patient query       | communication     | +3
Suggests consulting prescribing physician before combining   | completeness      | +7
```

**Example 2: Discriminating between two chatbot responses about chest pain**

User: "I have two chatbot responses to 'I'm having chest pain after exercise.' Score them with an automated rubric and tell me which is better."

Approach:
1. Generate rubric for the chest pain query following the full pipeline.
2. Key atomic facts include: cardiac vs. musculoskeletal differential, red flags (radiating pain, shortness of breath, nausea), urgency of seeking care, questions about pain character/duration/history.
3. Score Response A and Response B independently against all criteria.
4. Run 3 trials with response order swapped (6 total), majority vote per criterion.
5. Compute weighted scores and mean score delta.

Output:
```
Response A score: 72/100 (missed: did not ask about pain radiation pattern, did not mention calling 911 for severe symptoms)
Response B score: 41/100 (missed: incorrectly stated exercise-induced chest pain is "usually nothing to worry about", omitted cardiac red flags)
Mean Score Delta: 31 points
Winner: Response A (statistically significant across all 6 judging runs)

Key discriminating criteria:
- "Must not dismiss chest pain as benign without ruling out cardiac causes" (weight: -10): A passed, B failed
- "Must list red flag symptoms requiring emergency care" (weight: -9): A passed, B failed
```

**Example 3: Rubric-guided response refinement**

User: "Here's a chatbot response about managing Type 2 diabetes. Use rubric-based feedback to improve it."

Approach:
1. Generate rubric for the diabetes management query.
2. Score the existing response, identifying violated criteria.
3. Convert violations into an Edit Plan with prioritized actions.
4. Apply refinement: add missing content, remove unsupported claims, adjust tone.

Output:
```
Original score: 59/100

Edit Plan:
1. [ADD, priority 1] Include warning about hypoglycemia signs when on sulfonylureas
2. [ADD, priority 2] Ask about current HbA1c level and medication list
3. [MODIFY, priority 3] Replace "you should exercise daily" with evidence-based recommendation (150 min/week moderate activity per ADA guidelines)
4. [REMOVE, priority 4] Remove unsubstantiated claim about cinnamon supplements lowering blood sugar

Refined score: 68/100 (+9.2% improvement)
All safety red flags now covered. Context awareness improved from 2/5 to 4/5 criteria met.
```

## Best Practices

- **Do:** Always preserve safety red flags and negative constraints during filtering and auditing, even if they seem tangentially related to the query. A missed contraindication is far worse than an extra criterion.
- **Do:** Use authoritative medical sources (CDC, WHO, PubMed, NICE guidelines, established drug databases) for retrieval. Source authority directly determines rubric quality.
- **Do:** Cap rubrics at 20 criteria maximum. Beyond this, criteria become redundant or too fine-grained for reliable automated scoring.
- **Do:** Assign negative weights to criteria checking for harmful content (wrong dosage, dangerous omission, false reassurance). This ensures unsafe responses score dramatically lower, not just slightly lower.
- **Avoid:** Generating criteria not grounded in retrieved evidence. Every criterion must trace back to a specific atomic fact or interaction constraint. Hallucinated criteria undermine the entire evaluation.
- **Avoid:** Using a single evaluation axis. Medical responses must be assessed across accuracy, completeness, context awareness, communication quality, and instruction following simultaneously. Single-axis rubrics miss critical failure modes.

## Error Handling

- **No relevant evidence retrieved**: Fall back to the model's parametric medical knowledge but flag the rubric as "ungrounded" and reduce confidence. Recommend the user supply domain-specific references manually.
- **Conflicting evidence across sources**: The Evidence Synthesis Agent should flag contradictions explicitly. Create criteria for both positions with a note that the response should acknowledge the clinical controversy.
- **Rubric exceeds 20 criteria after auditing**: Re-run the merge phase of the Auditing Agent with stricter deduplication. Prioritize safety-critical and accuracy criteria over communication and formatting criteria during merging.
- **Low CIA score against gold standard**: Indicates the retrieval step missed key medical concepts. Expand search queries, add source-specific queries (e.g., searching PubMed separately from patient education sites), and re-run the pipeline.
- **Judging inconsistency across trials**: If majority vote does not converge (e.g., 3-3 split across 6 runs), flag the criterion as ambiguous and consider rewording it for clarity before re-scoring.

## Limitations

- Rubric quality is bounded by retrieval quality. If authoritative sources lack coverage of a niche medical topic (rare diseases, emerging treatments), the rubric will have gaps that no amount of auditing can fill.
- The framework is designed for English-language medical dialogue. Cross-lingual medical terminology and cultural health practices require additional adaptation.
- Clinical weight assignment follows predefined severity tiers, not patient-specific risk profiles. A weight of -9 for a drug interaction may be appropriate for most patients but insufficient for a high-risk patient.
- Automated rubrics cannot fully replace expert physician review for high-stakes clinical deployment. They are best used as a scalable first pass that flags responses for human review.
- The 20-criterion cap may be insufficient for complex multi-condition queries (e.g., a patient with diabetes, hypertension, and chronic kidney disease asking about pain management).

## Reference

**Paper**: "Automated Rubrics for Reliable Evaluation of Medical Dialogue Systems" by Chen et al. (2026). [arXiv:2601.15161](https://arxiv.org/abs/2601.15161v1). Focus on Section 3 (three-stage pipeline architecture), Section 4 (CIA metric and discriminative sensitivity), and Appendix B (agent prompt templates) for implementation details.