---
name: "sonic-o1-real-world-benchmark-evaluating"
description: "Evaluate multimodal LLMs on audio-video understanding using the SONIC-O1 benchmark framework. Covers three task types: video summarization, multiple-choice QA over video segments, and temporal localization with reasoning. Use when: 'evaluate my video model', 'benchmark audio-video understanding', 'test MLLM on temporal localization', 'assess demographic fairness in video QA', 'run SONIC-O1 evaluation', 'compare model performance on video tasks'."
---

# SONIC-O1: Evaluating Multimodal LLMs on Real-World Audio-Video Understanding

This skill enables Claude to help users set up, run, and interpret evaluations of multimodal large language models (MLLMs) using the SONIC-O1 benchmark. SONIC-O1 tests whether models can truly understand sequential audio-video content across three complementary tasks -- summarization, multiple-choice question answering, and temporal localization -- spanning 13 real-world conversational domains (job interviews, courtroom proceedings, medical consultations, etc.) with 4,958 human-verified annotations and demographic fairness metadata. Claude can guide users through benchmark setup, evaluation script configuration, metric interpretation, custom task design following SONIC-O1's methodology, and fairness auditing of model outputs.

## When to Use

- When a user wants to evaluate a multimodal model's ability to understand video + audio content beyond static images
- When benchmarking closed-source vs. open-source MLLMs on temporal reasoning (e.g., "When does event X happen in the video?")
- When the user needs to assess demographic fairness of a video understanding model across race, gender, and age groups
- When designing a custom audio-video QA evaluation pipeline inspired by SONIC-O1's three-task taxonomy
- When comparing model performance on what-understanding (MCQ) vs. when-understanding (temporal localization)
- When the user wants to generate human-verifiable annotations for video content using a structured multi-stage pipeline
- When analyzing why a model scores well on multiple-choice but poorly on timestamp prediction (the "what-vs-when" gap)

## Key Technique

SONIC-O1's core insight is that evaluating MLLMs on video requires disentangling three distinct capabilities that current benchmarks conflate. **Task 1 (Summarization)** tests whether a model can synthesize narrative information across an entire video (up to 60 minutes), scored via LLM-as-Judge (0-10), ROUGE-L, and cosine similarity. **Task 2 (MCQ)** tests fine-grained comprehension over 3-minute segments with four answer choices plus a critical fifth option -- "Not enough evidence" -- which penalizes models that hallucinate answers. **Task 3 (Temporal Localization)** tests whether a model can identify *when* events occur with precise start/end timestamps and temporal relations (before/during/after), measured by Recall@IoU, mean IoU, and Mean Absolute Error.

The benchmark revealed a critical finding: models can score 96.4% on MCQ accuracy while achieving only 25.4% Recall@0.5 on temporal localization. This "what-vs-when" disconnect means high MCQ scores mask a fundamental inability to ground understanding in time. Open-source models fare even worse, with less than 3% R@0.5 -- a 22.6% gap behind the best closed-source model. This three-task decomposition is the actionable framework: any serious video-understanding evaluation must separately test content comprehension, evidence-grounded reasoning, and temporal grounding.

The demographic fairness dimension adds another layer. SONIC-O1 annotates observable characteristics (race/ethnicity, gender, age) for each video participant and measures per-group performance. Results show systematic degradation for Black and Indigenous participants across most models, with up to 21.4% demographic gaps in temporal localization. This makes SONIC-O1 not just a capability benchmark but a fairness audit tool.

## Step-by-Step Workflow

1. **Install the SONIC-O1 evaluation framework** by cloning the repository and installing dependencies:
   ```bash
   git clone https://github.com/VectorInstitute/sonic-o1.git
   cd sonic-o1/sonic-o1
   pip install -r requirements_venv.txt
   ```

2. **Download the dataset from HuggingFace**, which includes video files, audio extracts, captions, and all three task annotation sets:
   ```bash
   pip install huggingface_hub
   huggingface-cli download vector-institute/sonic-o1 \
     --repo-type dataset --local-dir ./dataset
   ```
   Or load programmatically:
   ```python
   from datasets import load_dataset
   ds_summ = load_dataset("vector-institute/sonic-o1", "task1_summarization")
   ds_mcq = load_dataset("vector-institute/sonic-o1", "task2_mcq")
   ds_temporal = load_dataset("vector-institute/sonic-o1", "task3_temporal_localization")
   ```

3. **Configure the evaluation target** by editing `05_evaluation_inference/configs/*.yaml` to specify the model under test (e.g., VideoLLaMA2, Gemini, GPT-4o, VITA, or a custom model), selecting which tasks to run (t1, t2, t3), and which of the 13 topic domains to include.

4. **Run the evaluation pipeline** against your chosen model:
   ```bash
   cd 05_evaluation_inference
   python run_evaluation.py \
     --model your_model_name \
     --tasks t1,t2,t3 \
     --topics all \
     --dataset-path ../dataset \
     --vqa-path ../vqa
   ```

