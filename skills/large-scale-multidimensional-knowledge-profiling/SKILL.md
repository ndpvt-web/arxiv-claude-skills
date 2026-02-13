---
name: "large-scale-multidimensional-knowledge-profiling"
description: "Build multidimensional profiling pipelines for large scientific paper corpora. Combines BERTopic clustering, LLM-structured extraction, and weighted semantic retrieval to analyze research trends, topic lifecycles, dataset/model adoption, and methodological shifts. Use when: 'profile these research papers', 'analyze trends in this paper corpus', 'build a scientific literature pipeline', 'extract structured knowledge from papers', 'track topic evolution across conferences', 'find emerging research directions'."
---

This skill enables Claude to build end-to-end pipelines that transform large collections of scientific papers into structured, queryable knowledge bases with multidimensional profiling. Based on the technique from Xue et al. (2026), it combines PDF-to-Markdown conversion, BERTopic-based topic discovery, LLM-assisted structured extraction across five knowledge dimensions, and intent-driven weighted semantic retrieval -- turning unstructured paper corpora into analyzable research intelligence.

## When to Use

- When the user wants to build a pipeline to process and analyze a large collection of research papers (hundreds to 100K+)
- When the user asks to track how research topics emerge, grow, mature, or decline across years and conferences
- When the user needs to extract structured metadata (methods, datasets, models, metrics) from paper PDFs or abstracts at scale
- When the user wants to identify trending vs. declining research areas using temporal lifecycle analysis
- When the user asks to build a searchable research knowledge base with multi-field semantic retrieval
- When the user wants to analyze dataset adoption patterns, model architecture trends, or institutional research directions
- When the user needs to cluster papers into coherent topics and build topic hierarchies with LLM-generated summaries

## Key Technique

The pipeline operates in three interconnected stages. First, **topic discovery** uses BERTopic with `all-MiniLM-L6-v2` embeddings, UMAP dimensionality reduction (n_components=40, n_neighbors=120, min_dist=0.08, cosine metric), and HDBSCAN clustering (min_cluster_size=50, min_samples=1, EOM selection) to group papers into 300+ coherent topics. C-TF-IDF with bigrams (max_df=0.9, min_df=50) extracts representative terms per cluster. An LLM then generates hierarchical topic names and summaries from these cluster representations.

Second, **LLM-assisted structured extraction** parses each paper's content (converted from PDF to Markdown via minerU or similar tools) into five dimensions: Info (title, authors, venue, year), Summary (problem statement, keywords, contributions), Technical (architectures, datasets, metrics, loss functions, training setup), Analysis (novelty type, field positioning, limitations, future work), and System (GPU resources, compute requirements). This produces a uniform schema regardless of how each paper is written.

Third, **intent-driven retrieval** uses a two-stage approach: metadata filtering (conference, year, author) followed by weighted multi-field semantic search. A relevance score S_i = sum(w_j * s_ij) combines similarity across fields (abstract, methods, datasets, limitations) with learned weights. For temporal trend analysis, topics are plotted on a CAGR (Compound Annual Growth Rate) vs. mean-publication-year grid, with bubble size encoding paper count and color encoding weighted impact (alpha=0.6 for citations, beta=0.4 for topic count). This quadrant map classifies topics as Emerging, Booming, Mature, or Declining.

## Step-by-Step Workflow

1. **Collect and convert papers.** Gather PDFs from target venues (conferences, arXiv categories, or user-provided directories). Convert each PDF to structured Markdown using minerU, GROBID, or PyMuPDF, preserving section structure and metadata. Store raw Markdown files with venue/year metadata.

2. **Configure BERTopic for topic discovery.** Initialize BERTopic with `all-MiniLM-L6-v2` as the embedding model. Set UMAP parameters: `n_components=40, n_neighbors=120, min_dist=0.08, metric='cosine', random_state=42`. Set HDBSCAN: `min_cluster_size=50, min_samples=1, cluster_selection_method='eom'`. Configure C-TF-IDF with `(1,2)` n-gram range, `max_df=0.9`, `min_df=50`.

