---
name: "timbre-aware-llm-based-direct-speech-to-speech"
description: "Build direct speech-to-speech translation systems that preserve speaker identity using LLM-based architectures with timbre-controlled vocoders. Triggers: 'build a speech-to-speech translation pipeline', 'implement direct S2ST with speaker preservation', 'create a voice-preserving translation model', 'design an LLM-based speech translation system', 'set up a timbre-aware speech synthesis pipeline', 'implement DS2ST-LM architecture'"
---

# Timbre-Aware LLM-Based Direct Speech-to-Speech Translation

This skill enables Claude to help users implement direct speech-to-speech translation (S2ST) systems based on the DS2ST-LM architecture from arXiv:2601.16023. The core idea: wire a frozen Whisper speech encoder through a learnable projection module into a multilingual LLM (Qwen2-0.5B), then decode with a timbre-controlled vocoder (CosyVoice) -- producing translated speech in a single stage while preserving the original speaker's voice. This replaces fragile ASR->MT->TTS cascades with an end-to-end system that achieves better BLEU, METEOR, BLEURT, and COMET scores while maintaining 0.83 speaker cosine similarity.

## When to Use

- When the user wants to build an end-to-end speech-to-speech translation system that avoids cascaded error propagation
- When the user needs to preserve speaker identity (timbre/voice characteristics) across translated speech
- When the user is constructing parallel speech corpora using synthetic TTS data to overcome data scarcity
- When the user asks about projection module design for connecting speech encoders to LLM backbones
- When the user is comparing semantic token strategies (speech-derived vs. text-derived) for speech generation
- When the user wants to extend a bilingual S2ST system to multiple language pairs (French, Spanish, German, Hindi, Bengali, Urdu)
- When the user asks how to fine-tune an LLM to produce speech tokens conditioned on audio input

## Key Technique

**Single-Stage LLM-Mediated Translation.** DS2ST-LM replaces cascaded ASR->MT->TTS pipelines with a unified architecture: a frozen Whisper-small encoder extracts 50 Hz speech features, a projection module downsamples and maps these into the LLM embedding space, and a Qwen2-0.5B LLM autoregressively generates target-language semantic tokens. The critical insight is that an LLM pretrained on multilingual text already encodes cross-lingual semantic knowledge -- by projecting speech embeddings into its representation space, you leverage that knowledge for translation without intermediate text decoding.

**Semantic Token Design.** The system uses Supervised Semantic Speech (S3) tokens extracted via a SenseVoice ASR encoder with vector quantization (codebook size 4096) inserted between encoder layers. These tokens capture linguistic content stripped of acoustic identity. An alternative text-derived token strategy uses CosyVoice-300M-SFT's text-to-token LLM with teacher forcing, enabling training on speech-translation datasets that lack parallel target audio. The speech-derived tokens produce more stable training; the text-derived tokens unlock more data sources.

**Timbre-Controlled Synthesis.** CosyVoice's conditional flow-matching model takes the generated semantic tokens plus a 6-second speaker embedding prompt and synthesizes mel-spectrograms preserving the source speaker's voice. HiFi-GAN converts these to waveforms. This decouples *what is said* (semantic tokens) from *who says it* (speaker embedding), allowing independent control. Speaker similarity is measured via WavLM-based cosine similarity between source and synthesized embeddings.

## Step-by-Step Workflow

1. **Set up the speech encoder.** Load a frozen Whisper-small encoder (pretrained on 680K hours). Extract encoder hidden states at 50 Hz frame rate. Do not fine-tune the encoder -- keep it frozen to preserve multilingual acoustic representations.

