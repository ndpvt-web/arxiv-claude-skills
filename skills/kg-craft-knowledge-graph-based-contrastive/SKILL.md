---
name: "kg-craft-knowledge-graph-based-contrastive"
description: >
  Fact-check claims using knowledge graph-based contrastive reasoning. Constructs a KG from claims
  and evidence sources, generates contrastive questions ("Why X rather than Y?") grounded in the KG
  structure, distills evidence through targeted Q&A, and synthesizes a summary for veracity assessment.
  Trigger phrases: "fact-check this claim", "verify this statement against sources",
  "build a knowledge graph for claim verification", "contrastive reasoning for fact-checking",
  "check if this claim is true using these reports", "KG-CRAFT pipeline"
---

# KG-CRAFT: Knowledge Graph-Based Contrastive Reasoning for Fact-Checking

This skill enables Claude to fact-check claims by building a knowledge graph from claims and their associated evidence documents, then generating targeted contrastive questions that probe *why* a claim states one thing rather than plausible alternatives. The contrastive Q&A is distilled into a focused summary that dramatically improves veracity classification — even outperforming naive use of much larger LLMs. This implements the KG-CRAFT pipeline from Lourenço et al. (EACL 2026).

## When to Use

- When a user provides a claim and one or more source documents (articles, reports, press releases) and asks whether the claim is true
- When building an automated fact-checking pipeline that needs structured evidence reasoning
- When verifying news headlines, political statements, or social media claims against reference texts
- When the user wants to understand *why* a claim is misleading, not just whether it is true or false
- When processing datasets like LIAR, ClaimBuster, or similar claim-verification corpora
- When the user asks to "build a knowledge graph" from claims and reports for downstream reasoning

## Key Technique

**Knowledge Graph Construction**: KG-CRAFT begins by extracting a structured knowledge graph G = (E, R, T, C) from the claim and its associated reports using phased few-shot prompting. First, extract entities (E) from the text. Then classify each entity into a semantic category (C) — such as Person, Organization, Location, Date, Quantity. Finally, extract relations (R) connecting entity pairs as triples (head, relation, tail). This phased approach prevents error cascading and produces a clean, typed graph.

**Contrastive Question Generation**: The core innovation is generating questions that *contrast* the claim against plausible alternatives derived from the KG structure. For each triple in the claim subgraph (e.g., "Biden signed the Infrastructure Act"), the method finds alternative entities of the same semantic type from the report subgraph (e.g., other legislation or other signatories). It then formulates contrastive questions like "Why does the claim state Biden signed the Infrastructure Act rather than the CHIPS Act?" These questions are ranked using Maximal Marginal Relevance (MMR) to balance relevance to the claim and diversity across question topics. The top K questions (typically K=5) are selected.

**Evidence Distillation and Veracity Assessment**: Each contrastive question is answered by analyzing the source reports directly, producing evidence-grounded answers. These Q&A pairs are then summarized into a single concise paragraph that highlights the contrasts between what the claim states and what the evidence supports. This summary — not the raw reports — is used as the sole evidence for final veracity classification. This distillation step compresses noisy, lengthy reports into a focused evidence brief that LLMs can reason over effectively.

## Step-by-Step Workflow

1. **Parse the claim and evidence sources.** Separate the target claim from its associated reports/documents. If the user provides URLs, fetch and extract the text content. Structure the inputs as `{claim: string, reports: string[]}`.

2. **Extract entities from the claim.** Use few-shot prompting to identify all named entities in the claim. Output a list of `{entity: string, text_span: string}` objects.

3. **Extract entities from each report.** Apply the same entity extraction to every report document. Merge into a unified entity set, deduplicating by name normalization.

4. **Classify entities into semantic categories.** For each entity, assign a category (Person, Organization, Location, Date, Quantity, Event, Legislation, etc.). This typing is critical — it enables finding plausible alternatives of the same type.

5. **Extract relation triples.** Identify (head_entity, relation, tail_entity) triples from both the claim and the reports. Store these as the KG structure. Separate claim-triples from report-triples.

6. **Generate contrastive questions.** For each claim-triple, find alternative entities from report-triples that share the same semantic category as the head or tail. Formulate questions following the pattern: "Why does the claim state [original entity/relation] rather than [alternative]?" Generate multiple candidate questions per triple.

7. **Rank questions with MMR.** Score each question by its semantic similarity to the claim (relevance) minus its maximum similarity to already-selected questions (diversity penalty). Select the top K=5 questions.

