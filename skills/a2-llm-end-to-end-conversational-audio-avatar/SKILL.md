---
name: "a2-llm-end-to-end-conversational-audio-avatar"
description: "Build end-to-end conversational audio avatar systems that jointly generate speech and expressive 3D facial motion from a single LLM. Uses the A2-LLM architecture with FLAME-based facial representation, RVQ-VAE motion tokenization, and audio-anchored generation. Trigger phrases: 'build a talking avatar', 'generate facial animation from speech', 'end-to-end audio avatar pipeline', 'conversational digital human', 'FLAME facial motion generation', 'lip sync with emotion'"
---

# A2-LLM: End-to-End Conversational Audio Avatar

This skill enables Claude to design and implement end-to-end conversational avatar systems that jointly produce speech audio and emotionally expressive 3D facial motion from a unified large language model. Rather than cascading separate ASR, LLM, TTS, and face-animation modules (which accumulate errors and latency), the A2-LLM approach feeds text and audio tokens through a single autoregressive backbone while a lightweight Motion Connector decodes synchronized FLAME facial parameters from the LLM's hidden states. This produces avatars that respond in real time (~500 ms first-action latency, 0.7 RTF) with facial expressions driven by semantic understanding, not just phoneme-level lip sync.

## When to Use

- When the user asks to build a talking head or digital human that responds conversationally with facial animation
- When integrating speech generation and 3D face animation into a single model instead of a cascade pipeline
- When the user wants to tokenize FLAME facial parameters using RVQ-VAE for discrete sequence modeling
- When designing a multimodal LLM that outputs interleaved text, audio, and motion tokens
- When constructing a QA-format multimodal dataset that pairs conversational audio with aligned FLAME motion sequences
- When the user needs to reduce latency in an existing cascaded avatar pipeline by moving to an end-to-end architecture
- When adding emotional expressiveness to a lip-sync system that currently only tracks mouth openings

## Key Technique

**Unified autoregressive generation across three modalities.** A2-LLM extends a speech-capable LLM (Qwen2.5-7B backbone with LoRA, rank=64) so that it autoregressively generates interleaved text and audio tokens at 25 Hz. During this generation, the LLM's hidden states encode both semantic intent and acoustic prosody. A 6-layer Transformer decoder called the Motion Connector reads these hidden states, downsamples them via 1D convolution to match the motion frame rate (5 Hz after temporal grouping), and predicts hierarchical RVQ codes that reconstruct FLAME facial parameters. This "audio-anchored" strategy grounds facial motion in the same latent space as language and speech, so the avatar's expressions reflect what is being said and how it is being said.

**FLAME parameters as a structured motion space.** Instead of predicting raw mesh vertices or 2D landmarks, A2-LLM operates on FLAME's disentangled representation: 50 expression coefficients, 3 jaw-pose angles, 3 global head-rotation angles, and 2 eyelid parameters (D=58 total). These 58-dimensional vectors are compressed temporally (groups of 5 frames) and quantized through a Residual Vector Quantized VAE with 6 codebooks of 256 entries each. The reconstruction loss weights lip and face vertices at 10^5 and velocity/acceleration terms at 10^2, ensuring both spatial accuracy and temporal smoothness.

**Three-stage curriculum training.** Stage 1 pre-trains the Motion Connector with differential learning rates (LoRA at 1e-4, Connector at 1e-5) while the LLM is frozen. Stage 2 resets the LoRA weights to avoid overfitting, then jointly fine-tunes LoRA and Connector together, preserving linguistic capability. Stage 3 performs affective instruction tuning on ~1K emotion-rich samples, injecting nuanced expressiveness. The dataset (FLAME-QA, ~100K samples filtered from 800K VoxCeleb clips) pairs question audio with response audio and FLAME motion extracted via SMIRK, with transcripts cleaned by GPT-4.

## Step-by-Step Workflow

1. **Define the FLAME parameter space.** Set D=58 (50 expression + 3 jaw + 3 global pose + 2 eyelid). Use a fixed shape vector per identity. Implement the differentiable FLAME mesh function M(beta, psi, theta) that maps these parameters to a 5023x3 vertex mesh.

