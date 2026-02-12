---
name: "lemur-corpus-robust-fine-tuning"
description: "Build multilingual legal document retrieval systems by fine-tuning embedding models on domain-specific corpora with contrastive learning. Applies the LEMUR pipeline: PDF-to-text extraction with quality scoring, metadata-to-document pair construction, and Multiple Negatives Ranking (MNR) loss fine-tuning. Triggers: 'fine-tune embeddings for legal retrieval', 'build multilingual legal search', 'extract text from legal PDFs and measure quality', 'contrastive fine-tuning for domain-specific retrieval', 'cross-lingual legal document search', 'adapt embedding models for low-resource legal languages'."
---

# LEMUR: Multilingual Legal Embedding Fine-Tuning for Retrieval

This skill enables Claude to build end-to-end multilingual legal document retrieval systems by applying the LEMUR methodology: constructing domain-specific corpora from legislative PDF sources, measuring extraction quality with the Lexical Content Score (LCS), generating metadata-to-document training pairs, and fine-tuning multilingual embedding models (E5-Multilingual, Qwen3-0.6B, Qwen3-4B) using symmetric Multiple Negatives Ranking loss. The approach yields 8-12 percentage point Top-1 retrieval gains, with especially strong improvements for low-resource languages and cross-lingual transfer to unseen languages.

## When to Use

- When the user needs to build a semantic search system over multilingual legal documents (legislation, regulations, case law)
- When the user wants to fine-tune a multilingual embedding model (E5, BGE-M3, Qwen) for domain-specific retrieval rather than using generic embeddings
- When the user has a collection of legislative PDFs and needs to extract, clean, and validate text quality before indexing
- When the user asks about contrastive fine-tuning with metadata-as-query and document-as-target pair construction
- When the user needs cross-lingual retrieval where a query in one language retrieves documents in another
- When the user wants to measure PDF-to-text extraction fidelity using a reference corpus (HTML vs PDF comparison)
- When the user is working with EUR-Lex, national legislation databases, or any structured legal document collection

## Key Technique

**Corpus Construction with Quality Assurance.** LEMUR addresses a fundamental problem: legal PDFs produce noisy text. The pipeline uses olmOCR to convert PDFs to structured JSONL preserving table layouts in markdown. Quality is measured by the Lexical Content Score (LCS) — cosine similarity between bag-of-words vectors of PDF-extracted text and authoritative HTML reference text. Documents scoring below threshold are flagged for re-extraction or exclusion. High-resource languages achieve ~95% LCS; low-resource languages ~80-90%. This quality gate prevents training on garbage text.

**Metadata-to-Document Contrastive Training.** Each legislative act is split into a metadata block (act type, date, subject description, legal basis references) and the substantive body text. The metadata block serves as the query; the body serves as the positive document. This mirrors real legal search behavior where practitioners search using partial structured information. Training uses symmetric Multiple Negatives Ranking (MNR) loss: given a batch of (query, document) pairs, all other documents in the batch serve as in-batch negatives. The loss is `L = -1/2B * sum(log(exp(s_ii) / sum_j(exp(s_ij))) + log(exp(s_ii) / sum_j(exp(s_ji))))` where `s_ij` is temperature-scaled cosine similarity.

**Bilingual Extension for Cross-Lingual Transfer.** For cross-lingual retrieval, the bilingual variant treats all aligned translations of the same act as joint positives using grouped multi-positive MNR loss. This teaches the model that semantically identical content in different languages should cluster together. Results show fine-tuning primarily enhances language-independent legal concept representations rather than language-specific cues, enabling zero-shot transfer to unseen languages.

## Step-by-Step Workflow

1. **Extract text from legal PDFs using olmOCR.** Convert each PDF page to structured JSONL with fields: `law_id`, `language`, `page_number`, `metadata`, `text`, `is_table`, `is_diagram`. Preserve table structures as markdown. Average legal document runs ~19 pages with ~403 tokens/page.

