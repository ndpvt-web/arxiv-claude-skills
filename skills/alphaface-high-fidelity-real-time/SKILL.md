---
name: "alphaface-high-fidelity-real-time"
description: "Search and retrieve information about AlphaFace, a real-time face-swapping architecture that uses CLIP contrastive losses and cross-adaptive identity injection for pose-robust face swapping. Use when the user asks about 'face swapping with extreme poses', 'real-time face swap architecture', 'CLIP-based identity preservation', 'pose-robust face generation', 'AlphaFace implementation', or 'contrastive loss for face swapping'."
---

# AlphaFace: High-Fidelity Real-Time Pose-Robust Face Swapping

This skill equips Claude to help users understand, implement, debug, and extend the AlphaFace face-swapping architecture. AlphaFace combines an ArcFace identity encoder, a Cross-Adaptive Identity Injection (CAII) fusion module, and CLIP-informed visual/textual semantic contrastive losses to achieve state-of-the-art face swapping that remains robust under extreme facial poses while running at ~41.5 FPS (24.1 ms per frame on an A6000 GPU). Claude can guide users through setting up the official codebase, designing custom training pipelines, adapting the CAII module for other identity-transfer tasks, and diagnosing common failure modes.

## When to Use

- When the user wants to set up or run the AlphaFace official repository (`andrewyu90/Alphaface_Official`) for face-swapping inference or training.
- When the user needs to implement a Cross-Adaptive Identity Injection (CAII) module that bidirectionally normalizes source identity and target attribute features via AdaIN.
- When the user asks how to use CLIP image and text embeddings as contrastive losses to improve identity preservation or attribute fidelity in a generative model.
- When the user is building a face-swapping pipeline and needs to handle extreme head poses (profile views, large pitch/yaw angles) without quality degradation.
- When the user wants to compare real-time face-swapping approaches (AlphaFace vs. FaceDancer, SimSwap, InfoSwap, DiffSwap) on metrics like ID retrieval, pose error, expression error, or FID.
- When the user needs to generate text descriptions of facial attributes using a vision-language model (InternVL3-14B) as part of a training pipeline.
- When the user wants to adapt the CAII or CLIP contrastive loss strategy to a related domain such as hairstyle transfer, expression reenactment, or virtual try-on.

## Key Technique

AlphaFace's core insight is that existing face-swap methods either rely on explicit geometric features (3DMMs, landmarks) which add dependencies and computational cost, or use diffusion models which are too slow for real-time use. AlphaFace instead leverages **semantic-level supervision** via CLIP embeddings to enforce both identity consistency and attribute preservation without geometric intermediaries. The architecture has three stages: (1) an ArcFace-based source identity encoder extracts a compact identity code `c_s`, (2) a fusion encoder with stacked CAII blocks merges source identity into target features via bidirectional AdaIN, and (3) a progressive deconvolutional generator reconstructs the swapped face at high resolution.

The **Cross-Adaptive Identity Injection (CAII)** block is the architectural novelty. Standard approaches inject source identity into target features with one-directional AdaIN, which risks leaking irrelevant source attributes (background, pose, lighting). CAII performs bidirectional normalization: it normalizes target features using source identity statistics *and* normalizes source identity features using target statistics. The two normalized streams are combined via residual connections and element-wise operations, producing a fused representation that carries source identity but respects target geometry and attributes.

The **CLIP contrastive losses** are the training novelty. A vision-language model (InternVL3-14B) generates 70-word text descriptions of each target image's pose, background, accessories, and occlusions. During training, two cosine-similarity losses are computed: (1) **visual semantic contrastive loss** — the CLIP image embedding of the swapped face should be close to the source face (identity axis), and (2) **textual semantic contrastive loss** — the CLIP text embedding of the target description should match the swapped face (attribute axis). Critically, the textual loss is only activated when the swapped image is *less* consistent with the target description than the original target, preventing over-regularization. Combined with standard identity swap loss (ArcFace cosine similarity), attribute preservation losses (masked reconstruction, cyclic reconstruction, VGG16 perceptual), and adversarial loss, this yields a total objective with balancing weights: `lambda_ID=10.0, lambda_AP=0.5, lambda_Adv=1.0, lambda_CLIP=1.0`.

## Step-by-Step Workflow

### Setting Up AlphaFace

