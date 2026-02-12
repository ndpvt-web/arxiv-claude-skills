---
name: "forest-chat-adapting-vision-language-agents"
description: "Build LLM-orchestrated agents for satellite image change analysis using a multi-tool VLM pipeline. The agent routes natural language queries to specialized vision tools (change detection, captioning, counting, deforestation estimation, reasoning) and synthesizes results. Trigger phrases: 'build a forest change detection agent', 'satellite image change analysis pipeline', 'bi-temporal remote sensing agent', 'LLM-orchestrated vision tool pipeline', 'change detection and captioning system', 'interactive point-prompt segmentation agent'."
---

# Forest-Chat: LLM-Orchestrated Vision-Language Agents for Satellite Change Analysis

This skill teaches Claude to build **LLM-driven agent systems that orchestrate multiple vision-language tools for bi-temporal satellite image change interpretation**. Based on the Forest-Chat framework, the core pattern is an LLM controller that receives natural language queries, selects and invokes specialized perception modules (change detection, change captioning, object counting, deforestation estimation), and synthesizes their outputs into coherent natural language responses. This architecture generalizes beyond forestry to any domain requiring multi-tool vision analysis with conversational interfaces.

## When to Use

- When the user wants to build an agent that routes natural language queries to multiple computer vision tools and synthesizes their outputs
- When designing a bi-temporal image analysis pipeline (before/after comparison) with both pixel-level and semantic outputs
- When implementing an LLM orchestrator that generates Python code to invoke vision model APIs dynamically
- When building an interactive point-prompt interface where users click on satellite or aerial images to guide segmentation
- When creating a change detection system that combines supervised (fine-tuned) and zero-shot (foundation model) pathways
- When the user needs a multi-granularity captioning system that produces descriptions at different levels of detail from the same visual features
- When constructing datasets with hybrid human + rule-based annotation pipelines for change detection tasks

## Key Technique

**LLM-as-Orchestrator with Dual Vision Pathways.** Forest-Chat uses an LLM (GPT-4o-mini class) as a central controller that interprets user queries and dispatches them to specialized vision tools. The LLM generates and executes Python programs to invoke perception modules rather than calling them through rigid function signatures. This code-generation approach provides flexibility: the LLM can compose multi-step analyses, perform arithmetic on mask statistics, and chain tool outputs without a fixed task router. System prompts with few-shot examples constrain the LLM to reliable tool invocation patterns and consistent output formatting.

**Multi-Level Change Interpretation (MCI) Backbone.** The core vision model uses a dual-branch Siamese architecture. Two SegFormer B1 encoders process bi-temporal image pairs, extracting multi-scale features. Bi-temporal Iterative Interaction (BI3) layers combine Local Perception Enhancement (multi-scale convolutions capturing fine spatial detail) with Global Difference Fusion Attention (feature differencing + cross-attention highlighting true changes while suppressing noise). One branch produces pixel-level change masks through multi-scale fusion and deconvolution; the other projects visual features into a Transformer decoder for caption generation. A gradient-detached loss balancing scheme prevents either task from dominating training.

**Zero-Shot Change Detection via Foundation Models.** For scenarios without training data, Forest-Chat adapts SAM (Segment Anything Model) through bi-temporal latent matching. SAM generates mask proposals for each temporal image independently. Average embeddings are computed per mask, then cosine similarity identifies corresponding regions across time. Deviations in similarity scores indicate change. This works because SAM's latent space clusters same-category objects consistently, even across temporal imaging variations. Users can further refine detection by clicking points of interest, which generate SAM point prompts for object-level change scoring.

## Step-by-Step Workflow

1. **Define the tool registry.** Create a Python module that registers each vision capability as a callable tool with a name, description, input schema, and return type. Include at minimum: `detect_changes(image_a, image_b) -> mask`, `caption_changes(image_a, image_b) -> str`, `count_objects(mask, class_id) -> int`, `estimate_deforestation(mask) -> float`, `reason_about_changes(caption, mask_stats) -> str`. Each tool wraps a model inference call.

2. **Build the Siamese feature extraction pipeline.** Implement dual SegFormer B1 encoders sharing weights, processing bi-temporal image pairs (pre-change and post-change). Extract features at 4 scales (1/4, 1/8, 1/16, 1/32 resolution). Use `timm` or `transformers` library for the SegFormer backbone, loading pretrained weights.

3. **Implement BI3 interaction layers.** For each feature scale, apply: (a) Local Perception Enhancement using parallel depthwise convolutions at kernel sizes 3, 5, 7 with channel attention fusion, (b) Global Difference Fusion Attention by computing element-wise feature differences between temporal branches and applying multi-head cross-attention. Stack 2-3 BI3 layers with residual connections per scale.