2. **Compute Lexical Content Score (LCS) against reference text.** For each document, tokenize both the PDF-extracted text and an HTML reference version into bag-of-words vectors. Compute cosine similarity: `LCS = dot(v_html, v_pdf) / (norm(v_html) * norm(v_pdf))`. Flag documents with LCS below 0.80 for manual review or re-extraction. Log per-language LCS distributions to identify systematic extraction failures.

3. **Segment each document into metadata query and body text.** Extract the introductory metadata block (act type, date, subject, legal basis, publication notes) as the retrieval query. Concatenate remaining pages as the positive document. This yields one (query, document) pair per legislative act per language.

4. **Split data 60/20/20 into train/validation/test per language.** Ensure the split is by `law_id` so the same act never appears in both train and test. For bilingual training, align the same act across language pairs.

5. **Select and load a pretrained multilingual embedding model.** Choose based on resource constraints: E5-Multilingual (fast, 512-token limit, 20-30 min/language training), Qwen3-0.6B (2048-token limit, 2-4 hrs), or Qwen3-4B (2048-token limit, 6-8 hrs). Load with bfloat16 precision and gradient checkpointing enabled.

6. **Configure the MNR contrastive training loop.** Set up symmetric Multiple Negatives Ranking loss. Use L2-normalized embeddings with cosine similarity scoring. Enable linear warm-up schedule. Train for up to 30 epochs with early stopping on validation loss. For bilingual mode, use grouped multi-positive MNR where aligned translations share the positive label.

7. **Fine-tune the model.** Feed batches of (metadata_query, document_text) pairs. In-batch negatives provide the contrastive signal — no explicit hard negative mining is needed. Truncate inputs exceeding model sequence limits (512 for E5, 2048 for Qwen). Monitor validation loss for early stopping.

8. **Index documents with ChromaDB using the fine-tuned model.** Embed all document texts with the fine-tuned model, L2-normalize, and insert into a ChromaDB collection with cosine similarity distance. Store `law_id`, `language`, and `year` as metadata for filtered retrieval.

9. **Evaluate with Top-k retrieval accuracy.** For each metadata query in the test set, retrieve Top-1, Top-3, and Top-5 documents. Compute accuracy as the fraction where the correct document appears in the Top-k results. Compare against the base (non-fine-tuned) model to measure improvement. Expected gains: 8-12% absolute Top-1 improvement for high-resource languages, 10-15% for low-resource languages.

10. **Run cross-lingual evaluation on held-out languages.** Test retrieval where query language differs from document language, or evaluate on languages not seen during fine-tuning. This validates that the model learned domain-level legal representations rather than language-specific patterns.

## Concrete Examples

**Example 1: Fine-tuning E5-Multilingual for EU legal retrieval**

User: "I have a collection of EU environmental legislation PDFs in English, German, and French. I want to build a semantic search system where lawyers can find relevant legislation by describing what they're looking for."

Approach:
1. Extract text from PDFs using olmOCR, producing JSONL with page-level text and metadata
2. Validate extraction quality by computing LCS against EUR-Lex HTML versions
3. Segment each act into metadata query + body document pairs
4. Split 60/20/20 by law_id per language

```python
from sentence_transformers import SentenceTransformer, InputExample, losses
from torch.utils.data import DataLoader
import json

# Load training pairs
train_pairs = []
with open("train_pairs.jsonl") as f:
    for line in f:
        item = json.loads(line)
        train_pairs.append(InputExample(
            texts=[item["metadata_query"], item["document_text"]]
        ))

# Load pretrained model
model = SentenceTransformer("intfloat/multilingual-e5-large")

# Configure MNR loss (symmetric in-batch negatives)
train_dataloader = DataLoader(train_pairs, shuffle=True, batch_size=32)
train_loss = losses.MultipleNegativesRankingLoss(model=model)

# Fine-tune with early stopping
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=30,
    warmup_steps=100,
    evaluation_steps=500,
    output_path="./lemur-e5-legal-en-de-fr",
    use_amp=True,  # bfloat16
)
```

