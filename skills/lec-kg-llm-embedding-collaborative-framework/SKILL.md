---
name: "lec-kg-llm-embedding-collaborative-framework"
description: "Build domain-specific knowledge graphs from unstructured text using an iterative LLM + embedding validation loop. Combines hierarchical relation extraction, evidence-grounded Chain-of-Thought feedback, and semantic initialization for unseen entities. Use when: 'build a knowledge graph from these documents', 'extract entities and relations from domain text', 'construct KG from policy reports', 'validate knowledge graph triples', 'handle long-tail relations in KG construction', 'iterative KG refinement pipeline'."
---

# LEC-KG: LLM-Embedding Collaborative Knowledge Graph Construction

This skill enables Claude to build domain-specific knowledge graphs from unstructured text by implementing the LEC-KG bidirectional framework. The core idea: use an LLM for semantic extraction and a Knowledge Graph Embedding model (like RotatE) for structural validation, then loop them together so each improves the other. The LLM extracts candidate triples, the embedding model scores them for structural plausibility, low-scoring triples get re-evaluated with evidence-grounded feedback, and high-confidence triples improve the embedding model. This iterative refinement is particularly effective for domains with long-tail relation distributions where naive LLM extraction fails on rare relation types.

## When to Use

- When the user wants to extract structured knowledge (entities + relations) from a corpus of domain-specific documents (policy reports, scientific papers, legal texts, medical records)
- When building a knowledge graph from scratch without a pre-existing schema, or when the schema has many relation types with imbalanced frequency
- When the user needs to validate LLM-extracted triples against structural consistency rather than trusting raw LLM output
- When handling domains with long-tail relation distributions where common relations dominate and rare ones get missed
- When the user wants to process documents where new/unseen entities appear that weren't in training data
- When constructing a KG pipeline that improves over multiple passes rather than single-shot extraction

## Key Technique

**Bidirectional LLM-KGE collaboration.** Standard LLM-based KG construction suffers from two problems: (1) LLMs are biased toward frequent relation types, missing rare but important relations, and (2) LLMs lack structural awareness — they don't know if a triple is consistent with the graph's overall topology. LEC-KG solves both by pairing the LLM with a Knowledge Graph Embedding model (RotatE) in an iterative loop. The embedding model scores each candidate triple for structural plausibility using the formula `s(τ) = σ(-‖h ⊙ r - t‖)` where h, r, t are complex-valued embeddings. Triples scoring above a high threshold are accepted; those in a middle band get sent back to the LLM with diagnostic feedback; those below a low threshold are rejected.

**Hierarchical coarse-to-fine extraction.** Instead of asking the LLM to choose from all relation types at once (e.g., 89 types), group relations into coarse semantic categories (e.g., 8 clusters: Hierarchy, Spatiotemporal, Quantitative, Causality, etc.). The LLM first classifies into a coarse category, then picks the fine-grained relation from that subset (~11 candidates instead of 89). This reduces the search space and lets rare relations compete within their semantic neighborhood rather than against dominant head relations.

**Evidence-guided Chain-of-Thought feedback.** When the embedding model flags a triple as structurally implausible, it doesn't just return a score. The system retrieves the source sentences containing both entities, presents the top-3 alternative relations from RotatE, and asks the LLM to reason step-by-step: Does the original relation have textual support? Do alternatives fit better? Does the corrected triple satisfy schema constraints? This grounding in source text prevents hallucinated corrections. For unseen entities not in the embedding vocabulary, a learned projection `e_KGE = W · RoBERTa(e)[CLS] + b` maps text embeddings into the KGE space, enabling scoring of any triple regardless of training history.

## Step-by-Step Workflow

1. **Define the domain schema.** Enumerate entity types and relation types for the target domain. Organize relations into 5-10 coarse semantic categories (e.g., Causal, Temporal, Hierarchical, Quantitative). Each fine-grained relation belongs to exactly one category. Define permissible entity-type pairs per relation as schema constraints.

2. **Chunk the source documents.** Split input text into overlapping chunks (~2000 characters with ~200-character overlap). Track chunk provenance so evidence sentences can be retrieved later.

3. **Perform hierarchical coarse-to-fine extraction.** For each chunk, prompt the LLM with the full schema, few-shot examples, and the coarse-then-fine instruction. The LLM outputs candidate triples in the format `(head_entity, relation, tail_entity, evidence_span, coarse_category)`. Include 4-6 diverse few-shot demonstrations spanning different coarse categories.

4. **Apply schema constraint validation.** Filter candidates against the defined entity-type-pair constraints. Reject triples where the head/tail entity types are incompatible with the assigned relation. Normalize entity mentions (merge duplicates, resolve aliases).

