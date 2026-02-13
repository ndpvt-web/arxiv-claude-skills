---
name: "ic-eo-interpretable-code-based-assistant"
description: "Build conversational Earth Observation agents that turn natural-language queries into executable, auditable Python workflows. Uses a unified API covering classification, segmentation, detection, spectral indices, and geospatial operations. Trigger phrases: 'analyze satellite imagery', 'earth observation pipeline', 'land cover classification code', 'post-wildfire damage assessment', 'generate EO workflow', 'spectral index calculation'."
---

# IC-EO: Interpretable Code-Based Assistant for Earth Observation

This skill enables Claude to build **code-generating Earth Observation agents** that convert plain-English requests into executable, auditable Python pipelines. Based on the IC-EO framework (Lahouel et al., 2026), the core idea is a "code-only contract": instead of returning black-box predictions or free-text answers, the agent always outputs runnable Python that calls a structured, unified API of EO tools -- classification, segmentation, object detection (with oriented bounding boxes), spectral indices, and geospatial operators. Every result is transparent, reproducible, and verifiable at three levels: individual tool accuracy, code validity, and end-to-end task correctness.

## When to Use

- When a user asks to **analyze satellite or aerial imagery** using classification, segmentation, or object detection and wants reproducible Python code rather than a one-off answer.
- When building a **conversational agent or chatbot** that wraps EO foundation models (DOFA, Prithvi-EO-2.0, SAMGeo, YOLOv11-OBB) behind a natural-language interface.
- When a user needs to **compute spectral indices** (NDVI, NDWI, EVI, SAVI, NDSI, etc.) from Sentinel-2 or similar multispectral data and wants a code pipeline.
- When performing **land-cover composition analysis** -- e.g., "What percentage of this image is forest vs. cropland?"
- When assessing **post-disaster damage** (wildfire burn scars, flood extent) by combining remote sensing models with ancillary data.
- When a user wants an **extensible EO API design** where new models or tools can be added without rewriting the agent.
- When **auditability and interpretability** are requirements -- the user or reviewer must be able to inspect every processing step.

## Key Technique

The IC-EO architecture enforces a strict separation between the **LLM controller** (which plans and writes code) and the **execution environment** (which runs it in a sandboxed container). The LLM receives structured tool specifications -- each tool documented with its function signature, argument types, return types, supported sensors, normalization requirements, and dataset taxonomies. This structured schema dramatically reduces hallucination: IC-EO achieves an 87% execution success rate, with failures primarily caused by resource constraints rather than code errors. The key insight is that **constraining the LLM to emit only code that calls well-documented functions** is more reliable and auditable than letting it reason freely about imagery.

The unified API wraps several foundation models under a consistent interface: **DOFA** (a neural-plasticity-inspired encoder) for scene classification and semantic segmentation, **Prithvi-EO-2.0** for multi-temporal burn-scar mapping, **YOLOv11-OBB** for oriented bounding box detection, and **SAMGeo** for zero-shot prompt-based segmentation. Data ingestion tools connect to **Google Earth Engine** for Sentinel-1/2 access with built-in cloud masking, reprojection, and tiling. A spectral indices library covers the standard vegetation, water, and snow indices.

Evaluation operates at three levels: **(L1) Tool-level** benchmarks individual models on public datasets (e.g., RESISC45 at 97% top-1 accuracy, HLS Burn Scars at 87.5% IoU); **(L2) Agent-level** measures code validity and execution success rate; **(L3) Task-level** tests end-to-end performance on real use cases. This three-level framework is the recommended pattern for any code-generating agent -- it isolates whether a failure stems from a bad model, bad code, or a bad plan.

## Step-by-Step Workflow

1. **Define structured tool specifications.** For each EO capability (classify, segment, detect, compute index, fetch data), write a specification containing: function name, argument names and types, return type, supported sensor/band configurations, normalization requirements, and 1-2 usage examples. Store these as structured text or JSON that will be injected into the LLM's system prompt.

2. **Construct the system prompt with a code-only contract.** Instruct the LLM to output *only* executable Python -- no prose, no markdown fences. Require that the final result always be `print()`-ed. Include all tool specifications in the system prompt so the LLM knows exactly what functions are available and how to call them.

3. **Parse the user's natural-language query and map it to required tools.** The LLM performs both planning (which tools to call, in what order) and synthesis (generating the actual code) in a single generation step. For multi-step queries, the generated code chains tool calls sequentially.

4. **Generate executable Python that calls the unified API.** The code should: load input data (uploaded image path or GEE query), invoke the appropriate model tool(s), apply any post-processing (masking, area computation, class filtering), and print the final result. Every intermediate variable should have a descriptive name for auditability.

