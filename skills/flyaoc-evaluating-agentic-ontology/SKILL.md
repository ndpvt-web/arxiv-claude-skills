---
name: "flyaoc-evaluating-agentic-ontology"
description: "Build multi-agent systems for end-to-end ontology curation from scientific literature. Applies FlyAOC's agent architecture patterns—memorization, pipeline, single-agent, and multi-agent—to extract structured, ontology-grounded annotations from document corpora. Use when asked to: 'curate knowledge from papers into a structured ontology', 'build an agent pipeline for scientific literature extraction', 'design a multi-agent system for document annotation', 'extract Gene Ontology or controlled-vocabulary terms from text', 'reconcile evidence across multiple documents into structured annotations', 'build a retrieval-augmented scientific reasoning system'."
---

# Agentic Ontology Curation from Scientific Literature

This skill enables Claude to design and implement multi-agent systems that curate structured, ontology-grounded annotations from large document corpora. Based on the FlyAOC benchmark (FlyBench), it applies the finding that **multi-agent architectures with delegated paper analysis and bounded orchestrator context consistently outperform monolithic pipelines and single-agent designs** for scientific knowledge extraction. The core pattern—search, read, reconcile evidence, ground to controlled vocabularies—generalizes beyond Drosophila genetics to any domain requiring structured curation from unstructured text.

## When to Use

- When building a system that searches a paper corpus, extracts entities, and maps them to ontology terms (Gene Ontology, MeSH, SNOMED, custom vocabularies)
- When designing agent architectures for scientific literature mining and the user needs to choose between pipeline vs. single-agent vs. multi-agent approaches
- When implementing retrieval-augmented extraction where evidence must be reconciled across multiple documents
- When curating a knowledge base from primary literature—linking entity mentions to controlled vocabulary identifiers with evidence provenance
- When the user wants to extract structured annotations (functions, relationships, synonyms, expression patterns) from a collection of full-text papers given only an entity identifier
- When evaluating or benchmarking different agent designs for document-grounded information extraction

## Key Technique

FlyAOC evaluates four agent architectures for end-to-end ontology curation, where an agent receives only an entity name (e.g., a gene symbol) and must search a corpus of thousands of papers, read relevant documents, and produce structured annotations grounded in controlled vocabularies.

**The critical architectural insight** is that multi-agent designs—where an orchestrator delegates individual paper analysis to specialized subagents and never sees raw paper text—avoid the context overflow problem that degrades single-agent performance. Each subagent processes one document in isolation and returns pre-resolved annotations with ontology IDs. The orchestrator aggregates, deduplicates, and ranks results while keeping its context window bounded. This yields consistent performance scaling: more papers processed means better results, unlike single-agent designs where performance peaks then declines as context fills.

**The retrieval-confirmation finding** is equally important for system design: agents primarily use retrieval to confirm parametric knowledge rather than discover genuinely new information. This means the retrieval component should be designed to surface both confirmatory and contradictory evidence. For domains where terminology is specialized or historical (expression patterns, legacy synonyms), retrieval provides the largest gains over memorization. For well-represented domains (standard Gene Ontology terms), retrieval gains are marginal. Design your retrieval budget accordingly—allocate more retrieval effort to rare, domain-specific annotation types.

## Step-by-Step Workflow

1. **Define the annotation schema.** Specify the exact structured output: what entity types, relationship types, and controlled vocabulary identifiers the system must produce. Use JSON schemas with required fields for term ID, term name, evidence text, and source document. Example: `{ "go_id": "GO:0007391", "go_name": "dorsal closure", "aspect": "biological_process", "evidence": "...", "paper_id": "PMC123456" }`.

2. **Build or connect to the controlled vocabulary index.** Load the target ontology (Gene Ontology OBO files, MeSH XML, or custom vocabulary) into a searchable index. Implement two tools: `search_ontology(query) -> [(term_id, term_name, similarity)]` for free-text lookup and `validate_term(term_id) -> bool` for ID verification. This prevents hallucinated ontology IDs.

3. **Set up BM25 corpus search.** Index your document corpus with BM25 (using `rank-bm25` or `whoosh`). Implement `search_corpus(query, top_k) -> [doc_ids]` returning ranked document identifiers. Support both entity-name queries and refined queries combining entity names with functional keywords.

4. **Implement the document reader tool.** Create `read_paper(doc_id) -> {title, abstract, sections}` that returns structured full text. Partition long documents into sections (introduction, methods, results, discussion) so subagents can focus on relevant sections.