5. **Initialize the KGE model.** Train a RotatE model on the validated triples from step 4 (the cold-start seed). Use 512-dimensional complex embeddings with self-adversarial negative sampling (α=1.0). For entities not seen during training, compute initial embeddings via a projection from a pretrained text encoder (e.g., `W · encoder(entity_name)[CLS] + b`).

6. **Score all candidate triples.** Compute structural plausibility scores using the trained RotatE. Set dynamic thresholds using percentiles of the score distribution: `θ_low = Percentile_25(scores)`, `θ_high = Percentile_70(scores)`.

7. **Tri-partition and route triples.** Accept triples scoring ≥ θ_high into the validated graph. Send triples in [θ_low, θ_high) to the evidence-guided CoT feedback channel. Reject triples scoring < θ_low.

8. **Run evidence-guided CoT re-extraction.** For each flagged triple: retrieve source sentences containing both entities, present the structural score + top-3 alternative relations from RotatE, and prompt the LLM to reason through whether the original or an alternative relation is better supported. Allow up to K=3 retry attempts per triple. Corrected triples re-enter the scoring pipeline.

9. **Update the KGE model with validated triples.** Add high-confidence triples (top 30% by score) directly to training data. Verify medium-confidence triples (next 45%) via the CoT channel before adding. Discard the bottom 25%. Warm-start retrain RotatE, preserving previous embeddings.

10. **Iterate until convergence.** Repeat steps 6-9 for up to T=4 iterations. Stop early if the relative growth in validated triples falls below ε=0.01. Each iteration refines both the LLM's extraction quality (via structural feedback) and the KGE's representations (via new validated triples).

## Concrete Examples

**Example 1: Building a sustainability policy KG**

```
User: I have 50 PDF reports on urban sustainability. Build a knowledge graph
      from them with entities like organizations, metrics, policies, and locations.

Approach:
1. Define schema with coarse categories:
   - Hierarchy: [isPartOf, hasSubsystem, governedBy]
   - Quantitative: [hasValue, hasTarget, measuredBy]
   - Spatiotemporal: [locatedIn, occursIn, validDuring]
   - Causal: [causes, mitigates, exacerbates]
   - Provenance: [dataSourceOf, reportedBy, publishedIn]

2. Chunk all 50 PDFs into ~2000-char segments with 200-char overlap.

3. Prompt the LLM per chunk with hierarchical extraction:
   "Given this text, extract all (head, relation, tail) triples.
    First classify each relation into a coarse category, then
    select the fine-grained type from that category."

4. Cold-start: train RotatE on the ~500 initial triples.

5. Score, tri-partition, run CoT feedback, iterate 4 rounds.

Output (after 4 iterations):
  Validated triples: 2,847
  Entity count: 1,203
  Sample triples:
    (Beijing, hasTarget, "carbon neutrality by 2060")
    (Solar capacity, hasValue, "392 GW installed")
    (PM2.5 reduction, causes, Respiratory health improvement)
    (Green Building Standard, governedBy, Ministry of Housing)
```

**Example 2: Medical literature KG with rare relations**

```
User: Extract drug-gene-disease relationships from 200 PubMed abstracts.
      Many interaction types are rare — make sure we don't miss them.

Approach:
1. Define coarse relation categories:
   - Therapeutic: [treats, alleviates, prevents]
   - Mechanistic: [inhibits, activates, binds, regulates]
   - Genetic: [mutationIn, expressedIn, encodedBy]
   - Adverse: [causesAdverseEffect, contraindicatedWith]
   - Diagnostic: [biomarkerFor, indicativeOf]

2. The hierarchical extraction matters most here: rare relations
   like "contraindicatedWith" (maybe 3 occurrences in 200 abstracts)
   would be overwhelmed by "treats" (maybe 150 occurrences) in a
   flat extraction. By grouping into coarse categories, "contraindicatedWith"
   only competes with "causesAdverseEffect" in the Adverse cluster.

3. After initial extraction, RotatE scoring catches structurally
   implausible triples like (Aspirin, mutationIn, BRCA1) — the
   embedding model knows drugs don't have mutations.

4. CoT feedback for borderline triples retrieves the source sentence:
   "Aspirin showed interaction with the COX-2 pathway"
   → LLM corrects (Aspirin, mutationIn, COX-2) to (Aspirin, inhibits, COX-2)

Output:
  Total validated triples: 1,456
  Long-tail relation recovery: 2x improvement over single-pass extraction
  Sample corrections via feedback loop:
    Before: (Metformin, treats, insulin) → structurally implausible
    After:  (Metformin, regulates, insulin secretion) → validated
```

**Example 3: Implementing the pipeline in Python**

