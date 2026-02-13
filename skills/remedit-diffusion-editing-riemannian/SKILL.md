---
name: "remedit-diffusion-editing-riemannian"
description: "Implement Riemannian-geometry-based diffusion image editing pipelines using geodesic latent navigation, dual-SLERP blending, VLM prompt enrichment, and task-specific attention pruning. Use when: 'build a diffusion editing pipeline with Riemannian geometry', 'implement geodesic latent space interpolation for image editing', 'add SLERP-based blending to my diffusion model', 'prune attention tokens for faster diffusion inference', 'set up RemEdit-style image editing', 'implement manifold-aware latent navigation for semantic edits'."
---

# RemEdit: Efficient Diffusion Editing with Riemannian Geometry

This skill enables Claude to implement diffusion-based image editing systems that navigate the latent space as a Riemannian manifold rather than a flat Euclidean space. The core insight from the RemEdit framework is that treating diffusion latent representations as points on a curved manifold -- and computing geodesic paths between them -- yields smoother, more semantically faithful edits than linear interpolation. The skill covers the full pipeline: manifold-aware latent navigation via a Mamba-based learner, dual-SLERP (spherical linear interpolation) blending for fine-grained edit control, VLM-driven prompt enrichment for goal-aware editing, and task-specific attention pruning for inference acceleration without quality loss.

## When to Use

- When the user wants to build or modify a diffusion-based image editing pipeline that needs smooth semantic transitions (e.g., age progression, style transfer, attribute manipulation)
- When the user asks to replace naive linear interpolation in latent space with geometry-aware interpolation (SLERP or geodesic)
- When the user needs to implement attention pruning to accelerate diffusion inference while preserving edit-relevant tokens
- When the user is working with h-space (the intermediate hidden-state space of diffusion U-Nets) and wants principled navigation of edit directions
- When the user wants to combine vision-language model feedback with diffusion editing for more accurate prompt-guided results
- When the user asks to implement a Mamba-based (state-space model) module for learning latent manifold structure
- When the user needs to blend multiple editing objectives smoothly using spherical interpolation techniques

## Key Technique

**Riemannian Manifold Navigation:** Standard diffusion editing methods interpolate linearly between latent codes, but the latent space of diffusion models is not flat -- it has intrinsic curvature. RemEdit treats latent representations (specifically in h-space, the bottleneck features of the U-Net) as points on a Riemannian manifold. A Mamba-based module (a selective state-space model chosen for its linear-time sequence processing) learns the local metric tensor of this manifold from data. With the learned metric, the system computes geodesic paths -- the shortest curves on the manifold surface -- between a source latent and the target edit direction. This produces edits that follow the natural geometry of the learned representation, avoiding off-manifold artifacts like identity drift or semantic inconsistency.

**Dual-SLERP Blending:** RemEdit applies spherical linear interpolation at two levels. The *inner SLERP* operates on the h-space manifold to produce coherent edit direction vectors -- it interpolates between the source representation and the computed edit direction along a great-circle arc, preserving the norm and angular relationships of the feature vectors. The *outer SLERP* controls edit strength by blending the edited latent with the original at a configurable interpolation parameter `t in [0, 1]`. This two-level design decouples *what* to edit from *how much* to edit, giving fine-grained control without retraining.

**Attention Pruning for Speed:** A lightweight pruning head (a small MLP attached to each cross-attention layer) learns per-token importance scores conditioned on the edit task. Tokens scored below a learned threshold are dropped from the attention computation. Because the pruning is task-aware (it considers the edit prompt), it retains tokens in regions relevant to the edit while aggressively pruning background tokens. This achieves up to 50% token reduction with negligible quality degradation, unlike content-agnostic pruning methods that can remove semantically critical tokens.

## Step-by-Step Workflow

1. **Set up the environment and model weights.** Install PyTorch 2.0+, torchvision, and CUDA 11.7+. Clone the RemEdit repository and install dependencies via `conda env create -f environment.yml`. Download pretrained diffusion weights (CelebA-HQ, FFHQ, or AFHQ-Dog checkpoints are supported) and place them in `src/lib/asyrp/pretrained/`.

