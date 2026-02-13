---
name: "vectra-metric-dataset-visual"
description: "Assess visual quality of translated product images using Vectra's 14-dimension scoring framework. Use when: 'evaluate translated image quality', 'score e-commerce product rendering', 'assess in-image translation defects', 'build IIMT quality pipeline', 'rate visual rendering of translated text on images', 'detect text hallucination in product photos'."
---

# Vectra: Visual Quality Assessment for E-Commerce In-Image Translation

This skill enables Claude to implement and apply the Vectra framework for assessing visual quality in e-commerce In-Image Machine Translation (IIMT). Vectra decomposes visual quality into 14 interpretable dimensions across textual and scene categories, uses a spatially-aware Defect Area Ratio (DAR) to quantify defect severity, and combines scores via multiplicative aggregation that treats accuracy failures as non-compensatory. This approach replaces opaque reference-based metrics (SSIM, FID) and underspecified model-as-judge prompts with a structured, explainable quality assessment pipeline grounded in e-commerce domain knowledge.

## When to Use

- When the user asks to build a quality evaluation pipeline for translated product images or any in-image text rendering system
- When the user needs to score or rank visual quality of images where text has been overlaid, translated, or edited programmatically
- When the user wants to detect and categorize defects in translated e-commerce listings (hallucinated text, missing translations, style inconsistencies)
- When the user is constructing annotation guidelines or rubrics for human evaluation of image translation quality
- When the user needs a structured prompt template to evaluate visual rendering quality using a multimodal LLM (GPT-4o, Gemini, Qwen-VL, etc.)
- When the user wants to build a reward model or preference dataset for aligning image translation systems
- When the user asks to implement DAR-based spatial defect quantification for any image quality assessment task

## Key Technique

Vectra's core insight is that visual quality in translated product images cannot be captured by a single score or pixel-level similarity metric. Instead, it decomposes quality into **14 dimensions** organized in a two-level taxonomy. **Textual Visual Quality** covers 8 dimensions: Text Size, Text Color, Text Position, Font Style, Layout, Pixel Clarity Consistency (all style), plus Text Hallucination and Text Omission (accuracy). **Scene Visual Quality** covers 6 dimensions: Scene Size, Scene Color, Element Position, Pixel Clarity Consistency (style), plus Scene Hallucination and Scene Omission (accuracy). Each dimension is scored on a 3-point ordinal scale (1=Poor, 2=Fair, 3=Excellent) anchored by the Defect Area Ratio.

The **Defect Area Ratio (DAR)** measures what fraction of the relevant content area is affected by a defect. The scoring rule is: score 3 if DAR is approximately 0, score 2 if 0 < DAR <= 0.3, and score 1 if DAR > 0.3. The threshold of 0.3 was empirically calibrated: rejection rates stay below 40% for DAR < 0.3 and surge past 90% for DAR >= 0.3. This spatial grounding eliminates the ambiguity of subjective "good/bad" labels by tying scores to observable defect coverage.

The **final Vectra Score** uses multiplicative aggregation: `Score = 100 * phi(mean_accuracy) * phi(mean_style)`, where phi normalizes the [1,3] range to [0,1]. This is deliberately non-compensatory -- a catastrophic accuracy failure (hallucinated product spec, missing safety text) drives the score toward zero regardless of how good the styling looks. This reflects e-commerce reality: a mistranslated product feature can violate consumer protection regulations, and no amount of pretty typography compensates for it.

## Step-by-Step Workflow

1. **Define the dimension taxonomy.** Create a structured schema covering all 14 dimensions, split into Textual Visual Quality (8 dimensions) and Scene Visual Quality (6 dimensions), each tagged as either "style" or "accuracy" type. Store this as a JSON or Python dictionary that will drive downstream scoring.

2. **Build the DAR scoring function.** Implement the 3-point ordinal scoring rule: if the defect area ratio is ~0, return 3; if 0 < DAR <= 0.3, return 2; if DAR > 0.3, return 1. For automated pipelines, compute DAR as `defect_pixels / total_content_pixels` using bounding-box or segmentation masks. For MLLM-based pipelines, instruct the model to estimate DAR visually.

3. **Construct the evaluation prompt template.** Build a structured prompt that: (a) presents the image, (b) lists all 14 dimension definitions with DAR-anchored rubrics, (c) requires per-dimension reasoning in the pattern CONTENT -> ISSUE -> POSITION -> EFFECT (DAR estimate) -> SCORE, and (d) mandates structured output (XML or JSON) with both reasoning and numeric scores per dimension.

4. **Collect per-dimension scores.** For each image, run the evaluation (via human annotators or MLLM) to produce 14 individual scores. If using multiple annotators, resolve disagreements by statistical mode with lowest-value tiebreaking (conservative bias favoring defect detection).