```
User: Write me the core pipeline code for LEC-KG using OpenAI API and PyKEEN.

Approach:
1. Set up document chunking with overlap.
2. Implement hierarchical extraction prompts.
3. Use PyKEEN's RotatE for embedding training and scoring.
4. Build the tri-partition routing logic.
5. Implement evidence-guided CoT feedback prompts.
6. Wire the iterative loop with convergence checking.

Key code structure:
  lec_kg/
    schema.py          # Domain schema definition (coarse + fine relations)
    chunker.py         # Document chunking with overlap and provenance
    extractor.py       # Hierarchical LLM extraction with few-shot prompts
    embeddings.py      # RotatE training, scoring, semantic initialization
    feedback.py        # Evidence retrieval + CoT re-extraction prompts
    pipeline.py        # Iterative loop orchestrating all components
    validators.py      # Schema constraint checking, entity normalization

Core loop in pipeline.py:
  for iteration in range(max_iterations):
      candidates = extractor.extract(chunks, schema)
      candidates = validators.check_schema(candidates)
      scores = embeddings.score(candidates)
      accepted, feedback_set, rejected = tripartition(candidates, scores)
      corrected = feedback.reextract(feedback_set, evidence_store)
      graph.add(accepted + corrected)
      embeddings.update(graph, warm_start=True)
      if relative_growth(graph) < epsilon:
          break
```

## Best Practices

- **Do:** Always use hierarchical coarse-to-fine extraction when the relation schema has more than ~15 types. The long-tail problem is nearly universal in domain-specific KG construction.
- **Do:** Ground all KGE feedback in source text evidence. Presenting bare structural scores to the LLM produces hallucinated corrections. Always retrieve and include the relevant source sentences.
- **Do:** Use dynamic percentile-based thresholds (25th and 70th percentile) rather than fixed score cutoffs. Score distributions shift across iterations as the embedding model improves.
- **Do:** Warm-start the embedding model each iteration rather than retraining from scratch. This preserves learned structural patterns while incorporating new triples.
- **Avoid:** Running more than 4-5 iterations. Empirically, gains plateau and error accumulation begins. Monitor relative triple growth and stop when it drops below 1%.
- **Avoid:** Skipping schema constraint validation before embedding scoring. Without it, structurally plausible but semantically invalid triples (e.g., wrong entity types) pollute the graph.
- **Avoid:** Using the feedback loop on all low-scoring triples simultaneously. Batch them and limit retries to K=3 per triple to prevent infinite loops and excessive API costs.

## Error Handling

- **Unseen entities at scoring time:** Use the semantic initialization projection (`W · encoder(entity)[CLS] + b`) trained on known entities. If the projection model isn't trained yet (cold-start), skip embedding scoring for those triples and rely on schema validation alone until iteration 2+.
- **LLM returns malformed triples:** Validate output format strictly. If the LLM outputs triples missing fields or with unknown relation types, retry with a stricter prompt specifying the exact output format. Discard after 2 failed attempts.
- **KGE scores collapse to uniform distribution:** This happens when the training set is too small (<100 triples). Increase the cold-start seed by lowering the acceptance threshold for the first iteration only, then tighten in subsequent rounds.
- **CoT feedback loop produces worse triples:** If a corrected triple scores lower than the original, keep the original. Track correction acceptance rate; if it drops below 30%, the evidence retrieval may be returning irrelevant sentences — tighten the entity matching.
- **Entity normalization conflicts:** When the same real-world entity appears with different surface forms, use the LLM to cluster mentions before extraction. Maintain an alias table and merge embeddings for confirmed duplicates.

## Limitations

- **Requires a defined relation schema.** LEC-KG is not an open information extraction system. You need to enumerate relation types upfront, which requires domain expertise. Open-ended "extract everything" requests need a schema design phase first.
- **Embedding model needs a minimum seed.** RotatE needs at least ~200-300 initial triples to produce meaningful scores. For very small corpora (under 10 documents), the iterative loop provides minimal benefit over single-pass LLM extraction.
- **Computational cost scales with iterations.** Each iteration involves LLM calls for re-extraction and KGE retraining. For large corpora (1000+ documents), budget 4x the cost of single-pass extraction.
- **Language-specific components.** The semantic initialization relies on a language-matched pretrained encoder (e.g., Chinese-RoBERTa for Chinese text). Switching languages requires swapping the encoder and retraining the projection layer.
- **Not suitable for real-time or streaming KG construction.** The iterative batch process assumes a static corpus. For continuously arriving documents, adapt to incremental updates rather than full re-iteration.

## Reference

[LEC-KG: An LLM-Embedding Collaborative Framework for Domain-Specific Knowledge Graph Construction](https://arxiv.org/abs/2602.02090v1) — Zeng, Piao, Li (2026). Focus on Section 3 for the bidirectional architecture, Section 3.2 for hierarchical extraction, Section 3.4 for evidence-guided CoT feedback, and Table 5 for ablation results showing each component's contribution.