2. **Pre-compute h-space latent representations.** Run DDIM inversion on source images to obtain their latent trajectories. Extract the h-space features at the U-Net bottleneck for each timestep. Store these as the manifold points that will be navigated during editing.

   ```python
   # Pseudocode: DDIM inversion to h-space
   latents, h_features = ddim_invert(source_image, model, num_steps=1000)
   # h_features shape: [T, C, H, W] where T = diffusion timesteps
   ```

3. **Compute edit directions using attribute vectors.** For a target attribute (e.g., "smile", "young"), compute the direction in h-space using either PCA on labeled attribute pairs or mean-difference vectors between positive/negative attribute clusters. This gives the raw edit direction `delta_h`.

   ```python
   # Mean-difference direction
   delta_h = h_features_positive.mean(0) - h_features_negative.mean(0)
   delta_h = delta_h / delta_h.norm()  # Normalize on the manifold
   ```

4. **Learn the manifold metric with the Mamba module.** Train (or load pretrained) the Mamba-based manifold learner on h-space feature sequences. This module outputs the local Riemannian metric tensor at each point, enabling geodesic computation. The Mamba architecture processes the sequential h-space features with linear complexity, making it efficient for high-resolution latent sequences.

   ```python
   # Mamba manifold learner forward pass
   metric_tensor = mamba_manifold_learner(h_features)  # [T, C, C]
   # Use metric to compute geodesic: solve ODE on manifold
   geodesic_path = compute_geodesic(h_source, delta_h, metric_tensor)
   ```

5. **Apply inner SLERP for edit direction interpolation.** Instead of adding `delta_h` linearly, interpolate along the great circle between the source h-space vector and the target direction using SLERP. This preserves the angular structure and magnitude of the representation.

   ```python
   def slerp(v0, v1, t):
       """Spherical linear interpolation between v0 and v1."""
       v0_norm = v0 / v0.norm()
       v1_norm = v1 / v1.norm()
       omega = torch.acos(torch.clamp((v0_norm * v1_norm).sum(), -1, 1))
       return (torch.sin((1 - t) * omega) / torch.sin(omega)) * v0 + \
              (torch.sin(t * omega) / torch.sin(omega)) * v1

   # Inner SLERP: interpolate edit direction on h-space manifold
   h_edited_direction = slerp(h_source, h_source + delta_h, t_inner)
   ```

6. **Apply outer SLERP for edit strength control.** Blend the fully-edited latent with the original source latent using a second SLERP pass. Adjust `t_outer` from 0.0 (no edit) to 1.0 (full edit) to control intensity.

   ```python
   # Outer SLERP: control edit magnitude
   h_final = slerp(h_source, h_edited_direction, t_outer)
   ```

7. **Enrich the edit prompt with a VLM.** Pass the source image and the user's edit instruction to a vision-language model (e.g., LLaVA, GPT-4V) to generate an enriched prompt that captures spatial context, identifies the region of interest, and clarifies ambiguous instructions. Feed this enriched prompt to the diffusion model's text encoder.

   ```python
   enriched_prompt = vlm.generate(
       image=source_image,
       instruction=f"Describe how to edit this image to: {user_prompt}. "
                   f"Be specific about which regions to change and preserve."
   )
   text_embeddings = text_encoder(enriched_prompt)
   ```

8. **Apply task-specific attention pruning.** Attach the pruning head to each cross-attention layer. During inference, compute token importance scores conditioned on the edit prompt, then mask out low-scoring tokens before the attention operation. Set the pruning ratio (recommended: 30-50%).

   ```python
   class PruningHead(nn.Module):
       def __init__(self, dim, hidden=64):
           super().__init__()
           self.mlp = nn.Sequential(
               nn.Linear(dim, hidden), nn.GELU(), nn.Linear(hidden, 1)
           )

       def forward(self, tokens, edit_context):
           scores = self.mlp(tokens + edit_context).squeeze(-1)
           mask = scores > scores.quantile(1 - pruning_ratio)
           return tokens[mask], mask
   ```

9. **Run the forward diffusion pass with edited features.** Replace the original h-space features with `h_final` during the DDIM forward (denoising) pass. Apply the pruning masks at each attention layer. Decode the final latent to produce the edited image.