5. **Compute the aggregate Vectra Score.** Separate accuracy dimensions from style dimensions. Compute the mean score for each group. Normalize each mean from the [1,3] range to [0,1] via `phi(x) = (x - 1) / 2`. Multiply the two normalized values and scale by 100: `final_score = 100 * phi(mean_acc) * phi(mean_sty)`.

6. **Generate diagnostic reasoning.** For each image, produce a natural-language explanation citing the specific dimensions that failed, the estimated DAR, and the spatial location of defects. This transforms the score from an opaque number into an actionable diagnosis.

7. **Rank and compare systems.** To evaluate multiple IIMT systems, compute Vectra Scores across a benchmark set, then measure system-level agreement using Kendall's tau (rank correlation) and instance-level accuracy using Pearson's r against human ground truth.

8. **Build training data for a scoring model (optional).** If fine-tuning a smaller MLLM: (a) generate 30K reasoning annotations using a strong MLLM (e.g., Gemini-2.5-Pro) with the structured prompt, (b) collect 2.5-3.5K expert preference pairs for alignment, (c) fine-tune with supervised learning on the reasoning data, then align with GSPO using format + preference rewards.

9. **Balance the training distribution (optional).** Use quartile-based thresholding to identify underrepresented dimension-score pairs among the 42 possible combinations (14 dimensions x 3 levels). Augment rare pairs via synthetic generation with verification; prune overrepresented samples by inverse-frequency rarity scoring.

10. **Validate with inter-rater reliability.** Measure Krippendorff's alpha across annotators. Vectra's DAR-anchored rubrics should yield alpha > 0.8; if not, recalibrate the DAR threshold or clarify dimension definitions for your specific domain.

## Concrete Examples

**Example 1: Building a Vectra scoring function in Python**

User: "I need to evaluate the visual quality of translated product images. Can you implement the Vectra scoring system?"

Approach:
1. Define the 14-dimension taxonomy as a data structure
2. Implement the DAR-based scoring rule
3. Implement the multiplicative aggregation formula
4. Add diagnostic output

Output:
```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class DimensionType(Enum):
    ACCURACY = "accuracy"
    STYLE = "style"

class Category(Enum):
    TEXTUAL = "textual"
    SCENE = "scene"

VECTRA_DIMENSIONS = {
    # Textual Visual Quality - Style
    "text_size": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Translated text size consistency with original layout"},
    "text_color": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Color harmony of rendered text with background"},
    "text_position": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Spatial alignment of translated text within design regions"},
    "font_style": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Font choice consistency with brand and product context"},
    "text_layout": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Line breaking, spacing, and text block arrangement"},
    "text_pixel_clarity": {"category": Category.TEXTUAL, "type": DimensionType.STYLE,
        "description": "Sharpness and rendering quality of text pixels"},
    # Textual Visual Quality - Accuracy
    "text_hallucination": {"category": Category.TEXTUAL, "type": DimensionType.ACCURACY,
        "description": "Fabricated text content not in the original image"},
    "text_omission": {"category": Category.TEXTUAL, "type": DimensionType.ACCURACY,
        "description": "Original text content missing from the translation"},
    # Scene Visual Quality - Style
    "scene_size": {"category": Category.SCENE, "type": DimensionType.STYLE,
        "description": "Proportional consistency of scene elements after editing"},
    "scene_color": {"category": Category.SCENE, "type": DimensionType.STYLE,
        "description": "Color consistency of inpainted or modified scene regions"},
    "element_position": {"category": Category.SCENE, "type": DimensionType.STYLE,
        "description": "Spatial coherence of scene objects after text replacement"},
    "scene_pixel_clarity": {"category": Category.SCENE, "type": DimensionType.STYLE,
        "description": "Visual clarity of scene regions affected by editing"},
    # Scene Visual Quality - Accuracy
    "scene_hallucination": {"category": Category.SCENE, "type": DimensionType.ACCURACY,
        "description": "Fabricated visual elements introduced during editing"},
    "scene_omission": {"category": Category.SCENE, "type": DimensionType.ACCURACY,
        "description": "Original scene elements lost during editing"},
}

DAR_THRESHOLD = 0.3

def score_from_dar(dar: float) -> int:
    """Convert Defect Area Ratio to 3-point ordinal score."""
    if dar <= 0.01:  # approximately zero
        return 3  # Excellent
    elif dar <= DAR_THRESHOLD:
        return 2  # Fair
    else:
        return 1  # Poor

def compute_vectra_score(dimension_scores: dict[str, int]) -> dict:
    """Compute aggregate Vectra Score from per-dimension scores (1-3 each)."""
    acc_scores, sty_scores = [], []
    for dim_name, score in dimension_scores.items():
        dim_info = VECTRA_DIMENSIONS[dim_name]
        if dim_info["type"] == DimensionType.ACCURACY:
            acc_scores.append(score)
        else:
            sty_scores.append(score)

    mean_acc = sum(acc_scores) / len(acc_scores)
    mean_sty = sum(sty_scores) / len(sty_scores)

    # Normalize [1,3] -> [0,1]
    phi_acc = (mean_acc - 1) / 2
    phi_sty = (mean_sty - 1) / 2

    # Multiplicative aggregation (non-compensatory)
    final_score = 100 * phi_acc * phi_sty

    return {
        "vectra_score": round(final_score, 1),
        "accuracy_mean": round(mean_acc, 2),
        "style_mean": round(mean_sty, 2),
        "phi_accuracy": round(phi_acc, 3),
        "phi_style": round(phi_sty, 3),
        "per_dimension": dimension_scores,
    }
```

