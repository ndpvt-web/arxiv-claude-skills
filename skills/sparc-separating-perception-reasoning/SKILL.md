---
name: "sparc-separating-perception-reasoning"
description: |
  Decouple visual perception from reasoning when building VLM pipelines, image analysis agents,
  or multi-modal workflows. Implements the SPARC two-stage pattern: first localize question-relevant
  regions via visual search, then condition reasoning only on those regions. Dramatically reduces
  token budgets and improves accuracy on visually complex tasks.
  Trigger phrases:
  - "Build an image QA pipeline that scales at test time"
  - "Analyze this high-resolution image efficiently with a VLM"
  - "Separate perception from reasoning in my vision pipeline"
  - "Reduce token cost for visual reasoning tasks"
  - "Implement a two-stage visual search and answer system"
  - "Scale VLM inference with asymmetric compute allocation"
---

# SPARC: Separating Perception and Reasoning Circuits

SPARC enables Claude to architect vision-language pipelines that **explicitly decouple visual perception from reasoning**. Instead of feeding an entire image into a monolithic chain-of-thought (where small perceptual errors cascade into wrong answers), you build a two-stage pipeline: Stage 1 performs visual search to localize question-relevant regions and outputs bounding boxes or point coordinates; Stage 2 receives only those cropped regions (plus the original image for context) and reasons over focused visual evidence to produce the final answer. This separation yields 6-7 percentage point accuracy gains on hard VQA benchmarks while using up to 200x fewer tokens than "thinking with images" approaches.

## When to Use

- When building a VLM-powered QA system that must handle high-resolution images (4K, 8K, satellite imagery, medical scans) without blowing up token budgets
- When a user asks to create an image analysis agent that needs to find specific objects or regions before answering questions about them
- When implementing test-time scaling for vision models and wanting to allocate more compute to perception vs. reasoning independently
- When building a pipeline where perceptual mistakes (wrong object identified) are the primary failure mode and you need to isolate and fix them
- When the user wants to process very large images (e.g., remote sensing at 8500x8500px) by searching at low resolution and cropping at high resolution
- When creating a multi-step visual reasoning system where perception and logical inference are distinct bottlenecks
- When the user wants self-consistency / ensemble methods but cannot afford to run full reasoning chains N times

## Key Technique

**The core insight**: Monolithic VLM chains-of-thought entangle "what am I looking at?" (perception) with "what does it mean?" (reasoning) in a single token stream. This entanglement causes two problems: (1) perceptual errors early in the chain corrupt all downstream reasoning, and (2) you cannot independently scale or optimize either capability. SPARC solves this by enforcing an explicit architectural boundary between the two.

**Stage 1 — Implicit Relevance Detection (IRD)**: The VLM receives the image and the question but is explicitly instructed *not to answer*. Instead, it outputs bounding box coordinates (JSON format) or point coordinates identifying regions relevant to answering the question. This is "implicit" because the prompt doesn't name the target object — the model must infer which regions matter from the question context. Crucially, this stage can run at reduced resolution (256px) because localization doesn't require fine-grained detail, and multiple rollouts (N=8 at temperature 0.7) can be aggregated via Weighted Boxes Fusion (merging boxes with >= 50% IoU) to produce robust region proposals cheaply.

**Stage 2 — Perceptually Grounded Reasoning**: The VLM receives the original image, the high-resolution crops extracted from Stage 1 coordinates, and the question. It now reasons only over focused visual evidence. Because the reasoning context is lean (typically ~3 crops instead of the full image at max resolution), this stage is both cheaper and more accurate. KV-caches from the perception stage can be shared to avoid redundant computation.

## Step-by-Step Workflow

1. **Parse the visual question task**: Identify the input image(s), the question or task description, and whether the task is perception-bottlenecked (finding/identifying objects) or reasoning-bottlenecked (logical inference over known objects). This determines compute allocation.

2. **Design the IRD prompt**: Write a Stage 1 prompt that provides the image and question but explicitly instructs the model to localize, not answer. Use the pattern: *"Your role is not to answer the question, but to identify the objects relevant to answering it and return their 2D bounding boxes with labels in JSON format. DO NOT ANSWER THE QUESTION."*

