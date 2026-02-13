---
name: "wavlink-compact-audio-text-embeddings"
description: "Build compact audio-text embedding systems using WavLink's global Whisper token architecture with Matryoshka dimensionality. Use when: 'build an audio search engine', 'embed audio clips for retrieval', 'set up audio-text CLAP-style matching', 'implement Matryoshka embeddings for audio', 'zero-shot audio classification with text prompts', 'compact Whisper-based audio embeddings'."
---

# WavLink: Compact Audio-Text Embeddings with a Global Whisper Token

This skill enables Claude to help users build audio-text embedding and retrieval systems based on the WavLink architecture (ICASSP 2026). WavLink replaces the standard 1500-token Whisper output with a single compact embedding by appending a learnable global token to the Whisper encoder, then training it contrastively against a CLIP text encoder. The result is a system that produces L2-normalized audio embeddings (768D down to 96D via Matryoshka slicing) suitable for text-to-audio retrieval, audio-to-text retrieval, and zero-shot audio classification -- all from a single forward pass that outputs one token instead of 1500.

## When to Use

- When the user wants to build an audio search engine that matches text queries to audio clips (or vice versa)
- When the user needs to add audio understanding to an application using compact embeddings rather than full LLM-style 1500-token sequences
- When the user asks about replacing CLAP or HTS-AT-based audio encoders with a Whisper-based alternative
- When the user wants variable-dimension audio embeddings (Matryoshka style) to trade off index size vs. retrieval quality
- When the user needs zero-shot audio classification by comparing audio embeddings against text label embeddings
- When the user is designing a multi-modal retrieval pipeline that includes audio alongside text and images
- When the user asks how to efficiently pool Whisper encoder outputs into a single vector

## Key Technique

**Global Whisper Token Pooling.** Standard Whisper produces ~1500 frame-level features for a 30-second clip. WavLink appends a learnable parameter vector `a_cls` (shape `1 x D`) to the output of Whisper's convolutional front-end, before the Transformer layers. This token propagates through the full Transformer stack via self-attention, attending to all audio frames. Its final hidden state is extracted as the single pooled audio representation, then projected and L2-normalized to produce the embedding. This is analogous to the `[CLS]` token in BERT but applied to the audio domain on top of a pretrained Whisper encoder.

**Contrastive Training with CLIP Text Encoder.** The audio embedding is trained jointly with a CLIP text encoder using symmetric InfoNCE loss (CLIP-style cross-entropy over the cosine similarity matrix, applied to both rows and columns). A systematic design sweep showed this combination -- CLIP text encoder + CLIP loss + full finetuning of both towers -- outperformed alternatives like ModernBERT (despite its stronger standalone text benchmarks), SigLIP loss, and frozen/LoRA-only training modes.

**Two-Stage Recipe + Matryoshka Supervision.** Stage 1 trains on ~8M large-scale caption pairs (Auto-ACD + AudioSetCaps). Stage 2 fine-tunes on ~100K high-quality pairs (AudioCaps v2 + Clotho). Matryoshka representation learning applies the contrastive loss at multiple embedding slices simultaneously (e.g., 768/384/192/96 for the Large model), averaging the losses. This lets users truncate embeddings at deployment time with less than 1 point of retrieval performance loss at 8x compression (96D vs. 768D).

## Step-by-Step Workflow

### Building an Audio-Text Retrieval System

1. **Select the model size based on deployment constraints.** WavLink comes in three sizes: Large (761M params, 768D embeddings), Small (152M params, 512D), and Base (84M params, 512D). Choose Large for maximum retrieval quality, Base for edge/on-device deployment where the 84M parameter budget and single-token output are critical advantages.

2. **Set up the Whisper encoder with the global token modification.** Load the pretrained Whisper encoder (large-v2 for WavLink-Large, small/tiny for smaller variants). Insert a learnable `nn.Parameter(torch.randn(1, 1, D))` that gets concatenated to the conv front-end output along the sequence dimension before entering the Transformer blocks. Add a linear projection head on the global token's final hidden state to map it to the target embedding dimension.

3. **Set up the text encoder.** Load a pretrained CLIP text encoder (e.g., `openai/clip-vit-large-patch14` for the Large model, `openai/clip-vit-base-patch32` for smaller variants). Add a linear projection head mapping to the same embedding dimension as the audio side.

4. **Implement the CLIP-style contrastive loss with Matryoshka slicing.** For each batch, compute cosine similarities between all audio-text pairs. Apply cross-entropy loss over both rows (text-to-audio) and columns (audio-to-text), then average. For Matryoshka training, slice embeddings at multiple dimensions (e.g., `[768, 384, 192, 96]`), compute the contrastive loss at each slice, L2-normalize each slice independently, and average all losses.

