---
name: "st-raptor-agentic-system-semi-structured"
description: "Agentic system for answering questions about semi-structured tables using tree-based structural modeling and multi-step agent reasoning. Use when: 'analyze this spreadsheet and answer questions', 'parse this complex table with merged cells', 'extract data from a semi-structured table', 'answer questions about this financial/HR table', 'convert messy table to structured format', 'build a table QA pipeline'."
---

# ST-Raptor: Agentic Semi-Structured Table Question Answering

This skill enables Claude to answer natural language questions over semi-structured tables — tables with merged cells, nested headers, irregular layouts, and implicit hierarchical structure — by applying the ST-Raptor methodology. Instead of naively flattening tables into SQL-compatible formats (which loses structural information) or relying on raw multimodal vision (which struggles with complex layouts), this approach constructs a Hierarchical-Overlapping Tree (HO-Tree) representation, then uses an agent loop with retrieval-augmented reasoning to answer queries accurately.

## When to Use

- When the user provides a table (Excel, HTML, CSV, Markdown, or image) with merged cells, multi-level headers, or irregular layout and asks questions about it
- When a Text-to-SQL approach fails because the table cannot be cleanly mapped to a relational schema
- When the user needs to extract specific values from financial statements, HR forms, academic schedules, or sales reports with complex formatting
- When building a pipeline that must programmatically answer batches of questions over semi-structured tabular data
- When the user wants to convert a visually complex table into a navigable tree structure for downstream analysis
- When standard pandas or SQL operations cannot handle the table's hierarchical or overlapping structure

## Key Technique

**The core insight**: Semi-structured tables encode meaning in their *layout* — merged cells indicate grouping, indentation implies hierarchy, and spatial adjacency conveys association. Flattening these tables into rows and columns destroys this information. ST-Raptor preserves it by constructing a **Hierarchical-Overlapping Tree (HO-Tree)** — a tree data structure where each node represents a cell or cell region, parent-child relationships capture header-to-data mappings, and overlapping subtrees represent cells that belong to multiple logical groups (e.g., a cell under both a row header and a column header).

**The pipeline works in three stages**: (1) **Table Preprocessing & Tree Construction** — parse the raw table, detect merged cells and header regions, and build the HO-Tree by analyzing cell positions, spans, and content semantics. (2) **Embedding & Retrieval** — embed tree nodes using a multilingual embedding model, then retrieve the most relevant subtree(s) for a given question using semantic similarity. (3) **Agent-Driven Query Resolution** — a ReAct-style agent loop reasons over the retrieved subtree context, generates intermediate computations (arithmetic, filtering, aggregation), validates answers through a two-stage verification, and returns the final result.

**What makes this better**: By operating on a tree rather than a flat table, the agent can traverse parent-child relationships to resolve ambiguous references (e.g., "the Q3 revenue for the Northern region" requires navigating from "Northern" down to "Q3" then to the revenue cell). The two-stage validation catches hallucinated values by cross-checking against the original cell contents.

## Step-by-Step Workflow

1. **Ingest the table**: Read the input file (Excel, HTML, CSV, Markdown, or screenshot). If it is an image, use OCR or a vision model to extract cell contents and positions. For structured formats, parse directly using openpyxl, pandas, or BeautifulSoup, preserving merge info.

2. **Detect structural regions**: Identify header rows/columns, data regions, and merged cell spans. For Excel files, read `merged_cells` ranges. For HTML, parse `rowspan`/`colspan` attributes. Classify each cell as: header, sub-header, data, or annotation.

3. **Build the HO-Tree**: Construct a tree where:
   - The root represents the entire table
   - Level 1 children represent top-level column groups (from merged header cells)
   - Level 2+ children represent sub-headers and row groups
   - Leaf nodes represent individual data cells
   - Overlapping nodes are created when a cell logically belongs to multiple parent groups
   Store each node as `{id, content, cell_range, parent_ids, children_ids, level, metadata}`.

4. **Generate embeddings**: For each tree node, create a contextualized text representation by concatenating the node's content with its ancestor path (e.g., "Northern Region > Q3 2024 > Revenue > $1.2M"). Embed these strings using a sentence embedding model or the LLM's own embeddings.

5. **Parse the user's question**: Analyze the natural language query to identify: the target value(s) being asked about, any filtering conditions (e.g., "for department X"), the operation type (lookup, comparison, aggregation, counting), and any temporal or categorical constraints.

6. **Retrieve relevant subtree**: Compute semantic similarity between the question embedding and all node embeddings. Select the top-k most relevant nodes, then expand to include their ancestors and descendants to form a coherent subtree context.

