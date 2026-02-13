---
name: "orthogonal-hierarchical-decomposition-structure-aw"
description: "Decompose complex tables with multi-level headers, merged cells, and irregular layouts into orthogonal column/row trees for structure-aware question answering with LLMs. Use when: 'analyze this complex table', 'answer questions about this hierarchical table', 'parse table with merged cells', 'extract data from nested headers', 'table QA with multi-level columns', 'understand this pivot table structure'."
---

# Orthogonal Hierarchical Decomposition for Structure-Aware Table Understanding

This skill enables Claude to handle complex tables that defeat standard linearization (Markdown, HTML, CSV). When a table has multi-level headers, merged cells, or irregular layouts, Claude applies the Orthogonal Hierarchical Decomposition (OHD) framework: decomposing the table into independent column and row trees, reconstructing the full semantic lineage of each data cell through dual-pathway traversal, and producing a structure-preserving representation that resolves ambiguities flat formats cannot. This technique improved exact-match accuracy by +8.79 points over prior methods on the AITQA complex table QA benchmark.

## When to Use

- When the user provides a table with **multi-level or hierarchical column/row headers** and asks questions about it
- When a table contains **merged cells** that span multiple rows or columns, creating parent-child header relationships
- When a **pivot table, crosstab, or statistical report** has nested groupings (e.g., Region > Province > City across columns)
- When flat Markdown/HTML table rendering causes Claude to **misattribute data cells** to the wrong headers
- When the user asks to **extract, compare, or aggregate values** from tables where the meaning of a cell depends on tracing through multiple header levels
- When converting **complex spreadsheet data** into a structured format for downstream LLM reasoning

## Key Technique

**The core problem:** When LLMs encounter complex tables, standard linearization (Markdown rows, HTML `<table>`, CSV) collapses the hierarchical structure. A cell under a merged header like "Heilongjiang Province" spanning "Harbin" and "Other Cities" loses that parent-child relationship in flat text. This causes systematic errors — the LLM cannot trace which high-level category a data cell belongs to.

**The OHD solution:** Instead of flattening, decompose the table along two orthogonal axes independently. Build a **column tree** (capturing vertical hierarchical dependencies among column headers) and a **row tree** (capturing horizontal hierarchical dependencies among row headers). Each tree is constructed using spatial-semantic co-constraints: a child header must be (1) spatially contained within the parent's span, and (2) semantically subordinate to it (validated by LLM judgment). This dual constraint prevents errors where spatial proximity doesn't imply semantic hierarchy — e.g., "Details in 2007" appearing spatially under a "2016" column but being logically independent.

**Dual-pathway reconstruction:** Once both trees exist, every data cell's full meaning is reconstructed by tracing its lineage through both trees. The primary tree provides the "premise" context (e.g., column path: Country > Province > City), and the orthogonal tree provides the "attribute" context (e.g., row path: Metric > Sub-metric). These combine into a `Context > Key > Value` representation: "Heilongjiang Province > Harbin > Peak-season standard: 450 yuan". An LLM arbitrator then selects the most coherent merged representation from the column-major and row-major views.

## Step-by-Step Workflow

1. **Parse the table into a cell grid.** Convert the input (HTML, Markdown, image-extracted, spreadsheet) into a structured grid where each cell has: content, row span, column span, start row, start column. Identify which cells are column headers, row headers, or data cells. Merged cells retain their full span information.

2. **Sort headers lexicographically.** Order all column headers by (start_row, start_column) and all row headers by (start_column, start_row). This establishes a top-to-bottom, left-to-right processing order critical for tree construction.

3. **Build the column tree (Stage I: Skeleton Induction).** For each column header in sorted order, find the nearest preceding header whose column span contains the current header's span AND whose end row is at or above the current header's start row. Validate that the parent semantically subsumes the child (e.g., "Province" subsumes "City"). Add the edge. If no valid parent exists, attach to the root.

4. **Build the column tree (Stage II: Data Anchoring).** For each data cell, find the leaf-level column header(s) whose column span overlaps the cell's column span. If multiple candidates exist (conflict set), use row-boundary analysis and semantic validation to determine the correct parent. Attach the data cell as a leaf node.

5. **Repeat steps 3-4 for the row tree.** Apply the same two-stage process along the row axis, using row headers and their row spans. The row tree captures horizontal hierarchical structure independently of the column tree.

6. **Extract dual-pathway lineage for each data cell.** For each data cell, trace its path from root to leaf in both trees. The column-tree path gives the "premise lineage" (e.g., `Region > Subregion > City`). The row-tree path gives the "attribute lineage" (e.g., `Year > Quarter > Metric`).

7. **Reconstruct column-major and row-major representations.** Perform depth-first traversal of each tree. At each data cell leaf, emit its full lineage: `premise_lineage > (attribute_lineage => value)`. The column-major version uses the column tree as primary; the row-major version uses the row tree.

