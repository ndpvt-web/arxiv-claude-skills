---
name: "multivis-agent-multi-agent-framework-logic"
description: "Build reliable multi-agent data visualization pipelines with logic rule constraints. Use when: 'generate a chart from this database', 'create a visualization matching this reference image', 'refine this chart based on feedback', 'build a multi-agent visualization system', 'convert this matplotlib code to altair', 'visualize SQL query results with style references'."
---

# MultiVis-Agent: Logic-Rule-Constrained Multi-Agent Visualization Framework

This skill enables Claude to build reliable multi-agent systems for data visualization that accept cross-modal inputs (natural language, reference images, code examples, databases) and produce correct, executable chart code. The core technique from the SIGMOD 2026 paper is a four-layer logic rule framework that wraps LLM-driven agents with mathematical constraints — guaranteeing termination, parameter safety, and error recovery without replacing LLM flexibility. Apply this when building any agent pipeline where reliability matters as much as capability.

## When to Use

- When the user asks to generate a visualization from a database using natural language (text-to-vis)
- When the user provides a reference image and asks "make a chart that looks like this"
- When the user has matplotlib code and wants it adapted to altair (or vice versa) for a new dataset
- When the user wants to iteratively refine an existing chart ("move the legend to the top", "change colors")
- When building any multi-agent pipeline that calls tools/APIs and needs guaranteed termination and error recovery
- When the user needs a visualization system that handles SQL generation, code generation, and validation as separate concerns
- When designing agent orchestration with formal reliability properties (no infinite loops, bounded retries)

## Key Technique

**Four-Layer Logic Rules as Agent Guardrails.** MultiVis-Agent separates what LLMs are good at (reasoning about intent, generating code) from what they are bad at (consistent decision routing, parameter validation, knowing when to stop). Four layers of deterministic logic rules wrap around LLM reasoning:

1. **Coordination Rules (CR):** Deterministically classify each task into a type (basic generation, image-referenced, code-referenced, or iterative refinement) using priority ordering — no LLM ambiguity in routing. Validate tool prerequisites before execution. Map evaluation results to next actions deterministically.
2. **Tool Execution Rules (TE):** Clip parameters to valid ranges via a constraint function. Standardize code execution environments (namespace normalization, save-operation injection). Process reference materials (images, code) through type-specific handlers.
3. **Error Handling Rules (EH):** Classify parsing errors, define recovery strategies for code execution failures, and validate image operations — all deterministically, so the LLM never has to decide *how* to recover.
4. **ReAct Control Rules (RC):** Hard cap at 10 iterations per agent loop. Validate response format compliance. Select optimal model per content type.

**Why this works better than pure LLM agents:** Without these rules, the paper measured 65.10% code execution success and 74.48% task completion. With rules: 94.56% and 99.58%. The rules don't replace LLM reasoning — they constrain it at decision boundaries where determinism beats probabilistic generation.

**Three-Agent Architecture.** A Coordinator routes work to: (1) a Database & Query Agent that explores schemas and generates SQL, (2) a Visualization Implementation Agent that produces executable chart code, and (3) a Validation & Evaluation Agent that runs the code and scores output quality. Each agent operates in thought-action-observation loops internally, but the Coordinator's routing between them is deterministic.

## Step-by-Step Workflow

1. **Classify the task deterministically.** Inspect inputs to assign exactly one of four types: (A) text + database only, (B) text + database + reference image, (C) text + database + reference code, (D) existing code + modification instructions. Use priority ordering — if modification instructions exist, it's type D regardless of other inputs. Do not let the LLM decide the type.

2. **Validate prerequisites before any tool call.** Before invoking SQL execution, confirm the database connection is live and the target tables exist. Before running code, confirm required libraries are importable. Before processing an image reference, confirm the file is accessible and in a supported format. Gate every tool behind a predicate check.

3. **Run the Database & Query Agent.** Have it explore the schema (`list_tables`, `get_foreign_keys`, `find_fields`), generate SQL, execute it, and return a structured result set. If the SQL fails, the agent retries with error context — but is capped at the iteration limit.

4. **Process cross-modal references.** For reference images: use a vision-language model to extract visual properties (chart type, color palette, layout, axis style) and convert them to code-level specifications. For reference code: parse the code to extract reusable structural patterns (binning logic, aggregation style, legend placement) independent of the specific library.

