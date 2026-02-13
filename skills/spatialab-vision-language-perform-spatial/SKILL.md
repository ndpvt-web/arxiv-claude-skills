---
name: spatialab-vision-language-perform-spatial
description: >
  Evaluate and improve VLM spatial reasoning using the SpatiaLab benchmark taxonomy.
  Decomposes spatial tasks into 6 categories (positioning, depth, orientation, scale,
  navigation, 3D geometry) with 30 subcategories, applies multi-agent decomposition
  (SpatioXolver pattern), and structures prompts to maximize spatial accuracy.
  Trigger phrases: "evaluate spatial reasoning", "test VLM on spatial tasks",
  "spatial reasoning benchmark", "analyze image spatial relationships",
  "build spatial reasoning pipeline", "spatial QA evaluation"
---

# SpatiaLab: Structured Spatial Reasoning for Vision-Language Models

This skill enables Claude to build evaluation pipelines, prompting strategies, and multi-agent systems for spatial reasoning tasks over images. It applies the SpatiaLab taxonomy — 6 major spatial categories with 30 subcategories — to systematically decompose spatial questions, evaluate VLM outputs against human baselines, and orchestrate specialized agents for sub-tasks like depth ordering, occlusion inference, and 3D stability prediction. The core insight: spatial reasoning failures stem from weak perceptual grounding, not reasoning capacity, so effective systems must decompose perception before reasoning.

## When to Use

- When building an evaluation harness to test a VLM's spatial understanding on real-world images
- When a user asks to analyze spatial relationships in an image (e.g., "which object is in front?", "can you navigate from A to B?")
- When designing a multi-agent pipeline that decomposes spatial reasoning into perception and inference stages
- When comparing VLM spatial accuracy against human baselines using structured benchmarks
- When implementing Chain-of-Thought or self-reflection prompts specifically for spatial tasks
- When the user needs to classify a spatial question into a fine-grained category to select the right evaluation strategy
- When fine-tuning or prompt-engineering a VLM for spatial tasks and needing to identify which of the 30 subcategories are weakest

## Key Technique

**SpatiaLab Taxonomy.** The benchmark defines 6 major spatial reasoning categories, each with 5 subcategories (30 total). This taxonomy is the actionable framework: by classifying any spatial question into its subcategory, you can select the right prompting strategy, agent, or evaluation metric. The categories are: Relative Positioning (directional relations, proximity, alignment, betweenness, corner/angle), Depth & Occlusion (layering, partial occlusion, complete occlusion, transparency, reflections), Orientation (cardinal direction, rotation, facing, stacking, handedness), Size & Scale (relative size, perspective distortion, distance-size correlation, scale consistency, shadow projection), Spatial Navigation (path existence, obstacle avoidance, viewpoint visibility, spatial sequence, accessibility), and 3D Geometry (volume comparison, stability prediction, shape projection, containment, gravity effects).

**SpatioXolver Multi-Agent Decomposition.** Rather than prompting a single VLM monolithically, spatial reasoning improves when decomposed across 8 specialized agents: (1) Base Visual Analysis, (2) Object Segmentation, (3) Attribute Extraction, (4) Spatial Relation mapping, (5) Grouping & Symmetry, (6) Transformation Tracking, (7) Representation Standardization, and (8) Open-Ended Spatial Reasoning. Each agent produces structured intermediate outputs that feed the next, consolidating into a unified spatial representation. This approach yielded +36% improvement on orientation tasks in open-ended evaluation. However, agent pipelines can amplify errors when foundational perception is weak (depth/occlusion dropped -24%), so agents must include confidence gating.

**Critical Insight: Perception Before Reasoning.** CoT prompting alone provides minimal benefit for spatial tasks — and often reduces accuracy — because step-by-step reasoning amplifies flawed visual priors rather than correcting them. The effective pattern is: first extract and verify perceptual primitives (object locations, depths, sizes), then reason over the verified structure. Self-reflection helps modestly for MCQ geometry/depth but fails on open-ended tasks, confirming that reflection cannot substitute for missing perceptual grounding.

## Step-by-Step Workflow

