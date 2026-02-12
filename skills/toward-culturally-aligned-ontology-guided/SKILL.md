---
name: "toward-culturally-aligned-ontology-guided"
description: "Ontology-guided multi-agent reasoning for culturally aligned LLM outputs. Use when building systems that must respect cultural values, when designing multi-agent pipelines with demographic grounding, or when implementing value-aware decision making. Triggers: 'culturally aligned generation', 'ontology-guided agents', 'value-persona reasoning', 'demographic-aware LLM', 'cultural sensitivity pipeline', 'WVS-style value alignment'"
---

# Ontology-Guided Multi-Agent Reasoning for Cultural Alignment (OG-MAR)

This skill enables Claude to build multi-agent reasoning systems where each agent is grounded in a structured cultural ontology and real demographic value profiles. Based on the OG-MAR framework, the technique constructs a hierarchical ontology of cultural values (organized as domain-to-category classes with relational triples), retrieves demographically similar value profiles, instantiates multiple value-persona agents conditioned on those profiles, and synthesizes their outputs through a judgment agent that enforces ontology consistency. This replaces unstructured "be culturally sensitive" prompting with a principled, traceable pipeline.

## When to Use

- When the user is building an application that must produce culturally appropriate responses for different regions or demographics (e.g., a global survey tool, a localized chatbot, a cross-cultural recommendation engine)
- When designing a multi-agent system where each agent should represent a distinct value perspective grounded in real survey data rather than stereotypes
- When the user needs to implement ontology-based retrieval to constrain LLM reasoning to structured value relationships
- When building evaluation pipelines to measure cultural alignment of LLM outputs against regional benchmarks (e.g., WVS, ESS, CGSS, ISD)
- When the user asks to reduce cultural bias in an AI system by grounding it in demographic data rather than relying on pretraining priors
- When implementing a judgment/adjudication layer that must weigh multiple agent perspectives with explicit scoring criteria

## Key Technique

OG-MAR's core insight is that cultural values are not independent signals -- they form a relational structure. A person's views on gender equality relate to their views on economic redistribution, religious authority, and institutional trust in ways that vary systematically by culture. The framework captures this structure as an ontology: 76 value-category classes organized under 12 domains, connected by 150 relational triples (e.g., `(TrustInInstitutions, influences, EconomicRedistributionPreference)`). These triples are elicited via competency questions -- structured prompts like "How does trust in government relate to attitudes on wealth redistribution in East Asian contexts?" -- then validated by domain experts. This ontology acts as a semantic constraint: agents can only reason along paths the ontology permits, preventing hallucinated cultural generalizations.

At inference time, the pipeline works in three phases. First, a topic classifier identifies which ontology domains and categories are relevant to the query, and retrieves the top-M relational triples whose endpoints fall within those categories. Second, a demographic encoder (E5-base embeddings over demographic descriptions) retrieves the K most similar respondent profiles from a pre-processed World Values Survey dataset, extracting their category-specific value summaries. Third, value-persona agents are instantiated -- each conditioned on the concatenation of retrieved ontology triples, the individual's filtered value profile, and their demographic attributes -- producing an answer plus an explicit reasoning trace. A judgment agent then scores each persona output on reasoning grounding and ontology compliance, applying a near-tie rule and demographic-relevance tiebreaker to produce the final answer.

The key advantage over prompt-based steering ("respond as a Korean woman in her 40s") is that OG-MAR grounds every reasoning step in (a) real survey responses, (b) validated relational constraints, and (c) explicit scoring -- making the cultural alignment traceable, auditable, and robust across different LLM backbones.

## Step-by-Step Workflow

1. **Define the value taxonomy.** Create a hierarchical schema with 10-15 top-level value domains (e.g., `TrustAndGovernance`, `GenderAndFamily`, `ReligionAndSecularism`, `EconomicValues`) each containing 5-8 fine-grained category classes. Store this as a JSON or OWL file that all agents reference.

