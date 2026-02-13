---
name: "cgpt-cluster-guided-partial-tables"
description: "Improve table retrieval by constructing cluster-guided partial tables and generating synthetic training queries with LLMs. Use when: 'build a table search system', 'improve table retrieval accuracy', 'fine-tune embeddings for table search', 'generate training data for table QA', 'retrieve relevant tables for a question', 'cluster table rows for better retrieval'."
---

# CGPT: Cluster-Guided Partial Tables for Table Retrieval

This skill enables Claude to implement the CGPT framework for building high-accuracy table retrieval systems. The core idea: instead of embedding entire tables (which causes semantic compression and poor query matching), you cluster table rows by semantic similarity using K-means, sample across clusters to build diverse partial tables, generate synthetic queries via an LLM, and then fine-tune an embedding model with hard-negative contrastive learning. This pipeline yields a 16.5% average R@1 improvement over baselines across standard benchmarks.

## When to Use

- When the user needs to build a search system that retrieves relevant tables from a large corpus given natural language queries
- When table retrieval accuracy is poor because queries only match a subset of rows in large tables
- When the user wants to generate synthetic training data (query-table pairs) for fine-tuning a retrieval model
- When embedding full tables produces low recall and the user needs a principled way to decompose tables into retrievable units
- When the user is building a table QA pipeline and the retrieval stage is the bottleneck
- When the user wants to fine-tune BGE-M3 or similar bi-encoder models specifically for structured tabular data

## Key Technique

**The problem with table retrieval.** Standard text embedding models compress an entire table into a single vector. But tables are wide and semantically heterogeneous -- a table about countries might have rows about GDP, population, geography, and politics all mixed together. When a user asks "Which country has the highest GDP per capita?", the embedding of the full table dilutes the GDP-related signal with unrelated rows, causing retrieval failures.

**CGPT's solution: cluster, sample, generate, train.** CGPT addresses this in four stages. First, it embeds each row of a table using a dense encoder (BGE-M3) and runs K-means clustering on the row embeddings. This groups semantically similar rows together. Then it samples a bounded number of rows from each cluster to construct *partial tables* -- compact sub-tables that each cover a coherent semantic slice of the original table. Second, an LLM generates synthetic natural-language queries that each partial table could answer, creating (query, partial-table) training pairs. Third, hard negatives are mined: for each synthetic query, the system retrieves the most similar *wrong* partial tables via cosine similarity. Finally, the embedding model is fine-tuned with InfoNCE contrastive loss using these (query, positive-partial-table, hard-negatives) triplets. The result is an embedding model that maps queries to the specific *part* of a table that answers them.

**Why clustering matters over heuristics.** Prior work (QGpT) selected partial tables by heuristic column/row slicing, which misses semantic structure. K-means clustering ensures each partial table contains rows that are actually related, producing more coherent synthetic queries and harder, more informative negatives during training.

## Step-by-Step Workflow

1. **Prepare the table corpus in JSONL format.** Each line must be a JSON object with fields: `file_name` (string identifier), `sheet_name` (string), `header` (list of column names), and `instances` (list of row lists, each row being a list of cell values matching the header order).

2. **Embed table rows and cluster with K-means.** For each table, encode every row as a dense vector using BGE-M3 (or a comparable bi-encoder). Compute the cluster count as `n_clusters = min(max_chunks_per_table, max(1, num_rows / min_instances_per_chunk))`. Run `KMeans(n_clusters=n_clusters, random_state=42)` on the row embeddings. Group row indices by cluster label.

3. **Construct partial tables from clusters.** For each cluster, build a partial table (called a "chunk") by keeping the original headers and sampling up to `max_instances_per_representation` rows from that cluster. Serialize each chunk as a text representation: `"File: {name} | Sheet: {sheet} | Headers: {h1, h2, ...} | Rows: {row1}, {row2}, ..."`. Assign each chunk a unique ID.

4. **Generate synthetic queries with an LLM.** For each partial table chunk, prompt an LLM (GPT-4o, GPT-4o-mini, or a local model) to generate `questions_per_chunk` natural language questions that the partial table can answer. Use a prompt like: *"Given the following table excerpt, generate {n} diverse questions that a user might ask which this table can answer. Return JSON: {\"questions\": [...]}"*. Parse the response and create training pairs: `{query: q, pos: [chunk_representation]}`.

5. **Mine hard negatives via dense retrieval.** Encode all chunk representations and all synthetic queries with the same embedding model. For each query, compute cosine similarity against all chunks, exclude the known positive, and select the top `num_negatives` (default: 8) most similar chunks as hard negatives. Append these to the training record: `{query: q, pos: [chunk_repr], neg: [neg1, neg2, ...]}`.

6. **Fine-tune the embedding model with InfoNCE loss.** Train the bi-encoder with contrastive learning: the positive chunk should be closer to the query than any hard negative. Use InfoNCE loss with temperature `tau=0.01`, learning rate `1e-5`, for 2 epochs. Shuffle the dataset before training.

7. **Index the partial-table chunks for retrieval.** After fine-tuning, re-encode all chunks with the updated model. Build a vector index (FAISS or similar) over the chunk embeddings.

