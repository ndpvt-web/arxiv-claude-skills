---
name: "evaluation-entity-matching-recommender"
description: "Build and evaluate cross-dataset entity matching pipelines for recommender systems. Implements the Reddit-Amazon-EM methodology: rule-based, lexical, embedding-based, graph neural, and LLM-based entity matching with systematic evaluation. Use when: 'match products across catalogs', 'link entities between datasets', 'deduplicate items across platforms', 'entity resolution for recommendations', 'cross-dataset product mapping', 'evaluate entity matching methods'."
---

# Entity Matching Evaluation for Recommender Systems

This skill enables Claude to build rigorous cross-dataset entity matching pipelines following the methodology from the Reddit-Amazon-EM paper. The core technique systematically evaluates five families of entity matching methods — rule-based, lexical (BM25), embedding-based (FAISS + fuzzy), graph neural (GNEM), and LLM-based (zero-shot classification) — against a human-annotated gold standard. Claude can apply this to any domain where items described differently across two catalogs must be linked: products, movies, books, restaurants, or any named entities in recommender systems.

## When to Use

- When the user needs to match or link entities (products, movies, books) between two different datasets or platforms (e.g., Reddit mentions to an Amazon catalog)
- When building a conversational recommender system that must resolve free-text entity mentions to a structured product database
- When deduplicating item catalogs where the same real-world entity has different representations across sources
- When the user asks to evaluate which entity matching strategy works best for their domain
- When constructing knowledge-grounded datasets that require cross-referencing items from multiple corpora
- When designing a retrieval pipeline that must first resolve ambiguous entity references before making recommendations

## Key Technique

The Reddit-Amazon-EM methodology treats entity matching as a multi-stage pipeline rather than a single-model problem. The insight is that different matching families excel at different failure modes: lexical methods handle exact and near-exact title matches cheaply, embedding methods catch semantic paraphrases, graph methods exploit relational structure, and LLMs resolve genuinely ambiguous cases requiring world knowledge. The paper's evaluation framework measures precision, recall, F1, and hit rate across all five families on the same gold-standard pairs, enabling principled method selection.

The practical takeaway is a **cascade architecture**: start with cheap high-precision methods (exact string match, BM25 retrieval) to handle easy cases, then escalate uncertain candidates through embedding similarity (using sentence-transformers + FAISS for fast approximate nearest neighbor search combined with fuzzy string matching), and finally route the hardest cases to an LLM for zero-shot classification. This cascade balances accuracy against computational cost — the paper found LLM-based methods (GPT-3.5-Turbo, GPT-4) achieved the highest accuracy overall, but hybrid strategies combining lightweight retrieval with selective LLM verification are more practical at scale.

The gold-standard construction process is equally important: human annotators manually verified entity pairs across Reddit-Movies and Amazon '23 catalogs, creating positive matches and hard negatives. This annotation methodology — sampling candidate pairs from multiple matching methods, then having humans judge — produces evaluation sets that stress-test each approach's weaknesses rather than just confirming easy matches.

## Step-by-Step Workflow

1. **Define the entity schema for both sources.** Extract the key identifying fields from each dataset (e.g., title, year, author/director, category). Normalize field names into a common schema with columns like `source_id`, `entity_name`, `entity_year`, `entity_category`, and `source_dataset`.

2. **Preprocess and normalize entity text.** Apply lowercasing, strip special characters, expand common abbreviations, and handle brand synonyms. For movie/product titles, remove parenthetical qualifiers like "(2023 Edition)" or "[Blu-ray]" that differ across platforms but refer to the same entity.

3. **Build the candidate index using BM25 + FAISS.** Index the target dataset (e.g., Amazon catalog) with both a BM25 sparse index (using `rank_bm25` or Elasticsearch) and a dense FAISS index (using `sentence-transformers` embeddings like `all-MiniLM-L6-v2`). This dual index enables both lexical and semantic retrieval of candidates.

   ```python
   from sentence_transformers import SentenceTransformer
   import faiss
   import numpy as np

   model = SentenceTransformer('all-MiniLM-L6-v2')
   target_titles = [item['title'] for item in amazon_catalog]
   embeddings = model.encode(target_titles, normalize_embeddings=True)
   index = faiss.IndexFlatIP(embeddings.shape[1])
   index.add(embeddings.astype('float32'))
   ```

4. **Retrieve top-K candidates for each source entity.** For each entity in the source dataset (e.g., Reddit mentions), query both the BM25 and FAISS indexes. Merge results using reciprocal rank fusion or simple union of top-K (K=10-20) from each retriever. This generates a candidate set per source entity.

