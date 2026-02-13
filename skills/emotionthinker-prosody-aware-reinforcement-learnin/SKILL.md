---
name: "emotionthinker-prosody-aware-reinforcement-learnin"
description: "Build prosody-aware speech emotion reasoning pipelines using Chain-of-Thought RL. Implements EmotionThinker's GRPO-PTR training loop, prosody feature extraction, multi-dimensional reward models, and structured emotion CoT annotation. Triggers: 'build speech emotion reasoning system', 'implement prosody-aware emotion recognition', 'create emotion chain-of-thought pipeline', 'train SpeechLLM with GRPO-PTR', 'extract prosodic features for emotion analysis', 'build interpretable SER system'"
---

# EmotionThinker: Prosody-Aware RL for Explainable Speech Emotion Reasoning

This skill enables Claude to build speech emotion recognition (SER) systems that go beyond simple classification by generating interpretable Chain-of-Thought (CoT) explanations grounded in prosodic cues. Based on the EmotionThinker framework, it teaches how to extract prosodic features (pitch, energy, speech rate, stress, intonation), construct emotion reasoning datasets with CoT annotations, train SpeechLLMs with a custom GRPO-PTR reinforcement learning algorithm that balances outcome accuracy with reasoning quality, and evaluate results using multi-dimensional reward models.

## When to Use

- When the user asks to build a speech emotion recognition system that provides explanations, not just labels
- When implementing a reinforcement learning post-training loop for a SpeechLLM with both outcome and reasoning rewards
- When extracting prosodic features (pitch, energy, speech rate, word-level stress, intonation contours) from audio for emotion analysis
- When constructing a Chain-of-Thought emotion reasoning dataset from raw speech corpora (IEMOCAP, MELD, etc.)
- When building a multi-dimensional reward model that scores reasoning quality across factual alignment, interpretive quality, caption completeness, and fluency
- When the user needs to implement progressive reward scheduling in GRPO (introducing reasoning reward only after a baseline accuracy threshold)
- When designing a `<think>...</think><answer>...</answer>` structured output format for speech understanding tasks

## Key Technique

**Reformulating SER as Deep Reasoning.** Traditional SER maps audio to a discrete label. EmotionThinker instead trains a SpeechLLM to produce structured reasoning in `<think>` tags — describing speaker traits, prosodic patterns, and semantic content — before emitting the emotion label in `<answer>` tags. This is enabled by EmotionCoT-35K, a dataset of 35,000 speech-reasoning pairs where GPT-4o generates step-wise reasoning traces conditioned on automatically extracted prosodic annotations (pitch level, energy level, speech rate, stressed words, intonation contour, speaker gender/age).

**GRPO-PTR: Progressive Trust-aware Reasoning Reward.** Standard Group Relative Policy Optimization (GRPO) uses only rule-based outcome rewards. GRPO-PTR extends this with three innovations: (1) A composite reward `R = alpha_f * R_f + alpha_o * R_o + alpha_t * tau * R_t` combining format, outcome, and reasoning rewards. (2) A trustworthiness weight `tau` that compares mean reasoning scores of correct vs. incorrect predictions — if incorrect answers score higher on reasoning than correct ones, `tau` decays exponentially, preventing reward hacking. (3) Progressive scheduling that trains with only outcome and format rewards until 50% accuracy, then introduces reasoning reward for stable optimization.

**Prosody-Enhanced Foundation.** Before RL, a supervised fine-tuning stage trains EmotionThinker-Base on ~500 hours covering word-level stress prediction (WhiStress on Whisper embeddings), prosodic attribute classification, comparative augmentation tasks, and a seed set of 5K EmotionCoT samples. This prosody-enhanced base dramatically improves downstream emotion reasoning compared to starting from a generic SpeechLLM.

## Step-by-Step Workflow

1. **Extract low-level prosodic features from audio files.** For each utterance, compute frame-level pitch and energy from short-time acoustic frames, estimate speaking rate via phoneme-level forced alignment, and classify each into categorical bins (low/normal/high for pitch and energy; slow/normal/fast for rate).

2. **Detect word-level stress.** Run a WhiStress-style model over intermediate Whisper encoder embeddings to produce per-token stress predictions. This avoids requiring forced alignment at inference time and captures emphasis patterns critical to emotion.

3. **Analyze intonation contours.** Apply Savitzky-Golay smoothing to pitch-energy trajectories, then classify each utterance's intonation pattern: rising, falling, rising-falling, or falling-rising. Also flag expressive vs. flat delivery.

