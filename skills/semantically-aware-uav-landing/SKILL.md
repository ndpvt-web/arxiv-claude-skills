---
name: "semantically-aware-uav-landing"
description: >
  Build coarse-to-fine UAV emergency landing site assessment systems that combine
  semantic segmentation pre-screening with multimodal LLM reasoning and POI data fusion.
  Trigger phrases: "UAV landing site assessment", "drone emergency landing", "remote sensing
  landing safety", "semantic landing zone analysis", "aerial image hazard detection",
  "landing site risk scoring pipeline"
---

# Semantically Aware UAV Landing Site Assessment

This skill enables Claude to implement the coarse-to-fine pipeline from Hua et al. (2026) for assessing UAV emergency landing sites from remote sensing imagery. The core technique chains a lightweight semantic segmentation module (DeepLabV3+ via MMSegmentation) for rapid terrain pre-screening, a spatial density search with radially decaying convolution for candidate proposal, and a multimodal LLM reasoning agent that fuses satellite image patches with Point-of-Interest (POI) data and regulatory constraints to detect subtle hazards like crowds, temporary structures, and time-dependent risks. This replaces brittle geometric-only approaches with context-aware, interpretable safety verdicts.

## When to Use

- When the user asks to build a pipeline that identifies safe emergency landing zones from satellite or aerial imagery
- When the user needs to integrate semantic segmentation output with LLM-based visual reasoning for terrain assessment
- When the user wants to fuse OpenStreetMap/Amap POI data with image features for spatial risk scoring
- When the user is building a propose-and-verify candidate search with tabu-style suppression over segmentation maps
- When the user needs to evaluate landing site safety using JARUS SORA compliance criteria
- When the user asks to construct a benchmark dataset for landing site selection with multi-source annotations
- When the user wants to compare geometric baselines against MLLM-based hazard detection on remote sensing data

## Key Technique

The framework operates as a three-stage coarse-to-fine hierarchy. **Stage 1 (Semantic Filtering)** runs DeepLabV3+ with Atrous Spatial Pyramid Pooling (ASPP) on remote sensing tiles to classify every pixel into terrain categories (impervious surfaces, barren land, vegetation, water, buildings). Only pixels classified as candidate-safe classes (e.g., "Background/Impervious Surfaces" for Potsdam data at 0.05 m/px, "Background/Barren" for lower-resolution composites) pass through. This eliminates obviously unsuitable terrain — water bodies, dense vegetation, rooftops — before any expensive reasoning.

**Stage 2 (Visual Verification)** applies a spatial density search using a radially decaying convolution kernel: `K[i,j] = 1 - ((i-d)^2 + (j-d)^2) / (2d^2)`, where `d` is the kernel diameter matched to the UAV footprint. The kernel scores candidate patches by how much safe-class area they concentrate. An iterative propose-and-verify loop picks the highest-scoring candidate, sends its image patch to an MLLM for a Safe/Unsafe surface verdict, then applies hard suppression (`R'(x,y) = 0`) on accepted sites or Gaussian penalty decay on rejected ones. This tabu-style mechanism prevents re-evaluating the same area and efficiently explores the search space.

**Stage 3 (Context Reasoning)** feeds the MLLM four input streams via a structured prompt: (1) the cropped satellite image patch, (2) POI proximity vectors from OpenStreetMap quantifying distance to schools, gas stations, power lines, and other sensitive facilities, (3) dynamic context like time-of-day and weather, and (4) regulatory constraints from JARUS SORA guidelines (e.g., "Maintain 1:1 buffer distance from people"). The MLLM (paper evaluates Qwen-QVQ-max, Doubao-Vision-Pro, GPT-4.1) outputs a ranked safety verdict with a single-sentence justification. POI ablation experiments show removing POI data degrades accuracy by 23-28%, confirming its critical role.

## Step-by-Step Workflow

1. **Prepare the segmentation backbone.** Set up DeepLabV3+ via the MMSegmentation framework. Use pretrained weights from ISPRS Potsdam (0.05 m/px, 6 classes) or LoveDA depending on your target resolution. If your imagery resolution differs, apply domain adaptation (fine-tune the decoder head on a small labeled subset of your target domain) to prevent misclassification of surface types.

2. **Tile and segment the remote sensing imagery.** Split large orthophotos into non-overlapping tiles (paper uses 6000x6000 px tiles from 24000x30000 px mosaics). Run inference on each tile to produce a per-pixel class map. Extract a binary safe-candidate mask by selecting only target classes (impervious surfaces, barren land, or equivalent for your domain).

