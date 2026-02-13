---
name: "tangrampuzzle-evaluating-multimodal-compositional"
description: "Evaluate and build compositional spatial reasoning systems using geometry-grounded benchmarks and symbolic coordinate frameworks. Use when: 'build a tangram puzzle solver', 'evaluate spatial reasoning of a vision model', 'generate geometric assembly code from a shape outline', 'create a benchmark for compositional spatial tasks', 'verify geometric constraint satisfaction in 2D layouts', 'test if an LLM can decompose shapes into rigid pieces'."
---

# Compositional Spatial Reasoning with Geometry-Grounded Evaluation

This skill enables Claude to build, evaluate, and apply compositional spatial reasoning systems inspired by the TangramPuzzle benchmark. The core technique uses a Tangram Construction Expression (TCE) — a symbolic geometric framework that encodes 2D shape assemblies as exact, machine-verifiable coordinate specifications in JSON. Rather than relying on visual approximation or coarse bounding-box reasoning, TCE grounds every piece in algebraic coordinates with explicit transformation matrices, adjacency graphs, and edge relations. This allows rigorous automated validation of whether an assembly is geometrically correct: no overlaps, no gaps, pieces preserve their original area and perimeter, and the union matches a target silhouette.

## When to Use

- When the user asks to build a tangram puzzle solver or any system that decomposes a 2D silhouette into a fixed set of rigid polygonal pieces
- When evaluating whether a multimodal model can perform precise spatial reasoning (not just object recognition or coarse positioning)
- When generating code that assembles geometric primitives to fill a target outline, subject to rigid-body constraints
- When designing benchmarks or test suites for compositional spatial understanding in vision-language models
- When verifying that a proposed 2D tiling or layout satisfies no-overlap, no-gap, and boundary-matching constraints
- When building educational geometry tools that require exact coordinate-level reasoning about shape composition

## Key Technique

**Tangram Construction Expression (TCE)** is a 5-tuple symbolic representation: `TCE = <id, target_outline, initial_state, final_state, adjacency_graph>`. The target outline defines the silhouette as a list of vertices and edges. The initial state describes each of the seven standard tangram pieces (two large triangles, one medium triangle, two small triangles, one square, one parallelogram) with their canonical vertex coordinates. The final state encodes the assembled position of each piece via a 3x3 homogeneous transformation matrix (encoding rotation, translation, and optional reflection) plus the resulting world-space vertices. The adjacency graph captures which pieces share edges in the final assembly. All coordinates use exact algebraic expressions (e.g., `2*sqrt(2)`) rather than floating-point approximations.

**Three-stage validation** ensures geometric correctness. First, **syntax validation (TSE)** confirms the output is valid TCE JSON with exactly seven pieces. Second, **rigid geometry error checking (RGE)** verifies that each piece's area and perimeter are preserved after transformation — catching distortions or deformations. Third, **physical error checking (PE)** detects impermissible overlaps between pieces and verifies the union forms a single connected component matching the target. Quantitative metrics include IoU (global area overlap between predicted assembly and target), Hausdorff distance (boundary deviation), and Validation Pass Rate (percentage satisfying all constraints).

**The critical insight from the paper**: LLMs tend to prioritize silhouette matching while violating geometric constraints. Effective systems must enforce rigid-body invariants as hard constraints, not soft objectives. This means any solver or generator must validate piece preservation independently of outline matching.

## Step-by-Step Workflow

1. **Define the seven canonical tangram pieces** as polygons with exact algebraic vertex coordinates in a standard reference frame. Use a unit square of side 2 as the base, yielding pieces with vertices involving integers and `sqrt(2)` terms. Store these as the `initial_state` in TCE format.

2. **Encode the target silhouette** as an ordered list of vertices and edges forming a closed polygon (the `target_outline`). Normalize the target so its centroid sits at the origin and its bounding box fits within a consistent scale.

3. **For assembly/solving tasks**: formulate the problem as finding seven rigid transformations (rotation + translation + optional reflection) such that (a) each transformed piece preserves its original area and perimeter, (b) no two pieces overlap (intersection area < epsilon), (c) the union of all pieces equals the target outline (IoU > 0.99).

4. **Generate TCE JSON output** with `final_state` containing each piece's transformation matrix and world-space vertices. Use exact algebraic expressions where possible; when using floating point, keep precision to at least 6 decimal places.

5. **Run syntax validation**: confirm the JSON parses correctly, contains exactly 7 pieces, each with the correct piece type, vertex count, and edge list matching the canonical definition.

6. **Run rigid geometry validation**: for each piece, compute area (via shoelace formula) and perimeter of the transformed polygon. Compare against the canonical values. Flag any piece where `|area_transformed - area_canonical| / area_canonical > 0.01` or equivalent perimeter deviation.

7. **Run physical constraint validation**: compute pairwise intersection areas between all 21 piece pairs using polygon clipping (Sutherland-Hodgman or Shapely). Flag if any intersection area exceeds epsilon. Compute the union polygon and verify it forms a single connected component.

