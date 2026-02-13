---
name: "d-orca-dialogue-centric-optimization-robust"
description: "Build dialogue-centric audio-visual captioning pipelines that identify who spoke what and when in multi-party video conversations. Uses GRPO with speaker attribution, speech content, and temporal boundary reward functions. Triggers: 'caption dialogue in video', 'speaker diarization captioning', 'who said what in this video', 'multi-party dialogue transcription', 'audio-visual dialogue pipeline', 'build video dialogue captioning system'"
---

# D-ORCA: Dialogue-Centric Optimization for Robust Audio-Visual Captioning

This skill enables Claude to design and implement audio-visual captioning systems that jointly solve three intertwined problems: identifying *who* is speaking (speaker diarization), recognizing *what* they said (ASR), and pinpointing *when* they said it (temporal grounding). The core technique from D-ORCA is using Group Relative Policy Optimization (GRPO) with three domain-specific reward functions -- derived from speech processing evaluation metrics -- to train or fine-tune multimodal LLMs that produce structured dialogue captions from video input.

## When to Use This Skill

- When the user wants to build a pipeline that generates structured dialogue transcripts from multi-party video (e.g., meetings, talk shows, debates, film scenes)
- When the user needs to jointly optimize speaker attribution, speech recognition accuracy, and temporal alignment rather than treating them as independent pipeline stages
- When the user asks to fine-tune a multimodal LLM (like Qwen-VL) for dialogue-aware video captioning using reinforcement learning
- When the user wants to implement GRPO reward functions for audio-visual tasks, specifically cpWER-based, WER-based, or collar-based temporal rewards
- When the user needs to curate or process a multi-party dialogue video dataset with speaker labels, timestamps, and bilingual transcripts
- When the user asks to evaluate dialogue captioning quality using speaker-attributed metrics (cpWER, DER, WER with temporal IoU)

## Key Technique

**The joint optimization insight.** Traditional video captioning pipelines run speaker diarization, ASR, and timestamp extraction as separate stages, causing errors to cascade -- a diarization mistake corrupts the ASR output, which then misaligns timestamps. D-ORCA eliminates this cascade by training a single omni-modal LLM end-to-end, using reinforcement learning rewards that simultaneously assess all three dimensions. The model (built on Qwen3-VL-8B) processes raw audio and video frames together, producing structured captions in the format `[timestamp_start - timestamp_end] Speaker_ID: "Utterance text"`.

**GRPO with speech-processing rewards.** Rather than standard RLHF with a learned reward model, D-ORCA uses Group Relative Policy Optimization with three analytically computed reward signals: (1) **Speaker Attribution Reward** based on cpWER (concatenated minimum-permutation word error rate), which measures whether utterances are assigned to the correct speakers by finding the optimal permutation of speaker labels; (2) **Speech Content Reward** based on WER (word error rate), measuring transcription accuracy independent of speaker assignment; (3) **Temporal Boundary Reward** using collar-based evaluation that allows a tolerance window (typically 0.25s) around ground-truth segment boundaries, scored via temporal IoU between predicted and reference segments. These three rewards are combined during GRPO rollouts to produce a unified training signal.

**The DVD dataset strategy.** To provide high-quality training data, D-ORCA curates the DVD (Dialogue Video Dataset): ~40,000 multi-party dialogue videos with bilingual (English/Mandarin) annotations. Each sample includes per-utterance speaker IDs, verbatim transcripts, and precise start/end timestamps. The curation pipeline filters for videos with 2+ speakers, runs forced alignment for timestamp annotation, and applies cross-validation between ASR engines for transcript quality.

## Step-by-Step Workflow

1. **Define the structured output format.** Establish a consistent caption schema: `[HH:MM:SS.ms - HH:MM:SS.ms] Speaker_N: "Utterance text"`. Each line represents one utterance with temporal boundaries, speaker identity, and verbatim content. This format is the prediction target for both training and inference.