3. **Run topic clustering on paper abstracts.** Feed all abstracts through BERTopic's `fit_transform()`. This produces topic assignments, representative documents per topic, and C-TF-IDF term rankings. Expect 300+ topics for a 100K corpus; scale `min_cluster_size` proportionally for smaller corpora.

4. **Generate topic hierarchy with LLM.** For each topic cluster, send the top-10 representative documents and top-20 C-TF-IDF terms to an LLM. Prompt it to generate: (a) a concise topic name, (b) a one-paragraph description, and (c) a parent category for hierarchical grouping. Store topic metadata alongside cluster IDs.

5. **Extract structured knowledge per paper.** For each paper's Markdown content, prompt an LLM to extract fields across five dimensions: abstract summary, keywords with explanations, methods/architecture, loss functions, training setup, GPU info, datasets used, evaluation metrics, problem statement, contributions, novelty type, experiment details, results, limitations, future work, trend insights, and institutional affiliations. Use a structured JSON output schema.

6. **Build the knowledge base.** Store extracted records in a structured database (SQLite, PostgreSQL, or Elasticsearch). Index each paper with its topic assignment, venue, year, and all five extracted dimensions. Create embedding indices for abstract, methods, datasets, and limitations fields to support semantic search.

7. **Compute topic lifecycle metrics.** For each topic, calculate: paper count per year, CAGR = `(N_recent / N_earlier)^(1/period) - 1` over a two-year window, normalized mean publication year, and weighted impact `W_i = (0.6 * citations_i + 0.4 * count_i) / max(0.6 * citations + 0.4 * count)`. Plot topics on a four-quadrant grid (CAGR x novelty) to classify as Emerging, Booming, Mature, or Declining.

8. **Implement intent-driven retrieval.** Build a query interface that uses an LLM to decompose natural language queries into structured JSON: `{conferences, year_range, authors, keywords, search_fields_with_weights}`. Apply metadata filters first, then compute weighted semantic similarity `S_i = sum(w_j * s_ij)` across content fields. Return ranked results with relevance scores.

9. **Generate trend analysis outputs.** Produce visualizations: topic lifecycle quadrant plots, dataset adoption heatmaps (dataset x year x venue), model architecture usage timelines, and conference-level topic distribution charts. Use matplotlib or plotly for interactive exploration.

10. **Validate extraction quality.** Sample 50-100 papers and manually verify extracted fields against source text. Measure precision and recall per dimension. Tune LLM prompts for fields with accuracy below 85%. Re-run extraction on corrected prompts.

## Concrete Examples

**Example 1: Building a research trend analyzer for NeurIPS/ICML papers**

User: "I have 5,000 paper PDFs from NeurIPS and ICML (2022-2025). Help me build a pipeline to find emerging research topics."

Approach:
1. Convert PDFs to Markdown using minerU or PyMuPDF
2. Extract abstracts and metadata from the Markdown files
3. Run BERTopic clustering (adjust min_cluster_size to 20 for 5K papers)
4. Use an LLM to name each topic cluster
5. Compute CAGR and mean publication year per topic
6. Plot the lifecycle quadrant to identify Emerging topics (high CAGR, recent mean year)

Output:
```python
from bertopic import BERTopic
from umap import UMAP
from hdbscan import HDBSCAN
from sentence_transformers import SentenceTransformer

embedding_model = SentenceTransformer("all-MiniLM-L6-v2")
umap_model = UMAP(n_components=40, n_neighbors=120, min_dist=0.08,
                   metric="cosine", random_state=42)
hdbscan_model = HDBSCAN(min_cluster_size=20, min_samples=1,
                         cluster_selection_method="eom")

topic_model = BERTopic(
    embedding_model=embedding_model,
    umap_model=umap_model,
    hdbscan_model=hdbscan_model,
    n_gram_range=(1, 2),
    top_n_words=20,
    verbose=True
)

topics, probs = topic_model.fit_transform(abstracts)

# Compute lifecycle metrics per topic
import numpy as np
for topic_id in set(topics):
    papers = [p for p, t in zip(paper_metadata, topics) if t == topic_id]
    years = [p["year"] for p in papers]
    recent = sum(1 for y in years if y >= 2024)
    earlier = sum(1 for y in years if y <= 2023)
    cagr = (recent / max(earlier, 1)) ** 0.5 - 1
    mean_year = np.mean(years)
    # Classify: Emerging if cagr > 0.2 and mean_year > 2023.5
```