5. **Build the multi-agent architecture with three roles:**
   - **Orchestrator**: Receives the entity identifier, issues corpus searches, dispatches papers to subagents, aggregates results, performs deduplication and ranking. Never processes raw paper text.
   - **Paper Analyst (subagent, one per document)**: Receives a single paper plus the entity identifier and annotation task. Extracts candidate annotations, resolves them against the ontology tools, returns structured JSON.
   - **Validator (optional)**: Cross-checks aggregated annotations for consistency, flags contradictions between papers, and verifies all ontology IDs are valid.

6. **Implement iterative ontology grounding with retry.** When a subagent extracts a concept that doesn't match any ontology term exactly, it should retry with alternative phrasings—synonyms, broader terms, or natural language descriptions. This feedback loop is the key advantage over rigid pipelines. Example: if "wing disc development" fails, try "imaginal disc development" or "wing morphogenesis".

7. **Set a retrieval budget and allocation strategy.** Based on FlyAOC findings: allocate ~16 papers per entity for good coverage. For well-known annotation types (standard functional terms), fewer papers suffice. For rare annotation types (historical synonyms, tissue-specific expression), allocate more. Track which ground-truth supporting papers are retrieved to measure discovery effectiveness.

8. **Aggregate and rank predictions.** The orchestrator collects annotations from all subagents, deduplicates by ontology term ID, counts supporting evidence across papers, and produces a ranked list. Assign confidence scores based on: (a) number of independent papers supporting the annotation, (b) ontology similarity to other confirmed annotations, (c) specificity of the evidence text.

9. **Evaluate using semantic similarity scoring.** Don't require exact ontology term matches. Use Wang semantic similarity over the ontology DAG: `recall = sum(max(sim(gt, pred) for pred in predictions) for gt in ground_truth) / |ground_truth|`. This credits predictions that are ontologically close but not identical to expert annotations.

10. **Iterate on architecture, not model scale.** FlyAOC shows diminishing returns from scaling backbone models but significant gains from architectural improvements. Invest effort in better retrieval strategies, context management, and agent coordination rather than simply upgrading to a larger LLM.

## Concrete Examples

**Example 1: Gene Function Curation Agent**

```
User: Build a system that takes a gene symbol and produces Gene Ontology
annotations by searching a corpus of papers.

Approach:
1. Define output schema:
   {
     "gene": "dpp",
     "annotations": [
       {
         "go_id": "GO:0007391",
         "go_name": "dorsal closure",
         "aspect": "biological_process",
         "evidence_text": "dpp signaling is required for dorsal closure...",
         "source_paper": "PMC2345678",
         "confidence": 0.92
       }
     ]
   }

2. Load Gene Ontology OBO file into a search index (goatools + whoosh).
3. Index paper corpus with BM25 (rank_bm25 library).
4. Implement orchestrator that:
   - Searches corpus for "dpp" -> retrieves top 16 papers
   - Spawns one subagent per paper with prompt:
     "Read this paper about gene dpp. Extract all Gene Ontology-relevant
      statements. For each, identify the GO aspect (BP/MF/CC), find the
      best matching GO term using search_ontology(), validate with
      validate_term(), and return structured JSON."
   - Aggregates subagent outputs, deduplicates by go_id,
     ranks by evidence count.

Output:
[
  {"go_id": "GO:0007391", "go_name": "dorsal closure",
   "aspect": "BP", "papers": 4, "confidence": 0.95},
  {"go_id": "GO:0048814", "go_name": "wing morphogenesis",
   "aspect": "BP", "papers": 3, "confidence": 0.88},
  {"go_id": "GO:0005125", "go_name": "cytokine activity",
   "aspect": "MF", "papers": 2, "confidence": 0.72}
]
```

**Example 2: Historical Synonym Extraction**

```
User: I need to find all historical names a gene has been called across
decades of literature, linked back to the papers where each name appeared.

Approach:
1. Define synonym schema:
   {"gene": "N", "synonyms": [{"name": "Notch", "papers": [...]}]}

2. Orchestrator searches corpus for the canonical gene symbol.
3. For each retrieved paper, a subagent scans for:
   - Explicit alias statements ("also known as", "formerly called")
   - Parenthetical synonyms: "gene-X (also GeneY)"
   - Nomenclature mapping tables
   - References to the gene under different names in older citations
4. Orchestrator aggregates synonyms, normalizes casing,
   deduplicates, and records the earliest paper for each synonym.

Output:
{
  "gene": "N",
  "canonical_name": "Notch",
  "synonyms": [
    {"name": "split", "first_seen": "PMC001234", "year": 1917, "count": 12},
    {"name": "notch-1", "first_seen": "PMC045678", "year": 1985, "count": 5},
    {"name": "N(spl)", "first_seen": "PMC091011", "year": 1992, "count": 3}
  ]
}
```

