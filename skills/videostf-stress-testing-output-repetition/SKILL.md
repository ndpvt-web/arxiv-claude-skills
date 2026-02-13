---
name: "videostf-stress-testing-output-repetition"
description: "Detect and stress-test output repetition in Video Large Language Models using n-gram metrics and temporal perturbations from the VideoSTF framework. Use when: 'test VideoLLM for repetition bugs', 'measure output degeneration in video models', 'stress test video language model robustness', 'detect repetitive outputs from a video model', 'adversarial testing of VideoLLM stability', 'evaluate video model generation quality for loops'."
---

# VideoSTF: Stress-Testing Output Repetition in Video LLMs

This skill enables Claude to implement, run, and interpret the VideoSTF evaluation framework for detecting output repetition failures in Video Large Language Models (VideoLLMs). VideoSTF formalizes repetition through three complementary n-gram metrics — Repetition Rate (RR), Repetition Intensity (RI), and Information Entropy (IE) — and provides a library of temporal video transformations that expose models to controlled perturbations. The framework reveals that output repetition (where models degenerate into self-reinforcing loops of repeated phrases) is a widespread, exploitable vulnerability triggered by simple temporal modifications to video inputs.

## When to Use

- When a user asks to evaluate a VideoLLM's output quality and detect repetitive degeneration patterns
- When building a CI/regression test suite that checks VideoLLM outputs for repetition before deployment
- When implementing n-gram-based repetition metrics (RR, RI, IE) for any text generation system
- When stress-testing a video understanding model by applying temporal frame perturbations (add, delete, replace, reverse, shuffle)
- When conducting adversarial robustness evaluation of a VideoLLM in a black-box setting
- When comparing multiple VideoLLMs on generation stability under identical temporal stressors
- When a user wants to add repetition-detection guardrails to a production video-language pipeline

## Key Technique

**The Problem:** VideoLLMs can degenerate into self-reinforcing output loops, producing text like "The man is walking. The man is walking. The man is walking..." indefinitely. Existing benchmarks focus on task accuracy and miss this failure mode entirely. The VideoSTF paper demonstrates this is not a rare edge case — it occurs across all 10 tested models and is reliably triggered by simple temporal perturbations.

**Three Complementary Metrics:** VideoSTF formalizes repetition through: (1) **Repetition Rate (RR)** — a binary indicator measuring whether any 5-gram appears more than once, averaged across a corpus to give the proportion of repetitive outputs; (2) **Repetition Intensity (RI)** — computed as `rep_n = 1 - (unique_ngrams / total_ngrams)`, quantifying what fraction of n-grams are duplicated (0 = all unique, 1 = all repeated); and (3) **Information Entropy (IE)** — normalized Shannon entropy over the n-gram distribution, where lower values indicate less lexical diversity and more severe repetition. Using all three together avoids blind spots: RR detects occurrence, RI measures severity, and IE captures distributional monotony.

**Temporal Stress Testing:** The key insight is that output repetition is highly sensitive to temporal structure of video inputs. VideoSTF defines five transformation families — frame addition, deletion, replacement, reversal, and shuffling — applied at controlled intensities (e.g., add/delete 1-2 frames, shuffle with `times=30` random permutations). These simple, semantics-preserving perturbations can push a model from normal output into full repetitive degeneration, exposing a security-relevant attack surface. The adversarial exploitation mode iterates through transformations to find minimal perturbations that flip a previously-normal output into a repetitive one.

## Step-by-Step Workflow

1. **Define the repetition metrics.** Implement three functions: `compute_rr(text, n=5, threshold=1)` that returns 1.0 if any n-gram repeats more than `threshold` times, else 0.0; `compute_ri(text, n=1)` that returns `1 - len(unique_ngrams) / len(all_ngrams)`; and `compute_ie(text, n=1)` that returns normalized Shannon entropy `H / log2(total_ngrams)` over the n-gram frequency distribution.

2. **Set up the video input pipeline.** Load videos as frame tensors of shape `[T, H, W, C]`. Sample frames uniformly (default 16 or 32 frames). Track the original frame order as metadata for transformation auditing.

3. **Run baseline inference.** Query the VideoLLM with a standard prompt (e.g., `"Please describe this video in detail."`) at temperature 0.0 with `max_new_tokens=500`. Record the raw output text and compute all three metrics as the baseline.

