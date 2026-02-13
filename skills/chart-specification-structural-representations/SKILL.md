---
name: "chart-specification-structural-representations"
description: |
  Generate high-fidelity plotting code from chart images or descriptions using structured intermediate
  specifications. Decomposes charts into semantic topology (type, coordinates, domains, series) and
  runtime numerical facts before producing code, preventing hallucinated data and structural errors.
  Trigger phrases:
  - "convert this chart image to code"
  - "recreate this plot in matplotlib"
  - "generate plotting code from this chart"
  - "reproduce this visualization programmatically"
  - "write code that matches this chart exactly"
  - "chart to code"
---

# Chart Specification: Structural Representations for Chart-to-Code Generation

This skill enables Claude to generate structurally faithful plotting code from chart images or
descriptions by first constructing a **Chart Specification** -- a structured JSON intermediate
representation that decomposes a chart into its semantic topology and numerical primitives. Instead
of attempting to produce code directly (which leads to hallucinated values and wrong chart types),
Claude first reasons about the chart's structure declaratively, then generates code grounded in that
verified specification. This approach, from He et al. (2026), achieves up to 61.7% improvement over
direct generation on complex chart benchmarks.

## When To Use

- When the user provides a chart image and asks Claude to generate matplotlib, plotly, or seaborn code that reproduces it
- When the user describes a complex chart verbally and needs structurally correct plotting code
- When generated chart code has the wrong structure (e.g., bar chart instead of grouped bar, missing subplots, wrong axis ranges)
- When the user needs to reverse-engineer the data and layout from an existing visualization
- When reproducing multi-panel or composite charts where layout fidelity matters
- When the user asks to convert between plotting libraries while preserving chart semantics

## Key Technique

**The core problem:** Direct chart-to-code generation encourages surface-level token imitation. A VLM
sees a chart image and tries to write code that "looks right" token by token, but frequently
hallucinates data values, misidentifies chart types, drops series, or produces wrong axis ranges.
The code may execute without errors yet render a chart that is structurally wrong.

**The solution -- Chart Specification:** Before writing any code, decompose the chart into a structured
representation `S = <S_sem, S_code>`. The semantic component `S_sem` captures declarative intent:
chart type/family, panel layout, coordinate system (cartesian, polar, 3D), axis domains (ranges and
categories), series/legend entries, and analytic representations (functional forms like curve
equations). The code-level component `S_code` captures runtime numerical facts that are only
available through execution: computed wedge proportions in pie charts, statistical quartiles in box
plots, node-edge adjacency in graph visualizations, and interpolated curve sample points.

**Why it works:** By forcing explicit reasoning about structure before code generation, errors become
detectable at the specification level. A wrong chart type is caught before any code is written. A
missing series is flagged in the specification. Axis range mismatches show up as domain IoU failures.
This "specify first, code second" pipeline converts a fuzzy generation problem into a structured
verification problem, and is the key insight practitioners can apply even without the paper's
full RL training pipeline.

## Step-by-Step Workflow

1. **Analyze the chart input.** Examine the chart image or description. Identify what is being
   visualized at the highest level: is it a comparison, distribution, composition, relationship,
   or temporal trend?

2. **Extract global topology.** Determine the chart family (bar, line, scatter, pie, box, heatmap,
   etc.), the number of panels/subplots, and the layout grid (e.g., 1x2, 2x2). Record whether
   panels share axes or are independent.

3. **Identify coordinate systems.** For each panel, classify the coordinate space: Cartesian (x-y),
   polar (theta-r), 3D (x-y-z), or geographic (lat-lon). Note any axis inversions or log scales.

4. **Extract data domains.** For each axis, record the range (min, max) for numerical axes or the
   complete list of categories for categorical axes. For shared axes across panels, note the
   unified domain.

