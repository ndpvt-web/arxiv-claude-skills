---
name: "vowelprompt-hearing-speech-emotions"
description: "Build speech emotion recognition pipelines that augment LLMs with vowel-level prosodic features converted to natural language. Use when: 'analyze emotion in speech audio', 'build a speech emotion classifier with LLM', 'extract prosodic features from audio for emotion detection', 'augment text transcription with pitch and energy cues', 'create an interpretable emotion recognition system', 'convert acoustic features to natural language prompts'."
---

# VowelPrompt: Vowel-Level Prosodic Augmentation for Speech Emotion Recognition

This skill enables Claude to help users build speech emotion recognition (SER) systems that combine LLM reasoning with fine-grained acoustic prosody. The core technique from VowelPrompt (ICLR 2026) extracts pitch, energy, and duration descriptors specifically from vowel segments in speech, converts them to natural language descriptions, and feeds these alongside the transcript to an LLM. This bridges the gap between text-only LLM reasoning and the rich emotional information carried in how words are spoken, producing both accurate classifications and interpretable explanations grounded in actual acoustic evidence.

## When to Use

- When the user wants to build a speech emotion recognition system that uses an LLM as the reasoning backbone
- When the user needs to convert raw audio prosodic features (F0, energy, duration) into structured natural language for LLM consumption
- When the user asks to extract vowel-level acoustic features from speech using forced alignment
- When the user wants interpretable emotion predictions that cite specific prosodic evidence (not just a label)
- When the user is building a pipeline that combines ASR transcripts with acoustic analysis for sentiment or emotion tasks
- When the user needs to fine-tune an LLM for emotion recognition using SFT followed by GRPO-based reinforcement learning
- When the user asks how to improve zero-shot or cross-domain emotion recognition by grounding LLM prompts in acoustic features

## Key Technique

**Why vowels?** Phonetic research shows that vowels are the primary carriers of affective prosody in speech. Unlike consonants, vowels sustain voicing and allow pitch modulation, energy variation, and temporal stretching — the exact dimensions humans use to express emotion. By isolating vowel segments, VowelPrompt targets the acoustically richest regions of speech for emotion, discarding noisy or uninformative consonantal segments.

**The pipeline works in three stages.** First, use a forced aligner (Montreal Forced Aligner / MFA) to obtain phoneme-level time boundaries from the audio + transcript pair, then filter to keep only vowel phonemes. Second, extract three prosodic features from each vowel segment: (1) pitch contour statistics (mean F0, F0 slope/direction), (2) energy/intensity (mean RMS or dB), and (3) duration in milliseconds. These continuous values are discretized into categorical descriptors using speaker-normalized thresholds (e.g., "high pitch", "rising intonation", "elongated duration", "low energy"). Third, assemble a natural language prosodic description that augments the raw transcript, creating a unified text prompt the LLM can reason over.

**Two-stage LLM adaptation** then sharpens performance. Stage 1 is standard supervised fine-tuning (SFT) on emotion-labeled data with prosodic augmentation. Stage 2 applies Reinforcement Learning with Verifiable Reward (RLVR) via Group Relative Policy Optimization (GRPO) — the reward signal is whether the model's predicted emotion label matches the ground truth, which is verifiable without a learned reward model. This second stage improves structured output adherence, cross-domain generalization, and reasoning quality.

## Step-by-Step Workflow

1. **Set up forced alignment.** Install Montreal Forced Aligner (MFA) and download the appropriate acoustic model and pronunciation dictionary for the target language. Verify alignment works on a sample utterance.

2. **Obtain phoneme-level alignments.** Run MFA on each audio file paired with its transcript to produce TextGrid files containing phone-level start/end timestamps. Parse the TextGrid to extract all phoneme intervals.

3. **Filter to vowel segments only.** Using the language's phoneme inventory, retain only vowel phonemes (for English: AA, AE, AH, AO, AW, AY, EH, ER, EY, IH, IY, OW, OY, UH, UW and their stress variants). Discard consonants and silence.

