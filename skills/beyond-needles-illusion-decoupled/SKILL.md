---
name: "beyond-needles-illusion-decoupled"
description: "Decouple evidence access from evidence use when evaluating or building long-context and RAG systems under semantic interference. Use this skill when the user says: 'evaluate my RAG pipeline against hard negatives', 'stress-test retrieval with semantic distractors', 'build a decoupled retrieval benchmark', 'diagnose why my long-context QA is failing', 'create collision-tested evaluation data', 'measure evidence access vs answer quality separately'."
---

# Decoupled Evidence Evaluation Under Semantic Interference

This skill teaches Claude to apply the EverMemBench-S methodology: evaluating and hardening long-context LLM and RAG systems by **decoupling evidence access from evidence use** and introducing **collision-tested semantic interference**. Standard Needle-in-a-Haystack (NIAH) benchmarks use near-unique needles in irrelevant haystacks, producing inflated scores that collapse in production where documents overlap, paraphrase each other, and partially satisfy queries while violating key constraints. This skill enables you to build adversarial evaluation harnesses that expose the real bottleneck -- semantic discrimination, not context length.

## When to Use

- When a user wants to evaluate a RAG pipeline and needs harder-than-standard test cases with realistic distractors
- When a user's long-context QA system passes NIAH benchmarks but fails on real workloads, and they need to diagnose whether the problem is retrieval (access) or reasoning (use)
- When building an evaluation corpus that scales from small isolated contexts (64K) to large shared environments (millions of tokens) to measure degradation curves
- When constructing hard-negative test sets where distractors are semantically similar to gold evidence but don't actually support the answer
- When a user asks to stress-test a retriever or reranker against near-miss documents
- When designing multi-source queries that require aggregating evidence from multiple documents, exposing brittleness hidden by single-source recall

## Key Technique

**The core insight**: Separate the measurement of *can the system find the right evidence?* (evidence access) from *can it use that evidence to answer correctly?* (evidence use). Most benchmarks conflate these. A system might retrieve the right document but hallucinate the answer, or it might fail to retrieve but guess correctly from world knowledge. Decoupling reveals which stage is broken.

**Semantic interference** is what makes this adversarial. Instead of padding context with unrelated text, you inject *collision-tested hard negatives* -- documents that are semantically close to the gold evidence (same domain, overlapping entities, similar language) but do not actually support the answer. This forces the system to discriminate between near-miss and true-match documents. The construction uses embedding-based retrieval to find candidate distractors, then LLM verification to classify each as a hard negative (similar but unsupportive), a conflict (contradicts the answer -- discard), or a false negative (actually valid evidence -- add to gold set).

**The reference-corpus ladder** systematically increases interference. Start with domain-isolated contexts where gold documents have minimal competition. Then mix in cross-domain documents. Then merge all domains into a shared corpus. Finally inject global distractors from the full document pool. At each rung, measure both access metrics (Recall@K, full-set Recall@K for multi-source queries) and QA quality separately. The degradation curve reveals the scale at which your system breaks.

## Step-by-Step Workflow

1. **Curate a document pool with domain labels.** Collect your corpus and assign each document a domain tag (e.g., finance, medical, legal). Each document gets a unique `DocID`. For native long-context evaluation, prepend `[DocID=<id>]` headers so models can output IDs directly.

2. **Build (Query, Answer, RefDocSet) tuples.** For each evaluation query, identify the minimal set of reference documents that together contain sufficient evidence. Multi-source queries (requiring 2+ documents) are critical -- they expose brittleness that single-source queries miss.

3. **Mine hard negatives via embedding retrieval.** For each query, retrieve the top-K nearest non-reference documents using a dense embedder (e.g., any sentence-transformer or embedding API). These candidates are semantically close to the query but not in the gold set.

4. **Classify candidates with LLM verification.** For each retrieved candidate, prompt an LLM to classify it into one of three categories:
   - **Hard Negative**: Semantically similar but does not support the answer. Keep as a distractor.
   - **Conflict**: Contradicts the query or answer. Discard the entire sample to avoid ambiguity.
   - **False Negative**: Actually provides valid evidence. Add it to the gold RefDocSet.

5. **Construct the reference-corpus ladder.** Build evaluation contexts at increasing scales:
   - **Tier 1 (small, domain-isolated):** Gold documents + minimal same-domain padding.
   - **Tier 2 (medium, cross-domain):** Add documents from other domains.
   - **Tier 3 (large, shared corpus):** Merge all domain pools, deduplicate.
   - **Tier 4 (full scale):** Inject remaining hard negatives and global distractors.
   Use fixed random seeds at each tier for reproducibility.