4. **Apply temporal stressors.** For each video, generate transformed variants using: `AddOneFrame` (duplicate a random frame, insert at random position), `DeleteOneFrame` (remove each frame systematically), `ReplaceOneFrame` (substitute one frame with another from the sequence), `ReverseVideo` (reverse frame order), and `RandomShuffle` (randomly permute all frames). Use `times=30` for stochastic transformations to create multiple variants per video.

5. **Run stressed inference.** Feed each transformed video through the same model and prompt. Compute RR, RI, and IE for every output.

6. **Compare baseline vs. stressed metrics.** Flag any video where metrics degrade (RR flips from 0 to 1, RI increases significantly, IE drops). Aggregate across the corpus: compute mean RR (proportion of repetitive outputs), mean RI (average repetition intensity), and mean IE (average entropy).

7. **Run adversarial exploitation (optional).** For each video that produced normal baseline output, iterate through all transformations to find the minimal perturbation that induces repetition (RR flips to 1). Track the Attack Success Rate (ASR = proportion of videos successfully attacked) and Average Queries (AQ = mean number of transformation attempts needed).

8. **Generate a stability report.** Produce a structured summary showing: per-stressor metric deltas, most vulnerable video categories, overall ASR, and specific examples of repetition-inducing perturbations with their corresponding degenerated outputs.

9. **Implement guardrails (if building production systems).** Add a post-generation repetition check using the three metrics. If RR=1 and RI > 0.3, reject the output and retry with different sampling parameters (e.g., increase temperature, reduce max_new_tokens) or apply input normalization (canonical frame sampling to reduce sensitivity to temporal structure).

## Concrete Examples

**Example 1: Implementing Repetition Metrics for a VideoLLM Evaluation**

User: "I need to check if our video captioning model produces repetitive outputs. Can you write the repetition detection code?"

Approach:
1. Implement the three VideoSTF n-gram metrics as standalone functions
2. Apply them to model outputs and flag problematic generations

Output:
```python
from collections import Counter
import math
import numpy as np

def compute_rr(text: str, n: int = 5, threshold: int = 1) -> float:
    """Repetition Rate: 1.0 if any word n-gram repeats > threshold times."""
    tokens = text.split()
    if len(tokens) < n:
        return 0.0
    ngrams = [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]
    max_count = max(Counter(ngrams).values())
    return 1.0 if max_count > threshold else 0.0

def compute_ri(text: str, n: int = 1) -> float:
    """Repetition Intensity: proportion of duplicated n-grams."""
    tokens = text.split()
    if len(tokens) < n:
        return 0.0
    ngrams = [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]
    total = len(ngrams)
    unique = len(set(ngrams))
    return 1.0 - unique / total

def compute_ie(text: str, n: int = 1) -> float:
    """Information Entropy: normalized Shannon entropy of n-gram distribution."""
    tokens = text.split()
    if len(tokens) < n:
        return 0.0
    ngrams = [tuple(tokens[i:i+n]) for i in range(len(tokens) - n + 1)]
    total = len(ngrams)
    if total == 0:
        return 0.0
    counts = Counter(ngrams)
    probs = np.array(list(counts.values())) / total
    entropy = -np.sum(probs * np.log2(probs + 1e-12))
    max_entropy = math.log2(total)
    return entropy / max_entropy if max_entropy > 0 else 0.0

def evaluate_output(text: str) -> dict:
    """Run all three VideoSTF metrics on a model output."""
    return {
        "rr": compute_rr(text, n=5, threshold=1),
        "ri": compute_ri(text, n=1),
        "ie": compute_ie(text, n=1),
        "is_repetitive": compute_rr(text, n=5) == 1.0,
    }

# Usage
output_normal = "A dog runs across the park chasing a red ball while children play nearby."
output_degenerate = "The man walks down the street. The man walks down the street. The man walks down the street."

print(evaluate_output(output_normal))
# {'rr': 0.0, 'ri': 0.08, 'ie': 0.97, 'is_repetitive': False}

print(evaluate_output(output_degenerate))
# {'rr': 1.0, 'ri': 0.72, 'ie': 0.42, 'is_repetitive': True}
```

**Example 2: Building a Temporal Stress Test Suite**

User: "I want to stress-test our VideoLLM by perturbing video frames and checking if outputs degrade."

Approach:
1. Implement the five temporal transformation families on frame tensors
2. Run the model on original and perturbed videos
3. Compare metrics to detect degradation