1. **Clone the repository and install dependencies.** Clone `https://github.com/andrewyu90/Alphaface_Official.git`, then install requirements with `pip install -r requirement.txt`. Verify PyTorch with CUDA support is available (`torch.cuda.is_available()`).

2. **Download pretrained weights.** Obtain the main AlphaFace generator checkpoint and the ArcFace backbone model from the Google Drive links in the README. Place the main model in the project root and the ArcFace model in `Models/`.

3. **Run inference using the evaluation script.** Configure the model path in `configs/alphaface_eval_demo.py`, then execute `./eval.sh`. This loads a source image (identity donor) and target image (pose/attribute donor) and produces the swapped output in `Eval/save1/`.

### Training a Custom Model

4. **Prepare your dataset as image triplets.** Each training sample requires: a source face image, a target face image, and a face mask for the target. Organize these alongside precomputed 70-word text descriptions of each target image's pose, background, and occlusions.

5. **Generate text descriptions with a vision-language model.** Use InternVL3-14B (or a comparable open VLM) with the prompt: `"Describe pose, background, facial accessories, and all obstacles covering the face area in the given face image. Only 70 words are allowed."` Store these descriptions alongside your image triplets.

6. **Configure training hyperparameters.** In the config file, set: Adam optimizer, learning rate `0.01` decayed by `0.9` every 5 epochs, batch size `8`, 50 epochs, and loss weights `lambda_ID=10.0, lambda_AP=0.5, lambda_Adv=1.0, lambda_CLIP=1.0`. Training uses two A6000 GPUs (48 GB VRAM each).

7. **Launch training with the CLIP contrastive losses.** Execute `./train_clip.sh`. Monitor the individual loss components — identity swap loss should decrease steadily; the CLIP textual loss will activate intermittently (only when the swapped image underperforms the original target on attribute consistency).

8. **Evaluate on standard benchmarks.** Run evaluation on FF++ (in-distribution), MPIE (multi-pose), and LPFF (large-pose) datasets. Track: ID retrieval (higher is better), pose error and expression error (lower is better), FID (lower is better), and inference latency in ms.

### Adapting the CAII Module for Other Tasks

9. **Extract the CAII block for reuse.** The CAII module is architecture-agnostic — it takes two feature tensors and bidirectionally normalizes them via AdaIN. To reuse it, instantiate CAII with the channel dimensions of your source and target feature maps, then insert it between your encoder and decoder.

10. **Add CLIP contrastive losses to an existing generative pipeline.** Load pretrained CLIP image and text encoders. Compute cosine similarity between your generated output's CLIP image embedding and (a) the source's CLIP image embedding for identity, and (b) the target's CLIP text description embedding for attributes. Apply the conditional activation gate: only backpropagate the textual loss when `sim(swapped, text) < sim(target, text)`.

## Concrete Examples

**Example 1: Running inference on a source-target pair**

User: "I cloned the AlphaFace repo. How do I swap the face from `source.jpg` onto `target.jpg`?"

Approach:
1. Verify that pretrained weights are downloaded and placed correctly (main model in root, ArcFace in `Models/`).
2. Edit `configs/alphaface_eval_demo.py` to point `source_path` and `target_path` to the user's images.
3. Run `./eval.sh` (or equivalently `python eval.py --config configs/alphaface_eval_demo.py`).
4. The swapped output appears in `Eval/save1/`.

Output:
```
$ python eval.py --config configs/alphaface_eval_demo.py
Loading ArcFace model from Models/arcface.pth ... done
Loading AlphaFace generator ... done
Processing source.jpg -> target.jpg
  Identity code extracted: 512-d vector
  CAII fusion: 4 blocks applied
  Generator: 256x256 output produced
Saved to Eval/save1/swapped_0001.png
Inference time: 24.1 ms (41.5 FPS)
```

**Example 2: Implementing the CAII block from scratch in PyTorch**

User: "I want to add the bidirectional AdaIN identity injection from AlphaFace to my own face generator. Can you write the CAII module?"

Approach:
1. Implement AdaIN as a function that normalizes input features then applies affine transform from style features.
2. Build the CAII block with two AdaIN paths (target-normalized-by-source and source-normalized-by-target).
3. Combine via residual addition and element-wise multiplication.