7. **Agent reasoning loop**: Using a ReAct pattern, iterate:
   - **Think**: Identify what information is needed to answer the question
   - **Act**: Look up specific cells in the retrieved subtree, perform calculations, or request additional subtree expansions
   - **Observe**: Check the result against the tree structure
   - Repeat until the answer is determined or a max iteration limit is reached.

8. **Two-stage validation**: (a) Verify that every referenced cell value actually exists in the original table at the claimed position. (b) Re-derive the answer using an alternative reasoning path and check consistency.

9. **Format and return the answer**: Present the answer with a citation trail showing which cells were used, their positions in the original table, and the reasoning chain.

10. **Handle failure gracefully**: If the agent cannot resolve the query within the iteration limit, return partial results with an explanation of what was ambiguous or unresolvable, and suggest how the user might clarify.

## Concrete Examples

**Example 1: Financial Statement with Merged Headers**

```
User: "What was the total operating expense for Q2 2024?"

Input table (Excel with merged cells):
┌──────────────────────┬────────────┬────────────┐
│                      │   H1 2024  │   H2 2024  │
│                      ├──────┬─────┼──────┬─────┤
│                      │  Q1  │  Q2 │  Q3  │  Q4 │
├──────────────────────┼──────┼─────┼──────┼─────┤
│ Revenue              │ 500  │ 620 │ 710  │ 680 │
├──────────────────────┼──────┼─────┼──────┼─────┤
│ Operating Expenses   │      │     │      │     │
│   Salaries           │ 200  │ 230 │ 250  │ 240 │
│   Rent               │  50  │  50 │  55  │  55 │
│   Marketing          │  80  │  95 │ 100  │  90 │
├──────────────────────┼──────┼─────┼──────┼─────┤
│ Net Income           │ 170  │ 245 │ 305  │ 295 │
└──────────────────────┴──────┴─────┴──────┴─────┘

Approach:
1. Parse Excel, detect "H1 2024" spans columns Q1-Q2, "Operating Expenses" is
   a parent row group for Salaries/Rent/Marketing.
2. Build HO-Tree: Root -> [H1 2024 [Q1, Q2], H2 2024 [Q3, Q4]] and
   Root -> [Revenue, Operating Expenses [Salaries, Rent, Marketing], Net Income]
3. Question targets "total operating expense" + "Q2 2024" ->
   navigate to Operating Expenses children, filter to Q2 column.
4. Retrieve: Salaries/Q2=230, Rent/Q2=50, Marketing/Q2=95
5. Compute: 230 + 50 + 95 = 375
6. Validate: Each value exists in the table at the correct cell position.

Output: "The total operating expense for Q2 2024 was 375
(Salaries: 230 + Rent: 50 + Marketing: 95)."
```

**Example 2: HR Form with Irregular Layout**

```
User: "How many employees in the Engineering department have more than 5 years
of experience?"

Input: A semi-structured HR table where "Engineering" is a merged cell spanning
4 rows, with sub-rows for each employee containing Name, Role, and Years columns.

Approach:
1. Parse the table, detect "Engineering" as a merged row-header spanning
   employee sub-rows.
2. Build HO-Tree: Department node "Engineering" has children nodes for each
   employee row, each containing Name/Role/Years leaf nodes.
3. Question requires: filter to Engineering department, then count where
   Years > 5.
4. Traverse Engineering subtree, extract Years values for each child.
5. Filter: [3, 7, 12, 2] -> values > 5: [7, 12] -> count = 2.
6. Validate against original cell positions.

Output: "2 employees in Engineering have more than 5 years of experience."
```

**Example 3: Building a Batch QA Pipeline in Python**

```
User: "I have 50 Excel files with semi-structured tables and a JSONL file of
questions. Build me a pipeline to answer them all."

Approach:
1. Create a Python script with three modules:
   - table_parser.py: Uses openpyxl to read Excel files, detect merged cells,
     and output a standardized table dict with cell contents + merge metadata.
   - tree_builder.py: Takes the parsed table dict and constructs an HO-Tree
     as a nested dict/dataclass structure. Implements header detection heuristics
     (first N rows with merged cells are headers, indented rows are children).
   - query_agent.py: Implements the ReAct loop using an LLM API. Embeds tree
     nodes, retrieves relevant subtrees per question, runs the reasoning loop,
     and validates answers.
2. Main pipeline reads JSONL, groups questions by table_id, processes each table
   once (parse + build tree + embed), then answers all associated questions.
3. Output results as JSONL with fields: {id, question, answer, evidence_cells,
   confidence}.

Output: A working Python project with the three modules, a main.py entry point,
and a requirements.txt (openpyxl, sentence-transformers, openai).
```

## Implementation Pattern (Python)

