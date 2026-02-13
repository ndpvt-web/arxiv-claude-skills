---
name: "research-multi-stage-machine-learning"
description: "Build multi-stage search pipelines that separate recall from precision for discovering datasets, documents, or resources from heterogeneous metadata catalogs. Uses intent classification, hybrid retrieval (BM25 + embeddings + abbreviation expansion), and LLM reranking. Trigger phrases: 'build a dataset discovery pipeline', 'multi-stage search system', 'find relevant datasets for my research goal', 'scientific data search', 'hybrid retrieval with reranking', 'intent-aware search pipeline'"
---

# ReSearch: Multi-Stage Intent-Aware Search Pipelines

This skill enables Claude to design and implement multi-stage search systems inspired by the ReSearch framework (arXiv:2601.14176). The core insight is that dataset and document discovery should be decomposed into three explicit stages -- intent interpretation, high-recall hybrid retrieval, and LLM-powered reranking -- rather than handled by a single retrieval pass. This separation lets each stage optimize for its own objective (coverage vs. precision) and handles the semantic gap between abstract user goals ("I want to study Arctic ice loss trends") and concrete metadata fields (variable names, platform codes, temporal ranges).

## When to Use

- When the user asks to build a search system over a catalog of datasets, documents, or resources with heterogeneous metadata
- When the user needs to find scientific datasets matching a high-level research objective rather than a specific keyword
- When building a retrieval pipeline that must handle both precise keyword queries ("MODIS sea surface temperature") and abstract task queries ("analyze drought impact on crop yields")
- When the user wants to combine lexical search (BM25) with semantic embeddings and LLM reranking in a single pipeline
- When existing search returns too many irrelevant results or misses relevant items because metadata uses abbreviations, jargon, or indirect descriptions
- When the user wants to build an evaluation benchmark by extracting ground-truth dataset references from published papers

## Key Technique

**The Recall-Precision Separation Principle.** Traditional search systems optimize a single relevance score, forcing a tradeoff between finding all relevant items (recall) and ranking them well (precision). ReSearch decomposes this into two explicit phases. The recall phase casts a wide net using multiple complementary retrieval strategies -- BM25 lexical matching, dense semantic embeddings, and abbreviation-expanded metadata. These are fused into a candidate pool of ~100 items. The precision phase then applies an LLM reranker that reads each candidate's metadata against the original query context and reorders the list. This two-phase design avoids premature filtering that would drop relevant but oddly-described items.

**Intent Classification Drives Query Processing.** Before retrieval begins, the system classifies each query as Type A (specific: mentions variables, platforms, or dataset names) or Type B (broad: expresses an abstract research goal). Type A queries get spell correction and direct retrieval. Type B queries are rewritten by an LLM into data-oriented search formulations that extract structured constraints (temporal range, spatial bounds, required variables) from the natural language intent. This rewriting step is critical -- experiments show that task-based queries degrade retrieval performance by 10-20% without it.

**Abbreviation Expansion as a High-Leverage Preprocessing Step.** Scientific metadata is dense with abbreviations (MODIS, ERA5, SST, NDVI). The framework preprocesses all indexed metadata by detecting abbreviations and inserting their full-form expansions inline (e.g., "SST" becomes "SST (Sea Surface Temperature)"). This simple step improved BM25 recall@10 by 140% on keyword queries in the paper's evaluation -- a disproportionate gain relative to its implementation cost.

## Step-by-Step Workflow

1. **Index the metadata catalog.** Load all dataset/document records into a structured store. Each record needs at minimum: title, description/abstract, and any domain-specific fields (variables, spatial coverage, temporal range). Store as JSON or in a database.

2. **Expand abbreviations in metadata.** Scan each record's text fields for known abbreviations. Build or use a domain-specific abbreviation dictionary. Insert full-form expansions inline after each abbreviation. For scientific domains, seed the dictionary from authoritative glossaries (e.g., NASA's acronym list, CMIP6 variable tables).

3. **Build the BM25 lexical index.** Index the abbreviation-expanded metadata using a BM25 implementation (e.g., `rank_bm25` in Python, Elasticsearch, or Lucene). Index concatenated title + description + variable fields as the document text. Use default BM25 parameters (k1=1.5, b=0.75) as a starting point.

4. **Build the semantic embedding index.** Encode each record's title + description using a sentence embedding model (e.g., `sentence-transformers/all-MiniLM-L6-v2` for general use, or a domain-fine-tuned model for scientific text). Store vectors in a FAISS index or similar approximate nearest-neighbor structure for fast retrieval.

