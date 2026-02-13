---
name: "zero-shot-product-attribute-labeling"
description: "Extract and classify product attributes from images using Vision-Language Models with structured prompts and a three-tier evaluation framework. Handles conditional attributes (e.g., fabric type only when garment is visible) by separating applicability detection from classification. Triggers: 'extract product attributes from images', 'label fashion attributes zero-shot', 'classify clothing attributes with VLM', 'build product catalog enrichment pipeline', 'detect garment properties from photos', 'evaluate attribute prediction accuracy'."
---

# Zero-Shot Product Attribute Labeling with Vision-Language Models

This skill enables Claude to build systems that extract structured product attributes from images using Vision-Language Models (VLMs) without any task-specific training data. The core technique, from Shukla & Sonalkar (WACV 2026 PRAW), decomposes multi-attribute prediction into three diagnostic tiers: overall performance, applicability detection (is this attribute even relevant to the image?), and fine-grained classification (what is the specific value?). This separation is critical because VLMs are strong classifiers (70.8% F1) but weak at detecting when an attribute does not apply (34.1% NA-F1) -- knowing this bottleneck changes how you architect the system.

## When to Use

- When the user needs to extract structured attributes (fabric, pattern, shape, color) from product images for catalog enrichment or search indexing
- When building a visual search or recommendation pipeline that requires multi-attribute tagging at scale
- When the user wants to evaluate or compare VLM performance on attribute extraction tasks
- When designing prompts for GPT-4V, Gemini, or similar VLMs to classify product properties from photos
- When handling conditional attributes where some properties only apply to certain product types (e.g., "outer fabric" only matters if an outer garment is visible)
- When the user asks to build a zero-shot labeling pipeline that avoids training data collection costs

## Key Technique

**The Three-Tier Decomposition.** Standard accuracy metrics hide where a VLM actually fails. This framework splits evaluation into: Tier 1 (macro-F1 across all classes including NA), Tier 2 (binary NA-vs-non-NA detection measuring applicability awareness), and Tier 3 (macro-F1 only on samples where the attribute is determinable, measuring classification quality). The insight: if Tier 3 is high but Tier 2 is low, the model classifies well but cannot tell when to abstain -- a common VLM failure mode.

**Structured Prompt Design.** The prompt casts the VLM as a domain expert ("You are an expert fashion attribute analyzer"), provides the full taxonomy of valid values per attribute, and demands JSON output with three fields per attribute: `value` (from a closed set), `reasoning` (chain-of-thought explanation), and `confidence` (calibrated 0-1 score). Requiring reasoning before the final value forces the model to evaluate applicability before committing to a class. The prompt explicitly instructs: "Be precise: Choose the most specific value that matches what you see."

**Conditional Attribute Handling.** Attributes are grouped into shape (12), fabric (3), and pattern (3) categories. Of 18 total attributes, 16 include NA as a valid class meaning "item doesn't exist or is not visible in the image." The model must first determine whether a garment layer (upper, lower, outer) is present before classifying its fabric or pattern. This two-step reasoning -- detect, then classify -- is embedded in the prompt structure rather than requiring separate model calls.

## Step-by-Step Workflow

1. **Define the attribute taxonomy.** List every attribute, its category (shape/fabric/pattern), and the exhaustive set of valid values including NA where applicable. Each attribute must have a closed label space -- open-ended values break structured evaluation.

2. **Construct the structured VLM prompt.** Write a system prompt that (a) establishes the expert role, (b) lists all attributes with their valid values, (c) instructs the model to output JSON grouped by attribute category, and (d) requires `value`, `reasoning`, and `confidence` fields per attribute. Include explicit guidance on when to predict NA: "If the garment layer is not visible or does not exist in the image, predict NA."

3. **Format the JSON output schema.** Define the exact structure the VLM must return:
   ```json
   {
     "shape_attributes": {
       "sleeve_length": {"value": "...", "reasoning": "...", "confidence": 0.0},
       "neckline": {"value": "...", "reasoning": "...", "confidence": 0.0}
     },
     "fabric_attributes": {
       "upper_fabric": {"value": "...", "reasoning": "...", "confidence": 0.0}
     },
     "pattern_attributes": {
       "upper_pattern": {"value": "...", "reasoning": "...", "confidence": 0.0}
     }
   }
   ```

