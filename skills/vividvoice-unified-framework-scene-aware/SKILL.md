---
name: "vividvoice-unified-framework-scene-aware"
description: "Build scene-aware speech synthesis systems that generate speech conditioned on visual scenes, aligning timbre and environmental acoustics with image content using decoupled memory banks and cross-modal supervision. Use when: 'generate speech from a scene image', 'build visually-driven TTS', 'align audio with visual environments', 'scene-conditioned voice synthesis', 'multimodal speech generation from images', 'add environmental acoustics matching a photo'."
---

# VividVoice: Scene-Aware Visually-Driven Speech Synthesis

This skill enables Claude to help build speech synthesis systems that condition generated audio on visual scene content. The core technique from VividVoice (ICASSP 2026) uses **decoupled memory banks** to separately capture character-timbre and environment-sound mappings, then a **cross-modal hybrid supervision strategy** to align visual inputs (images) with acoustic outputs (speech with environmental acoustics). This lets a system look at an image of a rainy street and produce speech with appropriate reverb and rain ambiance, or see a forest scene and generate voice with outdoor acoustic characteristics — without requiring paired visual-audio training data for every scenario.

## When to Use

- When the user wants to build a TTS system that adapts voice timbre and room acoustics based on an input image or scene description
- When designing a multimodal pipeline that takes image + text and outputs speech with scene-appropriate environmental audio
- When constructing a synthetic multimodal dataset pairing images, speakers, and environmental sounds programmatically
- When implementing decoupled memory bank modules for cross-modal alignment between vision and audio
- When the user needs to disentangle speaker identity from environmental acoustics in a generation pipeline
- When building immersive audio for games, VR, or film where speech must match the visual environment

## Key Technique

**The Core Problem**: Existing TTS models generate speech in a vacuum — they ignore the physical environment visible in a scene. VividVoice solves this by decoupling the visual-to-audio mapping into two independent channels: (1) character appearance to voice timbre, and (2) scene environment to acoustic properties (reverb, ambient sounds, SNR).

**D-MSVA (Decoupled Multimodal Scene-Voice Alignment)**: The key module uses four learnable memory banks (each N=128 slots, dimension D): Character-Key (`M_pk`), Environment-Key (`M_ek`), Timbre-Value (`M_tv`), and Sound-Value (`M_sv`). An auditory pathway decomposes reference audio into timbre and sound components via cosine-similarity attention over value banks. A visual pathway queries the key banks using MetaCLIP image features to retrieve the corresponding auditory primitives. KL-divergence alignment loss forces the visual pathway's attention distributions to match the auditory pathway's, so at inference time the visual pathway alone can drive synthesis.

**Programmatic Dataset Construction (Vivid-210K)**: Rather than collecting expensive paired visual-audio data, VividVoice uses a generation pipeline: the same text prompt drives both FLUX.1 (image generation) and Stable Audio Open (environmental sound generation) to ensure semantic alignment. FLUX.1-Kontext-dev composites speaker face images into generated scenes. Speech is mixed with environmental audio at random 4-20 dB SNR. Quality filtering uses a VLM panel (Qwen2.5VL + GPT-4o + DeepSeek-V3) scoring semantic consistency, achieving 98.6% scene-matching and 95.4% audiovisual consistency rates.

## Step-by-Step Workflow

### 1. Define the Multimodal Input Schema

Establish the three input modalities your system will accept: (a) a visual scene image, (b) text content to be spoken, and (c) optionally a speaker reference audio clip. Design your data structures accordingly — the image feeds the scene perception pathway, the text feeds the content pathway.

### 2. Set Up Visual and Audio Encoders

Use MetaCLIP (or a comparable CLIP variant) as the visual encoder to extract scene-level features from images. Use CLAP as the audio encoder to extract environmental acoustic features from reference audio. Both encoders should be frozen during initial training — they provide the feature space that the memory banks learn to bridge.

### 3. Implement the Four Memory Banks

Create four learnable embedding matrices, each of shape `(N, D)` where N=128 and D matches your encoder output dimension:
- `M_pk` (Character-Key): stores character visual prototypes
- `M_ek` (Environment-Key): stores environment visual prototypes
- `M_tv` (Timbre-Value): stores voice timbre primitives
- `M_sv` (Sound-Value): stores environmental sound primitives

