---
name: "ar-omni-unified-autoregressive-any-to-any"
description: |
  Design and implement unified autoregressive multimodal systems that handle text, image, and speech generation through a single Transformer decoder with a shared token vocabulary. Applies AR-Omni's techniques: task-aware loss reweighting, token-level perceptual alignment, and finite-state decoding.
  Trigger phrases:
  - "Build a unified multimodal generation pipeline"
  - "Design an any-to-any autoregressive model"
  - "Implement task-aware loss reweighting for multimodal training"
  - "Create a single-decoder system for text, image, and speech"
  - "Add perceptual alignment loss to discrete token prediction"
  - "Implement finite-state decoding for mixed generation tasks"
---

# AR-Omni: Unified Autoregressive Any-to-Any Generation

This skill enables Claude to help design and implement unified autoregressive multimodal systems where a single Transformer decoder generates text, images, and speech through next-token prediction over a shared vocabulary. It applies three key techniques from AR-Omni: **task-aware loss reweighting** to handle modality imbalance, **token-level perceptual alignment** to improve visual fidelity in discrete image generation, and **finite-state decoding** to adaptively switch between greedy and sampling strategies per task type.

## When to Use

- When the user wants to build a single model that handles multiple input/output modalities (text-to-image, speech-to-text, image captioning, TTS) without separate expert decoders
- When training a multimodal model and one modality's loss dominates due to different token budgets across modalities
- When discrete image token generation (VQ-based) produces blurry or incoherent results and the user needs a perceptual alignment objective
- When the user needs a decoding strategy that automatically switches between deterministic and creative modes depending on the task
- When designing a vocabulary and tokenization scheme that unifies text BPE, image VQ codes, and speech codec tokens into a single sequence
- When implementing streaming speech generation with low first-token latency in an autoregressive framework

## Key Technique

**Unified Autoregressive Multimodal Modeling.** AR-Omni concatenates text, image, and speech tokens into a single interleaved sequence and trains a standard causal Transformer to predict the next token. Text uses SentencePiece BPE, images use a scene-aware VQ tokenizer producing 1D discrete codes, and speech uses WavTokenizer (a purely acoustic discrete tokenizer). Special boundary tokens like `<boi>/<eoi>` and `<boa>/<eoa>` mark modality transitions. This eliminates the need for separate diffusion models, vocoders, or expert decoders -- the entire system is one autoregressive pass.

**Three Practical Fixes for Unified Training.** First, *task-aware loss reweighting* assigns per-token scalar weights `w_t` in the NTP loss, amplifying supervision on response tokens (e.g., caption text in image captioning, speech tokens in TTS) to counteract imbalance from modalities with vastly different token counts. Second, *token-level perceptual alignment* adds an auxiliary L2 loss that aligns the decoder's hidden states (via a learned linear projection) to frozen pretrained image embeddings of the target VQ code -- this provides smoother gradient signal than one-hot cross-entropy, since perceptually similar VQ codes get similar embeddings. Third, *finite-state decoding* uses greedy decoding for deterministic tasks (ASR, TTS) and temperature-based sampling for creative tasks (text-to-image, open-ended dialogue), switched via task-type metadata.

**Training Recipe.** A two-stage process: (1) pre-training for 140K steps on mixed text-only, text-image, and text-speech data at a 0.5:1:2 ratio with batch size 480, and (2) instruction fine-tuning for 18K steps on omni-interleaved data with batch size 64. Training uses residual-post-norm (swin-norm) for stability, global gradient clipping at 1.0, and Adam with 5% linear warmup.

## Step-by-Step Workflow

