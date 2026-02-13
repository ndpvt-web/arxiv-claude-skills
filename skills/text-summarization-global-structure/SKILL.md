---
name: "text-summarization-global-structure"
description: "Summarize long documents while preserving global semantic structure and logical coherence using topology-guided pruning (GloSA-sum). Use when the user says: 'summarize this long document', 'compress this text for an LLM context window', 'extract key points while keeping logical flow', 'reduce this report but keep the reasoning chain', 'shorten this paper without losing structure', 'create a coherent summary of these meeting notes'."
---

# Text Summarization via Global Structure Awareness (GloSA-sum)

This skill enables Claude to summarize long documents by first identifying the global semantic backbone — core themes and logical dependency cycles — using concepts from topological data analysis, then iteratively pruning low-importance sentences while protecting that backbone. Unlike flat extractive methods (TextRank, LexRank) that rank sentences independently, this approach builds a semantic-weighted graph, identifies persistent structural features (theme clusters and argument loops), locks them into a protection pool, and prunes remaining sentences by their topological connectivity and task relevance. The result is a summary that preserves coherence and reasoning chains, not just salient sentences.

## When to Use

- When the user asks to summarize a long document (report, paper, transcript) and coherence matters more than just hitting keywords
- When preparing input context for an LLM downstream task — compressing a long document while retaining the reasoning chains the LLM needs
- When the user wants to reduce redundancy in a multi-section document (government report, legal filing, research paper) without breaking cross-section logic
- When summarizing debate transcripts, meeting notes, or argumentative text where circular/recurring argument structures must be preserved
- When the user explicitly asks to "keep the logical flow" or "preserve the structure" during summarization
- When compressing technical documentation or multi-chapter content that has natural hierarchical structure

## Key Technique

**Semantic-Weighted Graph + Topological Analysis.** GloSA-sum treats a document as a graph where each node is a sentence and edge weights blend semantic similarity with positional proximity: `w_ij = α * (1 - cos(e_i, e_j)) + (1-α) * exp(-|i-j|/τ)`. Sentence embeddings come from a model like `all-mpnet-base-v2` (768-dim, L2-normalized). Neighbors are found via adaptive mutual k-nearest-neighbors (k grows logarithmically with document length, bounded 5-20). This graph encodes both what sentences mean and where they sit relative to each other.

**Persistent Homology for Core Structure Detection.** The method applies persistent homology (H0 and H1) over the graph's filtration. H0 features (connected components) reveal core semantic themes — clusters of sentences that stay connected across many similarity thresholds. H1 features (cycles/loops) reveal logical recurrence — argument chains that loop back. The top-K most persistent H0 components and top-M most persistent H1 cycles are collected into a "protection pool." These sentences form the structural backbone and are never pruned.

**Topology-Guided Iterative Pruning.** With the backbone locked, remaining sentences are scored by a composite metric: `Score(s_i) = λ * TopoScore(s_i) + (1-λ) * TaskScore(s_i)`. TopoScore measures shortest-path distance to protected nodes (closer = more important). TaskScore combines embedding cosine similarity and BM25 against an optional query. Each iteration removes the lowest-scoring unprotected sentence until the target compression ratio is reached. Because the protection pool is computed once upfront, no repeated expensive topology computation is needed. For very long documents, a hierarchical strategy segments the text, compresses segments in parallel, then runs a final global pass to remove cross-segment redundancy.

## Step-by-Step Workflow

1. **Segment the input into sentences.** Split the document at sentence boundaries. For each sentence, generate an embedding vector (use `all-mpnet-base-v2` or equivalent). L2-normalize all vectors.

2. **Build the semantic-weighted graph.** For each sentence pair within the mutual k-nearest-neighbor set, compute edge weight: `w_ij = 0.5 * (1 - cosine_similarity(e_i, e_j)) + 0.5 * exp(-|i-j| / 10)`. Use k = max(5, min(20, int(log2(n)))) where n is total sentence count. Store as an adjacency list or sparse matrix.

3. **Compute persistent homology (H0 and H1).** Build a filtration over the graph by increasing edge-weight threshold. Track connected components (H0) and cycles (H1) as they appear (birth) and merge/close (death). Compute persistence = death - birth for each feature. Use a library like `ripser`, `gudhi`, or `giotto-tda` if implementing in Python; otherwise, simulate by tracking union-find for H0 and cycle detection for H1.

4. **Assemble the protection pool.** Select the top-K longest-lived H0 features and collect their landmark/representative sentences. Select the top-M longest-lived H1 cycles and collect all sentences participating in those cycles. Union these sets into the protection pool P. Default K=3, M=2; increase for very long or multi-topic documents.

