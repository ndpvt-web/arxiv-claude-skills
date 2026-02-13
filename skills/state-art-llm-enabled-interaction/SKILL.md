---
name: "state-art-llm-enabled-interaction"
description: "Build LLM-powered natural language interfaces for data visualization — NL2VIS pipelines, conversational chart analytics, multimodal chart explanation, and visualization recommendation systems. Use when: 'build a chat interface for my dashboard', 'let users query data in plain English', 'generate charts from natural language', 'add conversational analytics to my app', 'explain this chart automatically', 'recommend visualizations for this dataset'."
---

This skill enables Claude to design and implement LLM-integrated visualization systems based on the classification framework from Brossier et al.'s survey of 48 papers on LLM-enabled visualization interaction. It covers the full pipeline — from translating natural language queries into data operations and visualization specifications, to generating chart explanations and supporting multi-turn conversational analytics — using proven architectural patterns (RAG-grounded spec generation, agent-based tool orchestration, iterative refinement loops) and evaluation strategies drawn from the surveyed literature.

## When to Use

- When the user wants to build a natural-language-to-visualization (NL2VIS) pipeline that converts plain English queries into Vega-Lite, ECharts, Plotly, or D3 specs
- When adding conversational analytics to an existing dashboard so users can ask follow-up questions about their data
- When building a chart explanation system that describes trends, outliers, and patterns in generated or uploaded visualizations
- When implementing a visualization recommendation engine that suggests chart types based on data characteristics and user intent
- When designing a multimodal interface that accepts both text queries and chart images (e.g., "change the color of that bar chart" while pointing at a rendered chart)
- When creating accessible data exploration tools where users interact via voice or natural language instead of dropdown filters
- When building an agent-based analytics assistant that chains data querying, transformation, visualization, and explanation steps

## Key Technique

The paper's core contribution is a **six-dimension classification framework** that maps how LLMs participate across the visualization pipeline. The six dimensions are: (1) application domain, (2) visualization task, (3) visualization representation, (4) interaction modality, (5) LLM integration method, and (6) system evaluation. The actionable insight is that effective systems do not use LLMs monolithically — they assign LLMs specific roles at specific pipeline stages and combine them with deterministic validation layers.

The pipeline has four stages where LLMs act differently. In **data discovery/preparation**, the LLM translates natural language to SQL or filters (NL2SQL). In **specification generation**, it produces declarative visualization specs (Vega-Lite JSON, Plotly configs) grounded by schema context retrieved via RAG. In **rendering**, the LLM steps aside — deterministic engines (Vega, D3, Matplotlib) handle pixel output. In **explanation/refinement**, the LLM generates narrative summaries of visual patterns and handles multi-turn dialogue with conversation memory for reference resolution ("show the same view for Q3").

The most reliable pattern identified across the 48 papers is **few-shot prompting with schema-grounded RAG and post-generation validation**. Free-form LLM generation hallucinates column names, invents data, and produces invalid specs. Systems that embed the data schema, sample rows, and 3-5 similar query-spec exemplars into the prompt — then validate the output against the actual schema before rendering — achieve substantially higher accuracy. Agent-based architectures that loop (generate → validate → fix → re-validate) further reduce errors.

## Step-by-Step Workflow

1. **Characterize the data source.** Inspect the database schema or DataFrame structure — extract column names, data types, cardinalities, sample values, and table relationships. Store this as a retrievable context document (JSON or markdown).

2. **Classify the user's intent against visualization tasks.** Map the natural language query to one of: comparison, composition, distribution, relationship, trend, or geospatial. Use the query's verbs and nouns ("compare sales by region" → comparison + categorical dimension). This determines candidate chart types.

3. **Select the LLM integration strategy.** For simple, well-structured queries, use few-shot prompting with 3-5 exemplars matching the query pattern. For domain-specific terminology or internal schemas, use RAG to retrieve relevant schema fragments and prior query-spec pairs. For multi-step analytical workflows (filter → aggregate → visualize → explain), use an agent loop with tool calls.

4. **Construct the grounded prompt.** Include: (a) the data schema with types, (b) 2-5 sample rows, (c) the user's natural language query, (d) few-shot exemplars of similar queries mapped to valid specs, (e) the target output format (e.g., Vega-Lite JSON). Instruct the LLM to reason step-by-step before outputting the spec.

5. **Generate the visualization specification.** Call the LLM to produce a declarative spec (Vega-Lite, Plotly JSON, ECharts option, or code). Require structured JSON output — not freeform text with embedded code blocks.

6. **Validate the specification against the schema.** Check that every referenced column exists in the data, data types match the encoding channels (quantitative for axes, nominal for color), and aggregation functions are valid. Reject and re-prompt on failure, including the specific validation error in the retry prompt.

7. **Execute data queries and render.** Run any generated SQL/pandas operations against the actual data source. Pass the validated spec to the rendering engine. Capture the rendered output (SVG, PNG, or interactive HTML).