3. **Build the radially decaying convolution kernel.** Compute `K[i,j] = 1 - ((i-d)^2 + (j-d)^2) / (2d^2)` where `d` is half the UAV landing footprint in pixels. Convolve this kernel over the binary safe-candidate mask to produce a spatial density score map. Higher scores indicate larger contiguous safe areas centered at that point.

4. **Run the iterative propose-and-verify loop.** Select the pixel with the highest density score as the first candidate. Crop a patch centered on it (sized to the UAV footprint plus a safety buffer). Submit this patch to the MLLM with the surface verification prompt: *"Inspect this satellite image patch. Is the surface smooth, flat, and free of dynamic obstacles such as vehicles, people, or temporary structures? Respond Safe or Unsafe with a one-sentence justification."* If Safe, apply hard suppression (zero out a neighborhood around the accepted site). If Unsafe, apply Gaussian penalty decay to discourage but not fully exclude nearby points. Repeat until a target number of candidates are found or the score map is exhausted.

5. **Collect POI proximity vectors.** For each accepted candidate site, query OpenStreetMap (Overpass API) or equivalent POI source for all features within a configurable radius (500m-1km). Compute distance-to-nearest for each hazard category: schools, hospitals, gas stations, power lines, major roads, water bodies, residential density. Encode these as a structured vector or natural-language summary.

6. **Inject dynamic context and regulatory constraints.** Append time-of-day, day-of-week, weather conditions, and any known events (rush hour, public holiday, school hours) to the context. Embed JARUS SORA constraints as system-prompt rules, e.g., *"A landing site must maintain 1:1 horizontal buffer distance from any gathered people. Sites within 100m of fuel storage are prohibited."*

7. **Run the MLLM context reasoning stage.** Construct a multi-part prompt with: (a) the satellite image patch, (b) the POI proximity summary, (c) dynamic context, and (d) regulatory rules. Ask the model to output a structured risk assessment: overall safety rating (Safe/Marginal/Unsafe), identified hazards, distance-to-nearest-hazard, regulatory compliance status, and a human-readable justification. Parse the structured output programmatically.

8. **Rank and select final landing sites.** Score each candidate by combining the Stage 2 density score with the Stage 3 safety rating. Apply hard vetoes for any regulatory violations. Rank remaining candidates and return the top-N with their justifications.

9. **Evaluate against the ELSS benchmark (if validating).** Download the ELSS dataset (500 expert-annotated samples, 250 per region). Compute precision, recall, Right Rate (fraction correctly ranked as safe), and False Rate (fraction incorrectly ranked as safe). Compare against geometric baselines (flatness-only, slope-based) to quantify the MLLM reasoning uplift.

10. **Log and audit all decisions.** Store every MLLM prompt/response pair, the segmentation mask, POI vectors, and final scores. This audit trail enables post-incident review and supports the interpretability advantage of the approach.

## Concrete Examples

**Example 1: Building the segmentation pre-screening module**

User: "I have 0.3 m/px Google Earth composites of an urban area. Help me set up the semantic segmentation stage to filter candidate landing zones."

Approach:
1. Install MMSegmentation and download DeepLabV3+ pretrained on LoveDA (urban/rural RS dataset at comparable resolution).
2. Define the target class mapping — merge LoveDA classes into binary: safe candidates (barren, impervious surface, low vegetation below 0.5m) vs. excluded (buildings, water, forest, agriculture).
3. Write an inference script that tiles input imagery into 1024x1024 patches with 128px overlap, runs DeepLabV3+ on each, and stitches results back into a full-resolution binary mask.
4. Apply morphological opening (5x5 kernel) to remove noise and small isolated safe pixels that cannot physically accommodate a UAV.

Output:
```python
import mmcv
from mmseg.apis import init_model, inference_model
import numpy as np
from scipy.ndimage import binary_opening

config = "configs/deeplabv3plus/deeplabv3plus_r50-d8_4xb4-80k_loveda-512x512.py"
checkpoint = "checkpoints/deeplabv3plus_loveda.pth"
model = init_model(config, checkpoint, device="cuda:0")

SAFE_CLASSES = {0, 5}  # barren, impervious surface in LoveDA

def segment_tile(tile_img):
    result = inference_model(model, tile_img)
    pred = result.pred_sem_seg.data.cpu().numpy().squeeze()
    safe_mask = np.isin(pred, list(SAFE_CLASSES)).astype(np.uint8)
    safe_mask = binary_opening(safe_mask, structure=np.ones((5, 5)))
    return safe_mask
```

