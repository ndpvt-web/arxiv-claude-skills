---
name: "avere-improving-audiovisual-emotion"
description: "Build emotion-aware multimodal AI systems that resist spurious cue-emotion associations and hallucinated audiovisual evidence. Apply AVEm-DPO preference optimization and EmoReAlM-style evaluation to ground emotion reasoning in actual sensory inputs. Trigger phrases: 'emotion recognition pipeline', 'multimodal emotion analysis', 'audiovisual sentiment system', 'emotion reasoning from video', 'reduce emotion hallucinations', 'preference-optimize emotion model'"
---

# AVERE: Audiovisual Emotion Reasoning with Preference Optimization

This skill enables Claude to design, build, and evaluate multimodal emotion understanding systems that are grounded in actual audiovisual evidence rather than text-based shortcuts. It applies the AVEm-DPO framework from AVERE (ICLR 2026) to construct preference-optimized training pipelines that penalize two failure modes: (1) spurious associations where models link emotions to irrelevant cues (e.g., assuming "outdoor scene" implies "happiness"), and (2) hallucinated audiovisual evidence where models fabricate visual expressions or vocal tones based on language priors rather than actual input. The skill also covers building EmoReAlM-style benchmarks to rigorously evaluate these failure modes before deployment.

## When to Use

- When the user is building a video or audio emotion recognition system and wants to reduce false associations between scene context and predicted emotions
- When designing a multimodal LLM fine-tuning pipeline for affective computing tasks (sentiment, emotion, mood detection from video/audio)
- When the user asks to evaluate whether their emotion model hallucinates audiovisual cues it never actually observed
- When constructing preference pair datasets for DPO training on emotion-centric tasks
- When building evaluation benchmarks that test modality agreement (do audio-predicted and video-predicted emotions align?)
- When the user needs to add a regularization term that reduces text-prior reliance in a multimodal model
- When debugging an emotion classifier that performs well on text-heavy inputs but fails on audio-only or visual-only streams

## Key Technique

**AVEm-DPO** (AudioVisual Emotion DPO) extends standard Direct Preference Optimization for emotion-specific multimodal alignment. Standard DPO constructs preference pairs (chosen response > rejected response) from general quality judgments. AVEm-DPO constructs preferences along two targeted axes: *spurious association suppression* and *hallucination suppression*. A chosen response correctly identifies that a furrowed brow indicates frustration based on the actual video frame; a rejected response claims "the speaker sounds angry" when no audio was provided, or links "rainy background" to "sadness" without facial or vocal evidence. The preference pairs are built from audiovisual input pairs guided by emotion-centric textual prompts, ensuring the optimization signal is tied to genuine multimodal cues.

The critical innovation is a **text-prior regularization term** added to the DPO objective. Multimodal LLMs inherit strong text priors from their language backbone -- they "know" that funerals are sad and parties are happy, independent of what the audio or video actually shows. The regularization term explicitly penalizes response distributions that remain unchanged when audiovisual inputs are masked or replaced, forcing the model to attend to actual sensory signals rather than relying on textual context alone. This is measured by comparing model outputs with and without the audiovisual modality: if removing the video/audio stream doesn't change the prediction, the model is relying on text priors and gets penalized.

**EmoReAlM** (Emotion Reasoning and Alignment Metric) provides structured evaluation across five task categories: basic audio emotion reasoning, basic visual emotion reasoning, modality agreement (cross-modal consistency), audio hallucination stress tests (does the model fabricate audio cues?), and visual hallucination stress tests (does the model invent facial expressions?). This decomposed evaluation reveals *where* a model fails rather than just reporting aggregate accuracy.

## Step-by-Step Workflow

