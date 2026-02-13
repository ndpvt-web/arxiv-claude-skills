---
name: "aiano-enhancing-information-retrieval"
description: "Build AI-augmented annotation pipelines for creating high-quality information retrieval and QA datasets. Combines LLM-generated suggestions (questions, passage relevance scores, answer spans) with human review workflows to accelerate dataset creation. Use when: 'build an annotation pipeline for IR data', 'create a QA dataset from documents', 'annotate passages for retrieval training', 'generate question-answer pairs from a corpus', 'build a human-in-the-loop labeling tool for search', 'set up AI-assisted relevance annotation'."
---

This skill teaches Claude to build AI-augmented annotation systems for information retrieval dataset creation, based on the AIANO methodology. The core idea: instead of annotators writing questions and judging relevance from scratch, an LLM generates candidate questions, retrieves relevant passages via embeddings, and proposes relevance scores -- then a human annotator accepts, edits, or rejects each suggestion. This human-AI loop nearly doubles annotation throughput while improving retrieval accuracy compared to manual-only annotation.

## When to Use

- When the user needs to create a question-answering dataset from a document corpus (e.g., PDFs, knowledge bases, technical docs)
- When building a relevance annotation pipeline for training or evaluating a retrieval/RAG system
- When the user wants to generate synthetic QA pairs from documents and needs a human review step for quality control
- When setting up a labeling workflow where annotators judge query-passage relevance with AI pre-annotations
- When the user asks to build tooling that accelerates manual dataset curation with LLM assistance
- When evaluating or improving an existing retrieval system by creating gold-standard test sets

## Key Technique

AIANO's central insight is that **the annotation bottleneck in IR dataset creation is not judgment quality but annotation initiation** -- starting from a blank slate is slow. By having an LLM generate the first draft (candidate questions, suggested relevant passages, initial relevance labels), annotators shift from *creation* mode to *review-and-refine* mode, which is cognitively easier and substantially faster.

The system operates on a three-stage AI-augmented loop per document chunk: (1) **Question Generation** -- an LLM reads a passage and proposes natural-language questions that the passage could answer; (2) **Passage Retrieval** -- dense embeddings index the corpus and retrieve candidate passages for each generated question, ranked by semantic similarity; (3) **Relevance Annotation** -- the LLM scores each query-passage pair for relevance (e.g., 0-3 scale), and these pre-annotations are presented to the human annotator for confirmation or correction.

What makes this work in practice is the **confidence-gated suggestion interface**: AI suggestions come with confidence indicators, so annotators can quickly accept high-confidence items and focus effort on ambiguous cases. The human retains full override authority -- they can edit questions, reject irrelevant passages, or re-score relevance. This preserves dataset quality while cutting annotation time roughly in half.

## Step-by-Step Workflow

1. **Ingest and chunk the document corpus.** Parse source documents (PDF, HTML, markdown) into clean text. Split into passages using structure-aware chunking -- respect paragraph, section, and heading boundaries rather than fixed token windows. Target 100-300 tokens per chunk for retrieval-friendly granularity.

2. **Generate dense embeddings for all passages.** Use a sentence-transformer model (e.g., `all-MiniLM-L6-v2` or `bge-base-en-v1.5`) to embed each passage. Store embeddings in a vector index (FAISS, ChromaDB, or a simple NumPy array for small corpora) alongside passage metadata (source document, chunk index, section title).

3. **Generate candidate questions per passage.** For each passage, prompt an LLM with the passage text and ask it to produce 2-5 natural questions that the passage answers. Instruct the LLM to vary question types: factoid, definitional, comparative, and reasoning. Include the passage context (section title, surrounding text) to improve question quality.

4. **Retrieve candidate passages for each generated question.** Embed each candidate question using the same embedding model. Query the vector index to retrieve the top-k (typically k=5-10) most similar passages. This step identifies which passages in the corpus are relevant to each question, including passages beyond the one that generated the question.

5. **Score query-passage relevance with the LLM.** For each (question, passage) pair from step 4, prompt the LLM to assign a relevance grade on a defined scale (e.g., 0=irrelevant, 1=marginally relevant, 2=relevant, 3=highly relevant). Include the grading rubric in the prompt. Attach a confidence estimate (the LLM can self-report confidence, or you can use log-probabilities if available).

