---
name: "vision-representations-artificial-intelligence"
description: "Build autonomous driving safety systems using vision-language models (VLMs) for hazard detection, trajectory planning, and language-constrained motion planning. Use when: 'detect road hazards with CLIP', 'build a VLM-based driving safety pipeline', 'add language constraints to motion planner', 'CLIP hazard screening for autonomous driving', 'integrate vision-language embeddings into trajectory prediction', 'use natural language to constrain vehicle planning'."
---

# Vision-Language Representations for Driving Scene Safety and Autonomous Planning

This skill enables Claude to help build autonomous driving safety systems that leverage vision-language models (VLMs) across three complementary capabilities: (1) lightweight CLIP-based hazard screening that detects diverse road hazards via image-text similarity without explicit object detection, (2) integration of scene-level VLM embeddings into transformer-based trajectory planners, and (3) natural language behavioral constraints that suppress dangerous planning failures. The core insight from Greer et al. (2026) is that VLM representations succeed in driving safety when used for structured semantic grounding — expressing risk, intent, and constraints — rather than naive feature injection into planners.

## When to Use

- When the user wants to build a **road hazard detection system** that generalizes to out-of-distribution objects (animals, debris, fallen trees) without retraining a detector for each category
- When the user asks to **integrate CLIP or SigLIP embeddings** into a vehicle trajectory prediction or planning pipeline
- When the user needs to **add natural language instructions** (e.g., passenger commands, traffic rules) as behavioral constraints on a motion planner
- When the user is working with the **Waymo Open Dataset** or **nuScenes/doScenes** and wants to incorporate vision-language features
- When the user asks how to build a **semantic safety layer** that sits on top of an existing autonomous driving stack
- When the user wants to understand **why naive VLM feature concatenation fails** for trajectory planning and how to fix it with task-informed extraction

## Key Technique

**CLIP-Based Hazard Screening.** The first technique exploits CLIP's zero-shot image-text alignment to create a binary hazard signal. Instead of training an object detector per hazard category, you encode the driving scene image and a set of text prompts describing hazardous conditions (e.g., "a dangerous obstacle on the road", "a pedestrian in the vehicle's path"). The cosine similarity between the image embedding and each hazard prompt embedding produces a scalar score. When this score exceeds a calibrated threshold, a hazard flag is raised. This approach is category-agnostic — it detects novel hazards (construction equipment, animals, unusual debris) that a fixed-class detector would miss — and runs at low latency since CLIP inference is a single forward pass plus a dot product.

**VLM Embeddings for Trajectory Planning.** The second technique integrates global scene embeddings from a VLM (CLIP or SigLIP) into a transformer-based trajectory planner. The critical finding is negative: naively concatenating or adding a global VLM embedding to the planner's input features does **not** improve trajectory accuracy. The embedding captures holistic scene semantics but lacks the spatial and temporal granularity that planners need. The paper motivates **task-informed extraction**: instead of using the raw global embedding, you must extract task-relevant features — e.g., region-specific embeddings for nearby agents, text-guided attention masks highlighting safety-relevant areas, or structured VLM outputs (scene descriptions decomposed into agent states, lane topology, and traffic signals) that align with the planner's representation space.

**Language-Constrained Planning.** The third technique uses natural language instructions as explicit behavioral constraints on a motion planner. Using the doScenes dataset (which augments nuScenes with natural language driving instructions), passenger-style commands like "slow down, there's a cyclist ahead" or "take the right lane after the intersection" are encoded and fed into the planner as conditioning signals. This suppresses rare but catastrophic planning failures — especially in ambiguous scenarios where the planner's learned prior is uncertain — by providing a human-interpretable intent signal that biases the trajectory distribution toward safe, instruction-consistent behavior.

## Step-by-Step Workflow

### Use Case A: Building a CLIP-Based Hazard Screening Module

1. **Select a CLIP model variant** appropriate for your latency budget. Use `openai/clip-vit-base-patch32` for fastest inference (~5ms on GPU) or `openai/clip-vit-large-patch14` for higher accuracy. For production, consider SigLIP (`google/siglip-base-patch16-224`) which improves on CLIP's contrastive loss.

2. **Define a hazard prompt bank** — a list of text descriptions covering hazard categories relevant to your operational domain. Structure prompts in two tiers:
   - **General hazard prompts**: "a dangerous obstacle blocking the road", "an emergency situation on the street", "a hazard requiring immediate braking"
   - **Specific hazard prompts**: "a pedestrian crossing unexpectedly", "debris scattered on the highway", "a vehicle stopped in the travel lane", "an animal on the road"
   - **Negative anchor prompts**: "a clear open road with no obstacles", "normal safe driving conditions"

