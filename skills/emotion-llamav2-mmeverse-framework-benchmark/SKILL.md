---
name: "emotion-llamav2-mmeverse-framework-benchmark"
description: "Build multimodal emotion understanding systems using Emotion-LLaMAv2's architecture patterns: end-to-end multiview encoding, conv-attention pre-fusion, perception-to-cognition curriculum training, and multi-agent annotation pipelines. Use when asked to: 'build an emotion recognition system from video', 'design a multimodal fusion pipeline for affective computing', 'create a multi-agent data annotation workflow', 'implement curriculum instruction tuning for LLMs', 'set up emotion benchmarks across multiple datasets', 'fuse audio, video, and text for sentiment analysis'."
---

# Multimodal Emotion Understanding with Emotion-LLaMAv2 and MMEVerse

This skill enables Claude to design and implement multimodal emotion understanding systems following the architecture of Emotion-LLaMAv2 (arXiv:2601.16449). The core technique replaces brittle explicit face-detection pipelines with end-to-end multiview encoders, fuses audio/visual/text features through a parallel conv-attention module before the LLM backbone, and trains in a two-stage perception-to-cognition curriculum that first learns categorical emotion labels then graduates to free-form emotional reasoning. The skill also covers MMEVerse's multi-agent annotation pipeline (Qwen2-Audio + Qwen2.5-VL + GPT-4o) for producing high-quality emotion descriptions at scale.

## When to Use

- When the user asks to build a video-based emotion recognition or sentiment analysis system
- When designing a multimodal fusion architecture that combines audio, visual frames, and text/transcripts
- When creating a multi-agent pipeline to annotate emotional content in video datasets
- When implementing curriculum-style instruction tuning (simple classification first, then reasoning)
- When unifying multiple emotion datasets (IEMOCAP, MELD, DFEW, etc.) into a single training format
- When the user needs to fuse local features (convolution) with global context (attention) before an LLM
- When building benchmarks that test both emotion classification accuracy and reasoning quality

## Key Technique

**End-to-End Multiview Encoding.** Traditional affective computing pipelines rely on explicit face detectors (e.g., OpenFace) to crop facial regions before analysis. This is fragile -- detectors fail on occlusions, extreme poses, and low resolution. Emotion-LLaMAv2 eliminates this by processing full frames through EVA-ViT-G at 448x448 resolution for spatial tokens, uniformly sampling 16 frames and spatially pooling them into 2x2 embeddings per frame for temporal tokens, and encoding audio via Whisper-large-v3 normalized to 64 tokens. The model learns to implicitly attend to emotion-relevant regions (faces, hands, body posture) without hard-coded detection.

**Conv-Attention Pre-Fusion.** Rather than concatenating modality tokens and leaving all cross-modal reasoning to the LLM, a dedicated pre-fusion module operates externally. Two parallel branches process the features: (1) a convolutional branch with residual Conv1d blocks and Switch activation captures local temporal patterns (prosody shifts, micro-expressions), and (2) an attention branch computes cross-modal relevance weights across audio, global-visual, and temporal-visual features. Their outputs are summed element-wise. This design gives the LLM pre-aligned multimodal tokens that already encode cross-modal emotion cues, reducing the burden on the language model.

**Perception-to-Cognition Curriculum.** Training proceeds in two stages. Stage 1 (perception) trains on categorical emotion labels only -- the model learns to map multimodal inputs to discrete emotion categories (happy, sad, angry, etc.) using recognition-specific task identifiers in the prompt. Stage 2 (cognition) adds free-form reasoning supervision -- the model must now both predict the category and articulate *why*, grounding explanations in specific multimodal evidence (facial expressions, vocal tone, body language, spoken words). This curriculum prevents the model from generating plausible-sounding but ungrounded reasoning.

## Step-by-Step Workflow

1. **Define the multimodal input schema.** Specify modalities (video frames, audio waveform, transcript text). For video: extract a representative middle frame at 448x448 for spatial encoding, and uniformly sample 16 frames for temporal encoding. For audio: resample to 16kHz mono. For text: use ASR transcript or subtitle.