8. **Answer contrastive questions against reports.** For each selected question, prompt an LLM to answer it using only the report text as evidence. Each answer should cite specific passages or facts from the reports.

9. **Synthesize a contrastive summary.** Combine all Q&A pairs into a single concise paragraph that relates the contrastive findings. This summary should highlight where the claim aligns with and diverges from the evidence.

10. **Assess veracity.** Prompt for a final classification using only the original claim and the contrastive summary as input. Output a veracity label (e.g., true, mostly-true, half-true, barely-true, false, pants-on-fire for 6-class; or true, half-true, false for 3-class) with a brief justification grounded in the summary.

## Concrete Examples

**Example 1: Political Claim Verification**

```
User: Fact-check this claim against these two reports:

Claim: "The US unemployment rate hit a 50-year low of 3.5% in September 2019."

Report 1: "Bureau of Labor Statistics data shows the unemployment rate fell to 3.5%
in September 2019, matching the rate last seen in December 1969 — 50 years prior.
The rate had been 3.7% in August 2019."

Report 2: "While 3.5% was reached in September 2019, some economists noted that
the December 1969 rate was 3.5% as well, making the claim of a 'low' technically
a tie rather than a new record. The U-6 underemployment rate remained at 6.9%."
```

Approach:
1. Extract entities from claim: {US, unemployment rate, 50-year low, 3.5%, September 2019}
2. Extract entities from reports: {Bureau of Labor Statistics, 3.5%, September 2019,
   December 1969, 50 years, 3.7%, August 2019, U-6, 6.9%}
3. Classify: Dates(September 2019, December 1969, August 2019),
   Quantities(3.5%, 3.7%, 6.9%, 50-year), Organizations(Bureau of Labor Statistics)
4. Claim triples: (unemployment rate, hit, 3.5%), (3.5%, was, 50-year low),
   (event, occurred_in, September 2019)
5. Contrastive questions generated:
   - "Why does the claim describe 3.5% as a '50-year low' rather than a '50-year tie'?"
   - "Why does the claim reference the unemployment rate (U-3) rather than the
     underemployment rate (U-6) at 6.9%?"
   - "Why September 2019 rather than December 1969 as the reference point?"
6. Answers distilled from reports into summary:
   "The 3.5% rate in September 2019 matched — but did not beat — the December 1969
   rate, making it a tie rather than a new low. The claim uses the narrower U-3 measure;
   the broader U-6 underemployment rate was 6.9%. BLS data confirms the 3.5% figure."

Output:
Verdict: MOSTLY-TRUE
Justification: The 3.5% figure and September 2019 date are confirmed by BLS data.
However, describing it as a "50-year low" is slightly misleading — it tied the
December 1969 rate rather than breaking it. The core statistic is accurate but the
framing overstates the achievement.

**Example 2: Building a Fact-Checking Pipeline in Code**

```
User: Help me build a Python pipeline that implements KG-CRAFT for claim verification
using an LLM API.
```

Approach:
1. Create a structured pipeline with these modules:
   - `kg_extraction.py` — entity extraction, classification, and relation extraction
   - `contrastive_questions.py` — question generation and MMR ranking
   - `evidence_distillation.py` — Q&A against reports and summarization
   - `veracity_classifier.py` — final claim assessment
2. Implement the KG extraction with phased prompts (entities first, then types, then relations)
3. Implement MMR ranking with cosine similarity using sentence embeddings
4. Chain the pipeline: claim+reports → KG → questions → answers → summary → verdict

Output structure:
```python
class KGCraftPipeline:
    def __init__(self, llm_client, embedding_model, k=5):
        self.llm = llm_client
        self.embedder = embedding_model
        self.k = k  # number of contrastive questions

    def extract_kg(self, claim: str, reports: list[str]) -> KnowledgeGraph:
        """Phase 1: entity extraction, Phase 2: classification, Phase 3: relations"""

    def generate_contrastive_questions(self, kg: KnowledgeGraph) -> list[str]:
        """Generate and MMR-rank contrastive questions from claim vs report triples"""

    def distill_evidence(self, questions: list[str], reports: list[str]) -> str:
        """Answer questions against reports and synthesize into summary"""

    def classify_veracity(self, claim: str, summary: str) -> VeracityResult:
        """Final veracity assessment using only claim + contrastive summary"""

    def verify(self, claim: str, reports: list[str]) -> VeracityResult:
        """End-to-end pipeline"""
        kg = self.extract_kg(claim, reports)
        questions = self.generate_contrastive_questions(kg)
        summary = self.distill_evidence(questions, reports)
        return self.classify_veracity(claim, summary)
```

