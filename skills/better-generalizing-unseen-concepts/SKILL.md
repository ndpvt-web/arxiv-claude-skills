---
name: "better-generalizing-unseen-concepts"
description: "Build biomedical concept recognition systems that generalize to unseen ontology concepts using hierarchical indexing and LLM-based auto-labeling. Use when: 'recognize biomedical entities not in training data', 'build NER for rare phenotypes', 'auto-label biomedical text with ontology concepts', 'evaluate generalization to unseen concepts', 'construct hierarchical concept indices from ontologies', 'scale biomedical annotation with LLMs'."
---

# Better Generalizing to Unseen Concepts: Hierarchical Indexing + LLM Auto-Labeling for Biomedical Concept Recognition

This skill enables Claude to help users build biomedical concept recognition pipelines that generalize beyond training data to unseen ontology concepts. It implements the HI-ALD approach: constructing **hierarchical concept indices** (OSSI) from biomedical ontologies, generating **LLM-based auto-labeled data** (ALD) through a 5-stage pipeline, and evaluating generalization with purpose-built metrics (U-RC, U-CS). The technique is particularly valuable for Mention-Agnostic Biomedical Concept Recognition (MA-BCR), where the goal is to identify ontology concepts referenced in text without requiring explicit span extraction.

## When to Use

- When the user needs to recognize biomedical concepts (diseases, phenotypes, gene functions) that are rare or absent from available training annotations
- When building a concept normalization or entity linking system against a large ontology (HPO, Gene Ontology) where manual annotation covers <5% of concepts
- When the user asks to auto-label biomedical text using LLMs to augment scarce human annotations
- When evaluating how well a biomedical NER/concept recognition model generalizes to concepts never seen during training
- When constructing a hierarchical search index over an ontology that combines structural (parent-child) and semantic (embedding similarity) relationships
- When the user wants to move from span-based NER to mention-agnostic concept recognition that captures implicit references

## Key Technique

**Mention-Agnostic Biomedical Concept Recognition (MA-BCR)** differs from traditional NER by skipping span extraction entirely. Given a text passage, the system directly outputs a set of ontology concept IDs -- including *implicit* concepts not explicitly mentioned but logically entailed. For example, the phrase "melanin pigment synthesis" might map to the implicit concept "Generalized hypopigmentation" through domain inference. This is modeled as sequence-to-sequence generation where the output is a sequence of hierarchical concept indices.

**Hierarchical Concept Indices (OSSI)** encode each ontology concept as a path through a tree built from the ontology graph. The construction combines two edge types: **OSI edges** (ontology parent-child relations, weight 1.0) and **SSI edges** (top-10 nearest SapBERT embedding neighbors, weight proportional to cosine similarity). Louvain community detection recursively partitions this hybrid graph into clusters of at most 10 children, producing a tree where each concept's index is its root-to-leaf path. This gives the seq2seq model a structured output space -- even when it predicts a wrong concept, the hierarchical prefix narrows the search to a semantically and ontologically coherent neighborhood.

**The 5-Stage Auto-Labeling Pipeline** generates training data at scale using LLMs. It progresses from claim generation (breaking passages into atomic biomedical claims via LLaMA-3-8B), to candidate extraction and semantic matching (SapBERT embeddings with L2 distance, threshold >= 0.6), to concept classification (explicit/logically implicit/pragmatically implicit/not relevant via GPT-4o-mini), to relabeling with justification, to guideline-based filtering and quality scoring. Only passages rated "Average" or above are retained. This pipeline produced 54K-34K annotated passages at <$150 total cost, covering 54-69% of target ontologies versus <3% for manual annotations.

## Step-by-Step Workflow

1. **Define the target ontology and concept space.** Load the ontology (e.g., HPO, Gene Ontology) as a JSON file mapping concept IDs to `[name, definition, tree_paths, synonyms_list]`. Identify the subset of target concepts relevant to your domain.

2. **Generate SapBERT embeddings for all concept names and synonyms.** Use `cambridgeltl/SapBERT-from-PubMedBERT-fulltext` to encode each concept name and synonym into 768-dimensional vectors. Store as a pickle file mapping concept IDs to their embedding vectors.

3. **Build the hybrid OSSI graph.** Construct a weighted graph where nodes are concepts: add edges with weight 1.0 for ontology parent-child relations (OSI), and edges weighted by embedding similarity for the top-L=10 nearest semantic neighbors (SSI). Use Faiss for efficient nearest-neighbor search.

4. **Partition the graph into a hierarchical index.** Apply Louvain community detection with dynamic resolution to recursively partition the graph. Each partition level creates clusters of at most M=10 children. Fall back to METIS for sparse subgraphs. The result is a tree where each concept's index is its root-to-leaf cluster membership path (e.g., `3-7-2-1`).