3. **Pre-compute text embeddings** for all prompts at initialization. These are static and need only be computed once:
   ```python
   text_inputs = clip.tokenize(hazard_prompts).to(device)
   with torch.no_grad():
       text_features = model.encode_text(text_inputs)
       text_features = text_features / text_features.norm(dim=-1, keepdim=True)
   ```

4. **For each incoming frame**, encode the image and compute cosine similarity against all prompt embeddings:
   ```python
   image_features = model.encode_image(preprocessed_frame)
   image_features = image_features / image_features.norm(dim=-1, keepdim=True)
   similarities = (image_features @ text_features.T).squeeze(0)
   ```

5. **Generate a hazard score** by computing the difference between the max hazard-prompt similarity and the max safe-prompt similarity. This differential scoring reduces false positives from CLIP's baseline similarity bias:
   ```python
   hazard_score = similarities[hazard_indices].max() - similarities[safe_indices].max()
   is_hazard = hazard_score > calibrated_threshold
   ```

6. **Calibrate the threshold** on a held-out validation set (e.g., BDD100K or a domain-specific split) by plotting precision-recall curves. Target a recall >= 0.95 for safety-critical applications, accepting some false positives.

7. **Integrate as an asynchronous safety layer** that runs in parallel with the main perception stack. The hazard signal triggers downstream responses (speed reduction, planner re-query, driver alert) without blocking the primary pipeline.

### Use Case B: VLM Embeddings for Trajectory Planning

1. **Extract per-frame VLM embeddings** for training scenes from your dataset (e.g., Waymo Open Dataset). Store these as a precomputed feature alongside existing trajectory data.

2. **Do NOT simply concatenate the global VLM embedding** with your planner's agent-state input. This is the key lesson — global embeddings lack spatial structure and do not improve trajectory prediction metrics (ADE/FDE).

3. **Instead, apply task-informed extraction** using one of these strategies:
   - **Region-of-interest pooling**: Use the VLM's spatial feature map (before global average pooling) and extract features only from regions corresponding to nearby agents and the ego vehicle's planned corridor.
   - **Text-guided attention**: Generate a text prompt describing the planning-relevant scene element (e.g., "the vehicle turning left at the intersection ahead") and use CLIP's cross-attention to produce a spatially-weighted feature map.
   - **Structured scene description**: Use a VLM (e.g., LLaVA, GPT-4V) to generate a structured text description of the scene, then parse it into discrete tokens (agent intents, traffic state, lane structure) that align with the planner's input schema.

4. **Fuse extracted features** into the transformer planner via cross-attention rather than concatenation. Add a cross-attention layer where trajectory queries attend to VLM-derived spatial tokens.

5. **Train with a combined loss**: trajectory regression loss (smooth L1 on waypoints) plus an auxiliary scene-classification loss on the VLM features to maintain semantic grounding.

### Use Case C: Language-Constrained Motion Planning

1. **Collect or generate natural language driving instructions** paired with driving scenes. Use datasets like doScenes, or generate instructions with a VLM by prompting: "Describe what a cautious passenger would say to the driver in this scene."

2. **Encode instructions** with a language model (e.g., the text encoder from CLIP, or a sentence transformer) into a fixed-dimensional embedding.

3. **Condition the planner** on the language embedding via cross-attention or FiLM (Feature-wise Linear Modulation) conditioning:
   ```python
   # FiLM conditioning example
   gamma, beta = language_projection(instruction_embedding).chunk(2, dim=-1)
   conditioned_features = gamma * planner_features + beta
   ```

4. **During training**, pair each scene with its ground-truth instruction. During inference, instructions can come from a passenger interface, a rule-based system encoding traffic laws, or a VLM generating real-time scene descriptions.

5. **Evaluate on safety-critical metrics**: collision rate, off-road rate, and instruction-following accuracy. Focus especially on performance in the tail of the distribution — ambiguous intersections, occluded pedestrians, and unusual road geometries where the language constraint provides the most value.

## Concrete Examples

**Example 1: Zero-Shot Road Hazard Detection API**

User: "Build me a FastAPI endpoint that takes a dashcam image and returns whether there's a road hazard, using CLIP."

Approach:
1. Set up a FastAPI app with a `/detect_hazard` POST endpoint accepting an image upload
2. Load CLIP ViT-B/32 and pre-compute text embeddings for hazard and safe-condition prompts
3. On each request, preprocess the image, compute image embedding, calculate differential hazard score
4. Return JSON with hazard boolean, confidence score, and top matching hazard description