2. **Build the cultural ontology via competency questions.** For each pair of parent value classes, generate competency questions (e.g., "How does religiosity relate to views on gender roles?"). Prompt an LLM conditioned on diverse regional value profiles to produce candidate relational triples in the form `(class_a, relation_property, class_b)`. Have domain experts validate, edit, or remove spurious triples. Target ~150 validated triples across the taxonomy.

3. **Ingest and summarize respondent profiles.** For each respondent in your survey dataset (WVS or equivalent), separate demographic attributes from value-related responses. Use a summarization agent to produce a natural-language synopsis per category class: `summary = summarize(respondent_responses | category)`. Apply a "no-new-concepts" constraint -- the summary must use only terms from the taxonomy. Store the resulting structured value profiles in a vector database alongside demographic embeddings.

4. **Build the topic classification module.** Fine-tune or prompt a classifier (DeBERTa-scale or LLM-based) that, given an input query, returns the top-k relevant value domains and top-p fine-grained categories. This gates which ontology triples and profile summaries are retrieved downstream.

5. **Implement ontology-consistent triple retrieval.** Score each ontology triple by the relevance of its endpoint nodes to the selected categories: `score(triple) = max(relevance(class_a), relevance(class_b))`. Filter to triples where both endpoints are in the selected category set. Return the top-M triples (typically 3-9; ablation studies show diminishing returns beyond 9).

6. **Retrieve demographically similar profiles.** Encode the target demographic description with a sentence embedding model (E5-base or similar). Rank all respondent profiles by cosine similarity. Select the top-K individuals (default K=5). Extract their value summaries restricted to the categories identified in step 4.

7. **Instantiate value-persona agents.** For each retrieved individual, construct a conditioning context: `context = concat(retrieved_triples, filtered_value_summaries, demographic_attributes)`. Prompt each agent to produce (a) an answer to the query and (b) an explicit reasoning trace explaining how the ontology relations and value profile informed the answer.

8. **Run the judgment agent with constrained adjudication.** The judgment agent receives all persona outputs and scores each on: (i) reasoning grounded in provided evidence, (ii) ontology compliance (does the reasoning follow retrieved triples?). Apply the near-tie rule: if the top two options are within a small margin, consult the vote distribution. If still tied, select the option supported by the most demographically relevant personas.

9. **Return the final answer with a transparency trace.** Output the selected answer, the reasoning path, which ontology triples were activated, and which demographic profiles contributed. This trace enables auditing and debugging of cultural alignment.

10. **Evaluate against regional benchmarks.** Measure binary accuracy (midpoint-thresholded for ordinal items) and Mean Absolute Error against ground-truth survey distributions. Compare per-region to identify where the pipeline helps most (typically regions underrepresented in pretraining data).

## Concrete Examples

**Example 1: Building a culturally grounded survey response predictor**

User: "I need to predict how respondents from different countries would answer questions about trust in government. I have WVS wave 7 data."

Approach:
1. Parse WVS wave 7 CSV. Extract demographic columns (country, age, gender, education, income quintile) and trust-related questions (Q65-Q73).
2. Build value taxonomy with relevant domains: `TrustInInstitutions`, `PoliticalParticipation`, `MediaConsumption`, `EconomicSecurity`. Define 6-8 categories per domain.
3. Generate ontology triples via competency questions:
   ```
   CQ: "How does economic insecurity relate to institutional trust?"
   Triple: (EconomicInsecurity, erodes, GovernmentTrust)
   CQ: "How does media consumption relate to political trust?"
   Triple: (SocialMediaExposure, polarizes, InstitutionalTrust)
   ```
4. Summarize each respondent's trust-related answers into per-category synopses.
5. At prediction time for a new query ("A 35-year-old university-educated woman in Nigeria -- how would she rate trust in parliament?"):
   - Topic classifier selects `TrustInInstitutions`, `PoliticalParticipation`
   - Retrieve 5 triples: `(EconomicInsecurity, erodes, GovernmentTrust)`, etc.
   - Retrieve 5 most similar Nigerian female respondents by embedding similarity
   - Instantiate 5 persona agents, each producing a predicted rating + reasoning
   - Judgment agent scores and selects: "Rating: 2/4 (Not very much). Reasoning: Retrieved profiles consistently show low institutional trust among educated Nigerian women, mediated by perceived corruption (ontology triple: PerceivedCorruption -> undermines -> GovernmentTrust)."