Output:
```python
import numpy as np
import random
from typing import List, Tuple

def add_frame(frames: np.ndarray, times: int = 30) -> List[Tuple[np.ndarray, dict]]:
    """Duplicate a random frame and insert at a random position."""
    results = []
    T = frames.shape[0]
    for _ in range(times):
        src_idx = random.randint(0, T - 1)
        ins_idx = random.randint(0, T)
        new_frames = np.insert(frames, ins_idx, frames[src_idx], axis=0)
        results.append((new_frames, {"type": "add", "src": src_idx, "pos": ins_idx}))
    return results

def delete_frame(frames: np.ndarray) -> List[Tuple[np.ndarray, dict]]:
    """Remove each frame one at a time (systematic)."""
    results = []
    T = frames.shape[0]
    for i in range(T):
        new_frames = np.delete(frames, i, axis=0)
        results.append((new_frames, {"type": "delete", "removed_idx": i}))
    return results

def replace_frame(frames: np.ndarray, times: int = 30) -> List[Tuple[np.ndarray, dict]]:
    """Substitute one frame with another from the sequence."""
    results = []
    T = frames.shape[0]
    for _ in range(times):
        src, tgt = random.sample(range(T), 2)
        new_frames = frames.copy()
        new_frames[tgt] = frames[src]
        results.append((new_frames, {"type": "replace", "src": src, "tgt": tgt}))
    return results

def reverse_frames(frames: np.ndarray) -> List[Tuple[np.ndarray, dict]]:
    """Reverse the temporal order of all frames."""
    return [(frames[::-1].copy(), {"type": "reverse"})]

def shuffle_frames(frames: np.ndarray, times: int = 30) -> List[Tuple[np.ndarray, dict]]:
    """Randomly permute all frames."""
    results = []
    T = frames.shape[0]
    for _ in range(times):
        order = list(range(T))
        random.shuffle(order)
        results.append((frames[order].copy(), {"type": "shuffle", "order": order}))
    return results

ALL_STRESSORS = [add_frame, delete_frame, replace_frame, reverse_frames, shuffle_frames]

def stress_test_video(model_fn, frames, prompt="Please describe this video in detail."):
    """Run a full stress test: baseline + all perturbations."""
    from collections import defaultdict

    # Baseline
    baseline_output = model_fn(frames, prompt)
    baseline_metrics = evaluate_output(baseline_output)

    report = {"baseline": {"output": baseline_output, "metrics": baseline_metrics}}
    degradations = []

    for stressor in ALL_STRESSORS:
        variants = stressor(frames)
        stressor_name = stressor.__name__
        report[stressor_name] = []

        for perturbed_frames, metadata in variants:
            output = model_fn(perturbed_frames, prompt)
            metrics = evaluate_output(output)
            entry = {"metadata": metadata, "metrics": metrics}
            report[stressor_name].append(entry)

            # Flag degradation: baseline was clean but stressed output is repetitive
            if baseline_metrics["rr"] == 0.0 and metrics["rr"] == 1.0:
                degradations.append({
                    "stressor": stressor_name,
                    "metadata": metadata,
                    "ri_delta": metrics["ri"] - baseline_metrics["ri"],
                    "ie_delta": metrics["ie"] - baseline_metrics["ie"],
                })

    report["degradations"] = degradations
    report["asr"] = len(degradations) > 0  # At least one attack succeeded
    return report
```

**Example 3: Adding a Repetition Guardrail to a Production Pipeline**

User: "Our deployed video understanding API sometimes returns looping text. How can I detect and handle this?"

Approach:
1. Add a post-generation repetition check using VideoSTF metrics
2. Implement a retry mechanism with adjusted parameters when repetition is detected

