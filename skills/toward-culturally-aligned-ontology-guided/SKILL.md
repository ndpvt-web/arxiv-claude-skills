---
name: "toward-culturally-aligned-ontology-guided"
description: >
  Build culturally aligned AI systems using the OG-MAR (Ontology-Guided Multi-Agent Reasoning)
  framework. Constructs value ontologies from survey data, instantiates demographically grounded
  persona agents, and synthesizes culturally consistent outputs via a judgment agent.
  Trigger phrases: "culturally aligned generation", "value-aware multi-agent system",
  "cultural ontology reasoning", "demographic persona agents", "OG-MAR framework",
  "ontology-guided cultural alignment"
---

# Ontology-Guided Multi-Agent Reasoning for Cultural Alignment (OG-MAR)

This skill enables Claude to implement the OG-MAR framework from Seo et al. (2026): a multi-agent architecture that produces culturally aligned LLM outputs by grounding reasoning in structured value ontologies and demographically similar persona agents. Instead of treating cultural values as flat, independent signals, OG-MAR organizes them into an ontology of 76 fine-grained value categories with 150 curated relational triples, then retrieves relevant ontology subgraphs and real demographic profiles to instantiate persona agents whose outputs a judgment agent synthesizes with formal consistency constraints.

## When to Use

- When building a system that must produce culturally sensitive outputs for users across different world regions (e.g., content moderation, survey simulation, localized recommendations)
- When the user asks to simulate how people from specific demographics would respond to value-laden questions
- When implementing a multi-agent pipeline where each agent represents a distinct cultural perspective grounded in real survey data
- When constructing a value ontology from structured survey responses (World Values Survey, regional social surveys, or similar)
- When the user needs culturally aware NLG that goes beyond simple prompt-steering with country names or stereotypes
- When designing evaluation frameworks for measuring cultural alignment of LLM outputs

## Key Technique

**Ontology-Guided Structure.** OG-MAR's core insight is that cultural values are not independent dimensions -- they form a relational graph. The framework defines a fixed taxonomy of 12 top-level value domains (e.g., Social Values, Political Values, Religious Values) subdivided into 76 fine-grained categories. Domain experts write *competency questions* that probe interactions between pairs of parent classes (e.g., "How does religious belief influence attitudes toward gender equality?"). LLMs conditioned on real respondent profiles generate candidate relational triples `(category_a, relation, category_b)`, which are consolidated and expert-reviewed into a final ontology of ~150 curated object properties. This ontology is built once and reused at inference.

**Demographically Grounded Persona Agents.** At inference time, a query is classified into relevant value categories using a fine-tuned classifier, and the ontology is queried for relevant relational triples. Simultaneously, a dense retrieval step (E5 embeddings over demographic descriptors) finds the K most demographically similar respondents from a pre-summarized WVS corpus. Each respondent's raw survey answers have been pre-processed into per-category value summaries by a Summarization Agent. Each retrieved individual becomes a *value-persona agent* that receives: the relevant ontology triples, their own value summaries restricted to those triples, and their demographic attributes. Each persona generates both an answer prediction and a natural-language reasoning trace.

**Judgment Agent Synthesis.** A final judgment agent receives all persona outputs but *not* the raw ontology or profiles -- grounding transmits solely through the persona reasoning traces. The judge scores each response for evidence quality and ontology compliance, applies conditional voting only when top options are within a small margin, and breaks remaining ties by demographic proximity to the target. This produces a single culturally aligned output with an interpretable audit trail.

## Step-by-Step Workflow

1. **Define the value taxonomy.** Establish a fixed set of top-level value domains and fine-grained categories relevant to your application. For general cultural alignment, adopt the 12-domain / 76-category WVS-derived taxonomy. For domain-specific applications (e.g., healthcare ethics, content policy), define custom categories but keep the taxonomy frozen once defined -- no classes should be added, merged, or split during ontology construction.

