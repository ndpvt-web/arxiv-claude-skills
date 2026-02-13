---
name: "ex-omni-enabling-3d-facial"
description: "Build pipelines that generate synchronized 3D facial animation alongside speech from omni-modal LLMs, using decoupled semantic/temporal generation and gated fusion. Use when: 'generate 3D facial animation from text', 'build a talking head pipeline with blendshapes', 'add facial motion output to a speech LLM', 'synchronize ARKit blendshapes with TTS output', 'create an avatar animation system driven by an LLM', 'implement token-as-query gated fusion for multimodal generation'."
---

# Ex-Omni: Speech-Accompanied 3D Facial Animation from Omni-Modal LLMs

This skill enables Claude to design and implement systems that augment large language models with synchronized speech and 3D facial animation generation. The core technique, from the Ex-Omni paper, decouples high-level semantic reasoning (what to say, what emotion to convey) from dense temporal generation (frame-by-frame blendshape coefficients), using discrete speech units as an intermediate temporal scaffold and a Token-as-Query Gated Fusion (TQGF) mechanism for controlled semantic injection. This solves the fundamental mismatch between discrete LLM token prediction and the continuous, fine-grained dynamics required for realistic facial motion.

## When to Use

- When the user wants to build a pipeline that takes text input and produces both speech audio and synchronized 3D facial animation (ARKit blendshapes, FLAME parameters, or vertex displacements).
- When integrating a facial animation decoder into an existing omni-modal LLM (e.g., Qwen2.5-Omni, GPT-4o-style models) that already handles text and speech.
- When the user needs to generate ARKit-52 blendshape coefficient sequences aligned to speech unit boundaries.
- When designing a training pipeline that must handle the data scarcity problem for paired speech-face data by using staged training with synthetic annotations.
- When implementing cross-modal fusion where one modality (LLM semantic tokens) must condition another (dense temporal sequences) without overwhelming the temporal structure.
- When building real-time avatar systems that need lip-sync and expressive facial motion driven by conversational AI output.

## Key Technique

**The Representation Mismatch Problem.** LLMs operate on discrete tokens at a semantic level — one token might represent an entire syllable or concept. 3D facial animation requires dense, continuous output at 25-60 fps, where each frame contains 52 blendshape coefficients (in ARKit format). Directly predicting facial motion from LLM tokens fails because the optimization landscape is too difficult: the LLM must simultaneously learn what to express and how to temporally distribute that expression across hundreds of frames. Under limited paired data, this joint learning collapses.

**Decoupled Generation with Speech Unit Scaffolding.** Ex-Omni breaks generation into two stages. First, the LLM autoregressively predicts discrete speech units — compact representations that encode phonetic and prosodic content at a temporal granularity between tokens and audio frames. These speech units serve as a temporal scaffold: they carry timing, rhythm, and phonetic information that facial motion can latch onto. Second, a non-autoregressive facial decoder takes these speech units (temporally resampled via linear interpolation to the target video frame rate) as frame-level queries and produces blendshape coefficients in parallel. This separation means the LLM only needs to learn semantic-to-speech-unit mapping (a tractable discrete prediction), while the facial decoder only needs to learn speech-unit-to-motion mapping (a well-constrained temporal regression).

**Token-as-Query Gated Fusion (TQGF).** To inject semantic context (emotion, emphasis, speaking style) from the LLM into both the speech unit predictor and facial decoder without disrupting their temporal structure, Ex-Omni uses TQGF: `Q + σ(G(Q)) ⊙ Attn(Q, C)`, where Q is the target temporal sequence (speech units or facial frames), C is the LLM's semantic context, G(·) produces per-head gating factors, and σ is sigmoid. The gating ensures semantic information is injected selectively — the temporal query always retains its baseline structure, and cross-attended semantics are added only where the gate opens. This prevents semantic flooding that would destabilize temporal coherence.

## Step-by-Step Workflow

1. **Define the facial motion representation.** Choose ARKit-52 blendshape coefficients as the output format (52 floats per frame). Set the target frame rate (typically 30 fps). Each output frame `y_t ∈ R^52` maps directly to standard ARKit blendshape weights renderable in Blender, Unity, or Unreal Engine.