8. **Generate the explanation layer.** Prompt the LLM with the spec + actual query results to produce a natural language summary: what the chart shows, key patterns (trends, outliers, clusters), and caveats. Ground the explanation in actual data values — never let the LLM invent numbers.

9. **Support iterative refinement.** Store the conversation history (queries, specs, explanations) in a context window or memory store. When the user says "change that to a line chart" or "filter to just 2024," resolve references against the prior turn and modify the existing spec rather than regenerating from scratch.

10. **Evaluate with task-specific metrics.** Measure spec correctness (does the generated spec match a reference or pass schema validation?), explanation faithfulness (do stated numbers match the data?), and interaction quality (can users accomplish analytical tasks in fewer turns?). Use automated validation for spec correctness and spot-check explanations against source data.

## Concrete Examples

**Example 1: NL2VIS — Natural language to Vega-Lite chart**

User: "Build me a function that takes a user's question about a sales CSV and returns a Vega-Lite spec."

Approach:
1. Read the CSV schema (columns: date, region, product, revenue, quantity)
2. Build a prompt template that injects schema + sample rows + few-shot exemplars
3. Implement spec generation with JSON-mode output parsing
4. Add schema validation post-generation

Output:
```python
import json
import pandas as pd
from openai import OpenAI  # or any LLM client

SCHEMA_CONTEXT = """
Columns: date (date), region (string), product (string), revenue (float), quantity (int)
Sample rows:
| date       | region | product | revenue | quantity |
|------------|--------|---------|---------|----------|
| 2024-01-15 | West   | Widget  | 1200.50 | 45       |
| 2024-02-03 | East   | Gadget  | 890.00  | 32       |
"""

FEW_SHOT = [
    {
        "query": "Show monthly revenue trend",
        "spec": {
            "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
            "mark": "line",
            "encoding": {
                "x": {"field": "date", "type": "temporal", "timeUnit": "yearmonth"},
                "y": {"field": "revenue", "type": "quantitative", "aggregate": "sum"}
            }
        }
    }
]

def nl2vegalite(query: str, df: pd.DataFrame) -> dict:
    prompt = f"""You are a Vega-Lite specification generator.
Data schema:
{SCHEMA_CONTEXT}

Examples:
{json.dumps(FEW_SHOT, indent=2)}

User query: {query}

Respond with ONLY a valid Vega-Lite JSON spec. Reason step-by-step first,
then output the spec inside a JSON block."""

    client = OpenAI()
    resp = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"}
    )
    spec = json.loads(resp.choices[0].message.content)

    # Validate: check all referenced fields exist in DataFrame
    _validate_spec_fields(spec, df.columns.tolist())
    spec["data"] = {"values": df.to_dict(orient="records")}
    return spec

def _validate_spec_fields(spec: dict, valid_columns: list[str]):
    """Walk the spec and verify every 'field' value exists in the data."""
    if isinstance(spec, dict):
        if "field" in spec and spec["field"] not in valid_columns:
            raise ValueError(f"Unknown column: {spec['field']}. Valid: {valid_columns}")
        for v in spec.values():
            _validate_spec_fields(v, valid_columns)
    elif isinstance(spec, list):
        for item in spec:
            _validate_spec_fields(item, valid_columns)
```

**Example 2: Conversational analytics with multi-turn memory**

User: "Add a chat interface to my Streamlit dashboard where users can ask questions and get charts, then refine them."

Approach:
1. Maintain a conversation history list storing (query, spec, data_summary) tuples
2. On each turn, inject prior context so the LLM can resolve references ("make it a pie chart")
3. Detect modification intents vs. new-query intents

Output:
```python
import streamlit as st

if "history" not in st.session_state:
    st.session_state.history = []

query = st.chat_input("Ask about your data...")
if query:
    # Build context from history
    history_context = "\n".join(
        f"Turn {i+1}: User asked '{h['query']}', generated {h['chart_type']} of {h['fields']}"
        for i, h in enumerate(st.session_state.history[-5:])  # last 5 turns
    )

    prompt = f"""Prior conversation:
{history_context}

Current query: {query}
Schema: {schema_context}

If the user is refining a previous chart, modify that spec.
If it's a new question, generate a fresh spec.
Output valid Vega-Lite JSON."""

    spec = call_llm(prompt)  # your LLM wrapper
    spec = validate_and_attach_data(spec, df)

    st.vega_lite_chart(df, spec, use_container_width=True)
    st.session_state.history.append({
        "query": query,
        "chart_type": spec.get("mark", "unknown"),
        "fields": extract_fields(spec),
        "spec": spec
    })
```

**Example 3: Chart explanation from a rendered visualization**

User: "Given a Vega-Lite spec and the underlying data, generate a plain-English explanation of what the chart shows."

Approach:
1. Extract the spec's mark type, encodings, and aggregations to describe chart structure
2. Run the aggregation on real data to get actual values
3. Prompt the LLM with structure + real values to produce a grounded explanation

