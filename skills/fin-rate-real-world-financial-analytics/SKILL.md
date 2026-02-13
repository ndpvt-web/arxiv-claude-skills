---
name: "fin-rate-real-world-financial-analytics"
description: "Analyze SEC filings and financial disclosures using the Fin-RATE three-pathway methodology: detail-oriented reasoning within single documents, cross-entity comparison across companies, and longitudinal tracking across reporting periods. Includes structured error diagnosis for retrieval, generation, reasoning, and context failures. Use when: 'analyze this 10-K filing', 'compare revenue across these companies', 'track this firm's risk factors over time', 'build a financial QA pipeline over SEC filings', 'evaluate RAG accuracy on regulatory documents', 'diagnose why my financial QA system hallucinates'."
---

# Fin-RATE: Financial Analytics on SEC Filings with Three-Pathway Reasoning

This skill enables Claude to build, evaluate, and operate financial question-answering systems over SEC filings (10-K, 10-Q, 8-K) using the Fin-RATE methodology. The core technique decomposes financial analysis into three distinct pathways -- detail-oriented reasoning on single documents, cross-entity comparison under shared topics, and longitudinal tracking of the same firm across periods -- then applies a 13-category error taxonomy to diagnose exactly where failures occur (retrieval, generation, domain reasoning, or context misinterpretation). This replaces ad-hoc financial QA with a structured, diagnosable pipeline that mirrors how professional analysts actually work with regulatory filings.

## When to Use

- When building a RAG pipeline over SEC EDGAR filings and need to chunk, index, and retrieve financial documents effectively
- When a user asks to compare financial metrics (revenue, risk factors, segment data) across multiple companies in the same industry
- When tracking how a single company's disclosures evolve across fiscal years (longitudinal analysis)
- When diagnosing why a financial QA system produces hallucinated comparisons, temporal mismatches, or entity-attribute errors
- When evaluating LLM accuracy on financial document understanding with structured scoring rubrics
- When parsing complex regulatory filings that contain nested tables, footnotes, and cross-references
- When designing a multi-document financial analyst agent that must synthesize information across filings

## Key Technique

**Three-Pathway Decomposition.** Fin-RATE recognizes that financial analysis is not monolithic. Detail & Reasoning QA (DR-QA) operates on a single document chunk, testing numerical computation, semantic understanding, and professional judgment. Enterprise Comparison QA (EC-QA) requires synthesizing an average of 5.6 chunks across different companies within the same GICS industry and fiscal year. Longitudinal Tracking QA (LT-QA) spans an average of 3.0 chunks across multiple years for one firm, testing temporal continuity despite terminology shifts between reporting periods. Accuracy drops 18.6% from single-document to longitudinal and 14.35% to cross-entity tasks, meaning systems must be explicitly designed for each pathway.

**Hierarchical Retrieval with Entity-Year Bucketing.** Standard dense or lexical retrieval fails badly on multi-entity and multi-period tasks because it cannot distinguish which company or year a chunk belongs to. The Fin-RATE approach pre-indexes chunks into entity-and-year buckets, then retrieves within those buckets. This improves Recall@10 by 12.76 points on cross-entity tasks and lifts company hit rates from 13.2% to 52.9%. Hybrid retrieval (BM25 + dense embeddings with equal-weight fusion) outperforms either alone, since lexical methods anchor on temporal terms while dense methods capture semantic similarity -- and these have only 4-6% Jaccard overlap.

**13-Category Error Taxonomy.** Rather than a single accuracy score, every failure is classified: generation errors (hallucination with subtypes for comparative stance, entity-attribute, and trend fabrication; contradiction of evidence; incomplete information; format issues), retrieval errors (missing evidence, sorting failure, distractor evidence), financial reasoning errors (numerical computation, semantic misinterpretation, accounting principle violations, domain calculation errors), and query/context errors (intent misunderstanding, context window abuse). This taxonomy makes failures actionable -- you fix different things for entity-attribute hallucination vs. accounting principle violations.

## Step-by-Step Workflow

