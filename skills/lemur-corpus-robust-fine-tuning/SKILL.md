---
name: "lemur-corpus-robust-fine-tuning"
description: "Build multilingual legal embedding models fine-tuned for semantic retrieval using contrastive objectives on parallel legislative corpora. Triggers: 'fine-tune legal embeddings', 'multilingual legal retrieval', 'EUR-Lex embedding pipeline', 'cross-lingual legal search', 'legal document retrieval model', 'PDF legal corpus to embeddings'"
---

This skill enables Claude to build end-to-end pipelines for constructing multilingual legal corpora from PDF legislative sources and fine-tuning embedding models for semantic retrieval. It applies the LEMUR methodology: scraping parallel legislative documents (e.g., from EUR-Lex), quantifying PDF-to-text extraction quality via a Lexical Content Score (LCS), constructing contrastive training pairs from metadata-document alignments, and fine-tuning multilingual encoders (E5-Multilingual, Qwen3) with Multiple Negatives Ranking (MNR) loss in monolingual or bilingual configurations.

## When to Use

- When the user wants to build a **semantic search system over multilingual legal documents** (EU legislation, national law, treaties)
- When the user needs to **fine-tune an embedding model for legal-domain retrieval** rather than relying on general-purpose embeddings
- When the user is working with **PDF-based legislative sources** and needs to assess or improve text extraction quality before training
- When the user wants **cross-lingual legal retrieval** (e.g., query in English, retrieve German or Latvian legislation)
- When the user asks to build a **contrastive training dataset from parallel document collections** (same law in multiple languages)
- When the user needs to improve retrieval for **low-resource EU languages** (Maltese, Irish, Latvian) by leveraging multilingual fine-tuning

## Key Technique

**LEMUR's core insight** is that parallel legislative corpora -- where the same law exists in multiple official languages -- provide natural alignment for contrastive training without manual annotation. Each legislative document has structured metadata (CELEX ID, title, subject matter, date) that serves as a synthetic query, while the document text serves as the positive passage. In-batch negatives from other documents in the same training batch provide the contrastive signal. This avoids expensive human relevance judgments entirely.

**The training uses symmetric Multiple Negatives Ranking (MNR) loss.** For a batch of N query-document pairs {(q_i, d_i)}, each pair is positive and every other document d_j (j != i) is an implicit negative. The loss maximizes cosine similarity between q_i and d_i relative to all other documents via temperature-scaled softmax. In the bilingual extension, all translations of the same law are treated as positives using a grouped multi-positive MNR objective, which teaches the model language-independent legal representations.

**PDF extraction quality matters.** The Lexical Content Score (LCS) computes bag-of-words cosine similarity between PDF-extracted text and authoritative HTML reference text. Languages with LCS below ~90% (Maltese at ~80%, Latvian at ~90%) show degraded retrieval. The pipeline uses LCS to filter or flag noisy documents before training, preventing the model from learning corrupted representations.

## Step-by-Step Workflow

1. **Collect parallel legislative documents from EUR-Lex.** Use the EUR-Lex SPARQL endpoint or CELLAR API to download PDF documents by category (e.g., CELEX directory code `15.10` for environment). Store documents indexed by `law_id` and `language_code`. Target documents available in at least 10+ languages for robust cross-lingual alignment.

2. **Extract text from PDFs using olmOCR.** Run olmOCR (or PyMuPDF as fallback) on each PDF page. Store per-page results as JSON with fields: `law_id`, `language`, `page_number`, `text`, `metadata`, `is_table`, `is_diagram`. Flag pages containing tables or diagrams for special handling.

3. **Compute Lexical Content Score (LCS) against HTML references.** For each document where an HTML version exists, tokenize both versions into bag-of-words frequency vectors, then compute cosine similarity. Filter out documents with LCS < 0.85 to remove noisy extractions. Log per-language LCS distributions to identify systematic extraction failures.

4. **Construct contrastive training pairs.** For each document, form a query from concatenated metadata fields (title + subject matter descriptors + CELEX ID) and pair it with the full document text as the positive passage. Write pairs as JSONL with fields: `query`, `positive`, `law_id`, `language`. For bilingual training, group all language versions of the same `law_id` as co-positives.

5. **Split data 60/20/20 by law_id** (not by individual passages) into train/validation/test sets. Ensure no law_id leaks across splits -- all language versions of a given law must stay in the same split.