Initialize with Xavier uniform or similar. These banks are the **only** learnable parameters in the alignment module.

### 4. Build the Dual-Pathway Attention Mechanism

**Auditory pathway** (used during training): Compute cosine similarity between CLAP audio features `a` and each slot in `M_tv` and `M_sv`, apply softmax to get attention weights `w'_t` and `w'_s`, reconstruct timbre/sound components as weighted sums of value bank slots.

**Visual pathway** (used during training and inference): Compute cosine similarity between MetaCLIP image features and `M_pk`/`M_ek` key banks to get weights `w_p` and `w_e`, then cross-retrieve from value banks: `â_v = M_tv^T * w_p + M_sv^T * w_e`.

### 5. Define the Loss Functions

Implement four loss terms with the following weights:
- **Reconstruction loss** `L_rec`: MSE between auditory pathway output and ground-truth audio features
- **Alignment loss** `L_align` (lambda=10): KL divergence between visual and auditory attention distributions — `D_KL(w'_t || w_p) + D_KL(w'_s || w_e)`
- **Imitation loss** `L_imi` (lambda=2): MSE between visual pathway output and auditory pathway output
- **Contrastive disentanglement** `L_timbre_c` + `L_env_c` (lambda=0.5 each): enforce that same-character/different-environment pairs produce similar timbre, and different-character/same-environment pairs produce similar environmental sound

### 6. Construct the Paired Dataset Programmatically

If you lack paired visual-audio data, replicate the Vivid-210K pipeline:
1. Write scene description prompts (e.g., "a rainy city street at night")
2. Generate images from prompts using a text-to-image model (e.g., FLUX.1, SDXL)
3. Generate environmental audio from the same prompts using a text-to-audio model (e.g., Stable Audio Open)
4. Composite speaker face images into generated scenes using an instruction-guided editor
5. Mix clean speech with environmental audio at random SNR between 4-20 dB
6. Filter with a VLM panel: have 2-3 models score image-audio semantic alignment, discard pairs below threshold

### 7. Integrate with a Latent Diffusion Backbone

Use a U-Net-based latent diffusion model (AudioLDM architecture) as the synthesis backbone. The text content pathway produces frame-aligned text latents via Monotonic Alignment Search (MAS). The D-MSVA module produces scene embeddings. Both condition the U-Net denoiser. Decode latent outputs through a VAE + vocoder (HiFi-GAN) to produce waveforms.

### 8. Train in Two Phases

**Phase 1 — Main training**: 800K steps, AdamW optimizer, lr=1e-5, batch size 8 per GPU across 8 GPUs. Train the memory banks and conditioning layers while keeping encoders frozen.

**Phase 2 — Fine-tuning**: 100K steps on real-world data (decomposed from actual videos), lr=5e-5, batch size 16, with EMA and AMP enabled for stability.

### 9. Evaluate with Multimodal Metrics

Measure at minimum: WER (via Whisper-Large-v3) for content clarity, FAD for audio quality, KL divergence for distribution match, and a CLAP-based score for visual-acoustic consistency. For subjective evaluation, collect MOS scores on four axes: naturalness, scene consistency, timbre consistency, and content fidelity.

## Concrete Examples

**Example 1: Building a Scene-Conditioned TTS Module**

User: "I want to build a TTS system that takes a scene image and generates speech that sounds like it was recorded in that environment."

Approach:
1. Set up MetaCLIP to encode input scene images into 768-dim feature vectors
2. Implement the four memory banks (128 slots x 768 dims each) with the dual-pathway attention
3. Use a pretrained AudioLDM as the diffusion backbone, injecting scene embeddings via cross-attention layers in the U-Net
4. For training data, generate 50K image-audio pairs using the programmatic pipeline: same prompt drives FLUX.1 (image) and Stable Audio Open (ambient audio), mix with LibriTTS speech at random SNR
5. Train the memory banks and cross-attention layers for 200K steps with the four-term loss

