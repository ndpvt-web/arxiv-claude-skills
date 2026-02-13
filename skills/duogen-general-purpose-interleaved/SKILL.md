---
name: "duogen-general-purpose-interleaved"
description: "Design and implement interleaved multimodal generation pipelines that alternate between text and image generation using a decoupled MLLM + DiT architecture with special trigger tokens. Use when: 'build an interleaved text-image pipeline', 'create a step-by-step visual guide generator', 'design a multimodal generation architecture', 'implement BOV token routing between LLM and diffusion model', 'build a visual storytelling system', 'architect a two-stage multimodal training pipeline'."
---

# DuoGen: Interleaved Multimodal Generation Pipelines

This skill enables Claude to design and implement interleaved multimodal generation systems based on the DuoGen architecture — a framework that couples a Multimodal LLM (e.g., Qwen2.5-VL) with a Diffusion Transformer (e.g., Cosmos Predict 2.5) through a lightweight connector and a special `<Begin-of-Vision>` (BOV) trigger token. The core insight is a **decoupled two-stage training strategy**: first instruction-tune the MLLM to learn *when* and *what* to generate visually, then freeze the MLLM and align the DiT to its latent representations. This avoids catastrophic forgetting and enables modular component swapping.

## When to Use

- When the user wants to build a system that generates interleaved text and images (e.g., cooking recipes with step photos, how-to guides, visual stories)
- When designing a multimodal architecture that pairs an existing LLM with a separate image generation model (DiT, Stable Diffusion, etc.)
- When implementing a token-based routing mechanism that switches between text generation and image generation modes
- When creating a data curation pipeline for interleaved multimodal instruction-tuning datasets
- When building visual planning or visual reasoning tools that produce intermediate image drafts alongside text explanations
- When the user needs to implement context-conditioned image generation where previously generated images inform subsequent ones

## Key Technique

**Decoupled Architecture with BOV Token Routing.** DuoGen separates concerns cleanly: a 7B-parameter MLLM (Qwen2.5-VL) handles text understanding, reasoning, and deciding *when* to produce an image, while a 2B-parameter DiT (Cosmos Predict 2.5, pretrained on video) handles image synthesis. The MLLM generates text autoregressively until it emits a special `<Begin-of-Vision>` (BOV) token, at which point generation switches to the DiT. The DiT receives two conditioning signals: (1) all preceding images stacked along a temporal axis and VAE-encoded into latents, and (2) the MLLM's hidden states for all tokens before BOV, projected through a lightweight connector into the DiT's cross-attention interface. After image generation, the produced image is appended to the conversation context and text generation resumes.

**Two-Stage Decoupled Training.** Stage 1 instruction-tunes only the MLLM on 298k high-quality interleaved conversations using next-token prediction loss (masking user inputs). This teaches the model to emit BOV tokens at appropriate points and to continue coherent text after image placeholders. Crucially, context-alignment data is excluded in Stage 1 to preserve the MLLM's pretrained capabilities. Stage 2 freezes the MLLM entirely and trains only the connector and DiT using flow-matching loss on curated interleaved sequences — for each sequence, one target image is sampled, a random diffusion timestep is chosen, and the loss is computed against the ground-truth image. This decoupling prevents the text model from degrading while the image model learns to follow its semantic guidance.

**Data Curation at Scale.** The training data combines 268k website-rewritten conversations (from Instructables, eHow, StoryBird) processed through a three-step pipeline (LLM text rewriting, MLLM image captioning, dialogue conversion) with 30k synthetic examples generated from 1,500 seed prompts expanded across 151 subcategories. Alignment data for Stage 2 draws from 5M video clips segmented with scene detection, plus open-source editing datasets.

## Step-by-Step Workflow

1. **Define the interleaved generation schema.** Specify the output format as alternating text segments and image markers. Use a dedicated token (e.g., `<BOV>`) as the boundary signal. Define whether the system produces a fixed number of images or dynamically decides based on content.

2. **Select and configure the MLLM backbone.** Choose a vision-language model (Qwen2.5-VL, LLaVA, InternVL) as the text generation and routing engine. Add the BOV token to the tokenizer vocabulary. Initialize its embedding from the mean of existing special token embeddings.

3. **Select and configure the image generation model.** Choose a DiT or diffusion model (Cosmos Predict, Flux, SD3) for image synthesis. Prefer models pretrained on video data — they handle temporal/contextual coherence between multiple images better than single-image models.

4. **Build the connector module.** Implement a lightweight projection layer (linear or small MLP) that maps MLLM hidden states to the DiT's text-conditioning dimensionality. The connector takes hidden states from all decoder layers corresponding to tokens before BOV and projects them for cross-attention in the DiT.

5. **Curate interleaved training data.** For website data: scrape procedural content (how-tos, recipes, tutorials), strip HTML artifacts with an LLM, caption images with an MLLM, then convert to multi-turn dialogue format. For synthetic data: generate seed prompts across target domains, expand with an LLM, pair with generated or retrieved images.