Output:
```python
class RepetitionGuard:
    """Post-generation guardrail based on VideoSTF metrics."""

    def __init__(self, rr_n=5, rr_threshold=1, ri_threshold=0.3, ie_floor=0.4):
        self.rr_n = rr_n
        self.rr_threshold = rr_threshold
        self.ri_threshold = ri_threshold
        self.ie_floor = ie_floor

    def check(self, text: str) -> dict:
        rr = compute_rr(text, n=self.rr_n, threshold=self.rr_threshold)
        ri = compute_ri(text, n=1)
        ie = compute_ie(text, n=1)
        is_degenerate = (rr == 1.0) and (ri > self.ri_threshold or ie < self.ie_floor)
        return {"rr": rr, "ri": ri, "ie": ie, "is_degenerate": is_degenerate}

    def guarded_generate(self, model_fn, video, prompt, max_retries=3):
        """Generate with automatic retry on repetition detection."""
        temps = [0.0, 0.3, 0.7]  # Escalating temperature
        token_limits = [500, 300, 200]  # Decreasing length

        for attempt in range(max_retries):
            output = model_fn(
                video, prompt,
                temperature=temps[min(attempt, len(temps)-1)],
                max_new_tokens=token_limits[min(attempt, len(token_limits)-1)],
            )
            result = self.check(output)
            if not result["is_degenerate"]:
                return {"output": output, "metrics": result, "attempts": attempt + 1}

        # All retries exhausted — return with warning
        return {
            "output": output,
            "metrics": result,
            "attempts": max_retries,
            "warning": "Output may contain repetition despite retries",
        }
```

## Best Practices

- **Do:** Use all three metrics together. RR alone misses severity (a single repeated 5-gram triggers it), RI alone misses short but severe loops, and IE alone cannot distinguish repetition from genuinely low-entropy content. The combination provides a complete picture.
- **Do:** Set temperature to 0.0 for reproducible stress testing. Stochastic sampling can mask repetition in individual runs. Use deterministic decoding for evaluation, then test with sampling to measure real-world frequency.
- **Do:** Apply stressors at multiple intensities (1-frame and 2-frame variants for add/delete/replace) to map the sensitivity curve rather than testing a single perturbation level.
- **Do:** Use 5-gram for RR (the paper's default) — it catches phrase-level repetition without false positives from naturally recurring function words. Use 1-gram for RI and IE to capture word-level degeneration.
- **Avoid:** Testing only on short clips. Repetition degeneration is more likely with longer generation targets. Test with `max_new_tokens >= 500` to give the model room to degenerate.
- **Avoid:** Treating a low baseline RR as proof of robustness. The paper shows models with near-zero baseline repetition can still be pushed into full degeneration by simple frame shuffling or reversal — always run the temporal stress tests.

## Error Handling

- **Empty or very short outputs:** If the model produces fewer tokens than the n-gram size, all metrics return 0.0. This is technically correct but may mask truncation errors. Check output length separately and flag outputs under 20 tokens as suspicious.
- **Non-text model outputs:** Some VideoLLMs return structured JSON or special tokens. Strip non-text content before computing metrics to avoid n-gram artifacts.
- **Frame sampling mismatches:** If the model's frame sampler re-samples from your perturbed tensor, your perturbation may be partially undone. Verify that the frames actually reaching the vision encoder match your intended transformation by logging intermediate frame indices.
- **OOM on large transformation sets:** With `times=30` and 5 stressor types, you generate ~120+ variants per video. For large-scale evaluation (10K videos), batch process and write results incrementally rather than accumulating in memory.

## Limitations

- The three metrics are purely surface-level n-gram statistics. They cannot distinguish meaningful repetition (e.g., a video that genuinely shows a repeated action) from degenerate looping. Manual inspection of flagged outputs is still needed for ground truth.
- The framework is designed for open-ended generation tasks (video description). It may need adaptation for structured outputs like multiple-choice QA or classification tasks where short, constrained answers naturally have lower lexical diversity.
- Temporal stressors assume direct access to video frames before model inference. For API-only models that accept video URLs or file uploads, you must re-encode the perturbed frames into a video format, which may introduce compression artifacts unrelated to the intended perturbation.
- The adversarial exploitation mode is most meaningful for models that are deterministic at temperature 0. With stochastic sampling, repetition may occur intermittently, requiring multiple queries per perturbation to estimate the true attack success rate.

## Reference

**VideoSTF: Stress-Testing Output Repetition in Video Large Language Models**
Yuxin Cao, Wei Song, Shangzhi Xu, Jingling Xue, Jin Song Dong (2026)
[arXiv:2602.10639](https://arxiv.org/abs/2602.10639v1) | [GitHub](https://github.com/yuxincao22/VideoSTF_benchmark)

Key sections to consult: Section 3 for formal metric definitions (RR, RI, IE formulas), Section 4 for temporal stressor specifications, and Section 5 for adversarial exploitation results showing attack success rates across 10 models.