10. **Evaluate output quality.** Measure LPIPS (perceptual distance to source for identity preservation), CLIP-score (alignment between edit prompt and result), and FID (distributional quality). Use the provided evaluation scripts: `python main.py --lpips --config $config --edit_attr $attr --n_train_img 100`.

## Concrete Examples

**Example 1: Adding a smile to a face image with geodesic editing**

User: "I want to edit a CelebA face image to add a smile while preserving identity. Use Riemannian geodesic interpolation instead of linear editing."

Approach:
1. Load the pretrained CelebA-HQ diffusion model and Mamba manifold learner
2. Run DDIM inversion on the source face to get h-space features (1000 inversion steps)
3. Load the precomputed "smile" direction vector from the attribute library
4. Compute the geodesic path from h_source along the smile direction using the learned metric tensor
5. Apply inner SLERP with `t_inner=0.8` to follow the manifold curvature
6. Apply outer SLERP with `t_outer=0.6` for moderate smile intensity
7. Run DDIM forward pass with modified h-features, decode to image

Output:
```python
# Configuration
config = load_config("config/celeba.yml")
config.edit_attr = "smiling"
config.t_inner = 0.8    # Direction fidelity on manifold
config.t_outer = 0.6    # Edit strength
config.n_inv_step = 1000
config.n_train_img = 50

# Pipeline execution
source_latent, h_features = ddim_invert(source_img, model, config)
delta_h = load_attribute_direction("smiling")
metric = mamba_learner(h_features)
geodesic = compute_geodesic(h_features, delta_h, metric)
h_edited = dual_slerp(h_features, geodesic, config.t_inner, config.t_outer)
edited_image = ddim_forward(source_latent, h_edited, model)
# Result: natural smile added, identity preserved, no artifacts at mouth boundary
```

**Example 2: Accelerating a diffusion editing pipeline with task-aware pruning**

User: "My diffusion editing pipeline is too slow for production. I need to prune attention tokens without degrading edit quality."

Approach:
1. Identify the cross-attention layers in the U-Net (typically 16 layers in standard architectures)
2. Attach a PruningHead MLP to each layer (adds < 0.1% parameters)
3. Train pruning heads on 1000 edit pairs with a combined loss: reconstruction quality + edit fidelity + sparsity regularization
4. At inference, set pruning ratio to 0.4 (40% token removal) -- the sweet spot from RemEdit ablations
5. Profile before/after to verify speedup

Output:
```python
# Attach pruning heads to existing U-Net
for i, layer in enumerate(unet.cross_attention_layers):
    layer.pruning_head = PruningHead(dim=layer.dim, hidden=64)

# Training loop for pruning heads (freeze rest of model)
optimizer = torch.optim.Adam(
    [p for l in unet.cross_attention_layers for p in l.pruning_head.parameters()],
    lr=1e-4
)
for source, target, prompt in edit_dataset:
    edited = pipeline(source, prompt, pruning_ratio=0.4)
    loss = lpips_loss(edited, target) + 0.1 * sparsity_loss(pruning_masks)
    loss.backward()
    optimizer.step()

# Inference with pruning enabled
edited = pipeline(source, prompt, pruning_ratio=0.4)
# Result: ~1.8x speedup with < 0.02 LPIPS degradation vs unpruned
```

**Example 3: VLM-enriched prompt editing for ambiguous instructions**

User: "Make the person look more professional" -- this is vague. Use VLM enrichment to clarify the edit.

Approach:
1. Pass the source image + vague prompt to a VLM
2. VLM analyzes the image content and generates a specific edit description
3. Use the enriched prompt for more targeted diffusion editing
4. Apply dual-SLERP with conservative outer strength since the edit is subjective

Output:
```python
# VLM prompt enrichment
vague_prompt = "Make the person look more professional"
enriched = vlm.generate(
    image=source_img,
    instruction=(
        f"The user wants to: {vague_prompt}. "
        "Analyze the current image and describe specific visual changes needed. "
        "Mention clothing, grooming, lighting, and background changes."
    )
)
# VLM output: "Add a dark blazer over the t-shirt, smooth the hair to a neat
# side-part style, slightly brighten the face lighting to look like a studio
# headshot, and blur the cluttered background to a neutral gradient."

text_emb = text_encoder(enriched)
h_edited = dual_slerp(h_source, compute_geodesic(h_source, delta_h, metric), 0.7, 0.5)
edited_image = ddim_forward(latent, h_edited, model, text_emb)
```