Output:
```python
class DMSVA(nn.Module):
    def __init__(self, dim=768, num_slots=128):
        super().__init__()
        self.M_pk = nn.Parameter(torch.randn(num_slots, dim))  # Character-Key
        self.M_ek = nn.Parameter(torch.randn(num_slots, dim))  # Environment-Key
        self.M_tv = nn.Parameter(torch.randn(num_slots, dim))  # Timbre-Value
        self.M_sv = nn.Parameter(torch.randn(num_slots, dim))  # Sound-Value
        nn.init.xavier_uniform_(self.M_pk)
        nn.init.xavier_uniform_(self.M_ek)
        nn.init.xavier_uniform_(self.M_tv)
        nn.init.xavier_uniform_(self.M_sv)

    def auditory_pathway(self, audio_feat):
        """Decompose audio into timbre + sound via value banks."""
        w_t = F.softmax(F.cosine_similarity(
            audio_feat.unsqueeze(1), self.M_tv.unsqueeze(0), dim=-1
        ), dim=-1)
        w_s = F.softmax(F.cosine_similarity(
            audio_feat.unsqueeze(1), self.M_sv.unsqueeze(0), dim=-1
        ), dim=-1)
        timbre = w_t @ self.M_tv
        sound = w_s @ self.M_sv
        return timbre, sound, w_t, w_s

    def visual_pathway(self, image_feat):
        """Map visual features to auditory primitives via key banks."""
        w_p = F.softmax(F.cosine_similarity(
            image_feat.unsqueeze(1), self.M_pk.unsqueeze(0), dim=-1
        ), dim=-1)
        w_e = F.softmax(F.cosine_similarity(
            image_feat.unsqueeze(1), self.M_ek.unsqueeze(0), dim=-1
        ), dim=-1)
        timbre_hat = w_p @ self.M_tv  # Cross-retrieve from value banks
        sound_hat = w_e @ self.M_sv
        return timbre_hat, sound_hat, w_p, w_e
```

**Example 2: Creating a Synthetic Multimodal Dataset**

User: "I need paired image-audio data for training a scene-aware audio model but I don't have any. How do I build one?"

Approach:
1. Write 1000+ diverse scene description prompts covering indoor/outdoor environments
2. For each prompt, generate an image (FLUX.1 / SDXL) and environmental audio (Stable Audio Open) from the identical text
3. Source clean speech from LibriTTS or LRS3 and speaker face images from the same datasets
4. Composite faces into scene images using an inpainting/editing model
5. Mix speech with environmental audio at SNR uniformly sampled from [4, 20] dB
6. Run quality filtering: feed each image to a VLM, ask it to describe the scene, then score overlap with the original prompt

Output:
```python
import random

def build_paired_sample(prompt, speech_path, face_image_path):
    """Generate one paired (image, audio) sample from a scene prompt."""
    # Step 1: Generate scene image and environmental audio from same prompt
    scene_image = flux_generate(prompt)  # Returns PIL Image
    env_audio = stable_audio_generate(prompt, duration=10.0)  # Returns waveform

    # Step 2: Composite speaker face into scene
    composite_image = flux_kontext_edit(
        scene_image, face_image_path,
        instruction="Place this person naturally in the scene"
    )

    # Step 3: Mix speech with environmental audio
    snr_db = random.uniform(4.0, 20.0)
    speech = load_audio(speech_path, sr=16000)
    mixed = mix_at_snr(speech, env_audio, snr_db)

    # Step 4: Quality filter via VLM panel
    description = qwen_vl_describe(composite_image)
    score = semantic_similarity(description, prompt)
    if score < 0.85:
        return None  # Reject low-quality pairs

    return {
        "image": composite_image,
        "audio": mixed,
        "speech": speech,
        "env_audio": env_audio,
        "prompt": prompt,
        "snr_db": snr_db,
        "quality_score": score,
    }
```

**Example 3: Adding Scene-Aware Conditioning to an Existing TTS Model**

User: "I have a working AudioLDM-based TTS. How do I add visual scene conditioning without retraining from scratch?"

Approach:
1. Freeze the existing AudioLDM U-Net and text encoder weights
2. Add the D-MSVA module (4 memory banks) as a new conditioning pathway
3. Insert cross-attention layers in the U-Net that attend to scene embeddings alongside existing text embeddings
4. Train only the new parameters (memory banks + cross-attention) on your paired dataset
5. Use the alignment + imitation + contrastive losses to train the visual pathway to match auditory ground truth

