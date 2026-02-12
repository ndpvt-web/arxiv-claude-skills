---
name: "emotion-llamav2-mmeverse-framework-benchmark"
description: "Build multimodal emotion understanding systems using the Emotion-LLaMAv2 architecture and MMEVerse benchmark methodology. Applies multiview encoding, Conv-Attention pre-fusion, and perception-to-cognition curriculum training for emotion recognition and reasoning from video, audio, and text. Triggers: 'build emotion recognition pipeline', 'multimodal emotion analysis', 'emotion reasoning from video', 'create emotion dataset with multi-agent annotation', 'benchmark emotion understanding models', 'affective computing system design'."
---

# Multimodal Emotion Understanding with Emotion-LLaMAv2 and MMEVerse

This skill enables Claude to architect and implement multimodal emotion understanding systems based on the Emotion-LLaMAv2 framework. The core technique combines a dual-view visual encoder (global frame + temporal sequence), a Conv-Attention pre-fusion module that merges modalities before the LLM backbone, and a two-stage curriculum that moves from categorical perception to open-ended cognitive reasoning. When building emotion-aware applications, Claude applies these patterns to design data pipelines, model architectures, training loops, and evaluation harnesses grounded in the MMEVerse benchmark methodology.

## When to Use

- When the user asks to build or design a system that recognizes emotions from video, audio, or multimodal inputs.
- When creating a dataset annotation pipeline that uses multiple LLM agents (e.g., vision LLM + audio LLM + GPT-4o consolidator) to produce descriptive emotion labels.
- When implementing a multimodal fusion module that needs to capture both local temporal patterns and global cross-modal attention.
- When designing a curriculum learning strategy that first grounds a model on categorical labels, then expands to free-form reasoning.
- When benchmarking an emotion model across heterogeneous datasets (IEMOCAP, MELD, DFEW, MAFW, etc.) in a unified evaluation format.
- When the user wants to add emotion reasoning capabilities to an existing vision-language model.

## Key Technique

**Multiview Encoding Without Explicit Face Detection.** Traditional affective computing pipelines rely on an external face detector to crop and align faces before feature extraction. Emotion-LLaMAv2 eliminates this brittle dependency. A global visual encoder (EVA-ViT-G) processes a single representative middle frame at 448x448, retaining all patch tokens for fine-grained spatial detail. A temporal visual encoder samples 16 uniformly-spaced frames from the clip, processes each through EVA, then spatially pools to 2x2 embeddings per frame, yielding a compact 16x4 = 64-token temporal sequence. Audio passes through Whisper-large-v3, normalized to a fixed 64-token sequence via adaptive average pooling. This tri-stream design captures facial expressions, body language, scene context, and prosodic cues without any external detector.

**Conv-Attention Pre-Fusion.** Rather than feeding raw modality tokens directly into the LLM (which wastes backbone capacity on low-level alignment), a dedicated fusion module operates externally. Audio, global-visual, and temporal-visual features are projected to a shared dimension via MLPs, then processed in parallel: an attention branch computes cross-modal attention between a concatenated feature vector and a stacked feature tensor, capturing global inter-modal relationships; a convolutional branch applies Conv1d with residual blocks and Switch activation to capture local fine-grained temporal patterns. The two branches are summed element-wise, producing a compact fused representation that the LLM receives as a single token group alongside the per-modality tokens.

**Perception-to-Cognition Curriculum.** Stage 1 trains only on categorical emotion labels (e.g., "happy", "angry") using recognition-specific instruction templates, establishing a robust mapping from multimodal features to the discrete label space (71M trainable parameters via LoRA rank=64 on LLaMA2-7B). Stage 2 introduces the rich descriptive annotations from MMEVerse, requiring the model to both predict the label and articulate coherent reasoning about why the emotion is present, citing visual, auditory, and contextual evidence. This two-phase approach prevents the model from hallucinating reasoning before it can reliably classify.

## Step-by-Step Workflow

### 1. Define the modality streams and input specifications

Establish the three input channels with their preprocessing:
- **Global visual:** Extract the middle frame of each video clip, resize to 448x448, normalize with ImageNet stats.
- **Temporal visual:** Sample 16 uniformly-spaced frames, process identically, then spatially pool each to 2x2 patch embeddings.
- **Audio:** Extract audio at 16kHz mono, pass through a frozen Whisper-large-v3 encoder, adaptive-average-pool or zero-pad to exactly 64 tokens.