2. **Build the RVQ-VAE for motion tokenization.** Construct an encoder-decoder with base channels 128, 3 residual blocks each, latent dim 256. Use temporal grouping G=5 to compress 25 fps FLAME sequences to 5 Hz. Stack Nq=6 residual codebooks of size 256. Train with the composite loss: `L_rec = L_param + 1e5*(L_lips + L_face) + 1e2*(L_vel + L_acc)` plus codebook commitment loss (gamma=0.25).

3. **Prepare FLAME-QA format training data.** For each conversational sample: (a) transcribe audio with Whisper v3 Large, (b) extract FLAME parameters from video frames using SMIRK at 25 fps, (c) label speech emotion, (d) generate QA pairs from transcripts with an LLM, (e) synthesize question audio with a TTS model. Filter aggressively for quality (expect ~12% retention from raw clips).

4. **Set up the autoregressive LLM backbone.** Start from a speech-capable LLM (e.g., Qwen2.5-7B fine-tuned for interleaved text+audio token generation). Add LoRA adapters (rank=64, alpha=32, dropout=0.05). The LLM outputs interleaved text and audio tokens at 25 Hz.

5. **Attach the Motion Connector.** Build a 6-layer Transformer decoder (hidden dim 768, 12 attention heads, dropout 0.1). Extract LLM hidden states H during audio token generation. Downsample H from 25 Hz to 5 Hz via 1D convolution to match the RVQ temporal resolution (T/G). The Connector uses segment-wise autoregressive attention: current audio features as Queries, prior motion tokens as Keys/Values.

6. **Implement multi-head RVQ prediction.** Add Nq=6 classification heads to the Motion Connector output, one per codebook level. Each head predicts a discrete index (0-255). Train with cross-entropy loss per head. The unified loss is `L = L_seq + lambda * L_motion`.

7. **Execute three-stage training.** Stage 1: freeze LLM, train LoRA at 1e-4 and Connector at 1e-5 until convergence. Stage 2: discard LoRA, re-initialize fresh LoRA, jointly fine-tune LoRA + Connector. Stage 3: fine-tune on emotion-rich subset (~1K samples) for affective instruction tuning. Use 8x A100 GPUs, batch size 64 per GPU.

8. **Wire the inference pipeline.** User audio in -> LLM encodes and autoregressively generates text + audio tokens -> extract hidden states -> Motion Connector predicts RVQ codes in parallel with audio generation -> RVQ-VAE decoder reconstructs continuous FLAME parameters -> render mesh with a differentiable renderer. Target TTFT ~50 ms, TTFA ~535 ms, RTF < 1.0.

9. **Add auxiliary training tasks to prevent forgetting.** Include AQAA (audio question-audio answer), Repeat (speech resynthesis), TTS, and ASR tasks in the training mix. These stabilize the LLM's linguistic and acoustic capabilities during motion-aware fine-tuning.

10. **Evaluate with spatial and temporal metrics.** Measure MOD (Mouth Opening Difference, target < 6 mm), UFD (Upper Face Dynamics, higher = more expressive), temporal correlation (target > 0.4), lip-width correlation (target > 0.5), liveliness ratio (target near 1.0), and peak-align latency (target < 150 ms). Run user preference studies against baselines.

## Concrete Examples

**Example 1: Designing an RVQ-VAE for FLAME motion tokenization**

User: "I need to tokenize 25fps FLAME facial motion sequences into discrete codes for an autoregressive model. How should I set this up?"

Approach:
1. Define input as sequences of D=58 FLAME parameters at 25 fps
2. Apply temporal grouping (G=5) so the encoder processes chunks of 5 frames, reducing to 5 Hz
3. Build encoder: 1D conv layers with base channels 128, 3 residual blocks, output latent dim 256
4. Apply 6-level residual vector quantization, each codebook with 256 entries
5. Build symmetric decoder to reconstruct the 5-frame groups
6. Train with weighted reconstruction loss