5. **Enumerate data series and visual encodings.** List every distinct data series by its legend
   label. For each series, record: the visual encoding (line, bar, marker, area), color, and any
   distinguishing style (dashed, hatched, marker shape). Compute the Jaccard overlap with the
   legend to verify completeness.

6. **Extract numerical data or functional forms.** For data-driven charts, extract the actual values
   (or best estimates from the image). For function-driven charts (curves, density plots), identify
   the analytic expression (e.g., `y = np.exp(-(x-mu)**2 / (2*sigma**2))`). For statistical
   charts, extract the summary statistics (median, quartiles, whisker extents).

7. **Assemble the Chart Specification as structured JSON.** Organize all extracted information into
   the specification format (see template below). This is the verification checkpoint -- review the
   spec for internal consistency before proceeding.

8. **Generate plotting code grounded in the specification.** Write matplotlib/plotly/seaborn code
   that directly implements each field of the specification. Every axis range, series, color, and
   layout parameter must trace back to a spec field. Do not add visual elements not in the spec.

9. **Validate structural alignment.** Cross-check the generated code against the specification:
   Does the code create the right number of subplots? Are all series present? Do axis limits match
   the domain ranges? Are coordinate systems correct?

10. **Present the specification and code together.** Show the user both the structured spec (for
    verification) and the executable code. This lets them catch structural errors at the spec level
    rather than debugging rendered output.

## Chart Specification Template

```json
{
  "topology": {
    "chart_family": "grouped_bar",
    "num_panels": 1,
    "layout": [1, 1]
  },
  "panels": [
    {
      "coordinate_system": "cartesian",
      "x_axis": {
        "type": "categorical",
        "categories": ["Q1", "Q2", "Q3", "Q4"],
        "label": "Quarter"
      },
      "y_axis": {
        "type": "numerical",
        "domain": [0, 120],
        "label": "Revenue ($M)"
      },
      "series": [
        {
          "label": "Product A",
          "encoding": "bar",
          "color": "#4C78A8",
          "values": [45, 62, 78, 95]
        },
        {
          "label": "Product B",
          "encoding": "bar",
          "color": "#F58518",
          "values": [38, 55, 70, 110]
        }
      ]
    }
  ],
  "annotations": [],
  "title": "Quarterly Revenue by Product"
}
```

## Concrete Examples

**Example 1: Grouped bar chart from image**

User: "Here's a screenshot of a bar chart comparing test scores across three subjects for two classes. Recreate it in matplotlib."

Approach:
1. Examine the image. Identify: grouped bar chart, 1 panel, cartesian coordinates.
2. Extract categories on x-axis: ["Math", "Science", "English"].
3. Extract y-axis range: 0 to 100, label "Average Score".
4. Enumerate series: "Class A" (blue bars), "Class B" (orange bars).
5. Read approximate values from bar heights: Class A = [82, 75, 90], Class B = [78, 88, 85].
6. Assemble spec, then generate code.

Output specification:
```json
{
  "topology": { "chart_family": "grouped_bar", "num_panels": 1, "layout": [1, 1] },
  "panels": [{
    "coordinate_system": "cartesian",
    "x_axis": { "type": "categorical", "categories": ["Math", "Science", "English"], "label": "Subject" },
    "y_axis": { "type": "numerical", "domain": [0, 100], "label": "Average Score" },
    "series": [
      { "label": "Class A", "encoding": "bar", "color": "#1f77b4", "values": [82, 75, 90] },
      { "label": "Class B", "encoding": "bar", "color": "#ff7f0e", "values": [78, 88, 85] }
    ]
  }],
  "title": "Test Scores by Subject"
}
```