5. **Implement intent classification.** When a query arrives, classify it as Type A (specific) or Type B (broad) using an LLM prompt. Type A queries contain explicit dataset names, variable codes, or platform identifiers. Type B queries express research goals or scientific questions without naming specific data products.

6. **Rewrite Type B queries.** For broad queries, use an LLM to extract structured constraints and rewrite the query into a data-oriented formulation. The rewrite prompt should produce: (a) a reasoning chain explaining what data would be needed, (b) a rewritten query suitable for retrieval, and (c) extracted constraints (temporal range, spatial bounds, required variables) as structured JSON.

7. **Execute hybrid retrieval.** Run the (possibly rewritten) query against both the BM25 index and the embedding index. Retrieve top-K candidates from each (K=100 is a reasonable default). Merge the two candidate lists using reciprocal rank fusion (RRF) or simple union with score normalization.

8. **Apply LLM reranking.** For each candidate in the merged pool, construct a prompt containing the original user query, the extracted intent, and the candidate's metadata. Ask the LLM to score relevance on a scale (e.g., 0-10) with a brief justification. Sort candidates by score. Batch candidates into groups of 10-20 per prompt to reduce API calls.

9. **Return ranked results with explanations.** Present the top-N results with the LLM-generated relevance justifications, making it clear why each dataset matches the user's intent. Include metadata fields the user can use to verify relevance (temporal coverage, spatial extent, variables).

10. **Evaluate with Recall@K, MRR, and MAP.** If building a benchmark, extract ground-truth dataset references from published papers using an LLM, match them to catalog entries via fuzzy name matching, and compute standard IR metrics across both keyword and task-based query sets.

## Concrete Examples

**Example 1: Building a dataset discovery API for a climate data catalog**

```
User: I have a CSV catalog of 50,000 climate datasets with columns
[id, title, description, variables, temporal_range, spatial_extent].
Build a multi-stage search pipeline so researchers can find relevant
datasets using natural language queries.

Approach:
1. Load the CSV catalog into pandas. Expand abbreviations in title
   and description fields using a climate science abbreviation dictionary.
2. Build a BM25 index over concatenated title+description+variables
   using rank_bm25.
3. Encode title+description with sentence-transformers and store
   embeddings in a FAISS index.
4. Implement a /search endpoint that:
   a. Classifies the query as specific vs. broad using an LLM call
   b. Rewrites broad queries into data-oriented formulations
   c. Retrieves top-100 from BM25 and top-100 from FAISS
   d. Merges via reciprocal rank fusion
   e. Reranks top-50 candidates with an LLM
   f. Returns top-10 with relevance explanations

Output (Python with FastAPI):
```