2. **Author competency questions.** For each pair of top-level domains that may interact, write 1-3 competency questions probing their relationship. Format: "How does [Category A] relate to or influence attitudes toward [Category B]?" These questions drive the ontology's relational structure.

3. **Elicit relational triples under cultural conditioning.** For each competency question, prompt an LLM conditioned on value profiles from diverse respondents (aim for 20+ per world region, 6+ regions). Constrain outputs to: (a) use only predefined taxonomy categories, (b) introduce no new classes, (c) articulate only inter-class relations. Collect candidate triples `(category_a, relation_description, category_b)`.

4. **Consolidate and validate the ontology.** Deduplicate candidate triples, merge near-identical relations, and have human reviewers assess cultural plausibility. Remove spurious or inconsistent relations. Target ~150 validated object properties for a full cultural ontology. Serialize the final ontology as JSON-LD, OWL, or a simple triple store.

5. **Pre-summarize respondent profiles.** For each respondent in your survey corpus, run a Summarization Agent that produces a concise, category-specific synopsis per taxonomy class: `summary_i_j = Summarize(raw_responses_i | category_j)`. Store the full value profile `V_i = {summary_i_1, ..., summary_i_76}` alongside demographic metadata.

6. **Classify incoming queries.** When a culturally sensitive query arrives, classify it into the top-k value domains and top-p fine-grained categories. Use a fine-tuned classifier (e.g., DeBERTa-v2-xxlarge on WVS question-to-category mappings) or, for simpler setups, LLM-based zero-shot classification with the taxonomy as the label set.

7. **Retrieve ontology triples.** Compute embedding similarity between the query and each taxonomy category. Score each ontology triple by `max(sim(query, category_a), sim(query, category_b))`. Retrieve the top-M triples (typically 3-9) constrained to the categories identified in step 6.

8. **Retrieve demographically similar respondents.** Encode the target demographic descriptor (e.g., "35-year-old female, South Korea, university-educated, urban") with a dense encoder (E5-base). Rank all respondents by demographic embedding similarity. Select top-K individuals (default K=5). Extract their value profiles restricted to the categories referenced in the retrieved ontology triples.

9. **Instantiate and run persona agents.** For each of the K retrieved individuals, instantiate a value-persona agent with the prompt: `[Ontology triples] + [Individual's relevant value summaries] + [Individual's demographics] + [Query]`. Each agent produces: (a) an answer prediction, and (b) a reasoning trace explaining how the ontology relations and their value profile informed the answer.

10. **Run the judgment agent.** Feed all persona outputs (answers + reasoning traces) to the judgment agent. The judge must: (a) score each persona's reasoning for evidence grounding and ontology compliance, (b) aggregate scores per response option, (c) consult vote tallies only if top options are within a small margin, (d) break ties by demographic proximity to the target. Output the final answer with a synthesis rationale.

## Concrete Examples

**Example 1: Building a Cultural Alignment API for Survey Simulation**

```
User: I need to build a service that predicts how a 45-year-old male
from rural Egypt with secondary education would answer questions about
gender roles and religious authority. We have WVS Wave 7 data.

Approach:
1. Extract the relevant taxonomy subset: "Gender Equality" (Social Values)
   and "Religion & Secularism" (Religious Values) -- about 8-10 fine-grained
   categories like "Justifiability of divorce", "Importance of religion in
   politics", "Attitudes toward women working".

2. Author competency questions:
   - "How does importance of religious faith relate to views on gender
     equality in family decisions?"
   - "How do attitudes toward secular governance relate to views on
     women's economic participation?"

3. Elicit triples from LLM conditioned on 20 Egyptian respondent profiles:
   Triple example: ("Religious_Faith_Importance", "constrains_acceptance_of",
   "Women_Economic_Independence")

4. Pre-summarize WVS Wave 7 Egyptian respondents' profiles. For each
   respondent, generate category-specific summaries:
   summary_142_gender_eq = "Believes wife should obey husband but supports
   daughters' education; views divorce as rarely justifiable."

5. At query time ("Should women be allowed to work outside the home?"):
   - Classify -> Gender Equality, Religion & Secularism
   - Retrieve 6 ontology triples connecting these categories
   - Retrieve 5 demographically similar respondents (rural Egyptian males,
     40-50, secondary education)
   - Run 5 persona agents, each grounded in their value summaries + triples
   - Judgment agent synthesizes: "Disagree (conditional)" with reasoning
     trace citing religious-authority-to-gender-role ontology path

Output structure:
{
  "prediction": "Disagree",
  "confidence": 0.78,
  "reasoning_trace": "4/5 persona agents predicted Disagree. Ontology path
    Religious_Faith_Importance -> constrains_acceptance_of ->
    Women_Economic_Independence was cited by all agents. Persona #3
    (closest demographic match, cosine=0.94) noted that support for
    daughters' education coexists with opposition to wives working,
    consistent with WVS conditional-modernization pattern.",
  "dissenting_view": "Persona #2 predicted Agree, citing younger-cohort
    shift in urban-adjacent rural areas."
}
```

