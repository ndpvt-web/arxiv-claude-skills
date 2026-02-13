---
name: "soundbreak-systematic-study-audio-only"
description: |
  Implement and evaluate audio-only adversarial attacks against trimodal (audio-video-language) models using the SoundBreak framework.
  Covers six attack objectives targeting encoder representations, cross-modal attention, hidden states, and output likelihoods via PGD optimization.
  Also guides building defenses that enforce cross-modal consistency.
  Trigger phrases:
  - "adversarial attack on multimodal model"
  - "audio adversarial perturbation"
  - "test robustness of audio-video-language model"
  - "cross-modal attention attack"
  - "defend multimodal model against audio attacks"
  - "SoundBreak attack implementation"
---

# SoundBreak: Audio-Only Adversarial Attacks on Trimodal Models

This skill enables Claude to implement, evaluate, and defend against audio-only adversarial attacks on trimodal (audio-video-language) foundation models. Based on the SoundBreak framework, it covers six complementary attack objectives---targeting audio encoder embeddings, cross-modal attention distributions, hidden-state representations, and output token likelihoods---all using Projected Gradient Descent (PGD) with L-infinity bounded perturbations. The skill also guides construction of cross-modal consistency defenses that detect when a single-modality perturbation causes disproportionate disruption across modalities.

## When to Use

- When the user wants to **red-team a multimodal model** (e.g., VideoLLaMA2, Qwen2.5-Omni) by crafting adversarial audio inputs in an authorized security testing context
- When building a **robustness evaluation pipeline** that measures how audio perturbations degrade video-QA or audio-visual dialogue accuracy
- When implementing **PGD-based adversarial optimization** constrained to the audio modality while leaving video and text untouched
- When designing **cross-modal consistency defenses** that detect attention reallocation or hidden-state divergence caused by adversarial audio
- When analyzing **attack transferability** across different audio encoders (BEATs, Whisper-style) or across model families
- When measuring **perceptual distortion** of adversarial audio using LPIPS on log-mel spectrograms and SI-SNR in the waveform domain

## Key Technique

SoundBreak defines six loss functions that target distinct stages of trimodal processing. The most effective is **Encoder Cosine Similarity (L_cos)**, which minimizes the cosine similarity between clean and perturbed audio embeddings in the encoder's representation space, achieving up to 96% attack success rate (ASR) on AVQA while keeping perceptual distortion low (LPIPS <= 0.08, SI-SNR >= 0 dB). In contrast, **Negative Language Modeling (L_negLM)** directly suppresses the log-probability of the correct answer but produces higher perceptual distortion (LPIPS ~0.22). Attention-based objectives---**Vision Attention Suppression**, **Audio Attention Amplification**, and **Attention Randomization**---manipulate the cross-modal attention distribution, forcing the model to ignore visual tokens or attend randomly. **Hidden-State Similarity (L_hidden-cos)** disrupts internal transformer representations averaged across layers.

All attacks use iterative PGD: `delta^(k+1) = project_C(delta^(k) - eta * grad_delta L)`, where the projection operator enforces an L-infinity budget (epsilon in [0.3, 1.0] on normalized waveforms). A critical finding is that **extended optimization on smaller datasets outperforms limited iterations on larger data**, meaning attack quality scales with compute, not data volume. Perturbations are optimized as shared (universal) noise across a training set, not per-sample.

Transferability across models and encoder architectures is severely limited---attacks trained on VideoLLaMA2's BEATs encoder achieve only 0.50 cosine similarity on BEATs but 0.75 on Qwen's Whisper-style encoder, confirming that adversarial perturbations exploit model-specific feature geometries rather than universal multimodal vulnerabilities. This motivates defenses based on cross-modal consistency: if audio perturbations cause large hidden-state divergence or attention reallocation without corresponding changes in other modalities, the input is likely adversarial.

## Step-by-Step Workflow

1. **Select the target trimodal model and extract its audio processing pipeline.** Identify the audio encoder (e.g., BEATs for VideoLLaMA2, Whisper-style for Qwen-Omni), the projection layer mapping audio embeddings into the LLM's token space, and the cross-modal attention layers. You need gradient access through these components.

2. **Prepare the evaluation dataset with ground-truth answers.** Load a video-QA benchmark (AVQA, Music-AVQA) or dialogue benchmark (AVSD). Extract the audio waveform from each video, keeping video frames and text questions as fixed context. Record the model's clean predictions and accuracy as the baseline.

3. **Define the attack objective from the six SoundBreak losses.** Choose based on your goal:
   - `L_cos`: Maximize encoder-space disruption (highest ASR, low distortion---best default)
   - `L_negLM`: Suppress correct-answer probability (interpretable but high distortion)
   - `L_visionatt`: Suppress attention to visual tokens (targeted cross-modal attack)
   - `L_audioatt`: Amplify attention to perturbed audio tokens (force audio reliance)
   - `L_randatt`: Push attention toward uniform/random distribution via KL divergence
   - `L_hidden-cos`: Minimize hidden-state similarity across transformer layers

