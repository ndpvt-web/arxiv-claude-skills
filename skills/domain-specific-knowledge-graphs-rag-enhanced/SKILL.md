---
name: "domain-specific-knowledge-graphs-rag-enhanced"
description: >
  Build scope-matched knowledge graph RAG pipelines where retrieval precision beats breadth.
  Constructs domain-specific KGs from scientific literature, selects scope-aligned subgraphs
  for retrieval, and injects focused context into LLM prompts — avoiding the accuracy loss
  caused by union-based retrieval. Use when:
  "Build a knowledge graph RAG pipeline for medical questions",
  "Add domain-specific retrieval to my LLM app",
  "My RAG pipeline returns too much irrelevant context",
  "Help me scope my knowledge graph to match my query domain",
  "Design a biomedical QA system with knowledge graphs",
  "Reduce noise in my retrieval-augmented generation system"
---

# Domain-Specific Knowledge Graph RAG with Scope-Matched Retrieval

This skill teaches Claude to design and build RAG systems that use **precision-first, scope-matched knowledge graphs** instead of naive "retrieve everything" approaches. The core insight from Anuyah et al. (2026) is that narrowly scoped KGs aligned to the query domain consistently outperform broad graph unions, which introduce distractors that degrade accuracy — especially for smaller models. Claude applies this to help users construct domain-specific KGs from literature, select the right subgraph per query, and wire it into a RAG pipeline that actually improves LLM output rather than hurting it.

## When to Use

- When the user wants to build a RAG pipeline over structured domain knowledge (medical, legal, scientific)
- When a user's existing RAG system retrieves too much loosely-related context and answers are getting worse
- When designing a biomedical QA system that needs causal reasoning over disease mechanisms, drug interactions, or gene-protein relationships
- When the user asks how to combine multiple knowledge sources without diluting retrieval quality
- When choosing between a single large KG vs. several domain-scoped KGs for retrieval
- When the user needs to decide whether RAG is even worth adding to their LLM workflow (larger models may not benefit)
- When building evaluation probes to test whether KG-RAG is actually helping a specific use case

## Key Technique

