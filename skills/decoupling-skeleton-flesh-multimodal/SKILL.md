---
name: "decoupling-skeleton-flesh-multimodal"
description: "Disentangled structure-content reasoning for table images and structured data. Separates table skeleton (layout/structure) from flesh (cell content) to answer questions accurately. Use when: 'analyze this table image', 'answer questions about this spreadsheet screenshot', 'extract data from this table photo', 'reason over this financial table', 'compare values in this table image', 'what does this table show'."
---

# Decoupling Skeleton and Flesh: Disentangled Table Reasoning

This skill enables Claude to reason over table images and structured tabular data by applying the DiSCo + Table-GLS framework from Zhu et al. (2026). The core idea: instead of trying to understand a table all at once, explicitly **decouple structure (skeleton) from content (flesh)** first, then use a **global-to-local** reasoning pipeline that narrows focus from the full table to a minimal evidence sub-table before answering. This dramatically reduces errors from misaligned rows/columns, merged cells, and complex hierarchical headers.

## When to Use

- When the user provides a **table image** (screenshot, photo, scan) and asks questions about its contents
- When the user asks to **extract structured data** from a visual table representation
- When reasoning requires **cross-referencing multiple cells** (e.g., "which product had the highest growth between Q2 and Q4?")
- When tables have **complex layouts**: merged cells, multi-level headers, hierarchical row labels, or irregular grids
- When the user needs to **compare, aggregate, or compute** values scattered across a table image
- When building a **table QA pipeline** that must handle diverse, unseen table formats robustly
- When converting a **table image to structured format** (JSON, CSV, DataFrame) with layout fidelity

## Key Technique: Disentangled Alignment + Structure-Guided Reasoning

### The Problem with Naive Table Reading

Large vision-language models tend to entangle structure and content when reading tables. They might correctly OCR individual cell values but misattribute which row or column a value belongs to--especially in tables with merged headers, irregular spacing, or dense numeric data. This leads to confidently wrong answers.

### DiSCo: Skeleton-First, Then Flesh

The DiSCo framework decouples table understanding into two alignment phases:

1. **Structural Abstraction (Skeleton):** Recognize the table's layout independent of content. Identify row/column boundaries, header hierarchy, span markers, and cell grid coordinates. Think of this as producing an anonymized template: `Row 1, Col 1: [CELL] | Row 1, Col 2: [CELL] | ...` with headers and merge spans preserved but values blanked out. This forces attention to layout geometry.

2. **Semantic Grounding (Flesh):** Bind actual cell content to structural coordinates at two granularities:
   - *Global:* Summarize what each row/column represents semantically (e.g., "Column 3 contains quarterly revenue in USD millions")
   - *Local:* Retrieve the specific value at a given (row, column) coordinate (e.g., "Row 4, Column 3 = $2.7M")

This separation ensures the model builds a reliable structural map before filling in content, preventing the common failure of reading correct values from incorrect cells.

### Table-GLS: Global-to-Local Structured Reasoning

Once structure and content are disentangled, Table-GLS performs reasoning in three stages:

1. **Global Structure Exploration (GSE):** Analyze the full table layout, identify which columns and rows are relevant to the question, and form an initial retrieval plan.
2. **Self-Refined Sub-table Extraction (SSE):** Critically evaluate the initial plan, refine it if needed, then extract a minimal sub-table containing only the evidence cells required to answer.
3. **Evidence-Grounded Reasoning (EGR):** Answer the question using *only* the extracted sub-table as evidence, with explicit step-by-step reasoning grounded in specific cell references.

## Step-by-Step Workflow

1. **Receive the table input.** Accept a table image (PNG/JPG/PDF screenshot) or a structured table (HTML/Markdown/CSV). If the input is an image, read it visually; if structured text, parse it directly.

2. **Extract the skeleton (structural abstraction).** Map out the table's geometry: number of rows and columns, header rows, header columns, any merged/spanning cells, and hierarchical header levels. Produce a coordinate grid like `(row_index, col_index) -> cell_boundary`. Do NOT read cell values yet--focus only on layout.

3. **Annotate semantic roles globally.** For each column header and row label, summarize its semantic role: what kind of data it contains, units, time periods, categories. Example: "Col 2 = 'Revenue (USD, millions)', Col 3 = 'YoY Growth (%)'."

4. **Ground content locally.** For each cell in the grid, bind the actual value to its structural coordinate. Produce entries like: `Row 3, Col 2: 14.7`. Verify a sample of bindings by cross-referencing headers--e.g., confirm that the value at (Row 3, Col 2) makes sense given "Row 3 = Q3 2025" and "Col 2 = Revenue."

