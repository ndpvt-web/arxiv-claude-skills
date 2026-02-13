---
name: "core-comprehensive-ontological-relation"
description: "Detect and prevent semantic collapse in LLM outputs — where models fabricate spurious relationships between unrelated concepts. Apply CORE-style ontological relation evaluation to audit code, data pipelines, knowledge graphs, and AI systems for unrelatedness reasoning failures. Use when: 'check if these concepts are actually related', 'audit my ontology for spurious relations', 'evaluate semantic relationships in my knowledge graph', 'detect hallucinated connections in LLM output', 'validate entity relationships in my schema', 'test unrelatedness reasoning in my AI pipeline'."
---

# CORE: Comprehensive Ontological Relation Evaluation

This skill enables Claude to apply the CORE framework (Dwivedi et al., 2026) to detect **semantic collapse** — the systematic failure where language models fabricate meaningful relationships between genuinely unrelated concepts. The paper demonstrates that state-of-the-art LLMs achieve 86.5–100% accuracy identifying related pairs but collapse to 0–41.35% on unrelated pairs, while maintaining 92–94% confidence in both cases. This skill operationalizes that finding: it teaches Claude to rigorously evaluate whether semantic relationships in code, data, and AI outputs are genuine or spurious, treating unrelatedness as a first-class reasoning category rather than an afterthought.

## When to Use

- When a user asks to validate or audit relationships in a knowledge graph, ontology, or entity-relationship schema
- When reviewing LLM-generated output that claims connections between concepts (e.g., RAG pipeline results, automated tagging, entity linking)
- When building or evaluating a classification system that must distinguish "related" from "unrelated" pairs
- When designing test suites or benchmarks for semantic reasoning systems
- When a user needs to check whether concept mappings in their codebase (enums, taxonomies, category hierarchies) contain spurious groupings
- When debugging recommendation systems, search relevance, or semantic similarity pipelines that surface false connections
- When constructing multiple-choice evaluations or quiz systems that need valid distractor options (unrelated choices)

## Key Technique

**Semantic Collapse** is the core failure mode CORE identifies. When an LLM encounters two concepts — say "photosynthesis" and "corporate tax law" — it will often confabulate a plausible-sounding relation ("both involve resource conversion") with high confidence, rather than correctly identifying them as unrelated. The CORE paper quantifies this: across 29 models, the mean semantic collapse rate is 37.6%, meaning over a third of unrelated pairs are incorrectly assigned a fabricated relationship. Critically, model confidence stays at 92–94% regardless of whether the pair is related or not, so confidence scores cannot be used to filter these errors.

The CORE evaluation framework uses 24 semantic relation types organized across ontological categories: taxonomic relations (hypernymy, hyponymy, holonymy, meronymy), associative relations (causation, correlation, functional dependence), equivalence relations (synonymy, paraphrase), oppositional relations (antonymy, complementarity), temporal/spatial relations, and crucially, **unrelatedness** as an equal-weight category. The 203-question benchmark enforces equal representation of unrelated pairs — a design choice that exposes the asymmetry in LLM reasoning. The full 225K MCQ dataset spans 74 academic disciplines and drops LLM accuracy to ~2%, revealing that domain-specific unrelatedness reasoning is an even harder frontier.

The actionable insight is a **dual-verification protocol**: never accept a claimed semantic relationship without explicitly testing the null hypothesis (that the concepts are unrelated). This mirrors the paper's finding that Expected Calibration Error increases 2–4x on unrelated pairs — models are not just wrong, they are confidently wrong. Any system that relies on LLM-judged semantic relationships without adversarial unrelatedness testing will inherit this systematic bias toward spurious connections.

## Step-by-Step Workflow

1. **Identify the relationship claims under evaluation.** Extract all concept pairs and their asserted relationships from the target artifact — whether it's a knowledge graph, ontology file, LLM output, schema definition, or classification taxonomy. Represent each as a structured triple: `(Concept_A, Relation_Type, Concept_B)`.

2. **Classify each relation into one of the 24 CORE relation categories.** Map each asserted relation to the appropriate type: taxonomic (is-a, part-of, has-part, instance-of), associative (causes, enables, requires, co-occurs-with), equivalence (same-as, similar-to), oppositional (opposite-of, contradicts), temporal (precedes, follows), spatial (located-in, adjacent-to), functional (used-for, produced-by), or unrelatedness (no-meaningful-relation). If the relation doesn't fit cleanly, flag it for closer inspection.

3. **Apply the unrelatedness null hypothesis test.** For each pair, explicitly ask: "Can I construct a coherent argument that these two concepts have NO meaningful semantic relationship?" If you can, and the counter-argument is at least as strong as the claimed relation, mark the pair as a semantic collapse candidate. Do not rely on confidence or plausibility — the CORE finding is that spurious relations sound plausible by design.

4. **Check for cross-domain confabulation.** Pairs spanning distant domains (e.g., biology + finance, music theory + civil engineering) are the highest-risk for semantic collapse. Apply stricter scrutiny: require the relationship to be attested in domain literature, not just linguistically plausible. A pair like (mitochondria, stock_market) might yield "both involve energy exchange" — this is textbook semantic collapse.