4. **Branch into detection and captioning heads.** Detection branch: fuse multi-scale BI3 outputs using learned convolution-based fusion, then upsample through deconvolution to produce a binary change mask at input resolution. Captioning branch: project fused BI3 features to 512-dim embeddings, feed to a Transformer decoder (4-6 layers) with learned word embeddings to auto-regressively generate change descriptions.

5. **Integrate zero-shot change detection as a fallback pathway.** Load a pretrained SAM model. For each temporal image, generate mask proposals using the automatic mask generator. Compute average image encoder embeddings per proposal. Match proposals across temporal images using cosine similarity. Flag proposals whose best-match similarity falls below a threshold (default: 0.85) as changed regions. Merge flagged proposals into a unified change mask.

6. **Implement the point-prompt interface.** Accept user-provided (x, y) coordinates on either temporal image. Use SAM's point prompt encoder to generate a focused mask proposal at that location. Compute its average embedding and match against all proposals in the other temporal image. Return a per-object change score (cosine angle) and let the user set an acceptance threshold.

7. **Build the LLM orchestration layer.** Construct a system prompt that lists available tools with their signatures and example invocations. Include 3-5 few-shot examples showing: (a) a change detection query mapped to `detect_changes` + mask visualization, (b) a captioning query mapped to `caption_changes`, (c) a compound query like "How much forest was lost and why?" mapped to sequential calls to `detect_changes`, `estimate_deforestation`, and `reason_about_changes`. The LLM generates Python code blocks that the system executes in a sandboxed environment.

8. **Implement result synthesis.** After tool execution, pass raw outputs (mask statistics, captions, counts) back to the LLM with a synthesis prompt: "Given these analysis results, provide a coherent natural language summary addressing the user's original question." Format the response with both textual explanation and references to any generated visualizations.

9. **Create the dataset pipeline (if training the MCI backbone).** For each bi-temporal pair: generate one human-written caption, then programmatically derive 4 additional captions from mask properties—deforestation percentage binned into descriptors ("minimal" <5%, "moderate" 5-15%, "significant" 15-30%, "severe" >30%), patch size distribution, spatial clustering, and edge characteristics. Split 80/10/10 train/val/test.

10. **Evaluate with task-specific metrics.** Change detection: MIoU and per-class IoU (critical for imbalanced masks where change pixels are <5%). Captioning: BLEU-4 and CIDEr-D. Overall system: end-to-end latency per query, tool selection accuracy (does the LLM pick the right tool?), and response factual consistency (do claims match mask statistics?).

## Concrete Examples

**Example 1: Building the LLM Tool Orchestrator**

User: "I want to build an agent that takes natural language questions about satellite image pairs and routes them to the right vision model."

Approach:
1. Define the tool registry as a Python dictionary mapping tool names to functions
2. Write the system prompt with tool descriptions and few-shot routing examples
3. Implement the execution loop: parse LLM output for code blocks, execute in sandbox, return results

Output:
```python
# tool_registry.py
TOOLS = {
    "detect_changes": {
        "fn": run_change_detection,
        "description": "Produce a binary change mask from bi-temporal image pair",
        "inputs": {"image_a": "path", "image_b": "path"},
        "returns": "numpy array (H, W) with 0=no-change, 1=change"
    },
    "caption_changes": {
        "fn": run_change_captioning,
        "description": "Generate a natural language description of changes between two images",
        "inputs": {"image_a": "path", "image_b": "path"},
        "returns": "string describing the observed changes"
    },
    "count_changed_objects": {
        "fn": run_object_counting,
        "description": "Count distinct changed regions in a change mask",
        "inputs": {"mask": "numpy array"},
        "returns": "integer count of connected components"
    },
    "estimate_change_percentage": {
        "fn": run_percentage_estimation,
        "description": "Calculate the percentage of the image area that changed",
        "inputs": {"mask": "numpy array"},
        "returns": "float between 0.0 and 100.0"
    },
}

# system_prompt.py
SYSTEM_PROMPT = """You are a satellite image change analysis agent. You have access
to the following tools:

{tool_descriptions}

When the user asks a question, write a Python code block that calls the appropriate
tool(s) and stores results in a variable called `result`. For compound questions,
chain multiple tool calls.

Examples:
---
User: "What changed between these two images?"
```python
mask = detect_changes(image_a="pre.tif", image_b="post.tif")
caption = caption_changes(image_a="pre.tif", image_b="post.tif")
result = {"mask": mask, "caption": caption}
```
---
User: "How much deforestation occurred?"
```python
mask = detect_changes(image_a="pre.tif", image_b="post.tif")
pct = estimate_change_percentage(mask=mask)
result = {"deforestation_percent": pct, "mask": mask}
```
"""

# orchestrator.py
import ast, io, contextlib

def run_agent(user_query: str, image_a: str, image_b: str, llm_client):
    tool_desc = "\n".join(f"- {k}: {v['description']}" for k, v in TOOLS.items())
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT.format(tool_descriptions=tool_desc)},
        {"role": "user", "content": user_query},
    ]
    response = llm_client.chat(messages)
    code = extract_code_block(response)

    # Execute in sandboxed namespace with tool functions available
    namespace = {name: spec["fn"] for name, spec in TOOLS.items()}
    namespace.update({"image_a": image_a, "image_b": image_b})
    exec(code, namespace)

    raw_result = namespace.get("result", {})
    # Synthesize natural language response
    synthesis = llm_client.chat([
        {"role": "system", "content": "Summarize these analysis results clearly."},
        {"role": "user", "content": f"Query: {user_query}\nResults: {raw_result}"},
    ])
    return synthesis, raw_result
```

