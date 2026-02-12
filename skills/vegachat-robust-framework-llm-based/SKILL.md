---
name: "vegachat-robust-framework-llm-based"
description: "Generate validated Vega-Lite chart specifications from natural language queries using the VegaChat pipeline: few-shot prompting, schema validation with repair loops, and dual-metric assessment (Spec Score + Vision Score). Trigger phrases: 'create a chart from this data', 'generate a Vega-Lite visualization', 'turn this query into a chart', 'visualize this dataset', 'build a bar/line/scatter chart from natural language', 'assess chart quality with Spec Score'"
---

# VegaChat: Robust NL-to-Vega-Lite Chart Generation and Assessment

This skill enables Claude to convert natural language requests into validated Vega-Lite (v5) JSON specifications using the VegaChat framework. The approach uses few-shot prompting with dataset context, a schema validation and repair loop that achieves near-zero error rates, and two complementary evaluation metrics (Spec Score and Vision Score) for assessing chart quality. Unlike naive code generation, VegaChat treats chart generation as a structured pipeline: analyze the request, map columns, generate a spec, validate against the VL schema, repair if needed, and score the result.

## When to Use

- When the user provides a dataset (CSV, JSON, DataFrame) and asks for a chart or visualization in natural language
- When generating Vega-Lite specifications that must be schema-valid and renderable without manual fixing
- When the user wants to iterate on a chart through multi-turn conversation ("now color it by category", "add a trend line")
- When comparing or assessing the quality of two chart specifications against each other or against a reference
- When building a NL2VIS pipeline or chatbot that produces declarative visualizations
- When the user asks to evaluate whether a generated chart matches a natural language description
- When recommending chart types for a given dataset before the user even asks

## Key Technique

VegaChat addresses two fundamental problems in NL-to-visualization: (1) LLMs frequently produce invalid or empty chart specs, and (2) there is no standard way to measure whether a generated chart is correct. The framework solves both with a generation pipeline that validates and repairs specs, plus two scoring metrics that correlate well with human judgment.

**Generation pipeline.** The system sends the user's natural language request along with the first five rows of the dataset (as text) to an LLM via few-shot prompting at temperature zero. The few-shot examples specifically target features that base models struggle with: data transformations (filter, aggregate, calculate), faceting, and multi-encoding charts. A Request Analyzer maps the user's utterance to dataset columns with confidence scores, flagging missing or ambiguous columns before generation. After the LLM produces a Vega-Lite JSON spec, the system validates it against the VL v5 schema plus custom heuristics. Deterministic repairs are applied first (e.g., normalizing datetime strings like `"2006-01-01"` into structured `{"year": 2006, "month": "jan", "date": 1}` objects). If schema errors persist, the LLM receives Altair's validation trace and retries up to five times. Finally, the system checks for "empty but valid" charts by inspecting Vega's scene graph for the absence of rendered marks.

**Dual-metric assessment.** Spec Score is a deterministic, LLM-free metric that computes a weighted F1 score (beta=2, emphasizing recall) across three components: encoding correctness (highest weight — do x, y, color, size channels match?), mark correctness (is the chart type right?), and transform correctness (are filters/aggregations correct?). It allows partial credit for visually similar marks (e.g., circle vs. point at 50% penalty) and treats x/y axis swaps and row/column facet swaps as equivalent. Vision Score is an image-based metric that renders both charts and sends them to a multimodal LLM, which scores five weighted dimensions (0-2 each): visualization type, data encoding, data transformation, aesthetics, and prompt compliance. Vision Score is library-agnostic — it works on any rendered chart image, not just Vega-Lite.

## Step-by-Step Workflow

1. **Extract dataset context.** Read the user's dataset and format the first five rows as a text table. Identify all column names, infer data types (quantitative, temporal, nominal, ordinal), and flag any ambiguous headers (e.g., a column named "Q1" could be a quarter or a question).

2. **Analyze the natural language request.** Map the user's utterance to specific dataset columns. Assign confidence scores to each mapping. If a requested column doesn't exist (e.g., "plot revenue" but no revenue column), surface this immediately rather than hallucinating.

