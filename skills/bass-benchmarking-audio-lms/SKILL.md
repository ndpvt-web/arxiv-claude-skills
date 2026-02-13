---
name: "bass-benchmarking-audio-lms"
description: "Build evaluation benchmarks for audio language models using the BASS methodology — structured task taxonomies across structural segmentation, lyric transcription, musicological analysis, and artist collaboration. Trigger phrases: 'benchmark audio model', 'evaluate music understanding', 'music LLM evaluation', 'audio reasoning benchmark', 'test music AI capabilities', 'build music benchmark dataset'"
---

# BASS: Benchmarking Audio LMs for Musical Structure and Semantic Reasoning

This skill enables Claude to design, implement, and run structured evaluation benchmarks for audio language models following the BASS framework. BASS decomposes music understanding into four testable categories — structural segmentation, lyric transcription, musicological analysis, and artist collaboration — spanning 12 concrete tasks with task-specific metrics. Use this skill to build benchmark pipelines, write evaluation harnesses, craft prompt templates with paraphrase diversity, and interpret model performance gaps across musical reasoning dimensions.

## When to Use

- When the user wants to evaluate an audio LM's music understanding capabilities systematically
- When building a benchmark dataset for any audio reasoning task (not just music) and needs a rigorous task taxonomy design
- When the user asks to test whether a model can identify song structure, transcribe lyrics, classify genres, or attribute artists from audio
- When comparing multiple multimodal LMs on audio tasks and needing normalized scoring across heterogeneous metrics
- When designing prompt templates for audio-language tasks that need paraphrase diversity to reduce prompt sensitivity
- When the user wants to build a music recommendation or search evaluation pipeline grounded in model reasoning quality

## Key Technique

BASS structures music understanding evaluation around a four-category, twelve-task taxonomy. Rather than treating "music understanding" as a monolithic capability, it factors it into **structural segmentation** (can the model identify verse/chorus/bridge boundaries and labels?), **lyric transcription** (can it extract lyrics section-by-section?), **musicological analysis** (can it identify genres, instruments, and vocal attributes?), and **artist collaboration** (can it count, localize, and attribute individual artists in multi-performer tracks?). Each category uses distinct question formats and metrics suited to its nature — IoU for temporal segmentation, inverted Word Error Rate for transcription, normalized exact match for classification, and tolerance-windowed exact match for timestamp tasks.

The critical design choice is **metric normalization** to enable cross-task comparison. Raw accuracy on a 4-choice MCQ (25% random baseline) is not comparable to IoU on segmentation (near-0% random baseline). BASS normalizes classification accuracy as `(EMA - R) / (1 - R)` where R is random chance, making a 50% score on a 4-choice task equivalent to 33% normalized — removing the inflated floor. This lets you rank models across all 12 tasks on a common scale. The second key insight is **prompt paraphrasing**: each task uses 10 natural-language prompt variants (paraphrases of the same instruction) to measure robustness to phrasing, not just capability on a single template.

The benchmark reveals that current frontier models (Gemini 2.5 Pro scoring 26.5% normalized overall) handle text-adjacent tasks like lyric transcription far better than structural reasoning. This gap — strong linguistic priors but weak musical structure modeling — is the actionable diagnostic: it tells builders where to invest training effort and tells evaluators which tasks actually differentiate models.

## Step-by-Step Workflow

1. **Define your task taxonomy.** Map your evaluation domain into 3-5 orthogonal categories (e.g., structural, semantic, attribution). For music, use BASS's four: structural segmentation, lyric transcription, musicological analysis, artist collaboration. For other audio domains, adapt the pattern — e.g., for podcast understanding: topic segmentation, speaker diarization, content summarization, fact verification.

2. **Design 2-4 concrete tasks per category.** Each task must have a single unambiguous output format. Structural segmentation splits into Full Structural Segmentation (FSS: segment the entire song) and Section Structural Segmentation (SSS: identify instances of one section type). Musicological analysis splits into single-genre detection, pairwise-genre detection, genre attribution, and genre dominance ranking. Keep tasks graduated in difficulty within each category.