**Example 2: Implementing Zero-Shot Change Detection with SAM**

User: "I need change detection that works on new satellite imagery without any fine-tuning."

Approach:
1. Load pretrained SAM and generate mask proposals for both temporal images
2. Compute average embeddings per mask proposal
3. Match proposals across time via cosine similarity and threshold for changes

Output:
```python
import numpy as np
import torch
from segment_anything import sam_model_registry, SamAutomaticMaskGenerator

def zero_shot_change_detection(image_pre, image_post, sam_checkpoint,
                                model_type="vit_h", similarity_threshold=0.85):
    sam = sam_model_registry[model_type](checkpoint=sam_checkpoint)
    sam.eval()
    generator = SamAutomaticMaskGenerator(sam)

    # Generate mask proposals for each temporal image
    masks_pre = generator.generate(image_pre)
    masks_post = generator.generate(image_post)

    def compute_mask_embeddings(image, masks, model):
        """Average image encoder features within each mask region."""
        with torch.no_grad():
            features = model.image_encoder(preprocess(image))  # (1, C, H, W)
        embeddings = []
        for m in masks:
            binary = torch.from_numpy(m["segmentation"]).float()
            # Downsample mask to feature resolution
            binary_ds = torch.nn.functional.interpolate(
                binary[None, None], size=features.shape[-2:], mode="nearest"
            )[0, 0]
            masked_feats = features[0] * binary_ds[None]  # (C, H, W)
            avg_emb = masked_feats.sum(dim=(1, 2)) / (binary_ds.sum() + 1e-6)
            embeddings.append(avg_emb)
        return torch.stack(embeddings)  # (N, C)

    emb_pre = compute_mask_embeddings(image_pre, masks_pre, sam)
    emb_post = compute_mask_embeddings(image_post, masks_post, sam)

    # Bidirectional cosine matching
    sim_matrix = torch.nn.functional.cosine_similarity(
        emb_pre[:, None], emb_post[None, :], dim=-1
    )  # (N_pre, N_post)

    change_mask = np.zeros(image_pre.shape[:2], dtype=np.uint8)

    # Forward: pre masks with no good match in post -> disappeared
    for i, m in enumerate(masks_pre):
        if sim_matrix[i].max() < similarity_threshold:
            change_mask[m["segmentation"]] = 1

    # Backward: post masks with no good match in pre -> appeared
    for j, m in enumerate(masks_post):
        if sim_matrix[:, j].max() < similarity_threshold:
            change_mask[m["segmentation"]] = 1

    return change_mask
```

**Example 3: Multi-Granularity Caption Dataset Generation**

User: "I have bi-temporal satellite pairs with change masks. How do I generate training captions at multiple levels of detail?"

Approach:
1. Compute mask statistics (change percentage, patch count, spatial distribution)
2. Map statistics to natural language descriptors using rule-based binning
3. Combine with one human-written caption to produce 5 captions per pair