5. **Execute the generated code in a sandboxed environment.** Run with pinned dependencies, restricted network access, and resource limits (CPU/GPU allocation, memory caps). Capture stdout, stderr, and any generated figures or masks.

6. **Log all tool calls, arguments, and outputs for traceability.** Store the full execution trace so that any result can be reproduced. This is the core of the interpretability guarantee.

7. **Return both the result and the generated code to the user.** Always surface the Python code alongside the answer so the user can audit, modify, and re-run it.

8. **Handle failures at each evaluation level.** If tool-level: check model compatibility with the input sensor/resolution. If agent-level (code error): parse the traceback, identify the failing tool call, and regenerate with corrected arguments. If task-level (wrong answer): re-examine tool selection and post-processing logic.

9. **Extend the API by adding new tool specifications.** When a new model or data source is needed, write its specification following the same schema (function signature, supported inputs, examples) and append it to the system prompt. No changes to the agent loop are required.

## Concrete Examples

**Example 1: Land-Cover Composition Analysis**

User: "What percentage of this satellite image is forest, and is there any brushwood present?"

Approach:
1. Identify this as a semantic segmentation task requiring class-level area computation.
2. Generate code that loads the uploaded image, runs DOFA segmentation, computes per-class pixel counts, and checks for the brushwood class.
3. Return percentages and a boolean for brushwood presence.

Generated code:
```python
from iceo_api import dofa_segmentation_tool, get_uploaded_image_path
import numpy as np

image_path = get_uploaded_image_path()
segmentation_mask = dofa_segmentation_tool(image_path)

total_pixels = segmentation_mask.size
# Class IDs: 6 = forest, 8 = brushwood (from FLAIR-2 taxonomy)
forest_pixels = (segmentation_mask == 6).sum()
forest_pct = round(100.0 * forest_pixels / total_pixels, 1)

brushwood_present = 8 in segmentation_mask

print(f"Forest coverage: {forest_pct}%")
print(f"Brushwood present: {brushwood_present}")
```

Output:
```
Forest coverage: 34.7%
Brushwood present: True
```

---

**Example 2: Post-Wildfire Burn Scar Assessment**

User: "Show the burn scar extent near coordinates 34.05, -118.25 from the January 2025 wildfire using Sentinel-2 data."

Approach:
1. Identify as a multi-step pipeline: data retrieval from GEE, then burn-scar segmentation with Prithvi-EO-2.0.
2. Generate code that fetches a Sentinel-2 tile, applies cloud masking, runs burn-scar detection, and computes the affected area.

Generated code:
```python
from iceo_api import (
    fetch_sentinel2_tile, cloud_mask, prithvi_burn_scar_tool,
    compute_area_km2
)

# Fetch and preprocess Sentinel-2 data
tile = fetch_sentinel2_tile(
    lat=34.05, lon=-118.25,
    date_range=("2025-01-01", "2025-01-31"),
    bands=["B2", "B3", "B4", "B8A", "B11", "B12"]
)
clean_tile = cloud_mask(tile)

# Run Prithvi-EO-2.0 burn scar segmentation
burn_mask = prithvi_burn_scar_tool(clean_tile)

# Compute burned area
burned_area = compute_area_km2(burn_mask, class_id=1)
print(f"Burned area: {burned_area:.2f} km²")
```

Output:
```
Burned area: 12.38 km²
```

---

**Example 3: Object Detection in Aerial Imagery**

User: "Count the number of vehicles in this aerial photo and show their oriented bounding boxes."

Approach:
1. Identify as an oriented bounding box detection task using YOLOv11-OBB.
2. Generate code that runs detection, filters for the vehicle class, counts detections, and saves an annotated image.

Generated code:
```python
from iceo_api import yolo_obb_detect, get_uploaded_image_path, draw_obb_boxes

image_path = get_uploaded_image_path()
detections = yolo_obb_detect(image_path)

# Filter for vehicle class (NWPU VHR-10 taxonomy)
vehicles = [d for d in detections if d["class"] == "vehicle"]

annotated = draw_obb_boxes(image_path, vehicles, output_path="result.png")
print(f"Vehicles detected: {len(vehicles)}")
print(f"Annotated image saved to: {annotated}")
```

Output:
```
Vehicles detected: 23
Annotated image saved to: result.png
```

---

**Example 4: Spectral Index Computation**

User: "Calculate NDVI and NDWI for this Sentinel-2 scene and highlight areas where vegetation health is poor."