6. **Execute Stage 1: Instruction-tune the MLLM.** Train the MLLM on interleaved conversations with next-token prediction loss. Mask user turns. The model learns to produce BOV tokens at contextually appropriate points and to write coherent text that references generated images. Do NOT include alignment/editing data here.

7. **Execute Stage 2: Align the DiT.** Freeze the MLLM. Train the connector and DiT on interleaved sequences using flow-matching (or DDPM) loss. For each training sample, randomly select one target image from the sequence, sample a diffusion timestep, and compute loss against the ground truth. Condition on preceding images via temporal VAE latent stacking.

8. **Implement the inference loop.** Build the autoregressive-then-diffusion generation cycle: run MLLM token-by-token until BOV or EOS; on BOV, extract hidden states, stack preceding image latents, run DiT forward pass, decode with VAE, append image to context, resume MLLM generation.

9. **Implement classifier-free guidance for image generation.** During DiT inference, keep visual conditions (preceding images) fixed but drop the final text chunk from MLLM hidden states for the unconditional branch. This improves image-text alignment without sacrificing visual consistency.

10. **Evaluate across modalities.** Measure text quality (fluency, helpfulness), image fidelity (FID, CLIP score), and image-context alignment (entity consistency, style consistency) using both automated metrics and LLM-as-judge evaluation (e.g., GPT-4 for visual-semantic inconsistency detection).

## Concrete Examples

**Example 1: Building an Interleaved Recipe Generator**

```
User: I want to build a system that takes a dish name and generates
a step-by-step recipe with an AI-generated photo for each step.

Approach:
1. Define the output schema:
   - Title (text) -> Ingredients list (text) -> Step 1 text -> <BOV> ->
     Step 1 image -> Step 2 text -> <BOV> -> Step 2 image -> ...
2. Use Qwen2.5-VL-7B as the MLLM backbone. Add <BOV> and <EOV>
   tokens to the tokenizer.
3. Use a DiT model (e.g., Cosmos Predict 2.5) for step images.
4. Build a connector (2-layer MLP, MLLM hidden dim -> DiT cross-attn dim).
5. Curate training data from cooking sites (e.g., MM-Food-100k):
   - Extract step text + step images as interleaved sequences
   - Rewrite into instruction-following dialogues
6. Stage 1: Fine-tune MLLM on recipe dialogues (next-token prediction,
   mask user turns). Model learns to emit <BOV> after each step description.
7. Stage 2: Freeze MLLM, train connector + DiT on recipe image sequences.
   Each step image is conditioned on all previous step images (temporal
   stacking) + MLLM hidden states for the step description.

Output format at inference:
  "Pad Thai"
  -> "Ingredients: rice noodles, shrimp, bean sprouts..."
  -> "Step 1: Soak rice noodles in warm water for 30 minutes." <BOV>
  -> [Generated image: bowl of soaking noodles]
  -> "Step 2: Heat oil in a wok over high heat." <BOV>
  -> [Generated image: wok with oil, consistent kitchen scene]
  -> ...
```

**Example 2: Implementing the BOV Token Routing Logic**

```python
# Core inference loop for interleaved generation

class InterleavedGenerator:
    def __init__(self, mllm, dit, connector, vae, tokenizer):
        self.mllm = mllm          # Frozen after Stage 1
        self.dit = dit             # Trained in Stage 2
        self.connector = connector # Trained in Stage 2
        self.vae = vae
        self.tokenizer = tokenizer
        self.bov_token_id = tokenizer.convert_tokens_to_ids("<BOV>")

    def generate(self, prompt, max_segments=10):
        input_ids = self.tokenizer.encode(prompt)
        generated_images = []
        output_segments = []

        for _ in range(max_segments):
            # Phase 1: Autoregressive text generation until BOV or EOS
            text_ids, hidden_states = self.mllm.generate_until(
                input_ids,
                stop_tokens=[self.bov_token_id, self.tokenizer.eos_token_id]
            )

            output_segments.append(self.tokenizer.decode(text_ids))

            if text_ids[-1] == self.tokenizer.eos_token_id:
                break  # Generation complete

            # Phase 2: Image generation triggered by BOV
            # Extract MLLM hidden states for all tokens before BOV
            text_condition = self.connector(hidden_states[:, :-1, :])

            # Stack all preceding images along temporal axis
            if generated_images:
                visual_condition = self.vae.encode(
                    torch.stack(generated_images, dim=1)  # temporal stack
                )
            else:
                visual_condition = None

            # Run DiT with classifier-free guidance
            image_latent = self.dit.sample(
                text_condition=text_condition,
                visual_condition=visual_condition,
                cfg_scale=7.5
            )
            new_image = self.vae.decode(image_latent)
            generated_images.append(new_image)
            output_segments.append(new_image)

            # Append generated image to MLLM context for next segment
            input_ids = self.mllm.append_image_to_context(
                input_ids, new_image
            )

        return output_segments
```

**Example 3: Designing the Two-Stage Training Pipeline**