**Example 2: Crafting an MLLM evaluation prompt**

User: "I want to use GPT-4o to score my translated product images using the Vectra framework. Can you write the evaluation prompt?"

Approach:
1. Structure the prompt with dimension definitions and DAR rubrics
2. Enforce the CONTENT -> ISSUE -> POSITION -> EFFECT -> SCORE reasoning chain
3. Require structured XML output for parseability

Output:
```text
You are a visual quality assessor for e-commerce in-image machine translation.

Given a translated product image, evaluate it across 14 dimensions. For each
dimension, follow this reasoning chain:
  CONTENT: What text or scene element does this dimension cover?
  ISSUE: Is there a visible defect? Describe it specifically.
  POSITION: Where in the image is the defect located?
  EFFECT: Estimate the Defect Area Ratio (DAR) -- what fraction of the
          relevant content area is affected by the defect?
  SCORE: Assign 3 (Excellent, DAR~0), 2 (Fair, 0<DAR<=0.3),
         or 1 (Poor, DAR>0.3).

## Textual Visual Quality -- Style Dimensions
- text_size: Is the translated text size consistent with the original layout?
- text_color: Does the text color harmonize with the background?
- text_position: Is translated text properly aligned within design regions?
- font_style: Is the font appropriate for the brand/product context?
- text_layout: Are line breaks, spacing, and text blocks well arranged?
- text_pixel_clarity: Is the rendered text sharp and free of artifacts?

## Textual Visual Quality -- Accuracy Dimensions
- text_hallucination: Does the translation contain fabricated content?
- text_omission: Is any original text content missing from the translation?

## Scene Visual Quality -- Style Dimensions
- scene_size: Are scene elements proportionally consistent after editing?
- scene_color: Are inpainted/modified scene regions color-consistent?
- element_position: Are scene objects spatially coherent after text replacement?
- scene_pixel_clarity: Are edited scene regions visually clear?

## Scene Visual Quality -- Accuracy Dimensions
- scene_hallucination: Were new visual elements incorrectly introduced?
- scene_omission: Were original scene elements lost during editing?

Respond in this XML format for EACH dimension:
<dimension name="[dim_name]">
  <content>[what this dimension covers in this image]</content>
  <issue>[defect description or "none"]</issue>
  <position>[spatial location or "N/A"]</position>
  <dar>[estimated DAR as decimal, e.g. 0.15]</dar>
  <score>[1, 2, or 3]</score>
</dimension>

After all 14 dimensions, provide:
<summary>
  <vectra_score>[computed as 100 * phi(mean_accuracy) * phi(mean_style)]</vectra_score>
  <diagnosis>[1-2 sentence summary of primary quality issues]</diagnosis>
</summary>
```

**Example 3: Building a benchmark evaluation pipeline**

User: "I have 500 translated product images from two different IIMT systems. Help me compare them using Vectra."

Approach:
1. Score all images from both systems using the evaluation prompt
2. Parse structured outputs into per-dimension and aggregate scores
3. Compute system-level ranking metrics

Output:
```python
import json
import numpy as np
from scipy.stats import kendalltau, pearsonr

def evaluate_system_pair(scores_a: list[dict], scores_b: list[dict],
                         human_rankings: list[int]) -> dict:
    """Compare two IIMT systems using Vectra metrics.

    Args:
        scores_a: List of Vectra score dicts for system A (one per image)
        scores_b: List of Vectra score dicts for system B (one per image)
        human_rankings: Human preference per image (1=A better, 2=B better)
    """
    vectra_a = [s["vectra_score"] for s in scores_a]
    vectra_b = [s["vectra_score"] for s in scores_b]

    # Instance-level: which system scored higher per image
    auto_rankings = [1 if a >= b else 2 for a, b in zip(vectra_a, vectra_b)]

    # System-level means
    mean_a, mean_b = np.mean(vectra_a), np.mean(vectra_b)

    # Correlation with human judgments
    # Flatten to pairwise preference agreement
    agreement = sum(a == h for a, h in zip(auto_rankings, human_rankings))
    accuracy = agreement / len(human_rankings)

    # Rank correlation across all images (both systems pooled)
    all_auto = vectra_a + vectra_b
    all_human = human_rankings  # extend as needed for full ranking

    tau, tau_p = kendalltau(vectra_a, vectra_b)

    # Per-dimension diagnostics: find systematically weak dimensions
    dim_names = list(scores_a[0]["per_dimension"].keys())
    weak_dims_a = {}
    for dim in dim_names:
        dim_scores = [s["per_dimension"][dim] for s in scores_a]
        weak_dims_a[dim] = round(np.mean(dim_scores), 2)

    return {
        "system_a_mean": round(mean_a, 1),
        "system_b_mean": round(mean_b, 1),
        "pairwise_agreement_with_humans": round(accuracy, 3),
        "kendall_tau": round(tau, 3),
        "system_a_dimension_means": weak_dims_a,
    }
```