5. **Score candidates with embedding similarity + fuzzy matching.** Compute cosine similarity from the dense embeddings and token-level fuzzy match ratio (using `thefuzz` / `rapidfuzz`). Combine scores: `combined_score = 0.6 * cosine_sim + 0.4 * fuzzy_ratio`. Candidates above a threshold of 0.80 are classified as matches; those between 0.65-0.80 are uncertain.

   ```python
   from rapidfuzz import fuzz

   def combined_score(query_emb, candidate_emb, query_text, candidate_text):
       cosine = np.dot(query_emb, candidate_emb)
       fuzzy = fuzz.token_sort_ratio(query_text, candidate_text) / 100.0
       return 0.6 * cosine + 0.4 * fuzzy
   ```

6. **Route uncertain candidates to LLM verification.** For candidate pairs in the uncertainty band (0.65-0.80 combined score), construct a zero-shot prompt asking the LLM whether the two entities refer to the same real-world item. Include all available metadata (title, year, category) in the prompt. Use structured output to get a binary yes/no with a confidence score.

   ```
   Prompt template:
   "Do these two entries refer to the same movie?
   Entry A: {source_title} ({source_year})
   Entry B: {target_title} ({target_year}, {target_category})
   Answer 'yes' or 'no' with a confidence between 0 and 1."
   ```

7. **Construct positive and negative pairs for evaluation.** From confirmed matches (high-confidence automatic + LLM-verified), create positive pairs. Sample hard negatives from near-miss candidates (high similarity but confirmed non-match). Split into train/validation/test sets (60/20/20) if training a supervised matcher like GNEM.

8. **Evaluate all methods against the gold standard.** Compute precision, recall, F1, and hit@K for each matching method independently on the same test set. Use the gold-standard pairs as ground truth. Report results in a comparison table.

9. **Build the final mapping using the cascade.** Apply the full cascade (exact match -> BM25+FAISS -> embedding+fuzzy -> LLM verification) to produce the complete cross-dataset mapping. Store as a CSV with columns: `source_id`, `target_id`, `match_score`, `match_method`, `confidence`.

10. **Validate a sample with human evaluation.** Randomly sample 50-100 matched pairs and 50-100 non-matched pairs from the final mapping. Build a simple evaluation interface (Streamlit works well) for human annotators to verify. Compute inter-annotator agreement and adjust thresholds if precision or recall is inadequate.

## Concrete Examples

**Example 1: Matching Reddit Movie Mentions to a Product Catalog**

User: "I have a dataset of Reddit posts mentioning movies and an Amazon product catalog. I need to link each Reddit movie mention to the correct Amazon product entry."

Approach:
1. Extract movie titles from Reddit post text using regex or NER (e.g., titles in quotes or after "I just watched")
2. Normalize titles: lowercase, strip "the" prefix variations, remove year suffixes
3. Index Amazon movie titles with BM25 (sparse) and MiniLM embeddings (dense) in FAISS
4. For each Reddit title, retrieve top-10 candidates from each index, merge via union
5. Score with combined embedding+fuzzy metric; auto-match above 0.80, flag 0.65-0.80 for LLM
6. Send uncertain pairs to GPT-3.5-Turbo with zero-shot prompt including title + year
7. Output final mapping CSV

Output:
```csv
reddit_post_id,reddit_title,amazon_asin,amazon_title,score,method
r_001,"inception","B004LWZWGQ","Inception (2010) [Blu-ray]",0.94,embedding_fuzzy
r_002,"the dark night","B001GZ6QEC","The Dark Knight (2008)",0.87,embedding_fuzzy
r_003,"shawshank","B000P0J0AQ","The Shawshank Redemption",0.72,llm_verified
```

**Example 2: Evaluating Multiple Entity Matching Strategies**

User: "I want to compare different entity matching methods on my product dataset and find which works best."

Approach:
1. Prepare a gold-standard set: manually annotate 200-500 entity pairs as match/non-match
2. Implement each method independently:
   - Rule-based: exact title match after normalization
   - Lexical: BM25 top-1 retrieval
   - Embedding: cosine similarity with `all-MiniLM-L6-v2`, threshold 0.80
   - Hybrid: embedding + fuzzy string matching (EmbFuzzy)
   - LLM: zero-shot GPT-3.5-Turbo classification on all pairs
3. Run each method on the same test split
4. Compute precision, recall, F1, hit@1, hit@5 per method

Output:
```
| Method        | Precision | Recall | F1    | Hit@1 | Hit@5 |
|---------------|-----------|--------|-------|-------|-------|
| Exact Match   | 0.98      | 0.42   | 0.59  | 0.42  | 0.42  |
| BM25          | 0.81      | 0.73   | 0.77  | 0.68  | 0.85  |
| Embedding     | 0.76      | 0.82   | 0.79  | 0.71  | 0.89  |
| EmbFuzzy      | 0.83      | 0.80   | 0.81  | 0.76  | 0.91  |
| LLM (GPT-3.5) | 0.91     | 0.86   | 0.88  | 0.84  | 0.93  |
| Cascade       | 0.90      | 0.87   | 0.88  | 0.85  | 0.94  |
```

**Example 3: Building a Cross-Platform Item Deduplication Pipeline**