```python
import pandas as pd
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
from fastapi import FastAPI
from openai import OpenAI

# --- Stage 0: Preprocessing ---
catalog = pd.read_csv("catalog.csv")
ABBREVS = {"SST": "Sea Surface Temperature", "NDVI": "Normalized Difference Vegetation Index",
            "MODIS": "Moderate Resolution Imaging Spectroradiometer", "ERA5": "ECMWF Reanalysis v5"}

def expand_abbreviations(text: str) -> str:
    for abbr, full in ABBREVS.items():
        text = text.replace(abbr, f"{abbr} ({full})")
    return text

catalog["expanded_text"] = (catalog["title"] + " " + catalog["description"] + " " + catalog["variables"]).apply(expand_abbreviations)

# --- Stage 1a: BM25 Index ---
tokenized = [doc.lower().split() for doc in catalog["expanded_text"]]
bm25 = BM25Okapi(tokenized)

# --- Stage 1b: Embedding Index ---
model = SentenceTransformer("all-MiniLM-L6-v2")
embeddings = model.encode(catalog["expanded_text"].tolist(), show_progress_bar=True)
index = faiss.IndexFlatIP(embeddings.shape[1])
faiss.normalize_L2(embeddings)
index.add(embeddings)

# --- Search Pipeline ---
client = OpenAI()
app = FastAPI()

def classify_intent(query: str) -> str:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": "Classify this search query as 'specific' (mentions dataset names, variable codes, or platforms) or 'broad' (expresses a research goal). Reply with one word."},
                  {"role": "user", "content": query}])
    return resp.choices[0].message.content.strip().lower()

def rewrite_query(query: str) -> dict:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": "Rewrite this research goal into a data-oriented search query. Return JSON: {\"reasoning\": \"...\", \"rewritten_query\": \"...\", \"constraints\": {\"temporal\": \"...\", \"spatial\": \"...\", \"variables\": [...]}}"},
                  {"role": "user", "content": query}],
        response_format={"type": "json_object"})
    return json.loads(resp.choices[0].message.content)

def hybrid_retrieve(query: str, k: int = 100) -> list[int]:
    # BM25
    bm25_scores = bm25.get_scores(query.lower().split())
    bm25_top = np.argsort(bm25_scores)[-k:][::-1]
    # Embedding
    q_emb = model.encode([query])
    faiss.normalize_L2(q_emb)
    _, emb_top = index.search(q_emb, k)
    emb_top = emb_top[0]
    # Reciprocal Rank Fusion
    rrf_scores = {}
    for rank, idx in enumerate(bm25_top):
        rrf_scores[idx] = rrf_scores.get(idx, 0) + 1.0 / (60 + rank)
    for rank, idx in enumerate(emb_top):
        rrf_scores[idx] = rrf_scores.get(idx, 0) + 1.0 / (60 + rank)
    return sorted(rrf_scores, key=rrf_scores.get, reverse=True)

def llm_rerank(query: str, candidate_ids: list[int], top_n: int = 10) -> list[dict]:
    candidates = catalog.iloc[candidate_ids[:50]]
    batch_text = "\n".join(f"[{i}] {row['title']}: {row['description'][:200]}" for i, row in candidates.iterrows())
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "system", "content": "Score each dataset's relevance to the query (0-10). Return JSON array: [{\"id\": ..., \"score\": ..., \"reason\": \"...\"}]. Rank by score descending."},
                  {"role": "user", "content": f"Query: {query}\n\nDatasets:\n{batch_text}"}],
        response_format={"type": "json_object"})
    ranked = json.loads(resp.choices[0].message.content)
    return ranked[:top_n]

@app.get("/search")
def search(q: str):
    intent = classify_intent(q)
    search_query = q
    if intent == "broad":
        rewrite = rewrite_query(q)
        search_query = rewrite["rewritten_query"]
    candidates = hybrid_retrieve(search_query)
    results = llm_rerank(q, candidates)
    return {"intent": intent, "results": results}
```

**Example 2: Adding multi-stage search to an existing document retrieval system**

```
User: My Elasticsearch index has 200K research papers. Keyword search
works okay but misses papers when users ask vague questions like
"methods for predicting floods." How can I improve it?

Approach:
1. Keep Elasticsearch as the BM25 backend -- it already handles this well.
2. Add an embedding sidecar: encode all paper abstracts with a
   sentence-transformer, store in a FAISS index alongside ES doc IDs.
3. Add a query preprocessor that detects broad vs. specific intent.
4. For broad queries, rewrite them via LLM into concrete technical terms
   before searching (e.g., "methods for predicting floods" -> "hydrological
   forecasting rainfall-runoff model streamflow prediction").
5. Merge ES + FAISS results via reciprocal rank fusion.
6. Rerank top-50 merged candidates with an LLM scoring prompt.

Key changes to existing code:
- Add a `/search/enhanced` endpoint that wraps the existing ES query
- The existing `/search` endpoint stays unchanged for backward compatibility
- Abbreviation expansion applied at index time via an ES analyzer plugin
  or a preprocessing step in the indexing pipeline
```

**Example 3: Building an evaluation benchmark from published papers**

```
User: How do I evaluate whether my search pipeline is actually finding
the right datasets?

Approach:
1. Collect 20-30 published papers from the target domain.
2. For each paper, use an LLM to extract all referenced datasets
   (prompt: "List every dataset, data product, or data source
   explicitly cited in this paper with its full name and any URLs").
3. Match extracted dataset names to your catalog using:
   a. Exact URL pattern matching (if DOIs/URLs are available)
   b. Fuzzy string matching on dataset titles (fuzz.ratio > 85)
   c. Manual verification for ambiguous matches
4. For each paper, create two query types:
   - Keyword query: key domain terms from the paper's methods section
   - Task query: the paper's research objective stated as a goal
     ("I want to study X using Y approach")
5. Compute Recall@K, MRR, and MAP:

Output (evaluation code):
```