## Best Practices

- **Do** use the multiplicative aggregation formula. Accuracy and style scores must be multiplied, not averaged. This ensures that a hallucinated product name (accuracy=1) cannot be offset by good font choice (style=3). The non-compensatory property is central to the framework's validity.

- **Do** anchor every score to the DAR threshold of 0.3. When training annotators or prompting MLLMs, always specify this threshold explicitly. Unanchored "rate 1-3" instructions produce unreliable scores (Krippendorff's alpha drops from 0.86 to 0.44 without DAR grounding).

- **Do** require the CONTENT -> ISSUE -> POSITION -> EFFECT -> SCORE reasoning chain. Skipping intermediate reasoning degrades scoring accuracy. The chain forces the evaluator to identify the specific defect before scoring.

- **Do** separate accuracy from style dimensions in analysis. Report both sub-scores alongside the aggregate. A Vectra Score of 40 could mean "decent accuracy, poor style" or "catastrophic hallucination, great style" -- the sub-scores disambiguate.

- **Avoid** using weighted averaging or learned weights across dimensions. The paper found that simple mean-then-multiply outperforms more complex weighting schemes. Added complexity here reduces interpretability without improving correlation with human judgments.

- **Avoid** treating DAR as a precise pixel-level computation when using MLLM-based evaluation. Human annotators and MLLMs estimate DAR visually as a rough proportion. Demanding pixel-exact DAR values adds annotation cost without improving inter-rater agreement.

## Error Handling

- **Dimension score out of range:** If any dimension score is not in {1, 2, 3}, clamp it and log a warning. The aggregation formula only works correctly on the [1,3] range.
- **Missing dimensions in MLLM output:** If the MLLM omits a dimension, retry with an explicit reminder listing the missing dimensions. Do not impute scores -- missing dimensions usually indicate the model was confused by the image.
- **All scores are 3 (ceiling effect):** If an MLLM rates every dimension as 3 for most images, the prompt likely lacks sufficient rubric detail. Add concrete examples of Fair (score=2) cases to the prompt.
- **Low inter-rater agreement:** If Krippendorff's alpha falls below 0.67, review whether annotators are applying DAR consistently. The most common failure is inconsistent estimation of "what counts as the content area" for the denominator.
- **XML/JSON parse failures:** Wrap output parsing in try/catch and fall back to regex extraction of score values. MLLM outputs occasionally include extra text outside the requested structure.

## Limitations

- **Domain specificity:** The 14 dimensions and DAR threshold (0.3) were calibrated for e-commerce product images. Applying Vectra to other domains (medical imaging, document translation, game UI localization) requires recalibrating the threshold and potentially redefining dimensions.
- **Reference-free trade-off:** Vectra intentionally operates without a reference image, which makes it deployable at scale but means it cannot detect subtle semantic translation errors that require source-target comparison.
- **3-point scale granularity:** The coarse 1-3 scale is robust for annotation but may not differentiate fine quality differences between strong systems. For close-performing systems, the 2K benchmark size may be insufficient for statistical significance.
- **MLLM evaluator dependency:** Prompt-based evaluation quality depends heavily on the MLLM's visual understanding. Smaller or weaker vision-language models may not reliably estimate DAR or detect subtle defects like font style mismatches.
- **Dataset and model not yet public:** As of the paper's publication, the Vectra dataset and fine-tuned model are pending release upon acceptance. The framework described here is based on the methodology and can be implemented using the prompt templates and scoring formulas.

## Reference

**Paper:** [Vectra: A New Metric, Dataset, and Model for Visual Quality Assessment in E-Commerce In-Image Machine Translation](https://arxiv.org/abs/2602.07014v1) (Wu et al., 2026). Look for: Table 2 (full dimension taxonomy), Table 6 (annotation guidelines), Table 7 (MLLM prompt template), and Section 4.2 (DAR calibration experiments showing the 0.3 threshold derivation).