**Example 2: Extracting structured knowledge from papers with LLM**

User: "I want to extract datasets, models, and methods from 500 ML papers into a searchable database."

Approach:
1. Convert papers to Markdown
2. Design the extraction prompt covering all five dimensions
3. Process each paper through the LLM with structured JSON output
4. Store results in SQLite with full-text search

Output:
```python
EXTRACTION_PROMPT = """Analyze this research paper and extract structured information.
Return a JSON object with these fields:

{
  "title": "paper title",
  "summary": {
    "problem": "research problem addressed",
    "keywords": [{"term": "...", "explanation": "..."}],
    "contributions": ["contribution 1", "contribution 2"]
  },
  "technical": {
    "architectures": ["model names and descriptions"],
    "datasets": [{"name": "...", "role": "training|evaluation|both", "size": "..."}],
    "metrics": ["metric names with reported values"],
    "loss_functions": ["loss function descriptions"],
    "training": {"optimizer": "...", "lr": "...", "epochs": "...", "batch_size": "..."}
  },
  "analysis": {
    "novelty_type": "architecture|method|application|theoretical",
    "limitations": ["limitation 1"],
    "future_work": ["direction 1"]
  },
  "system": {
    "gpu_type": "...",
    "gpu_count": 0,
    "training_time": "..."
  }
}

Paper content:
{paper_markdown}
"""

# Process and store
import sqlite3, json
conn = sqlite3.connect("research_kb.db")
conn.execute("""CREATE TABLE IF NOT EXISTS papers (
    id INTEGER PRIMARY KEY, title TEXT, venue TEXT, year INTEGER,
    topic_id INTEGER, raw_json TEXT,
    datasets TEXT, architectures TEXT, methods TEXT
)""")

for paper in papers:
    result = llm_extract(EXTRACTION_PROMPT.format(paper_markdown=paper["content"]))
    parsed = json.loads(result)
    conn.execute("INSERT INTO papers VALUES (?,?,?,?,?,?,?,?,?)", (
        None, parsed["title"], paper["venue"], paper["year"],
        paper.get("topic_id"), result,
        json.dumps(parsed["technical"]["datasets"]),
        json.dumps(parsed["technical"]["architectures"]),
        json.dumps(parsed["summary"]["keywords"])
    ))
```

**Example 3: Building an intent-driven research paper search system**

User: "Build a search system where I can ask questions like 'What vision transformers were used for medical imaging in 2024?' and get relevant papers."

Approach:
1. Pre-index all papers with embeddings across multiple fields
2. Build an intent recognizer that decomposes queries into structured filters + semantic search
3. Apply two-stage retrieval: filter then rank

Output:
```python
INTENT_PROMPT = """Convert this research query into a structured search plan.
Return JSON:
{
  "filters": {"conferences": [], "year_range": [start, end], "authors": []},
  "semantic_search": {
    "keywords": ["key terms to match"],
    "field_weights": {
      "abstract": 0.3,
      "methods": 0.4,
      "datasets": 0.2,
      "limitations": 0.1
    }
  }
}

Query: {user_query}
"""

def search(query, paper_db, embedder):
    # Stage 1: Intent recognition
    intent = json.loads(llm_extract(INTENT_PROMPT.format(user_query=query)))

    # Stage 2: Metadata filtering
    candidates = paper_db.filter(
        conferences=intent["filters"]["conferences"],
        year_range=intent["filters"]["year_range"]
    )

    # Stage 3: Weighted multi-field semantic search
    query_emb = embedder.encode(query)
    scores = []
    weights = intent["semantic_search"]["field_weights"]
    for paper in candidates:
        score = sum(
            weights[field] * cosine_sim(query_emb, paper.embeddings[field])
            for field in weights
        )
        scores.append((paper, score))

    return sorted(scores, key=lambda x: -x[1])[:20]
```