2. **Prepare the multimodal input pipeline.** Extract video frames at a fixed rate (e.g., 1-2 fps for visual speaker cues) and audio as mel-spectrograms or raw waveforms. Feed both modalities into the model's encoder -- visual features capture face/lip movements and on-screen identity cues, audio features capture speech content and speaker voice characteristics.

3. **Implement the speaker attribution reward (R_speaker).** Compute cpWER between predicted and reference captions: (a) concatenate all utterances per predicted speaker label, (b) find the optimal permutation mapping predicted speaker IDs to ground-truth IDs that minimizes total word error rate, (c) compute WER under this optimal assignment. The reward is `R_speaker = 1.0 - cpWER`. Use the `meeteval` library or implement the Hungarian algorithm for optimal permutation.

4. **Implement the speech content reward (R_content).** Compute standard WER between all predicted text (ignoring speaker labels) and all reference text in temporal order. Normalize by reference length. The reward is `R_content = 1.0 - WER`. Use `jiwer` or equivalent for WER computation. This reward ensures transcription accuracy independent of who said it.

5. **Implement the temporal boundary reward (R_temporal).** For each predicted utterance segment, compute temporal IoU against the best-matching reference segment (matched by content similarity). Apply a collar tolerance of 0.25 seconds on both start and end boundaries before penalizing. The reward is the mean IoU across all matched segment pairs: `R_temporal = mean(IoU_i)`.

6. **Combine rewards for GRPO training.** During each rollout, generate K candidate captions per input video (K=4-8 typical). Score each candidate with the combined reward `R = w1 * R_speaker + w2 * R_content + w3 * R_temporal` (start with equal weights w1=w2=w3=1/3, then tune). Compute group-relative advantages by normalizing rewards within each rollout group, then update the policy using the GRPO objective (clipped surrogate loss with KL penalty).

7. **Build the SFT warmup stage.** Before GRPO, supervised fine-tune the base multimodal LLM on your dialogue caption dataset using standard cross-entropy loss. This gives the model a reasonable starting policy so GRPO rollouts produce parseable outputs from the start. Train for 1-2 epochs on the formatted caption data.

8. **Curate or adapt training data.** For each training video, produce annotations with: (a) unique speaker IDs consistent within the video, (b) verbatim transcripts per utterance, (c) start/end timestamps in seconds. If starting from raw video, use a pipeline of pyannote for diarization, Whisper for ASR, and forced alignment (e.g., CTC segmentation) for timestamps, then manually verify a subset.

9. **Evaluate with the full metric suite.** On held-out data, report: cpWER (speaker-attributed transcription), standard WER (transcription only), DER (diarization error rate for speaker segmentation), and temporal IoU (boundary accuracy). Compare against cascaded baselines (separate diarization + ASR + alignment) to demonstrate the benefit of joint optimization.

10. **Deploy with structured parsing.** At inference, generate captions with the trained model, then parse the structured output into a machine-readable format (JSON array of `{speaker, text, start, end}` objects) for downstream consumption by search, summarization, or subtitle systems.

## Concrete Examples

**Example 1: Building a meeting transcription captioner**

```
User: I want to build a system that watches meeting recordings and produces
transcripts showing who said what with timestamps. The output should be
structured so I can search by speaker.

Approach:
1. Use Qwen2.5-VL or similar multimodal LLM as the base model
2. Prepare meeting data in D-ORCA format:
   [00:00:02.1 - 00:00:05.8] Speaker_1: "Let's start with the Q3 results"
   [00:00:06.2 - 00:00:11.4] Speaker_2: "Revenue was up twelve percent"
   [00:00:11.8 - 00:00:14.1] Speaker_1: "And what about margins?"
3. SFT the model on formatted meeting transcripts (1-2 epochs)
4. Implement three reward functions:
   - R_speaker: cpWER using meeteval library
   - R_content: WER using jiwer
   - R_temporal: collar-based IoU (0.25s tolerance)
5. Run GRPO with K=4 rollouts per sample, combined reward
6. Parse output into JSON for searchable storage

Output format:
[
  {"speaker": "Speaker_1", "text": "Let's start with the Q3 results",
   "start": 2.1, "end": 5.8},
  {"speaker": "Speaker_2", "text": "Revenue was up twelve percent",
   "start": 6.2, "end": 11.4},
  {"speaker": "Speaker_1", "text": "And what about margins?",
   "start": 11.8, "end": 14.1}
]
```