5. **Run the Visualization Implementation Agent.** Feed it the data from step 3, the extracted specifications from step 4, and the user's natural language intent. It generates executable chart code (preferably Altair for declarative clarity, or matplotlib). Apply TE-Rule 2: normalize namespaces, inject save operations, and standardize the execution environment before running.

6. **Clip all parameters to valid ranges.** Before executing generated code, apply a constraint function: numerical parameters get clipped to `[min, max]` bounds (e.g., figure width between 1-50 inches, font size between 4-72pt). Invalid categorical values get mapped to defaults. This catches LLM hallucinated parameters silently.

7. **Execute and validate.** Run the generated code in a sandboxed environment. The Validation Agent checks: (a) does the code execute without errors, (b) does it produce a non-empty image, (c) does the output match the user's intent. Return structured feedback, not just pass/fail.

8. **Handle errors deterministically.** If code execution fails: classify the error (syntax, import, data type, runtime), apply the matching recovery strategy (fix syntax → re-parse, missing import → add it, type error → cast data), and retry. Do not ask the LLM "what went wrong?" — pattern-match the error and apply the fix.

9. **Enforce termination.** Track iteration count across the entire coordination loop. At iteration 10, return the best result so far with an explanation of remaining issues. Never allow an open-ended retry loop. Each agent sub-loop also has its own iteration cap.

10. **For iterative refinement (Type D): preserve and patch.** Do not regenerate the full visualization. Parse the modification instruction, identify which code sections to change, apply targeted edits, and re-validate. Maintain a diff so the user can see exactly what changed.

## Concrete Examples

**Example 1: Text-to-Visualization from Database**

```
User: "Show me the average salary by department from our company database, as a horizontal bar chart"

Approach:
1. Classify as Type A (text + database, no references).
2. Database Agent explores schema → finds `employees` table with `department` and
   `salary` columns.
3. Database Agent generates and executes:
   SELECT department, AVG(salary) as avg_salary FROM employees GROUP BY department
4. Visualization Agent generates Altair code:

   import altair as alt
   import pandas as pd

   data = pd.DataFrame({
       'department': ['Engineering', 'Sales', 'Marketing', 'HR'],
       'avg_salary': [125000, 95000, 88000, 78000]
   })

   chart = alt.Chart(data).mark_bar().encode(
       x=alt.X('avg_salary:Q', title='Average Salary ($)'),
       y=alt.Y('department:N', sort='-x', title='Department')
   ).properties(width=500, height=300)
   chart.save('output.html')

5. Validation Agent executes code, confirms HTML output is non-empty and
   contains a horizontal bar chart. Done.
```

**Example 2: Image-Referenced Chart Generation**

```
User: "Create a chart showing monthly revenue trends, styled like this" [attaches
a reference image of a minimalist line chart with a light gray background, blue
line, and circular markers]

Approach:
1. Classify as Type B (text + database + reference image).
2. Vision model analyzes the reference image and extracts:
   - Chart type: line chart
   - Background: light gray (#f5f5f5)
   - Line color: steel blue (#4682b4)
   - Markers: filled circles, ~6px
   - Grid: subtle horizontal only
   - Font: sans-serif, minimal axis labels
3. Database Agent queries: SELECT month, revenue FROM monthly_metrics ORDER BY month
4. Visualization Agent receives data + extracted style spec, generates:

   chart = alt.Chart(data).mark_line(
       color='#4682b4', point=alt.OverlayMarkDef(filled=True, size=60)
   ).encode(
       x=alt.X('month:T', title='Month'),
       y=alt.Y('revenue:Q', title='Revenue ($)')
   ).configure_view(
       fill='#f5f5f5', strokeWidth=0
   ).configure_axisY(
       grid=True, gridColor='#e0e0e0', gridOpacity=0.5
   ).configure_axisX(grid=False)

5. Validation confirms visual similarity to reference. Done.
```

**Example 3: Iterative Refinement**

```
User: "Take this existing bar chart code and move the legend to the bottom,
increase font size to 14, and change the color scheme to 'tableau10'"

Approach:
1. Classify as Type D (existing code + modification instructions).
2. Parse three discrete modifications from the instruction.
3. Visualization Agent patches the existing code:
   - Adds .configure_legend(orient='bottom')
   - Adds .configure_axis(labelFontSize=14, titleFontSize=14)
   - Changes color encoding to scale=alt.Scale(scheme='tableau10')
4. Parameter clipping: font size 14 is within [4, 72] → valid. Legend orient
   'bottom' is in {'top','bottom','left','right'} → valid.
5. Execute, validate output, return patched code with diff showing exactly
   the 3 changes made.
```

