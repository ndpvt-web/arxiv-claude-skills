---
name: "datacross-unified-benchmark-agent"
description: "Cross-modal data analysis agent that unifies structured sources (SQL, CSV, JSON) with unstructured visual documents (scanned PDFs, invoice images, chart screenshots) using divide-and-conquer sub-agents and iterative code generation. Triggers: 'analyze data from these mixed sources', 'combine CSV with scanned PDF table', 'extract table from image and join with database', 'cross-modal data analysis', 'zombie data activation', 'analyze heterogeneous data files together'"
---

# DataCross: Cross-Modal Heterogeneous Data Analysis Agent

This skill enables Claude to perform unified analysis across heterogeneous data modalities — combining directly queryable structured sources (SQL databases, CSV files, JSON) with high-value information locked inside unstructured visual documents (scanned reports, invoice images, chart screenshots). It implements the DataCrossAgent divide-and-conquer framework: assign specialized sub-agents per data source, profile each source independently via iterative code generation (reReAct), score sources by priority, then cross-pollinate findings across modalities to produce factually grounded, insight-driven analysis.

## When to Use

- When the user provides a mix of CSV/SQL files alongside scanned PDFs, images of tables, or chart screenshots and asks for a combined analysis
- When the user asks to "extract a table from this image" and then join or correlate it with a structured dataset
- When the user needs to answer questions that require reasoning across multiple files of different formats (e.g., "compare Q3 revenue in this spreadsheet with the figures in this scanned report")
- When the user has "zombie data" — valuable information trapped in visual documents that needs to be activated and linked to queryable databases
- When the user asks for multi-step analytical workflows spanning heterogeneous sources (finance reports + transaction CSVs, medical images + patient records)
- When the user needs robust, self-debugging code generation for data analysis that handles schema mismatches, encoding issues, and cross-source alignment

## Key Technique

**Divide-and-Conquer with Specialized Sub-Agents.** Rather than feeding all heterogeneous data into a single prompt, DataCrossAgent assigns each data source its own specialized "sub-agent" context. Each sub-agent performs deep exploration of its assigned source — profiling schemas, generating heuristic questions, and writing executable Python code to extract insights. This isolation prevents errors in one modality (e.g., OCR noise from a scanned PDF) from contaminating analysis of clean structured data.

**reReAct (Recursive Reasoning-Act).** Standard ReAct chains a single Thought-Action-Observation loop. reReAct adds a dual-loop structure: an outer reasoning layer decomposes the full task into a tree of sub-problems, while an inner loop handles iterative code generation, execution, error detection, and self-repair for each sub-problem. When code fails (wrong column name, type mismatch, empty DataFrame), the inner loop retries with corrected code rather than abandoning the entire chain. Ablation studies show this mechanism alone accounts for ~20% of factuality improvement.

**Priority Scoring and Cross-Pollination.** After independent exploration, sources are ranked using a hybrid priority score: `S_priority = 0.4 * S_obj + 0.3 * S_sem + 0.3 * S_LLM`, where S_obj measures data richness (completeness, column count, temporal indicators), S_sem measures keyword overlap between the analysis goal and source schema, and S_LLM is a high-level relevance judgment. High-scoring sources become "Primary Data" (the analytical pivot), while others become "Auxiliary Data." A Cross-Analysis Agent then generates executable code to physically merge, correlate, and statistically test hypotheses across sources.

## Step-by-Step Workflow

1. **Inventory all data sources.** List every file the user provides. Classify each as structured (CSV, SQL, JSON, Excel) or unstructured-visual (scanned PDF, image of table, chart screenshot, invoice photo). Record file paths, formats, and apparent domains.

2. **Extract tables from visual documents.** For each image or scanned PDF containing tabular data, use a classify-then-extract strategy:
   - **Tables/grids**: Parse into a pandas DataFrame using vision-based extraction (describe the visual grid, identify headers and rows, output as structured data).
   - **Charts/visualizations**: Generate a semantic text summary capturing trends, axis labels, data points, and key takeaways.
   - **Text metadata**: Extract and compress contextual text (titles, footnotes, annotations).

3. **Profile each structured source independently.** For each CSV/SQL/JSON file, generate and execute Python code to:
   - Load with robust encoding fallback (`utf-8` -> `latin1` -> `gbk`)
   - Inspect schema: column names, data types, row count, missing-value rates
   - Compute summary statistics and identify temporal columns
   - Generate 2-3 heuristic questions the source can answer relevant to the user's goal

