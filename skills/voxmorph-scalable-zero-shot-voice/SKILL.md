---
name: "voxmorph-scalable-zero-shot-voice"
description: "Build and deploy zero-shot voice identity morphing pipelines using disentangled prosody/timbre embeddings and Spherical Linear Interpolation. Use when the user asks to: 'morph two voice identities', 'blend speaker embeddings with Slerp', 'build a voice morphing pipeline', 'disentangle prosody and timbre from speech', 'test biometric spoofing defenses against voice morphs', 'interpolate speaker identity embeddings'."
---

# VoxMorph: Zero-Shot Voice Identity Morphing via Disentangled Embeddings

This skill enables Claude to help users implement, customize, and deploy voice identity morphing systems based on the VoxMorph framework. The core technique disentangles speech into two independent embedding spaces -- prosody (rhythm, pitch, speaking style) and timbre (vocal tract characteristics, formant structure) -- then fuses them via Spherical Linear Interpolation (Slerp) on a hypersphere before resynthesizing speech through an autoregressive language model and Conditional Flow Matching vocoder. This produces morphed voices that fool automated speaker verification systems at a 67.8% success rate while maintaining high audio quality and intelligibility.

## When to Use

- When the user wants to build a voice morphing or voice blending pipeline from audio samples
- When implementing Spherical Linear Interpolation (Slerp) for embedding fusion in any modality (voice, face, latent space)
- When disentangling speech attributes (prosody vs. timbre) for controllable synthesis
- When testing speaker verification systems against morphing attacks (biometric security research)
- When building a zero-shot voice cloning or voice conversion system that needs only 5 seconds of reference audio
- When the user needs to interpolate between two identity embeddings while preserving geometric structure
- When constructing evaluation pipelines for voice morphs using FAD, WER, EER, and MMPMR metrics

## Key Technique

**Disentangled Embedding Extraction.** VoxMorph separates each speaker's voice into two independent representations: a prosody embedding (e^P) encoding rhythm, pitch contour, and speaking cadence via an LSTM-based GE2E encoder, and a timbre embedding (e^T) capturing vocal identity through formant structure and tract resonance via ECAPA-TDNN, Wav2Vec2, or HuBERT encoders. This separation is critical -- prior methods that morphed a single monolithic speaker embedding produced artifacts because style and identity changes are entangled. By operating on each axis independently, the morph ratio (alpha) can weight style and identity differently if needed.

**Spherical Linear Interpolation (Slerp).** Instead of naive linear interpolation (`(1-a)*A + a*B`), which shrinks magnitude and distorts the embedding space, VoxMorph interpolates along the great circle arc on the unit hypersphere: `e_alpha = [sin((1-alpha)*Omega) / sin(Omega)] * e_A + [sin(alpha*Omega) / sin(Omega)] * e_B`, where Omega is the angular separation between the two embeddings. This preserves the norm and geometric structure of the embedding manifold, producing smoother identity transitions. The same Slerp is applied independently to both prosody and timbre embeddings.

**Autoregressive LM + Conditional Flow Matching Synthesis.** The fused prosody embedding conditions a Llama-based autoregressive language model that generates discrete speech tokens, while the timbre embedding conditions a Conditional Flow Matching (CFM) network that produces mel-spectrograms. A HiFTNet vocoder converts spectrograms to waveforms. This two-stage design (discrete tokens for structure, continuous flow for acoustics) yields a 2.6x audio quality improvement over prior voice morphing methods and a 73% reduction in word error rate.

## Step-by-Step Workflow

1. **Install the VoxMorph framework.** Clone `https://github.com/Bharath-K3/VoxMorph`, install Python 3.11 dependencies via `pip install -r requirements.txt`, and ensure CUDA-capable PyTorch is available. Download pretrained models from `BharathK333/VoxMorph-Models` on Hugging Face.

2. **Prepare source audio.** Collect at least 5 seconds of clean speech per speaker. For multi-shot mode (higher quality), place multiple clips per speaker in separate directories under `data/`. The framework auto-consolidates clips into stable speaker profiles.

3. **Select the speaker encoder.** Choose between ECAPA-TDNN (default, best for speaker verification compatibility), Wav2Vec2 (strongest self-supervised features), or HuBERT (best phonetic disentanglement) via the `--encoder` flag or `config.yaml`.

4. **Extract disentangled embeddings.** Run the pipeline to independently extract prosody embeddings (e^P) and timbre embeddings (e^T) for both speakers A and B. The framework handles this automatically during inference.

5. **Set the morphing weight alpha.** Choose alpha in [0.0, 1.0] where 0.0 = pure speaker A and 1.0 = pure speaker B. Start with 0.5 for an even blend. For biometric security testing, sweep alpha from 0.3 to 0.7 in 0.1 increments to find the optimal attack point.