1. **Define the unified vocabulary.** Merge three tokenizer vocabularies into one: text BPE tokens (e.g., SentencePiece with 65K vocab), image VQ codebook entries (e.g., 8192 codes from a scene-aware VQ-VAE), and speech codec tokens (e.g., WavTokenizer's single-codebook discrete tokens). Add special tokens for modality boundaries: `<boi>`, `<eoi>`, `<boa>`, `<eoa>`, `<bos>`, `<eos>`, plus task-type indicators.

2. **Build the token serialization pipeline.** For each training sample, convert all modalities to their discrete token sequences and concatenate them into one flat sequence with boundary markers. For example, an image-captioning sample becomes: `[image_tokens] <eoi> [caption_text_tokens]`. A TTS sample becomes: `[text_tokens] <boa> [speech_tokens] <eoa>`.

3. **Initialize from a pretrained text/image AR model.** Start from a 7B causal Transformer (e.g., one pretrained on interleaved image-text like Chameleon/Anole). Expand the embedding and output layers to accommodate the new speech token vocabulary. Initialize new embeddings randomly or from the speech tokenizer's codebook.

4. **Implement task-aware loss reweighting.** For each training sample, assign per-token weights `w_t` based on the task type. Set `w_t = 1.0` for all tokens by default. For X-to-Text tasks (ASR, captioning), increase weights on response text tokens (e.g., `w_t = 3.0`). For X-to-Image tasks, increase weights on image tokens. This compensates for modality imbalance without altering the data mixture.

   ```python
   # Pseudocode for weighted NTP loss
   def weighted_ntp_loss(logits, targets, weights):
       """weights: tensor of shape (batch, seq_len) with per-token scalars"""
       ce = F.cross_entropy(logits.view(-1, vocab_size), targets.view(-1), reduction='none')
       ce = ce.view(targets.shape)
       return (ce * weights).sum() / weights.sum()
   ```

5. **Add the perceptual alignment loss for image tokens.** Freeze a pretrained image encoder's VQ codebook embeddings `E`. Add a trainable linear projection `W_h` from the decoder's hidden dimension to the codebook embedding dimension. At image token positions, compute L2 loss between `W_h @ h_t` and `E[y_t]` (the target code's embedding). Scale with `lambda_perc` (start with 0.1, tune).

   ```python
   # Pseudocode for perceptual alignment loss
   def perceptual_loss(hidden_states, target_codes, projection, frozen_embeddings, image_mask):
       """Only applied at positions where image_mask is True"""
       projected = projection(hidden_states[image_mask])       # (N_img, embed_dim)
       targets = frozen_embeddings(target_codes[image_mask])    # (N_img, embed_dim)
       return F.mse_loss(projected, targets.detach())
   ```

6. **Configure the two-stage training schedule.** Stage 1 (pre-training): train on a balanced mixture of text-only, text-image, and text-speech data. Use a data ratio that prevents any single modality from dominating (e.g., 0.5:1:2). Stop iteration over a modality subset when it's exhausted rather than oversampling. Stage 2 (fine-tuning): use instruction-formatted multimodal data with lower learning rate.

7. **Apply residual-post-norm for training stability.** Replace standard pre-norm with swin-norm (apply LayerNorm to the residual branch output before adding to the residual stream). This prevents gradient explosion that commonly occurs when training on heterogeneous token distributions.

8. **Implement finite-state decoding at inference.** Route each generation request through a task classifier that selects the decoding strategy: greedy/beam search for deterministic tasks (ASR transcription, TTS synthesis, factual QA) and nucleus/temperature sampling for creative tasks (image generation, open-ended chat, story writing).

   ```python
   # Pseudocode for finite-state decoding
   def decode(model, prompt_tokens, task_type):
       if task_type in ('asr', 'tts', 'captioning'):
           return greedy_decode(model, prompt_tokens)
       elif task_type in ('text_to_image', 'creative_text'):
           return sample_decode(model, prompt_tokens, temperature=0.9, top_p=0.95)
       else:
           return sample_decode(model, prompt_tokens, temperature=0.7, top_p=0.9)
   ```

9. **Enable streaming speech output.** Since speech tokens are generated left-to-right and WavTokenizer uses a single codebook, begin decoding audio as soon as a sufficient chunk of speech tokens is available (e.g., every 40 tokens = 1 second at 40 tokens/sec). Feed chunks to the vocoder incrementally for sub-200ms first-token latency.

10. **Evaluate across all modalities independently.** Measure ASR with WER on LibriSpeech, TTS with WER + speaker similarity on VCTK, image generation with FID, image captioning with CIDEr, and text QA with standard benchmarks. Track the real-time factor (RTF) for speech to ensure it stays below 1.0.

## Concrete Examples

**Example 1: Implementing task-aware loss reweighting for a multimodal trainer**

User: "My unified model's ASR performance is poor even though speech-to-text samples are 30% of training data. The model seems to mostly learn image generation."

Approach:
1. Diagnose the imbalance: image tokens per sample (~1024) vastly outnumber ASR text response tokens (~50), so image token gradients dominate.
2. Add per-token weights to the training loop. For ASR samples, set `w_t = 5.0` for the response text tokens and `w_t = 1.0` for the input speech tokens.
3. Implement a `TaskWeightAssigner` that reads task-type metadata from each batch and returns a weight tensor.

Output:
```python
class TaskWeightAssigner:
    def __init__(self, task_weights: dict[str, tuple[float, float]]):
        # task_weights maps task_name -> (input_weight, response_weight)
        self.task_weights = task_weights

    def assign(self, task_types: list[str], input_masks: Tensor, response_masks: Tensor) -> Tensor:
        weights = torch.ones_like(input_masks, dtype=torch.float)
        for i, task in enumerate(task_types):
            w_in, w_resp = self.task_weights.get(task, (1.0, 1.0))
            weights[i] = weights[i] * w_in * input_masks[i] + weights[i] * w_resp * response_masks[i]
        return weights

# Usage
assigner = TaskWeightAssigner({
    'asr':        (1.0, 5.0),   # boost transcription text
    'captioning': (1.0, 3.0),   # boost caption text
    'tts':        (1.0, 2.0),   # boost speech output
    't2i':        (1.0, 1.0),   # image tokens already dominate
})
```

**Example 2: Adding perceptual alignment loss to a VQ image generation model**

User: "My autoregressive image generator produces images with correct structure but poor color coherence and local texture quality. How do I add the perceptual alignment loss from AR-Omni?"

Approach:
1. Extract the frozen VQ codebook embeddings from the image tokenizer.
2. Add a linear projection layer mapping decoder hidden states to the codebook embedding space.
3. Compute MSE between projected hidden states and target code embeddings at image token positions only.
4. Combine with the main cross-entropy loss using a scaling factor.

Output:
```python
class PerceptualAlignmentHead(nn.Module):
    def __init__(self, hidden_dim: int, codebook_dim: int, codebook: nn.Embedding):
        super().__init__()
        self.projection = nn.Linear(hidden_dim, codebook_dim)
        self.codebook = codebook
        self.codebook.requires_grad_(False)  # freeze

    def forward(self, hidden_states: Tensor, target_codes: Tensor, image_mask: Tensor) -> Tensor:
        if not image_mask.any():
            return torch.tensor(0.0, device=hidden_states.device)
        proj = self.projection(hidden_states[image_mask])
        target_emb = self.codebook(target_codes[image_mask])
        return F.mse_loss(proj, target_emb)

# In training loop
loss_ntp = weighted_ntp_loss(logits, targets, weights)
loss_perc = perc_head(hidden_states, targets, image_mask)
loss = loss_ntp + 0.1 * loss_perc  # lambda_perc = 0.1
```

**Example 3: Designing the unified vocabulary and serialization for a new omni model**

User: "I want to build a unified any-to-any model. How should I structure the vocabulary and token sequences?"

Approach:
1. Enumerate all token sources and reserve non-overlapping ID ranges.
2. Define special/boundary tokens for modality switching.
3. Build a serialization function that converts any multimodal sample into a flat token sequence.

Output:
```python
# Vocabulary layout
VOCAB = {
    'text':    range(0, 65536),          # SentencePiece BPE
    'image':   range(65536, 73728),      # 8192 VQ codes
    'speech':  range(73728, 77824),      # 4096 WavTokenizer codes
    'special': range(77824, 77856),      # 32 special tokens
}

SPECIAL = {
    '<boi>': 77824, '<eoi>': 77825,      # image boundaries
    '<boa>': 77826, '<eoa>': 77827,      # audio boundaries
    '<task_asr>': 77828, '<task_tts>': 77829,
    '<task_caption>': 77830, '<task_t2i>': 77831,
    '<pad>': 77832,
}

def serialize_sample(task: str, input_tokens: list[int], output_tokens: list[int],
                     input_modality: str, output_modality: str) -> list[int]:
    seq = [SPECIAL[f'<task_{task}>']]
    if input_modality == 'image':
        seq += [SPECIAL['<boi>']] + input_tokens + [SPECIAL['<eoi>']]
    elif input_modality == 'speech':
        seq += [SPECIAL['<boa>']] + input_tokens + [SPECIAL['<eoa>']]
    else:
        seq += input_tokens
    # Output tokens with appropriate boundaries
    if output_modality == 'image':
        seq += [SPECIAL['<boi>']] + output_tokens + [SPECIAL['<eoi>']]
    elif output_modality == 'speech':
        seq += [SPECIAL['<boa>']] + output_tokens + [SPECIAL['<eoa>']]
    else:
        seq += output_tokens
    return seq
```

## Best Practices

- **Do** assign higher loss weights to the response modality tokens rather than reducing weights on input tokens. The model still needs to encode inputs accurately; the issue is that response tokens are underrepresented.
- **Do** use a single-codebook speech tokenizer (like WavTokenizer) for streaming -- dual-codebook systems (semantic + acoustic) add latency and complexity that defeats the purpose of unified AR.
- **Do** stop iterating over exhausted data subsets rather than oversampling them. Oversampling small modality datasets causes overfitting on that modality while undertrained on others.
- **Do** use residual-post-norm (swin-norm) instead of pre-norm when training on mixed-modality token streams. The heterogeneous token distributions cause gradient instability with standard pre-norm.
- **Avoid** applying perceptual alignment loss to text or speech tokens -- it is specifically designed for image VQ codes where the codebook has meaningful geometric structure in embedding space.
- **Avoid** using a single decoding strategy for all tasks. Greedy decoding kills image generation diversity; sampling introduces errors in ASR transcription. The finite-state approach is critical.

## Error Handling

- **Modality collapse (model only generates one modality well):** Check the data mixture ratio and loss weights. Reduce the dominant modality's data share or increase loss weight on the underperforming modality's response tokens. A 0.5:1:2 ratio (text:image:speech) is a reasonable starting point.
- **Training instability / loss spikes:** Enable residual-post-norm and set gradient clipping to 1.0. If instability persists, reduce the learning rate and increase warmup steps. Multimodal training is inherently less stable than unimodal.
- **Image generation produces correct tokens but poor visual quality:** Increase `lambda_perc` for the perceptual alignment loss. Verify the frozen codebook embeddings come from a well-trained VQ-VAE. Check that the projection layer has sufficient capacity.
- **Speech streaming has high latency:** Ensure the speech tokenizer rate is low enough (40 tokens/sec or fewer). Check that chunk size for incremental decoding is not too large. First-token latency should target <200ms.
- **Vocabulary ID collisions across modalities:** Always use non-overlapping ID ranges. Validate at model initialization that no token ID is shared across modality tokenizers.

## Limitations

- Autoregressive image generation quality still lags behind diffusion-based systems (Stable Diffusion, DALL-E 3). If image generation quality is the primary goal, a diffusion decoder will outperform this approach.
- The unified approach requires significantly more compute than single-modality models. The 7B parameter scale with 140K pre-training steps on A100x8 is a substantial infrastructure requirement.
- Speech quality depends heavily on the discrete tokenizer. WavTokenizer works well at low token rates, but higher-fidelity speech may need larger codebooks that increase sequence length and training cost.
- The finite-state decoding mechanism requires task-type metadata at inference time. For ambiguous user requests, you need a reliable task classifier or explicit user specification.
- This approach has not been validated for video generation or other temporal modalities beyond speech. Extending to video would significantly increase sequence lengths.

## Reference

**Paper:** [AR-Omni: A Unified Autoregressive Model for Any-to-Any Generation](https://arxiv.org/abs/2601.17761) -- Cheng et al., 2026. Focus on Section 3 (method) for the weighted NTP formulation, perceptual alignment loss equations, and finite-state decoding design. Section 4 (experiments) provides the training hyperparameters and ablation studies showing the contribution of each component.