---
name: "forest-chat-adapting-vision-language-agents"
description: "Build LLM-orchestrated agents for bi-temporal satellite image change analysis, combining vision-language models with tool-routing for change detection, captioning, counting, and reasoning. Use when: 'analyze satellite image changes', 'build a forest monitoring agent', 'detect deforestation from imagery', 'create a change detection pipeline', 'VLM agent for remote sensing', 'bi-temporal image analysis system'."
---

# Forest-Chat: LLM-Orchestrated Vision-Language Agents for Satellite Change Analysis

This skill teaches Claude to build **LLM-driven agent systems** that orchestrate vision-language models (VLMs) and foundation models for bi-temporal satellite image change analysis. The core pattern from the Forest-Chat paper is an **LLM controller that routes natural language queries to specialized perception tools** — a dual-branch VLM backbone for joint change detection and captioning, a SAM-based zero-shot change detector with point-prompt support, and analytical functions for quantitative metrics — then synthesizes tool outputs into coherent natural language responses. This architecture generalizes beyond forestry to any domain requiring multi-task image change interpretation.

## When to Use

- When the user asks to build an agent that analyzes differences between two images taken at different times (bi-temporal analysis)
- When designing a tool-routing LLM agent that dispatches to multiple vision/perception backends
- When implementing a change detection + change captioning pipeline for satellite or aerial imagery
- When the user wants to create an interactive system where users can click on image regions to investigate specific changes
- When building a natural language interface over remote sensing image change interpretation (RSICI) tasks
- When the user needs a multi-task VLM system that jointly produces pixel-level masks and semantic descriptions
- When implementing zero-shot change detection using SAM-based foundation models on novel domains

## Key Technique

**LLM-as-orchestrator with specialized perception tools.** Forest-Chat uses an LLM (e.g., GPT-4o-mini or a local model like InternLM2.5-7B-Chat) as a controller that receives natural language queries and generates Python code to invoke a registry of tools. The tools span: (1) a supervised dual-branch VLM for joint change detection and captioning, (2) an AnyChange zero-shot detector extending SAM with bi-temporal latent matching, (3) analytical functions for deforestation area estimation and patch statistics, and (4) optional web search tools. System prompts with few-shot examples guide the LLM to select appropriate tools and format outputs consistently.

**Multi-level Change Interpretation (MCI) backbone.** The VLM uses a Siamese SegFormer-B1 encoder to extract multi-scale features from each temporal image. Shared features feed into two branches: a **change detection branch** using BI3 (Bi-temporal Iterative Interaction) layers with Local Perception Enhancement and Global Difference Fusion Attention, producing pixel-level binary masks via multi-scale CBF modules and deconvolution; and a **change captioning branch** using BI3 layers followed by a Transformer decoder (512-dim embeddings) that generates semantic descriptions. A normalized multi-task loss `L_total = L_det/detach(L_det) + L_cap/detach(L_cap)` balances the two objectives without manual weighting.

**Zero-shot change detection via bi-temporal latent matching.** The AnyChange module extends SAM to detect changes without task-specific training. It generates object masks independently for each temporal image, computes average mask embeddings from the SAM encoder, then uses cosine similarity between embeddings to score semantic changes. Bidirectional matching ensures temporal symmetry. Three operating modes are supported: fully automatic, threshold-filtered semi-automatic, and interactive point-prompted — where user-clicked coordinates seed SAM proposals for targeted analysis.

## Step-by-Step Workflow

1. **Define the tool registry.** Create a Python module that registers each perception tool with a name, description, input schema, and callable. Include at minimum: `detect_changes` (supervised mask generation), `caption_changes` (semantic description), `zero_shot_detect` (SAM-based AnyChange), `count_objects` (connected component counting on masks), `estimate_deforestation` (percentage of changed pixels), and `point_prompt_detect` (user-guided SAM analysis with click coordinates).

2. **Build the LLM orchestration layer.** Implement a controller that accepts a user query plus bi-temporal image paths, constructs a system prompt describing available tools with few-shot examples, sends this to the LLM, parses the LLM's response as Python tool invocations, executes them, and returns synthesized results. Use structured output (JSON or code blocks) to ensure reliable tool invocation parsing.

3. **Implement the Siamese feature extractor.** Load a SegFormer-B1 backbone (e.g., from HuggingFace `nvidia/segformer-b1-finetuned-ade-512-512`) in Siamese configuration — shared weights, separate forward passes for image_t1 and image_t2. Extract multi-scale feature maps at 4 spatial resolutions (1/4, 1/8, 1/16, 1/32 of input).

