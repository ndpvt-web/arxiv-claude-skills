---
name: "cost-efficient-rag-entity-matching"
description: "Build cost-efficient RAG pipelines for entity matching and deduplication using blocking-based batch retrieval and generation. Reduces LLM API calls and latency by grouping similar entity pairs into blocks before retrieval and inference. Use when the user asks to 'match entities across datasets', 'deduplicate records with LLMs', 'build a RAG pipeline for entity resolution', 'reduce cost of LLM-based record matching', 'link records between two tables', or 'entity matching with knowledge augmentation'."
---

# Cost-Efficient RAG for Entity Matching (CE-RAG4EM)

This skill enables Claude to design and implement cost-efficient retrieval-augmented generation pipelines for entity matching — the task of determining whether two records from different data sources refer to the same real-world entity. Rather than invoking an LLM once per candidate pair (which is prohibitively expensive at scale), this approach groups similar records into blocks, performs batch retrieval of contextual knowledge across entire blocks, and feeds multiple pairs into a single LLM prompt. The technique, based on the CE-RAG4EM architecture, typically achieves comparable or better F1 scores while drastically reducing API calls and end-to-end runtime.

## When to Use

- When the user needs to match or link entities across two datasets (e.g., product catalogs, citation databases, customer records)
- When the user wants to deduplicate a large table where naive pairwise LLM comparison is too expensive
- When building a RAG pipeline specifically for structured record comparison rather than free-text QA
- When the user asks to reduce the cost of an existing LLM-based entity matching system
- When integrating external knowledge (e.g., Wikidata, domain KGs) to improve matching accuracy on ambiguous records
- When the user needs to match records with heterogeneous schemas (different column names, missing fields, mixed data types)

## Key Technique

**Blocking-based batch retrieval and generation.** Standard RAG-for-entity-matching retrieves context and generates a match decision independently for each candidate pair. With N candidate pairs, that means N retrieval calls and N LLM invocations. CE-RAG4EM reduces this by first grouping records into similarity-based blocks using character q-gram or token-based blocking. Within each block, candidate pairs share overlapping attributes, so a single batch retrieval query can fetch contextual knowledge relevant to the entire block. This amortizes retrieval cost across all pairs in the block.

**Three retrieval granularities.** The framework supports entity-level retrieval (fetching Wikidata entity descriptions by vector similarity), predicate-level retrieval (fetching relation types), and triple-level retrieval (BFS or neighborhood expansion on a knowledge graph). Entity/predicate-level retrieval is cheaper and works well for textual attributes; triple-level retrieval captures richer structural context for numeric or ambiguous attributes but costs more.

**Batch generation.** Instead of one LLM call per pair, the system aggregates all pairs in a block (capped at `max_bs`, typically 4-6) into a single prompt with shared contextual knowledge. The LLM processes pairs sequentially within the prompt and outputs a match decision per pair. This reduces total LLM invocations by a factor roughly equal to the block size, with minimal impact on F1 when block size stays in the 4-6 range.

## Step-by-Step Workflow

1. **Load and normalize both tables.** Parse source table `Ts` and target table `Tt` into a common schema. Identify matching columns (title, name, manufacturer, price, etc.) and normalize data types. Handle missing values by leaving fields empty rather than imputing.

2. **Select blocking keys and build blocks.** Choose a blocking strategy based on attribute types:
   - **Q-gram blocking** (default): Generate character q-grams (q=3) from key attributes, group records sharing q-grams into blocks.
   - **Standard blocking**: Tokenize key attributes, group records sharing at least one token.
   - Assign each record to one or more blocks. Form candidate pairs within each block: `P_B = {(ri, rj) | ri in B ∩ Ts, rj in B ∩ Tt}`.

3. **Deduplicate and partition blocks.** Remove duplicate pairs that appear in multiple blocks (keep first occurrence only). If any block exceeds `max_bs` pairs (recommended: 4-6), split it into non-overlapping sub-blocks of at most `max_bs` pairs each.

4. **Construct batch retrieval queries.** For each block, aggregate the key attributes of all entity pairs into a single query string. Embed this query using a dense encoder (e.g., Jina Embeddings V3 or sentence-transformers).

5. **Retrieve contextual knowledge.** Query a vector index over your knowledge base (Wikidata, domain KG, or a curated reference table) with Top-k=2. Choose granularity:
   - **Entity-level**: Retrieve entity descriptions matching the aggregated query.
   - **Predicate-level**: Retrieve relevant relation types.
   - **Triple-level**: Use retrieved entities as seeds, run BFS (depth <= Dmax) or one-hop neighborhood expansion to collect structured triples.

6. **Enrich and filter retrieved context.** Rank retrieved items by cosine similarity to the original block query. Optionally use a lightweight LLM call to filter irrelevant results. Attach textual descriptions to entity IDs.