**Example 2: Implementing the cpWER reward function**

```
User: How do I implement the speaker attribution reward from D-ORCA?

Approach:
1. Parse predicted captions into per-speaker utterance groups
2. Parse reference captions into per-speaker utterance groups
3. Find optimal speaker permutation via Hungarian algorithm
4. Compute WER under the best permutation

Implementation (Python):
```python
from itertools import permutations
from jiwer import wer
import numpy as np
from scipy.optimize import linear_sum_assignment

def compute_cpwer_reward(pred_captions, ref_captions):
    """
    pred_captions: list of {"speaker": str, "text": str}
    ref_captions: list of {"speaker": str, "text": str}
    Returns: reward in [0, 1] where 1 = perfect attribution
    """
    # Group utterances by speaker
    pred_by_spk = {}
    for cap in pred_captions:
        pred_by_spk.setdefault(cap["speaker"], []).append(cap["text"])
    ref_by_spk = {}
    for cap in ref_captions:
        ref_by_spk.setdefault(cap["speaker"], []).append(cap["text"])

    pred_spks = list(pred_by_spk.keys())
    ref_spks = list(ref_by_spk.keys())

    # Build cost matrix: WER for each (pred_speaker, ref_speaker) pair
    n = max(len(pred_spks), len(ref_spks))
    cost = np.ones((n, n))  # default high cost
    for i, ps in enumerate(pred_spks):
        pred_text = " ".join(pred_by_spk[ps])
        for j, rs in enumerate(ref_spks):
            ref_text = " ".join(ref_by_spk[rs])
            cost[i][j] = wer(ref_text, pred_text)

    # Hungarian algorithm for optimal assignment
    row_ind, col_ind = linear_sum_assignment(cost)
    optimal_wer = cost[row_ind, col_ind].mean()

    return max(0.0, 1.0 - optimal_wer)
```

**Example 3: Temporal boundary reward with collar tolerance**

```
User: Implement the temporal IoU reward with collar-based tolerance for
evaluating timestamp accuracy in dialogue captions.

Implementation:
```python
def temporal_iou_with_collar(pred_start, pred_end, ref_start, ref_end,
                              collar=0.25):
    """
    Compute IoU between predicted and reference segments,
    applying collar tolerance to reference boundaries.
    collar: tolerance in seconds applied to both start and end.
    """
    # Expand reference boundaries by collar
    ref_start_adj = ref_start - collar
    ref_end_adj = ref_end + collar

    # Compute intersection
    inter_start = max(pred_start, ref_start_adj)
    inter_end = min(pred_end, ref_end_adj)
    intersection = max(0.0, inter_end - inter_start)

    # Compute union
    union_start = min(pred_start, ref_start_adj)
    union_end = max(pred_end, ref_end_adj)
    union = union_end - union_start

    if union <= 0:
        return 0.0
    return intersection / union


def temporal_boundary_reward(pred_segments, ref_segments):
    """
    pred_segments: list of {"text": str, "start": float, "end": float}
    ref_segments: list of {"text": str, "start": float, "end": float}
    Match by text similarity, then compute mean IoU.
    """
    from difflib import SequenceMatcher
    ious = []
    used_refs = set()

    for pred in pred_segments:
        best_iou, best_j = 0.0, -1
        for j, ref in enumerate(ref_segments):
            if j in used_refs:
                continue
            sim = SequenceMatcher(None, pred["text"], ref["text"]).ratio()
            if sim > 0.5:
                iou = temporal_iou_with_collar(
                    pred["start"], pred["end"],
                    ref["start"], ref["end"])
                if iou > best_iou:
                    best_iou = iou
                    best_j = j
        if best_j >= 0:
            used_refs.add(best_j)
            ious.append(best_iou)

    return sum(ious) / max(len(ref_segments), 1)
```