6. **Apply Slerp fusion.** The framework computes `Omega = arccos(dot(e_A, e_B) / (||e_A|| * ||e_B||))` and interpolates both prosody and timbre embeddings along the great circle. Verify that Omega > 0.01 radians to avoid numerical instability from near-identical embeddings.

7. **Synthesize morphed speech.** The autoregressive LM generates speech tokens conditioned on the fused prosody, the CFM network produces mel-spectrograms conditioned on fused timbre, and HiFTNet reconstructs the waveform. Output includes three files: speaker A clone, speaker B clone, and the morph.

8. **Evaluate the morph.** Measure audio quality with UTMOS/FAD (target FAD < 5.0), intelligibility with word error rate (target WER < 0.20), and biometric attack success with EER and MMPMR against a speaker verification system at FMR thresholds of 0.1% and 0.01%.

9. **Iterate on encoder and alpha.** If the morph sounds unnatural, try a different encoder. If the morphing attack success rate is low, adjust alpha toward the target speaker (higher alpha to sound more like speaker B). If intelligibility degrades, reduce alpha or use longer source audio.

10. **Deploy or integrate.** Use `app.py` for a Gradio web interface, `inference.py` for batch CLI processing, or import the pipeline components directly into a Python application for programmatic access.

## Concrete Examples

**Example 1: Basic two-speaker voice morph via CLI**

User: "I have two audio files, alice.wav and bob.wav. I want to create a morphed voice that's halfway between them."

Approach:
1. Verify audio files are at least 5 seconds of clean speech
2. Run the morphing pipeline with alpha=0.5
3. Output three comparison files

```bash
# Clone and setup
git clone https://github.com/Bharath-K3/VoxMorph.git
cd VoxMorph
pip install -r requirements.txt

# Run morphing with even blend
python inference.py \
  --source_a "/path/to/alice.wav" \
  --source_b "/path/to/bob.wav" \
  --alpha 0.5 \
  --text "The quick brown fox jumps over the lazy dog." \
  --output_dir "outputs/alice_bob_morph"
```

Output: `outputs/alice_bob_morph/` contains `speaker_a_clone.wav`, `speaker_b_clone.wav`, and `morphed_alpha_0.5.wav`.

---

**Example 2: Implementing Slerp for custom embedding interpolation**

User: "I have my own speaker embeddings from a different model. How do I implement the Slerp fusion VoxMorph uses?"

Approach:
1. Implement the Slerp formula with numerical safeguards
2. Apply independently to prosody and timbre dimensions
3. Normalize inputs to unit sphere before interpolation

```python
import numpy as np

def slerp(embedding_a: np.ndarray, embedding_b: np.ndarray, alpha: float) -> np.ndarray:
    """Spherical Linear Interpolation between two embeddings on the unit hypersphere.

    Args:
        embedding_a: First speaker embedding (will be L2-normalized).
        embedding_b: Second speaker embedding (will be L2-normalized).
        alpha: Blend weight in [0, 1]. 0 = pure A, 1 = pure B.

    Returns:
        Interpolated embedding on the unit hypersphere.
    """
    # Normalize to unit sphere
    a = embedding_a / np.linalg.norm(embedding_a)
    b = embedding_b / np.linalg.norm(embedding_b)

    # Compute angular separation
    dot = np.clip(np.dot(a, b), -1.0, 1.0)
    omega = np.arccos(dot)

    # Fall back to linear interpolation for near-identical embeddings
    if omega < 1e-6:
        return (1.0 - alpha) * a + alpha * b

    sin_omega = np.sin(omega)
    coeff_a = np.sin((1.0 - alpha) * omega) / sin_omega
    coeff_b = np.sin(alpha * omega) / sin_omega

    return coeff_a * a + coeff_b * b

# Usage: morph prosody and timbre independently
prosody_morphed = slerp(prosody_a, prosody_b, alpha=0.5)
timbre_morphed = slerp(timbre_a, timbre_b, alpha=0.5)
```

---

**Example 3: Biometric security evaluation sweep**

User: "I need to test our speaker verification system against morphing attacks across multiple alpha values."

Approach:
1. Generate morphs at alpha increments from 0.3 to 0.7
2. Run each morph against the speaker verification system for both enrolled speakers
3. Compute MMPMR at FMR thresholds of 0.1% and 0.01%

```bash
#!/bin/bash
# Sweep alpha values for morphing attack evaluation
for alpha in 0.3 0.4 0.5 0.6 0.7; do
  python VoxMorph.py \
    --source_a "data/speaker_1_dir" \
    --source_b "data/speaker_2_dir" \
    --alpha "$alpha" \
    --encoder ecapa \
    --output_dir "security_eval/alpha_${alpha}"
done

# Then run speaker verification scoring against each morph
# to compute Mated Morph Presentation Match Rate (MMPMR)
```