Output code:
```python
import matplotlib.pyplot as plt
import numpy as np

categories = ["Math", "Science", "English"]
class_a = [82, 75, 90]
class_b = [78, 88, 85]

x = np.arange(len(categories))
width = 0.35

fig, ax = plt.subplots(figsize=(8, 5))
ax.bar(x - width/2, class_a, width, label="Class A", color="#1f77b4")
ax.bar(x + width/2, class_b, width, label="Class B", color="#ff7f0e")

ax.set_xlabel("Subject")
ax.set_ylabel("Average Score")
ax.set_title("Test Scores by Subject")
ax.set_xticks(x)
ax.set_xticklabels(categories)
ax.set_ylim(0, 100)
ax.legend()
plt.tight_layout()
plt.savefig("chart.png", dpi=150)
plt.show()
```

**Example 2: Multi-panel chart with mixed types**

User: "Generate code for a 1x2 subplot layout. Left panel: line chart of temperature over 12 months with min/max shaded range. Right panel: pie chart of energy source breakdown."

Approach:
1. Topology: 2 panels, layout [1, 2]. Left = cartesian line+fill, Right = polar pie.
2. Left panel: x = months (categorical), y = temperature (numerical, domain [-5, 35]).
   Two series concept: mean line + shaded min-max band.
3. Right panel: polar coordinate, categories = ["Solar", "Wind", "Gas", "Nuclear"],
   values as proportions summing to 1.0.
4. Assemble spec with both panels, generate code with `plt.subplots(1, 2)`.

Output specification:
```json
{
  "topology": { "chart_family": "composite", "num_panels": 2, "layout": [1, 2] },
  "panels": [
    {
      "coordinate_system": "cartesian",
      "x_axis": { "type": "categorical", "categories": ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"] },
      "y_axis": { "type": "numerical", "domain": [-5, 35], "label": "Temperature (C)" },
      "series": [
        { "label": "Mean Temp", "encoding": "line", "color": "#e45756", "values": [2,3,8,14,19,24,27,26,21,14,8,3] },
        { "label": "Min-Max Range", "encoding": "fill_between", "color": "#e4575640",
          "min_values": [-3,-1,3,8,13,18,21,20,15,8,3,-1],
          "max_values": [7,8,13,20,25,30,33,32,27,20,13,7] }
      ]
    },
    {
      "coordinate_system": "polar",
      "series": [
        { "label": "Solar", "value": 0.30, "color": "#f58518" },
        { "label": "Wind", "value": 0.25, "color": "#4c78a8" },
        { "label": "Gas", "value": 0.28, "color": "#72b7b2" },
        { "label": "Nuclear", "value": 0.17, "color": "#b279a2" }
      ]
    }
  ],
  "title": "Climate and Energy Overview"
}
```

Output code:
```python
import matplotlib.pyplot as plt
import numpy as np

months = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"]
mean_temp = [2,3,8,14,19,24,27,26,21,14,8,3]
min_temp = [-3,-1,3,8,13,18,21,20,15,8,3,-1]
max_temp = [7,8,13,20,25,30,33,32,27,20,13,7]

energy_labels = ["Solar", "Wind", "Gas", "Nuclear"]
energy_values = [0.30, 0.25, 0.28, 0.17]
energy_colors = ["#f58518", "#4c78a8", "#72b7b2", "#b279a2"]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 5))

# Left panel: temperature line with shaded range
x = np.arange(len(months))
ax1.plot(x, mean_temp, color="#e45756", linewidth=2, label="Mean Temp")
ax1.fill_between(x, min_temp, max_temp, color="#e45756", alpha=0.25, label="Min-Max Range")
ax1.set_xticks(x)
ax1.set_xticklabels(months, rotation=45)
ax1.set_ylabel("Temperature (C)")
ax1.set_ylim(-5, 35)
ax1.legend()

# Right panel: pie chart
ax2.pie(energy_values, labels=energy_labels, colors=energy_colors, autopct="%1.0f%%", startangle=90)
ax2.set_aspect("equal")

fig.suptitle("Climate and Energy Overview")
plt.tight_layout()
plt.savefig("chart.png", dpi=150)
plt.show()
```

**Example 3: Correcting structurally wrong generated code**