Output:
```json
{
  "prediction": 2,
  "confidence": 0.73,
  "activated_triples": [
    "(PerceivedCorruption, undermines, GovernmentTrust)",
    "(EconomicInsecurity, erodes, GovernmentTrust)"
  ],
  "contributing_profiles": ["NG_F_32_univ", "NG_F_38_univ", "NG_F_35_sec"],
  "reasoning_trace": "3/5 persona agents predicted 'Not very much' citing corruption concerns. Judgment agent confirmed via ontology path: PerceivedCorruption -> undermines -> GovernmentTrust. Demographically closest profile (NG_F_32_univ) also predicted 2."
}
```

**Example 2: Culturally adaptive chatbot for a global health app**

User: "Our health app needs to give advice about mental health that's culturally appropriate. Users span Japan, Brazil, Egypt, and Sweden."

Approach:
1. Define a health-relevant value taxonomy: `MentalHealthStigma`, `FamilyObligation`, `IndividualismVsCollectivism`, `ReligiousFraming`, `InstitutionalHealthTrust`, `EmotionalExpression`.
2. Build ontology triples from regional health-culture literature:
   ```
   (CollectivistOrientation, amplifies, MentalHealthStigma)
   (ReligiousFraming, provides_alternative_to, ClinicalMentalHealthLanguage)
   (HighInstitutionalTrust, increases, TherapyAcceptance)
   ```
3. Ingest WVS profiles filtered to these four countries. Summarize per value category.
4. When a user from Japan asks about managing anxiety:
   - Topic classifier: `MentalHealthStigma`, `EmotionalExpression`, `CollectivistOrientation`
   - Retrieve triples linking stigma to collectivism and emotional expression norms
   - Retrieve 5 Japanese respondent profiles
   - Persona agents generate advice variants; most emphasize indirect language, normalize stress as shared experience, suggest group-compatible coping
   - Judgment agent selects response that aligns with Japanese value profiles while maintaining clinical accuracy

Output:
```
"Many people experience periods of heightened stress, and it can help
to find ways to manage it that fit naturally into daily life. Practices
like structured breathing, maintaining regular routines, and talking
with someone you trust -- whether a friend, family member, or
counselor -- are approaches many people find helpful. If stress begins
affecting your work or relationships, speaking with a professional is
a positive step that many take."
```
(Avoids individualistic "prioritize yourself" framing; uses "many people" normalization consistent with collectivist value profiles.)

**Example 3: Implementing the judgment agent adjudication logic**

User: "Show me how to implement the judgment agent that synthesizes persona outputs."

```python
def judgment_agent_adjudicate(persona_outputs, ontology_triples, demographic_weights):
    """
    Constrained meta-adjudication per OG-MAR.

    Args:
        persona_outputs: list of {answer, reasoning_trace, persona_id, demo_similarity}
        ontology_triples: list of activated (class_a, relation, class_b) triples
        demographic_weights: dict mapping persona_id -> similarity score
    """
    scored = []
    for output in persona_outputs:
        # Score 1: Reasoning grounded in provided evidence
        grounding = score_evidence_grounding(output["reasoning_trace"], ontology_triples)
        # Score 2: Ontology compliance -- does reasoning follow retrieved triples?
        compliance = score_ontology_compliance(output["reasoning_trace"], ontology_triples)
        # Combined evidence score
        evidence_score = 0.6 * grounding + 0.4 * compliance
        scored.append({**output, "evidence_score": evidence_score})

    # Sort by evidence score
    scored.sort(key=lambda x: x["evidence_score"], reverse=True)

    # Near-tie rule: if top two within margin, consult vote distribution
    MARGIN = 0.05
    if len(scored) >= 2 and (scored[0]["evidence_score"] - scored[1]["evidence_score"]) < MARGIN:
        # Count votes per answer option
        vote_counts = Counter(o["answer"] for o in scored)
        top_answer = vote_counts.most_common(1)[0][0]
        # Demographic relevance tiebreaker
        candidates = [o for o in scored if o["answer"] == top_answer]
        best = max(candidates, key=lambda o: demographic_weights[o["persona_id"]])
        return best

    return scored[0]
```