Output:
```python
import torch
import torch.nn as nn

class AdaIN(nn.Module):
    """Adaptive Instance Normalization."""
    def forward(self, content, style_mu, style_sigma):
        # content: (B, C, H, W), style_mu/sigma: (B, C)
        mu = content.mean(dim=[2, 3], keepdim=True)
        sigma = content.std(dim=[2, 3], keepdim=True) + 1e-6
        normalized = (content - mu) / sigma
        return style_sigma.unsqueeze(-1).unsqueeze(-1) * normalized + style_mu.unsqueeze(-1).unsqueeze(-1)

class CAII(nn.Module):
    """Cross-Adaptive Identity Injection block."""
    def __init__(self, channels, id_dim=512):
        super().__init__()
        self.adain = AdaIN()
        # Affine projections for source identity -> (mu, sigma)
        self.src_to_affine = nn.Linear(id_dim, channels * 2)
        # Affine projections for target features -> (mu, sigma)
        self.tgt_pool = nn.AdaptiveAvgPool2d(1)
        self.tgt_to_affine = nn.Linear(channels, id_dim * 2)
        # Projection back from id_dim to spatial channels
        self.id_proj = nn.Linear(id_dim, channels)
        self.gate = nn.Sigmoid()

    def forward(self, z_target, c_source):
        """
        z_target: (B, C, H, W) target encoder features
        c_source: (B, id_dim) source identity code from ArcFace
        """
        # Path 1: normalize target features with source identity statistics
        src_affine = self.src_to_affine(c_source)
        src_mu, src_sigma = src_affine.chunk(2, dim=1)
        z_t_norm = self.adain(z_target, src_mu, src_sigma.exp())

        # Path 2: normalize source identity with target statistics
        tgt_pooled = self.tgt_pool(z_target).flatten(1)
        tgt_affine = self.tgt_to_affine(tgt_pooled)
        tgt_mu, tgt_sigma = tgt_affine.chunk(2, dim=1)
        c_s_expanded = c_source
        c_norm = tgt_sigma.exp() * ((c_s_expanded - c_s_expanded.mean(dim=1, keepdim=True))
                 / (c_s_expanded.std(dim=1, keepdim=True) + 1e-6)) + tgt_mu
        c_spatial = self.id_proj(c_norm).unsqueeze(-1).unsqueeze(-1)

        # Combine with residual and gating
        gate_val = self.gate(c_spatial)
        fused = z_t_norm * gate_val + z_target * (1 - gate_val)
        return fused
```

**Example 3: Adding CLIP contrastive losses to an existing training loop**

User: "I have a face-swap model already training with L1 + perceptual + adversarial losses. How do I add the AlphaFace CLIP losses?"

Approach:
1. Load pretrained CLIP model (e.g., `openai/clip-vit-large-patch14`).
2. Precompute text descriptions of target images using InternVL3-14B or similar VLM.
3. Add visual contrastive loss (swapped vs. source identity) and textual contrastive loss (swapped vs. target description) with conditional activation.

Output:
```python
import clip
import torch

clip_model, clip_preprocess = clip.load("ViT-L/14", device="cuda")
clip_model.eval()
for p in clip_model.parameters():
    p.requires_grad = False

def clip_contrastive_losses(swapped_img, source_img, target_img, target_text_tokens):
    """
    swapped_img, source_img, target_img: (B, 3, 224, 224) preprocessed for CLIP
    target_text_tokens: (B, 77) tokenized target descriptions
    """
    with torch.no_grad():
        src_emb = clip_model.encode_image(source_img)
        tgt_img_emb = clip_model.encode_image(target_img)
        tgt_txt_emb = clip_model.encode_text(target_text_tokens)

    swap_emb = clip_model.encode_image(swapped_img)

    # Normalize embeddings
    swap_emb = swap_emb / swap_emb.norm(dim=-1, keepdim=True)
    src_emb = src_emb / src_emb.norm(dim=-1, keepdim=True)
    tgt_img_emb = tgt_img_emb / tgt_img_emb.norm(dim=-1, keepdim=True)
    tgt_txt_emb = tgt_txt_emb / tgt_txt_emb.norm(dim=-1, keepdim=True)

    # Visual contrastive: swapped should match source identity
    visual_loss = 1.0 - (swap_emb * src_emb).sum(dim=-1).mean()

    # Textual contrastive: swapped should match target attributes
    sim_swap_text = (swap_emb * tgt_txt_emb).sum(dim=-1)
    sim_tgt_text = (tgt_img_emb * tgt_txt_emb).sum(dim=-1)
    # Conditional activation: only penalize when swapped < target
    textual_loss = torch.clamp(sim_tgt_text - sim_swap_text, min=0.0).mean()

    return visual_loss, textual_loss

# In training loop:
# vis_loss, txt_loss = clip_contrastive_losses(swapped, source, target, text_tokens)
# total_loss = ... + 1.0 * (vis_loss + txt_loss)
```