2. **Set up the multiview encoder pipeline.** Use a vision transformer (e.g., EVA-ViT-G or CLIP-ViT-L) to encode the spatial frame into patch tokens (excluding CLS). Process the 16 temporal frames through the same or a video-specific encoder (VideoMAE/VideoMAEv2) and spatially pool each frame to 2x2 embeddings (4 tokens x 16 frames = 64 temporal tokens). Encode audio through Whisper-large-v3, applying adaptive average pooling to normalize to a fixed 64-token sequence.

3. **Implement the Conv-Attention pre-fusion module.** Project all three modality embeddings to a shared channel dimension via MLPs. Create two parallel branches:
   - *Conv branch:* Initial Conv1d followed by N residual blocks, each applying `F_out = F_in + Switch(Conv1d(F_in))`. Use the Switch activation: `Switch(x) = x * sigmoid(x)`.
   - *Attention branch:* Concatenate the three modality features along the sequence dimension into `F_d`, compute attention weights via an MLP, unsqueeze, and multiply element-wise with stacked modality features `F_s`.
   - Sum both branch outputs to produce fused feature tokens `u_f`.

4. **Construct the multimodal prompt template.** Format input as:
   ```
   [Vid] <FusionFeature> <ImageFeature> <VideoFeature> <AudioFeature> [Vid]
   The person in the video says: "<transcript>"
   <TaskIdentifier> <Instruction>
   ```
   Use separate task identifiers for recognition (`[REC]`) vs. reasoning (`[REASON]`) tasks so the LLM conditions its output format on the task type.

5. **Implement Stage 1: Perception training.** Fine-tune the full pipeline on categorical emotion labels only. Use the MMEVerse-style unified format where each sample maps to one of the target emotion categories. Randomly sample from a pool of recognition instruction prompts (e.g., "What emotion is the person expressing?", "Classify the emotional state shown in this video.") for diversity.

6. **Implement Stage 2: Cognition training.** Starting from Stage 1 weights, add reasoning supervision. Each training sample now includes both the emotion label and a multi-sentence description linking multimodal evidence to the affective interpretation. Train on instructions that require both prediction and explanation (e.g., "Identify the emotion and explain what visual, vocal, and textual cues support your answer.").

7. **Build the multi-agent annotation pipeline (for dataset creation).** To generate high-quality emotion reasoning labels at scale:
   - Run OpenFace on video to extract Facial Action Unit intensities; identify the peak-emotion frame.
   - Send the peak frame to a vision-language model (Qwen2.5-VL-72B or equivalent) to extract scene context, objects, interactions, and body language descriptions.
   - Send the audio to an audio-language model (Qwen2-Audio or equivalent) to extract prosodic cues: pitch variation, speaking rate, intensity, pauses, hesitations.
   - Feed all three descriptions plus the transcript into GPT-4o (or equivalent) to synthesize a coherent multimodal emotion description.

8. **Unify multiple datasets into a single instruction format.** Map each source dataset's label taxonomy to a shared emotion vocabulary. Standardize video clip boundaries, audio extraction, and transcript alignment. Store each sample as: `{video_path, audio_path, transcript, emotion_label, reasoning_description, source_dataset, split}`.

9. **Evaluate on both recognition and reasoning metrics.** For recognition: use weighted accuracy (WAcc) across emotion categories. For reasoning: use clue-overlap and label-overlap scores that measure whether generated explanations cite correct multimodal evidence and arrive at the right emotion label.

10. **Run ablation checks.** Validate that: (a) removing the Conv-Attention pre-fusion degrades performance, (b) skipping Stage 1 and training directly on reasoning hurts both recognition and reasoning, (c) increasing dataset scale and annotation quality (MMEVerse vs. smaller sets) yields measurable gains.

## Concrete Examples

**Example 1: Building a Video Emotion Classifier**

User: "I have a dataset of customer service video calls. I want to build a system that detects the customer's emotional state from the video, audio, and transcript."