8. **Retrieve at query time.** Encode the incoming user query with the fine-tuned model, perform approximate nearest-neighbor search against the chunk index, and return the top-k chunks. Map each chunk back to its parent table for downstream use (QA, analysis, etc.).

## Concrete Examples

**Example 1: Building a table search engine for CSV datasets**

```
User: I have 5,000 CSV files with product data. I need a search system where
users type a question like "cheapest laptop with 16GB RAM" and get the
right table back. Retrieval with vanilla embeddings only gets 40% accuracy.

Approach:
1. Convert each CSV to JSONL format:
   {"file_name": "electronics_q4.csv", "sheet_name": "Sheet1",
    "header": ["product", "category", "price", "ram", "storage"],
    "instances": [["ThinkPad X1", "laptop", "1299", "16GB", "512GB"], ...]}

2. Cluster rows within each table. A table with 200 product rows and
   min_instances_per_chunk=10 yields ~20 clusters. Cap at
   max_chunks_per_table=5, so K-means uses 5 clusters.

3. Sample 5 rows per cluster to build 5 partial tables per CSV.
   Total chunks: ~25,000 across the corpus.

4. Prompt GPT-4o-mini for each chunk:
   "Given this product table excerpt with columns [product, category,
   price, ram, storage] and rows: [ThinkPad X1, laptop, 1299, 16GB, 512GB],
   [MacBook Air, laptop, 999, 16GB, 256GB], ...
   Generate 5 questions users might ask. Return JSON."

   Output: {"questions": ["Which laptops have 16GB RAM under $1500?",
   "What is the cheapest laptop available?", ...]}

5. Mine hard negatives: "cheapest laptop with 16GB RAM" will be similar
   to phone/tablet chunks (which also mention prices and specs) -- these
   become hard negatives.

6. Fine-tune BGE-M3 for 2 epochs with InfoNCE loss (tau=0.01, lr=1e-5).

7. Re-index all 25,000 chunks. At query time, "cheapest laptop with
   16GB RAM" now retrieves the laptop-specific partial table chunk
   instead of being diluted by 200 mixed product rows.

Output: R@1 improves from ~40% to ~55-60% based on CGPT benchmark gains.
```

**Example 2: Multi-domain table QA with cross-domain generalization**

```
User: I'm building a QA system over tables from Wikipedia covering sports,
finance, and geography. Tables vary wildly in schema. How do I handle this?

Approach:
1. Load all Wikipedia tables into a unified JSONL corpus. Each table
   retains its own headers and structure -- no schema normalization needed.

2. Run the CGPT pipeline on the combined corpus. K-means clustering
   operates per-table, so a sports stats table clusters by team/season
   while a finance table clusters by sector/quarter. The clustering
   adapts to each table's internal semantics.

3. Generate synthetic queries per domain. The LLM naturally produces
   domain-appropriate questions:
   - Sports chunk -> "Which team scored the most goals in 2024?"
   - Finance chunk -> "What was Apple's Q3 revenue?"

4. Hard negative mining becomes cross-domain: a finance query about
   "quarterly revenue" might pull sports chunks about "quarterly stats"
   as hard negatives, teaching the model domain discrimination.

5. Fine-tune a single BGE-M3 model on the combined training set.
   The unified corpus setting in CGPT shows strong cross-domain
   generalization -- the model learns to distinguish domains without
   explicit domain labels.

Output: A single retrieval model that handles mixed-domain table
queries without per-domain fine-tuning.
```

**Example 3: Generating training data with a small local LLM**

```
User: I can't use GPT-4 for query generation due to cost/privacy.
Can I use a smaller model?

Approach:
1. Run Steps 1-3 (clustering and partial table construction) identically.

2. Swap the LLM to a local model (e.g., Llama-3-8B, Qwen-2-7B, or
   Mistral-7B) for synthetic query generation. Use the same prompt
   template but adjust for the model's instruction format.

3. The CGPT paper shows that smaller LLMs still produce effective
   training queries -- the quality of the partial tables matters more
   than the sophistication of the query generator. The clustering
   ensures each chunk is semantically coherent, making it easier for
   even a small LLM to produce relevant questions.

4. Proceed with hard negative mining and fine-tuning as normal.

Output: Comparable retrieval accuracy at a fraction of the generation
cost. The paper confirms effectiveness with smaller LLMs.
```

## Implementation Reference