**Example 3: Choosing the Right Agent Architecture**

```
User: I'm building a literature extraction system. Should I use a pipeline
or a multi-agent approach?

Decision framework (from FlyAOC findings):

| Factor                        | Pipeline          | Multi-Agent           |
|-------------------------------|-------------------|-----------------------|
| Implementation complexity     | Low               | Medium                |
| Handles ambiguous terms       | No (no retry)     | Yes (iterative)       |
| Scales with more documents    | Linear            | Linear                |
| Context overflow risk         | None (stages)     | None (delegated)      |
| Ontology grounding accuracy   | Low (batch, rigid)| High (retry loops)    |
| Best for well-defined schemas | Yes               | Overkill              |
| Best for open-ended curation  | No                | Yes                   |

Recommendation:
- Use Pipeline when: annotations map 1:1 to known patterns, ontology is
  small, and you need speed over accuracy.
- Use Multi-Agent when: documents are long, ontology is large, terms are
  ambiguous, or you need evidence reconciliation across papers.
- Avoid Single-Agent when: processing >8 documents per entity (context
  degradation observed in FlyAOC beyond this threshold).
```

## Best Practices

- **Do:** Keep the orchestrator's context bounded—it should only see structured subagent outputs, never raw document text. This is the single most impactful architectural choice from FlyAOC.
- **Do:** Implement ontology grounding with retry loops. When a free-text concept doesn't match an ontology term, rephrase and re-search. Pipeline architectures that batch-resolve in one pass lose ~10% accuracy versus iterative approaches.
- **Do:** Track evidence provenance (source paper, section, exact quote) for every annotation. This enables downstream verification and is essential for scientific credibility.
- **Do:** Use semantic similarity (Wang similarity over the ontology DAG) rather than exact-match evaluation. Ontologies are hierarchical—a prediction of "wing disc morphogenesis" when the ground truth is "wing morphogenesis" should get partial credit.
- **Avoid:** Over-investing in model scale. FlyAOC shows diminishing returns from larger models. Spend that budget on better retrieval, more papers per entity, or improved agent coordination instead.
- **Avoid:** Assuming retrieval will discover novel knowledge. Agents predominantly confirm what the LLM already knows. Design explicit prompts that ask subagents to look for surprising or contradictory findings, not just confirmatory evidence.

## Error Handling

- **Hallucinated ontology IDs**: Always validate term IDs against the actual ontology before including them in output. Implement `validate_term()` as a hard gate—reject any annotation with an invalid ID.
- **Context overflow in single-agent mode**: If using a single-agent design, monitor context usage. When approaching the limit, stop retrieving new papers and produce output from what's been gathered. Better: switch to multi-agent.
- **Empty retrieval results**: When corpus search returns no relevant papers for an entity, fall back to memorization (parametric knowledge) but flag annotations as "unsupported by corpus evidence" with lower confidence scores.
- **Ontology term ambiguity**: When multiple ontology terms match a concept with similar scores, return all candidates ranked by contextual fit. Let the orchestrator or a human curator disambiguate.
- **Contradictory evidence across papers**: When subagents return conflicting annotations for the same entity, the orchestrator should flag the conflict explicitly rather than silently picking one. Include both annotations with a `"conflict": true` flag and the supporting evidence for each.

## Limitations

- The multi-agent approach adds latency and cost proportional to the number of documents analyzed. For simple extraction tasks with well-structured documents, a pipeline may be more efficient.
- Retrieval quality is bounded by the corpus index. If relevant papers aren't in the corpus or the BM25 index doesn't surface them, no architecture can compensate. Consider hybrid retrieval (BM25 + dense embeddings) for better recall.
- Agents struggle to propose genuinely novel ontology terms—concepts not yet in the controlled vocabulary. FlyAOC found only 34% of missing terms received even a natural language description attempt, with low semantic similarity (mean 0.20). Human curation remains necessary for ontology extension.
- The approach assumes a well-maintained, searchable ontology exists. For domains without established controlled vocabularies, the ontology grounding step breaks down and the system degenerates to free-text extraction.
- Evaluation with semantic similarity scoring requires the ontology to have a DAG structure. Flat vocabularies or tag systems need different evaluation metrics.

## Reference

**Paper:** [FlyAOC: Evaluating Agentic Ontology Curation of Drosophila Scientific Knowledge Bases](https://arxiv.org/abs/2602.09163v1) — Zhang et al., 2026. Focus on Section 3 (agent architectures and tool definitions), Section 5 (architecture comparison results), and Section 6 (analysis of retrieval vs. parametric knowledge). Code: `https://github.com/xingjian-zhang/flyaoc`.