7. **Compose batch prompts.** Build an LLM prompt containing: (a) all entity pairs in the block formatted as numbered items, (b) the shared retrieved knowledge as "Additional Information", and (c) instructions to process each pair independently and output `Yes/No` per pair.

8. **Run batch LLM inference.** Send the prompt to the LLM with `temperature=0.5`, `top_p=0.8`, `max_tokens=1024`. Parse the response to extract per-pair match decisions.

9. **Aggregate results and evaluate.** Collect all match decisions across blocks. Compute precision, recall, and F1. If ground-truth labels are available, use them to tune `max_bs` and `Top-k` on a validation split.

10. **Iterate on configuration.** Adjust block size (larger blocks = fewer API calls but potential F1 drop), retrieval granularity (entity-level for text attributes, triple-level for numeric/ambiguous), and Top-k to find the cost-quality sweet spot for your dataset.

## Concrete Examples

**Example 1: Product catalog deduplication**

User: "I have two product CSVs from different vendors — `amazon.csv` and `google.csv` — with columns `title`, `manufacturer`, `price`. Help me find matching products without spending a fortune on API calls."

Approach:
1. Load both CSVs into DataFrames, normalize text (lowercase, strip whitespace).
2. Apply q-gram blocking (q=3) on the `title` column to generate candidate blocks.
3. Cap block size at `max_bs=6`, split oversized blocks.
4. For each block, concatenate all titles into a batch query, retrieve Top-2 Wikidata product entities via vector search.
5. Build a batch prompt with all pairs in the block plus retrieved product descriptions.
6. Call GPT-4o-mini once per block, parse Yes/No decisions.
7. Output matched pairs as a CSV.

```python
import pandas as pd
from collections import defaultdict

def qgram_blocking(ts, tt, attr="title", q=3, max_bs=6):
    """Group records into blocks using character q-grams."""
    blocks = defaultdict(lambda: {"source": [], "target": []})
    for idx, row in ts.iterrows():
        val = str(row[attr]).lower()
        for i in range(len(val) - q + 1):
            blocks[val[i:i+q]]["source"].append(idx)
    for idx, row in tt.iterrows():
        val = str(row[attr]).lower()
        for i in range(len(val) - q + 1):
            blocks[val[i:i+q]]["target"].append(idx)

    # Form candidate pairs, deduplicate, partition
    seen = set()
    batches = []
    current_batch = []
    for block in blocks.values():
        for s_idx in set(block["source"]):
            for t_idx in set(block["target"]):
                pair = (s_idx, t_idx)
                if pair not in seen:
                    seen.add(pair)
                    current_batch.append(pair)
                    if len(current_batch) >= max_bs:
                        batches.append(current_batch)
                        current_batch = []
    if current_batch:
        batches.append(current_batch)
    return batches

def build_batch_prompt(pairs, ts, tt, context, attrs):
    """Compose a single LLM prompt for an entire block of pairs."""
    lines = ["You are an expert in entity matching.\n## Input:"]
    for i, (s, t) in enumerate(pairs, 1):
        e1 = {a: str(ts.loc[s, a]) for a in attrs}
        e2 = {a: str(tt.loc[t, a]) for a in attrs}
        lines.append(f"Pair {i} - Entity 1: {e1}  Entity 2: {e2}")
    lines.append(f"\nAdditional Information: {context}")
    lines.append("\n## Instruction:")
    lines.append("Process each pair independently. Compare semantics and")
    lines.append("use the additional information to resolve ambiguity.")
    lines.append("\n## Output: For each pair, respond 'Yes' or 'No'.")
    return "\n".join(lines)
```

Output: A CSV of matched pairs with one LLM call per 6 pairs instead of one per pair — roughly 6x fewer API calls.

**Example 2: Citation record linking with knowledge graph augmentation**

User: "I need to link author records between DBLP and ACM datasets. Some authors have similar names but different publication histories. Can you use a knowledge graph to help disambiguate?"

Approach:
1. Load DBLP and ACM tables with columns `title`, `authors`, `venue`, `year`.
2. Apply standard token-based blocking on `authors` to group candidates.
3. For each block, retrieve Top-2 Wikidata scholar entities via vector search on author names.
4. Expand retrieved entities with one-hop neighborhood (EXP strategy) to pull in affiliated institutions, co-authors, and known publications.
5. Build batch prompts including the expanded triples as structured context.
6. Run batch inference, parse match decisions.