4. **Initialize the perturbation tensor and configure PGD hyperparameters.** Create `delta` as zeros matching the audio waveform shape. Set epsilon (L-infinity bound, start with 0.5), step size eta (typically epsilon/10 per iteration), and maximum iterations (train until convergence---100+ steps). For universal attacks, accumulate gradients across the training batch.

5. **Run iterative PGD optimization.** For each iteration: (a) add delta to the clean audio, (b) forward-pass through the model to compute the chosen loss, (c) backpropagate to get grad_delta, (d) update delta via signed gradient descent, (e) project delta back into the L-infinity ball. Log loss, ASR, LPIPS, and SI-SNR at checkpoints.

6. **Evaluate attack success rate on a held-out test set.** Apply the optimized perturbation to unseen audio samples. Measure ASR (fraction of originally-correct predictions now incorrect), LPIPS on log-mel spectrograms (target <= 0.08 for imperceptible attacks), and SI-SNR in the waveform domain (target >= 0 dB).

7. **Perform layer-wise analysis of where the attack takes effect.** Compute per-layer attention entropy and hidden-state cosine similarity between clean and perturbed inputs. Mid-level layers (11-18 in a 28-layer model) are most susceptible to vision attention suppression; lower layers (1-10) dominate audio-driven attacks.

8. **Test transferability to other models and encoders.** Apply the perturbation trained on Model A to Model B without re-optimization. Expect low transfer rates (typically <5% cross-model ASR). This confirms attacks are model-specific and helps justify defense strategies.

9. **Implement cross-modal consistency defense.** Monitor the ratio of hidden-state divergence (clean vs. received audio) against expected norms. Flag inputs where audio-induced attention reallocation exceeds a threshold without corresponding visual or textual changes. Compute `consistency_score = cosine_sim(h_clean, h_perturbed)` averaged across mid-layers; reject if below a calibrated threshold.

10. **Report results with standardized metrics.** Present ASR, LPIPS, SI-SNR, per-layer attention heatmaps, and confidence calibration (adversarial responses maintain ~0.93 confidence despite being wrong). Compare across attack objectives and epsilon budgets.

## Concrete Examples

**Example 1: Encoder-space attack on VideoLLaMA2 for video-QA robustness testing**

User: "I want to test how robust VideoLLaMA2 is to adversarial audio. Can you help me set up an encoder cosine similarity attack on the AVQA benchmark?"

Approach:
1. Load VideoLLaMA2 with its BEATs audio encoder and Qwen2-7B-Instruct LM backbone
2. Prepare the AVQA dataset, extracting audio waveforms at 16kHz and recording baseline accuracy
3. Implement L_cos: `loss = cosine_similarity(encoder(audio + delta), encoder(audio))`
4. Run PGD with epsilon=0.5, step_size=0.05, 200 iterations over a 50-sample training split
5. Evaluate on the held-out test split

Output:
```
Attack: L_cos (Encoder Cosine Similarity)
Model: VideoLLaMA2 (BEATs + Qwen2-7B)
Epsilon: 0.5 (L-inf)

Clean accuracy:   95.6%
Adversarial acc:  11.0%
Attack success:   89.1%
LPIPS (log-mel):  0.07
SI-SNR:           +0.8 dB
Convergence:      142 iterations

Layer analysis: Largest embedding divergence at layers 8-14
Confidence: Adversarial outputs maintain 0.91 avg confidence
```

**Example 2: Cross-modal attention manipulation to suppress visual grounding**

User: "I need to demonstrate that an audio perturbation alone can make a trimodal model ignore the video entirely. Set up a vision attention suppression attack."

Approach:
1. Implement L_visionatt: aggregate attention weights over visual tokens across all layers and heads
2. Minimize `S_v(delta) = sum over layers,heads of attention_to_visual_tokens(audio+delta, video, question)`
3. Run PGD with epsilon=0.7, 150 iterations on Music-AVQA training split
4. Visualize per-layer attention heatmaps before and after attack

Output:
```
Attack: L_visionatt (Vision Attention Suppression)
Target: Visual token attention across 28 layers, 32 heads

Pre-attack visual attention share:  62.3% (averaged across layers)
Post-attack visual attention share: 18.7%
ASR: 56.2%
LPIPS: 0.06 | SI-SNR: +0.33 dB

Layer breakdown:
  Layers 1-10:  Visual attn reduced 61% -> 34% (audio-dominant layers)
  Layers 11-18: Visual attn reduced 68% -> 12% (most affected)
  Layers 19-28: Visual attn reduced 58% -> 41% (most resistant)
```

**Example 3: Building a cross-modal consistency defense**

User: "Based on the SoundBreak findings, help me build a defense that detects adversarial audio inputs before they reach the model's decoder."

Approach:
1. Collect clean-input hidden states at mid-level layers (11-18) as reference distributions
2. For each incoming input, compute hidden-state cosine similarity between audio-included and audio-masked forward passes
3. If the audio modality causes disproportionate hidden-state divergence (beyond the visual/text baseline), flag it
4. Calibrate the detection threshold on a validation set with known adversarial examples