Approach:
1. Extract representative frames and 16-frame temporal sequences from each video clip.
2. Transcribe audio using Whisper; also encode raw audio into 64-token feature sequences.
3. Encode spatial frame via CLIP-ViT-L, temporal frames via VideoMAE, audio via Whisper encoder.
4. Implement the Conv-Attention pre-fusion: project all features to 768-dim, run conv branch (3 residual blocks) and attention branch in parallel, sum outputs.
5. Feed fused tokens into an LLM backbone (LLaMA-2-7B) with the prompt template: `[Vid] <fused> <spatial> <temporal> <audio> [Vid] The customer says: "I've been waiting for an hour..." [REC] What is the customer's emotional state?`
6. Train Stage 1 on categorical labels (frustrated, satisfied, neutral, angry, confused) for 3 epochs.
7. Train Stage 2 on labels + reasoning descriptions for 2 more epochs.

Output:
```json
{
  "emotion": "frustrated",
  "confidence": 0.87,
  "reasoning": "The customer's voice shows rising pitch and increased speaking rate, indicating agitation. Their facial expression displays furrowed brows and a tightened jaw. The verbal content ('I've been waiting for an hour') explicitly references a prolonged negative experience. These visual, vocal, and lexical cues collectively indicate frustration."
}
```

**Example 2: Multi-Agent Emotion Annotation Pipeline**

User: "I have 50,000 unlabeled video clips and need rich emotion annotations with reasoning. How do I build an automated annotation pipeline?"

Approach:
1. Run OpenFace on each clip to compute per-frame AU intensity; select the frame with maximum aggregate AU activation as the peak-emotion frame.
2. Send the peak frame to Qwen2.5-VL-72B with the prompt: "Describe the person's facial expression, body posture, gestures, and the surrounding scene context relevant to their emotional state."
3. Extract audio and send to Qwen2-Audio with the prompt: "Analyze the speaker's vocal characteristics: pitch, tempo, volume, pauses, breathiness, and any paralinguistic cues indicating emotion."
4. Feed all three outputs (visual description, audio description, transcript) into GPT-4o with: "Given the following multimodal evidence, synthesize a coherent 2-3 sentence description of the person's emotional state, citing specific cues from each modality. Then provide a single emotion category label."
5. Store the output in the unified format.

Output per clip:
```json
{
  "video_id": "clip_04291",
  "emotion_label": "sadness",
  "visual_description": "The subject's eyes are downcast with slightly raised inner brows (AU1+AU4). Shoulders are slumped forward. The room is dimly lit with no other people present.",
  "audio_description": "Speech is slow (approx 2.1 syllables/sec) with frequent pauses of 1-2 seconds. Pitch is low and monotone with occasional vocal tremor.",
  "transcript": "I just... I don't know what to do anymore.",
  "synthesized_reasoning": "The combination of downcast gaze with inner brow raise signals distress, while the slumped posture suggests low energy or defeat. The slow, monotone speech with frequent pauses reflects emotional weight, and the verbal content expresses helplessness. These converging cues across modalities indicate sadness."
}
```

**Example 3: Curriculum Instruction Tuning Setup**

User: "I already have an LLM fine-tuned on vision-language tasks. How do I adapt it for emotion understanding using the curriculum approach?"

Approach:
1. Prepare two instruction datasets from the same source clips:
   - **Stage 1 dataset**: Each sample has multimodal tokens + a recognition instruction + category label only.
   - **Stage 2 dataset**: Same samples but now include both the category label and a multi-sentence reasoning description.
2. Create an instruction pool for each stage:
   - Stage 1 pool: `["What emotion is shown?", "Classify the emotional state.", "Identify the person's feelings.", ...]`
   - Stage 2 pool: `["What emotion is shown and why?", "Identify the emotion and explain the multimodal evidence.", ...]`
3. Fine-tune Stage 1: Freeze the vision/audio encoders for the first epoch, then unfreeze all. Train for 3 epochs with learning rate 2e-5.
4. Fine-tune Stage 2: Starting from Stage 1 checkpoint, train for 2 epochs at learning rate 1e-5. Supervise both the predicted label and the reasoning text.
5. Validate: Check that Stage 2 model maintains Stage 1 recognition accuracy (should not degrade) while now producing grounded reasoning.