5. **Parse the user's question.** Identify what the question asks for: specific lookup, comparison, aggregation, trend analysis, or multi-step computation. Determine which columns and rows are relevant.

6. **Global structure exploration.** Based on the question, select the target column headers and row label conditions that scope the relevant region. Output this as a structured plan: `{ "reasoning": "...", "target_columns": [...], "target_rows": [...] }`.

7. **Self-refine and extract sub-table.** Critically review the plan: are the selected columns/rows sufficient? Are any missing? Revise if needed, then extract a minimal sub-table containing only the relevant cells in `Row R, Col C: value` format.

8. **Evidence-grounded reasoning.** Using ONLY the extracted sub-table, perform step-by-step reasoning to answer the question. Every arithmetic operation or comparison must cite specific cell coordinates. Do not hallucinate values outside the sub-table.

9. **Format and return the answer.** Present the answer clearly, followed by the supporting evidence (the sub-table and reasoning chain). If the question requires a specific format (number, percentage, name), comply exactly.

10. **Handle uncertainty.** If the table image is ambiguous (blurry cells, unclear boundaries), flag which specific cells have uncertain readings and state confidence levels rather than guessing silently.

## Concrete Examples

**Example 1: Financial Table Question Answering**

User: *[Provides a screenshot of a quarterly earnings table]* "Which quarter had the highest net income, and by how much did it exceed the next highest?"

Approach:
1. **Skeleton extraction:** Identify 5 columns (Quarter, Revenue, COGS, Operating Income, Net Income) and 4 data rows (Q1-Q4) plus a header row.
2. **Semantic annotation:** Col 5 = "Net Income (USD millions)"; Rows 1-4 = Q1 2025 through Q4 2025.
3. **Content grounding:** Bind values -- Row 1/Col 5: 12.3, Row 2/Col 5: 15.1, Row 3/Col 5: 14.7, Row 4/Col 5: 18.2.
4. **GSE plan:** Target column = "Net Income", target rows = all quarters (comparison needed).
5. **Sub-table extraction:**
   ```
   Row 1, Col 5 (Q1, Net Income): 12.3
   Row 2, Col 5 (Q2, Net Income): 15.1
   Row 3, Col 5 (Q3, Net Income): 14.7
   Row 4, Col 5 (Q4, Net Income): 18.2
   ```
6. **Evidence-grounded reasoning:** Max = 18.2 (Q4). Second highest = 15.1 (Q2). Difference = 18.2 - 15.1 = 3.1.

Output:
```
Q4 2025 had the highest net income at $18.2M, exceeding Q2 ($15.1M)
by $3.1M.

Evidence sub-table:
  Q1: $12.3M | Q2: $15.1M | Q3: $14.7M | Q4: $18.2M
```

**Example 2: Complex Table with Merged Headers**

User: *[Provides a table image with two-level headers: "2024" spanning cols 2-3, "2025" spanning cols 4-5, with sub-headers "H1" and "H2" under each year]* "What was the total sales across all periods for Product B?"

Approach:
1. **Skeleton extraction:** Detect merged header cells. Level-1 headers: col 1 = "Product", cols 2-3 = "2024", cols 4-5 = "2025". Level-2 sub-headers: col 2 = "H1", col 3 = "H2", col 4 = "H1", col 5 = "H2".
2. **Composite column names:** Col 2 = "2024/H1", Col 3 = "2024/H2", Col 4 = "2025/H1", Col 5 = "2025/H2".
3. **Locate Product B row:** Row 3 (assuming Row 1-2 are headers, Row 3 = Product A, Row 4 = Product B, etc.) -- verify by reading row labels.
4. **Sub-table extraction:**
   ```
   Row 4, Col 2 (Product B, 2024/H1): 340
   Row 4, Col 3 (Product B, 2024/H2): 410
   Row 4, Col 4 (Product B, 2025/H1): 385
   Row 4, Col 5 (Product B, 2025/H2): 460
   ```
5. **Reasoning:** Total = 340 + 410 + 385 + 460 = 1,595.

Output:
```
Total sales for Product B across all periods: 1,595

Breakdown:
  2024 H1: 340 | 2024 H2: 410 | 2025 H1: 385 | 2025 H2: 460
  Sum: 340 + 410 + 385 + 460 = 1,595
```

**Example 3: Table Image to Structured Data Conversion**

User: *[Provides a photo of a handwritten lab results table]* "Convert this to a CSV I can use in Python."