Output:
```python
import torch
import torch.nn as nn

class MotionRVQVAE(nn.Module):
    def __init__(self, motion_dim=58, group_size=5, latent_dim=256,
                 n_codebooks=6, codebook_size=256, base_channels=128):
        super().__init__()
        self.group_size = group_size
        input_dim = motion_dim * group_size  # 58 * 5 = 290

        # Encoder: grouped frames -> latent
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, base_channels * 2),
            nn.ReLU(),
            *[ResBlock(base_channels * 2) for _ in range(3)],
            nn.Linear(base_channels * 2, latent_dim),
        )

        # Residual VQ: 6 codebooks, 256 entries each
        self.codebooks = nn.ModuleList([
            VectorQuantize(latent_dim, codebook_size)
            for _ in range(n_codebooks)
        ])

        # Decoder: latent -> reconstructed grouped frames
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, base_channels * 2),
            nn.ReLU(),
            *[ResBlock(base_channels * 2) for _ in range(3)],
            nn.Linear(base_channels * 2, input_dim),
        )

    def encode(self, x):
        # x: (B, T, 58) -> group into (B, T//G, 58*G)
        B, T, D = x.shape
        x = x.reshape(B, T // self.group_size, D * self.group_size)
        z = self.encoder(x)
        codes, quantized, commit_loss = [], z, 0.0
        residual = z
        for cb in self.codebooks:
            q, idx, loss = cb(residual)
            codes.append(idx)
            residual = residual - q
            quantized = quantized  # sum reconstructed from all levels
            commit_loss += loss
        return codes, quantized, commit_loss

    def decode(self, quantized):
        return self.decoder(quantized)
```

**Example 2: Building the Motion Connector that reads LLM hidden states**

User: "I have an LLM generating audio tokens at 25 Hz. I need a module that takes the LLM hidden states and predicts FLAME RVQ codes synchronized with the audio."

Approach:
1. Downsample LLM hidden states from 25 Hz to 5 Hz with a 1D convolution (kernel=5, stride=5)
2. Build a 6-layer Transformer decoder that cross-attends to prior motion context
3. Add 6 parallel classification heads for multi-codebook prediction
4. Use segment-wise autoregressive masking

Output:
```python
class MotionConnector(nn.Module):
    def __init__(self, llm_dim=4096, hidden_dim=768, n_heads=12,
                 n_layers=6, n_codebooks=6, codebook_size=256, group_size=5):
        super().__init__()
        # Downsample LLM hidden states: 25Hz -> 5Hz
        self.downsample = nn.Conv1d(llm_dim, hidden_dim,
                                     kernel_size=group_size,
                                     stride=group_size)

        # Transformer decoder layers with causal self-attn
        decoder_layer = nn.TransformerDecoderLayer(
            d_model=hidden_dim, nhead=n_heads,
            dim_feedforward=hidden_dim * 4, dropout=0.1,
            batch_first=True
        )
        self.decoder = nn.TransformerDecoder(decoder_layer, n_layers)

        # Multi-head RVQ prediction (one head per codebook level)
        self.rvq_heads = nn.ModuleList([
            nn.Linear(hidden_dim, codebook_size)
            for _ in range(n_codebooks)
        ])

    def forward(self, llm_hidden, motion_history_embed):
        # llm_hidden: (B, T_audio, llm_dim) from LLM during generation
        # motion_history_embed: (B, T_prev, hidden_dim) prior motion context
        h = self.downsample(llm_hidden.transpose(1, 2)).transpose(1, 2)

        # Segment-wise autoregressive: current audio as Q, history as KV
        causal_mask = nn.Transformer.generate_square_subsequent_mask(h.size(1))
        out = self.decoder(h, motion_history_embed, tgt_mask=causal_mask)

        # Predict RVQ codes at each hierarchy level
        logits = [head(out) for head in self.rvq_heads]  # list of (B, T', 256)
        return logits
```

**Example 3: Constructing a FLAME-QA dataset pipeline**

User: "I have a large video dataset of talking heads. How do I convert this into training data for an end-to-end audio avatar model?"

Approach:
1. Extract audio, transcribe with Whisper v3, filter low-confidence transcriptions
2. Extract per-frame FLAME parameters with SMIRK face tracker at 25 fps
3. Run speech emotion recognition to label emotional content
4. Use an LLM to pair transcripts into QA format and clean text
5. Synthesize question audio with TTS, keeping original response audio
6. Align and package as (question_audio, response_text, response_audio, response_flame)