2. **Set up the speech unit extraction pipeline.** Use a self-supervised speech model (e.g., HuBERT, wav2vec 2.0) to extract discrete speech units from audio. Cluster the representations with k-means (typically 500-2000 clusters) to get a codebook of discrete unit IDs. These units run at ~50 Hz (one unit per 20ms frame), providing the temporal scaffold between LLM tokens (~3-5 per second) and video frames (30-60 fps).

3. **Prepare the LLM backbone with speech projector.** Take a pretrained omni-modal LLM (e.g., Qwen2.5-Omni or similar). Add a speech encoder projector (θ_P) that maps continuous speech encoder features into the LLM's token embedding space. Add a speech unit prediction head (θ_U) that predicts discrete unit IDs autoregressively from the LLM's hidden states, conditioned via TQGF on the LLM's semantic representations.

4. **Implement the TQGF module.** Build a cross-attention layer where: queries Q come from the target temporal sequence (speech units during unit prediction, or resampled units during facial decoding), keys/values C come from the LLM's last-layer hidden states. Add a gating network G(Q) that outputs per-attention-head scalar gates, passed through sigmoid. The fused output is `Q + σ(G(Q)) ⊙ CrossAttn(Q, C)`. Initialize G's bias negative so gates start near-closed, allowing gradual semantic injection during training.

5. **Build the non-autoregressive facial decoder (θ_F).** Design a transformer decoder that takes temporally resampled speech units as frame-level queries (interpolated from ~50 Hz to target fps via linear interpolation). Each query attends to neighboring speech units (local temporal context) and to LLM semantic tokens via a second TQGF layer. The output head projects to 52-dimensional blendshape coefficients per frame with sigmoid activation (blendshapes are bounded [0,1]).

6. **Construct the staged training pipeline.** Stage I: Train speech encoder + projector on ASR data (>3000 hours) with LLM frozen. Stage II: Train speech unit generator on TTS data (>6000 hours) with LLM frozen. Stage III: Co-train speech unit generator and facial decoder on paired TTS+blendshape data (~250 hours, using synthetic blendshapes from a model like NVIDIA Audio2Face-3D). Stage IV: Joint fine-tune all parameters on mixed-modality data including speech-to-speech + face, ASR, TTS+face, and text-to-text tasks.

7. **Generate synthetic facial animation training data.** Since paired speech-face data is scarce, use a pretrained audio-driven facial animation model (e.g., NVIDIA Audio2Face-3D) to generate ARKit-52 blendshape annotations from speech audio. Filter for quality: discard sequences where lip vertex error (LVE) against a reference exceeds a threshold. This synthetic data bootstraps Stage III training.

8. **Implement the inference pipeline.** At inference: (a) the LLM processes the text/audio input and produces hidden states, (b) the speech unit generator autoregressively predicts unit IDs conditioned on LLM states via TQGF, (c) unit IDs are converted to continuous embeddings and temporally resampled to the target frame rate, (d) the facial decoder produces blendshape coefficients non-autoregressively for all frames in parallel, (e) a vocoder (e.g., CosyVoice 2) synthesizes audio from the speech units, (f) blendshapes and audio are packaged together with matching timestamps.

9. **Evaluate with appropriate metrics.** Use Lip Vertex Error (LVE): render predicted and ground-truth blendshapes onto a reference mesh, compute L2 distance over lip vertices. Use FID on blendshape distributions for expressiveness. For speech quality, measure WER on the synthesized speech. For alignment, check that viseme timing matches phoneme boundaries within ±2 frames.

10. **Render and deploy.** Apply blendshape weights to a FACS-compatible 3D face mesh in Blender (offline) or Unity/Unreal (real-time). Stream blendshape coefficients at the target frame rate alongside audio for real-time avatar applications.

## Concrete Examples

**Example 1: Adding facial animation output to an existing speech LLM**

User: "I have a Qwen2.5-based model that generates speech tokens. I want to add 3D facial animation output synchronized with the speech."

Approach:
1. Keep the existing LLM and speech unit prediction head untouched.
2. Add a TQGF-conditioned facial decoder module after the speech unit prediction stage.
3. Extract speech unit embeddings from the existing unit prediction head.
4. Temporally resample unit embeddings from 50 Hz to 30 fps using `torch.nn.functional.interpolate(units.unsqueeze(0), size=T_video, mode='linear')`.
5. Build the facial decoder as a 4-layer transformer decoder with TQGF cross-attention to the LLM's hidden states.
6. Train only the facial decoder (θ_F) on paired speech-blendshape data while freezing everything else.