Approach:
1. **Skeleton extraction:** Identify grid lines/boundaries despite handwriting irregularity. Determine 4 columns and 6 rows (1 header + 5 data).
2. **Header grounding:** Col 1 = "Sample ID", Col 2 = "pH", Col 3 = "Temp (C)", Col 4 = "Result".
3. **Cell-by-cell content grounding:** Read each cell, flagging any ambiguous characters (e.g., "Is this a 7 or a 1?").
4. **Produce CSV with confidence notes.**

Output:
```csv
Sample ID,pH,Temp (C),Result
A-001,7.2,23.5,Positive
A-002,6.8,24.1,Negative
A-003,7.4,22.9,Positive
A-004,6.9,23.8,Negative
A-005,7.1,24.3,Positive
```
```
Note: Cell (Row 3, Col 2) was difficult to read -- interpreted as
"7.4" but could be "7.1". Please verify against the original.
```

## Best Practices

- **Do:** Always extract structure BEFORE content. Resist the temptation to read cell values on the first pass. Map the grid geometry first, then fill in values. This prevents the most common class of table reasoning errors.
- **Do:** Use explicit coordinate references (Row R, Col C) throughout your reasoning chain. This makes errors traceable and verifiable.
- **Do:** Extract a minimal sub-table before reasoning. Narrowing to only relevant cells reduces noise and prevents the model from getting confused by irrelevant data.
- **Do:** Self-check your structural plan before extracting. Ask: "Are these columns and rows actually sufficient to answer the question? Am I missing a comparison column?"
- **Avoid:** Reading the entire table into one flat text dump and then trying to reason over it. This entangles structure and content and leads to misalignment errors in large tables.
- **Avoid:** Skipping the self-refinement step in sub-table extraction. The initial retrieval plan is often incomplete--always verify and revise before answering.
- **Avoid:** Citing values without structural coordinates. Saying "revenue was 14.7" without specifying which row/column invites hallucination and makes verification impossible.

## Error Handling

| Problem | Solution |
|---|---|
| **Blurry or low-resolution table image** | Extract skeleton first (grid lines are often still visible). For unclear cell values, provide best-guess with explicit uncertainty markers: "Row 2, Col 3: ~47 (uncertain, possibly 41)" |
| **Merged cells break grid assumptions** | During skeleton extraction, explicitly record span information: "Cell at (1,2) spans columns 2-4". Adjust coordinate system to use composite headers. |
| **Question requires data outside the visible table** | State clearly which cells are needed but not present. Do not fabricate values. Suggest what additional information the user should provide. |
| **Inconsistent/contradictory cell values** | Flag the inconsistency with coordinates: "Row 3 total (Col 6) shows 150, but summing Cols 2-5 gives 148. Proceeding with individual cell values." |
| **Table is rotated or skewed in image** | Note the orientation issue, attempt to read with adjusted orientation, and flag reduced confidence in structural mapping. |

## Limitations

- **Image quality floor:** If the table image is severely degraded (very low resolution, heavy occlusion, extreme distortion), structural extraction will fail. The technique cannot recover information that is not visually present.
- **Very large tables (50+ rows/columns):** The global-to-local pipeline helps, but extremely large tables may still overwhelm context. In such cases, ask the user to crop to the relevant region or provide the data in structured format.
- **Non-grid layouts:** This technique assumes tabular structure (rows and columns). It does not apply to freeform infographics, flow charts, or Sankey diagrams that lack grid geometry.
- **Multi-table images:** If an image contains multiple distinct tables, each table must be processed separately. The skeleton extraction step assumes a single table grid.
- **Handwriting variability:** Cell content extraction from handwritten tables is inherently more error-prone than from printed/digital tables. Always flag confidence levels for handwritten inputs.
- **This is a reasoning strategy, not a model fine-tune.** The skill applies DiSCo/Table-GLS as a structured prompting and reasoning methodology. It does not replicate the paper's LoRA training procedure.

## Reference

**Paper:** Zhu, Y., Bai, X., Chen, K., Xiang, Y., & Pan, Y. (2026). *Decoupling Skeleton and Flesh: Efficient Multimodal Table Reasoning with Disentangled Alignment and Structure-aware Guidance.* arXiv:2602.03491v1. [https://arxiv.org/abs/2602.03491v1](https://arxiv.org/abs/2602.03491v1)

**What to look for:** Section 3 details the DiSCo alignment framework (structural abstraction via anonymized templates, global/local content grounding). Section 4 describes the Table-GLS three-stage reasoning pipeline (GSE, SSE, EGR). Appendix C contains the exact prompt templates used at each stage.