2. **Choose and implement the projection module.** Implement one of three architectures to map Whisper features into the LLM embedding space:
   - **Linear (recommended for final performance):** Group k consecutive encoder frames, pass through a two-layer MLP with ReLU activation. This downsamples temporal resolution to match the LLM's processing rate (~3 Hz for text tokens, using group size G=3).
   - **Conv1D-Linear (fastest convergence):** Apply 1D convolution with kernel size and stride k, followed by MLP projection. Converges faster but peaks lower.
   - **Q-Former (highest capacity):** Use N_q learnable query embeddings with cross-attention over encoder states (hidden dimension d_q). Most complex; does not outperform Linear at convergence.

3. **Prepare semantic token extraction.** Set up the S3 token pipeline: split a SenseVoice multilingual ASR encoder at layer 6, insert a vector quantization bottleneck (codebook size 4096, argmin L2 distance), and extract discrete token sequences from target speech. These tokens encode linguistic content without speaker acoustics.

4. **Construct or prepare the parallel speech corpus.** If parallel target speech is unavailable, synthesize it:
   - Use XTTS-v2 (or equivalent multilingual zero-shot TTS) with a 6-second reference prompt to generate target-language speech from text translations.
   - Filter pairs using SONAR cross-lingual embeddings -- retain only samples with cosine similarity above 0.9.
   - Target ~1000 hours of aligned bilingual speech pairs (the GigaS2S-1000 recipe).

5. **Configure the LLM backbone.** Load Qwen2-0.5B (multilingual, 0.5B parameters). Extend its vocabulary with semantic token IDs (4096 new tokens for the S3 codebook). Set up dual-output heads: one for text tokens, one for semantic speech tokens. Apply equal loss weighting (lambda_audio = lambda_text = 1).

6. **Train with mixed-modality supervision.** Fine-tune the projection module and LLM (LoRA or full fine-tune) with:
   - Learning rate: 1e-4 with 1000-step warmup and decay factor 0.85
   - Batch size: 8 per GPU (H100 80GB, FP16 mixed precision)
   - Maximum 4 epochs with early stopping on validation loss
   - Input: projected Whisper features; targets: interleaved text and semantic token sequences

7. **Set up the timbre-controlled vocoder.** Load CosyVoice with its 4096-token codebook. The vocoder takes semantic tokens + a speaker embedding extracted from a 6-second reference clip of the source speaker. The conditional flow-matching model generates mel-spectrograms; HiFi-GAN converts to 16kHz/22kHz waveforms.

8. **Implement the inference pipeline.** Wire the full forward pass: source speech -> Whisper encoder -> projection -> LLM autoregressive decoding (semantic tokens) -> CosyVoice vocoder (conditioned on source speaker embedding) -> target waveform.

9. **Evaluate with multi-level metrics.** Compute:
   - Lexical: BLEU, METEOR (ASR the output, compare to reference text)
   - Semantic: BLEURT, COMET (capture meaning preservation beyond surface form)
   - Speaker: WavLM cosine similarity between source and synthesized speaker embeddings (target: >0.8)
   - Naturalness: MOS or UTMOS perceptual quality scores

10. **Extend to new language pairs.** Add languages by preparing parallel data (real or synthetic) for the new pair, extending S3 token extraction to cover the target language, and fine-tuning the LLM on the combined multilingual corpus. The shared LLM backbone enables positive transfer across language pairs.

## Concrete Examples

**Example 1: Building an English-to-Chinese S2ST Pipeline**

User: "I want to build a direct speech-to-speech translation system from English to Chinese that keeps the speaker's voice."

Approach:
1. Load `whisper-small` from HuggingFace, freeze all parameters
2. Implement a Linear projection module (2-layer MLP, ReLU, group size k=3 to downsample from 50Hz to ~17Hz)
3. Load `Qwen/Qwen2-0.5B` and extend tokenizer with 4096 semantic token IDs
4. Download GigaS2S-1000 from HuggingFace or construct equivalent using GigaST + XTTS-v2 synthesis
5. Extract S3 tokens from target Chinese speech using SenseVoice encoder with VQ bottleneck
6. Train: project Whisper output -> feed to Qwen2 -> predict semantic tokens with cross-entropy loss
7. At inference: decode semantic tokens, pass to CosyVoice with source speaker's 6s reference clip