```python
# Compute MMPMR: fraction of morphs accepted as BOTH enrolled speakers
def compute_mmpmr(morph_scores_a, morph_scores_b, threshold):
    """
    morph_scores_a: similarity scores of morphs vs. enrolled speaker A
    morph_scores_b: similarity scores of morphs vs. enrolled speaker B
    threshold: decision threshold at target FMR (e.g., 0.01%)
    """
    accepted_both = sum(
        1 for sa, sb in zip(morph_scores_a, morph_scores_b)
        if sa >= threshold and sb >= threshold
    )
    return accepted_both / len(morph_scores_a)
```

Expected results at alpha=0.5: MMPMR ~67.8% at FMR=0.01%, indicating the morph fools the verification system for both identities in two-thirds of cases.

## Best Practices

- **Do:** Always L2-normalize embeddings before Slerp. The interpolation formula assumes unit vectors; unnormalized inputs produce distorted results that drift off the embedding manifold.
- **Do:** Use multi-shot mode (multiple clips per speaker in a directory) for production-quality morphs. A single 5-second clip works for zero-shot, but 30+ seconds consolidated from multiple utterances yields more stable speaker profiles.
- **Do:** Generate the triplet output (clone A, clone B, morph) for every experiment. Comparing the morph against clean clones is essential for diagnosing whether artifacts come from the morphing step or the synthesis step.
- **Do:** Test multiple encoder backends (ECAPA-TDNN, Wav2Vec2, HuBERT) when morphing acoustically dissimilar speakers. Different encoders capture different aspects of identity and one may disentangle better for a given pair.
- **Avoid:** Using linear interpolation (`lerp`) instead of Slerp for speaker embeddings. Linear blending shrinks the magnitude toward zero at alpha=0.5, producing washed-out, identity-less speech. Slerp maintains constant norm throughout the arc.
- **Avoid:** Morphing speakers with fewer than 3 seconds of audio. Below this threshold, the prosody encoder cannot extract stable rhythmic patterns and the morph quality degrades sharply.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| `Omega ~ 0` in Slerp (near-zero angle) | Speakers have near-identical embeddings | Fall back to linear interpolation; consider whether the pair is too similar to morph meaningfully |
| Garbled or unintelligible output | Insufficient source audio or heavy background noise | Use cleaner audio, increase duration to 10+ seconds, or apply denoising preprocessing |
| Low MMPMR despite high alpha | Speaker pair is acoustically very different | Try alpha=0.6-0.7 biased toward the target, or switch to a different encoder that better bridges the identity gap |
| CUDA out of memory during synthesis | Autoregressive LM + CFM + vocoder exceed GPU VRAM | Reduce batch size, use shorter synthesis text, or offload the vocoder to CPU |
| `NaN` in generated audio | Numerical instability in flow matching | Check that input embeddings are finite and normalized; reduce CFM step count if using custom settings |
| Poor prosody in morph | Prosody embedding dominated by one speaker | Adjust prosody alpha independently from timbre alpha if the framework supports decoupled blending |

## Limitations

- **Authorized use only.** Voice morphing for identity spoofing is illegal in most jurisdictions. This skill is for biometric security research, defensive testing, CTF challenges, and academic study. Always obtain consent from voice subjects.
- **English-centric.** The pretrained models are trained primarily on English speech data. Morphing quality degrades for tonal languages (Mandarin, Vietnamese) where prosody carries lexical meaning.
- **GPU required.** The autoregressive LM and CFM synthesis pipeline require CUDA-capable hardware. CPU-only inference is impractically slow for real-time or batch use.
- **Same-language constraint.** Both source speakers should speak the same language. Cross-lingual morphing is not supported and produces intelligibility failures.
- **Text-dependent synthesis.** The current pipeline generates morphed speech for a given text input, not arbitrary free-form conversion of existing recordings. It is a synthesis-based morph, not a signal-processing waveform blend.
- **Alpha is global.** The public implementation applies the same alpha to both prosody and timbre. Independent per-axis alpha control requires modifying `VoxMorph.py` to accept separate `--alpha_prosody` and `--alpha_timbre` arguments.

## Reference

**Paper:** [VoxMorph: Scalable Zero-shot Voice Identity Morphing via Disentangled Embeddings](https://arxiv.org/abs/2601.20883v1) (IEEE ICASSP 2026). Look for Section III on the Slerp fusion formula and Table 2 comparing MMPMR across encoder types and alpha values.

**Code:** [github.com/Bharath-K3/VoxMorph](https://github.com/Bharath-K3/VoxMorph) | **Models:** [HuggingFace BharathK333/VoxMorph-Models](https://huggingface.co/BharathK333/VoxMorph-Models) | **Demo:** [HuggingFace Spaces](https://huggingface.co/spaces/BharathK333/VoxMorph)