4. **Extract speaker attributes.** Use a Wav2Vec2.0-based classifier to predict speaker gender and age group (child, young adult, middle-aged, elderly). These condition the reasoning chain.

5. **Generate CoT annotations with an LLM.** Feed the extracted prosodic annotations, transcription, and ground-truth emotion label to GPT-4o (or equivalent) with a structured prompt that requests multi-step reasoning connecting speaker traits, prosodic cues, and semantic content to the emotion. Format output as `<think>reasoning</think><answer>emotion_label</answer>`.

6. **Supervised fine-tune a SpeechLLM on prosody tasks.** Train the audio encoder, audio adapter, and LLM backbone jointly on stress prediction, prosodic attribute classification, comparative prosody tasks, and seed CoT samples for one epoch (~500 hours total). This produces the prosody-enhanced base model.

7. **Build the multi-dimensional reward model.** Fine-tune a smaller model (e.g., Qwen2.5-Omni-3B) on ~100K (prompt, reasoning, scores) tuples to output JSON with four scores (1-5 each): `factual_alignment`, `interpretative_quality`, `caption_completeness`, `fluency_and_structural_clarity`. Normalize each by dividing by 5 and compute a weighted sum for the scalar reasoning reward `R_t`.

8. **Implement the GRPO-PTR training loop.** For each input, sample K=8 candidate responses. Compute format reward `R_f` (1 if output matches XML schema, 0 otherwise), outcome reward `R_o` (1 if emotion label matches ground truth, 0 otherwise), and reasoning reward `R_t` from the reward model. Calculate trustworthiness `tau`: if mean `R_t` of correct answers >= mean `R_t` of incorrect, set `tau=1`; otherwise `tau = exp(mean_correct - mean_incorrect)`. Combine as `R = 0.3*R_f + 1.0*R_o + 0.5*tau*R_t`. Only introduce `R_t` after the model exceeds 50% accuracy.

9. **Run RL post-training.** Train for ~3,000 steps on 30K samples with learning rate 1e-6, KL coefficient 0.04, using the composite GRPO-PTR reward. Monitor both emotion accuracy and reasoning quality scores throughout.

10. **Evaluate on benchmarks.** Test on held-out splits of IEMOCAP, MELD, RAVDESS, SAVEE. Report both classification accuracy and reasoning quality (four-dimensional scores from the reward model, plus human evaluation on a 100-sample subset).

## Concrete Examples

**Example 1: Building the Prosody Feature Extraction Pipeline**

User: "I need to extract prosodic features from a directory of WAV files for emotion analysis."

Approach:
1. Set up a Python pipeline using `librosa` for pitch/energy extraction and a forced aligner (e.g., MFA) for phoneme-level timing
2. Compute pitch via `librosa.pyin()`, energy via RMS, and speech rate as phonemes/second
3. Bin continuous values into categorical labels using corpus-level statistics (e.g., terciles)
4. Run a Whisper-based stress detection model for per-token stress predictions
5. Apply Savitzky-Golay smoothing to pitch contour and classify intonation pattern

Output:
```python
import librosa
import numpy as np
from scipy.signal import savgol_filter

def extract_prosody(audio_path: str, sr: int = 16000) -> dict:
    y, sr = librosa.load(audio_path, sr=sr)

    # Pitch extraction (F0)
    f0, voiced_flag, _ = librosa.pyin(y, fmin=50, fmax=500, sr=sr)
    f0_valid = f0[voiced_flag]
    mean_pitch = np.nanmean(f0_valid) if len(f0_valid) > 0 else 0.0

    # Energy (RMS)
    rms = librosa.feature.rms(y=y)[0]
    mean_energy = float(np.mean(rms))

    # Pitch contour smoothing for intonation classification
    f0_filled = np.nan_to_num(f0, nan=0.0)
    if len(f0_filled) > 11:
        smoothed = savgol_filter(f0_filled, window_length=11, polyorder=3)
    else:
        smoothed = f0_filled

    intonation = classify_intonation(smoothed)

    return {
        "mean_pitch": mean_pitch,
        "pitch_level": bin_value(mean_pitch, pitch_thresholds),  # "low"/"normal"/"high"
        "mean_energy": mean_energy,
        "energy_level": bin_value(mean_energy, energy_thresholds),
        "intonation_pattern": intonation,  # "rising"/"falling"/"rising-falling"/"falling-rising"
    }

def classify_intonation(smoothed_f0: np.ndarray) -> str:
    voiced = smoothed_f0[smoothed_f0 > 0]
    if len(voiced) < 4:
        return "flat"
    mid = len(voiced) // 2
    first_half_mean = np.mean(voiced[:mid])
    second_half_mean = np.mean(voiced[mid:])
    delta = second_half_mean - first_half_mean
    if delta > 15:
        return "rising"
    elif delta < -15:
        return "falling"
    else:
        return "flat"
```