**Scope-matched KG-RAG** rejects the assumption that more retrieval context is better. The paper constructs three PubMed-derived knowledge graphs — G1 (Type 2 Diabetes), G2 (Alzheimer's Disease), and G3 (combined AD+T2DM) — and tests them across seven LLMs in six retrieval configurations (No-RAG, individual graphs, pairwise unions, full union). The decisive finding: when the KG's scope aligns with the probe's domain, accuracy improves consistently (e.g., Mixtral macro-F1 jumped from 0.80 to 0.89 with scope-matched G2). When scope is misaligned or graphs are unioned indiscriminately, distractors flood the context and accuracy drops — Llama-3.3-70B fell from 0.96 to 0.71 when given the wrong graph.

The KG construction uses **CoDe-KG**, a pipeline that applies co-reference resolution (resolving synonyms like "T2DM" and "type 2 diabetes"), syntactic decomposition (breaking complex sentences into atomic clauses), and relation extraction with source tagging (paper ID, sentence ID, clause ID for traceability). Edges are limited to causal relations (`causes`, `because`, etc.), and entity names are canonicalized. This produces clean, typed triples like `(insulin_resistance, causes, neuronal_tau_phosphorylation)`.

A critical secondary finding: **model size determines whether RAG helps at all**. Larger models (70B+) often matched or exceeded KG-RAG performance on broad-domain probes using parametric knowledge alone. Smaller/mid-sized models (7B-32B) showed the clearest gains from well-scoped retrieval. Temperature had minimal impact — low temperatures (0.0-0.3) were consistently best. This means the first design decision isn't "what to retrieve" but "does this model even need retrieval for this domain?"

## Step-by-Step Workflow

1. **Define the query domain precisely.** Identify the specific subdomain your system must answer questions about. Narrow beats broad — "Alzheimer's disease mechanisms" is better than "neuroscience." Write down the entity types (diseases, genes, drugs, biomarkers) and relation types (causes, treats, inhibits) you need.

2. **Assess whether RAG is needed for your model.** If using a 70B+ parameter model on well-studied topics, run a No-RAG baseline first. Only add KG-RAG if the baseline shows gaps. For 7B-32B models, scope-matched retrieval is almost always beneficial.

3. **Construct domain-scoped knowledge graphs from literature.** For each subdomain, build a separate KG:
   - Collect domain-specific abstracts/papers (e.g., PubMed queries scoped to your disease/topic)
   - Apply co-reference resolution to normalize entity mentions across documents
   - Decompose complex sentences into atomic clauses before extraction
   - Extract (entity, relation, entity) triples with source provenance tags
   - Canonicalize entity names to prevent duplicate nodes
   - Filter extracted triples: remove vague heads/tails ("it", "this", "the study") via rule-based cleaning

4. **Store each KG separately — do not merge prematurely.** Use a graph database (Neo4j) or in-memory structure (NetworkX, dictionary of adjacency lists). Keep G1, G2, ..., Gn as independent graphs so you can select the right one at query time.

5. **Build a scope-matching retrieval layer.** Given an incoming query, determine which KG(s) are scope-aligned before retrieval:
   - Classify the query's domain using keyword matching, embedding similarity, or a lightweight classifier
   - Retrieve triples only from the matched KG — never default to "all graphs"
   - If a query genuinely spans two domains, use the intersection of the relevant KGs (entities/triples present in both), not their union

6. **Format retrieved triples as structured context.** Inject the retrieved subgraph into the prompt as a clearly delimited block of factual statements. Use a format like:
   ```
   Retrieved domain knowledge (causal relationships):
   - insulin_resistance → causes → neuronal_tau_phosphorylation
   - amyloid_beta_accumulation → causes → synaptic_dysfunction
   ```
   Keep the context focused — 10-30 relevant triples outperform 200 loosely related ones.

7. **Use a zero-shot instruction prompt that constrains the output format.** For structured QA, use:
   ```
   You are answering a domain-specific question. Use ONLY the retrieved
   knowledge below and your training to answer. If the retrieved knowledge
   conflicts with your prior understanding, prefer the retrieved knowledge.
   ```
   For open-ended questions, instruct the model to cite which retrieved triples support its answer.

8. **Set decoding temperature low (0.0-0.3).** Higher temperatures rarely help in domain-specific KG-RAG and often introduce hallucinated reasoning chains.

9. **Build diagnostic probes to evaluate your pipeline.** Create targeted test questions at three difficulty levels:
   - **Single-hop**: One retrieved triple answers the question directly
   - **Multi-hop**: Requires chaining 2-3 triples to reach the answer
   - **Fill-in-the-blank**: Masks a specific entity; tests precise recall
   Generate distractors matched by entity type and frequency to avoid trivial elimination.

10. **Iterate on scope boundaries.** If evaluation shows accuracy drops on certain question types, check whether the retrieved triples are scope-aligned. Common failure modes: directionality flips (A causes B vs. B causes A), chain ordering errors, and negation/exception misreads.

## Concrete Examples

**Example 1: Building a scoped medical QA pipeline**

User: "I'm building a QA system for oncologists about drug interactions in breast cancer treatment. I have 5,000 PubMed abstracts. How should I set up the RAG pipeline?"

Approach:
1. Scope the KG to breast cancer drug interactions specifically — do NOT include general oncology
2. Extract causal triples: `(tamoxifen, inhibits, estrogen_receptor_alpha)`, `(CYP2D6_polymorphism, reduces_efficacy_of, tamoxifen)`
3. Apply co-reference resolution to normalize drug names (e.g., "Nolvadex" → "tamoxifen")
4. Decompose multi-clause sentences before extraction to avoid entangled relations
5. Store as a single scoped KG; add a separate KG only if a distinct subdomain (e.g., radiation therapy interactions) is needed later
6. At query time, retrieve 10-20 triples matching the query entities, inject as structured context

Output structure:
```python
# Knowledge graph construction
from dataclasses import dataclass

@dataclass
class Triple:
    head: str
    relation: str
    tail: str
    source_paper: str
    source_sentence: int

# Scoped retrieval function
def retrieve_scoped(query_entities: list[str], kg: dict[str, list[Triple]], top_k: int = 20) -> list[Triple]:
    """Retrieve triples where head or tail matches query entities."""
    matches = []
    for entity in query_entities:
        canonical = canonicalize(entity)
        if canonical in kg:
            matches.extend(kg[canonical])
    # Rank by relevance: direct matches first, then one-hop neighbors
    return sorted(matches, key=lambda t: relevance_score(t, query_entities))[:top_k]

# Prompt injection
def build_prompt(question: str, triples: list[Triple]) -> str:
    context = "\n".join(f"- {t.head} → {t.relation} → {t.tail}" for t in triples)
    return f"""Retrieved domain knowledge (causal relationships):
{context}

Based on the above knowledge, answer the following question.
Question: {question}
Answer:"""
```

**Example 2: Diagnosing why RAG is hurting accuracy**

User: "I added a knowledge graph to my RAG pipeline but my LLM's accuracy dropped from 92% to 78%. What's going wrong?"

Approach:
1. Check scope alignment: Is the KG domain broader than the query domain? If the KG covers "all of cardiology" but questions are about "atrial fibrillation drug dosing," irrelevant triples are flooding context
2. Check model size: If using a 70B+ model, the parametric prior may already be strong — the KG is adding noise, not signal
3. Check for union pollution: If multiple KGs were merged, distractors from adjacent domains are likely the cause
4. Prescription:
   - Split the KG into subdomain-scoped subgraphs
   - Add a scope classifier before retrieval
   - Limit retrieved triples to 15-25 per query
   - Run A/B: No-RAG vs. scoped-RAG vs. current broad-RAG

Diagnostic checklist:
```
[ ] Is the KG scope narrower than or equal to the query domain?
[ ] Are retrieved triples directly relevant (inspect 20 random retrievals)?
[ ] Is the model large enough that No-RAG already performs well?
[ ] Are multiple KGs being unioned without filtering?
[ ] Are vague/generic triples ("it causes problems") being filtered out?
[ ] Is temperature set to 0.0-0.3?
```

**Example 3: Deciding scope boundaries for multi-domain queries**

User: "My users ask questions that span diabetes AND cardiovascular disease. Should I build one combined KG or two separate ones?"

Approach:
1. Build two separate KGs: G_diabetes and G_cardio
2. For queries spanning both domains, compute the **intersection** — triples where entities appear in both KGs — rather than the union
3. The intersection captures genuine cross-domain relationships (e.g., `insulin_resistance → increases_risk_of → atherosclerosis`) without importing domain-specific noise
4. Use embedding similarity (threshold ~0.65) to identify overlapping entities across KGs with different naming conventions
5. At query time: classify query domain → if single-domain, use that KG; if cross-domain, use the intersection subgraph

```python
def compute_kg_intersection(kg1: dict, kg2: dict, similarity_threshold: float = 0.65) -> dict:
    """Find triples with entities present in both KGs via embedding similarity."""
    intersection = {}
    kg2_entities = {e: embed(e) for e in kg2.keys()}
    for entity, triples in kg1.items():
        entity_emb = embed(entity)
        for kg2_entity, kg2_emb in kg2_entities.items():
            if cosine_sim(entity_emb, kg2_emb) >= similarity_threshold:
                canonical = pick_canonical(entity, kg2_entity)
                intersection[canonical] = triples + kg2.get(kg2_entity, [])
    return intersection
```

## Best Practices

- **Do:** Build one KG per well-defined subdomain and keep them separate. Scope alignment is the single biggest factor in whether KG-RAG helps.
- **Do:** Run a No-RAG baseline before investing in KG construction. For large models on well-studied topics, parametric knowledge may already suffice.
- **Do:** Canonicalize entity names aggressively — co-reference resolution eliminates duplicate nodes and improves retrieval precision.
- **Do:** Tag every extracted triple with source provenance (paper ID, sentence, clause) so answers can be traced back to evidence.
- **Avoid:** Merging all available KGs into one retrieval source. Graph unions consistently underperform scope-matched individual graphs.
- **Avoid:** Retrieving more than ~30 triples per query. Context pollution from marginally relevant triples degrades accuracy more than missing a few relevant ones.
- **Avoid:** Using high decoding temperatures (>0.3) with KG-RAG. Low temperature preserves the factual signal from retrieved triples.

## Error Handling

| Failure Mode | Cause | Fix |
|---|---|---|
| Accuracy drops after adding RAG | Scope mismatch between KG and query domain | Narrow the KG or add a scope classifier before retrieval |
| Directionality errors (A→B vs B→A) | Causal direction lost during extraction | Enforce directed edge validation; use multi-hop probes to test |
| Duplicate/conflicting triples | Entity synonyms not resolved | Apply co-reference resolution and canonical name normalization |
| Vague triples pollute context | Extraction captured pronouns/generic terms | Add rule-based filter: reject triples where head or tail is a pronoun, demonstrative, or <4 characters |
| Model ignores retrieved context | Prompt doesn't emphasize retrieved knowledge | Add explicit instruction: "Prefer the retrieved knowledge over your training data for this question" |
| Cross-domain queries return nothing | No intersection between domain KGs | Lower similarity threshold (try 0.50) or verify KGs actually share entities |

## Limitations

- **KG construction is expensive.** Extracting clean, canonicalized triples from literature requires NLP infrastructure (co-reference resolution, syntactic decomposition, relation extraction). Budget for this upfront.
- **Only causal relations are modeled.** The paper's KGs use `causes/because` edges. Domains requiring hierarchical (is-a), compositional (part-of), or temporal relations need schema extensions.
- **No learned reranking.** The paper uses rule-based filtering only. A neural reranker (cross-encoder over query + retrieved triples) would likely improve precision further but adds latency.
- **Evaluation was multiple-choice only.** Open-ended generation with KG-RAG is less studied — the benefits of scope matching may differ for free-text answers.
- **Domain drift.** KGs built from a fixed corpus become stale. Plan for periodic re-extraction as new literature is published.
- **The paper tests biomedical domains.** Scope-matching principles likely generalize, but the specific thresholds (e.g., 0.65 cosine similarity for intersection) may need recalibration for other fields.

## Reference

Anuyah, S., Kaushik, M. M., Dai, H., Shiradkar, R., & Durresi, A. (2026). *Domain-Specific Knowledge Graphs in RAG-Enhanced Healthcare LLMs.* arXiv:2601.15429v1. [https://arxiv.org/abs/2601.15429v1](https://arxiv.org/abs/2601.15429v1)

Key takeaway: Scope-matched retrieval from narrow domain KGs consistently outperforms broad graph unions. Precision of retrieved context matters more than volume — and for large models, No-RAG may already be sufficient.