6. **Structure pre-annotations into a reviewable format.** Organize each annotation unit as: `{question, passage_text, passage_id, ai_relevance_score, ai_confidence, source_document}`. Sort by confidence descending so annotators see high-confidence items first. Output as JSON, CSV, or feed into a review UI.

7. **Present pre-annotations for human review.** Build or use a review interface where annotators see the AI-proposed question, the candidate passage, and the suggested relevance score. Provide controls to: accept as-is, edit the question text, override the relevance score, reject the pair entirely, or flag for discussion.

8. **Collect and reconcile annotations.** Store human decisions alongside AI suggestions. Track acceptance rate, edit rate, and rejection rate per annotator -- these metrics indicate where the AI model is weak and where annotators disagree.

9. **Export the final dataset.** Produce the gold-standard dataset in a standard IR evaluation format: queries with associated relevant passage IDs and relevance grades. Common formats include TREC qrels, BEIR-compatible JSON, or HuggingFace Datasets.

10. **Iterate: use annotation feedback to improve suggestions.** Analyze rejection and edit patterns. Fine-tune the question generation prompt or re-rank retrieval results based on annotator corrections. Each annotation round should yield better AI suggestions for the next batch.

## Concrete Examples

**Example 1: Creating a QA dataset from technical documentation**

```
User: I have 200 markdown files of API documentation. I need to create a QA
evaluation dataset with ~500 question-answer pairs to test our RAG pipeline.

Approach:
1. Parse all 200 markdown files, split by heading sections into passages
   (~100-250 tokens each). Preserve heading hierarchy as metadata.
2. Embed all passages with sentence-transformers and index in FAISS.
3. For each passage, prompt the LLM:
   "Given this API documentation passage, generate 3 realistic questions
   a developer might ask that this passage answers. Vary between
   'how-to', 'what-is', and 'troubleshooting' question types."
4. For each generated question, retrieve top-5 passages by embedding similarity.
5. Score each (question, passage) pair for relevance (0-3 scale).
6. Output a review file:

   review_batch_001.jsonl:
   {"id": "q001", "question": "How do I authenticate with OAuth2?",
    "passage": "Authentication requires an OAuth2 bearer token...",
    "passage_id": "auth-guide-chunk-04",
    "ai_relevance": 3, "ai_confidence": 0.95,
    "status": "pending_review"}

7. Annotator reviews, accepts 80%, edits 15% of questions for clarity,
   rejects 5% as duplicates or nonsensical.
8. Export final dataset as BEIR-format JSON with queries.jsonl and qrels.tsv.
```

**Example 2: Building a relevance annotation tool for medical literature**

```
User: We need to annotate 1000 PubMed abstracts for relevance to 50 clinical
questions. Our domain experts are expensive -- how can we speed this up?

Approach:
1. Chunk each abstract into title + full-abstract passages (medical abstracts
   are short enough to treat as single passages).
2. Embed all 1000 abstracts with a biomedical embedding model (e.g., PubMedBERT).
3. For each of the 50 clinical questions, retrieve top-20 abstracts by similarity.
4. Prompt the LLM with each (question, abstract) pair:
   "You are a medical information specialist. Rate the relevance of this
   abstract to the clinical question on a 0-3 scale:
   0 = not relevant, 1 = marginally relevant (background only),
   2 = relevant (partially answers), 3 = highly relevant (directly answers).
   Also state your confidence (low/medium/high) and a one-sentence rationale."
5. Generate a review spreadsheet sorted by question, then by AI confidence:

   | Question | Abstract ID | AI Score | Confidence | Rationale | Human Score |
   |----------|-------------|----------|------------|-----------|-------------|
   | Q12      | PMC8834521  | 3        | high       | Directly reports trial results for... | [pending] |
   | Q12      | PMC7721003  | 1        | medium     | Mentions the condition but focuses on... | [pending] |

6. Domain experts review: high-confidence items (~60%) take ~10 seconds each
   (quick accept/reject). Low-confidence items get careful reading.
7. Result: 1000 relevance judgments completed in ~40% of fully-manual time.
```

**Example 3: Generating a synthetic QA training set with quality filtering**