3. **Set IRD resolution and rollout count**: Configure the perception stage to run at reduced resolution (256px or 512px longest dimension). For hard localization tasks or distribution-shifted images, use N=4-8 rollouts at temperature 0.7 to get diverse region proposals.

4. **Aggregate region proposals**: If using multiple rollouts, merge overlapping bounding boxes using Weighted Boxes Fusion (WBF). Group boxes with IoU >= 0.5, take the confidence-weighted average of coordinates, and deduplicate. Expect ~3-4 final regions from 8 rollouts.

5. **Extract high-resolution crops**: Map the bounding box coordinates back to the original full-resolution image. Extract crops at native resolution (or the highest resolution your reasoning model supports). Resize crops to the reasoning model's input resolution if needed.

6. **Design the reasoning prompt**: Write a Stage 2 prompt that provides the original image (for global context), the extracted crops (for focused detail), and the question. Use the pattern: *"You are given an image and relevant crops to answer the following question: [Q]. Use the crops to examine fine details."*

7. **Run Stage 2 reasoning**: Execute the reasoning model at the target resolution with the lean context (original image + crops + question). If the task has multiple-choice options, instruct the model to select from the given choices.

8. **Implement selective optimization**: Monitor per-stage accuracy. If perception is the bottleneck (wrong regions localized), increase IRD rollouts, fine-tune the perception stage with LoRA (rank=16, alpha=32, 2 epochs on ~20K samples), or raise perception resolution. If reasoning is the bottleneck, improve the reasoning prompt or use a larger model for Stage 2 only.

9. **Handle edge cases**: If Stage 1 returns no bounding boxes, fall back to full-image reasoning. If Stage 1 returns too many boxes (>6), filter by confidence score or apply non-maximum suppression more aggressively.

10. **Measure and iterate**: Compare end-to-end accuracy and total token count against a monolithic baseline (same model, full image, single chain-of-thought). The SPARC pipeline should show clear accuracy gains on perception-heavy tasks and substantial token savings on high-resolution inputs.

## Concrete Examples

**Example 1: High-Resolution Object Attribute Question**

```
User: Build a pipeline that can answer "What color is the scarf?" given a
high-resolution street photo where the scarf is a small region of the image.

Approach:
1. Stage 1 (IRD) — Send the image at 256px with the prompt:
   "You will be given an image and a question for context. Your role is not
    to answer the question, but identify the objects the user will ask about
    and return their 2D bounding box and label in JSON format.
    Question: What color is the scarf? DO NOT ANSWER THE QUESTION."

   Model output:
   [{"label": "scarf", "bbox": [0.42, 0.31, 0.58, 0.45]}]

2. Stage 2 (Reasoning) — Extract the crop at [0.42, 0.31, 0.58, 0.45]
   from the original 4K image, resize to 512px, and send with prompt:
   "You are given an image and a relevant crop. Answer: What color is the scarf?"

   Model output: "blue"

Result: Correct answer with ~4x fewer visual tokens than full-resolution
monolithic inference.
```

**Example 2: Remote Sensing with Very Large Images**

```
User: I have 8500x8500px satellite images. I need to answer questions like
"How many ships are docked at the port?" without exceeding my token budget.

Approach:
1. Stage 1 (IRD at 256px) — Downscale to 256px longest edge. Run 8 rollouts
   at T=0.7 with prompt:
   "Identify all regions containing ships or port infrastructure. Return
    bounding boxes in JSON. DO NOT ANSWER THE QUESTION."

   8 rollouts produce ~24 total boxes. After WBF (IoU >= 0.5): 5 unique regions.

2. Stage 2 (Reasoning at 512px crops) — Extract 5 crops from the original
   8500x8500 image at native resolution, resize each to 512px.
   Send with prompt:
   "Given these satellite image crops showing port regions, count the ships
    docked at the port."

   Model output: "7 ships are visible at the docking area."

Result: Processes ~0.1% of original visual tokens. Outperforms full-resolution
baseline which struggles with the massive context.
```

**Example 3: Implementing Self-Consistency on the Cheap**