6. **Measure evidence access independently.** Have the system output its top-K retrieved document IDs (or, for native long-context models, extract integer DocIDs from output). Score with:
   - **R@1**: Recall on single-source queries (did it find the one document?).
   - **SR@10**: Standard recall@10 across all queries.
   - **FR@10**: Full-set recall@10 -- requires ALL reference documents retrieved. This is the most diagnostic metric for multi-source queries.

7. **Measure evidence use independently.** Feed the system the full context (or retrieved passages) and score the generated answer on accuracy, completeness, and relevance (0-5 scale via LLM-as-judge or human evaluation). Compare this against an oracle condition where gold documents are provided directly.

8. **Compute the access-use gap.** For each corpus tier, compare evidence access scores against QA quality. A high access score with low QA score indicates a reasoning failure. A low access score with high QA score suggests reliance on parametric knowledge (fragile). Both metrics declining together confirms retrieval is the bottleneck.

9. **Track the multi-source gap.** At each tier, compute `SR@10 - FR@10`. This gap widens as interference increases and directly measures how well the system handles evidence aggregation across documents.

10. **Iterate on the weakest link.** If access degrades first: improve retriever discrimination (reranking, hard-negative fine-tuning, hybrid search). If use degrades first: improve downstream reasoning (better prompting, chain-of-thought, answer verification). If both degrade: prioritize access -- you cannot reason over evidence you never found.

## Concrete Examples

**Example 1: Diagnosing a RAG pipeline that passes NIAH but fails in production**

```
User: My RAG system gets 95% on NIAH but users report wrong answers
on our medical knowledge base. Help me figure out what's breaking.

Approach:
1. Sample 50 real user queries that produced wrong answers.
2. For each query, identify the gold reference documents in the corpus.
3. Run embedding retrieval (top-50) against the full corpus for each query.
4. Classify non-gold retrieved docs with an LLM:
   - "Given query Q, answer A, and reference docs R, does document D
     support the answer? Classify as: hard_negative, conflict, or false_negative."
5. Build three evaluation tiers:
   - Tier 1: Gold docs + 10 random unrelated docs (easy baseline)
   - Tier 2: Gold docs + 10 hard negatives (semantic interference)
   - Tier 3: Gold docs + 10 hard negatives + 30 random docs (scale + interference)
6. At each tier, measure:
   - Access: Did the retriever rank gold docs in top-10? (SR@10)
   - Access (strict): Did it find ALL gold docs? (FR@10)
   - Use: Score generated answers 0-5 with LLM-as-judge.

Output (example diagnostic table):
| Tier          | SR@10 | FR@10 | Avg QA Score |
|---------------|-------|-------|--------------|
| Easy baseline | 0.96  | 0.88  | 4.2          |
| Hard negatives| 0.71  | 0.43  | 2.8          |
| Full scale    | 0.65  | 0.31  | 2.4          |

Diagnosis: FR@10 drops 57 points from easy to full scale. The retriever
cannot discriminate gold docs from hard negatives, especially for
multi-source queries. QA tracks access -- this is a retrieval problem.
Recommendation: Fine-tune the retriever with these hard negatives as
training signal, or add a reranker stage.
```

**Example 2: Building a collision-tested evaluation set from scratch**

```
User: I have 10,000 documents in my knowledge base and 200 QA pairs.
Create an adversarial evaluation set with hard negatives.

Approach:
1. Embed all 10,000 documents using a dense encoder.
2. For each of the 200 QA pairs:
   a. Retrieve top-20 non-reference documents by cosine similarity.
   b. For each candidate, call LLM classifier:
      Prompt: "Query: {q}\nAnswer: {a}\nReference: {ref_summary}\n
      Candidate: {candidate_text}\n
      Classify this candidate as:
      - HARD_NEGATIVE: Topically similar but does not support the answer
      - CONFLICT: Contradicts the answer or contains incompatible info
      - FALSE_NEGATIVE: Actually supports the answer (missed gold doc)
      Output only the label and a one-sentence reason."
   c. Keep HARD_NEGATIVEs, discard CONFLICTs (and their queries),
      promote FALSE_NEGATIVEs to gold set.
3. Build corpus ladder:
   - 64K tier: gold docs + top-5 hard negatives per query
   - 256K tier: add cross-domain random documents
   - Full corpus: all 10,000 documents
4. Output: eval_set.jsonl with fields:
   {query, answer, gold_doc_ids, hard_negative_ids, tier, corpus_doc_ids}

Output:
- 200 queries -> 183 retained (17 had conflict-only candidates)
- 14 false negatives discovered and added to gold sets
- Average 8.3 hard negatives per query
- eval_set.jsonl ready for three-tier evaluation
```