**Example 2: Implementing the Ontology Construction Pipeline**

```
User: Help me implement the ontology construction step. I have a CSV
of 500 WVS respondents with demographic columns and 50 survey items.

Approach:
1. Define taxonomy mapping -- map each of the 50 survey items to the
   76-category taxonomy. Example:
   {
     "Q106": {"category": "Justifiability_Homosexuality",
              "domain": "Social_Values"},
     "Q199": {"category": "Confidence_Armed_Forces",
              "domain": "Political_Values"}
   }

2. Select 120 respondents (20 per region) stratified by world region.

3. For each competency question, build the elicitation prompt:

   PROMPT TEMPLATE:
   """You are reasoning about cultural values. Given the value profile
   of a {region} respondent:
   {value_profile_summary}

   Competency Question: {cq_text}

   Generate relational statements between the following categories ONLY:
   {category_a}, {category_b}
   Rules:
   - Use only categories from the provided taxonomy
   - Do not introduce new categories
   - Describe how category_a relates to category_b
   Format: (category_a, relation_description, category_b)"""

4. Collect candidate triples, deduplicate by semantic similarity
   (threshold cosine > 0.85), and output for human review:

   candidates = [
     ("Religious_Faith_Importance", "reinforces adherence to",
      "Traditional_Family_Values", score=0.91, sources=["EG","PK","NG"]),
     ("Trust_In_Government", "moderates acceptance of",
      "Income_Redistribution", score=0.87, sources=["SE","DE","US"]),
   ]

5. After expert review, serialize the validated ontology:
   ontology.json with schema:
   { "triples": [{"s": str, "p": str, "o": str, "regions": [str]}],
     "taxonomy": {"domains": [...], "categories": [...]} }

Output: A validated ontology file with ~150 triples ready for retrieval.
```

**Example 3: Multi-Agent Reasoning for Content Localization**

```
User: I'm localizing a health campaign about vaccination for both
rural India and urban Germany. How would OG-MAR help me generate
culturally appropriate messaging?

Approach:
1. Identify relevant taxonomy categories: "Trust_In_Science",
   "Religious_Authority", "Community_Collectivism",
   "Government_Trust", "Health_Autonomy".

2. Retrieve ontology triples for each target demographic:
   - Rural India: "Community_Collectivism" -> "amplifies influence of"
     -> "Religious_Authority" on health decisions
   - Urban Germany: "Trust_In_Science" -> "overrides" ->
     "Community_Pressure" in medical decisions

3. Instantiate persona agents:
   - India: 5 agents from rural Indian respondents (mixed religion,
     25-55 age range, varying education)
   - Germany: 5 agents from urban German respondents

4. India persona outputs converge on:
   "Frame vaccination as community protection and family duty.
   Include endorsement from respected religious/community leaders.
   Avoid framing that implies distrust of traditional medicine."

5. Germany persona outputs converge on:
   "Lead with clinical evidence and efficacy data.
   Emphasize individual health autonomy and informed choice.
   Avoid paternalistic or collectivist framing."

6. Judgment agent synthesizes per-region messaging guidelines with
   ontology-grounded justifications for each recommendation.

Output: Two culturally grounded messaging briefs with reasoning traces
linking each recommendation to specific ontology paths and persona evidence.
```