**Example 3: Investigating a Specific Claim Interactively**

```
User: I have a claim that "Tesla sold 500,000 vehicles in 2020" and three news articles.
Walk me through the KG-CRAFT reasoning step by step.
```

Approach:
1. Show the extracted KG: entities (Tesla, 500000, vehicles, 2020), categories, and triples
2. Show report KG: perhaps reports mention 499,550 deliveries (not sales), include
   other automakers' figures, and distinguish deliveries from production
3. Present the generated contrastive questions:
   - "Why does the claim say 'sold' rather than 'delivered'?"
   - "Why 500,000 rather than 499,550?"
   - "Why 'vehicles' rather than specifically 'Model 3 and Model Y'?"
4. Show each answer drawn from the report evidence
5. Present the synthesized summary and final verdict

Output:
KG-CRAFT Summary: "Tesla's 2020 figure was 499,550 deliveries — not sales — falling
just short of the 500,000 target. The claim rounds up and conflates deliveries with
sales, which are distinct metrics in automotive reporting."

Verdict: MOSTLY-FALSE — The figure is approximately correct but inflated by ~450 units,
and the claim mischaracterizes deliveries as sales.
```

## Best Practices

- **Do:** Extract the KG in phases (entities → types → relations). Phased extraction reduces error cascading compared to extracting everything in a single prompt.
- **Do:** Use K=5 contrastive questions as the default. The paper shows performance plateaus beyond 5, and fewer questions miss important contrasts.
- **Do:** Apply MMR ranking to ensure question diversity. Without it, questions cluster around the most obvious entity, missing subtler mismatches.
- **Do:** Use the contrastive summary as the *sole* evidence input for final classification. Passing raw reports dilutes the focused reasoning that makes this approach effective.
- **Avoid:** Generating contrastive questions with a generic LLM prompt instead of grounding them in the KG structure. The paper shows KG-grounded questions achieve F1=73.87% vs. 29.68% for LLM-generated questions on LIAR-RAW — a 44-point gap.
- **Avoid:** Skipping entity type classification. Without semantic types, the system cannot identify plausible alternative entities, and the contrastive questions become incoherent.

## Error Handling

- **Sparse KG from short claims**: If the claim yields fewer than 2 triples, supplement with co-reference resolution or expand entity extraction to noun phrases. Fall back to generating questions directly from entity-type contrasts rather than triple contrasts.
- **No alternative entities found**: When report entities don't share semantic types with claim entities, broaden the category hierarchy (e.g., treat "City" and "Country" both as "Location"). If still empty, generate "What evidence supports or contradicts [claim triple]?" as a non-contrastive fallback.
- **LLM refuses to classify**: Some LLMs resist making veracity judgments. Frame the final prompt as evidence-based summarization ("Based solely on this summary, which label best fits?") rather than opinion ("Is this true?").
- **Reports are irrelevant to the claim**: If contrastive Q&A produces answers like "the reports do not address this," flag the claim as UNVERIFIABLE rather than forcing a classification.
- **Entity extraction hallucination**: Validate extracted entities against the source text with string matching. Discard any entity not found as a substring (or close fuzzy match) in the original text.

## Limitations

- Requires associated evidence documents/reports for each claim. This method cannot verify claims without reference material — it is not a retrieval system.
- KG quality depends heavily on the LLM's entity and relation extraction capability. Highly technical or domain-specific claims (e.g., chemistry, medicine) may produce noisy graphs.
- The MMR ranking step requires an embedding model for computing semantic similarity. In environments without embeddings, a simpler diversity heuristic (e.g., one question per unique entity) can substitute, though with reduced performance.
- Contrastive reasoning works best when plausible alternatives exist. For purely novel claims with no comparable entities in the reports, the contrastive framing provides less benefit.
- The pipeline involves multiple sequential LLM calls (KG extraction, question answering, summarization, classification), which increases latency and cost compared to single-pass verification.

## Reference

**Paper:** [KG-CRAFT: Knowledge Graph-based Contrastive Reasoning with LLMs for Enhancing Automated Fact-checking](https://arxiv.org/abs/2601.19447v1) — Lourenço et al., EACL 2026. Focus on Algorithm 1 (contrastive question formulation), the MMR ranking mechanism, and Appendices D.1-D.5 for the exact prompts used at each pipeline stage.