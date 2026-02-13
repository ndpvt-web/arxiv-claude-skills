---
name: "paperbanana-automating-academic-illustration"
description: "Generate publication-ready academic illustrations using a multi-agent pipeline inspired by PaperBanana. Orchestrates retrieval, planning, styling, rendering, and self-critique agents to produce methodology diagrams and statistical plots. Use when: 'create a figure for my paper', 'generate a methodology diagram', 'make a publication-ready illustration', 'draw an architecture diagram for this system', 'create a statistical plot for these results', 'illustrate this pipeline'."
---

# PaperBanana: Automated Academic Illustration

This skill enables Claude to generate publication-ready academic illustrations — methodology diagrams, architecture figures, and statistical plots — by applying the five-agent orchestration pipeline from the PaperBanana framework (Zhu et al., 2026). Instead of producing a single monolithic prompt-to-image output, the approach decomposes illustration generation into retrieval, content planning, style planning, rendering, and iterative self-critique, yielding figures that score higher on faithfulness, conciseness, readability, and aesthetics than single-pass methods.

## When to Use

- When the user asks to create a methodology or architecture diagram for a research paper or technical document.
- When the user needs a publication-quality figure illustrating a system pipeline, model architecture, or data flow.
- When the user wants to generate statistical plots (bar charts, line plots, heatmaps) from experimental results with academic styling.
- When the user asks to "draw", "illustrate", or "diagram" a technical concept described in text.
- When the user provides a paper section or abstract and wants a corresponding figure.
- When the user has a rough sketch or description and wants it turned into a polished academic illustration.

## Key Technique

PaperBanana's core insight is that academic illustration is not a single generation task but a **multi-agent pipeline** with five distinct roles: Retriever, Planner, Stylist, Visualizer, and Critic. Each agent has a narrow responsibility, and information flows sequentially with a feedback loop at the end. This decomposition prevents the common failure mode where a single model tries to simultaneously understand content, choose layout, pick colors, and render — producing figures that are aesthetically passable but factually wrong (hallucinated connections, missing components, wrong data mappings).

The **dual rendering strategy** is particularly important: for statistical plots (bar charts, scatter plots, line graphs), **code generation** (Python/matplotlib/seaborn) preserves content fidelity perfectly since data maps directly to axes. For methodology diagrams and architecture figures, **programmatic SVG or TikZ generation** provides precise control over boxes, arrows, and labels while avoiding the hallucination-prone nature of diffusion-based image generation. The paper found that "image generation excels in presentation but underperforms in content fidelity" — so for code-assistable contexts, always prefer code-based rendering.

The **Critic agent** is what elevates output quality above single-pass methods. It evaluates the rendered illustration against the original specification on four axes — faithfulness (are all described components present and correctly connected?), conciseness (is there visual clutter?), readability (can labels and flow be parsed at a glance?), and aesthetics (does it meet publication standards?) — then feeds structured feedback back to the Visualizer for refinement. This loop runs 2-3 iterations, with diminishing returns beyond that.

## Step-by-Step Workflow

1. **Extract the illustration specification.** Parse the user's request to identify: (a) the type of figure (methodology diagram, architecture, statistical plot, flowchart), (b) the components/entities to depict, (c) the relationships or data flow between them, and (d) any stated style preferences (color scheme, font, target venue).

2. **Retrieve reference context (Retriever role).** If the user provides a paper section or related figures, analyze them to understand domain conventions. For ML papers, this means left-to-right data flow, color-coded module groups, and standard iconography (database cylinders, neural network blocks). If no references are provided, apply sensible defaults for the stated domain (NLP, CV, systems, etc.).

3. **Plan content layout (Planner role).** Produce a structured textual specification listing every visual element: boxes/nodes with their labels, directed edges with labels, grouping boundaries, and a top-level layout direction (left-to-right, top-to-bottom). Write this as a structured intermediate representation — a JSON or YAML description of the figure's content graph.