**Example 2: Implementing the GRPO-PTR Reward Computation**

User: "Implement the GRPO-PTR reward function with trustworthiness weighting."

Approach:
1. Define the three reward components: format, outcome, reasoning
2. Implement the trustworthiness weight `tau` that penalizes misaligned reasoning scores
3. Implement progressive scheduling that gates reasoning reward on accuracy threshold

Output:
```python
import math
from dataclasses import dataclass

@dataclass
class GRPOPTRConfig:
    alpha_f: float = 0.3    # Format reward weight
    alpha_o: float = 1.0    # Outcome reward weight
    alpha_t: float = 0.5    # Reasoning reward weight
    accuracy_threshold: float = 0.5  # Threshold to introduce reasoning reward

def compute_format_reward(response: str) -> float:
    """R_f: 1 if response follows <think>...</think><answer>...</answer> schema."""
    import re
    pattern = r"<think>.+?</think>\s*<answer>.+?</answer>"
    return 1.0 if re.search(pattern, response, re.DOTALL) else 0.0

def compute_outcome_reward(predicted: str, ground_truth: str) -> float:
    """R_o: 1 if predicted emotion matches ground truth."""
    return 1.0 if predicted.strip().lower() == ground_truth.strip().lower() else 0.0

def compute_trustworthiness(
    reasoning_scores: list[float],
    outcome_correct: list[bool],
) -> float:
    """Tau: penalizes when incorrect answers get higher reasoning scores."""
    correct_scores = [s for s, c in zip(reasoning_scores, outcome_correct) if c]
    wrong_scores = [s for s, c in zip(reasoning_scores, outcome_correct) if not c]

    if not correct_scores or not wrong_scores:
        return 1.0  # No comparison possible, trust fully

    mean_correct = sum(correct_scores) / len(correct_scores)
    mean_wrong = sum(wrong_scores) / len(wrong_scores)

    if mean_correct >= mean_wrong:
        return 1.0
    return math.exp(mean_correct - mean_wrong)  # Exponential decay

def compute_grpo_ptr_reward(
    responses: list[str],
    ground_truth: str,
    reward_model,  # Callable that returns 4-dim scores
    config: GRPOPTRConfig,
    current_accuracy: float,
) -> list[float]:
    """Compute composite GRPO-PTR reward for K sampled responses."""
    format_rewards = [compute_format_reward(r) for r in responses]
    outcomes = [compute_outcome_reward(extract_answer(r), ground_truth) for r in responses]

    # Progressive gating: only use reasoning reward after threshold
    if current_accuracy < config.accuracy_threshold:
        return [
            config.alpha_f * f + config.alpha_o * o
            for f, o in zip(format_rewards, outcomes)
        ]

    # Get multi-dimensional reasoning scores from reward model
    reasoning_scores = []
    for r in responses:
        scores = reward_model(r)  # Returns dict with 4 criteria, each 1-5
        normalized = sum(v / 5.0 for v in scores.values()) / len(scores)
        reasoning_scores.append(normalized)

    tau = compute_trustworthiness(
        reasoning_scores, [o == 1.0 for o in outcomes]
    )

    return [
        config.alpha_f * f + config.alpha_o * o + config.alpha_t * tau * rt
        for f, o, rt in zip(format_rewards, outcomes, reasoning_scores)
    ]
```

**Example 3: Generating CoT Emotion Annotations**

User: "I have prosodic features extracted from IEMOCAP. Help me generate Chain-of-Thought emotion reasoning annotations."

Approach:
1. Format prosodic features into a structured prompt
2. Call an LLM to generate reasoning traces connecting prosody to emotion
3. Parse and validate the `<think>/<answer>` output format