5. **Collect per-task metrics** from the output: ROUGE-L and Judge-Score for summarization, accuracy percentage for MCQ, and Recall@IoU / mIoU / MAE for temporal localization. Compare these separately -- never average across tasks, as they measure fundamentally different capabilities.

6. **Analyze the what-vs-when gap** by comparing your model's MCQ accuracy against its temporal localization Recall@0.5. A large gap (e.g., >50 percentage points) indicates the model understands content but cannot ground it temporally. This is the most diagnostic signal SONIC-O1 provides.

7. **Run demographic fairness analysis** by slicing results across the annotated demographic groups (race/ethnicity: White, Black, Asian, Hispanic, Indigenous, Arab; gender: Male, Female; age: 18-24, 25-39, 40+). Flag any group with performance more than 5% below the overall mean as a fairness concern.

8. **Examine per-domain performance** across the 13 topics (job interviews, courtroom proceedings, medical consultations, emergency response, etc.) to identify domain-specific weaknesses. Models often struggle with specialized vocabulary or multi-speaker scenarios in domains like courtroom proceedings and emergency response.

9. **Compare against the SONIC-O1 leaderboard** at `https://huggingface.co/spaces/vector-institute/sonic-o1-leaderboard` to contextualize your model's results against published baselines.

10. **Generate a structured evaluation report** with per-task scores, the what-vs-when gap magnitude, demographic disparity matrix, and domain-level breakdown, following the reporting format from the paper's Tables 2-5.

## Concrete Examples

**Example 1: Evaluating a custom video-language model**

User: "I fine-tuned VideoLLaMA2 on medical video data. How do I evaluate it with SONIC-O1?"

Approach:
1. Clone the SONIC-O1 repo and install dependencies
2. Download the benchmark dataset from HuggingFace
3. Add a model adapter in `05_evaluation_inference/` if the custom model interface differs from stock VideoLLaMA2
4. Run evaluation on all three tasks, filtering to the medical consultation domain first for domain-relevant signal
5. Compare against the baseline VideoLLaMA2 scores from the leaderboard

```bash
# Run evaluation on medical domain first
python run_evaluation.py \
  --model videollama2_medical \
  --tasks t1,t2,t3 \
  --topics patient_doctor_consultations \
  --dataset-path ../dataset \
  --vqa-path ../vqa

# Then run on all domains for full benchmark
python run_evaluation.py \
  --model videollama2_medical \
  --tasks t1,t2,t3 \
  --topics all \
  --dataset-path ../dataset \
  --vqa-path ../vqa
```

Output interpretation:
```
Task 1 (Summarization): Judge-Score 6.2/10, ROUGE-L 0.31
Task 2 (MCQ): Accuracy 72.4%
Task 3 (Temporal): R@0.5 = 2.1%, mIoU = 0.08, MAE = 45.2s

What-vs-When Gap: 72.4% MCQ vs 2.1% R@0.5 = 70.3 point gap
--> Model understands content but cannot localize events temporally.
```

**Example 2: Auditing a production model for demographic fairness**

User: "We're deploying a video analysis model for HR interview screening. How do we check if it's biased?"

Approach:
1. Load the SONIC-O1 benchmark, filtering to the `job_interviews` domain
2. Run evaluation across all three tasks
3. Slice results by demographic group using the dataset's built-in annotations
4. Compute per-group deltas and flag disparities

```python
from datasets import load_dataset
import pandas as pd

# Load MCQ task with demographic metadata
ds_mcq = load_dataset("vector-institute/sonic-o1", "task2_mcq")

# After running model inference, analyze results by demographic group
results_df = pd.DataFrame(evaluation_results)

# Compute per-group accuracy
demographic_breakdown = results_df.groupby('race_ethnicity')['correct'].mean()
overall_accuracy = results_df['correct'].mean()

# Flag groups with >5% disparity
for group, acc in demographic_breakdown.items():
    delta = overall_accuracy - acc
    if delta > 0.05:
        print(f"FAIRNESS FLAG: {group} accuracy {acc:.1%} "
              f"is {delta:.1%} below overall {overall_accuracy:.1%}")
```

Output:
```
Overall MCQ Accuracy: 78.2%
White: 81.4%    Asian: 79.1%    Hispanic: 76.8%
Black: 72.3%    Indigenous: 70.5%    Arab: 77.2%

FAIRNESS FLAG: Black accuracy 72.3% is 5.9% below overall 78.2%
FAIRNESS FLAG: Indigenous accuracy 70.5% is 7.7% below overall 78.2%
```

**Example 3: Building a custom evaluation task inspired by SONIC-O1**

User: "I want to create a temporal localization benchmark for surveillance footage. How should I structure it?"

Approach:
1. Follow SONIC-O1's annotation schema for Task 3 -- each instance needs an event description, temporal relation (before/during/after a reference event), and ground-truth start/end timestamps
2. Use the same metrics: Recall@IoU thresholds (0.3, 0.5, 0.7), mean IoU, and MAE in seconds
3. Include the "Not enough evidence" option to test hallucination resistance
4. Add demographic annotations if subjects are identifiable