Output:
```python
from fastapi import FastAPI, UploadFile
import clip, torch
from PIL import Image
import io

app = FastAPI()
device = "cuda" if torch.cuda.is_available() else "cpu"
model, preprocess = clip.load("ViT-B/32", device=device)

HAZARD_PROMPTS = [
    "a dangerous obstacle on the road",
    "a pedestrian stepping into traffic",
    "debris blocking the driving lane",
    "a stopped vehicle in the road ahead",
    "an animal crossing the road",
    "a fallen tree on the road",
]
SAFE_PROMPTS = [
    "a clear open road with no obstacles",
    "normal highway driving conditions",
]

all_prompts = HAZARD_PROMPTS + SAFE_PROMPTS
tokens = clip.tokenize(all_prompts).to(device)
with torch.no_grad():
    text_features = model.encode_text(tokens)
    text_features /= text_features.norm(dim=-1, keepdim=True)

HAZARD_THRESHOLD = 0.08  # calibrate on your validation set

@app.post("/detect_hazard")
async def detect_hazard(file: UploadFile):
    image = Image.open(io.BytesIO(await file.read())).convert("RGB")
    image_input = preprocess(image).unsqueeze(0).to(device)

    with torch.no_grad():
        image_features = model.encode_image(image_input)
        image_features /= image_features.norm(dim=-1, keepdim=True)
        sims = (image_features @ text_features.T).squeeze(0)

    hazard_sims = sims[:len(HAZARD_PROMPTS)]
    safe_sims = sims[len(HAZARD_PROMPTS):]
    hazard_score = hazard_sims.max().item() - safe_sims.max().item()

    return {
        "is_hazard": hazard_score > HAZARD_THRESHOLD,
        "hazard_score": round(hazard_score, 4),
        "top_hazard": HAZARD_PROMPTS[hazard_sims.argmax().item()],
        "top_hazard_similarity": round(hazard_sims.max().item(), 4),
    }
```

**Example 2: Language-Conditioned Trajectory Planner Module**

User: "I have a transformer trajectory planner. How do I add natural language conditioning so a passenger can say 'stay in the right lane' and it affects planning?"

Approach:
1. Add a text encoder (frozen CLIP text encoder or sentence-transformers) to encode instructions
2. Project the text embedding to match the planner's hidden dimension
3. Add a FiLM conditioning layer or cross-attention block in the planner
4. Train with paired (scene, instruction, trajectory) data

Output:
```python
import torch
import torch.nn as nn
from sentence_transformers import SentenceTransformer

class LanguageConditionedPlanner(nn.Module):
    def __init__(self, planner_hidden_dim=256, text_embed_dim=384):
        super().__init__()
        # Frozen text encoder
        self.text_encoder = SentenceTransformer("all-MiniLM-L6-v2")
        for p in self.text_encoder.parameters():
            p.requires_grad = False

        # Project text embedding to planner space
        self.text_proj = nn.Sequential(
            nn.Linear(text_embed_dim, planner_hidden_dim),
            nn.ReLU(),
            nn.Linear(planner_hidden_dim, planner_hidden_dim * 2),  # gamma + beta
        )

        # Your existing transformer planner layers
        self.trajectory_encoder = nn.TransformerEncoder(
            nn.TransformerEncoderLayer(d_model=planner_hidden_dim, nhead=8),
            num_layers=4,
        )
        self.trajectory_decoder = nn.Linear(planner_hidden_dim, 2 * 30)  # 30 waypoints, (x, y)

    def forward(self, agent_features, instruction_text):
        # Encode instruction
        with torch.no_grad():
            text_emb = torch.tensor(
                self.text_encoder.encode(instruction_text)
            ).to(agent_features.device)

        # FiLM modulation
        film_params = self.text_proj(text_emb)
        gamma, beta = film_params.chunk(2, dim=-1)
        conditioned = gamma.unsqueeze(0) * agent_features + beta.unsqueeze(0)

        # Plan trajectory
        encoded = self.trajectory_encoder(conditioned)
        waypoints = self.trajectory_decoder(encoded.mean(dim=0))
        return waypoints.view(-1, 30, 2)
```

**Example 3: Evaluating VLM Embedding Integration Strategies**

User: "I tried adding CLIP embeddings to my Waymo planner and ADE got worse. What's going wrong?"

Approach:
1. Diagnose: global CLIP embeddings lack spatial granularity — this is an expected result per Greer et al.
2. Recommend task-informed extraction: switch from global pooling to spatial feature maps
3. Provide concrete refactoring steps