3. **Select few-shot examples.** Choose 2-4 few-shot examples from a curated bank that match the complexity of the request. Prioritize examples that demonstrate the specific VL features needed: transformations for aggregate queries, faceting for "by category" requests, layered marks for combined charts.

4. **Construct the generation prompt.** Build the system prompt with: (a) the few-shot examples, (b) the dataset context (first 5 rows as text), (c) the instruction to output only valid Vega-Lite v5 JSON, and (d) for multi-turn conversations, concatenate all prior utterances into a single newline-delimited request (this outperforms sequential refinement).

5. **Generate the Vega-Lite specification.** Call the LLM at temperature 0 to produce a JSON spec. Parse the response, extracting the JSON object from any surrounding text or markdown fences.

6. **Validate against VL v5 schema.** Run the spec through Vega-Lite schema validation. Check for: (a) syntax errors (malformed JSON), (b) schema violations (invalid field names, wrong types), and (c) semantic issues (referencing columns not in the dataset).

7. **Apply deterministic repairs.** Fix common issues without re-invoking the LLM: normalize datetime strings to structured objects, correct common field name casing, remove hallucinated properties that aren't in the VL schema.

8. **Run LLM-assisted repair loop if needed.** If validation still fails, send the spec back to the LLM with the full validation error trace. Allow up to 5 retry attempts. On each retry, include the previous error to avoid repeating the same mistake.

9. **Detect empty charts.** Even if the spec is schema-valid, check whether it actually renders visible marks. Use the scene graph (or render to an image and check for content) to catch cases where filters eliminate all data points.

10. **Score the output (when evaluating).** Compute Spec Score by comparing encoding channels, mark type, and transforms against a reference spec using weighted F1 (beta=2). Compute Vision Score by rendering both charts and sending the images to a multimodal model with the five-dimension rubric.

## Concrete Examples

**Example 1: Simple bar chart from CSV data**

User: "Show me the average salary by department" (with a CSV containing columns: name, department, salary, start_date)

Approach:
1. Extract first 5 rows: `name,department,salary,start_date` with sample values
2. Map utterance: "average salary" -> aggregate mean on `salary`, "by department" -> `department` on x-axis
3. Select few-shot example demonstrating aggregate bar charts

Output Vega-Lite spec:
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"url": "data.csv"},
  "mark": "bar",
  "encoding": {
    "x": {"field": "department", "type": "nominal"},
    "y": {"field": "salary", "type": "quantitative", "aggregate": "mean"},
    "tooltip": [
      {"field": "department", "type": "nominal"},
      {"field": "salary", "type": "quantitative", "aggregate": "mean", "format": ",.0f"}
    ]
  }
}
```

**Example 2: Multi-turn refinement with transformation**

User turn 1: "Plot monthly sales over time"
User turn 2: "Filter to only 2024 and color by region"

Approach:
1. Concatenate both utterances: "Plot monthly sales over time\nFilter to only 2024 and color by region"
2. Map: `sales` -> y quantitative (sum aggregate), temporal field -> x, `region` -> color, year filter -> transform
3. Generate spec with timeUnit for monthly grouping and a filter transform

Output Vega-Lite spec:
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {"url": "sales.csv"},
  "transform": [
    {"filter": "year(datum.date) == 2024"}
  ],
  "mark": "line",
  "encoding": {
    "x": {"field": "date", "type": "temporal", "timeUnit": "yearmonth"},
    "y": {"field": "sales", "type": "quantitative", "aggregate": "sum"},
    "color": {"field": "region", "type": "nominal"}
  }
}
```

**Example 3: Assessing chart quality with Spec Score**

User: "Compare my generated chart spec against this reference and tell me how close it is"

Approach:
1. Parse both the generated and reference Vega-Lite specs
2. Compare **encodings**: For each channel (x, y, color, size, shape, row, column), check field name, type, and aggregate. Allow x/y swap as equivalent.
3. Compare **mark type**: Exact match = 1.0, visually similar (circle/point, bar/rect) = 0.5, mismatch = 0.0
4. Compare **transforms**: Match filter predicates, aggregate operations, and calculated fields
5. Compute weighted F1 (beta=2) per component, then weighted sum