```python
# Core CGPT pipeline in Python (pseudocode)
import numpy as np
from sklearn.cluster import KMeans
from FlagEmbedding import BGEM3FlagModel

# Step 1: Embed and cluster
model = BGEM3FlagModel("BAAI/bge-m3", use_fp16=True)

def build_partial_tables(table, min_per_chunk=10, max_chunks=5, max_sample=5):
    rows_text = [" | ".join(row) for row in table["instances"]]
    embeddings = model.encode(rows_text)["dense_vecs"]

    n_clusters = min(max_chunks, max(1, len(rows_text) // min_per_chunk))
    km = KMeans(n_clusters=n_clusters, random_state=42).fit(embeddings)

    chunks = []
    for label in range(n_clusters):
        indices = np.where(km.labels_ == label)[0]
        sampled = indices[:max_sample]
        chunk_rows = [table["instances"][i] for i in sampled]
        representation = serialize_chunk(table["header"], chunk_rows)
        chunks.append({"id": f"{table['file_name']}_{label}",
                        "representation": representation,
                        "all_indices": indices.tolist()})
    return chunks

# Step 2: Generate synthetic queries (call your LLM of choice)
def generate_queries(chunk_repr, n_questions=5):
    prompt = f"""Given this table excerpt:
{chunk_repr}
Generate {n_questions} diverse questions this table can answer.
Return JSON: {{"questions": [...]}}"""
    return call_llm(prompt)  # Returns list of question strings

# Step 3: Hard negative mining
def mine_hard_negatives(queries, chunks, num_neg=8):
    chunk_reprs = [c["representation"] for c in chunks]
    chunk_embs = model.encode(chunk_reprs)["dense_vecs"]
    training_data = []
    for q, pos_chunk in queries:
        q_emb = model.encode([q])["dense_vecs"][0]
        sims = chunk_embs @ q_emb / (np.linalg.norm(chunk_embs, axis=1) * np.linalg.norm(q_emb))
        ranked = np.argsort(-sims)
        negatives = [chunk_reprs[i] for i in ranked if chunk_reprs[i] != pos_chunk][:num_neg]
        training_data.append({"query": q, "pos": [pos_chunk], "neg": negatives})
    return training_data

# Step 4: Fine-tune with InfoNCE loss (tau=0.01, lr=1e-5, 2 epochs)
```

## Best Practices

- **Do:** Set `min_instances_per_chunk` based on your table sizes. For tables with 50-200 rows, 10 is a good default. For very large tables (1000+ rows), increase to 20-50 to avoid too many tiny clusters.
- **Do:** Generate 5+ synthetic queries per chunk to provide sufficient supervision signal. Fewer than 3 queries per chunk degrades fine-tuning quality.
- **Do:** Use at least 8 hard negatives per query. The contrastive loss needs enough negatives to learn fine-grained distinctions between semantically similar chunks.
- **Do:** Keep the contrastive temperature low (0.01). Higher temperatures soften the loss and reduce the model's ability to discriminate hard negatives.
- **Avoid:** Embedding entire tables as single documents. This is the exact problem CGPT solves -- semantic compression destroys retrieval accuracy for large, heterogeneous tables.
- **Avoid:** Using random row sampling instead of K-means clustering. Random sampling produces incoherent partial tables that confuse the LLM during query generation and yield weaker training signal. The clustering is the key contribution.
- **Avoid:** Skipping the hard negative mining step. Training with random negatives instead of retrieval-based hard negatives significantly reduces the effectiveness of contrastive fine-tuning.

## Error Handling

- **LLM returns malformed JSON during query generation.** Retry up to 5 times with the same prompt. If it still fails, skip that chunk -- losing a few training samples is acceptable.
- **Table has fewer rows than `min_instances_per_chunk`.** Set `n_clusters=1`, treating the entire table as a single chunk. Small tables don't need partitioning.
- **K-means produces empty clusters.** This can happen with very skewed data. Filter out empty clusters before building chunks. Scikit-learn's KMeans rarely produces empty clusters, but verify with `np.unique(km.labels_)`.
- **Hard negative mining retrieves the positive chunk.** Always filter by exact string match on the chunk representation, not by chunk ID alone, to prevent data leakage.
- **Out of GPU memory during embedding.** Process tables in batches. The CGPT implementation supports multi-GPU distribution by assigning tables round-robin across GPUs.

## Limitations

- **Column-heavy tables with few rows.** If a table has 100 columns but only 3 rows, row-level K-means clustering is ineffective. Consider transposing or using column-level clustering instead.
- **Tables with no semantic structure in rows.** If rows are randomly ordered and semantically uniform (e.g., a list of random numbers), clustering adds no value over random sampling.
- **LLM query quality ceiling.** The synthetic queries are only as good as the LLM generating them. Domain-specific tables (medical, legal) may need a domain-aware LLM or few-shot examples in the prompt.
- **Scalability of hard negative mining.** Computing cosine similarity of every query against every chunk is O(n*m). For millions of chunks, use approximate nearest neighbor (FAISS IVF) instead of brute force.
- **Not designed for real-time table updates.** Adding new tables requires re-running clustering and re-indexing. This is a batch pipeline, not a streaming one.

## Reference

**Paper:** [CGPT: Cluster-Guided Partial Tables with LLM-Generated Supervision for Table Retrieval](https://arxiv.org/abs/2601.15849v1) (WWW 2026). Look for Section 3 (methodology) for the clustering algorithm, Section 4 for the contrastive training setup, and Table 2 for benchmark results across MimoTable, OTTQA, FetaQA, and E2E-WTQ.

**Code:** [github.com/yumeow0122/CGPT](https://github.com/yumeow0122/CGPT) -- full pipeline with BGE-M3, K-means chunking, LLM query generation, hard negative sampling, and InfoNCE training.