5. **Prepare training data in two stages.** Stage 1 uses large-scale audio-caption pairs (~2M from Auto-ACD derived from AudioSet/VGGSound, ~6M from AudioSetCaps). Stage 2 fine-tunes on high-quality pairs (~50K AudioCaps v2 + ~50K Clotho training split). Audio should be resampled to 16kHz and padded/truncated to 30 seconds for Whisper's expected input format.

6. **Train Stage 1 with full finetuning of both towers.** Unfreeze both the Whisper audio encoder and CLIP text encoder. Train with AdamW, learning rate ~1e-5 for pretrained parameters, ~1e-4 for the new global token and projection heads. Use a cosine schedule with warmup.

7. **Fine-tune Stage 2 on high-quality data.** Load Stage 1 checkpoint. Continue training on the smaller, cleaner dataset with a lower learning rate (~1e-6 to 1e-5). This stage sharpens retrieval performance on standard benchmarks.

8. **Build the retrieval index.** Extract audio embeddings for your corpus in a single forward pass per clip (one vector per audio, not 1500). L2-normalize and store in a vector database (FAISS, Qdrant, Pinecone, pgvector). Choose the Matryoshka dimension that fits your latency/storage budget -- 96D gives 8x compression with <1 point recall loss.

9. **Implement query-time retrieval.** For text-to-audio: encode the text query with the text encoder, L2-normalize, and perform nearest-neighbor search against the audio index. For audio-to-text: encode the audio query with the Whisper+global-token encoder and search against precomputed text embeddings.

10. **Implement zero-shot classification (optional).** Create text embeddings for each class label using the template `"the sound of {label}"`. For a given audio clip, compute its embedding and return the label with highest cosine similarity. WavLink-Large achieves 83.0% on ESC-50 and 74.5% on UrbanSound8K with this approach.

## Concrete Examples

**Example 1: Audio Search Engine with FAISS**

User: "I want to build a text-to-audio search engine over 100K audio clips. I need it to be fast and fit in memory on a single GPU."

Approach:
1. Use WavLink-Base (84M params) with 128D Matryoshka embeddings for compact indexing.
2. Precompute audio embeddings -- each 30s clip produces a single 128D vector, so 100K clips = ~50MB of float32 vectors.
3. Build a FAISS `IndexFlatIP` (inner product, since embeddings are L2-normalized, IP = cosine similarity).
4. At query time, encode text with the CLIP text encoder, truncate to 128D, L2-normalize, and search.

```python
import torch
import faiss
import numpy as np

# Precompute audio embeddings (pseudocode using WavLink model)
audio_embeddings = []
for audio_path in audio_corpus:
    mel = whisper.log_mel_spectrogram(audio_path)  # [1, 80, 3000]
    emb = wavlink_model.encode_audio(mel)            # [1, 512]
    emb = emb[:, :128]                               # Matryoshka slice to 128D
    emb = torch.nn.functional.normalize(emb, dim=-1)
    audio_embeddings.append(emb.cpu().numpy())

audio_matrix = np.vstack(audio_embeddings).astype("float32")  # [100000, 128]

# Build FAISS index
index = faiss.IndexFlatIP(128)
index.add(audio_matrix)

# Query
query_emb = wavlink_model.encode_text("a dog barking loudly")[:, :128]
query_emb = torch.nn.functional.normalize(query_emb, dim=-1).cpu().numpy()
scores, indices = index.search(query_emb, k=10)
```

Output: Top-10 audio clips ranked by relevance to "a dog barking loudly", retrieved in <1ms.

**Example 2: Zero-Shot Environmental Sound Classification**

User: "I have a stream of audio from a security camera. Classify each 10-second clip into categories like siren, gunshot, glass breaking, dog bark, speech, silence."

Approach:
1. Use WavLink-Large for maximum classification accuracy.
2. Precompute text embeddings for each label using the template `"the sound of {label}"`.
3. For each incoming audio clip, compute its embedding and pick the label with highest cosine similarity.

```python
import torch

labels = ["siren", "gunshot", "glass breaking", "dog bark", "speech", "silence"]
templates = [f"the sound of {label}" for label in labels]

# Precompute label embeddings once
label_embs = wavlink_model.encode_text(templates)        # [6, 768]
label_embs = torch.nn.functional.normalize(label_embs, dim=-1)

# Classify incoming audio
def classify_audio(audio_waveform):
    audio_emb = wavlink_model.encode_audio(audio_waveform)  # [1, 768]
    audio_emb = torch.nn.functional.normalize(audio_emb, dim=-1)
    similarities = (audio_emb @ label_embs.T).squeeze()     # [6]
    predicted_idx = similarities.argmax().item()
    return labels[predicted_idx], similarities[predicted_idx].item()

label, confidence = classify_audio(clip)
# -> ("glass breaking", 0.34)
```

Output: Real-time classification with no training data needed for new categories.

**Example 3: Adding a Global Token to an Existing Whisper Model**

User: "I already use Whisper in my pipeline. How do I add the global token for pooling?"

Approach:
1. Subclass or wrap the Whisper encoder to inject the learnable token.
2. The token is concatenated after the conv layers, before the Transformer blocks.