5. Index with ChromaDB and evaluate Top-k accuracy

```python
import chromadb

client = chromadb.PersistentClient(path="./legal_db")
collection = client.get_or_create_collection(
    name="eu_env_law",
    metadata={"hnsw:space": "cosine"}
)

# Embed and index all documents
doc_embeddings = model.encode(documents, normalize_embeddings=True)
collection.add(
    embeddings=doc_embeddings.tolist(),
    documents=documents,
    ids=law_ids,
    metadatas=[{"language": lang, "year": year} for lang, year in zip(langs, years)]
)

# Query with metadata-style input
results = collection.query(
    query_embeddings=model.encode(
        ["Commission Regulation on mercury emissions limits 2018"],
        normalize_embeddings=True
    ).tolist(),
    n_results=5
)
```

Output: Top-1 accuracy improves from ~81% to ~89% (English), ~73% to ~84% (low-resource languages).

---

**Example 2: Measuring PDF extraction quality with LCS**

User: "I extracted text from 500 legal PDFs but I'm not sure how much noise the OCR introduced. How can I measure extraction quality?"

Approach:
1. Obtain reference HTML versions of the same documents (e.g., from EUR-Lex HTML endpoint)
2. Compute LCS per document comparing PDF-extracted vs HTML text

```python
from sklearn.feature_extraction.text import CountVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

def compute_lcs(html_text: str, pdf_text: str) -> float:
    """Lexical Content Score: BoW cosine similarity between reference and extracted text."""
    vectorizer = CountVectorizer()
    try:
        vectors = vectorizer.fit_transform([html_text, pdf_text])
        return cosine_similarity(vectors[0], vectors[1])[0, 0]
    except ValueError:
        return 0.0  # Empty document

# Compute per-document and per-language LCS
lcs_scores = {}
for doc in documents:
    score = compute_lcs(doc["html_text"], doc["pdf_text"])
    lcs_scores.setdefault(doc["language"], []).append(score)

# Report per-language quality
for lang, scores in lcs_scores.items():
    mean_lcs = np.mean(scores)
    low_quality = sum(1 for s in scores if s < 0.80)
    print(f"{lang}: mean LCS={mean_lcs:.3f}, below threshold={low_quality}/{len(scores)}")
```

Output:
```
EN: mean LCS=0.952, below threshold=12/500
DE: mean LCS=0.941, below threshold=18/500
MT: mean LCS=0.823, below threshold=87/500
```

3. Re-extract or exclude documents with LCS < 0.80
4. Investigate per-language patterns — low-resource languages with non-Latin scripts consistently score lower

---

**Example 3: Cross-lingual legal retrieval with bilingual fine-tuning**

User: "I want a system where a German-language query retrieves relevant French legislation."

Approach:
1. Construct bilingual training pairs: for each act, pair the German metadata query with both the German and French document texts as joint positives
2. Fine-tune with grouped multi-positive MNR loss

```python
# Bilingual pair construction
bilingual_pairs = []
for law_id in law_ids:
    de_meta = get_metadata(law_id, "de")
    de_doc = get_document(law_id, "de")
    fr_doc = get_document(law_id, "fr")
    # Both aligned translations are positives for the same query
    bilingual_pairs.append(InputExample(texts=[de_meta, de_doc]))
    bilingual_pairs.append(InputExample(texts=[de_meta, fr_doc]))
    # Symmetric: French query retrieves both
    fr_meta = get_metadata(law_id, "fr")
    bilingual_pairs.append(InputExample(texts=[fr_meta, fr_doc]))
    bilingual_pairs.append(InputExample(texts=[fr_meta, de_doc]))
```

3. After fine-tuning, query in German retrieves French documents with >80% Top-5 accuracy
4. The model transfers to unseen language pairs (e.g., German query retrieving Italian documents) without explicit Italian training

## Best Practices