1. **Classify the spatial question** into one of the 30 SpatiaLab subcategories. Parse the question for keywords: "left/right/above/below" → Relative Positioning; "in front/behind/hidden/occluded" → Depth & Occlusion; "facing/rotated/tilted" → Orientation; "bigger/smaller/scale" → Size & Scale; "path/route/reach" → Navigation; "volume/stable/fits inside" → 3D Geometry. Refine to the specific subcategory (e.g., "partial occlusion" vs. "complete occlusion").

2. **Extract perceptual primitives from the image** before reasoning. List all visible objects with bounding descriptions, estimate relative depths (foreground/midground/background), note occlusion relationships, and identify spatial reference frames. Output this as structured JSON.

3. **Design the evaluation format.** For accuracy benchmarking, use MCQ with 4 options (expect ~55% VLM ceiling vs ~88% human). For generative capability testing, use open-ended questions (expect ~40% VLM ceiling vs ~65% human). Always run both formats to measure the MCQ-to-open-ended performance gap (typically 10-25%).

4. **Construct category-specific prompts.** For each subcategory, prepend a perceptual grounding preamble that forces the model to enumerate objects and their spatial attributes before answering. For orientation tasks only, add CoT scaffolding ("First identify the facing direction, then determine the rotation angle, then answer"). For all other categories, use direct prompting with grounding — CoT hurts more than it helps.

5. **Implement multi-agent decomposition for complex queries.** Route the image through the SpatioXolver pipeline: Visual Analysis → Segmentation → Attribute Extraction → Spatial Relation → final Reasoning agent. Each agent outputs structured JSON. Use low temperature (0.1-0.2) for deterministic intermediate outputs. Gate agent outputs with confidence thresholds — if the Spatial Relation agent reports low confidence on depth ordering, flag the result rather than propagating errors.