5. **Evaluate confidence calibration.** If the system provides confidence scores, test for the CORE calibration asymmetry: are confidence levels for related and unrelated pairs suspiciously similar (within 5%)? If so, the confidence signal is unreliable and should not be used for filtering. Report the Expected Calibration Error gap.

6. **Compute the semantic collapse rate.** Count the number of unrelated pairs incorrectly assigned a relation, divided by total unrelated pairs. A rate above 20% indicates systemic problems. The CORE benchmark mean is 37.6% — use this as a reference point for severity assessment.

7. **Generate adversarial unrelated pairs for stress-testing.** For each legitimate relation in the system, construct a matched unrelated pair by substituting one concept with a same-domain-distant or cross-domain concept. These become test cases: if the system assigns the same relation type to the adversarial pair, it has failed the unrelatedness test.

8. **Produce the audit report.** For each evaluated pair, output: the original claim, the CORE relation category, the unrelatedness test result (pass/fail/ambiguous), the semantic collapse risk level (low/medium/high/critical), and a remediation recommendation if the pair fails.

9. **Recommend structural safeguards.** Based on the audit, propose concrete changes: adding explicit "unrelated" categories to classification schemas, implementing dual-path verification in pipelines (one path argues for the relation, one argues against), or adding adversarial unrelated pairs to any training or evaluation dataset.

## Concrete Examples

**Example 1: Auditing a Knowledge Graph for Spurious Relations**

User: "I built a knowledge graph from LLM-extracted triples. Can you check if the relationships make sense?"

```
Input triples:
  (Python, is_used_for, web_development)
  (TCP, enables, reliable_data_transfer)
  (photosynthesis, relates_to, blockchain_consensus)
  (gradient_descent, optimizes, neural_network_loss)
  (CRISPR, similar_to, version_control)
  (HTTP, communicates_via, TCP)
```

Approach:
1. Classify each triple into CORE relation types
2. Apply unrelatedness null hypothesis to each pair
3. Flag cross-domain pairs for elevated scrutiny

```
Audit Results:

| Triple | CORE Type | Unrelatedness Test | Collapse Risk |
|--------|-----------|-------------------|---------------|
| (Python, is_used_for, web_development) | Functional | PASS - attested relation | Low |
| (TCP, enables, reliable_data_transfer) | Causal/Functional | PASS - definitional | Low |
| (photosynthesis, relates_to, blockchain_consensus) | ??? | FAIL - cross-domain confabulation | CRITICAL |
| (gradient_descent, optimizes, neural_network_loss) | Functional | PASS - attested relation | Low |
| (CRISPR, similar_to, version_control) | Equivalence? | FAIL - superficial analogy | HIGH |
| (HTTP, communicates_via, TCP) | Functional/Dependency | PASS - protocol stack relation | Low |

Semantic Collapse Rate: 2/6 = 33.3% (above 20% threshold)

Flagged pairs:
- (photosynthesis, blockchain_consensus): CRITICAL. These concepts share no
  meaningful ontological relation. Any claimed connection ("both involve
  distributed processes") is a textbook semantic collapse — a linguistically
  plausible but semantically vacuous fabrication.

- (CRISPR, version_control): HIGH. The superficial analogy "both edit
  sequences" conflates biological gene editing with software source control.
  These are unrelated concepts from different domains. Remove or reclassify
  as "unrelated".

Recommendation: Remove the two flagged triples. Add an explicit "unrelated"
edge type to your schema. Re-run LLM extraction with adversarial prompting
that includes "none of the above / no relation" as a valid output.
```

**Example 2: Designing Adversarial Test Cases for a Semantic Similarity API**

User: "I have a semantic similarity endpoint. How do I test it for the kind of failures CORE describes?"

Approach:
1. Take known valid pairs from the system's domain
2. Generate matched adversarial unrelated pairs
3. Structure as a CORE-style evaluation with equal related/unrelated representation

```
Test Suite Design:

Given your API handles software engineering concepts, here are matched pairs:

RELATED (expected similarity > 0.7):
  (React, Vue)           — same-category: frontend frameworks
  (SQL injection, XSS)   — same-category: security vulnerabilities
  (Docker, Kubernetes)    — functional-dependency: containerization stack
  (unit test, assertion)  — part-of: testing components

UNRELATED (expected similarity < 0.3):
  (React, SQL injection)       — cross-category adversarial
  (Docker, assertion)          — cross-category adversarial
  (Vue, Kubernetes)            — superficially co-occurring but unrelated
  (XSS, unit test)             — security vs. testing, no direct relation

EVALUATION PROTOCOL:
1. Run all 8 pairs through the API
2. Check: Do unrelated pairs score below 0.3?
3. Check: Is the confidence gap between related and unrelated pairs > 0.4?
4. If unrelated pairs score > 0.5, your system has semantic collapse
5. If confidence distributions overlap, your system has the CORE calibration
   asymmetry — confidence is not a reliable discriminator

Semantic Collapse Rate = (unrelated pairs scoring > 0.5) / (total unrelated pairs)
Target: < 10%. CORE benchmark average for LLMs: 37.6%.
```