**Example 3: Measuring a long-context model's semantic discrimination**

```
User: I want to test whether GPT-4-turbo or Claude can actually find
specific evidence in a 128K context with distractors, not just pass NIAH.

Approach:
1. Select 30 multi-source queries (each requiring 2-3 gold documents).
2. For each query, construct a 128K-token context:
   - Gold documents with [DocID=N] headers
   - 15 collision-tested hard negatives with their own DocIDs
   - Random padding documents to fill to 128K
   - Shuffle document order randomly
3. Prompt format for access measurement:
   "Given the documents above, which document IDs contain evidence
   needed to answer: {query}? Output up to 10 DocIDs as integers."
4. Prompt format for use measurement:
   "Given the documents above, answer: {query}"
5. Score:
   - Access: Parse integer IDs from output, compute SR@10 and FR@10
   - Use: LLM-as-judge scores answer 0-5

Output:
| Model         | SR@10 | FR@10 | QA Score | Access-Use Gap |
|---------------|-------|-------|----------|----------------|
| Model A       | 0.82  | 0.47  | 3.1      | Retrieval-bound |
| Model B       | 0.78  | 0.52  | 3.8      | Balanced        |

Model A finds individual docs but misses multi-source sets (low FR@10).
Model B retrieves better multi-source sets and reasons more faithfully.
```

## Best Practices

- **Do** always include multi-source queries (requiring 2+ documents). Single-source recall hides the most important failure mode -- systems often find one relevant doc but miss the second or third.
- **Do** use FR@10 (full-set recall) as your primary access metric. Standard recall is insufficient because partial retrieval of a multi-doc evidence set still produces wrong answers.
- **Do** validate hard negatives with LLM classification. Unverified distractors may include actual evidence (false negatives) or contradictions, both of which corrupt your benchmark.
- **Do** fix random seeds when constructing corpus tiers so results are reproducible across runs and models.
- **Avoid** using only unrelated documents as distractors. This recreates the benign NIAH illusion where high scores don't predict real-world performance.
- **Avoid** reporting only end-to-end QA scores. Without decoupled access metrics, you cannot tell whether a failure is from bad retrieval or bad reasoning, and you'll waste effort optimizing the wrong component.

## Error Handling

- **Ambiguous LLM classifications**: When the classifier is uncertain between hard_negative and false_negative, default to false_negative (add to gold set). This is conservative -- it avoids penalizing correct retrieval.
- **No hard negatives found**: If embedding retrieval returns only obviously unrelated documents, the query may be too domain-specific. Try broadening the retrieval pool or lowering the similarity threshold. If no plausible distractors exist, the query is not suitable for adversarial evaluation.
- **Conflict-heavy queries**: If >50% of candidates for a query are classified as conflicts, the query or answer may be ambiguous. Discard the sample rather than risk an unreliable benchmark item.
- **DocID extraction failures**: When native LLMs output free-text instead of integer IDs, use regex extraction (`\b\d+\b`) and take the first K unique matches. If extraction yields zero IDs, score as zero recall rather than skipping the sample.
- **Context overflow**: If your corpus tier exceeds the model's context window, truncate by removing random (non-gold, non-hard-negative) padding documents first. Never truncate gold or hard-negative documents.

## Limitations

- **Hard-negative quality depends on the embedding model.** If your embedder is weak, retrieved candidates may not be semantically close enough to create genuine interference. Use the strongest available embedder for mining.
- **LLM-as-judge scoring is noisy.** QA scores on a 0-5 scale have inter-rater variance. Use multiple judge calls or consensus voting for high-stakes evaluations.
- **Does not test temporal or causal reasoning directly.** The decoupled protocol measures access and factual use, but not whether the system correctly handles time-dependent or causal relationships in evidence.
- **Corpus ladder construction requires substantial compute.** Embedding 100K+ documents, running LLM classification on thousands of candidates, and building multiple tiers is resource-intensive. Budget accordingly.
- **Multi-source query creation is labor-intensive.** Automatically generating queries that genuinely require multiple documents (not just benefit from them) remains an open challenge. Manual curation or careful LLM-assisted synthesis with human validation is recommended.

## Reference

- **Paper**: [Beyond the Needle's Illusion: Decoupled Evaluation of Evidence Access and Use under Semantic Interference at 326M-Token Scale](https://arxiv.org/abs/2601.20276v1) -- Look for Section 3 (MemoryBank construction and three-stage query pipeline), Section 4 (reference-corpus ladder), and Table 2/3 (degradation curves across tiers).