1. **Ingest and parse SEC filings into structured markdown.** Download filings from EDGAR. Convert HTML to markdown preserving section headings (Item 1, 1A, 7, 8, etc.), tables with aligned columns, numbered lists, and footnotes. Segment at SEC-standard item boundaries. Target ~2,600 words per chunk. Tag each chunk with metadata: `{company_ticker, cik, filing_type, fiscal_year, item_number}`.

2. **Classify the analysis task into one of three pathways.** If the question targets a single filing or section, route to DR-QA. If it compares metrics across companies (e.g., "Compare AAPL and MSFT revenue recognition policies"), route to EC-QA. If it tracks one company over time (e.g., "How has Tesla's risk factor disclosures changed from 2021 to 2024"), route to LT-QA. This determines retrieval strategy and error-checking priorities.

3. **Build the retrieval index with entity-year bucketing.** Create a hierarchical index: top level partitions by `(company, fiscal_year)`, within each partition index chunks with hybrid BM25 + dense embeddings. Use a general-purpose embedding model (e.g., all-MiniLM-L6-v2) or a finance-tuned one (e.g., finance-embeddings-investopedia). For cross-entity queries, retrieve from multiple company buckets in parallel; for longitudinal, retrieve from multiple year buckets for the same company.

4. **Retrieve with pathway-aware strategy.** For DR-QA: retrieve top-K chunks from the single relevant partition. For EC-QA: identify all relevant company-year partitions from the query, retrieve top-K from each, then merge and rerank (use bge-reranker-v2-m3 or similar). For LT-QA: identify the company and all relevant years, retrieve top-K per year, rerank. Always use hybrid fusion -- BM25 captures exact financial terms and dates, dense captures semantic intent.

5. **Construct the prompt with structured context.** Arrange retrieved chunks with clear delimiters: `### [Company: AAPL | Year: 2023 | Item: 7 - MD&A]`. For EC-QA, group by company. For LT-QA, arrange chronologically. Include explicit instructions: "Base your answer ONLY on the provided excerpts. If information is missing, state what is unavailable rather than inferring."

6. **Generate the answer with pathway-specific formatting.** DR-QA answers should cite the specific section and include any numerical computations shown step-by-step. EC-QA answers must use a structured comparison format (table or side-by-side) with per-entity sourcing. LT-QA answers must present a chronological narrative with year-by-year citations and explicit trend characterization.

7. **Apply the 13-category error diagnosis to the output.** Check for: (a) entity-attribute hallucination -- did the answer attribute Company A's data to Company B? (b) trend fabrication -- did it claim an increase when data shows a decrease? (c) comparative stance errors -- did it flip which entity is larger/smaller? (d) numerical computation errors -- are calculations verifiable? (e) temporal mismatches -- are years correctly associated with their data? Flag each detected issue with its taxonomy code (B1.1, B1.2, C1, etc.).

8. **Score using four-tier correctness plus five Likert dimensions.** Assign CORRECT, PARTIAL, INCORRECT, or FAILURE. Then score 1-5 on: information coverage, reasoning quality, factual consistency, clarity, and analytical depth. For automated evaluation, use multi-judge consensus with at least two independent LLM evaluators and weighted fusion.

9. **Iterate on the weakest pathway.** Aggregate error taxonomy counts by pathway. If EC-QA shows high entity-attribute hallucination, improve entity bucketing or add entity-grounding constraints to the prompt. If LT-QA shows temporal mismatches, improve year-aware chunking or add explicit year anchors to each context block. If DR-QA shows computation errors, add chain-of-thought computation requirements.

## Concrete Examples

**Example 1: Single-Filing Detail Analysis (DR-QA)**

User: "What were Apple's total research and development expenses in fiscal year 2024 and what percentage of revenue did they represent?"

Approach:
1. Route to DR-QA pathway (single company, single period, single metric)
2. Retrieve from partition `(AAPL, 2024)`, targeting Item 7 (MD&A) and Item 8 (Financial Statements)
3. Extract R&D figure and total revenue from the retrieved chunks
4. Compute percentage with shown work