Generated code:
```python
from iceo_api import (
    fetch_sentinel2_tile, compute_spectral_index, get_uploaded_image_path
)
import numpy as np

image_path = get_uploaded_image_path()

ndvi = compute_spectral_index(image_path, index="NDVI")  # (B8 - B4) / (B8 + B4)
ndwi = compute_spectral_index(image_path, index="NDWI")  # (B3 - B8) / (B3 + B8)

# Poor vegetation health: NDVI < 0.2 in non-water areas (NDWI < 0)
poor_health_mask = (ndvi < 0.2) & (ndwi < 0)
poor_health_pct = round(100.0 * poor_health_mask.sum() / poor_health_mask.size, 1)

print(f"NDVI range: [{ndvi.min():.3f}, {ndvi.max():.3f}]")
print(f"NDWI range: [{ndwi.min():.3f}, {ndwi.max():.3f}]")
print(f"Area with poor vegetation health: {poor_health_pct}%")
```

## Best Practices

- **Do:** Always include structured tool specifications with exact function signatures, argument types, and return types in the system prompt. This is the single most effective measure against hallucinated function calls.
- **Do:** Enforce the code-only contract -- the LLM must output executable Python with no surrounding prose. This eliminates parsing ambiguity and makes outputs directly runnable.
- **Do:** Log every tool invocation with its arguments and return values. This audit trail is what makes IC-EO interpretable; without it, you lose the core advantage over black-box systems.
- **Do:** Include dataset taxonomies (class IDs and their meanings) in tool specifications so the LLM generates correct class filters without guessing.
- **Avoid:** Letting the LLM choose arbitrary model names or invent tool functions not in the specification. If a tool doesn't exist in the API, the agent should say so rather than hallucinate one.
- **Avoid:** Skipping the sandboxed execution step. Running LLM-generated code without resource limits and network restrictions is a security risk, especially with GEE credentials in scope.
- **Avoid:** Evaluating only at the task level. A wrong answer could be caused by a bad model (L1), bad code (L2), or bad planning (L3) -- always diagnose at all three levels.

## Error Handling

| Failure Mode | Level | Detection | Recovery |
|---|---|---|---|
| CUDA out of memory | L1 (tool) | RuntimeError in traceback | Reduce input tile size, switch to CPU, or tile the image and process in patches |
| Hallucinated function call | L2 (agent) | `NameError` or `AttributeError` | Re-prompt with explicit reminder of available tool names; tighten system prompt |
| Wrong class ID used | L2 (agent) | Code runs but answer is incorrect | Include full class taxonomy in tool spec; validate class IDs against known set before execution |
| Cloud-contaminated input | L1 (data) | Anomalous spectral values or blank segmentation | Apply cloud masking as a mandatory preprocessing step; reject tiles above a cloud-cover threshold |
| GEE quota exceeded | L1 (data) | API rate-limit error | Implement backoff/retry; cache previously fetched tiles |
| Incorrect sensor normalization | L1 (tool) | Degraded model accuracy | Document per-sensor normalization in tool specs; validate band value ranges before model inference |

## Limitations

- **Sensor-specific models:** DOFA and Prithvi are trained on specific sensor configurations. Applying them to imagery from unsupported sensors (e.g., commercial VHR with different band definitions) without retraining or fine-tuning will degrade accuracy.
- **Single-step code generation:** The current architecture generates the full pipeline in one LLM call. For highly complex multi-stage workflows (e.g., time-series change detection across dozens of tiles), iterative planning with intermediate feedback would be more robust.
- **GPU resource dependency:** Several models (DOFA segmentation, Prithvi) require GPU inference. The 13% failure rate in the paper was primarily due to CUDA memory limits, not code errors. Plan for resource-constrained environments.
- **Fixed taxonomies:** Class sets are tied to training datasets (FLAIR-2, RESISC45, EuroSAT). Custom land-cover categories not in these taxonomies require model fine-tuning.
- **No temporal reasoning built in:** While Prithvi supports multi-temporal inputs, the agent does not inherently reason about temporal dynamics. Time-series analysis queries require explicit tool support.
- **Accuracy ceiling:** IC-EO achieves 64.2% on land-composition questions -- better than general-purpose VLMs but not yet expert-level. Critical decisions should involve human review of the generated code and results.

## Reference

**Paper:** Lahouel, L., Lopata, L., Gruening, S., Meoni, G., & Petit, G. (2026). *IC-EO: Interpretable Code-based assistant for Earth Observation.* arXiv:2602.00117v1.
https://arxiv.org/abs/2602.00117v1

**Key takeaway:** Constraining an LLM to generate code against a well-documented, structured API of EO tools yields more accurate and auditable results than prompting general-purpose VLMs to answer EO questions directly. The three-level evaluation framework (tool / agent / task) is a reusable pattern for any code-generating agent system.