```json
{
  "video_id": "surv_001",
  "domain": "parking_lot",
  "duration_seconds": 1200,
  "question": "When does the person in the blue jacket enter the frame?",
  "temporal_relation": "before",
  "reference_event": "The delivery truck arrives",
  "ground_truth": {
    "start_timestamp": 142.5,
    "end_timestamp": 148.0
  },
  "options": [
    "0:02:22 - 0:02:28",
    "0:05:10 - 0:05:15",
    "0:08:45 - 0:08:50",
    "0:12:30 - 0:12:35",
    "Not enough evidence"
  ],
  "correct_option_index": 0,
  "rationale": "The person in blue enters at 2:22, which is before the truck at 5:10."
}
```

Evaluation metrics to compute:
```python
def compute_temporal_iou(pred_start, pred_end, gt_start, gt_end):
    intersection = max(0, min(pred_end, gt_end) - max(pred_start, gt_start))
    union = max(pred_end, gt_end) - min(pred_start, gt_start)
    return intersection / union if union > 0 else 0.0

def recall_at_iou(predictions, ground_truths, threshold=0.5):
    hits = sum(1 for p, g in zip(predictions, ground_truths)
               if compute_temporal_iou(p[0], p[1], g[0], g[1]) >= threshold)
    return hits / len(ground_truths)
```

## Best Practices

- **Do:** Always evaluate all three tasks separately. MCQ accuracy alone is misleading -- SONIC-O1 showed a model can score 96% on MCQ and only 25% on temporal localization. Report results per task, never as a single aggregate.
- **Do:** Include the "Not enough evidence" option in MCQ tasks. This is critical for detecting hallucination -- models that never select this option are fabricating answers when the video doesn't contain sufficient information.
- **Do:** Slice results by demographic group even when fairness isn't the primary goal. SONIC-O1 found up to 21.4% demographic gaps that were invisible in aggregate metrics.
- **Do:** Test across video duration categories (short <5min, medium 5-20min, long 20-60min) since model performance often degrades non-linearly with duration.
- **Avoid:** Averaging scores across the three tasks into a single number. Summarization, MCQ, and temporal localization test fundamentally different capabilities, and averaging obscures the what-vs-when gap.
- **Avoid:** Evaluating only on short video clips. SONIC-O1 includes videos up to 60 minutes specifically because real-world audio-video content is long-form. Short-clip benchmarks do not predict long-form performance.

## Error Handling

- **Model fails to produce timestamps:** Some models return natural language descriptions instead of numeric timestamps for Task 3. Implement a timestamp parser that handles formats like "2:30", "2m30s", "150 seconds", and "around the 2-minute mark" with fallback regex extraction.
- **Out-of-memory on long videos:** Videos up to 60 minutes at 1080p exceed many GPU memory limits. Use the evaluation framework's frame sampling configuration to reduce frame rate (e.g., 1 fps instead of 30 fps) while preserving temporal coverage.
- **API rate limits for closed-source models:** When evaluating Gemini or GPT-4o, the evaluation can hit API rate limits on the full 4,958 annotations. Implement exponential backoff and checkpoint results to enable resumption.
- **Missing audio modality:** Some models accept only video frames, not audio. Run evaluations both with and without audio to measure the contribution of the audio signal -- SONIC-O1 showed audio is essential for conversational domains.
- **Inconsistent IoU scores:** If temporal localization IoU is near zero for all predictions, verify that timestamps are in the same unit (seconds vs. milliseconds) and that predictions reference the same temporal origin as ground truth.

## Limitations

- The benchmark covers 13 conversational domains, but these are predominantly English-language, indoor, multi-person scenarios. Performance on non-English content, outdoor settings, or single-speaker formats (lectures, vlogs) may not be predicted by SONIC-O1 scores.
- Temporal localization ground truth was generated through a pipeline involving AI-assisted annotation with human verification. Edge cases in timestamp precision (especially for gradual events without sharp boundaries) may have annotation noise of 1-3 seconds.
- The demographic fairness analysis covers observable characteristics only (race/ethnicity, gender, age). Intersectional analysis (e.g., young Black women) is limited by sample size within each cross-group cell.
- SONIC-O1 evaluates understanding, not generation. It does not test whether models can produce video, generate audio responses, or engage in real-time dialogue about streaming video.
- The benchmark requires downloading ~60 hours of video data. Users with bandwidth or storage constraints should use the HuggingFace streaming API or filter to specific domains.

## Reference

**Paper:** [SONIC-O1: A Real-World Benchmark for Evaluating Multimodal Large Language Models on Audio-Video Understanding](https://arxiv.org/abs/2601.21666v1) -- Look for Tables 2-5 for baseline model scores, the what-vs-when gap analysis in Section 5, and the demographic fairness breakdown in Section 6.

**Resources:**
- Dataset: `https://huggingface.co/datasets/vector-institute/sonic-o1`
- Code: `https://github.com/vectorinstitute/sonic-o1`
- Leaderboard: `https://huggingface.co/spaces/vector-institute/sonic-o1-leaderboard`