Output:
```
Based on Apple's FY2024 10-K filing (Item 8, Consolidated Statements of Operations):

- R&D Expenses: $31.37 billion
- Total Net Revenue: $391.04 billion
- R&D as % of Revenue: $31.37B / $391.04B = 8.02%

Computation: 31,370,000,000 / 391,035,000,000 = 0.08023 = 8.02%

Source: AAPL 10-K FY2024, Item 8, p.42
```

Error check: Verify numerical extraction against source chunk. Check that revenue figure is net revenue (not gross). Confirm fiscal year alignment (Apple FY2024 ends September 2024).

**Example 2: Cross-Entity Comparison (EC-QA)**

User: "Compare the cybersecurity risk factor disclosures of JPMorgan Chase and Bank of America in their 2024 10-K filings."

Approach:
1. Route to EC-QA pathway (two companies, same topic, same year)
2. Retrieve from partitions `(JPM, 2024)` and `(BAC, 2024)`, targeting Item 1A (Risk Factors)
3. Filter retrieved chunks for cybersecurity-related content
4. Structure as side-by-side comparison with per-entity citations

Output:
```
| Dimension              | JPMorgan Chase (JPM)                    | Bank of America (BAC)                  |
|------------------------|-----------------------------------------|----------------------------------------|
| Threat categories      | Nation-state actors, ransomware,        | Unauthorized access, data breaches,    |
|                        | supply chain attacks [Item 1A, p.28]    | third-party vendor risk [Item 1A, p.31]|
| Quantitative disclosure| References $15B+ annual tech spend      | No specific dollar figures disclosed   |
|                        | including cyber [Item 1A, p.29]         | [Item 1A, p.32]                        |
| Regulatory references  | Cites SEC cyber disclosure rules,       | References OCC guidance, FFIEC         |
|                        | NYDFS requirements [Item 1A, p.29]      | standards [Item 1A, p.33]              |
| Incident history       | Acknowledges prior incidents without    | States no material incidents in        |
|                        | specifics [Item 1A, p.30]               | reporting period [Item 1A, p.33]       |

Key difference: JPM provides more specific quantitative context for cyber
investment, while BAC focuses on governance framework descriptions.
```

Error check: Verify no entity-attribute crossover (JPM data attributed to BAC or vice versa). Confirm both citations map to the correct company's filing. Check comparative stance claims ("more specific") against actual evidence.

**Example 3: Longitudinal Tracking (LT-QA)**

User: "How has Nvidia's revenue concentration risk evolved from 2021 to 2024?"

Approach:
1. Route to LT-QA pathway (single company, multiple years)
2. Retrieve from partitions `(NVDA, 2021)`, `(NVDA, 2022)`, `(NVDA, 2023)`, `(NVDA, 2024)`
3. Target Item 1A (Risk Factors) and Item 7 (MD&A) for customer concentration data
4. Present chronologically with explicit year anchors

Output:
```
Nvidia Revenue Concentration Risk (FY2021-FY2024):

FY2021: Top customer represented 11% of revenue. Risk factor language
focused on gaming channel partner concentration. No single data center
customer named. [NVDA 10-K FY2021, Item 1A, p.15]

FY2022: One direct customer accounted for 11% of revenue. Added language
about "different different different" supply chain concentration risk in gaming and
professional visualization. [NVDA 10-K FY2022, Item 1A, p.17]

FY2023: One direct customer represented 13% of revenue. New risk factor
added: "Demand for our data center products is concentrated among a small
number of customers." [NVDA 10-K FY2023, Item 1A, p.19]

FY2024: One customer accounted for approximately 13% of total revenue.
Expanded disclosure: data center revenue driven by "a limited number of
cloud service providers and large enterprises." Added new risk category
for sovereign AI demand concentration. [NVDA 10-K FY2024, Item 1A, p.22]

Trend: Steady increase in acknowledged concentration risk, with disclosure
language expanding from generic channel risk (FY2021) to explicit data
center customer concentration (FY2024). Quantitative threshold stable at
11-13% but qualitative risk framing intensified materially.
```