## Best Practices

- **Do:** Keep the taxonomy frozen once defined. OG-MAR's consistency guarantees depend on a stable category set -- adding categories mid-deployment invalidates existing ontology triples and pre-computed summaries.
- **Do:** Use real survey data (WVS or equivalent) for profile summarization. Synthetic or stereotyped profiles defeat the purpose of demographic grounding and reproduce the biases the framework is designed to avoid.
- **Do:** Include dissenting persona outputs in the judgment agent's synthesis. Cultural alignment is not unanimity -- surfacing minority viewpoints within a demographic group improves transparency and prevents false confidence.
- **Do:** Set temperature to 0 for all agent calls during evaluation to ensure reproducibility. For production use, low temperature (0.1-0.3) can add natural variation.
- **Avoid:** Letting the judgment agent access the raw ontology or value profiles directly. The design principle is that grounding transmits through persona reasoning traces, which forces each persona to explicitly justify its reasoning rather than the judge cherry-picking evidence.
- **Avoid:** Using OG-MAR for individual-level prediction. The framework aligns with aggregate cultural tendencies of demographic groups, not the specific opinions of any single person. Never present outputs as "what person X thinks."

## Error Handling

- **Category classification failure.** If the query doesn't map cleanly to taxonomy categories (e.g., a highly novel or technical topic), fall back to the top-3 most semantically similar categories by embedding distance and flag the output as low-confidence. Log these cases for taxonomy review.
- **Insufficient demographic matches.** If fewer than 3 respondents match the target demographic at cosine similarity > 0.7, broaden the search to adjacent demographics (e.g., expand age range, include neighboring regions) and annotate the output with the demographic drift.
- **Persona agent disagreement above threshold.** If persona agents produce no majority answer (e.g., 2-2-1 split across 5 agents), the judgment agent should report the distribution rather than forcing a single answer. This is signal, not noise -- it indicates genuine cultural heterogeneity within the demographic group.
- **Ontology triple sparsity.** If retrieved triples don't connect the query's categories (disconnected subgraph), the persona agents lack relational grounding. Fall back to direct value-profile conditioning without ontology triples and flag the output as ontology-ungounded.

## Limitations

- **Data dependency.** OG-MAR requires a substantial pre-processed survey corpus (WVS has ~94,000 respondents across 7 waves). Without real survey data, the framework degenerates into stereotype-driven persona simulation.
- **Taxonomy rigidity.** The fixed 76-category taxonomy may not cover emerging cultural dimensions (e.g., AI ethics attitudes, digital privacy norms). Extending the taxonomy requires re-running ontology construction and profile summarization.
- **Western-survey bias.** The WVS itself has known sampling biases. OG-MAR inherits these -- underrepresented populations in the survey will have poor demographic retrieval coverage.
- **Computational cost.** Running K persona agents + 1 judgment agent per query multiplies inference cost by K+1. For K=5, this is 6x the cost of a single LLM call. Batch processing or caching persona outputs for repeated demographic profiles can mitigate this.
- **Not for individual prediction.** The framework captures group-level cultural tendencies. It should never be used to predict or simulate a specific individual's beliefs or responses.

## Reference

Seo, W., Choi, W., Koh, J., Lee, J., & An, H. (2026). *Toward Culturally Aligned LLMs through Ontology-Guided Multi-Agent Reasoning.* arXiv:2601.21700v2. Key sections: Section 3 (ontology construction with competency questions), Section 4 (multi-agent inference pipeline), Section 5 (judgment agent adjudication protocol), Appendix A (full 76-category taxonomy), Appendix B (prompt templates).