4. **Build the BI3 interaction layers.** For each feature scale, implement: (a) Local Perception Enhancement via depth-wise convolutions capturing local spatial context, and (b) Global Difference Fusion Attention that computes cross-temporal difference features and applies self-attention to capture long-range change patterns. Stack 3 BI3 layers per branch with residual connections.

5. **Implement the change detection branch.** After BI3 processing, apply Cross-scale Bridge Fusion (CBF) modules across the 4 spatial scales to aggregate multi-resolution change features. Use a progressive deconvolution pathway to upsample to original resolution and produce a binary change mask via sigmoid activation.

6. **Implement the change captioning branch.** After BI3 processing, apply 1x1 convolution to project fused features to 512 dimensions, flatten spatially, and feed into a Transformer decoder (standard autoregressive with cross-attention to visual features) that generates word tokens. Use beam search (width 3-5) at inference.

7. **Integrate AnyChange for zero-shot detection.** Load a SAM model (ViT-H recommended). For each temporal image, run SAM's automatic mask generator to produce object proposals. Compute mask embeddings by averaging the SAM encoder features within each mask region. Match embeddings between t1 and t2 using cosine similarity. Flag proposals with similarity below a threshold (tuned via validation) as changed. Support point-prompt mode by accepting (x, y) coordinates to seed SAM proposals instead of automatic generation.

8. **Implement analytical post-processing tools.** Write functions that operate on the binary change mask: `estimate_deforestation(mask)` computes changed-pixel percentage; `count_patches(mask)` uses connected components to count distinct change regions; `patch_statistics(mask)` returns per-patch area, centroid, and bounding box; `spatial_distribution(mask)` describes whether changes are clustered or dispersed.

9. **Construct the system prompt with few-shot examples.** Write a system prompt that describes each tool's purpose and expected input/output format. Include 3-5 few-shot examples covering different query types: "What changed between these images?" -> detect + caption; "How much forest was lost?" -> detect + estimate_deforestation; "What's happening in this area?" (with coordinates) -> point_prompt_detect + caption.

10. **Wire up the interactive interface.** Build a Gradio, Streamlit, or CLI interface that accepts bi-temporal image pairs, displays them side-by-side, supports click-to-coordinate interaction for point prompts, shows the LLM's tool invocation chain for transparency, and renders both the change mask overlay and natural language response.

## Concrete Examples

**Example 1: Building the tool registry and LLM router**

User: "I want to build an LLM agent that can analyze changes between two satellite images. It should detect changes, describe them, and estimate deforestation."

Approach:
1. Define tool schemas with typed inputs/outputs
2. Build the LLM routing layer with system prompt
3. Implement execution and result synthesis

Output:
```python
from dataclasses import dataclass
from typing import Callable, Any
import json

@dataclass
class Tool:
    name: str
    description: str
    parameters: dict
    fn: Callable

class ForestChatAgent:
    def __init__(self, llm_client, tools: list[Tool]):
        self.llm = llm_client
        self.tools = {t.name: t for t in tools}
        self.system_prompt = self._build_system_prompt()

    def _build_system_prompt(self) -> str:
        tool_descriptions = "\n".join(
            f"- {t.name}: {t.description}\n  Parameters: {json.dumps(t.parameters)}"
            for t in self.tools.values()
        )
        return f"""You are Forest-Chat, an agent for satellite image change analysis.
You have these tools:
{tool_descriptions}

Given a user query about bi-temporal satellite images, respond with a JSON array
of tool calls to execute. Format:
[{{"tool": "tool_name", "args": {{"param": "value"}}}}]

Examples:
User: "What changed between these images?"
Response: [{{"tool": "detect_changes", "args": {{"image_t1": "t1.tif", "image_t2": "t2.tif"}}}},
           {{"tool": "caption_changes", "args": {{"image_t1": "t1.tif", "image_t2": "t2.tif"}}}}]

User: "How much deforestation occurred?"
Response: [{{"tool": "detect_changes", "args": {{"image_t1": "t1.tif", "image_t2": "t2.tif"}}}},
           {{"tool": "estimate_deforestation", "args": {{"mask": "$detect_changes.output"}}}}]
"""

    def query(self, user_input: str, image_t1: str, image_t2: str) -> str:
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": f"Images: {image_t1}, {image_t2}\nQuery: {user_input}"}
        ]
        plan = self.llm.chat(messages)
        tool_calls = json.loads(plan)

        results = {}
        for call in tool_calls:
            tool = self.tools[call["tool"]]
            # Resolve references like "$detect_changes.output"
            args = self._resolve_refs(call["args"], results)
            results[call["tool"]] = tool.fn(**args)

        # Synthesize natural language response from tool outputs
        synthesis_prompt = f"User asked: {user_input}\nTool results: {json.dumps(results, default=str)}\nProvide a clear analysis."
        return self.llm.chat([{"role": "user", "content": synthesis_prompt}])

    def _resolve_refs(self, args: dict, results: dict) -> dict:
        resolved = {}
        for k, v in args.items():
            if isinstance(v, str) and v.startswith("$"):
                ref_tool = v.split(".")[0][1:]
                resolved[k] = results[ref_tool]
            else:
                resolved[k] = v
        return resolved
```