8. **Compute evaluation metrics**: calculate IoU between the union of pieces and the target outline, Hausdorff distance between their boundaries, and report the Validation Pass Rate across a test set.

9. **For outline prediction tasks** (discriminative): given a set of positioned pieces, compute their union polygon and match against candidate silhouettes using IoU ranking.

10. **Iterate on failures**: when validation fails, diagnose whether the issue is a piece distortion (RGE failure) or a placement error (PE failure), and adjust the corresponding transformation matrix accordingly.

## Concrete Examples

**Example 1: Building a Tangram Piece Library in Python**

User: "Create a Python module that defines the 7 standard tangram pieces with exact coordinates and can apply rigid transformations to them."

Approach:
1. Define canonical pieces on a 2x2 grid with algebraic coordinates
2. Implement a transformation function using 3x3 homogeneous matrices
3. Add validation that area/perimeter are preserved after transformation

Output:
```python
import numpy as np
from shapely.geometry import Polygon
from typing import List, Tuple
import math

# Canonical tangram pieces on a 2x2 square base
CANONICAL_PIECES = {
    "large_triangle_1": [(0, 0), (2, 0), (1, 1)],
    "large_triangle_2": [(2, 0), (2, 2), (1, 1)],
    "medium_triangle":  [(0, 0), (1, 1), (0, 2)],
    "small_triangle_1": [(1, 1), (1.5, 0.5), (2, 1)],  # exact: midpoint-derived
    "small_triangle_2": [(0, 2), (1, 1), (0.5, 1.5)],
    "square":           [(1, 1), (1.5, 0.5), (2, 1), (1.5, 1.5)],
    "parallelogram":    [(0.5, 1.5), (1, 1), (2, 1), (1.5, 1.5)],  # may need reflection
}

def make_transform(angle_deg: float, tx: float, ty: float, reflect: bool = False) -> np.ndarray:
    """Build a 3x3 homogeneous rigid transformation matrix."""
    theta = math.radians(angle_deg)
    c, s = math.cos(theta), math.sin(theta)
    R = np.array([[c, -s], [s, c]])
    if reflect:
        R = R @ np.array([[1, 0], [0, -1]])  # reflect across x-axis before rotation
    M = np.eye(3)
    M[:2, :2] = R
    M[:2, 2] = [tx, ty]
    return M

def apply_transform(vertices: List[Tuple[float, float]], M: np.ndarray) -> List[Tuple[float, float]]:
    """Apply 3x3 homogeneous transform to polygon vertices."""
    pts = np.array(vertices)
    ones = np.ones((pts.shape[0], 1))
    homogeneous = np.hstack([pts, ones])
    transformed = (M @ homogeneous.T).T
    return [(round(x, 8), round(y, 8)) for x, y in transformed[:, :2]]

def validate_rigid(original: List[Tuple], transformed: List[Tuple], tol: float = 0.01) -> bool:
    """Verify area and perimeter preservation after transformation."""
    p_orig = Polygon(original)
    p_trans = Polygon(transformed)
    area_ok = abs(p_trans.area - p_orig.area) / p_orig.area < tol
    perim_ok = abs(p_trans.length - p_orig.length) / p_orig.length < tol
    return area_ok and perim_ok
```

**Example 2: Validating a Tangram Assembly**

User: "Write a validation pipeline that checks if a proposed tangram assembly is geometrically valid."

Approach:
1. Parse the TCE JSON with piece positions
2. Run three-stage validation (syntax, rigid geometry, physical constraints)
3. Report metrics (IoU, Hausdorff, pass/fail per stage)

Output:
```python
from shapely.geometry import Polygon, MultiPolygon
from shapely.ops import unary_union
from shapely.validation import make_valid
import json

def validate_tce(tce_json: dict, canonical: dict) -> dict:
    report = {"syntax": True, "rigid": True, "physical": True, "metrics": {}}

    # Stage 1: Syntax validation
    pieces = tce_json.get("final_state", {})
    if len(pieces) != 7:
        report["syntax"] = False
        return report

    # Stage 2: Rigid geometry validation
    polys = []
    for piece_id, piece_data in pieces.items():
        verts = piece_data["vertices"]
        poly = Polygon(verts)
        polys.append((piece_id, poly))
        canon = Polygon(canonical[piece_data["type"]])
        area_err = abs(poly.area - canon.area) / canon.area
        perim_err = abs(poly.length - canon.length) / canon.length
        if area_err > 0.01 or perim_err > 0.01:
            report["rigid"] = False
            report[f"rge_{piece_id}"] = {"area_err": area_err, "perim_err": perim_err}

    # Stage 3: Physical constraint validation
    for i in range(len(polys)):
        for j in range(i + 1, len(polys)):
            intersection = polys[i][1].intersection(polys[j][1])
            if intersection.area > 1e-6:
                report["physical"] = False
                report[f"overlap_{polys[i][0]}_{polys[j][0]}"] = intersection.area

    # Compute metrics
    assembly = unary_union([p for _, p in polys])
    target = Polygon(tce_json["target_outline"]["vertices"])
    iou = assembly.intersection(target).area / assembly.union(target).area
    hausdorff = assembly.hausdorff_distance(target)

    report["metrics"]["iou"] = round(iou, 4)
    report["metrics"]["hausdorff"] = round(hausdorff, 4)
    report["metrics"]["valid"] = report["syntax"] and report["rigid"] and report["physical"]
    return report
```