8. **Arbitrate between representations.** Present both the column-major and row-major reconstructions. Evaluate which provides better logical cohesion, information completeness, and readability for the specific question. Select or merge the best representation.

9. **Answer the question using the structure-aware representation.** Feed the chosen representation to the LLM with the user's question. The hierarchical lineage ensures every data value carries its full semantic context, preventing misattribution.

10. **Validate the answer by cross-referencing both tree paths.** If the answer involves aggregation or comparison, verify that the cells being compared share the correct hierarchical context by checking their tree paths align at the appropriate level.

## Concrete Examples

**Example 1: Provincial Statistics with Merged Column Headers**

User: "Here's a table of hotel pricing by province and city. In how many provinces does the peak-season standard exceed 450 yuan?"

```
Input table structure:
| (merged)              | Heilongjiang Province (cols 1-2) | Jilin Province (cols 3-4) | Liaoning Province (cols 5-6) |
|                       | Harbin        | Other Cities    | Changchun  | Other Cities  | Shenyang    | Other Cities    |
| Peak-season standard  | 480           | 350             | 460        | 380           | 520         | 400             |
| Off-season standard   | 380           | 280             | 360        | 300           | 420         | 320             |
```

Approach:
1. Identify column headers: Level 1 = Province names (merged), Level 2 = City categories
2. Build column tree:
   ```
   root
   ├── Heilongjiang Province
   │   ├── Harbin [480, 380]
   │   └── Other Cities [350, 280]
   ├── Jilin Province
   │   ├── Changchun [460, 360]
   │   └── Other Cities [380, 300]
   └── Liaoning Province
       ├── Shenyang [520, 420]
       └── Other Cities [400, 320]
   ```
3. Build row tree: `root > Peak-season standard`, `root > Off-season standard`
4. Reconstruct lineage for each peak-season cell:
   - "Heilongjiang Province > Harbin > Peak-season standard: 480"
   - "Heilongjiang Province > Other Cities > Peak-season standard: 350"
   - "Jilin Province > Changchun > Peak-season standard: 460"
   - etc.
5. Answer: Check each *province* (not city). Heilongjiang has Harbin at 480 (>450). Jilin has Changchun at 460 (>450). Liaoning has Shenyang at 520 (>450). Answer: **3 provinces**.

Note: Flat Markdown loses the province grouping, causing models to count individual cities (yielding 3 cities, not 3 provinces) or misattribute "Other Cities" rows.

**Example 2: Time-Series Table with Irregular Row Headers**

User: "What was the population percentage with less than a bachelor's degree in the first half of 2007?"

```
Input table (simplified):
| (merged)           | Education Level | 2016 H1 | 2016 H2 | Details in 2007 (cols 4-5) |
|                    |                 |          |          | 2007 H1    | 2007 H2       |
| Graduate+          | Masters+        | 15%      | 16%      | 10%        | 11%            |
|                    | Bachelors       | 25%      | 26%      | 20%        | 21%            |
| Below Bachelor     | Associates      | 30%      | 29%      | 35%        | 34%            |
|                    | High School     | 30%      | 29%      | 35%        | 34%            |
```

Approach:
1. Column tree construction: "Details in 2007" spatially sits beside "2016" columns — NOT under them. Semantic validation confirms "2007" is not a child of "2016". Tree correctly places them as siblings under root.
2. Row tree: `root > Graduate+ > {Masters+, Bachelors}`, `root > Below Bachelor > {Associates, High School}`
3. Target cells: Row tree path containing "Below Bachelor" intersected with column tree path "Details in 2007 > 2007 H1"
4. Sum: Associates 35% + High School 35% = **70%** (or read directly if aggregated)

**Example 3: Programmatic Table Decomposition**

User: "Write Python code to decompose this HTML table into column and row trees."

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class Cell:
    content: str
    row_start: int
    col_start: int
    row_span: int = 1
    col_span: int = 1
    role: str = "data"  # "col_header", "row_header", or "data"

    @property
    def row_end(self): return self.row_start + self.row_span
    @property
    def col_end(self): return self.col_start + self.col_span

@dataclass
class TreeNode:
    cell: Optional[Cell]
    children: list = field(default_factory=list)
    parent: Optional['TreeNode'] = None

    def lineage(self) -> list[str]:
        """Trace path from root to this node."""
        path = []
        node = self
        while node and node.cell:
            path.append(node.cell.content)
            node = node.parent
        return list(reversed(path))