Output:
```python
EMOTION_COT_PROMPT = """You are annotating speech emotion data. Given the following prosodic analysis of an utterance, generate a step-by-step reasoning chain that explains why the speaker is feeling the given emotion.

**Transcription:** {transcription}
**Ground-truth emotion:** {emotion}
**Speaker:** {gender}, {age_group}
**Pitch level:** {pitch_level}
**Energy level:** {energy_level}
**Speech rate:** {speech_rate}
**Stressed words:** {stressed_words}
**Intonation pattern:** {intonation}

Write your response in this exact format:
<think>
Step 1: Describe the speaker's vocal characteristics.
Step 2: Analyze how prosodic features (pitch, energy, rate, stress) relate to emotional expression.
Step 3: Connect the semantic content with the acoustic cues to infer emotion.
</think>
<answer>{emotion}</answer>
"""

def generate_emotion_cot(
    sample: dict,
    llm_client,
) -> dict:
    prompt = EMOTION_COT_PROMPT.format(
        transcription=sample["transcription"],
        emotion=sample["emotion"],
        gender=sample["gender"],
        age_group=sample["age_group"],
        pitch_level=sample["pitch_level"],
        energy_level=sample["energy_level"],
        speech_rate=sample["speech_rate"],
        stressed_words=", ".join(sample["stressed_words"]),
        intonation=sample["intonation"],
    )

    response = llm_client.chat(prompt)

    # Validate format
    import re
    match = re.search(
        r"<think>(.*?)</think>\s*<answer>(.*?)</answer>",
        response, re.DOTALL,
    )
    if not match:
        raise ValueError(f"Invalid CoT format in response: {response[:100]}...")

    return {
        "audio_path": sample["audio_path"],
        "transcription": sample["transcription"],
        "emotion": sample["emotion"],
        "reasoning": match.group(1).strip(),
        "cot_response": response.strip(),
    }
```

## Best Practices

- **Do:** Extract prosodic features at multiple granularities — frame-level (pitch, energy), word-level (stress), and utterance-level (intonation contour, speech rate). Each granularity captures different emotional signals.
- **Do:** Use progressive reward scheduling. Introduce reasoning reward (`R_t`) only after the model reaches ~50% accuracy with outcome reward alone. Premature reasoning optimization causes training instability.
- **Do:** Compute the trustworthiness weight `tau` per training batch. It dynamically prevents reward hacking where the model generates plausible-sounding but incorrect reasoning chains.
- **Do:** Validate that CoT annotations follow the `<think>...</think><answer>...</answer>` schema before including them in training data. Malformed examples degrade RL stability.
- **Avoid:** Treating emotion recognition as a simple classification head on top of speech embeddings. The core insight is that structured reasoning over prosodic cues yields both better accuracy and interpretability.
- **Avoid:** Using a single scalar reward during RL. The multi-dimensional reward model (factual alignment, interpretive quality, caption completeness, fluency) prevents the model from gaming any single metric.

## Error Handling

- **Pitch extraction fails on noisy audio:** Fall back to energy-only features and mark pitch as "unavailable" in the prosody annotation. The CoT generator can still reason from partial cues.
- **CoT generation produces malformed output:** Implement a retry loop (max 3 attempts) with the LLM, and if still malformed, exclude the sample from training rather than force-fitting.
- **Trustworthiness `tau` is always near 0:** This indicates the reward model is systematically scoring incorrect answers higher. Retrain the reward model with more balanced correct/incorrect examples.
- **RL training accuracy plateaus below 50%:** The reasoning reward will never activate. Lower the progressive threshold or extend SFT warm-up with more CoT seed samples.
- **K sampled responses are all identical:** Increase sampling temperature or reduce KL coefficient (default 0.04) to encourage exploration.

## Limitations

- Requires substantial compute: SFT on ~500 hours of audio + 3,000 RL steps with K=8 sampling per input demands significant GPU resources.
- CoT annotation quality depends on the prosodic feature extraction pipeline. Errors in pitch tracking or stress detection propagate into reasoning chains.
- The 9-category emotion taxonomy (neutral, happy, sad, angry, contempt/disgust, confused, whisper, surprise, fear) may not cover domain-specific emotions (e.g., sarcasm, boredom).
- The trustworthiness mechanism assumes correct and incorrect prediction groups both exist in each batch — with very high or low accuracy, `tau` degenerates.
- Prosody extraction tools (forced aligners, WhiStress) add pipeline complexity and may not generalize well to languages beyond English without retraining.

## Reference

[EmotionThinker: Prosody-Aware Reinforcement Learning for Explainable Speech Emotion Reasoning](https://arxiv.org/abs/2601.15668v1) — Wang et al., 2026. Focus on Section 3 (GRPO-PTR algorithm), Section 2.2 (EmotionCoT-35K construction pipeline), and Table 2 (ablation showing prosody enhancement boosts accuracy by ~5% absolute). Project code: [github.com/dingdongwang/EmotionThinker](https://github.com/dingdongwang/EmotionThinker).