4. **Score sources by priority.** Compute a hybrid priority score for each source:
   - **Objective Richness (40%)**: `(1 - missing_rate) * column_richness * row_richness * temporal_bonus`
   - **Semantic Relevance (30%)**: Keyword overlap between user's question and column names/values
   - **Subjective Assessment (30%)**: Your judgment of how central this source is to the analysis goal
   - Rank sources as Primary (analytical pivot) or Auxiliary.

5. **Deep-explore primary sources via reReAct.** For each primary source, iteratively:
   - Reason about what sub-question to answer
   - Generate Python code to answer it
   - Execute and observe results
   - If code fails (KeyError, TypeError, empty result), diagnose the error and regenerate corrected code
   - Repeat until the sub-question is answered or max retries (3) exhausted

6. **Generate a Cross-Source Analysis Checklist.** Based on findings from each source, formulate specific hypotheses that require cross-source verification. Example: "Does the revenue decline in Q3 (from CSV) correlate with the shipping delays mentioned in the scanned logistics report (from image)?"

7. **Execute cross-source merges and tests.** Write Python code to:
   - Align keys across sources (normalize date formats, entity names, units)
   - Perform physical joins/merges on aligned DataFrames
   - Run statistical tests or compute correlations to verify each hypothesis
   - Handle schema mismatches with explicit column mapping

8. **Synthesize findings into a structured report.** Combine results into a coherent analysis covering:
   - Key factual findings with source attribution (which file/table each fact came from)
   - Cross-source insights that could not have been derived from any single source alone
   - Confidence indicators noting where extraction was uncertain (e.g., low OCR confidence)

9. **Validate factuality.** Re-check all numerical claims against source data. Ensure no hallucinated statistics by tracing each number back to executed code output.

10. **Present results with reproducible code.** Provide the user with both the narrative analysis and the complete Python code used, so they can re-run, modify, or extend the analysis.

## Concrete Examples

**Example 1: Financial Report + Transaction CSV**

```
User: I have Q3_transactions.csv with daily sales data and this scanned_annual_report.png
which shows a summary table of quarterly performance. Can you tell me whether the
transaction-level data is consistent with the annual report figures?

Approach:
1. Extract the quarterly summary table from scanned_annual_report.png into a DataFrame
   with columns [Quarter, Revenue, Expenses, Net_Profit].
2. Load Q3_transactions.csv, handle encoding, inspect schema.
3. Aggregate Q3 transactions: total revenue = sum of 'amount' where type='sale',
   total expenses = sum of 'amount' where type='expense'.
4. Compare aggregated Q3 totals against the Q3 row extracted from the image.
5. Report discrepancies with exact figures and percentage differences.

Output:
## Consistency Check: Q3 Transactions vs. Annual Report

| Metric     | CSV Aggregated | Report (extracted) | Delta   | Status |
|------------|---------------:|-------------------:|--------:|--------|
| Revenue    |     $2,341,500 |         $2,350,000 |  +0.36% | Match  |
| Expenses   |     $1,890,200 |         $1,875,000 |  -0.81% | Review |
| Net Profit |       $451,300 |           $475,000 |  +5.25% | Flag   |

Revenue aligns within rounding tolerance. Expense discrepancy of $15,200
suggests the scanned report may exclude certain line items. Net profit
divergence follows from the expense gap. Recommend auditing expense
categories not present in the CSV (e.g., depreciation, amortization).
```

**Example 2: Healthcare — Patient Records + Lab Report Images**

```
User: I have patient_visits.csv and three photos of lab result printouts
(lab_001.jpg, lab_002.jpg, lab_003.jpg). Which patients have abnormal lab
values and how do those correlate with their visit frequency?

Approach:
1. Extract structured tables from each lab image:
   - Identify patient_id, test_name, result_value, reference_range, flag
2. Load patient_visits.csv, profile schema (patient_id, visit_date, diagnosis).
3. Score sources: patient_visits.csv is Primary (richer, more rows),
   lab images are Auxiliary (fewer records but critical clinical values).
4. Merge on patient_id after normalizing ID formats (strip leading zeros).
5. Flag abnormal results where result_value falls outside reference_range.
6. Compute visit frequency per patient, correlate with abnormal flag count.

Output:
## Cross-Modal Patient Analysis

Extracted 47 lab results from 3 images (42 parsed successfully, 5 flagged
for low-confidence extraction).

| Patient | Abnormal Labs | Visit Freq (6mo) | Correlation |
|---------|--------------|-------------------|-------------|
| P-1042  | 3 (glucose, HbA1c, triglycerides) | 8 visits | High-freq + metabolic flags |
| P-0887  | 1 (WBC elevated) | 12 visits | Highest freq, possible chronic condition |
| P-1205  | 2 (ALT, AST) | 2 visits | Low freq despite liver markers — follow up |

[Full code provided below for reproducibility]
```