def build_column_tree(col_headers: list[Cell], data_cells: list[Cell]) -> TreeNode:
    """OTI Stage I + II for column axis."""
    root = TreeNode(cell=None)
    # Sort lexicographically: (row_start, col_start)
    sorted_headers = sorted(col_headers, key=lambda c: (c.row_start, c.col_start))
    nodes = {id(root): root}
    header_nodes = []

    # Stage I: Skeleton Induction
    for h in sorted_headers:
        best_parent = root
        for prev_node in reversed(header_nodes):
            prev = prev_node.cell
            if (h.col_start >= prev.col_start and
                h.col_end <= prev.col_end and
                prev.row_end <= h.row_start):
                best_parent = prev_node
                break
        node = TreeNode(cell=h, parent=best_parent)
        best_parent.children.append(node)
        header_nodes.append(node)

    # Stage II: Anchor data cells to leaf headers
    leaves = [n for n in header_nodes if not n.children]
    for d in data_cells:
        matching = [l for l in leaves
                    if (d.col_start < l.cell.col_end and
                        d.col_end > l.cell.col_start)]
        if matching:
            parent = matching[0]  # resolve conflicts via semantic check
            data_node = TreeNode(cell=d, parent=parent)
            parent.children.append(data_node)
    return root

def reconstruct_lineage(col_tree: TreeNode, row_tree: TreeNode,
                        data_cells: list[Cell]) -> list[str]:
    """Dual-pathway lineage reconstruction."""
    descriptions = []
    for d in data_cells:
        col_path = _find_cell_lineage(col_tree, d)
        row_path = _find_cell_lineage(row_tree, d)
        desc = " > ".join(col_path) + " | " + " > ".join(row_path) + f": {d.content}"
        descriptions.append(desc)
    return descriptions

def _find_cell_lineage(tree: TreeNode, target: Cell) -> list[str]:
    """DFS to find target cell and return its path."""
    def _dfs(node, path):
        current_path = path + ([node.cell.content] if node.cell else [])
        if node.cell and node.cell is target:
            return current_path
        for child in node.children:
            result = _dfs(child, current_path)
            if result:
                return result
        return None
    return _dfs(tree, []) or []
```

## Best Practices

- **Do:** Always identify cell roles (column header, row header, data) before building trees. The decomposition depends entirely on correct role classification.
- **Do:** Validate parent-child relationships semantically, not just spatially. A cell spatially nested under another may be logically independent (e.g., "2007 details" under a "2016" column).
- **Do:** Build column and row trees independently. Cross-dimensional interference is the primary source of errors in flat linearization.
- **Do:** Reconstruct both column-major and row-major representations before answering. Some questions are better served by one axis than the other.
- **Avoid:** Flattening multi-level headers into concatenated strings (e.g., "Heilongjiang Province - Harbin") without preserving the tree structure. This loses the ability to aggregate at intermediate levels.
- **Avoid:** Assuming all tables need OHD. Simple tables with single-level headers and no merged cells work fine with standard Markdown linearization.

## Error Handling

- **Ambiguous cell roles:** If you cannot determine whether a cell is a header or data, treat cells in the topmost rows and leftmost columns as headers by default, then validate semantically.
- **Conflicting spans:** When a data cell's column span overlaps multiple leaf headers, check row boundaries to disambiguate. If still ambiguous, present both interpretations to the user.
- **Semantic validation failure:** If two headers are spatially nested but you cannot determine semantic subsumption (e.g., domain-specific jargon), flag the uncertainty and ask the user to clarify the header hierarchy.
- **Extremely large tables (>50x50):** The dual-pathway reconstruction can produce very long text. Apply boundary-aware truncation: only reconstruct lineages for cells relevant to the user's question, not the entire table.
- **Missing headers:** Some tables omit explicit headers for the row dimension. Infer row headers from the leftmost non-numeric column, or treat the table as column-only hierarchy with a degenerate row tree.

## Limitations

- **Requires identifiable header structure.** Tables that are purely data grids with no hierarchical headers gain no benefit from OHD.
- **Depends on correct span detection.** If the input source (HTML, image OCR) misidentifies merged cell boundaries, the tree construction will be wrong. Always verify spans.
- **Semantic validation is imperfect.** The LLM-based semantic subsumption check can fail on highly domain-specific terminology or when header labels are abbreviations.
- **Not designed for relational joins.** OHD handles single-table understanding. For questions requiring joins across multiple tables, additional techniques are needed.
- **Context length limits still apply.** For very large tables, even the structure-preserving representation may exceed context windows. Prioritize relevant subtrees.

## Reference

Paper: [Orthogonal Hierarchical Decomposition for Structure-Aware Table Understanding with Large Language Models](https://arxiv.org/abs/2602.01969v1) (Cao et al., 2026). Key sections: Section 3 for OTI algorithm details, Section 4 for dual-pathway protocol, Section 5 for AITQA/HiTab benchmark results showing +8.79 EM improvement over prior art.