6. **Configure the embedding model and tokenizer.** Load a pretrained multilingual encoder (E5-Multilingual-Large for 512-token contexts, or Qwen3-Embedding-0.6B for 2048-token contexts). Set max sequence length based on document length distribution. Apply truncation tracking: log what percentage of documents require truncation and how many tokens are lost.

7. **Fine-tune with MNR contrastive loss.** Train using bfloat16 precision with gradient checkpointing. Use a linear warmup schedule over the first 10% of steps. Train up to 30 epochs with early stopping on validation loss (patience of 3 epochs). For bilingual training, use the grouped multi-positive MNR variant where all translations share the positive label.

8. **Evaluate with Top-k retrieval accuracy.** Index all test documents into a vector store (ChromaDB or FAISS). For each test query, retrieve Top-1, Top-3, and Top-5 documents and measure exact-match accuracy against the known `law_id`. Report per-language and aggregate metrics.

9. **Run cross-lingual transfer evaluation.** Take the model fine-tuned on Language A and evaluate it on Language B without further training. Both queries and documents remain in Language B. Compare against the baseline (no fine-tuning) and the Language B-specific fine-tuned model to quantify transfer.

10. **Deploy as a retrieval service.** Wrap the fine-tuned model in a sentence-transformers-compatible API. Index the full corpus with FAISS or ChromaDB. Expose a `/search` endpoint accepting queries in any supported language with optional language filtering.

## Concrete Examples

**Example 1: Building a monolingual legal retrieval system for German EU law**

User: "I have 5,000 German EU environmental regulations as PDFs. I want to build a semantic search system so lawyers can find relevant legislation by describing their legal question."

Approach:
1. Extract text from PDFs using olmOCR, storing per-page JSON.
2. Download HTML versions from EUR-Lex for LCS validation. Compute LCS per document; discard those below 0.85.
3. Extract metadata (title, subject descriptors) from EUR-Lex API for each CELEX ID.
4. Create JSONL training pairs: `{"query": "Richtlinie uber Industrieemissionen - Umweltverschmutzung", "positive": "<full document text>"}`.
5. Split by law_id into 60/20/20.
6. Fine-tune E5-Multilingual-Large with MNR loss for up to 30 epochs.
7. Index test documents in ChromaDB and evaluate Top-5 accuracy.

Output:
```json
{
  "model": "e5-multilingual-large-legal-de",
  "test_metrics": {
    "top_1_accuracy": 0.91,
    "top_3_accuracy": 0.96,
    "top_5_accuracy": 0.98
  },
  "training_docs": 3000,
  "val_docs": 1000,
  "test_docs": 1000,
  "avg_lcs": 0.97
}
```

**Example 2: Cross-lingual retrieval -- query in English, retrieve Latvian legislation**

User: "Our legal team writes queries in English but needs to find matching Latvian regulations. Can we fine-tune an embedding model for this?"

Approach:
1. Collect parallel EN-LV document pairs from EUR-Lex (same law_id, both languages).
2. Extract text and compute LCS for both languages (expect ~97% EN, ~90% LV).
3. Build bilingual training pairs: for each law_id, the query is EN metadata, positives include both the EN and LV document texts.
4. Fine-tune using grouped multi-positive MNR loss -- all language versions of a law are co-positives.
5. Evaluate: embed English queries and Latvian documents separately, measure Top-k retrieval of correct LV documents given EN queries.

Output:
```json
{
  "model": "e5-multilingual-large-legal-en-lv",
  "cross_lingual_metrics": {
    "en_query_lv_doc_top1": 0.84,
    "en_query_lv_doc_top5": 0.94
  },
  "baseline_top1": 0.72,
  "improvement": "+12% Top-1 over unfine-tuned E5"
}
```

**Example 3: Assessing PDF extraction quality before training**

User: "I scraped 10,000 legislative PDFs from a national legal database. How do I know if the text quality is good enough to train on?"

Approach:
1. For a stratified sample (500 documents across languages/years), obtain HTML or XML reference versions.
2. Tokenize both PDF-extracted and reference texts into lowercased word frequency vectors, removing punctuation.
3. Compute LCS (cosine similarity of frequency vectors) per document.
4. Aggregate by language and decade. Flag languages with mean LCS < 0.90 and individual documents with LCS < 0.85.
5. Investigate low-LCS documents: check for OCR failures on scanned pages, table-heavy layouts, or non-Latin scripts.