3. **Select task-appropriate metrics.** Match the metric to the output type:
   - Temporal segmentation tasks: Intersection over Union (IoU) between predicted and ground-truth time spans
   - Transcription tasks: Inverted Word Error Rate = `1 / (1 + WER)` so higher is better
   - Classification/MCQ tasks: Exact Match Accuracy, normalized as `(EMA - R) / (1 - R)`
   - Timestamp-based tasks: Exact match with a tolerance window (BASS uses +/- 3 seconds)

4. **Curate ground-truth data from authoritative sources.** Use existing expert-annotated datasets rather than generating labels with LLMs. BASS draws structural annotations from the Harmonix Set (professional music analysts), lyrics from Genius (crowd-verified), musicological attributes from MGPHot (professional musicologist 0-5 ratings), and artist collaboration data from CoSoD. Always verify annotation quality on a sample before scaling.

5. **Construct questions with distractor design for MCQ tasks.** Use 4-8 answer options (more options reduce random-chance inflation). Draw distractors from the same domain — for genre classification, use plausible alternative genres from the dataset, not absurd options. For open-ended tasks, define the expected output schema precisely (e.g., "return a JSON array of {section_label, start_time, end_time}").

6. **Write 10 prompt paraphrases per task.** Do not evaluate on a single prompt template. Write 10 natural-language rephrasings of each task instruction. Example for structural segmentation:
   - "Divide this track into its major sections with timestamps."
   - "Identify each structural segment (verse, chorus, bridge, etc.) and when it starts and ends."
   - "List all the sections in this song with their time boundaries."
   This measures prompt robustness and averages out phrasing-specific biases.

7. **Build the evaluation harness.** Write code that: (a) loads audio files or references, (b) randomly selects one of the 10 prompt paraphrases per question, (c) calls the model API with audio + text prompt, (d) parses the model's response into the expected output schema, (e) computes the task-specific metric, and (f) stores raw and normalized scores.

8. **Normalize scores for cross-task comparison.** Apply `(score - random_baseline) / (1 - random_baseline)` to all classification metrics. For IoU and inverted WER, the random baseline is effectively 0, so normalization is a no-op. Report both raw and normalized scores.

9. **Run evaluation across models and aggregate results.** Test each model on all tasks, computing per-task scores, per-category averages, and an overall normalized average. Present results in a matrix (models x tasks) to reveal category-level strengths and weaknesses.

10. **Analyze failure modes by category.** The diagnostic value of BASS-style benchmarks is in the gaps. A model strong on lyric transcription but weak on structural segmentation is leveraging linguistic priors without modeling temporal structure. A model weak on artist collaboration but strong on genre detection lacks voice discrimination. Use these diagnostics to guide model improvement or selection.

## Concrete Examples

**Example 1: Building a music understanding benchmark evaluation harness**

User: "I want to evaluate Gemini and Qwen on music understanding tasks. Help me build a benchmark."

Approach:
1. Define the BASS task taxonomy with the user's target tasks
2. Scaffold the evaluation harness in Python
3. Implement metric computation for each task type