**Example 2: Implementing the propose-and-verify candidate search**

User: "I have a binary safe-zone mask from segmentation. How do I find the best landing candidates using the radially decaying kernel approach?"

Approach:
1. Build the kernel sized to the UAV footprint (e.g., 20m diameter = 67px at 0.3 m/px).
2. Convolve over the mask to get the density score map.
3. Iteratively extract top candidates with suppression.

Output:
```python
import numpy as np
from scipy.signal import fftconvolve

def build_kernel(d_pixels):
    size = 2 * d_pixels + 1
    y, x = np.mgrid[-d_pixels:d_pixels+1, -d_pixels:d_pixels+1]
    kernel = 1.0 - (x**2 + y**2) / (2 * d_pixels**2)
    kernel = np.clip(kernel, 0, 1)
    return kernel

def find_candidates(safe_mask, d_pixels, n_candidates, mllm_verify_fn, image):
    kernel = build_kernel(d_pixels)
    score_map = fftconvolve(safe_mask.astype(float), kernel, mode="same")
    candidates = []
    for _ in range(n_candidates * 3):  # budget 3x attempts
        if score_map.max() <= 0:
            break
        cy, cx = np.unravel_index(score_map.argmax(), score_map.shape)
        patch = image[cy-d_pixels:cy+d_pixels, cx-d_pixels:cx+d_pixels]
        verdict = mllm_verify_fn(patch)  # returns {"safe": bool, "reason": str}
        if verdict["safe"]:
            candidates.append({"x": cx, "y": cy, "reason": verdict["reason"]})
            # Hard suppression
            score_map[max(0,cy-d_pixels):cy+d_pixels, max(0,cx-d_pixels):cx+d_pixels] = 0
            if len(candidates) >= n_candidates:
                break
        else:
            # Gaussian penalty decay
            yy, xx = np.mgrid[-d_pixels:d_pixels+1, -d_pixels:d_pixels+1]
            penalty = np.exp(-(xx**2 + yy**2) / (2 * (d_pixels/2)**2))
            sy, sx = max(0,cy-d_pixels), max(0,cx-d_pixels)
            ey, ex = min(score_map.shape[0], cy+d_pixels+1), min(score_map.shape[1], cx+d_pixels+1)
            score_map[sy:ey, sx:ex] *= (1 - penalty[:ey-sy, :ex-sx])
    return candidates
```

**Example 3: MLLM context reasoning with POI fusion**

User: "I have candidate landing patches and nearby POI data from OpenStreetMap. Build the Stage 3 reasoning prompt."

Approach:
1. Structure the POI data as a proximity summary.
2. Add dynamic context and JARUS SORA rules.
3. Construct the multi-part MLLM prompt and parse the response.

Output:
```python
import json

def build_reasoning_prompt(poi_data, dynamic_context, image_description=None):
    poi_summary = "\n".join(
        f"- {poi['type']}: {poi['name']} at {poi['distance_m']:.0f}m {poi['direction']}"
        for poi in sorted(poi_data, key=lambda p: p["distance_m"])
    )
    prompt = f"""You are a UAV emergency landing safety assessor. Evaluate this candidate
landing site using the satellite image, nearby facilities, and current conditions.

## Nearby Facilities (within 1km)
{poi_summary}

## Current Conditions
- Time: {dynamic_context['time']}
- Day: {dynamic_context['day_of_week']}
- Weather: {dynamic_context['weather']}
- Special events: {dynamic_context.get('events', 'None')}

## JARUS SORA Compliance Rules
- Maintain 1:1 horizontal buffer distance from gathered people
- Minimum 100m clearance from fuel storage facilities
- No landing within 50m of high-voltage power lines
- Avoid active roadways unless traffic is confirmed stopped

## Task
Examine the satellite image patch. Considering the visual terrain, nearby facilities,
current conditions, and regulatory rules, provide your assessment as JSON:
{{
  "rating": "Safe|Marginal|Unsafe",
  "hazards": ["list of identified hazards"],
  "nearest_hazard_m": <distance in meters>,
  "regulatory_violations": ["list or empty"],
  "justification": "<one paragraph explanation>"
}}"""
    return prompt

# Usage with an MLLM API (e.g., OpenAI GPT-4.1 vision)
def assess_site(image_path, poi_data, dynamic_context, client):
    import base64
    with open(image_path, "rb") as f:
        img_b64 = base64.b64encode(f.read()).decode()
    prompt = build_reasoning_prompt(poi_data, dynamic_context)
    response = client.chat.completions.create(
        model="gpt-4.1",
        messages=[{
            "role": "user",
            "content": [
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{img_b64}"}},
                {"type": "text", "text": prompt}
            ]
        }],
        response_format={"type": "json_object"}
    )
    return json.loads(response.choices[0].message.content)
```