## Best Practices

- **Do:** Use the conditional activation gate on the textual contrastive loss. Without it, the CLIP text loss can over-constrain the generator and produce blurry outputs when the swapped image already matches target attributes well.
- **Do:** Freeze the CLIP encoders during training. They serve as fixed semantic anchors — fine-tuning them collapses the contrastive signal.
- **Do:** Keep the VLM-generated text descriptions concise (70 words max). Longer descriptions introduce noise from hallucinated details that degrade the textual loss signal.
- **Do:** Stack multiple CAII blocks (the paper uses 4) rather than a single large one. Progressive injection produces smoother identity-attribute fusion.
- **Avoid:** Skipping the bidirectional normalization in CAII and using only source-to-target AdaIN. The reverse path (target-to-source) is what prevents source background/lighting leakage into the output.
- **Avoid:** Setting `lambda_ID` too low relative to `lambda_CLIP`. The ArcFace identity loss (`lambda_ID=10.0`) anchors identity transfer; CLIP losses (`lambda_CLIP=1.0`) refine it. Inverting this ratio causes identity drift.

## Error Handling

- **CUDA out of memory during training:** Reduce batch size from 8 to 4, or use gradient accumulation over 2 steps. The CLIP encoders (frozen, no gradients) still consume ~2 GB VRAM for ViT-L/14.
- **Identity leak from source (background/pose bleeds through):** Verify the CAII block includes the reverse normalization path. If using a custom implementation, check that target statistics are applied to source features before fusion.
- **Poor attribute preservation (wrong pose, missing glasses):** Inspect the VLM text descriptions — if they omit key attributes, the textual contrastive loss cannot enforce them. Regenerate descriptions with a more explicit prompt or a larger VLM.
- **CLIP loss not decreasing:** Confirm that images fed to CLIP are preprocessed with the correct normalization (CLIP's own `preprocess` transform, not ImageNet defaults). Mismatched normalization produces meaningless embeddings.
- **Inference slower than expected (>30 ms):** Ensure the CLIP encoders are not loaded during inference — they are training-only. The inference path is ArcFace encoder + CAII fusion + generator only.

## Limitations

- **Identity ceiling:** AlphaFace achieves 98.77 ID retrieval on FF++, slightly below FaceDancer (98.84). For applications requiring maximum identity fidelity above all else and where latency is not a concern, slower methods may be preferable.
- **Resolution:** The generator outputs 256x256 faces. For higher resolution (512+), a super-resolution postprocessing stage or architectural modifications to the generator are needed.
- **Occlusion handling:** While the VLM descriptions capture occlusions (hands, masks, sunglasses), heavy occlusions still degrade output quality because the generator has limited information to reconstruct occluded identity regions.
- **Training data dependency:** The CLIP contrastive losses require precomputed text descriptions for every target image, adding a preprocessing step that scales linearly with dataset size and requires VLM inference.
- **Single-face assumption:** AlphaFace processes one source-target pair at a time. Multi-face scenes require a face detection and cropping preprocessing step (e.g., RetinaFace or MTCNN) before passing individual faces to the pipeline.

## Reference

**Paper:** [AlphaFace: High Fidelity and Real-time Face Swapper Robust to Facial Pose](https://arxiv.org/abs/2601.16429v1) (Yu et al., 2026). Key sections: Section 3.2 for the CAII module architecture, Section 3.3 for the CLIP contrastive loss formulations with conditional activation, and Table 1 for benchmark comparisons across FF++/MPIE/LPFF.

**Code:** [https://github.com/andrewyu90/Alphaface_Official](https://github.com/andrewyu90/Alphaface_Official) — MIT licensed. See `train_clip.py` for the training loop with CLIP losses, `Models/` for the CAII and generator implementations, and `configs/` for hyperparameter configurations.