Output:
```python
import numpy as np
from scipy import ndimage

def generate_multi_granularity_captions(mask: np.ndarray, human_caption: str) -> list[str]:
    """Generate 5 captions: 1 human + 4 rule-based from mask properties."""
    h, w = mask.shape
    total_pixels = h * w
    change_pixels = mask.sum()
    change_pct = (change_pixels / total_pixels) * 100

    # Label connected components for patch analysis
    labeled, num_patches = ndimage.label(mask)
    patch_sizes = [np.sum(labeled == i) for i in range(1, num_patches + 1)]

    # Bin deforestation percentage
    if change_pct < 5:
        severity = "minimal"
    elif change_pct < 15:
        severity = "moderate"
    elif change_pct < 30:
        severity = "significant"
    else:
        severity = "severe"

    # Spatial distribution descriptor
    if num_patches == 0:
        distribution = "No change is observed in the scene."
    elif num_patches <= 3:
        distribution = f"Change is concentrated in {num_patches} distinct region(s)."
    else:
        distribution = f"Change is scattered across {num_patches} separate patches."

    # Patch size descriptor
    if patch_sizes:
        avg_size = np.mean(patch_sizes)
        max_size = np.max(patch_sizes)
        size_desc = (f"The largest changed area covers {max_size} pixels "
                     f"({max_size/total_pixels*100:.1f}% of the image), "
                     f"with an average patch size of {avg_size:.0f} pixels.")
    else:
        size_desc = "No changed patches are detected."

    captions = [
        human_caption,
        f"The scene shows {severity} deforestation affecting {change_pct:.1f}% of the area.",
        distribution,
        size_desc,
        (f"Analysis reveals {severity} forest loss with {change_pct:.1f}% area change "
         f"distributed across {num_patches} patches. {distribution}"),
    ]
    return captions
```

## Best Practices

- **Do:** Use gradient detachment between detection and captioning loss terms when training multi-task models. Normalize each loss to unit scale before combining, then detach gradients so neither branch dominates. This is critical for stable joint training.
- **Do:** Implement bidirectional matching for zero-shot change detection. Checking only pre-to-post misses newly appeared objects; checking only post-to-pre misses disappeared objects. Always match in both directions.
- **Do:** Provide few-shot examples in the LLM system prompt showing exact tool invocation patterns. This dramatically improves tool selection reliability over zero-shot instructions alone.
- **Do:** Evaluate change detection with per-class IoU, not just MIoU, when masks are heavily imbalanced (common in forestry where <5% of pixels typically change). MIoU can be misleadingly high when the no-change class dominates.
- **Avoid:** Letting the LLM generate arbitrary code without sandboxing. Always execute LLM-generated Python in a restricted namespace with only the registered tool functions available.
- **Avoid:** Using a single similarity threshold across all domains for zero-shot detection. The optimal threshold varies significantly between urban, agricultural, and forest environments. Expose it as a user-configurable parameter.

## Error Handling

- **LLM selects wrong tool:** Validate tool selection by checking that the generated code only calls registered tool names. If an unregistered function is called, re-prompt the LLM with the error message and the tool registry listing. Limit retries to 3 attempts.
- **SAM out-of-memory on large satellite tiles:** Tile the input into overlapping patches (e.g., 1024x1024 with 128px overlap), run SAM per tile, then merge mask proposals using non-maximum suppression on IoU. Stitch the final change mask while averaging overlap regions.
- **Extreme class imbalance in training:** Use weighted binary cross-entropy with change-class weight set to `(total_pixels / change_pixels)` clamped to a maximum of 50. Alternatively, use Dice loss for the detection branch which is inherently robust to class imbalance.
- **Captioning hallucination:** Cross-check generated captions against mask statistics. If the caption claims "significant deforestation" but the mask shows <2% change, flag the inconsistency and regenerate with the statistics injected into the prompt.
- **Temporal registration errors:** Bi-temporal images must be spatially aligned. If change detection produces excessive false positives along edges or linear features, apply image co-registration (e.g., using OpenCV feature matching with RANSAC) as a preprocessing step.

## Limitations

- Zero-shot change detection via SAM is 15-17x slower than supervised inference (~32s vs ~1.8s per image pair on CPU). It is not suitable for real-time or very large-scale batch processing without GPU acceleration.
- The MCI backbone requires paired training data with both masks and captions. If you only have masks (no captions), you must either generate synthetic captions via rule-based methods or train only the detection branch.
- Performance on the Forest-Change dataset (IoU_change: 38%) is significantly lower than on LEVIR-MCI-Trees (IoU_change: 80%), reflecting the difficulty of subtle forest changes at 30m resolution compared to clear structural changes in mixed urban-tree scenes.
- The framework assumes bi-temporal inputs (exactly two images). Multi-temporal time series analysis (3+ dates) requires extending the architecture, e.g., by processing sequential pairs and aggregating change trajectories.
- LLM orchestration adds latency and cost per query. For high-throughput batch processing of thousands of image pairs, bypass the LLM layer and call vision tools directly via scripted pipelines.

## Reference

**Paper:** [Forest-Chat: Adapting Vision-Language Agents for Interactive Forest Change Analysis](https://arxiv.org/abs/2601.14637v1) (Brock, Zhang, Anantrasirichai, 2026). Focus on Section 3 (methodology) for the MCI dual-branch architecture and BI3 layer design, Section 3.3 for the AnyChange zero-shot detection mechanism, and Section 4 for the Forest-Change dataset construction pipeline and evaluation protocol.