5. **Score all unprotected sentences.** For each sentence not in P, compute: (a) TopoScore = negative sum of shortest-path distances from the sentence to every sentence in P (use Dijkstra on the weighted graph); (b) TaskScore = cosine similarity to the query or document centroid (if no query, use document centroid); (c) Combined = 0.7 * TopoScore + 0.3 * TaskScore. Normalize both sub-scores to [0,1] before combining.

6. **Iteratively prune lowest-scoring sentences.** Remove the sentence with the lowest combined score. Remove its node and edges from the graph. Do NOT recompute topology — just update shortest-path distances incrementally. Repeat until the desired compression ratio (e.g., 30% of original) is reached.

7. **For long documents (>50 sentences), use hierarchical compression.** Split the document into segments at natural boundaries (section headers, paragraph groups, or fixed ~20-sentence chunks). Run steps 1-6 on each segment independently (parallelizable). Concatenate compressed segments and run a final global pass (steps 1-6 again) to eliminate cross-segment redundancy.

8. **Order surviving sentences by original position.** Reconstruct the summary by arranging retained sentences in their original document order to preserve narrative flow.

9. **Post-process for readability.** Check for dangling references (pronouns or "this" referring to pruned sentences). Replace ambiguous references with the noun they pointed to, or flag them for the user. Optionally, add lightweight transition phrases between non-adjacent retained sentences.

10. **Present the summary with structural annotations.** Label which sentences were protected as core themes vs. logical backbone vs. scored-and-retained, so the user can see why each sentence survived.

## Concrete Examples

**Example 1: Summarizing a government report for LLM context**

User: "Summarize this 40-page government report to fit in an 8k token context window while keeping the reasoning intact."

Approach:
1. Parse the report into ~320 sentences. Generate embeddings for each.
2. Build semantic graph with k=8 (log2(320) ~ 8.3, within bounds).
3. Run persistent homology. H0 reveals 5 core topic clusters (budget, infrastructure, education, healthcare, defense). H1 reveals 2 recurring argument cycles (budget <-> infrastructure dependency, education <-> healthcare workforce pipeline).
4. Protection pool: 18 sentences covering the 3 most persistent topic clusters and 2 argument cycles.
5. Score remaining 302 sentences. Prune iteratively to ~96 sentences (30% ratio).
6. Final summary: 96 sentences preserving the budget-infrastructure dependency chain and the education-healthcare pipeline — both critical for downstream Q&A.

Output:
```
## Summary (96 of 320 sentences retained, 70% compression)

### Protected Core Themes
- [Budget]: "The federal budget allocates $4.2T across..."
- [Infrastructure]: "Highway Trust Fund faces a $23B shortfall..."

### Protected Logical Dependencies
- [Budget <-> Infrastructure cycle]: "Infrastructure spending directly
  impacts revenue projections because..." → "...which in turn requires
  revised budget allocations for..."

### Supporting Detail (scored and retained)
- "The Department of Education reported a 12% increase..."
- "Medicare enrollment projections indicate..."
[...remaining sentences in document order...]
```

**Example 2: Compressing a research paper for quick review**

User: "Give me a structured summary of this ML paper, keeping the key claims and their supporting evidence connected."

Approach:
1. Parse the paper into ~150 sentences across Introduction, Related Work, Method, Experiments, Conclusion.
2. Use hierarchical strategy: compress each section independently first.
3. For the Method section (60 sentences): H0 identifies 3 core technique clusters (encoder architecture, loss function, training procedure). H1 identifies the feedback loop between loss and architecture design.
4. Protection pool locks 8 method sentences. Prune to 18 sentences (30%).
5. Global pass across all section summaries removes redundant claims restated in Introduction and Conclusion.
6. Final output: 40 sentences with clear method-to-evidence connections.

Output:
```
## Paper Summary (40 of 150 sentences, 73% compression)

**Claim**: "Our approach achieves SOTA on 3 benchmarks."
  ↳ Evidence (protected logical chain):
    "The encoder uses multi-scale attention..." →
    "Combined with the contrastive loss..." →
    "Table 2 shows +2.3 BLEU improvement..."

**Claim**: "Training is 4x faster than baseline."
  ↳ Evidence: "The sparse attention pattern reduces
    FLOPs from O(n²) to O(n log n)..."
```

**Example 3: Meeting transcript compression**