1. **Define the emotion taxonomy and modality channels.** Specify the target emotions (e.g., Ekman's 6 basic emotions, valence-arousal, or domain-specific labels) and which input modalities are available (video frames, audio waveform, transcribed text). Document which cues map to which emotions per modality (e.g., audio: pitch contour, speech rate, voice quality; visual: facial action units, body posture, gaze direction).

2. **Build the base multimodal inference pipeline.** Set up the MLLM (e.g., VideoLLaMA, LLaVA-Video, or similar) with separate encoders for each modality. Ensure the pipeline can accept partial inputs -- audio-only, video-only, or combined -- since modality-ablation is required for both training and evaluation.

3. **Generate candidate responses for preference pair construction.** For each training sample, run the base model under multiple conditions: (a) full audiovisual input, (b) audio-only with video masked, (c) video-only with audio masked, (d) text prompt only with all AV masked. Collect the model's emotion reasoning outputs from each condition.

4. **Construct AVEm-DPO preference pairs.** Create two types of preference data:
   - *Spurious association pairs*: Chosen = response that cites observable cues (e.g., "the speaker's raised voice and furrowed brow suggest anger"); Rejected = response that cites irrelevant context (e.g., "the dark room suggests sadness").
   - *Hallucination pairs*: Chosen = response that correctly states "no audio information is available" when audio is absent; Rejected = response that fabricates audio cues (e.g., "the trembling voice indicates fear") when no audio was provided.

5. **Implement the AVEm-DPO training objective.** Start with the standard DPO loss over preference pairs, then add the text-prior regularization:
   ```
   L_total = L_dpo(chosen, rejected) + lambda * KL(p(y|text_only) || p_uniform)
   ```
   where `lambda` controls regularization strength and the KL term penalizes confident predictions made without audiovisual input. In practice, compute the model's output distribution with AV inputs zeroed out and penalize it for diverging from a uniform (uncertain) distribution over emotion labels.

6. **Train with modality-aware batching.** Construct training batches that mix full-modality, audio-only, and video-only samples. For each batch, compute the DPO loss on the preference pairs and the regularization term on the text-only forward pass. Use a low learning rate (1e-6 to 5e-6) with LoRA or QLoRA for parameter-efficient training.

7. **Build an EmoReAlM-style evaluation suite.** Create test questions across five categories:
   - *Audio reasoning*: "What emotion does the speaker's tone convey?" (audio provided)
   - *Visual reasoning*: "What emotion is shown by the person's facial expression?" (video provided)
   - *Modality agreement*: "Do the facial expression and voice tone convey the same emotion?" (both provided)
   - *Audio hallucination stress test*: "Describe the speaker's vocal emotion cues." (no audio provided -- correct answer: "No audio is available")
   - *Visual hallucination stress test*: "Describe the person's facial expression." (no video provided -- correct answer: "No visual information is available")

8. **Run decomposed evaluation and compute per-category metrics.** Report Unweighted Average Recall (UAR), Weighted Average Recall (WAR), and F1 per category. Critically, report the hallucination rate (% of responses that fabricate cues for missing modalities) and spurious association rate (% of responses citing irrelevant context cues) separately.

9. **Iterate on preference data quality.** Inspect failure cases from evaluation. If the model still hallucinates specific cue types (e.g., always fabricating "trembling voice"), add targeted preference pairs for those patterns. If spurious associations persist for specific emotion-context pairs (e.g., "outdoor = happy"), add contrastive examples where the same context appears with different emotions.

10. **Deploy with runtime modality verification.** In production, add a preprocessing check that verifies which modalities actually contain signal before prompting the model. If audio energy is below threshold, explicitly indicate "audio stream contains no speech" in the prompt rather than passing silent audio, which reduces hallucination triggers.

## Concrete Examples

**Example 1: Building a video emotion analysis API**

User: "I want to build an API that takes a video clip and returns the dominant emotion with reasoning about what visual and audio cues support that conclusion."

Approach:
1. Set up a multimodal pipeline that extracts video frames at 1 FPS and audio as mel-spectrogram
2. Use a base MLLM (e.g., VideoLLaMA2) with the prompt: "Analyze the person's emotion in this video. Cite specific visual cues (facial expression, body language) and audio cues (voice tone, speech rate) that support your conclusion."
3. Generate preference training data by comparing outputs from full-AV vs. text-only conditions
4. Fine-tune with AVEm-DPO, setting lambda=0.1 for the text-prior regularization term
5. Evaluate on held-out set using EmoReAlM-style categories to verify hallucination rate < 5%

Output:
```json
{
  "emotion": "frustration",
  "confidence": 0.82,
  "visual_cues": [
    {"cue": "furrowed_brow", "timestamp": 1.2, "confidence": 0.88},
    {"cue": "tight_lip_press", "timestamp": 1.5, "confidence": 0.75}
  ],
  "audio_cues": [
    {"cue": "elevated_pitch", "timestamp": 0.8, "confidence": 0.79},
    {"cue": "increased_speech_rate", "timestamp": 1.0, "confidence": 0.71}
  ],
  "reasoning": "The speaker shows frustration through a furrowed brow visible at 1.2s combined with elevated vocal pitch starting at 0.8s. No contextual shortcuts were used -- the neutral office background does not inform the emotion prediction."
}
```

**Example 2: Evaluating an existing emotion model for hallucination**

User: "My emotion recognition model seems to always predict 'sadness' for slow-paced videos even when the person is calm and neutral. How do I diagnose and fix this?"

Approach:
1. This is a classic spurious association: slow pacing -> sadness. Build a diagnostic test set:
   - Collect slow-paced clips with neutral, happy, and sad ground truth labels
   - Collect fast-paced clips with the same emotion distribution
2. Run the model on both sets and compute per-emotion accuracy stratified by pacing
3. If accuracy for non-sad emotions drops significantly on slow-paced clips, the spurious association is confirmed
4. Construct targeted preference pairs:
   - Chosen: "The person appears calm and neutral. The slow video pacing is a recording artifact, not an emotional signal."
   - Rejected: "The slow, drawn-out footage conveys a sense of melancholy and sadness."
5. Fine-tune with AVEm-DPO using these pairs plus the text-prior regularization to break the pacing-sadness link

Output (diagnostic report):
```
Spurious Association Diagnostic: video_pacing -> sadness

| Condition        | Neutral Acc | Happy Acc | Sad Acc |
|------------------|-------------|-----------|---------|
| Slow-paced clips | 31%         | 28%       | 89%    |
| Fast-paced clips | 74%         | 81%       | 72%    |

Finding: Model accuracy for non-sad emotions drops 40-53% on slow-paced clips.
Root cause: Spurious association between temporal pacing and sadness.
Recommendation: Apply AVEm-DPO with targeted preference pairs (n=500) to decouple pacing from emotion prediction.
```

**Example 3: Creating preference pairs from an emotion dataset**

User: "I have the RAVDESS dataset. How do I construct AVEm-DPO preference pairs from it?"

Approach:
1. For each RAVDESS clip (which has both audio and video with emotion labels):
   - Run base model with full AV input -> get response R_full
   - Run base model with audio masked (silent) -> get response R_noaudio
   - Run base model with video masked (black frames) -> get response R_novideo
   - Run base model with text prompt only -> get response R_textonly
2. Construct preference pairs:
   - If R_noaudio still claims to hear vocal cues: pair(chosen=R_full, rejected=R_noaudio) tagged as "audio hallucination"
   - If R_novideo still describes facial expressions: pair(chosen=R_full, rejected=R_novideo) tagged as "visual hallucination"
   - If R_textonly predicts correctly without any AV: pair(chosen=R_full, rejected=R_textonly) tagged as "text-prior reliance"
3. Filter pairs where chosen and rejected actually differ in quality (skip if both are correct or both wrong)
4. Balance pairs across emotion categories to avoid class-imbalanced DPO training

Output (preference pair example):
```json
{
  "prompt": "What emotion does this person express? Explain what audio and visual cues support your answer.",
  "chosen": "The person expresses anger. Their eyebrows are sharply lowered (AU4), jaw is clenched, and their voice has a raised fundamental frequency with harsh vocal quality.",
  "rejected": "The person expresses anger. Their voice trembles with rage and their eyes are wide with fury.",
  "rejection_reason": "hallucination",
  "note": "Rejected response fabricates 'trembling voice' and 'wide eyes' not present in the actual clip. Ground truth AU coding shows AU4+AU23 (brow lowerer + lip tightener), not AU5 (upper lid raiser)."
}
```

## Best Practices

- **Do:** Always test your model with modality-ablated inputs (audio-only, video-only, text-only) before and after training. The hallucination rate on missing-modality inputs is the most revealing metric.
- **Do:** Include the text-prior regularization term even if your preference pairs seem sufficient. Text priors are subtle and persistent -- models can learn to satisfy preference pairs while still relying on text shortcuts for edge cases.
- **Do:** Balance preference pairs across emotion categories. If 80% of your "rejected" examples involve fabricated sadness cues, the model will overfit to suppressing sadness-related language rather than learning general grounding.
- **Do:** Use emotion-specific evaluation (per-emotion recall, per-cue-type hallucination rate) rather than aggregate accuracy. Aggregate numbers hide category-level failures.
- **Avoid:** Using DPO preference pairs where the chosen and rejected responses are identical except for the emotion label. The preference signal should capture *reasoning quality*, not just label correctness.
- **Avoid:** Training on preference pairs from a single modality condition. Mix full-AV, audio-only, and video-only conditions in your preference data to teach cross-modal grounding.
- **Avoid:** Setting the regularization lambda too high (>0.5), which can suppress all text-based reasoning including legitimate linguistic context. Start at lambda=0.05-0.1 and tune on a validation set.

## Error Handling

- **Model produces identical outputs with and without AV inputs:** The text-prior regularization lambda is too low or the base model's AV encoders are not properly connected. Verify encoder outputs are non-zero and increase lambda incrementally.
- **DPO training collapses (all responses become generic):** Preference pairs may be too similar or the learning rate too high. Reduce LR to 1e-6, increase margin between chosen/rejected quality, and add a KL penalty to the reference model.
- **Hallucination rate increases for one modality after fixing another:** This is a known seesaw effect. Add joint preference pairs that test both modalities simultaneously, and ensure the regularization term covers both audio and visual masking conditions in each batch.
- **Evaluation scores are high but deployed model still hallucinates:** The evaluation set may not cover adversarial conditions. Add stress tests with misleading context (e.g., a smiling person at a funeral, an angry tone reading a joke) to detect residual spurious associations.

## Limitations

- This technique requires a base MLLM that accepts multimodal inputs with the ability to selectively mask modalities. It does not apply to unimodal text-only emotion classifiers.
- Preference pair construction requires either expert annotation of audiovisual cues or a reliable base model to generate candidate responses. Low-quality preference data degrades DPO training.
- The text-prior regularization assumes you can run the model in text-only mode. Architectures that fuse modalities early (before the language model) may not support clean modality ablation.
- Real-time deployment adds latency from the modality verification preprocessing step. For latency-sensitive applications, the verification can be approximated with simple energy/motion thresholds.
- The approach is validated on acted/posed emotion datasets (RAVDESS, DFEW). Performance on spontaneous, naturalistic emotions with subtle expressions may differ and requires additional evaluation.

## Reference

**Paper:** [AVERE: Improving Audiovisual Emotion Reasoning with Preference Optimization](https://arxiv.org/abs/2602.07054) (ICLR 2026)
**Key insight:** Constructing DPO preference pairs that explicitly target spurious cue-emotion associations and modality hallucinations, combined with a text-prior regularization term, yields 6-19% relative gains in zero-shot emotion reasoning -- look for Section 3 (AVEm-DPO formulation) and Section 4 (EmoReAlM benchmark design) for implementation details.
**Project page:** https://avere-iclr.github.io