4. **Send images to the VLM with the structured prompt.** Pass each product image alongside the system prompt. Use the model's vision capability (e.g., OpenAI's `gpt-4o` with image input, or Google's `gemini-2.5-pro`). Parse the returned JSON, validating each value against the closed label set.

5. **Post-process predictions.** Reject any value not in the valid set for that attribute. Apply confidence thresholds: the paper uses calibrated bands (1.0 = certain, 0.8-0.9 = very confident, 0.5-0.7 = moderate, below 0.5 = uncertain). Flag low-confidence predictions for human review.

6. **Evaluate with the three-tier framework.** Compute Tier 1 macro-F1 across all classes (including NA) on the full test set. For Tier 2, binarize predictions (NA=1, non-NA=0) and compute NA-F1 to measure applicability detection. For Tier 3, filter to samples where ground truth is not NA and compute macro-F1 on remaining predictions.

7. **Diagnose failure modes.** If Tier 1 is low but Tier 3 is high, the bottleneck is NA detection -- add explicit applicability checks or a pre-classification filter. If Tier 3 is low, the model struggles with fine-grained distinctions -- refine the prompt with more detailed value descriptions or few-shot examples for ambiguous classes.

8. **Select the cost-appropriate model tier.** Efficient models (GPT-4o-mini, Gemini Flash) achieve over 90% of flagship performance. Use flagship models only for ambiguous cases or as a second-pass validator. Build a cascade: fast model first, escalate low-confidence predictions to a stronger model.

9. **Iterate on the prompt using tier-specific metrics.** Track per-attribute Tier 2 and Tier 3 scores independently. Focus prompt improvements on the weakest attributes rather than optimizing globally.

## Concrete Examples

**Example 1: Fashion Catalog Enrichment Pipeline**

User: "Build a script that takes product images and extracts fabric, pattern, and shape attributes using GPT-4o, outputting structured JSON for our catalog database."

Approach:
1. Define the attribute schema with valid values per attribute
2. Build the structured prompt with role, taxonomy, and output format
3. Call the OpenAI API with vision for each image
4. Parse and validate the JSON output
5. Write results to the catalog database

Output (Python implementation sketch):
```python
import openai, json

ATTRIBUTE_SCHEMA = {
    "shape_attributes": {
        "sleeve_length": ["sleeveless", "short-sleeve", "medium-sleeve", "long-sleeve"],
        "neckline": ["V-shape", "square", "round", "standing", "lapel", "suspenders", "NA"],
        "lower_clothing_length": ["three-point", "medium short", "three-quarter", "long", "NA"],
    },
    "fabric_attributes": {
        "upper_fabric": ["denim", "cotton", "leather", "furry", "knitted", "chiffon", "other", "NA"],
        "lower_fabric": ["denim", "cotton", "leather", "furry", "knitted", "chiffon", "other", "NA"],
        "outer_fabric": ["denim", "cotton", "leather", "furry", "knitted", "chiffon", "other", "NA"],
    },
    "pattern_attributes": {
        "upper_pattern": ["floral", "graphic", "striped", "pure color", "lattice", "other", "color block", "NA"],
        "lower_pattern": ["floral", "graphic", "striped", "pure color", "lattice", "other", "color block", "NA"],
        "outer_pattern": ["floral", "graphic", "striped", "pure color", "lattice", "other", "color block", "NA"],
    },
}

SYSTEM_PROMPT = """You are an expert fashion attribute analyzer. Analyze the clothing in this image
and predict the following attributes. For each attribute, return a JSON object with:
- "value": one of the valid values listed below (use exactly these strings)
- "reasoning": brief explanation of what you observe
- "confidence": float 0-1 (1.0=certain, 0.8-0.9=very confident, 0.5-0.7=moderate, <0.5=uncertain)

CRITICAL: If a garment layer is not visible or does not exist in the image, predict "NA".
For outer_fabric/outer_pattern: only classify if an outer garment (jacket, coat, cardigan) is visible.
Be precise: choose the most specific value that matches what you see.

Valid values per attribute:
""" + json.dumps(ATTRIBUTE_SCHEMA, indent=2) + """

Return a single JSON object grouped by category (shape_attributes, fabric_attributes, pattern_attributes)."""

def extract_attributes(image_path: str) -> dict:
    import base64
    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()

    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": [
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}},
                {"type": "text", "text": "Analyze this product image."}
            ]}
        ],
        response_format={"type": "json_object"},
    )
    result = json.loads(response.choices[0].message.content)
    return validate_and_clean(result, ATTRIBUTE_SCHEMA)

def validate_and_clean(result: dict, schema: dict) -> dict:
    """Reject predictions not in the valid label set."""
    cleaned = {}
    for category, attrs in schema.items():
        cleaned[category] = {}
        for attr, valid_values in attrs.items():
            pred = result.get(category, {}).get(attr, {})
            value = pred.get("value", "NA")
            if value not in valid_values:
                value = "NA"  # fallback for invalid predictions
                pred["confidence"] = 0.0
            cleaned[category][attr] = {**pred, "value": value}
    return cleaned
```

