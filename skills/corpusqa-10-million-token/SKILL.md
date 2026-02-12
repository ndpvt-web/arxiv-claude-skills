---
name: "corpusqa-10-million-token"
description: "Corpus-level QA over massive document collections using memory-augmented agentic processing. Synthesize answers that require global integration, comparison, and statistical aggregation across hundreds of documents. Use when: 'analyze all these documents and answer...', 'compare metrics across this corpus', 'aggregate statistics from these reports', 'what patterns exist across all files in...', 'summarize findings across the entire dataset', 'rank entities by computed metrics from these documents'."
---

# Corpus-Level QA with Memory-Augmented Agentic Processing

This skill enables Claude to answer complex analytical questions over large document corpora (hundreds to thousands of files) where evidence is dispersed across many documents and cannot be resolved by retrieving a few relevant chunks. Based on the CorpusQA framework, the core technique decouples reasoning from raw text by extracting structured schemas from each document, aggregating them into queryable tables, and executing programmatic computations — then presenting synthesized answers grounded in the full corpus. When the corpus exceeds context limits, a memory-augmented iterative processing loop replaces naive RAG.

## When to Use

- When the user asks a question requiring data from dozens or hundreds of documents (e.g., "Which companies across these 200 annual reports had revenue growth above 15%?")
- When the answer demands statistical aggregation — averages, rankings, ratios, counts, percentages — computed across an entire document set
- When the user needs multi-condition filtering combined with comparison (e.g., "Among universities with admission rates below 50%, which ones have six-year graduation rates exceeding the group average by more than 15 points?")
- When standard search/retrieval would fail because no single document contains the answer — the answer only emerges from synthesizing information globally
- When the user has a directory of reports, filings, transcripts, or records and wants cross-document analytical queries answered
- When building a QA pipeline or evaluation benchmark over large unstructured corpora with verifiable ground-truth answers

## Key Technique

**Decoupling reasoning from text.** The CorpusQA framework separates the analytical reasoning task from the unstructured text problem. Instead of asking an LLM to reason directly over millions of tokens of raw prose, you first extract structured key-value schemas from each document (entity attributes like revenue, enrollment, square footage), aggregate those schemas into a unified table (entities as rows, attributes as columns), and then execute the analytical query against that structured representation. This guarantees programmatically verifiable answers — the ground truth comes from SQL-like execution over extracted data, not from LLM generation.

**Why RAG collapses.** Standard retrieval-augmented generation assumes answers live in a few retrievable chunks. For corpus-level analytical queries, the relevant evidence is spread across every document in the collection. A question like "What is the median tuition across all 300 universities?" requires data from all 300 documents — no retrieval system will fetch all of them. CorpusQA experiments show RAG accuracy drops to 1-2% at scale while memory-augmented approaches retain 10-22%.

**Memory-augmented agentic architecture.** When the corpus exceeds context limits, process documents iteratively in chunks, maintaining a fixed-size memory buffer that accumulates extracted structured data. Each chunk is read, relevant schema fields are extracted and merged into the memory buffer, and the chunk is discarded. After all documents are processed, the final answer is computed solely from the consolidated memory. This avoids the context-window ceiling and the retrieval bottleneck simultaneously.

## Step-by-Step Workflow

1. **Inventory the corpus.** List all documents, count them, estimate total token size. Determine whether the corpus fits in context or requires iterative chunked processing (threshold: if total tokens exceed available context, use the memory-augmented loop).

2. **Analyze the query to identify required attributes.** Parse the user's question to determine what entity-level attributes are needed. For "Which companies had revenue growth above 15%?", the required attributes are: company name, revenue (current period), revenue (prior period). Write these down explicitly as the extraction schema.

3. **Extract structured schemas from each document.** For each document, extract the required key-value pairs into a JSON object. Use consistent field names across all documents. If a document lacks a required field, record it as `null`. Apply validation: if working with critical data, use multi-pass extraction or cross-check extracted values against the source text.

   ```json
   {"entity": "Acme Corp", "revenue_2024": 4200000, "revenue_2023": 3500000, "sector": "Technology"}
   ```

4. **Aggregate schemas into a unified table.** Merge all per-document JSON objects into a single array (or DataFrame/CSV). Each row is one entity (document), each column is an attribute. This is the structured representation of the entire corpus.