**Example 2: Implementing bi-temporal Siamese feature extraction with BI3 layers**

User: "Help me implement the dual-branch VLM backbone for joint change detection and captioning from bi-temporal images."

Approach:
1. Set up Siamese SegFormer encoder with shared weights
2. Implement BI3 interaction layers (LPE + GFDA)
3. Build detection and captioning branches with balanced loss

Output:
```python
import torch
import torch.nn as nn
from transformers import SegformerModel

class LocalPerceptionEnhancement(nn.Module):
    """Depth-wise conv for local spatial context in bi-temporal features."""
    def __init__(self, dim: int, kernel_size: int = 3):
        super().__init__()
        self.dwconv = nn.Conv2d(dim, dim, kernel_size, padding=kernel_size//2, groups=dim)
        self.norm = nn.LayerNorm(dim)

    def forward(self, x):  # x: (B, H, W, C)
        x = x.permute(0, 3, 1, 2)
        x = self.dwconv(x)
        x = x.permute(0, 2, 3, 1)
        return self.norm(x)

class GlobalDifferenceFusionAttention(nn.Module):
    """Cross-temporal difference features with self-attention."""
    def __init__(self, dim: int, num_heads: int = 8):
        super().__init__()
        self.attn = nn.MultiheadAttention(dim, num_heads, batch_first=True)
        self.norm = nn.LayerNorm(dim)

    def forward(self, feat_t1, feat_t2):
        B, H, W, C = feat_t1.shape
        diff = feat_t1 - feat_t2  # Temporal difference
        diff_flat = diff.reshape(B, H * W, C)
        attended, _ = self.attn(diff_flat, diff_flat, diff_flat)
        return self.norm(attended.reshape(B, H, W, C) + diff)

class BI3Layer(nn.Module):
    """Bi-temporal Iterative Interaction: LPE + GFDA with residual."""
    def __init__(self, dim: int):
        super().__init__()
        self.lpe = LocalPerceptionEnhancement(dim)
        self.gfda = GlobalDifferenceFusionAttention(dim)

    def forward(self, feat_t1, feat_t2):
        feat_t1 = self.lpe(feat_t1) + feat_t1
        feat_t2 = self.lpe(feat_t2) + feat_t2
        fused = self.gfda(feat_t1, feat_t2)
        return fused

class MCIBackbone(nn.Module):
    """Multi-level Change Interpretation: joint detection + captioning."""
    def __init__(self, num_bi3_layers: int = 3, caption_dim: int = 512, vocab_size: int = 10000):
        super().__init__()
        self.encoder = SegformerModel.from_pretrained("nvidia/segformer-b1-finetuned-ade-512-512")
        hidden_dim = 256  # Fused feature dim after projection

        # Detection branch
        self.det_bi3 = nn.ModuleList([BI3Layer(hidden_dim) for _ in range(num_bi3_layers)])
        self.det_head = nn.Sequential(
            nn.ConvTranspose2d(hidden_dim, 128, 2, stride=2),
            nn.ReLU(),
            nn.ConvTranspose2d(128, 64, 2, stride=2),
            nn.ReLU(),
            nn.Conv2d(64, 1, 1),
            nn.Sigmoid()
        )

        # Captioning branch
        self.cap_bi3 = nn.ModuleList([BI3Layer(hidden_dim) for _ in range(num_bi3_layers)])
        self.cap_proj = nn.Conv2d(hidden_dim, caption_dim, 1)
        decoder_layer = nn.TransformerDecoderLayer(d_model=caption_dim, nhead=8, batch_first=True)
        self.caption_decoder = nn.TransformerDecoder(decoder_layer, num_layers=4)
        self.word_embed = nn.Embedding(vocab_size, caption_dim)
        self.output_proj = nn.Linear(caption_dim, vocab_size)

    def forward(self, img_t1, img_t2, captions=None):
        # Siamese encoding (shared weights)
        feat_t1 = self.encoder(img_t1).last_hidden_state  # Simplified
        feat_t2 = self.encoder(img_t2).last_hidden_state
        B, N, C = feat_t1.shape
        H = W = int(N ** 0.5)
        feat_t1 = feat_t1.reshape(B, H, W, C)
        feat_t2 = feat_t2.reshape(B, H, W, C)

        # Detection branch
        det_feat = feat_t1
        for layer in self.det_bi3:
            det_feat = layer(det_feat, feat_t2)
        mask = self.det_head(det_feat.permute(0, 3, 1, 2))

        # Captioning branch
        cap_feat = feat_t1
        for layer in self.cap_bi3:
            cap_feat = layer(cap_feat, feat_t2)
        memory = self.cap_proj(cap_feat.permute(0, 3, 1, 2))
        memory = memory.flatten(2).permute(0, 2, 1)  # (B, HW, D)

        if captions is not None:
            tgt = self.word_embed(captions)
            out = self.caption_decoder(tgt, memory)
            logits = self.output_proj(out)
        else:
            logits = None

        return mask, logits

def balanced_loss(mask_pred, mask_gt, logits, caption_gt):
    """Normalized multi-task loss: L_det/detach(L_det) + L_cap/detach(L_cap)"""
    l_det = nn.functional.binary_cross_entropy(mask_pred, mask_gt)
    l_cap = nn.functional.cross_entropy(logits.reshape(-1, logits.size(-1)), caption_gt.reshape(-1))
    return l_det / l_det.detach() + l_cap / l_cap.detach()
```