## Best Practices

- **Do:** Use cpWER (not standard WER) for evaluating speaker attribution -- it finds the optimal speaker label permutation, avoiding penalization for arbitrary label naming differences (e.g., "Speaker_A" vs "Speaker_1")
- **Do:** Apply SFT warmup before GRPO -- the model must produce parseable structured captions before RL rewards can meaningfully guide optimization; skip this and rollouts produce garbage
- **Do:** Set collar tolerance (0.2-0.5s) for temporal rewards -- penalizing sub-second boundary errors is counterproductive because even human annotators disagree at that granularity
- **Do:** Weight rewards based on your application priority -- subtitle generation benefits from higher R_temporal weight; searchable archives benefit from higher R_speaker weight
- **Avoid:** Running speaker diarization, ASR, and timestamp extraction as fully independent pipeline stages -- error cascading between stages is the core problem D-ORCA solves
- **Avoid:** Using more than 8 rollouts per sample in GRPO for this task -- dialogue captions are long sequences, and memory scales linearly with K; 4-6 rollouts balance quality with compute

## Error Handling

- **Unparseable model output:** If the model generates captions that don't match the structured format, assign reward 0.0 for that rollout. During SFT warmup, add format-enforcement examples. At inference, implement a regex-based fallback parser that extracts partial structure.
- **Speaker count mismatch:** When the model predicts a different number of speakers than the reference, the cpWER reward handles this gracefully through the cost matrix padding in the Hungarian algorithm. Log mismatches for data quality review.
- **Missing timestamps:** If the model omits temporal boundaries, set R_temporal = 0 but still compute R_speaker and R_content. This partial reward prevents the model from learning to drop timestamps entirely.
- **Overlapping speaker segments:** In real conversations, speakers overlap. Handle this by allowing overlapping temporal segments in the output format and matching each predicted segment independently during reward computation.
- **Language mixing:** For bilingual content (e.g., code-switching between English and Mandarin), use a language-aware WER computation that handles mixed scripts, or compute WER separately per detected language segment.

## Limitations

- The approach requires a multimodal LLM base model that can process both audio and video -- pure text or vision-only models cannot be adapted with this technique
- GRPO training demands significant compute: each sample requires K forward passes for rollouts, multiplied by long sequence lengths for dialogue captions
- Speaker identification relies heavily on visual cues (faces, lip movements); audio-only scenarios or videos without visible speakers degrade R_speaker significantly
- The method is optimized for dialogue-heavy content; it provides no benefit for videos without speech (nature documentaries, music videos) or single-speaker narration
- Temporal boundary accuracy depends on the base model's audio encoder resolution -- models with coarse audio tokenization (>0.5s per token) cannot achieve fine-grained timestamps regardless of reward tuning
- The DVD dataset and technique are validated primarily on English and Mandarin; performance on other languages depends on the base model's multilingual capability

## Reference

**Paper:** [D-ORCA: Dialogue-Centric Optimization for Robust Audio-Visual Captioning](https://arxiv.org/abs/2602.07960v1) (Tang et al., 2026)
**Code:** [github.com/WeChatCV/D-ORCA](https://github.com/WeChatCV/D-ORCA)
**Key insight:** Look at Section 3 for the three GRPO reward function formulations (cpWER, WER, collar-IoU) and Section 4 for ablations showing that joint optimization outperforms cascaded pipelines by 15-25% on speaker-attributed captioning metrics.