**Example 3: Multi-Source Business Intelligence**

```
User: Analyze these together: sales.db (SQLite), customer_feedback.json,
and this screenshot of a competitor pricing table (competitor_prices.png).
What pricing opportunities exist?

Approach:
1. Extract competitor pricing table from image → DataFrame with
   [product, competitor_price, features].
2. Query sales.db: SELECT product, avg(price), sum(quantity), sum(revenue)
   GROUP BY product.
3. Parse customer_feedback.json: extract sentiment scores and
   price-related complaint frequency per product.
4. Priority scoring: sales.db is Primary (richest, most granular),
   competitor image is Auxiliary, feedback is Auxiliary.
5. Cross-pollinate: join all three on product name (fuzzy match where needed).
6. Identify products where our_price > competitor_price AND sentiment < 0.5.
7. Identify products where our_price < competitor_price AND high volume
   (potential margin opportunity).

Output:
## Pricing Opportunity Analysis

### Overpriced (risk of churn):
- Widget Pro: Ours $49.99 vs Competitor $39.99 (+25%), sentiment 0.31,
  18% of complaints mention "too expensive"
  → Recommend price reduction to $42.99

### Underpriced (margin opportunity):
- Basic Plan: Ours $9.99 vs Competitor $14.99 (-33%), sentiment 0.82,
  highest volume product, zero price complaints
  → Recommend test increase to $11.99 (projected +$48K/quarter)
```

## Best Practices

- **Do:** Always trace every numerical claim back to executed code output. Never state a number you haven't computed or extracted programmatically.
- **Do:** Use robust encoding fallback chains when loading CSVs (`utf-8` -> `latin1` -> `gbk`). Real-world files frequently have encoding issues.
- **Do:** Verify exact column names from the actual data before writing join/filter code. Schema assumptions from visual extraction are error-prone.
- **Do:** Clearly distinguish high-confidence extractions (clean CSV data) from low-confidence ones (OCR'd tables) in your output.
- **Avoid:** Merging sources on assumed key columns without first checking for format mismatches (leading zeros in IDs, date format differences, currency symbols).
- **Avoid:** Treating chart screenshots as tabular data — extract semantic descriptions (trends, comparisons) rather than forcing them into DataFrames.
- **Avoid:** Running a single monolithic code block for the entire analysis. Use the reReAct pattern: small code blocks, execute, observe, fix, iterate.

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| `KeyError` on column access | Column name mismatch between assumed schema and actual data | Re-inspect actual columns with `df.columns.tolist()`, use fuzzy matching or manual mapping |
| Empty DataFrame after merge | Join keys don't align (format mismatch) | Normalize join keys: strip whitespace, standardize case, convert date formats before retry |
| Image extraction returns garbled text | Low-resolution scan, complex table layout, or watermarks | Fall back to describing the image semantically; ask user to provide higher-quality scan if critical |
| Encoding error on CSV load | Non-UTF-8 file encoding | Try encoding fallback chain; if all fail, read as binary and detect with `chardet` |
| Type mismatch in aggregation | Numeric columns stored as strings (e.g., "$1,234") | Strip currency symbols and commas, cast to float before aggregation |
| Code execution timeout | Large dataset or expensive join | Sample data first to validate logic, then run on full dataset; use chunked processing if needed |

## Limitations

- **Image table extraction quality depends on input quality.** Blurry scans, handwritten tables, or complex merged-cell layouts will produce unreliable extractions. Always flag confidence levels.
- **No native OCR execution.** Claude cannot run Tesseract or similar OCR engines. For image tables, Claude relies on its vision capabilities, which work well for clean printed tables but degrade on noisy inputs.
- **Cross-source alignment requires shared keys.** If the structured and visual sources share no common identifiers (no matching product names, dates, or IDs), automated merging is not possible without user-provided mapping rules.
- **Scale limits.** This workflow is designed for analytical tasks (hundreds to low thousands of rows). For million-row datasets, the iterative code-execute-observe loop becomes impractical — recommend database-side aggregation first.
- **Domain knowledge gaps.** The priority scoring formula works best when the analysis goal is clearly stated. Vague goals like "find interesting patterns" make semantic relevance scoring unreliable.

## Reference

**Paper:** [DataCross: A Unified Benchmark and Agent Framework for Cross-Modal Heterogeneous Data Analysis](https://arxiv.org/abs/2601.21403v1) — Qi, Liu, Zhang (2026). Look for: the reReAct dual-loop mechanism (Section 4), the hybrid priority scoring formula (Section 4.2), and the cross-pollination checklist pattern (Section 4.3) for the core implementation details that drive the 29.7% factuality improvement over single-agent baselines.