Output training config:
```yaml
curriculum:
  stage_1:
    task: "recognition_only"
    epochs: 3
    lr: 2e-5
    data: "mmeverse_train_recognition.jsonl"
    loss: "cross_entropy_on_label_tokens"
  stage_2:
    task: "recognition_and_reasoning"
    epochs: 2
    lr: 1e-5
    data: "mmeverse_train_full.jsonl"
    loss: "autoregressive_on_full_response"
    init_from: "stage_1_checkpoint"
```

## Best Practices

- **Do** use the full-frame input instead of cropped face regions. The end-to-end approach captures body language, scene context, and gestural cues that face crops miss. The model learns to attend to faces implicitly.
- **Do** keep the pre-fusion module lightweight relative to the LLM backbone. Its purpose is cross-modal alignment, not deep reasoning -- 3-4 residual conv blocks and a single attention layer suffice.
- **Do** use diverse instruction prompts during training (a pool of 10-20 paraphrases per task type) to prevent the model from overfitting to specific phrasings.
- **Do** validate annotation quality by sampling 200-500 clips and manually checking synthesized descriptions against video content before scaling to the full dataset.
- **Avoid** skipping Stage 1 and training directly on reasoning. The perception-first curriculum is critical; without it, the model generates plausible-sounding explanations that cite the wrong cues or misidentify emotions.
- **Avoid** relying on a single annotator model. The multi-agent pipeline works because each model contributes complementary expertise (facial AUs, scene context, vocal prosody). A single VLM misses audio cues; a single audio model misses visual ones.

## Error Handling

- **Face detector failures (legacy pipelines):** If migrating from an OpenFace-dependent system, handle cases where AU extraction fails by falling back to the full-frame encoder path. Log clips where OpenFace returns no detections -- these are precisely the cases the end-to-end approach handles better.
- **Audio-visual misalignment:** Verify that audio and video timestamps are synchronized before encoding. A drift of even 500ms can cause the model to associate vocal cues with the wrong facial expressions. Use ffprobe to check stream durations and re-mux if needed.
- **Annotation model hallucination:** GPT-4o may hallucinate emotions not present in the video. Mitigate by including the ground-truth label (if available) as a constraint in the synthesis prompt, or by adding a verification step where a second model scores the coherence between the description and the video.
- **Class imbalance across datasets:** Emotion datasets are notoriously imbalanced (e.g., MELD has far more "neutral" than "disgust"). Apply weighted sampling during training or use focal loss to prevent the model from defaulting to majority classes.
- **Temporal token pooling failures:** If video frames are corrupted or mostly black, the 2x2 spatial pooling produces near-zero tokens. Add a check for low-variance temporal features and flag those clips for review.

## Limitations

- The full pipeline requires substantial compute: EVA-ViT-G + Whisper + LLaMA-2 backbone is GPU-intensive. For inference-constrained deployments, consider distilling to smaller encoders.
- The multi-agent annotation pipeline incurs significant API costs at scale (50k+ clips through GPT-4o). Budget accordingly or use open-weight alternatives for the synthesis step.
- Emotion taxonomies vary across datasets (Ekman 6, VAD continuous, sentiment polarity). The unified label mapping loses granularity -- "anger" in IEMOCAP (dyadic conversation) differs from "anger" in DFEW (film clips). Interpret cross-dataset benchmarks with this caveat.
- The approach is trained primarily on English-language datasets. Emotion expression norms vary across cultures and languages; direct application to non-English content may reduce accuracy.
- Free-form reasoning quality depends heavily on the annotation pipeline quality. If the synthesized descriptions contain errors, Stage 2 training will learn flawed reasoning patterns.

## Reference

**Paper:** [Emotion-LLaMAv2 and MMEVerse: A New Framework and Benchmark for Multimodal Emotion Understanding](https://arxiv.org/abs/2601.16449v1) (Peng et al., 2026). Key sections: Section 3 for the multiview encoder and Conv-Attention pre-fusion architecture, Section 4 for the perception-to-cognition curriculum, Section 5 for MMEVerse benchmark construction and the multi-agent annotation pipeline.