```
User: How should I structure training for my interleaved generation model?

Approach:
1. Stage 1 — MLLM Instruction Tuning:
   - Dataset: 298k interleaved conversations (website-rewritten + synthetic)
   - Loss: next-token prediction, user turns masked
   - All MLLM parameters trainable; DiT and connector are NOT initialized yet
   - Key learning objectives:
     a) When to emit <BOV> (after descriptive text that warrants visualization)
     b) How to continue text that references a generated image
     c) Maintaining pretrained text quality and reasoning
   - Exclude image editing / alignment data to avoid capability regression

2. Stage 2 — DiT Context Alignment:
   - Dataset: 5M video-derived frame pairs + open-source edit datasets
     + Stage 1 instruction data (images only)
   - Freeze: MLLM completely frozen
   - Train: Connector (projection layers) + DiT (all parameters or LoRA)
   - Loss: flow-matching on randomly sampled target images
     For each interleaved sequence:
       target_img = random_choice(sequence.images)
       t = random_timestep()
       noise = randn_like(target_img_latent)
       noisy_latent = scheduler.add_noise(target_img_latent, noise, t)
       pred = dit(noisy_latent, t, text_cond, visual_cond)
       loss = mse(pred, noise)  # or velocity prediction
   - Visual conditioning: preceding images VAE-encoded, temporally stacked,
     concatenated with noisy target latent along channel dim

Output: A model where MLLM controls generation flow and DiT faithfully
renders images matching the MLLM's semantic intent and visual context.
```

## Best Practices

- **Do:** Keep the MLLM and DiT as separately pretrained modules. The decoupled design lets you swap either component independently (e.g., upgrade to a better DiT without retraining the MLLM).
- **Do:** Use video-pretrained diffusion models for the image generator. They handle multi-image temporal coherence (consistent characters, scenes, styles across steps) far better than single-image diffusion models.
- **Do:** Stack preceding images along the temporal axis for visual conditioning rather than averaging or concatenating latents — this preserves per-image spatial detail and lets the DiT attend to specific prior frames.
- **Do:** Limit MLLM input image resolution (e.g., max 480px per side) during training to manage memory, while allowing the DiT to generate at higher resolution.
- **Avoid:** Training the MLLM and DiT jointly end-to-end. This causes catastrophic forgetting in the MLLM and gradient conflicts between the text and image losses.
- **Avoid:** Including image editing or alignment data in Stage 1. This degrades the MLLM's pretrained text understanding. Reserve all visual alignment for Stage 2 where the MLLM is frozen.
- **Avoid:** Using the same classifier-free guidance strategy as standard text-to-image. For interleaved generation, drop only the text condition (not visual condition) in the unconditional branch to maintain visual consistency across the sequence.

## Error Handling

- **BOV token never emitted:** If the MLLM generates only text without triggering image generation, check that (1) the BOV token was properly added to the vocabulary and trained in Stage 1, (2) the training data contains sufficient examples where images appear mid-conversation, not just at the end.
- **Generated images inconsistent with prior images:** Verify that visual conditioning is working — all preceding images must be encoded and stacked temporally. Check that the VAE encoder is processing images at consistent resolution and that temporal positional encodings increment correctly per image.
- **Text quality degrades after image generation:** Ensure the generated image is properly tokenized/encoded before being appended to the MLLM context. The MLLM needs its own vision encoder to "see" the generated image, not raw pixel data.
- **Training instability in Stage 2:** If the DiT loss diverges, reduce the learning rate for the connector relative to the DiT. The connector bridges two pretrained representation spaces and can amplify gradients if poorly calibrated.
- **Memory overflow with many images:** Implement a sliding window for visual conditioning — use only the N most recent images (e.g., N=5) rather than the full history when sequences grow long.

## Limitations

- **No real-time generation.** The sequential text-then-image loop means each image adds significant latency (diffusion sampling). Not suitable for interactive chat where sub-second responses are expected.
- **Image quality bounded by DiT capacity.** The 2B DiT produces competent but not photorealistic images. For production-quality visuals, you'd need a larger DiT or post-processing upscaler.
- **Fixed modality pair.** DuoGen addresses text + image interleaving. Extending to audio, video clips, or 3D assets requires additional generator modules and new trigger tokens — the architecture supports it conceptually but no training data or alignment strategy is provided.
- **Data-hungry alignment.** Stage 2 uses 5M+ video clips for alignment. Smaller datasets produce weaker image-context coherence, particularly for maintaining character/scene consistency across many generated images.
- **Single-image generation per BOV.** Each BOV token triggers exactly one image. Generating image grids, comparisons, or multiple simultaneous views requires workarounds (multiple BOV tokens in sequence or post-hoc composition).

## Reference

**Paper:** [DuoGen: Towards General Purpose Interleaved Multimodal Generation](https://arxiv.org/abs/2602.00508v2) (Shi et al., 2026). Focus on Section 3 (Architecture) for the BOV token routing and connector design, Section 4 (Data) for the three-step website rewriting pipeline, and Section 5 (Training) for the two-stage decoupled strategy with loss formulations.