Output:
```
LCS Quality Report:
  English:  mean=0.97, std=0.02, flagged=12/2000 docs
  German:   mean=0.96, std=0.03, flagged=18/2000 docs
  Latvian:  mean=0.90, std=0.07, flagged=89/1500 docs
  Maltese:  mean=0.81, std=0.11, flagged=203/800 docs

Recommendation: Exclude Maltese documents or apply manual
correction. Latvian is borderline -- expect 5-8% retrieval
accuracy degradation vs. high-LCS languages.
```

## Best Practices

- **Do:** Split data by `law_id`, never by page or passage. Leaking different pages of the same law across train/test inflates accuracy by 10-15%.
- **Do:** Compute LCS before training and set a quality threshold (0.85 minimum). Noisy text teaches the model to match noise patterns rather than legal semantics.
- **Do:** Use metadata-as-query rather than inventing synthetic queries. Legislative metadata (title + subject descriptors) closely mirrors how legal professionals actually search.
- **Do:** Track truncation statistics. If >20% of documents are truncated, consider a model with longer context (Qwen3 at 2048 tokens vs. E5 at 512).
- **Avoid:** Training on all 24 languages simultaneously in a single model without evaluation per language. Low-resource languages can be drowned out. Train bilingual pairs or small language groups instead.
- **Avoid:** Using general-purpose chunking strategies (e.g., 256-token sliding windows) for legal documents. Legal text has hierarchical structure (articles, paragraphs, annexes) that should guide segmentation.

## Error Handling

**PDF extraction produces empty or garbled text:** Check `is_table` and `is_diagram` flags. Pages that are entirely tabular or scanned images will have near-zero LCS. Fall back to dedicated table extraction (Camelot, Tabula) or OCR (Tesseract) for these pages.

**LCS computation fails due to missing HTML references:** For documents without HTML versions (common pre-2000), use cross-language consistency as a proxy: if the same law_id has high LCS in 20 languages but low LCS in one, that one language likely has extraction errors.

**Training loss plateaus early:** MNR loss with in-batch negatives is sensitive to batch size. If batch size is too small (<32), negatives are too easy. Increase batch size or add hard negatives mined from the top-k nearest non-matching documents.

**Cross-lingual retrieval underperforms for specific language pairs:** Check if both languages were well-represented in the base model's pretraining data. For truly low-resource languages (Maltese, Irish), bilingual fine-tuning with a high-resource anchor (English) provides 10%+ gains over monolingual fine-tuning alone.

**Out-of-memory during fine-tuning:** Enable gradient checkpointing and switch to bfloat16. For Qwen3-4B, an 80GB A100 is required. E5-Multilingual fits on a 48GB A6000.

## Limitations

- The approach assumes **parallel document availability** -- the same law in multiple languages. National-only legislation without translations cannot leverage the bilingual training objective.
- **Domain transfer is narrow.** Models fine-tuned on EU environmental law improve on other EU law domains but do not necessarily transfer to common law jurisdictions, case law, or contract analysis.
- **512-token limits** (E5-Multilingual) force truncation of 8-15% of legal documents, losing 40-50% of tokens in affected documents. This disproportionately impacts long annexes and technical regulations.
- The LCS metric uses bag-of-words and is **insensitive to word order errors** in extraction. A document with correct vocabulary but scrambled sentences will score high LCS but produce poor training signal.
- **Evaluation uses exact law_id matching.** In practice, multiple laws may be relevant to a query. The methodology does not capture partial relevance or ranking quality beyond Top-k hit rate.

## Reference

[LEMUR: A Corpus for Robust Fine-Tuning of Multilingual Law Embedding Models for Retrieval](https://arxiv.org/abs/2602.09570v1) -- Ahmadi et al., EACL SRW 2026. Focus on Section 4 (contrastive training setup and grouped multi-positive MNR loss) and Section 5.2 (cross-lingual transfer results showing language-independent representation learning). Code: [github.com/nargesbh/eur_lex](https://github.com/nargesbh/eur_lex). Dataset: [huggingface.co/datasets/G4KMU/LEMUR](https://huggingface.co/datasets/G4KMU/LEMUR).