```
User: I want ensemble-style robustness for my VQA system but can't afford
to run 8 full reasoning chains.

Approach:
1. Run Stage 1 (IRD) 8 times at T=0.7 — this is cheap because:
   - Runs at 256px resolution
   - Outputs only bounding box JSON (few tokens)
   - Total cost: ~8x a single short inference call

2. Aggregate with WBF — 8 rollouts with ~4 boxes each = ~32 raw boxes.
   After merging (IoU >= 0.5): ~3-4 robust region proposals.
   Confidence scores from WBF indicate localization certainty.

3. Run Stage 2 ONCE with the 3-4 deduplicated crops.

Result: Ensemble-quality perception (8 diverse samples) with single-pass
reasoning cost. Accuracy improves monotonically with rollout count
(51.0% -> 55.7% on V* benchmark with 4B model).
```

## Best Practices

- **Do**: Always instruct the IRD stage to NOT answer the question. The explicit prohibition ("DO NOT ANSWER THE QUESTION") is critical — without it, the model leaks reasoning into the perception stage, re-entangling the two circuits.
- **Do**: Run perception at the lowest resolution that still produces accurate localization (256px is often sufficient). This is the core efficiency win — global search is cheap, local detail is expensive.
- **Do**: Use multiple IRD rollouts with WBF aggregation when localization accuracy matters. The marginal cost is low (short JSON outputs at low resolution) and accuracy gains are monotonic.
- **Do**: Share KV-caches between stages when using the same model for both, to avoid recomputing image embeddings.
- **Avoid**: Passing the full-resolution image to Stage 1. The entire point of the compressed context strategy is that localization works at low resolution; high-resolution perception wastes tokens without accuracy benefit.
- **Avoid**: Using more than ~6 crops in Stage 2. Beyond this, the reasoning context becomes cluttered and accuracy degrades. If Stage 1 returns many boxes, filter by confidence.
- **Avoid**: Fine-tuning both stages simultaneously. The power of SPARC is selective optimization — diagnose which stage is the bottleneck and improve only that stage.

## Error Handling

| Failure Mode | Symptom | Recovery |
|---|---|---|
| Stage 1 returns empty boxes | No regions localized | Fall back to full-image reasoning at reduced resolution; consider rephrasing the IRD prompt to be more explicit about target objects |
| Stage 1 localizes wrong region | Final answer is confidently wrong about a different object | Increase IRD rollouts (N=8+), lower temperature threshold, or fine-tune the perception stage with LoRA on corrective examples |
| Too many overlapping crops | Stage 2 context is cluttered, accuracy drops | Tighten WBF IoU threshold (0.6+), apply NMS, or cap crops at 4-5 highest-confidence regions |
| Reasoning fails despite correct crops | Model has enough visual evidence but answers incorrectly | This is a reasoning bottleneck — use a larger model for Stage 2, add chain-of-thought prompting within Stage 2, or provide answer format constraints |
| Coordinate format mismatch | Crops extracted from wrong image regions | Validate that IRD outputs normalized coordinates [0,1] vs. pixel coordinates and convert consistently before cropping |

## Limitations

- **Requires bounding-box-capable VLMs**: The IRD stage needs a model that can output spatial coordinates (e.g., Qwen3-VL, Molmo2). Not all VLMs support grounded outputs natively.
- **Ill-posed perception target**: The "optimal" crop depends on what maximizes downstream accuracy, not necessarily the tightest bounding box around the named object. This means IRD sometimes needs to include surrounding context, which is hard to specify in prompts.
- **Overhead on easy questions**: For simple questions where the answer is visible at any resolution (e.g., "Is this photo taken indoors?"), the two-stage pipeline adds unnecessary latency. Use SPARC selectively for perception-heavy tasks.
- **Not suited for holistic scene understanding**: Questions like "Describe the overall mood of this image" don't benefit from region localization. SPARC is designed for tasks where specific visual details determine the answer.
- **Fine-tuning requires teacher traces**: To train a better IRD stage, you need crop coordinates extracted from a larger model's reasoning traces with rejection sampling (~20K examples). This is a non-trivial data pipeline.

## Reference

**Paper**: [SPARC: Separating Perception And Reasoning Circuits for Test-time Scaling of VLMs](https://arxiv.org/abs/2602.06566v2) (Avogaro et al., 2026)

Look for: Section 3 (Method) for the full IRD prompt templates, WBF aggregation details, and compressed context strategy; Table 2 for self-consistency scaling curves; Table 3 for fine-tuning ablations; Figure 2 for the resolution-vs-crop-overlap tradeoff that justifies low-resolution global search.