User: "Summarize this 2-hour meeting transcript, but don't lose the decision chains — I need to see how we arrived at each conclusion."

Approach:
1. Parse transcript into ~400 utterances. Treat each speaker turn as a "sentence."
2. Build graph with α=0.3 (lower semantic weight, higher positional weight — meetings have more temporal coherence than written text).
3. H1 cycles are especially valuable here: they capture "discussion loops" where the group revisits and refines a topic.
4. Protect all sentences in the top-3 discussion loops and top-5 topic clusters.
5. Prune to 25% while keeping decision chains intact.

Output:
```
## Meeting Summary (100 of 400 utterances, 75% compression)

### Decision Chain: API Migration
- [10:05] Alice: "We need to decide between REST and GraphQL..."
- [10:12] Bob: "GraphQL reduces over-fetching but adds complexity..."
- [10:30] Alice: "Given Bob's point, let's prototype both..."  ← cycle
- [10:45] Carol: "Prototype results show GraphQL is 40% fewer requests"
- [10:52] Alice: "Decision: proceed with GraphQL for v2."

### Decision Chain: Launch Timeline
- [11:15] Dave: "Q3 target depends on API migration..."
  [→ links to API Migration chain above]
```

## Best Practices

- **Do:** Adjust α (semantic vs. positional weight) based on document type. Use α=0.5 for structured writing (reports, papers). Use α=0.3 for temporal text (transcripts, logs) where position matters more. Use α=0.7 for loosely structured text (web pages, concatenated notes).
- **Do:** Increase K and M (protection pool sizes) for documents with many distinct topics. A 5-topic report needs K >= 5 to avoid dropping an entire theme.
- **Do:** Use the hierarchical strategy for documents over ~50 sentences. It enables parallelism and prevents the graph from becoming too dense to analyze meaningfully.
- **Do:** Present the protection pool contents to the user — it serves as a table of contents for the structural backbone and builds trust in the summary.
- **Avoid:** Blindly pruning to a fixed ratio without checking whether protected sentences alone already exceed the target length. If protection pool = 40% of the document and target = 30%, you must either reduce K/M or accept a longer summary.
- **Avoid:** Recomputing persistent homology after each pruning step. The entire point of the protection-pool-then-prune architecture is to pay the topology cost once. Recomputing defeats the efficiency gain.

## Error Handling

- **Document too short (<10 sentences):** Skip the full pipeline. Topological analysis on very small graphs produces degenerate results. Fall back to simple centroid-based ranking.
- **All sentences score similarly:** This happens with highly homogeneous text (e.g., repetitive logs). Increase α toward 1.0 to rely more on semantic differentiation, or switch to pure positional sampling.
- **Protection pool is too large (>50% of document):** Reduce K and M, or raise the persistence threshold — only protect features with persistence above the 75th percentile.
- **Disconnected graph components:** If the k-NN graph splits into isolated subgraphs, increase k or lower the similarity threshold. Disconnected components make shortest-path TopoScore infinite for cross-component pairs; handle by assigning a large finite penalty instead.
- **Missing sentence boundaries:** If the input lacks clear sentence breaks (e.g., OCR text, chat logs), pre-process with a sentence segmenter or fall back to fixed-length chunks of ~50 tokens.

## Limitations

- This approach is **extractive only** — it selects and orders existing sentences. It does not generate new bridging text or rephrase for conciseness. Pair with an abstractive step if the user needs a polished narrative.
- Persistent homology computation is **O(n log n)** for the initial pass, which is fast but not instant for very large documents (10,000+ sentences). The hierarchical strategy mitigates this.
- The method assumes **sentence-level granularity**. For code, structured data, or bullet-point lists, you need to define appropriate "atomic units" before applying the pipeline.
- **Query-dependent summarization** (TaskScore) requires a clear query or topic. For generic "summarize this" requests, substitute the document centroid embedding for the query vector.
- The technique preserves **structural coherence** but does not verify **factual accuracy**. Protected sentences are structurally important, not necessarily correct.

## Reference

Zhang, J., Zhang, C., Chen, S., Liu, Y., & Li, C. (2026). *Text summarization via global structure awareness.* arXiv:2602.09821v2. https://arxiv.org/abs/2602.09821v2

Key sections to reference: Section 3.3 (semantic graph construction and persistent homology), Section 3.4 (protection pool assembly), Section 3.5 (topology-guided iterative pruning with proxy metrics), and Section 3.6 (hierarchical compression strategy). Table 3 contains the hyperparameter sensitivity analysis showing optimal α=0.5, λ=0.7, K=3.