```
User: I want to generate training data for a retrieval model from our internal
wiki. No human reviewers available -- can we still use this approach?

Approach:
1. Chunk wiki pages into passages. Embed and index as usual.
2. Generate 3 questions per passage using the LLM.
3. Retrieve top-5 passages per question and score relevance.
4. Without human review, apply automated quality filters as a substitute:
   - Discard questions where AI confidence < 0.7
   - Discard questions that are too similar to each other (cosine similarity
     of question embeddings > 0.9)
   - Discard (question, passage) pairs where the top-1 retrieved passage
     is NOT the source passage (indicates the question is ambiguous)
   - Keep only pairs with AI relevance score >= 2
5. Output a silver-standard dataset. Flag it as AI-generated (not gold).
6. Use this for initial retrieval model training; create a small gold-standard
   subset with human review later for evaluation.

Note: This degraded-mode workflow trades quality for speed. The dataset will
have noise (~10-15% incorrect labels). Suitable for training, not evaluation.
```

## Best Practices

- **Do:** Include the passage's surrounding context (section title, adjacent passages) when prompting for question generation. Context prevents questions that are too narrow or ambiguous.
- **Do:** Use a grading rubric in relevance scoring prompts. Without explicit criteria, LLM relevance scores drift across sessions and document types.
- **Do:** Sort review items by AI confidence descending. High-confidence items are fast to verify, building annotator momentum and catching the AI's confident mistakes early.
- **Do:** Track inter-annotator agreement on a sample of overlapping items. If two annotators disagree frequently on AI-suggested scores, the rubric needs refinement.
- **Avoid:** Generating questions from passages that are too short (< 30 tokens) or purely structural (table of contents, headers-only). These produce low-quality questions.
- **Avoid:** Using fixed-size chunking (e.g., every 512 tokens). Splitting mid-sentence or mid-paragraph degrades both question generation and retrieval quality. Always chunk on structural boundaries.
- **Avoid:** Skipping the human review step entirely for evaluation datasets. AI-generated relevance labels have systematic biases (tendency toward higher scores, inability to detect subtle factual errors) that corrupt evaluation metrics.

## Error Handling

- **LLM generates nonsensical or repetitive questions:** Add diversity instructions to the prompt ("generate questions of different types"). Implement post-hoc deduplication by comparing question embeddings and discarding pairs with cosine similarity > 0.85.
- **Retrieval returns only irrelevant passages:** The embedding model may not suit the domain. Try a domain-specific model (e.g., `BAAI/bge-base-en-v1.5` for general, `pritamdeka/S-PubMedBert-MS-MARCO` for biomedical). Also check that chunk sizes are appropriate -- very long chunks dilute embedding quality.
- **AI relevance scores cluster at extremes (all 0 or all 3):** The scoring prompt likely lacks calibration examples. Add 2-3 few-shot examples in the prompt showing each score level with rationale.
- **Annotators disagree with AI suggestions >50% of the time:** The AI suggestions are not saving time. Re-evaluate the LLM model, prompt design, or domain fit. Consider using a stronger model for suggestions or narrowing the annotation task scope.
- **Corpus is too large for full embedding:** Process in batches. Use approximate nearest neighbor search (FAISS IVF or HNSW) rather than brute-force. For corpora >1M passages, consider a two-stage retrieval: BM25 first-pass then re-rank with embeddings.

## Limitations

- This approach assumes the source documents contain sufficient information to generate meaningful questions. Highly structured data (spreadsheets, code-only repos) may not produce good natural-language QA pairs without additional context.
- LLM-generated questions tend toward factoid and definitional types. Complex reasoning questions, multi-hop questions, and questions requiring cross-document synthesis are underrepresented and often need manual creation.
- The quality of AI pre-annotations depends heavily on the LLM's domain knowledge. For specialized domains (legal, medical, financial), general-purpose LLMs may produce plausible-sounding but technically incorrect suggestions that require expert scrutiny.
- The speed gains (roughly 2x) assume annotators trust and engage with AI suggestions. If annotators distrust the AI and re-verify everything from scratch, the pre-annotation step adds overhead rather than saving time.
- This workflow creates datasets for **passage-level** retrieval evaluation. It does not directly address answer extraction, multi-turn dialogue, or generative QA evaluation without adaptation.

## Reference

- **Paper:** [AIANO: Enhancing Information Retrieval with AI-Augmented Annotation](https://arxiv.org/abs/2602.04579v1) (Khattab et al., 2026). Key sections: the three-stage annotation workflow (question generation, passage retrieval, relevance scoring), the within-subject user study design showing 2x speedup, and the analysis of when AI suggestions help vs. hinder annotator performance.