4. **Extract prosodic features per vowel.** For each vowel segment, compute:
   - **Pitch**: Mean F0 (Hz) and F0 slope (rising/falling/flat) using a pitch tracker like `parselmouth` (Praat in Python)
   - **Energy**: Mean intensity (dB) over the segment
   - **Duration**: Segment length in milliseconds

   Normalize features per speaker (z-score against speaker's own vowel statistics) to handle inter-speaker variability.

5. **Discretize into natural language descriptors.** Map normalized values to categorical labels using thresholds:
   - Pitch: "low" (z < -0.5), "moderate" (-0.5 <= z <= 0.5), "high" (z > 0.5); direction: "rising", "falling", "flat"
   - Energy: "soft", "moderate", "loud" (same z-score bins)
   - Duration: "shortened", "normal", "elongated"

6. **Assemble the prosodic prompt.** Construct a natural language description aggregating vowel-level descriptors into an utterance-level summary. Include both the overall pattern and notable outliers (e.g., stressed syllables with extreme pitch).

7. **Build the LLM prompt.** Combine the transcript, the prosodic description, and a task instruction into a single prompt. The LLM should be asked to reason over both textual content and prosodic cues before producing an emotion label and explanation.

8. **Fine-tune with SFT (Stage 1).** Train on emotion-labeled datasets (IEMOCAP, MELD, etc.) using standard cross-entropy on the target emotion label, with prosodic-augmented prompts as input. Use LoRA or QLoRA for parameter-efficient fine-tuning.

9. **Apply GRPO reinforcement learning (Stage 2).** After SFT, run GRPO where the reward is 1.0 if the predicted emotion matches ground truth and 0.0 otherwise. Generate multiple candidate responses per prompt, rank them by reward, and update policy using relative advantages within each group. This improves format compliance and cross-domain transfer.

10. **Evaluate across conditions.** Test on held-out splits, unseen datasets (cross-domain), and other languages (cross-linguistic) to validate generalization. Report weighted F1 and accuracy.

## Concrete Examples

**Example 1: Building the vowel feature extraction pipeline**

User: "I have WAV files and transcripts. Help me extract vowel-level prosodic features for emotion recognition."

Approach:
1. Install dependencies and set up MFA alignment
2. Extract vowel segments and compute features
3. Discretize and format as natural language

Output (Python pipeline):
```python
import parselmouth
from parselmouth.praat import call
import textgrid
import numpy as np

ENGLISH_VOWELS = {
    'AA', 'AE', 'AH', 'AO', 'AW', 'AY', 'EH', 'ER', 'EY',
    'IH', 'IY', 'OW', 'OY', 'UH', 'UW',
    'AA0','AA1','AA2','AE0','AE1','AE2','AH0','AH1','AH2',
    'AO0','AO1','AO2','AW0','AW1','AW2','AY0','AY1','AY2',
    'EH0','EH1','EH2','ER0','ER1','ER2','EY0','EY1','EY2',
    'IH0','IH1','IH2','IY0','IY1','IY2','OW0','OW1','OW2',
    'OY0','OY1','OY2','UH0','UH1','UH2','UW0','UW1','UW2',
}

def extract_vowel_prosody(audio_path, textgrid_path):
    """Extract pitch, energy, duration from each vowel segment."""
    snd = parselmouth.Sound(audio_path)
    pitch_obj = snd.to_pitch()
    intensity_obj = snd.to_intensity()
    tg = textgrid.TextGrid.fromFile(textgrid_path)
    phone_tier = tg.getFirst("phones")

    vowel_features = []
    for interval in phone_tier:
        phone = interval.mark.strip().upper()
        if phone not in ENGLISH_VOWELS or interval.mark.strip() == '':
            continue
        start, end = interval.minTime, interval.maxTime
        dur_ms = (end - start) * 1000

        # Pitch: mean F0 and slope
        f0_values = [pitch_obj.get_value_at_time(t)
                     for t in np.linspace(start, end, 10)
                     if pitch_obj.get_value_at_time(t) is not None
                     and not np.isnan(pitch_obj.get_value_at_time(t))]
        mean_f0 = np.mean(f0_values) if f0_values else 0.0
        f0_slope = (f0_values[-1] - f0_values[0]) if len(f0_values) >= 2 else 0.0

        # Energy: mean intensity
        mean_energy = call(intensity_obj, "Get mean", start, end, "energy")

        vowel_features.append({
            "phone": interval.mark.strip(),
            "start": start, "end": end,
            "mean_f0": mean_f0,
            "f0_slope": f0_slope,
            "mean_energy_db": mean_energy,
            "duration_ms": dur_ms,
        })
    return vowel_features


def normalize_and_discretize(vowel_features, speaker_stats):
    """Convert raw features to categorical descriptors."""
    descriptors = []
    for vf in vowel_features:
        # Z-score normalize against speaker stats
        z_pitch = (vf["mean_f0"] - speaker_stats["f0_mean"]) / (speaker_stats["f0_std"] + 1e-8)
        z_energy = (vf["mean_energy_db"] - speaker_stats["energy_mean"]) / (speaker_stats["energy_std"] + 1e-8)
        z_dur = (vf["duration_ms"] - speaker_stats["dur_mean"]) / (speaker_stats["dur_std"] + 1e-8)

        pitch_level = "high" if z_pitch > 0.5 else ("low" if z_pitch < -0.5 else "moderate")
        pitch_dir = "rising" if vf["f0_slope"] > 10 else ("falling" if vf["f0_slope"] < -10 else "flat")
        energy_level = "loud" if z_energy > 0.5 else ("soft" if z_energy < -0.5 else "moderate")
        dur_level = "elongated" if z_dur > 0.5 else ("shortened" if z_dur < -0.5 else "normal")

        descriptors.append({
            "phone": vf["phone"],
            "pitch": f"{pitch_level}, {pitch_dir}",
            "energy": energy_level,
            "duration": dur_level,
        })
    return descriptors
```

**Example 2: Generating the natural language prosodic prompt**

User: "How do I convert the extracted vowel features into a text prompt for the LLM?"

Approach:
1. Aggregate vowel-level descriptors into utterance-level summary
2. Highlight outlier vowels that carry strong emotional signal
3. Format as a structured natural language block

Output:
```python
def build_prosodic_description(descriptors):
    """Convert vowel descriptors to natural language for LLM prompting."""
    if not descriptors:
        return "No vowel prosodic information available."

    # Aggregate patterns
    pitch_counts = {}
    energy_counts = {}
    dur_counts = {}
    for d in descriptors:
        p_level = d["pitch"].split(",")[0].strip()
        pitch_counts[p_level] = pitch_counts.get(p_level, 0) + 1
        energy_counts[d["energy"]] = energy_counts.get(d["energy"], 0) + 1
        dur_counts[d["duration"]] = dur_counts.get(d["duration"], 0) + 1

    dominant_pitch = max(pitch_counts, key=pitch_counts.get)
    dominant_energy = max(energy_counts, key=energy_counts.get)
    dominant_dur = max(dur_counts, key=dur_counts.get)

    summary = (
        f"The speaker's vowels predominantly exhibit {dominant_pitch} pitch, "
        f"{dominant_energy} energy, and {dominant_dur} duration. "
    )

    # Highlight notable vowels (outliers from dominant pattern)
    notable = [d for d in descriptors
               if d["pitch"].split(",")[0].strip() != dominant_pitch
               or d["energy"] != dominant_energy]
    if notable:
        highlights = "; ".join(
            f"'{d['phone']}' has {d['pitch']} pitch and {d['energy']} energy"
            for d in notable[:3]
        )
        summary += f"Notable variations: {highlights}."

    return summary


def build_llm_prompt(transcript, prosodic_desc, emotion_labels):
    """Assemble the final prompt for LLM-based emotion recognition."""
    labels_str = ", ".join(emotion_labels)
    return f"""You are an expert in speech emotion recognition. Given a spoken utterance's transcript and its vowel-level prosodic analysis, determine the speaker's emotion.

Transcript: "{transcript}"

Prosodic Analysis: {prosodic_desc}

Based on both the linguistic content and the prosodic cues described above, identify the speaker's emotion from [{labels_str}]. First explain your reasoning by citing specific textual and prosodic evidence, then state your final answer as: Emotion: <label>"""
```

**Example 3: GRPO reinforcement learning stage**

User: "How do I set up the GRPO training stage after SFT?"

Approach:
1. Generate multiple candidate outputs per prompt
2. Score each with verifiable reward (exact match on emotion label)
3. Compute group-relative advantages and update policy

Output:
```python
# Using trl library with GRPO trainer
from trl import GRPOConfig, GRPOTrainer
import re

def emotion_reward_fn(completions, ground_truth_emotions):
    """Verifiable reward: 1.0 if predicted emotion matches ground truth."""
    rewards = []
    for completion, gt in zip(completions, ground_truth_emotions):
        # Extract predicted emotion from structured output
        match = re.search(r"Emotion:\s*(\w+)", completion)
        predicted = match.group(1).lower().strip() if match else ""
        reward = 1.0 if predicted == gt.lower().strip() else 0.0
        rewards.append(reward)
    return rewards

# GRPO config after SFT checkpoint
grpo_config = GRPOConfig(
    output_dir="./vowelprompt-grpo",
    num_generations=8,        # candidates per prompt (group size)
    learning_rate=1e-6,
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    max_completion_length=512,
    num_train_epochs=2,
    logging_steps=10,
    bf16=True,
)

trainer = GRPOTrainer(
    model=sft_model,           # model from Stage 1
    config=grpo_config,
    tokenizer=tokenizer,
    train_dataset=prosodic_augmented_dataset,
    reward_funcs=[emotion_reward_fn],
)
trainer.train()
```

## Best Practices

- **Do:** Normalize prosodic features per-speaker before discretizing. Raw F0 values vary dramatically across speakers (male ~120Hz, female ~220Hz); z-score normalization makes descriptors comparable.
- **Do:** Include both the dominant prosodic pattern and notable exceptions in the natural language description. Emotional shifts often manifest as deviations from the speaker's baseline.
- **Do:** Use stressed vowels (marked with `1` in ARPAbet, e.g., `AE1`) as higher-weight evidence — stressed syllables carry more emotional information than unstressed ones.
- **Do:** Validate forced alignment quality before trusting vowel boundaries. Misaligned segments produce garbage features. Spot-check a random sample in Praat.
- **Avoid:** Extracting prosodic features from consonants or silence — they are noisy and phonetically uninformative for emotion.
- **Avoid:** Using absolute pitch/energy thresholds across speakers. Always normalize per-speaker or per-utterance to handle variability.
- **Avoid:** Skipping the GRPO stage. SFT alone tends to produce unstructured outputs and weaker cross-domain transfer. The RL stage specifically improves format compliance and generalization.

## Error Handling

- **MFA alignment fails on noisy audio:** Preprocess with noise reduction (e.g., `noisereduce` library) or VAD-based segmentation before alignment. Fall back to a simpler vowel detection heuristic if alignment is unreliable.
- **No vowels detected in segment:** This can happen with very short utterances or interjections. Fall back to utterance-level prosodic statistics (global F0 mean, energy) rather than producing an empty description.
- **Pitch tracker returns NaN/undefined:** Unvoiced regions within a vowel segment (creaky voice, whispering) cause pitch tracking failures. Filter NaN values and compute statistics over remaining valid samples. If >50% are NaN, mark pitch as "indeterminate."
- **Speaker normalization impossible (single utterance):** When you have only one utterance per speaker, use dataset-level statistics or a gender-based prior for normalization instead.
- **GRPO reward is too sparse:** If most generations get 0 reward early in training, increase the group size (e.g., from 8 to 16) to improve the chance of at least one correct candidate per group, or warm-start from a stronger SFT checkpoint.

## Limitations

- Requires audio + transcript pairs — cannot work on text-only or audio-only input without modification.
- Montreal Forced Aligner needs a pronunciation dictionary and acoustic model for each target language. Expanding to low-resource languages requires additional setup.
- Discretization thresholds (z-score bins at +/-0.5) are heuristic. Optimal boundaries may vary across emotional corpora or cultural contexts.
- The natural language prosodic description is lossy — fine-grained pitch contour shapes (e.g., rise-fall patterns within a single vowel) are simplified to "rising"/"falling"/"flat."
- GRPO training requires generating multiple completions per prompt, making it significantly more compute-intensive than SFT alone.
- Performance depends on alignment quality; spontaneous speech with disfluencies, overlapping speakers, or heavy background noise degrades vowel boundary accuracy.

## Reference

**Paper:** [VowelPrompt: Hearing Speech Emotions from Text via Vowel-level Prosodic Augmentation](https://arxiv.org/abs/2602.06270v1) (ICLR 2026)
**Key insight:** Vowels carry the bulk of emotional prosody in speech; extracting per-vowel pitch, energy, and duration descriptors and converting them to natural language lets LLMs reason jointly over what was said and how it was said, outperforming both audio-only and text-only approaches across domains and languages.