Output:
```python
class FacialDecoder(nn.Module):
    def __init__(self, d_model=512, n_heads=8, n_layers=4, n_blendshapes=52):
        super().__init__()
        self.unit_proj = nn.Linear(d_unit, d_model)
        self.layers = nn.ModuleList([
            FacialDecoderLayer(d_model, n_heads) for _ in range(n_layers)
        ])
        self.head = nn.Sequential(nn.Linear(d_model, n_blendshapes), nn.Sigmoid())

    def forward(self, speech_units, llm_hidden_states, target_fps=30, unit_fps=50):
        # Resample speech units to video frame rate
        units = self.unit_proj(speech_units)  # (B, T_unit, D)
        T_video = int(units.shape[1] * target_fps / unit_fps)
        frames = F.interpolate(
            units.permute(0, 2, 1), size=T_video, mode='linear'
        ).permute(0, 2, 1)  # (B, T_video, D)

        for layer in self.layers:
            frames = layer(frames, llm_hidden_states)  # TQGF inside each layer
        return self.head(frames)  # (B, T_video, 52)
```

**Example 2: Implementing the TQGF mechanism**

User: "How do I implement Token-as-Query Gated Fusion so that my temporal decoder can selectively use LLM semantic context?"

Approach:
1. Build standard multi-head cross-attention (Q from temporal sequence, K/V from LLM context).
2. Add a gating network that takes Q and outputs per-head scalar gates.
3. Apply sigmoid to gates and element-wise multiply with attention output.
4. Add the gated result as a residual to the original Q.
5. Initialize gate bias to -2.0 so gates start near-closed (~0.12).

Output:
```python
class TQGF(nn.Module):
    """Token-as-Query Gated Fusion: selective semantic injection."""
    def __init__(self, d_model, n_heads, d_head=64):
        super().__init__()
        self.cross_attn = nn.MultiheadAttention(d_model, n_heads, batch_first=True)
        # Per-head gating: project Q to n_heads scalars
        self.gate_proj = nn.Linear(d_model, n_heads)
        # Initialize bias negative so gates start near-closed
        nn.init.constant_(self.gate_proj.bias, -2.0)
        self.n_heads = n_heads
        self.d_head = d_head

    def forward(self, Q, C):
        """
        Q: (B, T_temporal, D) - temporal queries (speech units or facial frames)
        C: (B, T_semantic, D) - LLM hidden states (semantic context)
        Returns: (B, T_temporal, D) - Q with selective semantic injection
        """
        # Cross-attention: Q attends to semantic context C
        attn_out, _ = self.cross_attn(Q, C, C)  # (B, T_temporal, D)

        # Per-head gating
        gates = torch.sigmoid(self.gate_proj(Q))  # (B, T_temporal, n_heads)
        # Expand gates to match head dimension
        gates = gates.unsqueeze(-1).expand(-1, -1, -1, self.d_head)
        gates = gates.reshape(Q.shape[0], Q.shape[1], -1)  # (B, T, D)

        # Gated residual: temporal structure preserved, semantics injected selectively
        return Q + gates * attn_out
```

**Example 3: Building a synthetic training data pipeline**

User: "I don't have paired speech-blendshape data. How do I create training data for the facial decoder?"

Approach:
1. Collect a large speech corpus (e.g., LibriSpeech, Emilia).
2. Run NVIDIA Audio2Face-3D (or a similar audio-driven face model) on each audio clip to produce ARKit-52 blendshape sequences.
3. Extract speech units from the same audio using HuBERT + k-means.
4. Filter: discard samples where the blendshape variance is below a threshold (static faces) or where the model confidence is low.
5. Store as `(speech_units, blendshape_sequence, llm_text_embedding)` triples.

Output:
```python
import numpy as np
from pathlib import Path

def build_synthetic_dataset(audio_dir, output_dir, a2f_model, hubert_model, kmeans):
    """Generate paired speech-unit / blendshape training data."""
    pairs = []
    for audio_path in Path(audio_dir).glob("*.wav"):
        # 1. Generate blendshapes from audio via Audio2Face-3D
        blendshapes = a2f_model.predict(audio_path)  # (T_video, 52)

        # 2. Quality filter: skip low-motion sequences
        if blendshapes.std(axis=0).mean() < 0.01:
            continue

        # 3. Extract speech units
        features = hubert_model.extract_features(audio_path)  # (T_unit, D)
        unit_ids = kmeans.predict(features)  # (T_unit,)

        # 4. Save aligned pair
        np.savez(
            output_dir / f"{audio_path.stem}.npz",
            speech_units=unit_ids,
            blendshapes=blendshapes.astype(np.float32),
            unit_fps=50,
            video_fps=30,
        )
        pairs.append(audio_path.stem)

    print(f"Generated {len(pairs)} training pairs")
    return pairs
```