**Example 4: Code-Referenced Cross-Library Adaptation**

```
User: "I have this matplotlib scatter plot code. Recreate the same visualization
in Altair using our product database."

Approach:
1. Classify as Type C (text + database + reference code).
2. Parse the matplotlib code to extract structural patterns:
   - Chart type: scatter, x=price, y=rating
   - Color mapped to category column
   - Size mapped to sales_volume
   - Log scale on x-axis
   - Custom tooltip annotations
3. Database Agent queries the product database for the required columns.
4. Visualization Agent generates equivalent Altair code preserving all
   extracted patterns but using Altair's declarative API.
5. Validate that both charts express the same data mappings.
```

## Best Practices

- **Do:** Make task classification deterministic with priority ordering. If inputs include modification instructions, it's always Type D — don't let the LLM second-guess. The classification function should be a simple if/elif chain, not a prompt.
- **Do:** Clip numerical parameters before code execution. LLMs sometimes hallucinate absurd values (figure width = 10000px). A simple `min(max(val, lower), upper)` prevents crashes silently.
- **Do:** Separate the concerns — one agent for data retrieval, one for code generation, one for validation. Agents that both generate and judge their own output are unreliable.
- **Do:** Inject save/export operations into generated code automatically (TE-Rule 2). LLMs frequently generate chart code that displays but never saves, breaking headless pipelines.
- **Avoid:** Letting the LLM decide error recovery strategy. Pattern-match the error type and apply a deterministic fix. LLMs asked "how should I fix this error?" often change unrelated code.
- **Avoid:** Open-ended retry loops. Always enforce a hard iteration cap (10 is the paper's recommendation). Return the best partial result rather than looping forever.

## Error Handling

| Error Type | Detection | Recovery |
|---|---|---|
| SQL syntax error | Database execution returns parse error | Re-prompt Database Agent with error message + schema context |
| Missing import | `ModuleNotFoundError` in code execution | Deterministically add the import based on the module name |
| Data type mismatch | `TypeError` or encoding error in chart spec | Cast the column to the expected type (e.g., `pd.to_datetime`) |
| Empty query result | Result set has 0 rows | Report to user — do not generate a chart from empty data |
| Invalid chart spec | Altair/matplotlib `ValueError` | Clip invalid parameters to defaults, retry once |
| Reference image unreadable | File format or access error (EH-Rule 3) | Fall back to Type A (text-only) and inform user |
| Iteration limit reached | Counter hits 10 | Return best result so far with structured explanation of remaining issues |

## Limitations

- **Vision model dependency for Type B tasks.** Extracting style from reference images requires a capable VLM. Without one, fall back to Type A and ask the user to describe the desired style in text.
- **Library-specific code generation.** The framework targets Altair and matplotlib. Generating Plotly, D3.js, or other library code requires extending the tool execution rules and code examples.
- **Database-centric.** The pipeline assumes data comes from a SQL database. For CSV/JSON/API data sources, the Database Agent step needs adaptation (or bypass).
- **Single-chart scope.** The framework handles one visualization per task. Dashboards with multiple coordinated views require a higher-level orchestration layer.
- **Logic rules require upfront engineering.** The four-layer rule system must be designed per domain. The rules from the paper are specific to visualization — applying this pattern to other agent tasks (e.g., code review, deployment) requires defining new rule sets for those domains.

## Reference

**Paper:** [MultiVis-Agent: A Multi-Agent Framework with Logic Rules for Reliable and Comprehensive Cross-Modal Data Visualization](https://arxiv.org/abs/2601.18320v1) (SIGMOD 2026)
**Code:** [github.com/Jinwei-Lu/MultiVis](https://github.com/Jinwei-Lu/MultiVis)
**Key insight to extract:** The four-layer logic rule framework (Section 4) and the formal reliability theorems (Section 4.5) — these show how to wrap LLM agents with deterministic constraints at decision boundaries to achieve 94.56% code execution success vs. 65.10% without rules.