Output:
```python
# bass_evaluator.py
import json
import random
from dataclasses import dataclass
from typing import Literal

@dataclass
class BenchmarkQuestion:
    task: str
    category: str
    audio_path: str
    prompt_variants: list[str]
    ground_truth: dict
    question_type: Literal["open", "mcq"]
    mcq_options: list[str] | None = None

TASK_METRICS = {
    "structural_segmentation": "iou",
    "lyric_transcription": "inverted_wer",
    "musicological_analysis": "normalized_ema",
    "artist_collaboration": "tolerance_match",
}

PROMPT_VARIANTS = {
    "fss": [
        "Divide this track into its major sections. For each, provide the section label, start timestamp, and end timestamp.",
        "Segment the entire song into structural parts (intro, verse, chorus, bridge, outro, etc.) with time boundaries.",
        "Identify all structural sections in this song and their timestamps.",
        # ... 7 more paraphrases
    ],
    "sgd": [
        "Listen to this song. Which of the following genres is most dominant? Options: {options}",
        "What is the primary genre of this track? Choose from: {options}",
        "Of the genres listed below, which best describes this song? {options}",
        # ... 7 more paraphrases
    ],
}

def compute_iou(predicted_segments: list[dict], gt_segments: list[dict]) -> float:
    """IoU between predicted and ground-truth temporal segments with label matching."""
    total_iou, matched = 0.0, 0
    for gt in gt_segments:
        best_iou = 0.0
        for pred in predicted_segments:
            if pred["label"].lower() == gt["label"].lower():
                inter_start = max(pred["start"], gt["start"])
                inter_end = min(pred["end"], gt["end"])
                intersection = max(0, inter_end - inter_start)
                union = (pred["end"] - pred["start"]) + (gt["end"] - gt["start"]) - intersection
                best_iou = max(best_iou, intersection / union if union > 0 else 0)
        total_iou += best_iou
        matched += 1
    return total_iou / matched if matched > 0 else 0.0

def compute_inverted_wer(predicted: str, reference: str) -> float:
    """Inverted WER so higher is better. Uses 1/(1+WER)."""
    # Requires jiwer or similar WER library
    from jiwer import wer
    error_rate = wer(reference, predicted)
    return 1.0 / (1.0 + error_rate)

def normalize_ema(accuracy: float, num_choices: int) -> float:
    """Normalize exact match accuracy by removing random-chance floor."""
    random_chance = 1.0 / num_choices
    if accuracy <= random_chance:
        return 0.0
    return (accuracy - random_chance) / (1.0 - random_chance)

def evaluate_question(question: BenchmarkQuestion, model_response: str) -> float:
    metric = TASK_METRICS[question.category]
    if metric == "iou":
        predicted = parse_segments(model_response)
        return compute_iou(predicted, question.ground_truth["segments"])
    elif metric == "inverted_wer":
        return compute_inverted_wer(model_response, question.ground_truth["lyrics"])
    elif metric == "normalized_ema":
        correct = model_response.strip().lower() == question.ground_truth["answer"].lower()
        return normalize_ema(float(correct), len(question.mcq_options))
    elif metric == "tolerance_match":
        pred_time = parse_timestamp(model_response)
        gt_time = question.ground_truth["timestamp"]
        return 1.0 if abs(pred_time - gt_time) <= 3.0 else 0.0

def run_benchmark(questions: list[BenchmarkQuestion], model_fn) -> dict:
    """Run full benchmark, returning per-task and per-category scores."""
    results = {}
    for q in questions:
        prompt = random.choice(q.prompt_variants)
        response = model_fn(q.audio_path, prompt)
        score = evaluate_question(q, response)
        results.setdefault(q.task, []).append(score)

    task_scores = {task: sum(s) / len(s) for task, s in results.items()}
    return task_scores
```

**Example 2: Designing an evaluation for a custom audio domain (podcast understanding)**

User: "I want to adapt the BASS approach to evaluate models on podcast understanding."

Approach:
1. Map BASS's four-category structure to the podcast domain
2. Define task-specific metrics following BASS's normalization pattern
3. Design prompt paraphrases for each task

Output:
```
Adapted Taxonomy:
Category                    | Tasks                              | Metric
--------------------------- | ---------------------------------- | ----------------------
Structural Segmentation     | Topic boundary detection           | IoU (timestamps)
                            | Segment labeling (intro/ad/topic)  | IoU (timestamps+labels)
Content Transcription       | Full transcript                    | Inverted WER
                            | Section-specific transcript        | Inverted WER
Semantic Analysis           | Topic classification (MCQ)         | Normalized EMA
                            | Sentiment/tone detection (MCQ)     | Normalized EMA
                            | Key claim extraction (open)        | ROUGE-L
Speaker Attribution         | Speaker count                      | Exact match
                            | Speaker diarization                | DER (Diarization Error Rate)
                            | Speaker-to-utterance mapping       | Tolerance match (+-2s)

Prompt paraphrases for topic boundary detection (3 of 10):
1. "Identify where the main topics change in this podcast episode. Return timestamps."
2. "Segment this podcast into distinct topic sections with start and end times."
3. "At what points does the conversation shift to a new subject? List all transitions."

Normalization: MCQ tasks with 5 options use R=0.2. A model scoring 60% raw
  gets (0.6 - 0.2) / (1 - 0.2) = 50% normalized.
```

**Example 3: Analyzing benchmark results to diagnose model weaknesses**

User: "My audio model scores 80% on lyric transcription but 15% on structural segmentation. What does this mean and how do I fix it?"