**Example 3: Zero-shot change detection with SAM-based point prompts**

User: "I need zero-shot change detection on satellite images I don't have training data for. Users should be able to click on areas of interest."

Approach:
1. Load SAM and generate masks for both temporal images
2. Compute embeddings and match via cosine similarity
3. Support point-prompt mode for user-guided analysis

Output:
```python
import numpy as np
from segment_anything import sam_model_registry, SamAutomaticMaskGenerator, SamPredictor

class AnyChangeDetector:
    """Zero-shot bi-temporal change detection extending SAM."""

    def __init__(self, sam_checkpoint: str, model_type: str = "vit_h", threshold: float = 0.85):
        self.sam = sam_model_registry[model_type](checkpoint=sam_checkpoint)
        self.sam.eval()
        self.auto_generator = SamAutomaticMaskGenerator(self.sam)
        self.predictor = SamPredictor(self.sam)
        self.threshold = threshold

    def _get_mask_embedding(self, image: np.ndarray, mask: np.ndarray) -> np.ndarray:
        """Average SAM encoder features within a mask region."""
        self.predictor.set_image(image)
        features = self.predictor.get_image_embedding().squeeze().cpu().numpy()
        H_f, W_f = features.shape[1], features.shape[2]
        mask_resized = np.array(
            Image.fromarray(mask.astype(np.uint8)).resize((W_f, H_f), Image.NEAREST)
        ).astype(bool)
        masked_features = features[:, mask_resized]
        return masked_features.mean(axis=1) if masked_features.size > 0 else np.zeros(features.shape[0])

    def detect_automatic(self, image_t1: np.ndarray, image_t2: np.ndarray) -> np.ndarray:
        """Fully automatic zero-shot change detection."""
        masks_t1 = self.auto_generator.generate(image_t1)
        masks_t2 = self.auto_generator.generate(image_t2)
        change_mask = np.zeros(image_t1.shape[:2], dtype=np.uint8)

        for m1 in masks_t1:
            emb1 = self._get_mask_embedding(image_t1, m1["segmentation"])
            best_sim = 0.0
            for m2 in masks_t2:
                emb2 = self._get_mask_embedding(image_t2, m2["segmentation"])
                sim = np.dot(emb1, emb2) / (np.linalg.norm(emb1) * np.linalg.norm(emb2) + 1e-8)
                best_sim = max(best_sim, sim)
            if best_sim < self.threshold:
                change_mask[m1["segmentation"]] = 1

        return change_mask

    def detect_with_point(self, image_t1: np.ndarray, image_t2: np.ndarray,
                          point_x: int, point_y: int) -> dict:
        """Point-prompted change detection at a user-specified location."""
        point = np.array([[point_x, point_y]])
        label = np.array([1])  # foreground

        # Get mask at clicked point in t1
        self.predictor.set_image(image_t1)
        masks_t1, scores_t1, _ = self.predictor.predict(point_coords=point, point_labels=label, multimask_output=True)
        best_t1 = masks_t1[scores_t1.argmax()]
        emb_t1 = self._get_mask_embedding(image_t1, best_t1)

        # Get mask at same point in t2
        self.predictor.set_image(image_t2)
        masks_t2, scores_t2, _ = self.predictor.predict(point_coords=point, point_labels=label, multimask_output=True)
        best_t2 = masks_t2[scores_t2.argmax()]
        emb_t2 = self._get_mask_embedding(image_t2, best_t2)

        similarity = float(np.dot(emb_t1, emb_t2) / (np.linalg.norm(emb_t1) * np.linalg.norm(emb_t2) + 1e-8))
        changed = similarity < self.threshold

        return {
            "changed": changed,
            "similarity": similarity,
            "mask_t1": best_t1,
            "mask_t2": best_t2,
            "point": (point_x, point_y)
        }
```