5. **Generate auto-labeled training data using the 5-stage LLM pipeline:**
   - **Stage 1 (Claim Generation):** Feed biomedical passages to LLaMA-3-8B to generate atomic claims, then extract candidate concept names from claims. Match candidates to ontology via SapBERT embedding similarity (L2 distance, keep top-1 if >= 0.6 threshold).
   - **Stage 2 (Classification):** Use GPT-4o-mini to classify each candidate as explicit, logically implicit, pragmatically implicit, or not relevant. Discard "not relevant."
   - **Stage 3 (Relabeling):** Re-evaluate each passage-concept pair with justifications (strengths/weaknesses). Identify missing concepts and add them.
   - **Stage 4 (Guideline Filtering):** Summarize domain annotation guidelines and use the LLM to filter concepts that violate them.
   - **Stage 5 (Quality Scoring):** Rate each passage on a 5-tier scale. Retain only "Average," "Good," or "Excellent" passages.

6. **Train a seq2seq recognizer.** Fine-tune BART-large (or similar) to map input passages to sequences of hierarchical concept indices. Use learning rate 1e-5, batch size 4, max sequence length 1024, for up to 50 epochs.

7. **Evaluate with generalization metrics.** Ensure at least 30% of test concepts are unseen (absent from training). Compute:
   - **U-RC (Unseen Recall-oriented Closeness):** Average longest common prefix between predicted and gold indices for unseen concepts, normalized by index depth. Higher is better.
   - **U-CS (Unseen Candidate-set Size):** Harmonic mean of the number of concepts sharing the predicted prefix with each unseen gold concept. Lower is better (tighter search space).
   - Standard F1 for seen concepts.

8. **Iterate on ALD volume and quality thresholds.** Progressively add more ALD (200, 400, 800, ... passages). Monitor U-RC for consistent improvement -- ALD shows monotonic gains on generalization metrics even when standard F1 plateaus.

9. **Deploy a recognition-to-reranking pipeline.** Use the recognizer's top-k predictions as candidates, then apply a reranker. U-RC correlates strongly with downstream reranking F1 (Spearman rho = 0.874), making it the primary optimization target.

## Concrete Examples

**Example 1: Building an HPO Phenotype Recognizer from Scratch**

User: "I have clinical case reports and need to tag Human Phenotype Ontology concepts, but I only have 228 manually annotated passages covering 2% of HPO."

Approach:
1. Load HPO ontology (`concept_info.json`) with 18,354 target concepts and 21,987 synonyms.
2. Generate SapBERT embeddings for all HPO concept names/synonyms.
3. Build OSSI graph: 18,354 nodes, OSI edges from HPO hierarchy, SSI edges from top-10 embedding neighbors.
4. Partition into hierarchical index using Louvain (resolution auto-tuned per level).
5. Run 5-stage ALD pipeline on PubMed abstracts mentioning phenotype-related terms. Cost: ~$50-100 for 50K passages.
6. Train BART-large on 228 MLD passages + 54,301 ALD passages.
7. Evaluate: expect ALD to push concept coverage from 2.35% to ~69%, with U-RC improving from ~25 to ~41 as ALD volume increases.

Output: A recognizer that, given "Patient presents with delayed motor development and hypotonia," outputs HPO indices mapping to concepts like `HP:0001263` (Motor delay), `HP:0001252` (Hypotonia), and potentially implicit concepts like `HP:0001270` (Motor deterioration).

**Example 2: Evaluating Generalization to Unseen Gene Ontology Concepts**

User: "I trained a GO concept recognizer but I suspect it only works for concepts it saw in training. How do I measure this?"

Approach:
1. Partition test concepts into "seen" (appeared in training) and "unseen" (at least 30% of test set).
2. Build an OSSI hierarchical index over the GO ontology (29,367 concepts, 87,705 synonyms).
3. For each unseen gold concept in the test set, find the model's closest prediction by longest common prefix of their hierarchical indices.
4. Compute U-RC: average `len(lcp(predicted_index, gold_index)) / len(gold_index)` across all unseen concepts. A score of 0.4 means predictions share 40% of the hierarchical path on average.
5. Compute U-CS: harmonic mean of candidate set sizes at the predicted prefix level. If U-CS = 33, the model narrows search to ~33 candidates on average.

Output:
```
Generalization Report:
  Seen concepts:    F1 = 73.1
  Unseen concepts:  U-RC = 40.9 (OSSI index)
                    U-CS = 33.2 (SSI index)
  Coverage:         Training covers 54.4% of ontology
  Recommendation:   Add more ALD -- U-RC shows monotonic improvement with data volume.
```

**Example 3: Running the Auto-Labeling Pipeline on New Biomedical Text**

User: "I have 10,000 PubMed abstracts about rare diseases. How do I auto-label them with HPO concepts?"