## Best Practices

- **Do:** Always run semantic segmentation as a pre-filter before MLLM calls. The paper shows this reduces MLLM invocations by 50-80% compared to evaluating random patches, and the filtering pass rate jumps from 18-74% (random) to 50-89% (semantic).
- **Do:** Include POI data in the reasoning prompt. Ablation experiments show removing POI degrades accuracy by 23-28%. Even approximate POI data is far better than none.
- **Do:** Apply domain adaptation when your imagery resolution differs from the segmentation model's training data. The paper documents significant misclassification when running Potsdam-trained models (0.05 m/px) on 0.3 m/px composites without adaptation.
- **Do:** Use the Gaussian penalty decay (not hard suppression) for rejected candidates. This allows nearby but distinct areas to still be considered, which matters in dense urban environments.
- **Avoid:** Skipping the regulatory constraint injection. Without JARUS SORA rules in the prompt, the MLLM will miss compliance violations that look visually benign (e.g., a flat parking lot 30m from a gas station).
- **Avoid:** Using a single MLLM call for both surface verification (Stage 2) and context reasoning (Stage 3). The paper separates these because surface verification needs only the image patch (fast, focused), while context reasoning needs the full multi-source input. Combining them degrades both tasks.

## Error Handling

- **Segmentation domain mismatch:** If the pre-screening stage marks almost everything as unsafe (>95% excluded), the model likely has a domain gap. Fall back to a coarser classification (e.g., NDVI-based vegetation filtering) and fine-tune the segmentation model on a small labeled sample from the target region.
- **MLLM refusal or hallucination:** If the MLLM returns "I cannot assess safety" or produces clearly fabricated distances, retry with a more explicit prompt specifying that this is a research/simulation context. Validate all numeric outputs (distances, counts) against the POI data you provided.
- **POI data unavailable:** If OpenStreetMap coverage is sparse for the target region, degrade gracefully: run Stage 3 without POI vectors but flag the result as "low confidence — no POI data available" and increase the safety buffer by 50%.
- **Kernel size mismatch:** If the radially decaying kernel is much larger than actual safe patches, you get zero candidates. Dynamically reduce `d` by 25% and re-run if no candidates are found in the first pass.

## Limitations

- The approach requires remote sensing imagery at 0.3 m/px or better. At coarser resolutions (e.g., 1 m/px Sentinel-2), small hazards like vehicles and temporary structures become invisible to both the segmentation and MLLM stages.
- MLLM reasoning is not real-time. Each candidate evaluation takes 2-10 seconds depending on the model; this is suitable for pre-flight planning but not for in-flight emergency response within seconds.
- POI databases are static snapshots. Temporary hazards (construction sites, outdoor events, parked vehicle clusters) will not appear in OpenStreetMap and must be detected visually by the MLLM — which is less reliable than structured data.
- The ELSS benchmark covers two regions (Potsdam, Nanjing). Performance on radically different terrain (desert, coastal, mountainous) is unvalidated and may require new segmentation training data.
- The paper reports 68% Right Rate and 8-11% False Rate for the full pipeline. This means roughly 1 in 10 sites rated "safe" may actually be unsafe — human review remains essential for safety-critical deployment.

## Reference

Hua, C., Yang, Z., Zhang, L., Sun, J., & Chen, F. (2026). *Semantically Aware UAV Landing Site Assessment from Remote Sensing Imagery via Multimodal Large Language Models.* arXiv:2602.01163v1. https://arxiv.org/abs/2602.01163v1

Key sections: Section 3 for the three-stage pipeline architecture, Section 4 for the ELSS benchmark construction, and Table 2/3 for ablation results showing the impact of POI data and segmentation pre-filtering on accuracy.