## Best Practices

- **Do:** Ground every value-persona agent in real survey data. The power of OG-MAR comes from demographic specificity, not cultural stereotypes. Use actual WVS or equivalent respondent profiles, not invented personas.
- **Do:** Validate ontology triples with domain experts or at minimum cross-reference with published cultural psychology literature. Spurious triples (e.g., `(Religiosity, causes, Xenophobia)`) poison the entire reasoning chain.
- **Do:** Include the transparency trace in outputs. The ontology-grounded reasoning path is what makes this approach auditable and distinguishes it from black-box cultural steering.
- **Do:** Tune K (number of retrieved profiles) and M (number of triples) via ablation on your target benchmark. The paper finds K=5 and M=3-9 work well, but this varies by domain.
- **Avoid:** Treating the ontology as static. Cultural values shift over time; re-elicit triples when new WVS waves or regional data become available.
- **Avoid:** Using this framework to make deterministic claims about individuals. It predicts distributional tendencies for demographic groups, not individual beliefs. Always surface uncertainty.
- **Avoid:** Skipping the topic classification step. Retrieving all triples and all profiles floods persona agents with irrelevant context and degrades performance.

## Error Handling

- **Insufficient demographic coverage:** If the survey dataset has fewer than 5 respondents matching the target demographic, widen the similarity threshold or fall back to the closest available region. Log a coverage warning in the transparency trace.
- **Ontology gaps:** If the topic classifier selects categories with no connecting triples, the persona agents lack relational constraints. Fall back to demographic-only grounding (no ontology triples) and flag the gap for ontology expansion.
- **Persona agent disagreement beyond adjudication:** If all 5 persona agents produce different answers with similar evidence scores, the judgment agent should return a distribution rather than a single answer, noting high cultural variance on this topic.
- **Stale profiles:** If using WVS data from a wave that is 10+ years old for a rapidly changing region, note the temporal gap in the output and recommend updating the profile database.
- **Topic classifier misclassification:** If the query is ambiguous (e.g., "Is it okay to lie?"), the classifier may select wrong domains. Implement a confidence threshold; below it, expand to adjacent domains or ask the user to disambiguate.

## Limitations

- The framework requires access to structured survey data (WVS or equivalent) with sufficient demographic coverage. It cannot produce grounded cultural alignment for populations absent from the dataset.
- Ontology construction requires human expert validation. Fully automated ontology generation risks encoding LLM biases into the very structure meant to correct them.
- The approach optimizes for distributional alignment with survey populations, not individual accuracy. It should not be used to make predictions about specific people.
- Computational cost scales with K (profiles) x M (triples) agent instantiations. For real-time applications, consider caching frequent demographic-topic combinations.
- Cultural values are contested and evolving. The ontology represents a snapshot, not a ground truth. Regions undergoing rapid social change may be poorly served by older survey waves.
- The framework has been validated primarily on social-survey benchmarks. Generalization to open-ended generation tasks (creative writing, advice giving) requires additional evaluation.

## Reference

- **Paper:** [Toward Culturally Aligned LLMs through Ontology-Guided Multi-Agent Reasoning](https://arxiv.org/abs/2601.21700v2) (Seo et al., 2026)
- **Key takeaway:** Cultural alignment improves when values are structured as relational ontologies and agents are grounded in real demographic profiles, rather than treated as independent prompt-level signals. Focus on Sections 3 (ontology construction), 4 (inference pipeline), and 5 (ablation studies on K and M parameters).