**Example 2: Three-Tier Evaluation Script**

User: "I have VLM predictions and ground truth labels. Help me evaluate using the three-tier framework from the paper."

Approach:
1. Load predictions and ground truth
2. Compute Tier 1 (full macro-F1), Tier 2 (NA detection F1), Tier 3 (classification-only F1)
3. Report per-attribute diagnostics

Output:
```python
from sklearn.metrics import f1_score
import numpy as np

def three_tier_evaluation(predictions: dict, ground_truth: dict, attributes: list[str]):
    """
    predictions/ground_truth: {image_id: {attr_name: predicted_value, ...}, ...}
    """
    results = {}
    for attr in attributes:
        y_true = [ground_truth[img][attr] for img in ground_truth]
        y_pred = [predictions[img].get(attr, "NA") for img in ground_truth]

        # Tier 1: Full macro-F1 (all classes including NA)
        tier1 = f1_score(y_true, y_pred, average="macro", zero_division=0)

        # Tier 2: NA detection (binary: NA=1, non-NA=0)
        y_true_bin = [1 if v == "NA" else 0 for v in y_true]
        y_pred_bin = [1 if v == "NA" else 0 for v in y_pred]
        tier2 = f1_score(y_true_bin, y_pred_bin, pos_label=1, zero_division=0)

        # Tier 3: Classification quality (exclude NA ground-truth samples)
        mask = [i for i, v in enumerate(y_true) if v != "NA"]
        if mask:
            y_true_t3 = [y_true[i] for i in mask]
            y_pred_t3 = [y_pred[i] for i in mask]
            tier3 = f1_score(y_true_t3, y_pred_t3, average="macro", zero_division=0)
        else:
            tier3 = None  # attribute is always NA in this dataset

        results[attr] = {"tier1_f1": tier1, "tier2_na_f1": tier2, "tier3_cls_f1": tier3}

    # Aggregate
    avg = lambda key: np.mean([v[key] for v in results.values() if v[key] is not None])
    results["_aggregate"] = {
        "tier1_f1": avg("tier1_f1"),
        "tier2_na_f1": avg("tier2_na_f1"),
        "tier3_cls_f1": avg("tier3_cls_f1"),
    }
    return results
```

**Example 3: Cost-Optimized Cascading Pipeline**

User: "I need to label 100K product images but want to minimize API costs. How should I architect this?"

Approach:
1. Run all images through a cheap model (Gemini Flash or GPT-4o-mini) first
2. Use confidence scores to triage: high-confidence predictions are accepted, low-confidence ones escalate
3. Send only low-confidence predictions to a flagship model (Gemini Pro or GPT-4o)