Output:
```
The paper by Greer et al. (2026) specifically found that naively conditioning
planners on global VLM embeddings does not improve (and can degrade) trajectory
accuracy. The global embedding captures scene-level semantics ("intersection
with pedestrians") but lacks the spatial and temporal structure your planner
needs to predict precise waypoints.

Three fixes, in order of implementation complexity:

1. **Spatial feature extraction**: Use CLIP's vision transformer features
   BEFORE the final pooling layer. Extract the patch-level feature grid
   (e.g., 7x7x768 for ViT-B/32) and treat each patch as a spatial token
   for cross-attention in your planner.

2. **Text-guided attention masking**: Compute per-patch similarity to
   planning-relevant prompts ("vehicle ahead", "intersection boundary")
   to create a soft attention mask. Apply this mask to your BEV features
   before the planner consumes them.

3. **Structured VLM output**: Replace the raw embedding with a structured
   scene description from a generative VLM. Parse it into discrete tokens
   (agent_count, traffic_light_state, lane_type) that match your planner's
   feature schema.

Start with option 1 — it requires the smallest architectural change.
```

## Best Practices

- **Do:** Use differential scoring (hazard similarity minus safe-baseline similarity) rather than raw cosine similarity for hazard detection. Raw scores are poorly calibrated across image types.
- **Do:** Pre-compute all text embeddings at initialization. They are deterministic and recomputing them per frame wastes latency.
- **Do:** Calibrate hazard thresholds per operational domain. A threshold tuned on highway data will produce excessive false positives in dense urban scenes.
- **Do:** Use spatial (patch-level) VLM features for trajectory planning, not global embeddings. Align the representation granularity to the task's spatial requirements.
- **Avoid:** Naively concatenating a single global VLM embedding vector to your planner's input. This adds noise without useful spatial information and degrades trajectory metrics.
- **Avoid:** Using VLM-generated free-text scene descriptions directly as planner inputs without structured parsing. Unstructured text introduces variability and latency that safety-critical systems cannot tolerate.

## Error Handling

- **CLIP similarity scores cluster near 0.2-0.3**: This is normal for CLIP. Do not expect clear 0/1 separation. Use differential scoring and calibrated thresholds, not absolute similarity values.
- **Hazard detection false positives on unusual but safe scenes** (e.g., murals, billboards): Add adversarial negative prompts to your safe prompt bank: "a billboard showing a road scene", "artwork depicting traffic". Expand your calibration set to include these edge cases.
- **Trajectory planner diverges after adding VLM features**: Reduce the learning rate for the VLM-feature pathway or freeze VLM projection layers during initial training. The planner's existing features are well-calibrated; new features must be introduced gently.
- **Language encoder produces inconsistent embeddings for paraphrased instructions**: Normalize instruction text during preprocessing. Use a sentence transformer fine-tuned on paraphrase similarity (e.g., `all-MiniLM-L6-v2`) rather than raw CLIP text encoding for instruction conditioning.
- **Latency exceeds budget**: Batch CLIP inference across cameras. Use ONNX/TensorRT export for the CLIP model. For hazard screening, subsample frames (every 3rd frame at 30fps still yields <100ms detection latency).

## Limitations

- **CLIP hazard screening is not a replacement for object detection.** It provides a semantic hazard signal, not localization (no bounding boxes). Use it as a complementary safety layer, not a primary perception module.
- **Global VLM embeddings genuinely do not help trajectory planning** when injected naively. If you cannot implement task-informed extraction (spatial features, structured outputs), skip VLM integration for the planner entirely — it will hurt more than help.
- **Language constraints require paired training data.** Without (scene, instruction, trajectory) triples, the planner cannot learn to follow instructions. The doScenes dataset provides this for nuScenes; equivalent annotations do not exist for all datasets.
- **CLIP's training distribution biases hazard detection.** CLIP was trained on internet image-text pairs, not driving-specific data. Rare driving hazards (e.g., fallen power lines, sinkholes) may have weak representation. Fine-tuning on driving data or using driving-specific VLMs (e.g., DriveVLM) can mitigate this.
- **This approach assumes access to camera frames.** It does not apply to LiDAR-only pipelines. For LiDAR, you would need a point-cloud-to-language model (e.g., ULIP) which is a different architectural problem.

## Reference

Greer, R., Keskar, M., Martinez-Sanchez, A., Roy, P., & Shriram, S. (2026). *Vision and Language: Novel Representations and Artificial Intelligence for Driving Scene Safety Assessment and Autonomous Vehicle Planning.* arXiv:2602.07680. Key takeaway: VLM representations improve driving safety through structured semantic grounding (hazard signals, spatial features, language constraints), not through direct feature injection into planners.