Output:
```python
def explain_chart(spec: dict, df: pd.DataFrame) -> str:
    # Extract chart metadata
    mark = spec.get("mark", "unknown")
    encodings = spec.get("encoding", {})
    x_field = encodings.get("x", {}).get("field", "")
    y_field = encodings.get("y", {}).get("field", "")
    y_agg = encodings.get("y", {}).get("aggregate", "raw values")

    # Compute actual summary statistics from the data
    if y_agg == "sum":
        summary = df.groupby(x_field)[y_field].sum().to_dict()
    elif y_agg == "mean":
        summary = df.groupby(x_field)[y_field].mean().round(2).to_dict()
    else:
        summary = df[y_field].describe().to_dict()

    prompt = f"""Describe this chart for a non-technical audience.

Chart type: {mark}
X-axis: {x_field}
Y-axis: {y_agg} of {y_field}
Actual data summary: {json.dumps(summary)}

Rules:
- Only reference numbers that appear in the data summary above
- Mention the highest and lowest values
- Note any clear trends or outliers
- Keep it to 2-3 sentences"""

    return call_llm(prompt)
```

## Best Practices

**Do:**
- Always inject the real data schema (column names, types, sample rows) into every prompt — this is the single most impactful accuracy improvement identified across the surveyed systems
- Validate generated specs against the schema before rendering; treat LLM output as untrusted input
- Use few-shot exemplars (3-5) that match the complexity and domain of the user's query
- Decompose complex analytical questions into sub-steps (filter → aggregate → visualize) using chain-of-thought or agent loops
- Store conversation history for multi-turn interactions and resolve anaphoric references ("that chart", "same data") against prior turns
- Ground explanations in actual computed data values — never let the LLM narrate from the spec alone

**Avoid:**
- Letting the LLM choose chart types without schema awareness — it will pick scatterplots for categorical data and pie charts for 50-category dimensions
- Generating SQL or data transformations without executing and validating them before visualization
- Using a single monolithic prompt for the entire pipeline (query → transform → visualize → explain) — modular stages with validation between them are more reliable
- Trusting LLM-generated numerical claims in chart explanations without verifying against the source data
- Skipping the rendering validation step — an LLM can produce syntactically valid but semantically wrong specs (e.g., encoding a date as nominal)

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Invalid column reference in spec | Schema validation catches unknown field names | Re-prompt with error message + valid column list |
| Wrong data type mapping (e.g., temporal encoded as nominal) | Type-check encoding channels against DataFrame dtypes | Override encoding type programmatically or re-prompt |
| Generated SQL returns empty results | Check result set length after execution | Inform user no data matches; suggest loosening filters |
| Hallucinated numbers in explanation | Compare stated values against actual query results | Regenerate explanation with stricter grounding prompt |
| Ambiguous user query ("show me the data") | No clear visualization task detected | Ask a clarifying question: "What would you like to compare or explore?" |
| Spec produces a valid but unhelpful chart (e.g., 1000 bars) | Heuristic checks: cardinality > 30 for categorical axis | Suggest aggregation or filtering; offer top-N alternative |
| Context window overflow in multi-turn conversations | Token count exceeds limit | Summarize older turns; keep only last 3-5 full specs in context |

## Limitations

- **Spatial reasoning**: LLMs are poor at understanding geometric relationships within rendered charts. They cannot reliably answer questions like "is the blue bar taller than the red one in the third group?" from a chart image alone. Use computed data for such comparisons instead of vision models.
- **Complex joins and aggregations**: NL2SQL accuracy degrades sharply on queries requiring multi-table joins, window functions, or nested subqueries. For complex data operations, provide worked examples via RAG or decompose into simpler sub-queries.
- **Large datasets**: Injecting full datasets into prompts is impractical. Systems must use schema + sample rows + computed summaries rather than raw data.
- **Evaluation gap**: There is no widely accepted benchmark for end-to-end LLM-visualization systems. You will need to build task-specific evaluation suites combining spec correctness, explanation faithfulness, and user task completion metrics.
- **Accessibility**: Most current systems assume sighted users interacting via text. Voice-first and screen-reader-compatible interfaces require additional design work not well covered by existing implementations.
- **Determinism**: The same query may produce different specs across runs. For production systems, use temperature=0, seed parameters, and cached responses for reproducibility.

## Reference

Brossier, M., Isenberg, T., Schonborn, K., Unger, J., & Romero, M. (2026). *State of the Art of LLM-Enabled Interaction with Visualization.* arXiv:2601.14943v2. [https://arxiv.org/abs/2601.14943v2](https://arxiv.org/abs/2601.14943v2)

Look for: The six-dimension classification framework (Section 3), the visualization pipeline role mapping (Section 4), design pattern taxonomy for NL2VIS/NL2SQL/conversational systems (Section 5), and the evaluation methodology discussion (Section 6) identifying gaps in current benchmarking approaches.