Output:
```python
# Pipeline pseudocode for FLAME-QA construction
pipeline_config = {
    "source": "voxceleb2",          # ~800K raw clips
    "target_retention": 0.12,        # expect ~100K after filtering

    "step_1_transcribe": {
        "model": "whisper-v3-large",
        "min_confidence": 0.85,
        "language_filter": ["en"],
    },
    "step_2_flame_extract": {
        "tracker": "SMIRK",
        "fps": 25,
        "output_params": ["expression_50", "jaw_3", "global_pose_3", "eyelid_2"],
    },
    "step_3_emotion_label": {
        "model": "speech_emotion_recognition",
        "categories": ["neutral", "happy", "sad", "angry", "surprise"],
    },
    "step_4_qa_generation": {
        "model": "gpt-4",
        "task": "rewrite_as_qa_pair",
        "clean_disfluencies": True,
    },
    "step_5_question_tts": {
        "model": "IndexTTS2",
        "match_speaker": False,  # question comes from different speaker
    },
    "step_6_package": {
        "format": "qa_triplet",
        "fields": ["q_audio", "r_text", "r_audio", "r_flame_params"],
        "frame_alignment": "dtw",
    },
}
```

## Best Practices

- **Do:** Use temporal grouping (G=5) when tokenizing FLAME sequences -- it compresses 25 fps to 5 Hz, making autoregressive generation tractable without losing expressive detail.
- **Do:** Weight lip and face vertex losses heavily (1e5) relative to parameter-space L2 loss. Raw parameter error doesn't reflect perceptual importance; vertex-level supervision in the lip region is critical for intelligible lip sync.
- **Do:** Include velocity and acceleration terms in the reconstruction loss (weight 1e2). Without these, generated motion tends toward frame-by-frame jitter or over-smoothing.
- **Do:** Reset LoRA weights between training stages 1 and 2. The LoRA from stage 1 overfits to the connector pre-training objective and degrades linguistic performance if carried forward.
- **Avoid:** Training the full LLM backbone end-to-end. LoRA (rank 64) is sufficient and prevents catastrophic forgetting of language and speech capabilities.
- **Avoid:** Predicting FLAME parameters as continuous regression targets directly from the LLM. Discretization via RVQ-VAE converts the problem to next-token prediction, which is what autoregressive LLMs are optimized for.

## Error Handling

- **RVQ codebook collapse:** If certain codebook entries are never used, apply codebook reset (replace dead entries with encoder outputs). Monitor codebook utilization per level; all 6 levels should have >50% active codes.
- **Temporal jitter in generated motion:** Check that velocity/acceleration loss terms are active. If motion still jitters, increase temporal loss weight or add a post-hoc Savitzky-Golay filter (window=5, order=2) as a fallback.
- **Audio-motion desynchronization:** Verify the 1D convolution downsampling aligns audio token timestamps with FLAME frame indices. Off-by-one errors in the group boundary cause visible lip-sync drift. Test with a known utterance and measure peak-align latency (<150 ms is acceptable).
- **Catastrophic forgetting after Stage 3:** If the model loses conversational quality after affective tuning, reduce the emotion-rich fine-tuning set or mix in auxiliary tasks (ASR, TTS, Repeat) at a 4:1 ratio.
- **SMIRK tracking failures on training data:** Filter samples where SMIRK returns implausible jaw angles (>45 degrees) or expression coefficient magnitudes >4 standard deviations from the dataset mean.

## Limitations

- Requires a speech-capable LLM backbone (e.g., Step-Audio-2-mini or equivalent). You cannot bolt this onto a text-only LLM without first adding audio token generation capability.
- FLAME's 58-parameter space cannot represent fine-grained details like wrinkles, tongue, or teeth. For photorealistic rendering, an additional neural renderer (e.g., Gaussian splatting) is needed on top of the FLAME mesh.
- The FLAME-QA dataset construction pipeline requires SMIRK or equivalent face tracker, which needs frontal or near-frontal video. Profile views and heavy occlusion produce unusable training data.
- Training requires 8x A100 80GB GPUs. The RVQ-VAE alone is lightweight, but joint fine-tuning of a 7B LLM with LoRA plus the Motion Connector demands significant compute.
- The system is designed for single-speaker avatar generation. Multi-party conversation with multiple avatars would require separate Motion Connectors or identity conditioning.
- Real-time performance (0.7 RTF) assumes optimized inference with KV caching and batched RVQ decoding. Naive implementations will not meet the latency targets.

## Reference

**Paper:** [A2-LLM: An End-to-end Conversational Audio Avatar Large Language Model](https://arxiv.org/abs/2602.04913v1) (Hu et al., 2026). Look for: the three-stage curriculum training procedure (Section 3.3), the RVQ-VAE motion tokenization architecture (Section 3.2), and the audio-anchored Motion Connector design (Section 3.2) that grounds facial generation in LLM hidden states.