```python
from collections import defaultdict

def recall_at_k(retrieved_ids: list, relevant_ids: set, k: int) -> float:
    return len(set(retrieved_ids[:k]) & relevant_ids) / len(relevant_ids) if relevant_ids else 0.0

def mrr(retrieved_ids: list, relevant_ids: set) -> float:
    for rank, rid in enumerate(retrieved_ids, 1):
        if rid in relevant_ids:
            return 1.0 / rank
    return 0.0

def mean_avg_precision(retrieved_ids: list, relevant_ids: set) -> float:
    hits, sum_prec = 0, 0.0
    for rank, rid in enumerate(retrieved_ids, 1):
        if rid in relevant_ids:
            hits += 1
            sum_prec += hits / rank
    return sum_prec / len(relevant_ids) if relevant_ids else 0.0

# Run evaluation across all query-groundtruth pairs
results = {"recall@10": [], "recall@100": [], "mrr": [], "map": []}
for query, ground_truth_ids in benchmark:
    retrieved = search_pipeline(query)
    results["recall@10"].append(recall_at_k(retrieved, ground_truth_ids, 10))
    results["recall@100"].append(recall_at_k(retrieved, ground_truth_ids, 100))
    results["mrr"].append(mrr(retrieved, ground_truth_ids))
    results["map"].append(mean_avg_precision(retrieved, ground_truth_ids))

for metric, values in results.items():
    print(f"{metric}: {sum(values)/len(values):.3f}")
```

## Best Practices

- **Do:** Expand abbreviations at index time, not query time. This is a one-time cost that yields permanent recall improvements (140% gain on keyword queries in the paper's evaluation).
- **Do:** Use reciprocal rank fusion (RRF) with k=60 to merge BM25 and embedding results. RRF is parameter-light and consistently outperforms linear score combination.
- **Do:** Preserve the original user query for the reranking stage even after rewriting it for retrieval. The reranker needs the user's actual intent, not the retrieval-optimized reformulation.
- **Do:** Batch candidates into groups of 10-20 per LLM reranking call to balance cost and ranking quality. Scoring one-at-a-time is too expensive; scoring all 100 at once exceeds context limits and degrades judgment.
- **Avoid:** Skipping the intent classification step. Applying query rewriting to already-specific queries adds latency and can introduce noise (e.g., rewriting "ERA5 2m temperature" into a verbose paraphrase).
- **Avoid:** Using embedding-only retrieval without BM25. Embeddings have higher MRR but lower recall than BM25 in domain-specific catalogs. The hybrid combination consistently outperforms either alone.

## Error Handling

- **Empty BM25 results:** If BM25 returns no matches (all scores near zero), fall back to embedding-only retrieval. This typically happens with novel terminology not in the index vocabulary.
- **LLM reranking failures:** If the reranking LLM returns malformed JSON or times out, fall back to the hybrid retrieval ordering without reranking. Log the failure for debugging but never block the user.
- **Abbreviation expansion collisions:** Some abbreviations are ambiguous across domains (e.g., "SAR" could be Synthetic Aperture Radar or Successive Approximation Register). Scope abbreviation dictionaries to the target domain and prefer the most common expansion.
- **Query rewrite hallucination:** The LLM may invent constraints not implied by the user's query (e.g., adding a temporal range). Validate extracted constraints against the original query text and drop any that lack supporting evidence.
- **Catalog drift:** If the catalog is updated frequently, rebuild the BM25 index and embedding index on a schedule. Stale indexes miss new entries entirely.

## Limitations

- **LLM reranking cost:** Reranking 50 candidates per query requires LLM API calls that add latency (1-3 seconds) and cost. For high-throughput systems (>100 QPS), consider a cross-encoder model instead of a generative LLM for reranking.
- **Domain-specific abbreviation dictionaries:** The abbreviation expansion step requires a curated dictionary per domain. Building this for a new domain takes manual effort; automated extraction from domain corpora is possible but noisy.
- **Task-based queries remain hard:** Even with the full pipeline, the paper reports recall@100 of only ~0.28 on abstract research goal queries. Multi-stage search improves over baselines but does not solve the fundamental semantic gap for highly abstract intents.
- **Benchmark construction requires domain papers:** Evaluation depends on access to published papers that cite specific datasets. Domains without this citation culture (e.g., proprietary enterprise data) need alternative benchmark strategies.
- **Cold start on small catalogs:** The hybrid retrieval advantage diminishes when catalogs have fewer than ~1,000 items. For small catalogs, simple keyword search with LLM reranking may suffice.

## Reference

**Paper:** Sun, Wen, Yang. "ReSearch: A Multi-Stage Machine Learning Framework for Earth Science Data Discovery." arXiv:2601.14176v1 (2026). https://arxiv.org/abs/2601.14176v1

Look for: Table 2 (retrieval metrics comparison), the three prompt templates in Appendix B (intent classification, query rewrite, paper information extraction), and the benchmark construction methodology in Section 4 that shows how to build ground-truth query-dataset pairs from published literature.