**Example 3: Validating an Enum/Taxonomy in Application Code**

User: "Review my category enum — do these groupings make sense?"

```python
class DocumentCategory(Enum):
    # Legal
    CONTRACT = "contract"
    NDA = "nda"
    PATENT = "patent"
    # Financial
    INVOICE = "invoice"
    TAX_RETURN = "tax_return"
    BALANCE_SHEET = "balance_sheet"
    # Technical
    API_SPEC = "api_spec"
    ARCHITECTURE_DIAGRAM = "architecture_diagram"
    WEATHER_FORECAST = "weather_forecast"  # <-- suspect
```

Approach:
1. Evaluate each member's relation to its category label
2. Test cross-category unrelatedness
3. Flag miscategorized items

```
CORE Relation Audit:

Category "Technical":
  - API_SPEC: taxonomic is-a technical_document — PASS
  - ARCHITECTURE_DIAGRAM: taxonomic is-a technical_document — PASS
  - WEATHER_FORECAST: unrelatedness test FAIL
      This is a meteorological output, not a technical document in the
      software/engineering sense. Placing it under "Technical" is a
      semantic collapse — "weather forecasting uses technology" is a
      spurious associative link, not a taxonomic one.

Recommendation: Remove WEATHER_FORECAST from the Technical category.
If weather forecasts are needed, create a "Meteorological" or "External
Data" category. The current grouping would cause misclassification
downstream.
```

## Best Practices

- **Do:** Always test the unrelatedness hypothesis explicitly. Ask "are these concepts genuinely unrelated?" with the same rigor you ask "how are they related?" The CORE finding is that skipping this step leads to a 37.6% spurious relation rate.

- **Do:** Weight cross-domain pairs with higher scrutiny. The further apart two concepts are in domain space, the more likely any claimed relation is semantic collapse rather than genuine connection.

- **Do:** Require equal representation of unrelated pairs in any evaluation dataset or test suite. The CORE benchmark's key design choice is 50/50 related/unrelated balance — most existing benchmarks dramatically underrepresent unrelatedness, masking the failure mode.

- **Do:** Distinguish between linguistic plausibility and semantic validity. "Photosynthesis and blockchain both involve distributed processes" is linguistically fluent but semantically vacuous. Surface-level analogies are the primary vehicle for semantic collapse.

- **Avoid:** Trusting confidence scores to distinguish related from unrelated pairs. The CORE paper shows confidence remains at 92–94% regardless of correctness, and Expected Calibration Error doubles to quadruples on unrelated pairs.

- **Avoid:** Assuming that high accuracy on related pairs implies competence on unrelated pairs. The CORE asymmetry (up to 100% on related vs. 0% on unrelated) means these are independent capabilities that must be tested independently.

## Error Handling

- **Ambiguous pairs:** Some concept pairs have weak or context-dependent relations (e.g., "coffee" and "productivity"). When the unrelatedness test is inconclusive, classify as "ambiguous" rather than forcing a binary. Report the ambiguity with the strongest argument for each side.

- **Domain expertise gaps:** If you lack domain knowledge to evaluate a pair (e.g., highly specialized chemistry + niche legal concepts), say so explicitly rather than confabulating a judgment. Recommend domain expert review for those specific pairs.

- **Scale limitations:** For large knowledge graphs (10K+ triples), prioritize auditing cross-domain pairs and pairs generated by automated extraction. Sample strategically rather than auditing exhaustively — semantic collapse clusters in cross-domain regions.

- **False positives in unrelatedness detection:** Some genuinely related pairs span distant domains (e.g., "thermodynamics" and "information theory" share deep mathematical connections). When flagging a pair, always provide the reasoning so the user can override. The goal is surfacing candidates for review, not automated deletion.

## Limitations

- This approach is most effective for **categorical and ontological relationships** (is-a, part-of, causes, etc.). It is less applicable to graded similarity judgments where "somewhat related" is a valid answer.
- The CORE benchmark focuses on **binary classification** (related vs. unrelated). Real-world ontologies often need fine-grained relation typing, which this framework supports but doesn't fully resolve.
- Semantic collapse detection requires **domain context**. Two concepts that appear unrelated in general knowledge may be legitimately connected in a specialized field. Always consider the target domain.
- The 24 relation types from CORE provide good coverage but are **not exhaustive**. Highly domain-specific relation types (e.g., "catalyzes" in biochemistry) may need to be added for specialized applications.
- This skill addresses **evaluation and auditing**, not training. It can identify where systems fail at unrelatedness reasoning but cannot directly fix the underlying model behavior.

## Reference

**Paper:** Dwivedi, S., Ghosh, S., Dwivedi, S., Kumari, N., & Thakur, A. (2026). CORE: Comprehensive Ontological Relation Evaluation for Large Language Models. arXiv:2602.06446v1. https://arxiv.org/abs/2602.06446v1

**Key takeaway:** Look for the semantic collapse rate metric (Section on unrelated pair evaluation), the 24 relation type taxonomy, and the confidence calibration analysis showing that LLM confidence is uninformative for distinguishing genuine from spurious relations.