4. **Plan visual style (Stylist role).** Select a color palette (prefer accessible palettes like ColorBrewer for academic work), font choices (sans-serif for diagrams, matching the paper's body font if known), line weights, arrow styles, and spacing. Encode these as style parameters that the renderer will consume. For statistical plots, select appropriate chart types and axis formatting.

5. **Choose the rendering approach.** For statistical plots: generate Python code using matplotlib/seaborn with the planned style parameters. For methodology diagrams: generate Python code using matplotlib with patches/arrows, or SVG markup, or TikZ if the user works in LaTeX. Prefer code-based rendering over natural-language-to-image to maximize content fidelity.

6. **Render the initial illustration (Visualizer role).** Write and execute the rendering code. For matplotlib: produce a high-resolution PNG (300 DPI minimum). For SVG: produce clean, well-structured markup. Ensure all text is legible at the target display size.

7. **Self-critique the output (Critic role).** Evaluate the rendered figure against the content plan from step 3, checking: Are all specified components present? Are connections correct (no hallucinated or missing edges)? Is the layout balanced without excessive whitespace or crowding? Are labels readable? Does the color scheme maintain sufficient contrast?

8. **Refine based on critique.** Address each identified issue by modifying the rendering code. Common fixes: adjusting spacing/padding, resizing text, correcting arrow endpoints, rebalancing color contrast, removing redundant visual elements. Re-render after fixes.

9. **Repeat critique-refine once more.** Run a second critique pass. If no issues are found, the figure is complete. If issues persist, apply targeted fixes. Do not exceed 3 total iterations — diminishing returns set in quickly.

10. **Deliver the final output.** Present the final rendering code and image to the user. Include the structured content plan so the user can make manual adjustments later. If the figure was rendered as code, provide the complete executable script.

## Concrete Examples

**Example 1: Methodology Diagram for a Retrieval-Augmented Generation System**

User: "Create a methodology diagram for my RAG paper. The system takes a user query, retrieves relevant documents from a vector store, reranks them with a cross-encoder, then feeds the top-k documents with the query into an LLM to generate the answer."

Approach:
1. Extract components: User Query, Vector Store, Retriever, Cross-Encoder Reranker, Top-K Docs, LLM, Generated Answer.
2. Plan layout: left-to-right flow. Group retrieval components (Vector Store + Retriever + Reranker) in a dashed box labeled "Retrieval Pipeline".
3. Style: Blue tones for retrieval components, green for generation, gray for input/output. Sans-serif labels, rounded rectangles.
4. Render with matplotlib using `FancyBboxPatch` for nodes and `FancyArrowPatch` for directed edges.
5. Critique: Verify all 5 arrows are present, labels fit inside boxes, no overlapping elements.

Output (Python/matplotlib):
```python
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
from matplotlib.patches import FancyBboxPatch, FancyArrowPatch

fig, ax = plt.subplots(1, 1, figsize=(14, 4), dpi=300)
ax.set_xlim(0, 14)
ax.set_ylim(0, 4)
ax.axis('off')

# Style parameters (Stylist output)
colors = {'input': '#E8E8E8', 'retrieval': '#D4E6F1', 'rerank': '#AED6F1',
           'generation': '#A9DFBF', 'output': '#E8E8E8'}
font = {'family': 'sans-serif', 'size': 9, 'weight': 'bold'}

# Nodes (Planner output)
nodes = [
    ("User Query", 0.5, 1.7, 2.0, 0.8, colors['input']),
    ("Vector\nStore", 3.2, 1.7, 1.5, 0.8, colors['retrieval']),
    ("Cross-Encoder\nReranker", 5.4, 1.7, 2.0, 0.8, colors['rerank']),
    ("Top-K\nDocs", 8.1, 1.7, 1.4, 0.8, colors['retrieval']),
    ("LLM", 10.2, 1.7, 1.4, 0.8, colors['generation']),
    ("Answer", 12.3, 1.7, 1.4, 0.8, colors['output']),
]

for label, x, y, w, h, color in nodes:
    box = FancyBboxPatch((x, y), w, h, boxstyle="round,pad=0.1",
                          facecolor=color, edgecolor='#2C3E50', linewidth=1.2)
    ax.add_patch(box)
    ax.text(x + w/2, y + h/2, label, ha='center', va='center', fontdict=font)

# Retrieval group box
group = FancyBboxPatch((2.9, 1.2, ), 6.9, 1.8, boxstyle="round,pad=0.15",
                        facecolor='none', edgecolor='#2C3E50',
                        linewidth=1.0, linestyle='--')
ax.add_patch(group)
ax.text(6.35, 3.15, "Retrieval Pipeline", ha='center', fontsize=8,
        fontstyle='italic', color='#2C3E50')

plt.tight_layout()
plt.savefig("rag_methodology.png", dpi=300, bbox_inches='tight')
```

**Example 2: Statistical Results Plot**

User: "Plot these results as a grouped bar chart for my paper. Models: GPT-4, Claude, Llama. Metrics: Accuracy (92.1, 89.3, 85.7), F1 (90.5, 88.1, 83.2), Latency-ms (120, 95, 45)."

Approach:
1. Identify chart type: grouped bar chart, 3 models x 3 metrics.
2. Note Latency is on a different scale — use twin axes or separate subplot.
3. Style: academic color palette, hatching for grayscale compatibility, grid lines, proper axis labels.
4. Render with matplotlib/seaborn. Critique: verify data values match, bars are distinguishable in grayscale, legend is clear.

Output (Python/matplotlib):
```python
import matplotlib.pyplot as plt
import numpy as np

models = ['GPT-4', 'Claude', 'Llama']
accuracy = [92.1, 89.3, 85.7]
f1 = [90.5, 88.1, 83.2]
latency = [120, 95, 45]

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(10, 4), dpi=300,
                                gridspec_kw={'width_ratios': [2, 1]})

x = np.arange(len(models))
width = 0.3

# Quality metrics subplot
bars1 = ax1.bar(x - width/2, accuracy, width, label='Accuracy',
                color='#2980B9', hatch='//', edgecolor='white')
bars2 = ax1.bar(x + width/2, f1, width, label='F1 Score',
                color='#27AE60', hatch='\\\\', edgecolor='white')
ax1.set_ylabel('Score (%)', fontsize=10)
ax1.set_ylim(75, 100)
ax1.set_xticks(x)
ax1.set_xticklabels(models, fontsize=10)
ax1.legend(frameon=False, fontsize=9)
ax1.grid(axis='y', alpha=0.3)
ax1.set_title('Quality Metrics', fontsize=11, fontweight='bold')

# Latency subplot
ax2.bar(x, latency, width=0.5, color='#E74C3C', hatch='xx',
        edgecolor='white')
ax2.set_ylabel('Latency (ms)', fontsize=10)
ax2.set_xticks(x)
ax2.set_xticklabels(models, fontsize=10)
ax2.grid(axis='y', alpha=0.3)
ax2.set_title('Inference Latency', fontsize=11, fontweight='bold')

plt.tight_layout()
plt.savefig("results_comparison.png", dpi=300, bbox_inches='tight')
```

**Example 3: Neural Network Architecture Diagram**

User: "Illustrate a transformer encoder block for my paper: input embeddings go through multi-head self-attention, then add & norm, then a feed-forward network, then another add & norm, producing the output."

Approach:
1. Extract components: Input, Multi-Head Self-Attention, Add & Norm (x2), Feed-Forward Network, Output. Plus two residual skip connections.
2. Layout: vertical (bottom-to-top), matching the standard transformer diagram convention.
3. Style: soft pastel fills, thin black borders, curved residual arrows.
4. Render with matplotlib, using rectangles for layers and curved arrows for skip connections.
5. Critique: verify residual connections go around each sub-layer correctly (not through them), labels are centered, vertical spacing is even.
6. Refine: adjust arrow curvature so skip connections don't overlap with layer boxes.

## Best Practices

- **Do: Decompose before rendering.** Always produce a structured content plan (node list + edge list) before writing any rendering code. This catches logical errors (missing connections, wrong flow direction) before they become visual bugs.
- **Do: Use code-based rendering for statistical plots.** matplotlib/seaborn guarantees that data values map correctly to visual positions. Never use image generation models for data-driven figures — hallucinated bar heights or axis labels are unacceptable in publications.
- **Do: Include hatching patterns alongside colors.** Academic figures must be legible in grayscale print. Always add hatching (`//`, `\\`, `xx`, `..`) to bar charts and use marker shapes plus line styles for line plots.
- **Do: Set DPI to 300+ and use vector formats when possible.** Conference and journal submissions require high-resolution figures. Prefer SVG/PDF output for methodology diagrams.
- **Avoid: Overcrowding.** If a diagram has more than 10-12 nodes, group related components into labeled sub-regions (dashed boxes) to reduce visual complexity. Conciseness is a core evaluation criterion.
- **Avoid: Skipping the critique step.** The single most common failure in generated illustrations is hallucinated or missing connections between components. Always verify the rendered figure against the content plan before delivering.

## Error Handling

- **Missing components in rendering:** If the critique step finds that a node from the content plan is absent in the rendered figure, add it explicitly and re-render. Do not assume the renderer "implied" it.
- **Overlapping labels or boxes:** Increase figure dimensions or adjust node spacing. For matplotlib, use `fig.set_size_inches()` and recompute positions proportionally.
- **Connection errors (wrong arrow endpoints):** This is the most frequent failure mode identified in PaperBanana's evaluation. Verify every edge in the content plan has a corresponding arrow with the correct source and target nodes. Compare edge-by-edge.
- **Color accessibility issues:** Run a contrast check — ensure adjacent elements have a contrast ratio of at least 3:1. Replace low-contrast pairs with alternatives from the ColorBrewer qualitative palettes.
- **Matplotlib rendering failures:** If complex patch arrangements cause clipping or z-order issues, set explicit `zorder` values (background=0, boxes=1, arrows=2, text=3) and use `bbox_inches='tight'` on save.

## Limitations

- **Complex 3D or perspective diagrams** are beyond what code-based rendering handles well. Matplotlib is inherently 2D; for isometric or 3D architectural views, recommend the user use dedicated tools (Blender, Figma, draw.io).
- **Photorealistic or artistic illustrations** (e.g., conceptual teaser figures with real-world imagery) require diffusion models and are outside this code-generation workflow. The tradeoff is that diffusion models hallucinate content — labels, connections, and text in generated images are unreliable.
- **Very large diagrams** (20+ nodes with dense interconnections) become unreadable regardless of rendering quality. For these, recommend the user decompose into multiple subfigures or use hierarchical zoom-in panels.
- **Domain-specific notation** (circuit diagrams, chemical structures, musical scores) requires specialized rendering libraries beyond standard matplotlib. Direct the user to domain tools (Circuitikz, RDKit, Lilypond).
- **The self-critique loop relies on Claude's visual understanding.** If Claude cannot view rendered images in the current environment, the critique step must operate on the code structure rather than the visual output, reducing its effectiveness at catching layout/spacing issues.

## Reference

[PaperBanana: Automating Academic Illustration for AI Scientists](https://arxiv.org/abs/2601.23265v1) — Zhu et al., 2026. Key sections: the five-agent pipeline architecture (Retriever, Planner, Stylist, Visualizer, Critic), the dual rendering strategy (code vs. image generation), and the PaperBananaBench evaluation showing that iterative self-critique improves faithfulness and conciseness over single-pass baselines.