User: "I asked another tool to generate a box plot from this image, but it produced a bar chart instead. Can you fix it?"

Approach:
1. Read the existing code to identify the structural mismatch.
2. Analyze the chart image. Build spec: topology = box plot, not bar chart.
3. Extract statistical semantics: categories on x-axis, distribution statistics per group.
4. Rewrite code grounded in the corrected specification, using `ax.boxplot()` or `ax.bxp()`.
5. Cross-check: the spec says "box" encoding, the code uses `boxplot` -- aligned.

The specification catches the error at the topology level ("chart_family": "box" vs the incorrect "bar"),
making the fix systematic rather than guesswork.

## Best Practices

- **Do:** Always produce the specification before writing code. Even for simple charts, the 30-second
  spec check catches errors that would take minutes to debug from rendered output.
- **Do:** Use the specification as a communication artifact. Show it to the user for validation before
  generating code -- they can correct "that axis should go to 200, not 100" at the spec level.
- **Do:** Match every code element to a spec field. If a line of code adds something not in the spec
  (a grid, a color, an annotation), either add it to the spec first or remove the code.
- **Do:** Extract domain ranges explicitly. The most common chart reproduction error is wrong axis
  limits; the spec forces you to commit to concrete numbers before coding.
- **Avoid:** Skipping the spec for "simple" charts. A single pie chart still benefits from listing all
  wedge labels and proportions before writing `plt.pie()`.
- **Avoid:** Guessing data values from chart images. If values cannot be read with confidence,
  explicitly mark them as approximate in the spec and tell the user.
- **Avoid:** Over-specifying style details (font sizes, tick rotation) at the expense of structural
  correctness. Get the structure right first; polish styling after.

## Error Handling

| Problem | Detection | Resolution |
|---------|-----------|------------|
| Wrong chart type identified | Topology field contradicts visual evidence | Re-examine image; check for dual-axis or composite types that look like simpler charts |
| Missing data series | Series count in spec < legend entries in image | Enumerate legend items explicitly; compare spec series labels to image legend |
| Axis range mismatch | Domain IoU between spec and image is low | Read axis tick labels carefully; account for padding matplotlib adds beyond data range |
| Code executes but renders wrong | Spec-code cross-check reveals divergence | Walk through spec fields one by one, verify each has a corresponding code statement |
| Cannot read exact values from image | Data values marked as approximate | State the uncertainty; offer to generate code with placeholder data the user can fill in |
| Composite/unusual chart type | No matching chart_family in standard taxonomy | Decompose into constituent primitives (e.g., a lollipop chart = scatter + vlines) |

## Limitations

- **Image resolution dependency.** When working from chart images, data extraction accuracy is limited
  by image clarity. Small or low-resolution charts may yield approximate values only.
- **Highly custom visualizations.** Charts with bespoke rendering (custom SVG overlays, non-standard
  coordinate projections, 3D surface meshes) may not decompose cleanly into the spec template.
- **Interactive chart features.** The specification captures static structure. Tooltips, zoom
  interactions, and dynamic filtering from plotly/bokeh are outside the spec scope.
- **Exact color matching.** Colors extracted from images are approximations. If exact brand colors
  matter, the user should provide hex codes rather than relying on image extraction.
- **Library-specific idioms.** The spec is library-agnostic, but translation to code requires knowing
  the target library's API. Some spec fields (like `fill_between`) map differently across matplotlib,
  plotly, and seaborn.

## Reference

He, M., Dai, M., Zhang, J., Liu, Y., & Tao, S. (2026). *Chart Specification: Structural
Representations for Incentivizing VLM Reasoning in Chart-to-Code Generation.*
[arXiv:2602.10880](https://arxiv.org/abs/2602.10880v1)

Key takeaway: structured intermediate representations with hierarchical verification (topology
before semantics before numerical details) dramatically improve chart-to-code fidelity, even
with minimal training data.