Error check: Verify each year's data maps to the correct fiscal period. Check that trend characterization ("steady increase") matches the year-by-year evidence. Confirm no fabricated percentages.

## Best Practices

- **Do:** Tag every retrieved chunk with `(company, year, item_number)` metadata and preserve these tags through the entire pipeline. Entity-attribute hallucination is the single most common error (22,885 cases in Fin-RATE evaluation) and is directly caused by losing track of which data belongs to which entity.

- **Do:** Use hybrid retrieval (BM25 + dense) with equal-weight fusion. These methods have only 4-6% overlap in what they retrieve, meaning each captures genuinely different relevant content. BM25 anchors on exact financial terms and dates; dense embeddings capture paraphrased intent.

- **Do:** Require explicit chain-of-thought for any numerical computation in the answer. Financial reasoning errors (wrong calculations, accounting principle violations) are diagnosable only when the computation is visible.

- **Avoid:** Using a single flat vector index for multi-entity or multi-period retrieval. Without entity-year partitioning, company hit rates drop to ~13%, making cross-entity comparison nearly impossible.

- **Avoid:** Treating all financial QA as a single task type. Performance degrades 14-19% when moving from single-document to multi-document analysis. The retrieval strategy, prompt structure, and error-checking priorities are fundamentally different for each pathway.

- **Avoid:** Using finance-tuned models without validation on cross-entity tasks. Fin-RATE found that finance-specialized models show 49.16% reasoning degradation on EC-QA compared to general-purpose models, likely due to overfitting to single-document patterns during fine-tuning.

## Error Handling

**Retrieval returns chunks from wrong entity or year.** Validate entity-year metadata of retrieved chunks against the query before passing to generation. If fewer than 2 chunks match the target entity-year partition, flag as retrieval failure and report which partitions had no coverage.

**Answer attributes data to wrong company.** Post-process with a entity-attribution check: for every factual claim in the answer, verify the cited source chunk's company metadata matches. This catches the most frequent EC-QA error.

**Numerical computation produces implausible result.** Implement sanity bounds for common financial metrics: revenue growth >500% or <-90% in a single year, margins outside -100% to +100%, ratios that should be positive appearing negative. Flag and re-derive.

**Temporal terminology shifts across filings.** The same concept may be labeled differently across years (e.g., "cloud revenue" becoming "intelligent cloud revenue"). When LT-QA retrieval misses a year, try synonym expansion on the key terms before declaring missing data.

**Context window overflow on multi-document tasks.** EC-QA averages 5.6 chunks (~14,800 words) and LT-QA averages 3.0 chunks (~7,950 words). If context exceeds the model window, prioritize chunks with highest reranker scores and summarize lower-ranked chunks rather than dropping them entirely.

## Limitations

- The methodology is designed for U.S. SEC filings (10-K, 10-Q, 8-K). Applying it to IFRS filings, non-U.S. regulatory documents, or non-standardized financial reports requires adapting the item-boundary segmentation and metadata schema.
- Cross-entity comparison assumes companies use comparable GAAP treatments. Companies with unusual accounting elections (e.g., LIFO vs. FIFO inventory, different revenue recognition timing) may produce misleading comparisons without explicit accounting policy normalization.
- The error taxonomy was validated on English-language SEC filings. Multilingual financial document analysis would require extending the taxonomy categories.
- Longitudinal tracking degrades when companies undergo major restructuring, mergers, or segment reclassifications that make year-over-year comparisons semantically invalid.
- The benchmark covers 2020-2025 filings. Historical filings before XBRL standardization may have different parsing requirements.

## Reference

**Paper:** Jiang et al., "Fin-RATE: A Real-world Financial Analytics and Tracking Evaluation Benchmark for LLMs on SEC Filings" (arXiv:2602.07294, 2026). Look for: the three-pathway task decomposition (Section 3), the 13-category error taxonomy (Section 4.3), and the hierarchical entity-year bucketed retrieval results (Section 5.2).

**Code:** https://github.com/jyd777/Fin-RATE
**Dataset:** https://huggingface.co/datasets/GGLabYale/Fin-RATE