6. **Build the evaluation harness.** For MCQ: extract the selected option letter/number, compare against ground truth, compute accuracy per subcategory. For open-ended: use an LLM judge (target Cohen's kappa >= 0.7 against human annotations). Validate the judge on a calibration set of ~50 examples before deploying.

7. **Compute per-category and aggregate metrics.** Report accuracy broken down by all 6 major categories and 30 subcategories. Identify the weakest subcategories — these indicate where the target VLM's spatial grounding fails. Compare against the SpatiaLab human baselines to quantify the gap.

8. **Iterate on the weakest categories.** For categories where accuracy is below 40%: try the multi-agent pipeline. For categories between 40-55%: try perceptual grounding prompts. For categories above 55%: the model's spatial grounding is adequate; focus efforts elsewhere.

9. **Validate with inter-annotator agreement.** If generating new spatial QA pairs, compute Fleiss' kappa across 3+ annotators (target >= 0.77). For LLM judge evaluation, compute Cohen's kappa against human majority vote (target >= 0.74).

10. **Document failure modes per category.** Record whether failures are perceptual (wrong object identification) or inferential (correct objects, wrong spatial conclusion). This distinction determines whether to improve visual grounding or reasoning scaffolding.

## Concrete Examples

**Example 1: Building a spatial reasoning evaluation pipeline**

User: "I want to evaluate how well GPT-4o handles spatial reasoning on my dataset of indoor scene images."

Approach:
1. Classify the user's target tasks into SpatiaLab subcategories — indoor scenes typically involve Relative Positioning, Depth & Occlusion, and Size & Scale.
2. Generate QA pairs in both MCQ (4 options) and open-ended format for each relevant subcategory.
3. Build the evaluation harness with structured output parsing and an LLM judge for open-ended responses.

Output:
```python
import json
from pathlib import Path

SPATIALAB_CATEGORIES = {
    "relative_positioning": [
        "directional_relations", "proximity_gradients", "alignment_patterns",
        "betweenness", "corner_angle_positioning"
    ],
    "depth_occlusion": [
        "layering_order", "partial_occlusion", "complete_occlusion",
        "transparency_effects", "reflective_surfaces"
    ],
    "orientation": [
        "cardinal_direction", "object_rotation", "facing_direction",
        "stacking_orientation", "tool_handedness"
    ],
    "size_scale": [
        "relative_size", "perspective_distortion", "distance_size_correlation",
        "scale_consistency", "shadow_size_projection"
    ],
    "spatial_navigation": [
        "pathway_existence", "obstacle_avoidance", "viewpoint_visibility",
        "spatial_sequence", "accessibility_constraints"
    ],
    "3d_geometry": [
        "volume_comparison", "stability_prediction", "shape_projection",
        "spatial_containment", "gravity_effects"
    ],
}

def build_grounded_prompt(image_description: str, question: str, category: str) -> str:
    """Prepend perceptual grounding to spatial questions."""
    grounding = (
        "First, list every visible object in the image with its approximate "
        "position (left/center/right, top/middle/bottom) and depth layer "
        "(foreground/midground/background). Note any occlusions. "
        "Then answer the following question.\n\n"
    )
    # Only add CoT for orientation tasks
    if category == "orientation":
        grounding += "Think step-by-step about directional alignment.\n\n"
    return grounding + f"Question: {question}"

def evaluate_mcq(model_answer: str, correct_answer: str) -> bool:
    return model_answer.strip().upper() == correct_answer.strip().upper()

def evaluate_open_ended(model_answer: str, reference: str, judge_model) -> dict:
    """Use LLM judge to score open-ended spatial answers."""
    judge_prompt = (
        f"Reference answer: {reference}\n"
        f"Model answer: {model_answer}\n\n"
        "Is the model's spatial reasoning correct? "
        "Score as 'correct', 'partially_correct', or 'incorrect'. "
        "Focus on spatial accuracy, not wording."
    )
    verdict = judge_model.generate(judge_prompt)
    return {"verdict": verdict, "model_answer": model_answer}
```

**Example 2: Multi-agent spatial decomposition for a complex scene**

User: "Given this warehouse image, determine whether a forklift can navigate from the loading dock to shelf B3 without collision."

Approach:
1. Classify as Spatial Navigation → Obstacle Avoidance + Pathway Existence.
2. Deploy SpatioXolver-style agent chain: Visual Analysis → Segmentation → Spatial Relation → Navigation Reasoning.
3. Each agent outputs structured JSON; gate on confidence before passing to the next stage.

Output:
```python
def spatioxolver_navigation(image, query: str) -> dict:
    """Multi-agent spatial navigation analysis."""

    # Agent 1: Base Visual Analysis
    visual_analysis = vlm_call(
        image=image,
        prompt="List all objects visible in this image. For each, state: "
               "name, bounding region, estimated depth layer, and whether "
               "it is movable or fixed.",
        temperature=0.1
    )

    # Agent 2: Spatial Relation Mapping
    spatial_relations = vlm_call(
        image=image,
        prompt=f"Given these objects: {visual_analysis}\n"
               "Map spatial relationships: adjacency, blocking, gaps, "
               "and clearances between objects. Output as a graph of "
               "spatial edges with estimated distances.",
        temperature=0.1
    )

    # Agent 3: Navigation Reasoning (only if spatial confidence > 0.7)
    if spatial_relations["confidence"] > 0.7:
        navigation = vlm_call(
            image=image,
            prompt=f"Spatial map: {spatial_relations}\n"
                   f"Query: {query}\n"
                   "Determine if a clear path exists. Identify obstacles "
                   "and whether the clearance is sufficient. Output: "
                   "path_exists (bool), obstacles (list), reasoning (str).",
            temperature=0.2
        )
    else:
        navigation = {"path_exists": None, "confidence": "low",
                       "note": "Spatial grounding insufficient for navigation inference"}

    return navigation
```

**Example 3: Diagnosing spatial reasoning weaknesses**

User: "My VLM gets 70% on object detection but fails on spatial questions. Help me figure out why."

Approach:
1. Run the model across all 30 SpatiaLab subcategories with a representative sample.
2. Compute per-subcategory accuracy to identify the failure pattern.
3. Classify failures as perceptual vs. inferential.

Output:
```
Diagnostic Results:
Category                    MCQ Acc.  Open-Ended Acc.  Gap
─────────────────────────────────────────────────────────────
Relative Positioning        62%       45%              17%
  - Directional Relations   71%       52%              19%
  - Betweenness             48%       31%              17%  ← WEAK
Depth & Occlusion           38%       22%              16%  ← WEAK
  - Layering Order          45%       28%              17%
  - Complete Occlusion      25%       12%              13%  ← CRITICAL
Orientation                 55%       40%              15%
Size & Scale                60%       48%              12%
Spatial Navigation          35%       18%              17%  ← WEAK
3D Geometry                 32%       15%              17%  ← CRITICAL

Diagnosis: Model has adequate 2D positional grounding (Relative
Positioning, Size & Scale) but fails on tasks requiring 3D
understanding (Depth, Navigation, 3D Geometry). Failures are
primarily perceptual — the model misidentifies depth ordering
in 72% of errors, not inferential. Recommendation: Apply
multi-agent decomposition for 3D categories; use direct
grounded prompting for 2D categories.
```

## Best Practices

- **Do:** Always extract perceptual primitives (objects, positions, depths) as structured data before attempting spatial reasoning. This grounds the model and prevents hallucinated spatial relationships.
- **Do:** Use MCQ and open-ended evaluation together. The MCQ-to-open-ended gap (typically 10-25%) reveals whether the model truly understands spatial relationships or is pattern-matching among choices.
- **Do:** Apply CoT prompting selectively — only for orientation/direction tasks. For depth, scale, navigation, and 3D geometry, CoT amplifies flawed visual priors and reduces accuracy.
- **Do:** Gate multi-agent pipeline outputs on confidence. If an intermediate agent reports low confidence on depth/occlusion, surface the uncertainty rather than propagating errors downstream.
- **Avoid:** Assuming fine-tuning on spatial data will generalize. SFT shows +11% MCQ gains but only +1% open-ended, and causes catastrophic forgetting of linguistic priors.
- **Avoid:** Using synthetic or LLM-generated spatial scenes for evaluation. Real-world images with visual noise, varied lighting, and complex occlusion are necessary to measure true spatial capability.

## Error Handling

- **Agent pipeline error propagation:** If the Segmentation or Spatial Relation agent produces incorrect object boundaries, all downstream agents inherit the error. Mitigate by running visual analysis at multiple scales and cross-checking object counts.
- **LLM judge disagreement:** When the judge's verdict conflicts with human annotation on >25% of a calibration set (Cohen's kappa < 0.7), recalibrate the judge prompt or switch to a stronger judge model.
- **Category misclassification:** A question about "which object is closer" could be Relative Positioning (proximity) or Depth & Occlusion (layering). When ambiguous, run evaluation under both categories and report the lower score.
- **Open-ended answer parsing:** Free-form spatial answers vary widely in phrasing. Normalize directional terms ("to the left" = "left of") and use semantic similarity rather than exact matching.