5. **Translate the user's question into a computable query.** Convert the natural language question into a concrete computation over the aggregated table — filtering conditions, aggregation functions (SUM, AVG, COUNT, MEDIAN), sorting/ranking, ratio calculations, or multi-step derived metrics. Express this as pseudocode or SQL.

   ```sql
   SELECT entity, (revenue_2024 - revenue_2023) / revenue_2023 * 100 AS growth_pct
   FROM corpus_table
   WHERE growth_pct > 15
   ORDER BY growth_pct DESC
   ```

6. **Execute the computation.** Run the query against the aggregated table. For simple queries, compute inline. For complex multi-step queries, use Python/pandas or actual SQL. This produces the ground-truth answer.

7. **Format and present the answer with provenance.** Return the computed result alongside the entities and source documents that contributed to it. Include the computation logic so the user can verify.

8. **Handle oversized corpora with iterative memory processing.** If the corpus cannot be fully loaded, process documents in batches:
   - Initialize an empty memory buffer (JSON array or table).
   - For each batch: read documents, extract schema fields, append rows to the memory buffer, discard raw text.
   - After all batches: execute the analytical query against the complete memory buffer.
   - Report which documents contributed data and flag any extraction failures.

9. **Validate and cross-check.** For numerical answers, sanity-check results (are counts reasonable given corpus size? are percentages in valid ranges?). For ranking queries, verify that the ordering criterion was applied correctly. Flag documents where extraction confidence was low.

10. **Report limitations transparently.** State how many documents were processed vs. total, extraction success rate, and any documents that were skipped or had missing fields. If the answer depends on incomplete data, say so.

## Concrete Examples

**Example 1: Cross-corpus statistical comparison**

User: "I have 150 university annual reports in `./reports/`. What is the average six-year graduation rate for universities with admission rates below 50%, and which schools exceed that average by more than 15 percentage points?"

Approach:
1. Glob `./reports/*.pdf` or `*.md` to inventory all 150 files.
2. Define extraction schema: `{university, admission_rate, graduation_rate_6yr}`.
3. Process each report, extracting the three fields into a JSON array.
4. Filter to universities where `admission_rate < 50`.
5. Compute `AVG(graduation_rate_6yr)` for the filtered set.
6. Filter again for `graduation_rate_6yr > avg + 15`.
7. Return the average and the list of qualifying universities.

Output:
```
Filtered corpus: 87 universities with admission rate < 50%
Average 6-year graduation rate (filtered): 72.4%
Threshold (avg + 15pp): 87.4%

Universities exceeding threshold:
| University          | Admission Rate | 6-Year Grad Rate |
|---------------------|---------------|-------------------|
| Stanford University | 4.3%          | 95.2%             |
| MIT                 | 3.9%          | 94.8%             |
| Yale University     | 6.1%          | 93.1%             |
| ... (12 more)       |               |                   |

Data extracted from 87/150 reports (63 had admission rates >= 50%).
3 reports had missing graduation rate data and were excluded.
```

**Example 2: Financial metric ranking across filings**

User: "From the 10-K filings in `./filings/`, rank the top 10 companies by debt-to-equity ratio and flag any with ratio above 3.0."

Approach:
1. Inventory `./filings/` — find 200 10-K filings.
2. Schema: `{company, total_debt, total_equity}`.
3. Extract from each filing. Use iterative memory processing if corpus is large.
4. Compute `debt_to_equity = total_debt / total_equity` per company.
5. Sort descending, take top 10. Flag those above 3.0.

Output:
```
Top 10 companies by debt-to-equity ratio:
| Rank | Company       | Total Debt ($M) | Equity ($M) | D/E Ratio | Flag |
|------|---------------|-----------------|-------------|-----------|------|
| 1    | TelcoCorp     | 89,200          | 12,400      | 7.19      | !!   |
| 2    | RetailMax     | 45,100          | 8,900       | 5.07      | !!   |
| ...  |               |                 |             |           |      |

4 companies flagged with D/E ratio > 3.0.
Extraction success: 195/200 filings (5 had non-standard formatting, skipped).
```

**Example 3: Building a verifiable QA benchmark from a document set**

User: "I want to create a QA benchmark from our internal knowledge base (500 docs). Generate questions with provable answers."