```
Block 3 prompt (max_bs=4):
---
Pair 1 - Entity 1: {title: "Graph databases", authors: "A. Khan", venue: "VLDB"}
         Entity 2: {title: "Graph database systems", authors: "Arijit Khan", venue: "PVLDB"}
Pair 2 - Entity 1: {title: "RDF indexing", authors: "P. Groth", venue: "ISWC"}
         Entity 2: {title: "RDF index structures", authors: "Paul Groth", venue: "ISWC 2023"}

Additional Information:
- Arijit Khan: Professor at Aalborg University, research areas: graph databases, knowledge graphs
- Paul Groth: Professor at University of Amsterdam, research areas: data provenance, knowledge graphs

Match Decisions: [Yes, Yes]
```

**Example 3: Reducing cost of an existing entity matching pipeline**

User: "Our current entity matching pipeline calls GPT-4o for every candidate pair. We have 50,000 pairs and it costs too much. How can I reduce the cost?"

Approach:
1. Audit the current pipeline to identify per-pair retrieval and generation calls.
2. Introduce blocking: apply q-gram blocking on the primary matching attribute to group the 50,000 pairs into blocks of ~6.
3. Replace per-pair retrieval with batch retrieval (one embedding + one vector search per block instead of per pair). This cuts retrieval calls from 50,000 to ~8,333.
4. Replace per-pair generation with batch generation (one LLM call per block). This cuts LLM calls from 50,000 to ~8,333.
5. Total API call reduction: ~6x for both retrieval and generation.
6. Monitor F1 on a labeled sample to verify quality holds. If F1 drops, reduce `max_bs` from 6 to 4.

## Best Practices

- **Do** start with q-gram blocking as the default — it handles typos and partial matches better than token-based blocking for most entity matching tasks.
- **Do** keep `max_bs` between 4 and 6. This range consistently yields near-peak F1 while providing substantial cost reduction. Going above 8 degrades quality.
- **Do** use entity-level (QID) or predicate-level (PID) retrieval for textual and date attributes. Reserve triple-level (BFS/EXP) retrieval for numeric or structurally ambiguous attributes where richer context is needed.
- **Do** deduplicate candidate pairs across blocks before retrieval — a pair appearing in 3 blocks should only be evaluated once.
- **Avoid** setting `Top-k` higher than 2 for retrieval. Experiments show diminishing returns beyond k=2 with increasing noise and cost.
- **Avoid** putting unrelated entity pairs in the same batch. The blocking step ensures pairs in a block share context — random batching loses this advantage and degrades quality.

## Error Handling

- **Oversized blocks**: If a blocking key is too common (e.g., a frequent manufacturer name), blocks can explode in size. Always enforce `max_bs` and split oversized blocks into sub-blocks. Log blocks that exceed 2x the cap as warnings.
- **LLM output parsing failures**: Batch prompts may produce malformed output (e.g., "Maybe" instead of "Yes/No", missing pairs). Implement regex-based parsing with fallback: if fewer decisions than pairs are returned, re-run the missing pairs individually.
- **Empty retrieval results**: If the knowledge base has no relevant entities for a block, fall back to zero-shot matching (no augmentation) for that block rather than injecting noise.
- **Blocking recall loss**: Aggressive blocking may miss true matches. Monitor pair completeness (fraction of true matches captured in candidate pairs). If recall drops below 95%, switch to a more permissive blocking strategy or use multiple blocking keys.
- **Token limit overflow**: Large blocks with rich retrieved context can exceed the model's context window. Truncate retrieved knowledge first (keep highest-similarity items), then reduce `max_bs` if still over limit.

## Limitations

- **Requires a blocking key**: If the two tables share no comparable attributes (completely disjoint schemas with no overlapping semantics), blocking cannot form meaningful groups. In that case, fall back to per-pair matching with embedding-based pre-filtering.
- **Knowledge base dependency**: The retrieval augmentation quality depends on the knowledge base coverage. For niche domains without Wikidata presence (e.g., internal enterprise records), the RAG component adds cost without proportional benefit. Consider building a domain-specific reference index instead.
- **Batch generation precision trade-off**: Batch prompts can cause the LLM to "contaminate" reasoning across pairs, especially when pairs in a block have conflicting signals. For high-stakes matching (e.g., financial record reconciliation), prefer per-pair generation with batch retrieval only.
- **Not suitable for small datasets**: If you have fewer than ~100 candidate pairs, the overhead of blocking, retrieval indexing, and prompt engineering exceeds the cost savings. Use direct per-pair LLM calls instead.
- **Blocking quality varies by domain**: Q-gram blocking works well for product names and author names but poorly for highly structured identifiers (e.g., serial numbers, UUIDs). Choose blocking strategy based on attribute characteristics.

## Reference

**Paper**: [Cost-Efficient RAG for Entity Matching with LLMs: A Blocking-based Exploration](https://arxiv.org/abs/2602.05708v1) — Ma et al., 2026. Look for Table 1 (six CE-RAG4EM design variants), Figure 2 (unified framework pipeline), and Section 5 (experiments across 9 benchmarks showing F1 improvements of +2-24% with reduced API calls).