When building an HO-Tree from a parsed table, use this data structure:

```python
from dataclasses import dataclass, field

@dataclass
class TreeNode:
    node_id: str
    content: str
    cell_range: tuple  # (row_start, col_start, row_end, col_end)
    node_type: str     # "header", "sub_header", "data", "group"
    parent_ids: list[str] = field(default_factory=list)
    children_ids: list[str] = field(default_factory=list)
    level: int = 0
    metadata: dict = field(default_factory=dict)

    def ancestor_path(self, tree: dict[str, "TreeNode"]) -> str:
        """Build context string from root to this node."""
        parts = [self.content]
        current = self
        while current.parent_ids:
            current = tree[current.parent_ids[0]]
            parts.append(current.content)
        return " > ".join(reversed(parts))
```

For merged cell detection in openpyxl:

```python
import openpyxl

def extract_merged_regions(filepath: str) -> list[dict]:
    wb = openpyxl.load_workbook(filepath)
    ws = wb.active
    regions = []
    for merge_range in ws.merged_cells.ranges:
        top_left = ws.cell(merge_range.min_row, merge_range.min_col)
        regions.append({
            "content": top_left.value,
            "range": (merge_range.min_row, merge_range.min_col,
                      merge_range.max_row, merge_range.max_col),
            "row_span": merge_range.max_row - merge_range.min_row + 1,
            "col_span": merge_range.max_col - merge_range.min_col + 1,
        })
    return regions
```

## Best Practices

- **Do**: Always preserve merge metadata when parsing tables — this is the primary signal for hierarchy detection. Never discard `rowspan`/`colspan` or Excel merge ranges.
- **Do**: Build the ancestor path string for each node before embedding. Embedding "Revenue" alone is ambiguous; embedding "Company X > Q2 2024 > Revenue > $620" is precise.
- **Do**: Use two-stage validation — first verify cell values exist at claimed positions, then re-derive the answer via an alternative path. This catches hallucinated numbers.
- **Do**: When the table is too large for the LLM context window, retrieve only the relevant subtree rather than serializing the entire table.
- **Avoid**: Flattening semi-structured tables into a single pandas DataFrame as a first step — this destroys the hierarchical information you need. Parse structure *first*, then flatten only the leaf data if needed for computation.
- **Avoid**: Treating all tables as relational. If a table has merged cells, nested headers, or annotation rows, a SQL-first approach will produce incorrect results. Check for structural complexity before choosing a strategy.

## Error Handling

- **Ambiguous cell references**: If multiple cells match a query term (e.g., "Revenue" appears in both row headers and a column), use the tree structure to disambiguate by requiring the user to specify the parent context, or rank by relevance to the full question.
- **OCR/parsing errors**: When working from images, cross-validate extracted values against numerical patterns (e.g., a currency column should contain numbers). Flag anomalies for human review.
- **Missing values**: If the target cell is empty or contains "N/A", report this explicitly rather than hallucinating a value. State: "Cell [X, Y] is empty in the original table."
- **Tree construction failures**: If the heuristic header detection misclassifies data rows as headers (or vice versa), fall back to treating the first row as the header and building a flat tree, then ask the user to confirm the structure.
- **Agent loop divergence**: Cap the ReAct loop at 5-8 iterations. If no answer is found, return the best partial result with the reasoning trace so the user can guide the next step.

## Limitations

- **Purely visual tables** (tables in images with no underlying structured format) require a VLM or OCR step that introduces extraction errors — accuracy degrades for handwritten or low-resolution inputs.
- **Very large tables** (1000+ cells) may exceed context limits even with subtree retrieval. Consider chunking strategies or pre-filtering by section.
- **Tables with no structural markers** (no merged cells, no indentation, purely flat grids) do not benefit from tree-based modeling — use standard pandas/SQL for these.
- **Domain-specific semantics**: The tree captures layout hierarchy but not domain knowledge. A financial table where "EBITDA" must be computed from other rows requires domain-aware post-processing rules, not just structural navigation.
- **Multi-table joins**: ST-Raptor targets single-table QA. Cross-table questions require an additional orchestration layer to identify and join relevant tables.

## Reference

- **Paper**: [ST-Raptor: An Agentic System for Semi-Structured Table QA](https://arxiv.org/abs/2602.07034v1) — Focus on Section 3 (HO-Tree construction algorithm) and Section 4 (agent-driven query pipeline with two-stage validation) for implementation details.
- **Code**: [github.com/weAIDB/ST-Raptor](https://github.com/weAIDB/ST-Raptor) — Reference `table2tree/feature_tree.py` for tree construction and `query/primitive_pipeline.py` for the agent reasoning loop.