Output architecture (PyTorch pseudocode):
```python
class DS2STLM(nn.Module):
    def __init__(self):
        self.encoder = WhisperModel.from_pretrained("openai/whisper-small").encoder
        self.encoder.requires_grad_(False)  # frozen
        self.projector = LinearProjector(
            in_dim=768, out_dim=896,  # Whisper-small -> Qwen2-0.5B
            group_size=3
        )
        self.llm = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2-0.5B")
        # Extend embeddings for 4096 semantic tokens
        self.llm.resize_token_embeddings(self.llm.config.vocab_size + 4096)

    def forward(self, speech_input, labels):
        enc_out = self.encoder(speech_input).last_hidden_state  # [B, T, 768]
        projected = self.projector(enc_out)  # [B, T//3, 896]
        outputs = self.llm(inputs_embeds=projected, labels=labels)
        return outputs.loss

class LinearProjector(nn.Module):
    def __init__(self, in_dim, out_dim, group_size):
        super().__init__()
        self.group_size = group_size
        self.mlp = nn.Sequential(
            nn.Linear(in_dim * group_size, out_dim),
            nn.ReLU(),
            nn.Linear(out_dim, out_dim)
        )

    def forward(self, x):
        B, T, D = x.shape
        T_new = T // self.group_size
        x = x[:, :T_new * self.group_size].reshape(B, T_new, D * self.group_size)
        return self.mlp(x)
```

**Example 2: Constructing a Synthetic Parallel Speech Corpus**

User: "I have English speech with Chinese text translations but no Chinese speech. How do I create training data?"

Approach:
1. Use XTTS-v2 for zero-shot multilingual TTS with a consistent speaker prompt
2. Filter generated pairs for quality using cross-lingual embedding similarity
3. Extract S3 tokens from the synthesized speech for training targets

```python
from TTS.api import TTS
import torch
from sonar.inference_pipelines import TextToEmbeddingPipeline

# 1. Synthesize target Chinese speech
tts = TTS("tts_models/multilingual/multi-dataset/xtts_v2")
for sample in dataset:
    tts.tts_to_file(
        text=sample["zh_text"],
        speaker_wav="reference_speaker_6s.wav",  # 6-second prompt
        language="zh-cn",
        file_path=f"synth_zh/{sample['id']}.wav"
    )

# 2. Filter with SONAR embeddings (retain cosine sim > 0.9)
text_embedder = TextToEmbeddingPipeline(encoder="sonar_text_encoder")
en_emb = text_embedder.predict([s["en_text"] for s in dataset], source_lang="eng")
zh_emb = text_embedder.predict([s["zh_text"] for s in dataset], source_lang="cmn")
similarities = torch.cosine_similarity(en_emb, zh_emb)
filtered = [s for s, sim in zip(dataset, similarities) if sim > 0.9]

# 3. Extract S3 tokens from synthesized audio
# Use SenseVoice encoder with VQ layer after layer 6
s3_tokens = extract_s3_tokens("synth_zh/001.wav")  # -> [T] int tensor
```

**Example 3: Adding a New Language Pair (French-English)**

User: "I have a working English-Chinese model. How do I extend it to French-English?"

Approach:
1. Prepare French-English parallel speech data (use Seamless-Align or CVSS corpora, or synthesize with XTTS-v2)
2. Extract S3 tokens for both English targets using the same SenseVoice VQ pipeline
3. Continue fine-tuning the existing model on combined En-Zh + Fr-En data
4. The Whisper encoder already handles French; the Qwen2 LLM already knows French text
5. Evaluate with BLEU/COMET on FLEURS fr-en test set