## Best Practices

- **Do:** Scale `min_cluster_size` proportionally to corpus size. Use 50 for 100K papers, 20 for 5K, 10 for 1K. Too small produces noisy topics; too large merges distinct areas.
- **Do:** Use structured JSON output schemas in LLM extraction prompts. Enforce the schema with response validation and retry on malformed output. This keeps downstream processing reliable.
- **Do:** Compute topic lifecycle metrics over at least a 2-year window. Single-year fluctuations are noisy; CAGR over 2+ years reveals genuine trends.
- **Do:** Weight citation count (alpha=0.6) more than paper count (beta=0.4) when computing topic impact. Raw paper count inflates broad but shallow topics.
- **Avoid:** Running BERTopic on full paper text instead of abstracts. Full text introduces noise from boilerplate sections (related work citations, acknowledgments). Abstracts are the densest signal.
- **Avoid:** Using a single embedding field for retrieval. Multi-field weighted search (abstract + methods + datasets + limitations) significantly outperforms single-field search because different queries target different aspects of papers.

## Error Handling

- **PDF conversion failures:** Some papers have scanned images or corrupted formatting. Fall back to raw text extraction with PyMuPDF, or skip and log failures. Expect 2-5% conversion failures on large corpora.
- **LLM extraction returning malformed JSON:** Wrap extraction in a retry loop (max 3 attempts) with explicit error messages fed back to the LLM. Use `json.loads()` validation before accepting output.
- **BERTopic producing too many outliers (topic -1):** If more than 30% of papers are unassigned, reduce `min_cluster_size` or increase `n_neighbors` in UMAP to capture more local structure.
- **UMAP/HDBSCAN memory issues on large corpora:** For 100K+ papers, use UMAP's `low_memory=True` flag and consider batch embedding with `SentenceTransformer.encode(batch_size=256)`. Alternatively, subsample for initial clustering and assign remaining papers via `topic_model.transform()`.
- **Topic naming hallucination:** LLMs may invent plausible-sounding but inaccurate topic names. Cross-validate generated names against C-TF-IDF top terms. If the name doesn't match the top-5 terms, regenerate.

## Limitations

- **Abstract-only clustering** misses papers whose abstracts are vague but whose methods sections are distinctive. For methods-heavy analysis, consider clustering on extracted method descriptions instead.
- **LLM extraction accuracy** depends heavily on paper formatting quality. Papers with non-standard structures, heavy use of figures/tables without text descriptions, or very short abstracts yield lower extraction quality.
- **Temporal analysis requires sufficient time span.** CAGR and lifecycle classification need at least 3 years of data to be meaningful. Single-year corpora cannot support trend analysis.
- **The pipeline is English-centric.** Papers in other languages or with significant non-English technical terminology may cluster poorly with `all-MiniLM-L6-v2`.
- **Cost at scale.** LLM extraction on 100K papers is expensive. Budget for approximately 1-2 API calls per paper. Consider using smaller models (e.g., Qwen-32B, DeepSeek) for extraction and reserving larger models for topic naming and intent recognition.

## Reference

Xue, Z., Zhang, J., Jiang, J., Liu, J., & He, H. (2026). *Large-Scale Multidimensional Knowledge Profiling of Scientific Literature.* arXiv:2601.15170. [https://arxiv.org/abs/2601.15170](https://arxiv.org/abs/2601.15170)

Look for: The five-dimension extraction schema (Section 3), BERTopic configuration details (Section 4), topic lifecycle quadrant methodology (Section 5), and the two-stage intent-driven retrieval architecture (Section 6).