Output:
```
Spec Score: 0.82
  - Encoding score: 0.90 (x, y, color all match; missing size encoding in generated)
  - Mark score: 1.00 (both use "bar")
  - Transform score: 0.50 (filter matches, but generated is missing the calculate transform)

The generated spec captures the core chart correctly but omits
a derived field calculation present in the reference.
```

## Best Practices

**Do:**
- Always provide the first 5 rows of the actual dataset as context — the LLM needs to see real column names and value formats to produce correct field references
- Use temperature 0 for deterministic, reproducible chart generation
- Concatenate all turns of a multi-turn conversation into a single prompt rather than refining sequentially — this produces higher Spec Scores (86.5% vs lower for sequential)
- Validate every generated spec against the VL v5 schema before presenting to the user, and apply deterministic repairs before falling back to LLM retries
- Include few-shot examples specifically targeting the VL features the request needs (transformations, faceting, layered marks)

**Avoid:**
- Do not skip the empty-chart detection step — a spec can be schema-valid but render nothing if filters eliminate all data
- Do not hallucinate VL fields or APIs that don't exist in the schema; constrain generation to the documented VL v5 spec
- Do not rely solely on Spec Score or solely on Vision Score — use them together since they capture different failure modes (structural vs. visual)
- Do not use sequential turn-by-turn refinement for multi-turn requests; concatenation is empirically superior
- Do not omit the `$schema` field — it is required for proper Vega-Lite validation and rendering

## Error Handling

| Error | Detection | Resolution |
|-------|-----------|------------|
| Malformed JSON output | JSON parse failure | Strip markdown fences, extract JSON object via regex, retry if still invalid |
| Schema violation | VL schema validator rejects spec | Apply deterministic fixes first (datetime normalization, field casing), then retry with validation trace (up to 5 times) |
| Empty chart | Scene graph shows zero rendered marks | Re-examine filter transforms; relax or remove filters that eliminate all data; alert user if the data genuinely has no matching rows |
| Column not found | Request Analyzer finds no matching column | Report the missing column to the user with suggestions for closest matches (fuzzy match on column names) |
| Ambiguous column mapping | Multiple columns could match the request | Present the ambiguity to the user: "Did you mean column 'Q1_revenue' or 'Q1_quantity'?" |
| Hallucinated VL property | Schema validation flags unknown keys | Remove the property and retry; add the offending property name to a blocklist for the current session |

## Limitations

- **Code-based features out of scope.** Vega-Lite is declarative. Complex programmatic transformations (custom statistical models, multi-table joins, recursive calculations) cannot be expressed in VL and require a code-based approach like Python + matplotlib/plotly instead.
- **Interactive features not assessed.** The current Spec Score and Vision Score metrics do not evaluate interactive elements (selections, tooltips, zoom/pan) — only the static chart structure and appearance.
- **Dataset size constraint.** Only the first 5 rows are sent as context. If the dataset has complex structure that isn't visible in the first 5 rows (e.g., rare categories, outliers), the LLM may produce incorrect mappings.
- **Real-world complexity.** On the more challenging ChartLLM benchmark (real-world scenarios), scores drop significantly (Spec Score 52.6%, Vision Score 56.7%) compared to the NLV Corpus (83.9%, 85.1%), indicating that complex analytical queries remain difficult.
- **Single-dataset assumption.** VegaChat assumes a single flat dataset. Multi-source or relational data must be preprocessed into a single table before use.
- **Vision Score requires a multimodal LLM.** Computing Vision Score needs an image-capable model (e.g., GPT-4o), adding cost and latency compared to the deterministic Spec Score.

## Reference

**Paper:** [VegaChat: A Robust Framework for LLM-Based Chart Generation and Assessment](https://arxiv.org/abs/2601.15385v1) — Hostnik, Kurbanov, Sokolov, Trofimov (2026). Focus on Section 3 (generation pipeline and repair loop), Section 4 (Spec Score and Vision Score formulas), and Table 1 (comparative results showing near-zero error rates).

**Code and artifacts:** https://zenodo.org/records/17062309