Approach:
1. Interpret the gap using BASS's framework for linguistic priors vs. structural reasoning
2. Identify likely causes and suggest targeted improvements

Output:
```
Diagnosis (per BASS findings):
- High lyric transcription + low structural segmentation is the most common
  failure pattern across all 14 models BASS evaluated.
- This indicates your model has strong linguistic priors (it can decode speech
  from audio) but weak temporal-structural reasoning (it cannot identify
  when musical sections begin/end or what they are).

Root causes:
1. Training data likely emphasizes speech/text alignment over musical form
2. Model may lack temporal attention mechanisms needed for long-range structure
3. Audio encoder may compress temporal information too aggressively

Targeted improvements:
1. Add structural segmentation examples to fine-tuning data (Harmonix Set
   provides 912 songs with section annotations)
2. Augment training with explicit timestamp prediction tasks — force the
   model to output start/end times, not just labels
3. Consider longer context windows — structural segmentation requires
   reasoning over the full song (avg 4.16 min in BASS), while lyric
   transcription can work on short segments
4. Evaluate with BASS's IoU metric during training as a validation signal
   to track structural reasoning improvement independently
```

## Best Practices

- **Do:** Use 10+ prompt paraphrases per task to measure robustness, not just single-prompt capability. Prompt sensitivity is a real confounder in audio LM evaluation.
- **Do:** Normalize classification metrics by random chance before comparing across tasks. A 50% score on a binary task is not the same as 50% on a 6-way MCQ.
- **Do:** Use task-appropriate metrics — IoU for temporal tasks, inverted WER for transcription, exact match for classification. Do not force accuracy onto everything.
- **Do:** Draw ground truth from expert annotations (musicologists, verified crowd-sourced databases), not from LLM-generated labels.
- **Avoid:** Evaluating on source-separated audio (vocals only, instruments only). BASS found that mixed audio consistently outperforms separated tracks, likely because separation artifacts degrade model input quality.
- **Avoid:** Using fewer than 4 MCQ options. BASS uses 4-8 options specifically to reduce the random-chance floor and make normalized scores more meaningful.

## Error Handling

- **Model returns unstructured text instead of expected format:** Build a robust response parser with fallback regex patterns. For timestamp extraction, try ISO format, "MM:SS", and natural language ("at two minutes thirty seconds"). Log parse failures separately from incorrect answers.
- **Audio file is unavailable or corrupted:** BASS references YouTube links rather than distributing audio directly. If a source is unavailable, skip the question and note the dropout rate. A benchmark with >5% dropout should flag data freshness issues.
- **Model refuses to answer or returns "I cannot process audio":** Score as 0 for that question but track refusal rate separately. A high refusal rate indicates capability gaps, not benchmark problems.
- **Normalization produces negative scores:** This happens when a model scores below random chance (e.g., adversarial failure modes). Clamp normalized scores to 0 — negative normalized scores indicate the model is worse than guessing, which is diagnostically useful but should be reported clearly.

## Limitations

- BASS requires actual audio input, so it cannot evaluate text-only LLMs. Models must accept audio as a modality.
- The benchmark's musicological ground truth depends on Western music theory conventions (verse/chorus structure, genre taxonomies). It may not generalize well to non-Western musical traditions with different structural norms.
- Evaluation of open-ended tasks (structural segmentation, lyric transcription) requires careful output parsing. Models that give correct answers in unexpected formats may be penalized unfairly — invest in robust parsers.
- The best-performing model (Gemini 2.5 Pro) scored only 26.5% normalized overall. This means the benchmark is currently too hard for meaningful model-vs-model differentiation on many tasks — useful for tracking progress over time, but individual task scores may have high variance.
- BASS does not evaluate music generation, only understanding. It tells you whether a model comprehends music, not whether it can produce it.

## Reference

**Paper:** [BASS: Benchmarking Audio LMs for Musical Structure and Semantic Reasoning](https://arxiv.org/abs/2602.04085v1) — Jang et al., 2026. Look for Table 1 (normalized model scores across all 12 tasks), Table 5 (prompt templates), and Section 4.2 (metric normalization formulas). Code and data: `github.com/minjang10/bass_music_benchmark`.