- **Do:** Use metadata blocks (structured descriptions, act types, dates) as queries rather than random sentences — this mirrors real legal search behavior and produces better contrastive training signal.
- **Do:** Measure extraction quality with LCS before training. Noisy input text directly degrades embedding quality. Set a per-language LCS threshold (0.80 minimum) and exclude or re-extract failing documents.
- **Do:** Start with E5-Multilingual for rapid prototyping (20-30 min training per language) before scaling to larger models like Qwen3-4B for production.
- **Do:** Use early stopping on validation loss with up to 30 epochs — legal corpora are small enough that overfitting is a real risk.
- **Avoid:** Explicitly mining hard negatives for this task. In-batch negatives with MNR loss are sufficient and much simpler. The paper found no benefit from hard negative mining on legal metadata-to-document retrieval.
- **Avoid:** Training bilingual models indiscriminately. The paper found E5 benefits from bilingual training but Qwen models showed degradation. Test monolingual first, then compare bilingual on your specific model.
- **Avoid:** Ignoring token truncation. E5 truncates at 512 tokens, Qwen at 2048. Legal documents average 7,781 tokens — 8-15% of documents will be truncated. For long documents, consider chunking and aggregating scores.

## Error Handling

- **LCS scores uniformly low (<0.70) for a language:** The OCR pipeline is failing for that script/language. Switch OCR engines (e.g., from olmOCR to Nougat) or preprocess with rotation correction (`rotation_correction` field in LEMUR schema).
- **Training loss plateaus immediately:** Batch size may be too small for effective in-batch negatives. MNR loss requires diverse negatives — increase batch size to at least 32, preferably 64+.
- **Top-k accuracy drops after fine-tuning:** Likely overfitting on a small corpus. Reduce epochs, increase dropout, or add more training languages to regularize.
- **Cross-lingual retrieval much worse than monolingual:** The model may not have strong multilingual alignment in its base weights. Switch to a model with explicit multilingual pretraining (E5-Multilingual over monolingual variants).
- **ChromaDB query returns wrong language documents:** Add language metadata filtering to queries, or ensure the embedding space is language-agnostic by using bilingual fine-tuning.
- **Documents with tables score low on LCS:** Table-heavy pages extract poorly from PDFs. Use the `is_table` flag to identify these pages and apply specialized table extraction (markdown table format via olmOCR).

## Limitations

- The LEMUR corpus covers only EU environmental legislation (category 15.10). Fine-tuned models may not generalize to criminal law, contract law, or non-EU jurisdictions without additional domain data.
- Metadata-to-document retrieval is a specific task formulation. If your use case is passage retrieval (finding a specific paragraph within a long document), you need to chunk documents and reformulate pairs accordingly.
- The approach assumes authoritative HTML reference text exists for computing LCS. For jurisdictions without dual-format publication, you cannot validate extraction quality this way — consider manual sampling instead.
- Models with 512-token limits (E5) truncate most legal documents. For production systems over long legislation, prefer models with 2048+ token context or implement a chunking-and-reranking pipeline.
- Bilingual fine-tuning showed mixed results across model architectures. Always benchmark monolingual vs bilingual on your specific model before committing to a training strategy.
- In-batch negatives become less effective with very small datasets (<500 acts per language). For small corpora, consider augmenting with synthetic queries or using explicit hard negatives.

## Reference

- **Paper:** [LEMUR: A Corpus for Robust Fine-Tuning of Multilingual Law Embedding Models for Retrieval](https://arxiv.org/abs/2602.09570v1) — Focus on Section 4 (fine-tuning methodology), Section 3.3 (LCS metric), and Tables 2-4 (retrieval results by language).
- **Code:** [github.com/nargesbh/eur_lex](https://github.com/nargesbh/eur_lex) — Full pipeline from PDF extraction through fine-tuning and evaluation.
- **Dataset:** [huggingface.co/datasets/G4KMU/LEMUR](https://huggingface.co/datasets/G4KMU/LEMUR) — 24,953 documents across 25 EU languages with page-level annotations.