Output:
```python
CONFIDENCE_THRESHOLD = 0.75

def cascading_pipeline(image_paths: list[str]):
    # Stage 1: Cheap model pass
    stage1_results = {}
    for path in image_paths:
        result = extract_attributes(path, model="gpt-4o-mini")  # ~10x cheaper
        stage1_results[path] = result

    # Stage 2: Identify low-confidence attributes per image
    escalation_queue = {}
    for path, result in stage1_results.items():
        low_conf_attrs = []
        for category in result.values():
            for attr, pred in category.items():
                if pred["confidence"] < CONFIDENCE_THRESHOLD:
                    low_conf_attrs.append(attr)
        if low_conf_attrs:
            escalation_queue[path] = low_conf_attrs

    # Stage 3: Re-predict only low-confidence attributes with flagship model
    for path, attrs in escalation_queue.items():
        flagship_result = extract_attributes(path, model="gpt-4o")
        # Merge: replace only the escalated attributes
        for category in stage1_results[path]:
            for attr in stage1_results[path][category]:
                if attr in attrs:
                    stage1_results[path][category][attr] = flagship_result[category][attr]

    return stage1_results
    # Typical escalation rate: 15-25% of images, saving 75-85% on flagship costs
```

## Best Practices

- **Do:** Always include NA as a valid prediction class for conditional attributes. The biggest VLM failure mode is hallucinating attribute values for items that aren't visible.
- **Do:** Require `reasoning` in the output schema. Chain-of-thought before the value prediction improves accuracy and makes errors debuggable.
- **Do:** Validate every predicted value against the closed label set. VLMs frequently return synonyms ("short sleeves" instead of "short-sleeve") or invented values.
- **Do:** Evaluate Tier 2 and Tier 3 separately per attribute. Aggregate metrics hide that some attributes (e.g., `socks`, `hat`) have much higher NA rates and different error profiles.
- **Avoid:** Treating NA prediction as a simple "I don't know." NA means the attribute category is structurally inapplicable (no outer garment exists), not that the model is uncertain.
- **Avoid:** Using open-ended prompts like "describe this clothing." Constrained label sets with explicit valid values consistently outperform free-form descriptions for structured catalog tasks.

## Error Handling

- **Invalid JSON from VLM:** Wrap the API call in a retry with explicit JSON mode (`response_format={"type": "json_object"}` for OpenAI). If the model still returns malformed JSON, fall back to regex extraction of key-value pairs.
- **Values outside valid set:** Map common synonyms (e.g., "short sleeves" -> "short-sleeve", "plain" -> "pure color") with a normalization dictionary. Default unmapped values to NA with confidence 0.
- **Missing attributes in response:** If the VLM omits an attribute entirely, treat it as NA with confidence 0 and flag for review.
- **Confidence miscalibration:** VLMs tend to be overconfident. Calibrate thresholds empirically on a validation set rather than trusting raw confidence values. The paper's efficient models showed confidence distributions skewed toward 0.8-1.0 even on incorrect predictions.
- **Rate limiting at scale:** Batch images and use async API calls. For 100K+ images, implement checkpointing so failures don't require reprocessing the entire set.

## Limitations

- **NA detection remains weak.** Even the best VLMs achieve only 34.1% NA-F1. For production systems requiring high NA precision (e.g., preventing hallucinated attributes), add a dedicated garment detection stage before attribute classification.
- **Domain-specific taxonomies.** The paper's 18-attribute schema is fashion-specific. Applying this to other product domains (electronics, furniture) requires building a new taxonomy with domain-appropriate NA conditions.
- **Single-image limitation.** The approach analyzes individual product images. It does not aggregate attributes across multiple views of the same product, which could improve accuracy for occluded attributes.
- **Cost at scale.** Even with efficient models, processing millions of images through VLM APIs is expensive compared to trained classifiers. For very high-volume, low-attribute-count tasks, a fine-tuned CLIP-based classifier may be more cost-effective (though at 3x lower accuracy per the paper's findings).
- **Evaluation requires labeled data.** The three-tier framework needs ground truth with explicit NA annotations, which most product datasets lack. Building the evaluation set is a prerequisite investment.

## Reference

[Zero-Shot Product Attribute Labeling with Vision-Language Models: A Three-Tier Evaluation Framework](https://arxiv.org/abs/2601.15711v1) -- Shukla & Sonalkar, WACV 2026 PRAW Workshop. Key sections: Section 3 for the three-tier evaluation definitions and formulas, Section 4 for the prompt template and attribute taxonomy, Table 2 for per-model tier scores.