### 2. Build the multiview encoder

Instantiate two frozen EVA-ViT-G encoders (or share weights with different input routing). The global encoder outputs all patch tokens from the single frame. The temporal encoder outputs pooled tokens per frame. Add a modality-specific linear adapter (MLP) per stream projecting to 4096-dimensional LLM embedding space.

### 3. Implement the Conv-Attention pre-fusion module

```python
# Pseudocode for the fusion module
class ConvAttentionFusion(nn.Module):
    def __init__(self, dim=4096):
        self.proj_audio = MLP(dim, dim)
        self.proj_global = MLP(dim, dim)
        self.proj_temporal = MLP(dim, dim)
        # Attention branch
        self.attn_q = nn.Linear(dim * 3, dim)
        self.attn_k = nn.Linear(dim, dim)
        # Conv branch
        self.conv1d = nn.Conv1d(dim, dim, kernel_size=3, padding=1)
        self.res_blocks = nn.Sequential(*[ResConvBlock(dim) for _ in range(3)])
        self.switch_act = SwitchActivation()

    def forward(self, audio_feat, global_feat, temporal_feat):
        a = self.proj_audio(audio_feat)       # (B, dim)
        g = self.proj_global(global_feat)      # (B, dim)
        t = self.proj_temporal(temporal_feat)   # (B, dim)
        # F_d: concatenated; F_s: stacked
        F_d = torch.cat([a, g, t], dim=-1)     # (B, 3*dim)
        F_s = torch.stack([a, g, t], dim=1)    # (B, 3, dim)
        # Attention branch
        q = self.attn_q(F_d).unsqueeze(1)      # (B, 1, dim)
        k = self.attn_k(F_s)                   # (B, 3, dim)
        attn_out = torch.bmm(q, k.transpose(1,2))  # (B, 1, 3)
        attn_out = (F.softmax(attn_out, dim=-1) @ F_s).squeeze(1)
        # Conv branch
        conv_in = F_s.transpose(1, 2)          # (B, dim, 3)
        conv_out = self.switch_act(self.res_blocks(self.conv1d(conv_in)))
        conv_out = conv_out.mean(dim=-1)        # (B, dim)
        return attn_out + conv_out
```

### 4. Design the prompt template with explicit token ordering

Structure model inputs as: `[FusionFeature] [ImageFeature] [VideoFeature] [AudioFeature] [Text] [TaskIdentifier]`. The task identifier distinguishes recognition-only vs. reasoning tasks, enabling curriculum control.

### 5. Prepare the dataset in MMEVerse unified instruction format

For each clip, produce a JSON record:
```json
{
  "video_path": "MELD/train/dia0_utt0.mp4",
  "audio_path": "MELD/train/dia0_utt0.wav",
  "instruction": "Analyze the emotions expressed in this video clip.",
  "label": "sadness",
  "reasoning": "The speaker's voice trembles with a falling pitch...",
  "source_dataset": "MELD",
  "task_type": "recognition"
}
```

### 6. Implement the multi-agent annotation pipeline for reasoning labels

When generating descriptive annotations for a new or existing emotion dataset:
1. Run OpenFace on all frames to extract Facial Action Unit intensities; select the peak-AU frame as anchor.
2. Feed the anchor frame to a vision LLM (e.g., Qwen2.5-VL-72B) to extract scene layout, objects, body posture, and interaction descriptions.
3. Feed the audio to an audio LLM (e.g., Qwen2-Audio) to extract prosodic cues: pitch variation, speaking rate, pauses, intensity.
4. Send all three evidence streams plus the ground-truth label to GPT-4o with a consolidation prompt that synthesizes a coherent multimodal emotion description, resolving any cross-modal conflicts.

### 7. Train Stage 1: Perception (categorical alignment)

- Freeze visual and audio encoders.
- Train fusion module + modal adapters from scratch; apply LoRA (rank=64, alpha=16) to LLaMA2's W_q and W_v matrices.
- Use only categorical labels; randomly sample recognition instruction templates.
- AdamW, weight_decay=0.05, cosine LR with peak 1e-4, warmup 1000 steps, ~50k steps on 4xA100.