Output architecture modification:
```python
# In U-Net block, add scene conditioning alongside text conditioning
class SceneConditionedBlock(nn.Module):
    def __init__(self, existing_block, dim):
        super().__init__()
        self.existing_block = existing_block  # Frozen
        self.scene_cross_attn = CrossAttention(dim, dim)  # New, trainable
        self.gate = nn.Parameter(torch.zeros(1))  # Learnable gate, init 0

    def forward(self, x, text_cond, scene_emb):
        # Original text-conditioned output (frozen path)
        h = self.existing_block(x, text_cond)
        # Scene conditioning (trainable path, gated to start from zero)
        scene_out = self.scene_cross_attn(h, scene_emb)
        return h + torch.sigmoid(self.gate) * scene_out
```

## Best Practices

- **Do**: Decouple character identity from environmental acoustics in your memory banks — this is the key insight. Concatenating all visual features into one vector loses the ability to independently control timbre vs. reverb/ambiance.
- **Do**: Use the same text prompt for both image and audio generation when building synthetic datasets. This semantic anchoring is what makes the programmatic pipeline work without manual annotation.
- **Do**: Set the alignment loss weight significantly higher than other losses (lambda=10 vs. 0.5-2 for others). The KL divergence between visual and auditory attention distributions is the critical training signal.
- **Do**: Use 128 memory bank slots as the default. The paper's ablation shows this is optimal — fewer slots underfit, more slots overfit and degrade FAD.
- **Avoid**: Training memory banks and the diffusion backbone simultaneously from scratch. Freeze the backbone first, train alignment, then fine-tune jointly if needed.
- **Avoid**: Using SNR values below 4 dB when mixing speech with environmental audio for training data. The speech becomes unintelligible and hurts content clarity (WER degrades).

## Error Handling

- **Memory bank collapse** (all attention concentrates on few slots): Add entropy regularization to the attention weights, or re-initialize underused slots periodically during training.
- **Visual-auditory misalignment diverges**: If `L_align` increases during training, reduce its weight temporarily and ensure both encoders (MetaCLIP, CLAP) are frozen — gradient flow through encoders can destabilize the memory banks.
- **Generated audio has correct environment but wrong timbre** (or vice versa): This indicates incomplete decoupling. Increase the contrastive disentanglement loss weights (`lambda_3`, `lambda_4`) and verify your training data includes sufficient same-character/different-environment and different-character/same-environment pairs.
- **Low quality scores from VLM panel filtering**: If more than 30% of synthetic pairs are rejected, your scene prompts may be too abstract. Use concrete, physically grounded descriptions ("a sunlit kitchen with tile floors" rather than "a warm domestic setting").
- **High WER in generated speech**: The content pathway (text encoder + MAS) may be undertrained. Pre-train it on clean speech data before adding scene conditioning.

## Limitations

- The technique requires scene images at inference time — it cannot infer environmental acoustics from text descriptions alone (for that, use VoiceLDM or AudioBox directly).
- The programmatic dataset pipeline produces synthetic pairings, not real recorded data. Fine-tuning on real video-extracted data (as VividVoice does in Phase 2) is important for bridging this domain gap.
- Memory bank size (N=128) is fixed per training run. Environments not well-represented in training data will produce generic acoustic properties rather than specific ones.
- The approach focuses on environmental acoustics (reverb, ambiance, SNR) and speaker timbre. It does not model prosody, emotion, or speaking rate from visual cues — those require separate conditioning mechanisms.
- Evaluation of scene consistency remains partially subjective. Automated metrics (CLAPcap) correlate with human judgment but are not a complete substitute.

## Reference

**Paper**: [VividVoice: A Unified Framework for Scene-Aware Visually-Driven Speech Synthesis](https://arxiv.org/abs/2602.02591v1) (ICASSP 2026)
**Demo**: https://chengyuann.github.io/VividVoice/
**Key takeaway**: The decoupled memory bank architecture (D-MSVA) with cross-modal KL-divergence alignment is what enables a visual-only inference pathway to produce audio that matches both speaker identity and environmental acoustics — look at Section 3.2 and Table 2 (ablation) for the design rationale and empirical justification.