```python
import torch
import torch.nn as nn

class WhisperWithGlobalToken(nn.Module):
    def __init__(self, whisper_model, embed_dim=768, proj_dim=768):
        super().__init__()
        self.whisper = whisper_model.encoder
        self.global_token = nn.Parameter(torch.randn(1, 1, embed_dim) * 0.02)
        self.proj = nn.Linear(embed_dim, proj_dim)

    def forward(self, mel_input):
        # Run Whisper's conv front-end
        x = self.whisper.conv1(mel_input)
        x = nn.functional.gelu(x)
        x = self.whisper.conv2(x)
        x = nn.functional.gelu(x)
        x = x.permute(0, 2, 1)  # [B, T, D]

        # Append global token
        batch_size = x.size(0)
        cls = self.global_token.expand(batch_size, -1, -1)  # [B, 1, D]
        x = torch.cat([x, cls], dim=1)  # [B, T+1, D]

        # Add positional embeddings (extend by 1 position or use learned pos for cls)
        # Then run through Transformer blocks
        for block in self.whisper.blocks:
            x = block(x)
        x = self.whisper.ln_post(x)

        # Extract global token (last position)
        pooled = x[:, -1, :]  # [B, D]
        return torch.nn.functional.normalize(self.proj(pooled), dim=-1)
```

Output: A drop-in replacement that converts Whisper's 1500-token output into a single embedding vector.

## Best Practices

- **Do:** Use CLIP's text encoder (not ModernBERT or other alternatives) as the text tower -- the WavLink design sweep showed CLIP text + CLIP loss consistently outperforms other combinations, even though ModernBERT scores higher on standalone text benchmarks.
- **Do:** L2-normalize all embeddings before storage and retrieval. WavLink's contrastive training assumes unit-norm vectors; skipping normalization will degrade cosine similarity rankings.
- **Do:** Use Matryoshka slicing at deployment time by simply truncating the embedding vector to the desired dimension (e.g., `emb[:, :96]`). No retraining is needed -- the model was trained with supervision at all slice points.
- **Do:** Train in two stages -- large-scale noisy data first, then small-scale clean data -- rather than mixing everything in one stage. This consistently improves retrieval metrics.
- **Avoid:** Freezing the Whisper encoder or using LoRA-only fine-tuning. The design sweep showed full finetuning of both audio and text towers is the optimal training mode.
- **Avoid:** Using SigLIP loss as a drop-in replacement for CLIP loss. Despite SigLIP's theoretical advantages for larger batch sizes, CLIP loss performed better in WavLink's contrastive setup.

## Error Handling

- **Audio shorter than 30 seconds:** Pad with zeros to Whisper's expected 30-second input (480,000 samples at 16kHz). Whisper's conv front-end and attention mechanism handle padded inputs gracefully; the global token still attends to the actual content.
- **Audio longer than 30 seconds:** Truncate to the first 30 seconds, or segment into 30-second chunks and average their embeddings. For retrieval, chunking with max-pooling over chunk similarities often works better than averaging.
- **Mismatched embedding dimensions between audio and text:** Ensure both projection heads output the same dimension. If using Matryoshka slicing, truncate both audio and text embeddings to the same target dimension.
- **OOM during training with large batches:** Contrastive learning benefits from large batches (the paper uses the full batch as negatives). Use gradient accumulation or gather embeddings across GPUs before computing the loss matrix to maintain effective batch size without exceeding memory.
- **Poor zero-shot classification accuracy:** Experiment with prompt templates. Instead of just `"the sound of {label}"`, try `"{label}"`, `"audio of {label}"`, or ensembles of multiple templates averaged together.

## Limitations

- WavLink requires Whisper as the audio backbone, which expects 16kHz mono audio and processes fixed 30-second windows. It cannot handle streaming or variable-length input natively.
- The model is trained primarily on environmental sounds, music, and speech captions. It may underperform on highly specialized audio domains (medical auscultation, industrial machinery) without domain-specific fine-tuning.
- Zero-shot classification accuracy (83% ESC-50, 74.5% US8K) is competitive but below supervised baselines. For production classification tasks with fixed label sets, a supervised head trained on top of the frozen embeddings will outperform zero-shot.
- No official pretrained weights or code repository has been released as of the paper's publication. Implementation requires building the architecture from the paper's description.
- The single global token captures a holistic audio summary. For tasks requiring temporal localization (e.g., "at what timestamp does the dog bark?"), the 1500-frame output from standard Whisper is more appropriate.

## Reference

[WavLink: Compact Audio-Text Embeddings with a Global Whisper Token](https://arxiv.org/abs/2601.15118v2) (ICASSP 2026) -- Focus on Table 1 (design sweep results), Table 2 (Matryoshka dimension ablation with retrieval scores), Table 3 (zero-shot classification), and Table 4 (AIR-Bench comparison showing WavLink-Base matches 43-100x larger models).