Approach:
1. Extract structured schemas from all 500 documents using multi-field extraction.
2. Aggregate into a master table.
3. Generate queries at three difficulty levels:
   - **Easy**: Single-condition filter + simple aggregation ("How many documents mention X?")
   - **Medium**: Multi-condition filter + ratio/grouped comparison ("What percentage of products in category A have rating > 4.5?")
   - **Hard**: Multi-step derived metric ("Among teams with >10 members, compute avg project completion rate, then find teams exceeding that by >20pp who also had budget under $50K")
4. Execute each query against the master table to produce ground-truth answers.
5. Output QA pairs as JSON with question, SQL-equivalent logic, answer, and source document IDs.

Output:
```json
[
  {
    "question": "What is the average project completion rate across all teams with more than 10 members?",
    "difficulty": "medium",
    "sql": "SELECT AVG(completion_rate) FROM teams WHERE member_count > 10",
    "answer": "78.3%",
    "source_docs": ["team_alpha.md", "team_beta.md", ... ],
    "num_sources": 34
  }
]
```

## Best Practices

- **Do:** Always define the extraction schema explicitly before processing documents. A precise schema (exact field names, expected types, units) prevents inconsistent extractions that corrupt downstream aggregation.
- **Do:** Process documents in deterministic order and log extraction results per document. This makes the pipeline reproducible and debuggable.
- **Do:** Use multi-pass or multi-model extraction for critical data. The CorpusQA framework achieved 94-99% accuracy using multi-model voting (extracting with multiple models and taking consensus).
- **Do:** Validate extracted numerical values against sanity bounds (e.g., percentages should be 0-100, revenue should be positive). Catch extraction errors before they propagate.
- **Avoid:** Attempting to answer corpus-level analytical queries by stuffing all documents into a single prompt. LLM accuracy degrades sharply beyond 128K tokens for aggregation tasks (50% at 1M tokens, near-zero at 10M).
- **Avoid:** Using RAG for queries that require data from most or all documents. If the answer requires global aggregation (averages, counts, rankings across the full set), retrieval will miss most of the evidence. Extract-then-compute is the correct pattern.
- **Avoid:** Skipping the structured extraction step and trying to "reason" over raw text for numerical/statistical queries. The decoupling of reasoning from text is what makes answers verifiable.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| Extraction returns `null` for expected fields | Document uses non-standard formatting or terminology | Re-attempt with a more specific extraction prompt that includes the document's actual terminology. Log and exclude if still failing. |
| Aggregated counts don't match corpus size | Some documents failed extraction or were duplicates | Report `N extracted / M total` in output. Check for duplicate entities and merge. |
| Computed percentages exceed 100% or are negative | Unit mismatch or misidentified fields (e.g., extracting revenue in thousands vs. millions) | Normalize units during extraction. Add unit to schema definition. |
| Memory buffer grows too large during iterative processing | Too many attributes extracted per document | Reduce schema to only query-relevant fields. Summarize/compress buffer periodically. |
| Answer differs from user's expectation | Ambiguous query interpretation (e.g., "average" could mean mean or median) | Clarify the computation before executing. Show the SQL/pseudocode equivalent for user confirmation. |

## Limitations

- **Extraction quality is the ceiling.** If structured data cannot be reliably extracted from the documents (e.g., highly narrative text with no quantifiable attributes), this approach loses its advantage. It works best on semi-structured corpora: reports, filings, records, profiles.
- **Schema must be knowable in advance.** The extraction step requires defining what fields to look for. Open-ended exploratory questions ("What interesting patterns exist?") need a discovery phase first.
- **Numerical precision depends on source documents.** If documents report rounded or inconsistent numbers, aggregated results inherit that imprecision.
- **Not suitable for subjective or interpretive queries.** Questions like "What is the overall sentiment across these documents?" are better served by different approaches. This skill targets computation-intensive, fact-grounded analytical queries.
- **Iterative memory processing trades recall for scalability.** At extreme corpus sizes (10M+ tokens), even the memory-augmented approach showed ~11% accuracy in CorpusQA benchmarks. For mission-critical queries over very large corpora, consider breaking the problem into domain-specific sub-corpora.

## Reference

**Paper:** [CorpusQA: A 10 Million Token Benchmark for Corpus-Level Analysis and Reasoning](https://arxiv.org/abs/2601.14952v1) (Lu et al., 2026). Key sections: Section 3 for the six-step data synthesis pipeline, Section 4.3 for the memory-augmented agent (MemAgent) architecture, Table 2-3 for performance degradation curves showing RAG collapse vs. agentic robustness.