Approach:
1. Precompute SapBERT embeddings for all HPO concepts. Build a Faiss L2 index.
2. For each abstract, generate 3-5 atomic biomedical claims using LLaMA-3-8B with prompt: "Break the following biomedical passage into independent factual claims: {passage}."
3. Extract candidate concept names from each claim. Query Faiss index with SapBERT-encoded candidates; keep matches with L2 distance corresponding to similarity >= 0.6.
4. Batch candidate-concept pairs to GPT-4o-mini for classification (explicit/implicit/not relevant). Estimated cost: ~$15 for 10K abstracts.
5. Relabel surviving pairs with justifications. Filter using summarized HPO annotation guidelines.
6. Score passage quality. Expect ~60-70% of passages to meet "Average" or above threshold.

Output: JSON lines file with structure:
```json
{"passage_id": "PMID_12345", "text": "...", "concepts": [
  {"id": "HP:0001263", "index": "3-7-2-1", "type": "explicit", "quality": "good"},
  {"id": "HP:0001252", "index": "3-7-4-2", "type": "logically_implicit", "quality": "average"}
]}
```

## Best Practices

- **Do:** Use the OSSI (hybrid) index rather than OSI or SSI alone. The combination of ontology structure and semantic similarity produces the most informative hierarchical partitioning, particularly for ontologies with uneven depth.
- **Do:** Start with a small ALD batch (200 passages) and scale up in powers of 2. Monitor U-RC at each increment -- it should improve monotonically. If it plateaus early, revisit quality filtering thresholds.
- **Do:** Use claim generation (Stage 1) as an intermediate step. Direct passage-to-concept matching achieves only ~30% recall; claim decomposition boosts this to ~62%.
- **Do:** Track U-RC alongside F1. ALD-trained models may show lower F1 than MLD-only models but significantly better generalization (U-RC), which correlates more strongly with downstream task performance (rho = 0.874 vs. 0.554).
- **Avoid:** Using ALD as a full replacement for manual annotations. ALD improves coverage and generalization but introduces noise. Always include available MLD in training.
- **Avoid:** Setting the SapBERT matching threshold below 0.6 -- lower thresholds introduce excessive false positive concepts that propagate through later pipeline stages.
- **Avoid:** Skipping the guideline-based filtering stage. Domain-specific annotation guidelines catch systematic errors (granularity mismatches, semantic scope shifts) that quality scoring alone misses.

## Error Handling

- **Sparse ontology subgraphs cause Louvain to produce degenerate partitions.** Detect clusters with >10 children and fall back to METIS partitioning for those subgraphs. If METIS hangs on large subgraphs, restart with `resume=True`.
- **SapBERT embedding mismatch across ontology versions.** Always regenerate embeddings when the ontology version changes. Cached embeddings from a prior version will produce incorrect nearest-neighbor matches.
- **LLM hallucination in claim generation.** Claims may introduce concepts not grounded in the passage. Stages 2-4 are specifically designed to filter these -- do not skip intermediate stages even if they seem redundant.
- **Low quality pass rate (<50%).** If too many passages are filtered at Stage 5, relax the threshold to include "Poor" quality temporarily, but weight these lower during training or use curriculum learning.
- **Index depth imbalance.** If some branches of the hierarchical index are much deeper than others, U-RC scores become incomparable across subtrees. Normalize by index depth per concept (already built into the U-RC formula).

## Limitations

- The auto-labeling pipeline requires access to both an instruction-tuned LLM (GPT-4o-mini or equivalent) and a biomedical language model (LLaMA-3-8B, SapBERT). Cost is low (~$150 for 90K passages) but not zero.
- ALD-trained models consistently underperform MLD-trained models on standard F1 for seen concepts. ALD is a supplement for generalization, not a replacement for human annotation.
- The hierarchical index quality depends heavily on the ontology's structural completeness. Flat or poorly organized ontologies produce shallow, uninformative indices.
- U-RC and U-CS metrics require a hierarchical index to compute -- they cannot be applied to flat label sets or arbitrary tagsets.
- The pipeline is validated on HPO (phenotypes) and Gene Ontology (biological processes). Adaptation to other biomedical ontologies (SNOMED-CT, MeSH, ChEBI) requires re-running index construction and may need guideline summarization for Stage 4.
- Implicit concept recognition (concepts not explicitly mentioned in text) remains the hardest case and benefits least from increased ALD volume.

## Reference

- **Paper:** [Better Generalizing to Unseen Concepts (EACL 2026)](https://arxiv.org/abs/2601.16711v1) -- Focus on Section 3 (evaluation framework and metrics), Section 4 (auto-labeling pipeline stages), and Tables 3-4 (scaling behavior of ALD on generalization metrics).
- **Code & Data:** [github.com/bio-ie-tool/hi-ald](https://github.com/bio-ie-tool/hi-ald) -- Scripts for index construction (`get_concept_embeddings.py`, `build_graph.py`, `partition_graph.py`, `map_index.py`) and pre-built ALD datasets for HPO and Gene Ontology.