User: "I'm merging two e-commerce catalogs and need to find which items are the same product listed differently."

Approach:
1. Define matching fields: product name, brand, category, price range
2. Normalize both catalogs: standardize brand names, strip model number formatting differences
3. Block on category (only compare items in the same category) to reduce O(n*m) comparisons
4. Within each block, compute BM25 + embedding scores on concatenated name+brand strings
5. Apply the 0.80/0.65 threshold cascade with LLM fallback for uncertain pairs
6. Output deduplicated catalog with canonical IDs and source provenance

Output:
```json
{
  "canonical_id": "prod_00123",
  "matched_entries": [
    {"source": "catalog_a", "id": "A-789", "title": "Sony WH-1000XM5 Headphones"},
    {"source": "catalog_b", "id": "B-456", "title": "Sony WH1000XM5 Wireless Noise Canceling"}
  ],
  "match_score": 0.92,
  "match_method": "embedding_fuzzy"
}
```

## Best Practices

- **Do:** Use blocking/filtering before pairwise comparison. Comparing every entity in source A against every entity in source B is O(n*m). Block on shared attributes (category, year, first letter) to reduce candidates by 10-100x.
- **Do:** Combine multiple matching signals. The paper shows that hybrid methods (EmbFuzzy combining embeddings + fuzzy string matching) consistently outperform any single signal. Weight the combination based on your domain.
- **Do:** Create a human-annotated gold standard before committing to a method. Even 200 annotated pairs dramatically improve method selection and threshold tuning.
- **Do:** Use the cascade pattern (cheap methods first, expensive LLM only for hard cases) in production. This preserves LLM-level accuracy at a fraction of the cost.
- **Avoid:** Relying solely on exact string matching. Entity names vary wildly across platforms — abbreviations, reorderings, extra qualifiers, and typos make exact match recall unacceptably low.
- **Avoid:** Using LLM-based matching for all pairs without pre-filtering. It is accurate but prohibitively expensive at scale. Reserve LLM calls for the 10-20% of pairs that fall in the uncertainty band.
- **Avoid:** Ignoring hard negatives in evaluation. Near-miss non-matches (e.g., "The Dark Knight" vs. "The Dark Knight Rises") are where methods truly differentiate. Include these in your test set.

## Error Handling

- **Ambiguous entities with no clear match:** When no candidate scores above 0.65, return `null` rather than forcing a low-confidence match. Flag these for manual review. In the Reddit-Amazon-EM study, approximately 15-20% of mentions had no catalog match.
- **Multiple high-confidence candidates:** When two or more candidates score above 0.80, use additional metadata (year, director, category) as tiebreakers. If still ambiguous, route to LLM with all candidates presented simultaneously.
- **Encoding and language mismatches:** Entity names may contain unicode characters, non-English text, or HTML entities. Apply `unicodedata.normalize('NFKD', text)` and strip HTML before any matching step.
- **Index staleness:** If the target catalog is updated frequently, rebuild the FAISS index periodically. Stale embeddings cause silent recall degradation.
- **LLM hallucination in verification:** LLMs may confidently affirm false matches. Always require the LLM to cite specific matching evidence (title overlap, year match) in its response; discard affirmations without supporting details.

## Limitations

- **Domain specificity:** The Reddit-Amazon-EM benchmark targets movies. The optimal thresholds (0.80/0.65) and score weights (0.6/0.4) are tuned for movie titles and may not transfer directly to other domains (electronics, books, restaurants). Recalibrate on a domain-specific gold set.
- **Scale constraints on LLM cascade:** Even with pre-filtering, the LLM verification step becomes a bottleneck at millions of entities. For very large catalogs, consider fine-tuning a smaller classifier on LLM-generated labels rather than calling the LLM at inference time.
- **Cold-start for graph methods:** Graph-based approaches (GNEM) require pre-existing relational structure (co-purchase graphs, co-mention graphs). They cannot be applied when entities have no graph context.
- **Temporal drift:** Product catalogs and user-generated mentions evolve. A matching model trained on 2023 data may degrade on 2025 mentions due to new naming conventions, sequels, or rebrands.
- **Single-language assumption:** The evaluated methods assume both sources use the same language. Cross-lingual entity matching requires multilingual embeddings (e.g., `paraphrase-multilingual-MiniLM-L12-v2`) and is not covered by this methodology.

## Reference

- **Paper:** [Evaluation on Entity Matching in Recommender Systems](https://arxiv.org/abs/2601.17218v2) — Huang et al., 2026. Focus on Section 4 (experimental evaluation) for method-by-method results and Section 3 for the gold-standard annotation methodology.
- **Code & Data:** [github.com/huang-zihan/Reddit-Amazon-Entity-Matching](https://github.com/huang-zihan/Reddit-Amazon-Entity-Matching) — Contains implementations of BM25+FAISS, EmbFuzzy, ComEM, GNEM, and LLM-based matchers, plus the annotated Reddit-Amazon-EM gold set.