## Best Practices

- **Do:** Initialize TQGF gate biases to negative values (e.g., -2.0). This ensures gates start near-closed, so the temporal sequence maintains its structure early in training and semantic injection ramps up gradually as the model learns when context is helpful.
- **Do:** Use staged training — freeze the LLM when training downstream decoders. Unfreezing everything simultaneously on limited facial data will catastrophically forget the LLM's language capabilities.
- **Do:** Temporally resample speech units with linear interpolation rather than nearest-neighbor. Linear interpolation produces smoother frame-level queries, which reduces jitter in the decoded facial motion.
- **Do:** Evaluate lip sync with Lip Vertex Error (LVE) on rendered meshes, not just blendshape MSE. Two blendshape sets can have low coefficient error but produce visually different lip shapes due to non-linear mesh deformation.
- **Avoid:** Predicting blendshape coefficients autoregressively. Non-autoregressive parallel decoding is faster and avoids error accumulation that causes drift in long sequences. The speech units already provide temporal structure.
- **Avoid:** Using raw audio features as queries for the facial decoder. Discrete speech units are a better scaffold because they compress away acoustic variability (speaker identity, recording conditions) while retaining phonetic and prosodic timing.

## Error Handling

- **Temporal misalignment between speech and face:** If blendshapes visibly lag or lead the audio, check the fps assumptions in the resampling step. Ensure `unit_fps` and `target_fps` match the actual data rates. Off-by-one errors in frame counts compound over long sequences.
- **Gate collapse (all gates near 0 or 1):** Monitor gate activation statistics during training. If all gates saturate to 1, the gating is not selective — reduce learning rate on gate parameters or increase the negative bias initialization. If all gates collapse to 0, the semantic signal is being ignored — check that LLM hidden states are properly detached/not detached as intended.
- **Jaw/lip jitter in generated animation:** Apply temporal smoothing as a post-processing step (Gaussian filter with σ=1-2 frames on blendshape trajectories). If jitter persists, increase the local temporal context window in the facial decoder's self-attention.
- **Mode collapse to neutral face:** The facial decoder may learn to predict near-zero blendshapes if the training data is dominated by neutral expressions. Oversample expressive segments or add a diversity loss penalizing low variance in predicted blendshape sequences.
- **Out-of-vocabulary speech units:** If the HuBERT/k-means codebook encounters unseen acoustic conditions at inference, unit predictions may be noisy. Use a larger codebook (1024+ clusters) or add codebook dropout during training for robustness.

## Limitations

- Requires a pretrained speech unit extraction pipeline (HuBERT + k-means or equivalent) and a pretrained audio-driven face model for synthetic data generation. These are non-trivial dependencies.
- The paper's paired speech-face training data is synthetic (generated by Audio2Face-3D), which introduces a ceiling on realism — the facial decoder can only be as good as the annotation model.
- ARKit-52 blendshapes cover facial expressions but not head pose, gaze, or body gestures. A complete avatar system needs additional channels beyond what this pipeline produces.
- Non-autoregressive facial decoding assumes the full speech unit sequence is available, which adds latency equal to the full utterance duration. Streaming/chunked generation requires architectural modifications not covered in the paper.
- Performance is validated primarily on English and Chinese speech. Other languages may require codebook and training data adjustments.
- The technique assumes access to a capable base OLLM (7B+ parameters). Smaller models may lack the semantic richness needed for expressive gating.

## Reference

**Paper:** [Ex-Omni: Enabling 3D Facial Animation Generation for Omni-modal Large Language Models](https://arxiv.org/abs/2602.07106v1) — Zhang et al., 2026. Focus on Section 3 (method) for the TQGF mechanism and decoupled generation architecture, Section 4 for the InstructEx dataset construction and staged training protocol, and Section 5 for ablation studies showing the contribution of gating vs. ungated fusion.