## Limitations

- The SpatiaLab benchmark caps at 1,400 QA pairs. For production evaluation, you may need to augment with domain-specific spatial questions following the same taxonomy.
- Multi-agent decomposition (SpatioXolver) improves orientation (+36%) but degrades depth/occlusion (-24%) and navigation (-12%). It is not a universal fix — use it selectively per category.
- Current best VLMs achieve ~55% MCQ accuracy vs. ~88% human baseline. No prompting or agent strategy closes this gap; architectural innovations in geometric encoding are needed for human-level performance.
- The evaluation framework assumes access to a VLM API capable of processing images. Text-only models cannot be evaluated.
- LLM-as-judge for open-ended spatial answers achieves ~0.74 Cohen's kappa — reliable but not perfect. High-stakes evaluations should include human review.

## Reference

**Paper:** [SpatiaLab: Can Vision-Language Models Perform Spatial Reasoning in the Wild?](https://arxiv.org/abs/2602.03916v1) (ICLR 2026)
**Dataset:** [HuggingFace: ciol-research/SpatiaLab](https://huggingface.co/datasets/ciol-research/SpatiaLab)
**Look for:** The 30-subcategory taxonomy (Table 1), SpatioXolver 8-agent architecture (Section 5.5), evaluation protocol with LLM judge calibration (Appendix D), and the per-category performance breakdown showing where each prompting strategy helps vs. hurts (Tables 3-6).