Output:
```python
class CrossModalConsistencyDetector:
    def __init__(self, model, reference_layers=range(11, 19), threshold=0.85):
        self.model = model
        self.reference_layers = reference_layers
        self.threshold = threshold

    def extract_hidden_states(self, audio, video, question, layers):
        """Forward pass returning hidden states at specified layers."""
        with torch.no_grad():
            outputs = self.model(audio=audio, video=video, question=question,
                                 output_hidden_states=True)
        return [outputs.hidden_states[l] for l in layers]

    def compute_consistency_score(self, audio, video, question):
        """Compare hidden states with and without audio contribution."""
        h_full = self.extract_hidden_states(audio, video, question, self.reference_layers)
        h_muted = self.extract_hidden_states(
            torch.zeros_like(audio), video, question, self.reference_layers)

        similarities = []
        for h_f, h_m in zip(h_full, h_muted):
            sim = F.cosine_similarity(h_f.mean(dim=1), h_m.mean(dim=1))
            similarities.append(sim.item())
        return sum(similarities) / len(similarities)

    def is_adversarial(self, audio, video, question):
        score = self.compute_consistency_score(audio, video, question)
        return score < self.threshold  # Low similarity = likely adversarial
```

## Best Practices

- **Do:** Start with L_cos (encoder cosine similarity) as your default attack objective---it achieves the highest ASR with the lowest perceptual distortion across benchmarks.
- **Do:** Invest in optimization iterations rather than larger training sets. SoundBreak shows that 200 iterations on 50 samples outperforms 50 iterations on 500 samples.
- **Do:** Monitor both LPIPS on log-mel spectrograms and SI-SNR on raw waveforms---they capture complementary aspects of perceptual quality.
- **Do:** Perform layer-wise analysis to understand where your attack or defense is most effective. Mid-layers (11-18 in a 28-layer model) are the critical battleground for visual grounding.
- **Avoid:** Assuming adversarial audio transfers across models. Cross-model transfer rates are typically below 5% ASR---always evaluate on the specific target.
- **Avoid:** Using L_negLM (negative language modeling) when imperceptibility matters---it produces LPIPS ~0.22 and SI-SNR ~-11.5 dB, which is audibly distorted.
- **Avoid:** Trusting model confidence scores as a defense signal. Adversarial responses maintain ~0.93 confidence even when wrong (vs. 0.85 for correct clean predictions).

## Error Handling

- **Gradient vanishing through the audio encoder:** Some audio encoders use non-differentiable components (e.g., discrete tokenization). Use straight-through estimators or attack at the post-encoder projection layer instead.
- **Perturbation clipping artifacts:** When epsilon is large (>0.7), clipping to valid waveform range [-1, 1] after L-inf projection can create audible pops. Apply the L-inf constraint before amplitude clipping, and reduce epsilon if distortion metrics spike.
- **OOM during layer-wise attention extraction:** Extracting attention weights across all layers and heads for long sequences is memory-intensive. Use gradient checkpointing or compute attention statistics in chunks across layers.
- **Low ASR despite convergence:** If loss plateaus but ASR stays low, the attack objective may not align with the model's decision boundary. Switch from attention-based objectives to L_cos or L_negLM, which more directly target the output.
- **Defense false positives on noisy real-world audio:** The consistency detector may flag naturally noisy audio (e.g., concert recordings). Calibrate the threshold on a diverse clean set that includes noisy conditions, not just studio-quality audio.

## Limitations

- **White-box access required:** All six attack objectives require gradient backpropagation through the target model. Black-box transfer attacks are largely ineffective (cross-model ASR <5%).
- **Model-specific perturbations:** Attacks exploit encoder-specific feature geometries. A perturbation crafted for BEATs (VideoLLaMA2) does not transfer to Whisper-style encoders (Qwen-Omni), limiting real-world applicability.
- **Untargeted only:** SoundBreak studies untargeted attacks (causing any wrong answer). Targeted attacks (forcing a specific wrong answer) are not addressed and likely require different objective formulations.
- **Static perturbation model:** The universal perturbation is fixed after training. Adaptive defenses that apply input transformations (e.g., audio compression, resampling) may neutralize static perturbations.
- **Evaluation scope:** Results are demonstrated on video-QA and dialogue benchmarks. Generalization to other trimodal tasks (audio-visual generation, multimodal retrieval) is not established.

## Reference

- **Paper:** [SoundBreak: A Systematic Study of Audio-Only Adversarial Attacks on Trimodal Models](https://arxiv.org/abs/2601.16231v1) (Hussain et al., 2026)
- **Key takeaway:** Encoder-space cosine similarity attacks (L_cos) are the most efficient single-modality attack vector against trimodal models, achieving 89-96% ASR at near-imperceptible distortion levels, while cross-model transferability remains fundamentally limited due to encoder-specific feature geometries.