## Best Practices

- **Do:** Use the normalized multi-task loss (`L/detach(L)` for each task) when training joint detection+captioning models — it automatically balances tasks without manual loss weighting.
- **Do:** Train the backbone jointly first, then freeze it and train the detection and captioning branches separately. This two-stage approach prevents one branch from dominating early training.
- **Do:** Include few-shot examples in the LLM system prompt that cover each tool and common multi-tool chains (detect -> caption, detect -> estimate percentage). This dramatically improves tool invocation reliability.
- **Do:** Implement bidirectional matching in zero-shot detection (t1->t2 and t2->t1) to ensure temporal symmetry and reduce false positives from one-directional matching artifacts.
- **Avoid:** Relying solely on zero-shot detection for production accuracy — supervised models outperform zero-shot by 8-40 MIoU points depending on domain. Use zero-shot for exploration and out-of-distribution generalization, supervised for known domains.
- **Avoid:** Using LLM-generated captions as training data for the captioning branch — prompt bias can degrade model quality. Prefer rule-based caption generation from mask properties (area, patch count, spatial distribution) supplemented with limited human annotation.

## Error Handling

- **Unaligned bi-temporal images:** Verify spatial alignment before processing. If images have different resolutions or projections, apply co-registration (e.g., using `rasterio` reprojection) before feeding to the model. Misaligned inputs produce meaningless change masks.
- **Extreme class imbalance (>95% no-change):** Forest change datasets are heavily skewed. Use per-class IoU (not overall accuracy) for evaluation. Apply focal loss or weighted BCE to prevent the model from predicting all-no-change.
- **LLM tool invocation failures:** Validate the LLM's JSON output against the tool schema before execution. Implement retry with a clarified prompt if parsing fails. Log all tool invocations for debugging.
- **SAM memory on large images:** SAM's automatic mask generator on high-resolution satellite tiles (e.g., 4096x4096) can exhaust GPU memory. Tile the input into overlapping 1024x1024 patches, run detection per tile, and stitch results with overlap blending.
- **Cosine similarity threshold sensitivity:** The change/no-change threshold for zero-shot detection is domain-dependent. Tune it on a small validation set (even 10-20 annotated pairs) using Bayesian optimization over the 4 key parameters: similarity threshold, mask confidence threshold, stability score threshold, and minimum mask area.

## Limitations

- The MCI backbone requires bi-temporal image pairs that are spatially aligned and similarly preprocessed (same resolution, normalization). It does not handle arbitrary multi-temporal sequences or unregistered imagery.
- Zero-shot detection with AnyChange is significantly slower (~32s vs ~2s per pair) than supervised detection due to per-mask embedding computation. Not suitable for real-time applications.
- The captioning branch generates relatively short descriptions (1-2 sentences). It does not produce detailed narratives or support multi-turn dialogue about the changes — that capability comes from the LLM synthesis layer.
- Performance is validated primarily on tropical/subtropical forest loss. Applicability to other change types (urban expansion, agricultural rotation, disaster damage) requires domain-specific fine-tuning or validation.
- The point-prompt interface requires a frontend with click-to-coordinate mapping. CLI-only deployments lose this interactive capability.

## Reference

[Forest-Chat: Adapting Vision-Language Agents for Interactive Forest Change Analysis](https://arxiv.org/abs/2601.14637v1) — Focus on Section 3 (Methodology) for the LLM orchestration architecture and tool routing design, Section 3.2 for the MCI dual-branch backbone with BI3 layers, Section 3.3 for AnyChange zero-shot detection, and Section 4 for the Forest-Change dataset construction methodology including rule-based caption generation.