### 8. Train Stage 2: Cognition (joint recognition + reasoning)

- Continue from Stage 1 checkpoint.
- Introduce descriptive reasoning annotations from Step 6.
- Instructions now require both label prediction and articulated reasoning.
- Train for an additional ~50k steps with the same optimizer settings.

### 9. Evaluate on MMEVerse-Bench

Use task-specific metrics:
- **Hit rate** for MER2023, MELD-emotion, IEMOCAP.
- **Weighted Average F-score** for sentiment tasks (MOSI, MOSEI, CH-SIMS).
- **mAP** for multi-label tasks (MAFW-multi, BOLD).
- **GPT-4o scoring** (0-10) for open reasoning: clue overlap, label overlap, process completeness.

### 10. Ablate and iterate

Run ablations on: (a) fusion module (remove conv branch, remove attention branch, replace with simple concatenation), (b) curriculum stages (skip Stage 1, joint training from scratch), (c) number of temporal frames (8, 16, 32), (d) annotation quality (original labels vs. multi-agent re-annotations).

## Concrete Examples

**Example 1: Building an emotion recognition API from video clips**

User: "I want to build a REST API that takes a short video clip and returns the detected emotion with an explanation of why."

Approach:
1. Set up a FastAPI server with an `/analyze` endpoint accepting video uploads.
2. Implement the tri-stream preprocessor: extract middle frame (global), sample 16 frames (temporal), extract audio at 16kHz.
3. Load frozen EVA-ViT-G for visual encoding and frozen Whisper-large-v3 for audio encoding.
4. Instantiate the ConvAttentionFusion module and a LoRA-adapted LLaMA2-7B backbone.
5. Load the Stage 2 checkpoint weights (perception + cognition trained).
6. On each request, run the three encoders, fuse features, construct the prompt template with a reasoning task identifier, and generate.

Output:
```json
{
  "emotion": "frustration",
  "confidence": 0.87,
  "reasoning": "The speaker's brow is furrowed with AU4 (brow lowerer) activated. Their vocal pitch rises sharply mid-sentence before trailing off, indicating suppressed anger. The crossed arms and turned-away posture reinforce defensive frustration. The office setting with scattered papers suggests work-related stress."
}
```

**Example 2: Creating a multi-agent annotation pipeline for a custom emotion dataset**

User: "I have 5,000 therapy session clips with basic emotion labels. I want richer descriptive annotations for training."

Approach:
1. Extract all frames and run OpenFace to compute AU intensities per frame. For each clip, select the frame with maximum summed AU activation.
2. Batch the peak frames through Qwen2.5-VL-72B with the prompt: "Describe the person's facial expression, body language, scene context, and any visible emotional cues in this frame."
3. Batch the audio tracks through Qwen2-Audio with: "Analyze the speaker's vocal characteristics: pitch, rate, volume, pauses, breathiness, and emotional tone."
4. For each clip, send the visual description, audio description, original label, and transcript to GPT-4o with a consolidation prompt:

```
Given these multimodal observations about a video clip labeled "{label}":
- Visual: {visual_description}
- Audio: {audio_description}
- Transcript: "{transcript}"

Write a coherent 3-4 sentence description explaining why this clip
expresses {label}, citing specific visual, auditory, and verbal evidence.
Resolve any contradictions between modalities.
```