## Best Practices

- **Do:** Always use SLERP instead of linear interpolation (LERP) when blending latent vectors in diffusion models. SLERP preserves vector norms and angular relationships, preventing the "washed out" artifacts of LERP at intermediate interpolation values.
- **Do:** Pre-compute and cache h-space features for your source images. DDIM inversion is the most expensive step and its output is deterministic -- reuse it across multiple edits of the same source.
- **Do:** Start with conservative edit strengths (`t_outer` around 0.3-0.5) and increase gradually. Geodesic navigation reduces artifacts but extreme edits still risk identity loss.
- **Do:** Train the pruning heads on your specific edit domain (faces, scenes, objects) rather than using a generic checkpoint. Task-specificity is the entire point of the pruning mechanism.
- **Avoid:** Pruning above 60% of tokens. RemEdit shows that quality degrades sharply beyond 50% pruning, and 30-40% is the practical sweet spot for most tasks.
- **Avoid:** Skipping the VLM enrichment step for ambiguous prompts. The gap between "make it look better" and a spatially-grounded edit description is where most editing failures originate.
- **Avoid:** Using the Mamba manifold learner without sufficient training data. At minimum, use 1000+ images from the target domain to learn meaningful manifold structure. On tiny datasets, fall back to SLERP-only (without learned geodesics).

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Identity drift after editing | `t_outer` too high or geodesic overshooting | Reduce `t_outer` to 0.3-0.4; verify DDIM inversion step count is >= 500 |
| Artifacts at edit boundaries | Linear interpolation used instead of SLERP | Replace all `lerp()` calls with `slerp()`; check that vector norms are preserved |
| Pruning causes blurry outputs | Pruning ratio too aggressive or pruning head undertrained | Reduce pruning ratio to 0.3; train pruning heads for more iterations with lower learning rate |
| VLM enrichment produces irrelevant edits | VLM prompt template not grounded | Add explicit spatial grounding instructions to VLM prompt; include "preserve regions that are not mentioned" |
| Mamba module produces NaN metric tensors | Numerical instability in metric computation | Add epsilon (1e-6) to metric tensor diagonal; clamp eigenvalues to positive range |
| DDIM inversion reconstruction is poor | Insufficient inversion steps | Increase `n_inv_step` from 50 to 500-1000; verify classifier-free guidance scale matches training |
| Out of VRAM on inference | Full attention + high resolution | Enable attention pruning at 0.4 ratio; use gradient checkpointing; reduce batch size to 1 |

## Limitations

- **Domain-specific training required.** The Mamba manifold learner and pruning heads must be trained per-domain (faces, churches, animals). There is no universal pretrained checkpoint.
- **Resolution constraints.** Supported resolutions are 256x256 for the provided checkpoints. Higher resolutions require retraining the manifold learner and pruning heads.
- **DDIM-only pipeline.** The h-space navigation is tied to DDIM sampling. It does not directly transfer to other samplers (DPM-Solver, Euler) without adaptation.
- **No text-to-image generation.** RemEdit is an editing framework, not a generation framework. It requires an existing source image to edit.
- **GPU requirements.** Minimum 32GB VRAM for training, ~16GB for inference with pruning enabled. Not suitable for consumer GPUs without significant model surgery.
- **Attribute directions are supervised.** Computing edit directions requires labeled attribute pairs (positive/negative samples), limiting edits to predefined attributes unless combined with CLIP-guided direction discovery.

## Reference

**Paper:** [RemEdit: Efficient Diffusion Editing with Riemannian Geometry](https://arxiv.org/abs/2601.17927v1) -- Adhikarla & Davison, 2026. Focus on Section 3 (Riemannian Manifold Navigation and Dual-SLERP formulation), Section 4 (Task-Specific Attention Pruning architecture), and the ablation studies in Section 5 comparing geodesic vs. Euclidean paths and pruning ratio vs. quality tradeoffs.

**Code:** [github.com/eashanadhikarla/RemEdit](https://www.github.com/eashanadhikarla/RemEdit) (MIT License). Built on top of Asyrp and DiffusionCLIP.