**Example 3: Generating a TCE JSON for a Target Shape**

User: "Given a square target, produce a valid TCE JSON placing all 7 tangram pieces."

Approach:
1. Start with the canonical 2x2 square as target
2. Place pieces in their default positions (identity transforms)
3. Validate and output TCE

Output:
```json
{
  "instance_id": "square_001",
  "target_outline": {
    "vertices": [[0,0], [2,0], [2,2], [0,2]],
    "edges": [[0,1], [1,2], [2,3], [3,0]]
  },
  "initial_state": { "...canonical piece definitions..." },
  "final_state": {
    "large_triangle_1": {
      "type": "large_triangle",
      "transform_matrix": [[1,0,0],[0,1,0],[0,0,1]],
      "vertices": [[0,0],[2,0],[1,1]]
    },
    "large_triangle_2": {
      "type": "large_triangle",
      "transform_matrix": [[1,0,0],[0,1,0],[0,0,1]],
      "vertices": [[2,0],[2,2],[1,1]]
    }
  },
  "adjacency_graph": [
    ["large_triangle_1", "large_triangle_2"],
    ["large_triangle_1", "square"],
    ["square", "small_triangle_1"]
  ]
}
```

## Best Practices

- **Do:** Use exact algebraic coordinates (`sqrt(2)/2`, not `0.707`) whenever possible. Floating-point drift across multiple transformations can push pieces out of valid configurations.
- **Do:** Validate rigid-body invariants (area, perimeter) independently of silhouette matching. A piece that matches the outline but has distorted geometry is invalid.
- **Do:** Use the Shapely library (Python) or equivalent computational geometry library for polygon operations. Manual intersection calculations are error-prone.
- **Do:** Separate the three validation stages — a syntax error is fundamentally different from a geometric distortion, and debugging requires knowing which stage failed.
- **Avoid:** Using pixel-level or raster-based comparison for geometric validation. Vector/polygon operations give exact answers; rasterization introduces aliasing errors.
- **Avoid:** Treating the parallelogram like other pieces — it is the only tangram piece that requires reflection (flip) in addition to rotation and translation, and omitting this degree of freedom makes many puzzles unsolvable.

## Error Handling

| Failure Mode | Symptom | Fix |
|---|---|---|
| Piece distortion (RGE) | Area or perimeter deviates >1% | Recompute transformation matrix; likely a non-rigid transform was applied (scaling or shear) |
| Piece overlap (PE) | Intersection area > epsilon between two pieces | Adjust translation/rotation of the offending piece; check for off-by-one rotation angles |
| Disconnected assembly | Union polygon is a MultiPolygon | Nudge pieces to close gaps; verify adjacency graph edges correspond to actual shared boundaries |
| Missing reflection | Parallelogram placement fails | Add reflection flag to transformation; the parallelogram is chiral and needs explicit flip handling |
| Floating-point accumulation | Validation passes locally but fails after chained transforms | Use `sympy` for exact arithmetic or increase precision tolerance; round only at output time |
| Invalid target outline | Self-intersecting polygon | Run `make_valid()` from Shapely; reorder vertices to ensure consistent winding |

## Limitations

- The standard tangram set is fixed at exactly 7 pieces with specific shapes. This framework does not generalize directly to arbitrary dissection puzzles with different piece counts or shapes without significant modification.
- Solving the inverse problem (finding valid placements from a silhouette) is computationally hard in general. For complex outlines, exhaustive search over rotation/translation/reflection space is impractical without heuristic pruning or constraint propagation.
- The TCE framework assumes 2D Euclidean geometry only. It does not extend to 3D spatial reasoning or curved surfaces.
- Human performance on the constructive task averaged ~73% with ~7 minutes per puzzle, indicating these problems are genuinely difficult. Automated solvers should not be expected to achieve 100% on complex silhouettes.
- Evaluation metrics (IoU, Hausdorff) require the target outline to be a simple polygon. Targets with holes or disconnected regions are not supported by the standard formulation.

## Reference

**Paper:** [TangramPuzzle: Evaluating Multimodal Large Language Models with Compositional Spatial Reasoning](https://arxiv.org/abs/2601.16520v1) — Liu et al., 2026. Look for: the TCE 5-tuple formal definition (Section 3), the three-stage validation pipeline (Section 4), and the finding that models prioritize silhouette matching over geometric constraint satisfaction (Section 5 analysis).