5. Store results in the unified MMEVerse JSON format. Run inter-annotator agreement (Cohen's kappa) on a 200-clip subset with human reviewers.

Output: 5,000 clips with structured annotations like:
```
"The therapist displays genuine concern through raised inner brows (AU1)
and a slight forward lean. Her voice softens noticeably with slower
pacing and deliberate pauses between phrases, conveying empathy. The
words 'I understand how difficult this must be' align with the compassionate
vocal tone, though a brief lip press suggests she is holding back her
own emotional response."
```

**Example 3: Benchmarking a new emotion model against MMEVerse-Bench**

User: "I've fine-tuned a VideoLLaMA variant for emotion tasks. How do I evaluate it properly?"

Approach:
1. Download the 12 MMEVerse source datasets and organize into the unified split structure (130k train / 36k test).
2. Format all test clips into the standardized instruction template with task identifiers matching the 18 benchmarks.
3. Run inference on all 36k test clips, collecting predicted labels and reasoning text.
4. Compute per-benchmark metrics:
   - Hit rate for MER2023, MELD-e, IEMOCAP, DFEW, CAER, E3.
   - Weighted Avg F-score for MOSI, MOSEI, CH-SIMS, CH-SIMSv2.
   - mAP for MAFW-multi, BOLD.
   - Accuracy for MAFW-s, MELD-s, MC-EIU-intent, MC-EIU-emotion.
5. For reasoning quality, sample 500 clips and score with GPT-4o on clue overlap (0-10), label overlap (0-10), process completeness (0-10).
6. Report Avg-9 (MER-UniBench overlap) and Avg-18 (full MMEVerse-Bench). Compare against baselines: AffectGPT (54.4%), Emotion-LLaMA v1 (64.7%), Emotion-LLaMAv2 (66.6%).

## Best Practices

- **Do:** Keep visual and audio encoders frozen during both training stages. Only 0.92% of parameters need training (fusion module + adapters + LoRA), making this feasible on 4xA100 in ~3 hours.
- **Do:** Always include the fusion feature tokens alongside per-modality tokens in the LLM prompt. The fusion captures cross-modal interactions the LLM backbone cannot efficiently learn on its own.
- **Do:** Use the perception-first curriculum. Training recognition and reasoning jointly from scratch degrades classification accuracy because the model tries to generate plausible-sounding reasoning before it can reliably detect the emotion.
- **Do:** Resolve cross-modal conflicts during annotation. When the face shows a smile but the voice trembles, the consolidation LLM must acknowledge and reason about the discrepancy (e.g., masking behavior) rather than ignoring one modality.
- **Avoid:** Using explicit face detection as a preprocessing step. It creates a single point of failure for occluded, side-profile, or multi-person scenes. The multiview encoder with full-frame input allows the LLM to implicitly attend to emotion-relevant regions.
- **Avoid:** Skipping spatial pooling on temporal frames. Without pooling from full patch tokens to 2x2 per frame, the temporal stream produces an unmanageably long sequence (16 frames x 257 patches = 4112 tokens) that overwhelms the context window.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| Audio tokens all zeros | Clip has no audio track or is silent | Zero-pad to 64 tokens as designed; the fusion module learns to down-weight silent audio via the attention branch |
| Peak AU frame selection fails | OpenFace cannot detect a face in any frame | Fall back to the middle frame (same as global encoder input); the system works without AU-based selection |
| Multi-agent annotations contradict ground truth | GPT-4o consolidation overwrites the original label | Pin the ground-truth label in the consolidation prompt; instruct the LLM to explain the given label, not reassign it |
| Stage 2 reasoning degrades recognition accuracy | Too many reasoning steps dilute classification signal | Increase the ratio of recognition-only instructions in Stage 2 (e.g., 40% recognition / 60% reasoning) |
| OOM during temporal encoding | Too many frames sampled or resolution too high | Reduce to 8 temporal frames or lower resolution to 224x224 for temporal stream only |

## Limitations

- The architecture is built on LLaMA2-7B. Scaling to larger backbones (13B, 70B) is untested and may require re-tuning the LoRA rank and fusion module dimensions.
- MMEVerse covers 12 datasets but is English-dominated. Mandarin datasets (CH-SIMS) are included but other languages are not represented.
- The multi-agent annotation pipeline depends on commercial APIs (GPT-4o), making large-scale re-annotation costly. Budget roughly $0.01-0.03 per clip for the consolidation step.
- Real-time inference is constrained by the three-encoder forward pass plus LLM generation. Expect ~2-4 seconds per clip on a single A100, not suitable for sub-100ms latency requirements.
- The Conv-Attention fusion module is designed for three modalities. Adding a fourth (e.g., physiological signals) requires architectural changes to the stacking and attention dimensions.

## Reference

**Paper:** [Emotion-LLaMAv2 and MMEVerse: A New Framework and Benchmark for Multimodal Emotion Understanding](https://arxiv.org/abs/2601.16449v1) (Peng et al., 2026). Focus on Section 3 for the multiview encoder and Conv-Attention fusion design, Section 4 for the MMEVerse annotation pipeline, and Section 5 for the curriculum training procedure and ablation results.

**Code:** [https://github.com/ooochen-30/Emotion-LLaMA-v2](https://github.com/ooochen-30/Emotion-LLaMA-v2)