```python
# Extend training dataset
combined_dataset = ConcatDataset([
    GigaS2S1000Dataset("en-zh"),     # existing
    SeamlessAlignDataset("fr-en"),    # new pair
])

# Continue training with same hyperparameters
trainer = Trainer(
    model=ds2st_lm,
    train_dataset=combined_dataset,
    lr=1e-4, warmup_steps=1000, max_epochs=4,
    fp16=True, batch_size=8
)
trainer.train()  # LLM transfers cross-lingual knowledge
```

## Best Practices

- **Do:** Keep the Whisper encoder frozen. Its pretrained multilingual representations are more robust than any fine-tuned version on limited S2ST data.
- **Do:** Start with the Linear projector. Despite being simplest, it outperforms Conv1D-Linear and Q-Former at convergence (BLEU 14.71 vs lower for others). Only use Q-Former if you need faster early convergence for prototyping.
- **Do:** Use equal loss weighting (lambda_audio = lambda_text = 1) for the dual text + semantic token objective. This stabilizes training.
- **Do:** Filter synthetic parallel data aggressively with SONAR embeddings (cosine similarity > 0.9). Low-quality pairs degrade translation accuracy significantly.
- **Avoid:** Generating semantic tokens from raw audio without stripping speaker information. S3 tokens must encode *content only* -- acoustic identity belongs in the vocoder's speaker embedding.
- **Avoid:** Skipping the text-token auxiliary loss. Joint text + semantic token prediction acts as a regularizer that improves semantic consistency, even though only semantic tokens are used at inference.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Semantic tokens are unintelligible | VQ codebook collapse during S3 extraction | Use codebook reset or EMA updates; verify codebook utilization > 80% |
| Speaker voice not preserved | Speaker embedding too short or noisy | Ensure reference clip is 6+ seconds of clean speech; use WavLM verification to pre-check embedding quality |
| Training loss plateaus early | Projection module bottleneck | Increase MLP hidden dimension; verify group_size aligns frame rates correctly (50Hz / k should match LLM token rate) |
| BLEU is low but COMET is acceptable | Surface form divergence with preserved meaning | This is expected for direct S2ST -- semantic metrics (BLEURT, COMET) are more reliable indicators than BLEU for speech-mediated translation |
| Out-of-memory on training | Model + optimizer exceeds GPU RAM | Use FP16 mixed precision, gradient accumulation, or LoRA (rank 16-64) on the LLM instead of full fine-tuning |
| Synthesized speech has wrong language | LLM generating tokens for wrong target | Prepend a language tag token to the decoder input; verify training data language labels are correct |

## Limitations

- **Data dependency:** The approach requires ~1000 hours of parallel speech or high-quality synthetic equivalents per language pair. Low-resource languages without good TTS coverage are hard to bootstrap.
- **Speaker similarity ceiling:** Timbre preservation depends on the vocoder's speaker embedding quality. Speakers with unusual vocal characteristics (children, heavily accented) may not transfer well with a 6-second prompt.
- **Latency:** Autoregressive LLM decoding of semantic tokens is sequential. Real-time streaming S2ST requires speculative decoding or non-autoregressive alternatives not covered by this architecture.
- **Small LLM tradeoff:** Qwen2-0.5B is chosen for efficiency, but translation quality on distant language pairs (e.g., Hindi-English) lags behind larger LLMs. Scaling to 1.5B-7B would improve quality at the cost of inference speed.
- **Evaluation requires ASR:** Lexical metrics (BLEU, METEOR) require ASR transcription of the output speech, introducing ASR errors into evaluation. Semantic metrics (COMET, BLEURT) partially mitigate this but still depend on transcript quality.

## Reference

**Paper:** [Timbre-Aware LLM-based Direct Speech-to-Speech Translation Extendable to Multiple Language Pairs](https://arxiv.org/abs/2601.16023v1) (Arya et al., 2026). Look for Section 3 (architecture details), Section 4 (GigaS2S-1000 construction), and Tables 2-4 (ablation results comparing projection modules, token types, and baselines).