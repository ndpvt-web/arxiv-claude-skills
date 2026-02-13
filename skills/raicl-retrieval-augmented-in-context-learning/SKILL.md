---
name: "raicl-retrieval-augmented-in-context-learning"
description: "Build retrieval-augmented in-context learning (RAICL) pipelines that convert time-series or signal data into images and classify them with vision-language models using dynamically retrieved few-shot examples. Trigger phrases: 'classify EEG with a VLM', 'retrieval-augmented in-context learning', 'convert signals to images for GPT/Gemini', 'few-shot VLM classification', 'RAICL pipeline', 'time-series image classification with LLM'"
---

# Retrieval-Augmented In-Context Learning (RAICL) for Signal Classification via Vision-Language Models

This skill enables Claude to build pipelines that classify physiological signals (EEG, ECG, EMG) or other multivariate time-series data by converting them into stacked waveform images and feeding them to vision-language models (VLMs) with dynamically retrieved few-shot examples. Instead of training task-specific neural networks, the RAICL approach renders signals as color-coded waveform plots, retrieves the most representative and morphologically similar labeled examples from a reference pool, constructs multimodal prompts with domain-expert chain-of-thought reasoning, and queries off-the-shelf VLMs for zero-retraining classification.

## When to Use

- When the user wants to classify EEG, ECG, EMG, or other physiological signals using a VLM (Gemini, GPT-4o, Qwen-VL, InternVL) instead of training a dedicated model
- When the user asks to build a few-shot classification system where labeled examples are selected dynamically per query rather than fixed
- When the user needs to convert multivariate time-series data into images suitable for vision model input
- When the user wants to leverage domain expertise (e.g., neuroscience diagnostic criteria) as structured text prompts alongside visual inputs
- When labeled training data is scarce and the user wants to maximize classification performance with only a handful of examples per class
- When the user asks about retrieval-augmented in-context learning for any image classification task where query-specific example selection matters

## Key Technique

**Signal-to-Image Conversion with Chromatic Encoding.** RAICL converts multivariate signals X of shape (C channels x T timesteps) into stacked waveform images. Each channel is vertically offset using `y(c,t) = alpha * Normalize(x(c,t)) + delta * c`, where `alpha` controls amplitude scaling and `delta` provides vertical separation. Each channel receives a distinct RGB color to prevent visual confusion when waveforms cross. Lines are rendered with thickened strokes onto a clean canvas (no grid lines, no margins, no axis ticks) to preserve morphological detail at the target resolution of 224x224 pixels. This visual encoding preserves temporal synchrony across channels -- a VLM can scan vertical slices to identify cross-channel events like seizure discharges.

**Two-Stage Retrieval for Few-Shot Selection.** The core innovation is how few-shot examples are chosen. Static random selection fails because EEG and similar signals are highly non-stationary -- a patient's baseline brain activity shifts with mental state, artifacts, and time. RAICL uses a two-stage retrieval strategy: (1) **Representativeness selection** finds non-task anchors by computing the centroid of a subject's resting-state embeddings and selecting the M examples closest to that centroid (via cosine distance on CLIP embeddings). This establishes a subject-specific baseline that minimizes distributional shift. (2) **Similarity selection** finds task-class examples by computing medoids of each class's embedding cluster from an auxiliary pool, then selecting the M medoids with lowest cosine distance to the test query. The result is a support set where non-task anchors are prototypical for the specific subject and task exemplars are morphologically similar to the query. This two-stage approach yields over 10% balanced classification accuracy gains versus random selection.

**Domain-Expert Prompting with Chain-of-Thought.** The text prompt integrates diagnostic criteria (e.g., "seizure patterns exhibit rhythmic spike-and-wave discharges at 3 Hz with high amplitude across multiple channels") and enforces a structured analysis protocol: first describe what you see in each channel, then reason about temporal patterns, then conclude with a classification. This separation of perception from decision-making improves accuracy and produces explainable outputs.

## Step-by-Step Workflow

1. **Preprocess raw signals into fixed-length segments.** Load the multivariate time-series data, apply bandpass filtering appropriate to the domain (e.g., 0.5-50 Hz for EEG), segment into fixed windows (e.g., 10-second epochs), and normalize each channel independently using robust scaling (median subtraction, IQR division) to handle outliers and artifacts.

2. **Render each segment as a stacked waveform image.** For each segment, plot all C channels on a single image canvas. Vertically offset channels with consistent spacing (`delta`). Assign each channel a distinct saturated color from a perceptually uniform colormap (e.g., tab20). Use thickened lines (linewidth >= 1.5), remove all axes, grids, titles, and margins. Export at 224x224 pixels (or the VLM's native resolution) as PNG.

3. **Extract visual embeddings for all rendered images.** Pass each waveform image through a CLIP visual encoder (e.g., `openai/clip-vit-large-patch14`) to obtain a fixed-dimensional embedding vector. Store these embeddings alongside their labels and subject/session metadata in an indexed retrieval store (e.g., FAISS or a simple numpy array with cosine distance search).

4. **Build the non-task anchor pool per subject.** For each test subject, collect embeddings from their non-task (resting-state or baseline) segments. Compute the centroid of these embeddings. Select the top M examples (M=2 recommended) closest to this centroid by cosine distance. These become the subject-specific non-task anchors.

5. **Build the task-class exemplar pool from auxiliary subjects.** From labeled data of other subjects (not the test subject), compute class-wise centroids for each task class. Identify the medoid of each class (the real example closest to the class centroid). At query time, rank medoids by cosine similarity to the test query embedding and select the top M per class.

6. **Assemble the support set.** Combine the non-task anchors (step 4) with the task-class exemplars (step 5) into a support set S = {(image_1, label_1), ..., (image_K, label_K)}. Order them consistently: non-task examples first, then task examples grouped by class.

7. **Construct the multimodal prompt.** Build a prompt with three components: (a) a system message containing domain-specific diagnostic criteria and the analysis protocol, (b) the support set as interleaved image-text pairs where each image is followed by "Classification: [label]", and (c) the test query image followed by the analysis protocol instructions requesting step-by-step reasoning before a final classification.

8. **Query the VLM with temperature=0.** Send the assembled prompt to the VLM API (Gemini, GPT-4o, Qwen-VL, etc.) with temperature set to 0 for deterministic output. Parse the structured response to extract both the chain-of-thought reasoning and the final classification label.

9. **Post-process and aggregate predictions.** If multiple segments belong to the same recording session, aggregate segment-level predictions via majority voting or probability averaging. Log the chain-of-thought outputs for clinical review and explainability.

10. **Evaluate with balanced classification accuracy (BCA).** Compute per-class recall and average them to get BCA, which handles class imbalance common in medical signal datasets. Compare against random few-shot selection baseline to quantify retrieval benefit.

## Concrete Examples

**Example 1: EEG Seizure Detection Pipeline**

User: "I have EEG recordings from the TUH Seizure Corpus. I want to classify 10-second epochs as seizure vs. non-seizure using Gemini Flash, without training any model. Can you build a RAICL pipeline?"

Approach:
1. Load EEG data using MNE-Python, apply 0.5-50 Hz bandpass filter, segment into 10s epochs
2. Render each epoch as a stacked waveform image with 19 channels (10-20 system), each in a distinct color
3. Extract CLIP embeddings for all epoch images
4. For each test subject, select 2 non-task anchors from their baseline segments (closest to resting-state centroid)
5. From other subjects' labeled data, retrieve 2 seizure medoids and 2 non-seizure medoids most similar to the test query
6. Build prompt with neuroscience diagnostic criteria and 6-shot support set
7. Query Gemini with temperature=0, parse classification

```python
import mne
import numpy as np
import matplotlib.pyplot as plt
from PIL import Image
from transformers import CLIPModel, CLIPProcessor
import faiss

# Step 1: Load and preprocess EEG
raw = mne.io.read_raw_edf("patient_001.edf", preload=True)
raw.filter(0.5, 50.0)
epochs = mne.make_fixed_length_epochs(raw, duration=10.0, overlap=0.0)
data = epochs.get_data()  # shape: (n_epochs, n_channels, n_samples)

# Step 2: Render stacked waveform images
COLORS = plt.cm.tab20(np.linspace(0, 1, data.shape[1]))

def render_waveform(segment, colors, output_path, img_size=224):
    """Convert a (C, T) array into a stacked waveform PNG."""
    fig, ax = plt.subplots(figsize=(3, 3), dpi=img_size // 3)
    n_channels = segment.shape[0]
    for c in range(n_channels):
        # Robust normalize per channel
        median = np.median(segment[c])
        iqr = np.percentile(segment[c], 75) - np.percentile(segment[c], 25)
        normed = (segment[c] - median) / (iqr + 1e-8)
        offset = c * 2.5  # vertical spacing
        ax.plot(normed + offset, color=colors[c], linewidth=1.5)
    ax.axis("off")
    plt.subplots_adjust(left=0, right=1, top=1, bottom=0)
    fig.savefig(output_path, bbox_inches="tight", pad_inches=0, dpi=img_size // 3)
    plt.close(fig)

for i, seg in enumerate(data):
    render_waveform(seg, COLORS, f"waveforms/epoch_{i:04d}.png")

# Step 3: Extract CLIP embeddings
clip_model = CLIPModel.from_pretrained("openai/clip-vit-large-patch14")
clip_proc = CLIPProcessor.from_pretrained("openai/clip-vit-large-patch14")

def get_embedding(img_path):
    img = Image.open(img_path).convert("RGB")
    inputs = clip_proc(images=img, return_tensors="pt")
    return clip_model.get_image_features(**inputs).detach().numpy().flatten()

embeddings = np.array([get_embedding(f"waveforms/epoch_{i:04d}.png") for i in range(len(data))])
embeddings /= np.linalg.norm(embeddings, axis=1, keepdims=True)

# Step 4: Select non-task anchors (resting-state epochs for this subject)
baseline_mask = labels == "baseline"  # boolean mask for non-task segments
baseline_embs = embeddings[baseline_mask]
centroid = baseline_embs.mean(axis=0, keepdims=True)
centroid /= np.linalg.norm(centroid)
dists = 1 - (baseline_embs @ centroid.T).flatten()
anchor_indices = np.argsort(dists)[:2]  # M=2 closest to centroid

# Step 5: Retrieve task-class medoids from auxiliary pool
# (aux_embeddings, aux_labels loaded from other subjects)
def get_medoids(embs, labs, query_emb, class_label, M=2):
    class_mask = labs == class_label
    class_embs = embs[class_mask]
    class_centroid = class_embs.mean(axis=0, keepdims=True)
    class_centroid /= np.linalg.norm(class_centroid)
    # Find medoid (closest to class centroid)
    inner_dists = 1 - (class_embs @ class_centroid.T).flatten()
    medoid_order = np.argsort(inner_dists)
    # Among top medoids, rank by similarity to query
    top_medoids = class_embs[medoid_order[:10]]
    query_sims = (top_medoids @ query_emb.reshape(-1, 1)).flatten()
    return medoid_order[np.argsort(-query_sims)[:M]]
```

```
# Step 7: Prompt structure sent to Gemini
SYSTEM_PROMPT = """You are a board-certified clinical neurophysiologist analyzing EEG waveform images.

Diagnostic Criteria:
- SEIZURE: Rhythmic spike-and-wave discharges, evolving frequency (typically 3-8 Hz),
  high amplitude (>100 uV equivalent), spreading across multiple channels, sustained >10 seconds.
- NON-SEIZURE: Irregular mixed-frequency activity, no evolving rhythmic patterns,
  amplitude varies normally across channels, may contain isolated artifacts.

Analysis Protocol:
1. Describe the dominant frequency pattern visible in the waveforms.
2. Note any rhythmic or evolving discharges and which channels are involved.
3. Assess amplitude: are there sudden high-amplitude bursts across channels?
4. State your classification: SEIZURE or NON-SEIZURE.
5. Confidence: HIGH, MEDIUM, or LOW."""

# Few-shot examples are interleaved image + label pairs
# Then the test query image is appended with:
QUERY_SUFFIX = "Analyze this EEG epoch following the protocol above. Provide your reasoning, then your classification."
```

**Example 2: ECG Arrhythmia Classification**

User: "I want to classify single-lead ECG strips as normal sinus rhythm vs. atrial fibrillation using GPT-4o with retrieved examples."

Approach:
1. Segment ECG into 10-second windows, bandpass filter 0.5-40 Hz
2. Render each window as a single-channel waveform (black line on white, thick stroke, no axes)
3. Extract CLIP embeddings, build FAISS index over labeled pool
4. For each query, retrieve 2 normal and 2 AFib examples closest to query embedding
5. Construct prompt with cardiology diagnostic criteria (irregular R-R intervals, absent P-waves for AFib)
6. Query GPT-4o, parse response

```python
# Adapted for single-lead ECG -- same RAICL pattern, different domain prompts
CARDIOLOGY_PROMPT = """You are a cardiologist analyzing ECG rhythm strips.

Diagnostic Criteria:
- NORMAL SINUS RHYTHM: Regular R-R intervals, consistent P-wave before each QRS,
  rate 60-100 bpm, narrow QRS complexes.
- ATRIAL FIBRILLATION: Irregularly irregular R-R intervals, absent or chaotic P-waves,
  fibrillatory baseline between QRS complexes.

Analysis Protocol:
1. Assess R-R interval regularity.
2. Identify P-waves: present, absent, or replaced by fibrillatory waves?
3. Measure approximate heart rate from R-R spacing.
4. Classification: NORMAL or AFIB.
5. Confidence: HIGH, MEDIUM, or LOW."""
```

**Example 3: Industrial Vibration Anomaly Detection**

User: "I have accelerometer data from factory equipment. Can I use a VLM to detect anomalies without training a model?"

Approach:
1. Segment 3-axis accelerometer data into fixed windows
2. Render as 3-channel stacked waveform (red=X, green=Y, blue=Z)
3. Use RAICL retrieval: non-task anchors from known-normal operation of the same machine, task exemplars from labeled anomaly library of similar machines
4. Prompt with vibration analysis expertise (bearing fault signatures, imbalance patterns)

This demonstrates RAICL generalizes beyond biomedical signals to any domain where time-series patterns have visual signatures that domain experts can articulate as diagnostic criteria.

## Best Practices

- **Do:** Use robust scaling (median/IQR) per channel before rendering -- it handles outliers and artifacts far better than min-max or z-score normalization.
- **Do:** Assign perceptually distinct colors to channels. Use a colormap like `tab20` rather than sequential colormaps where adjacent channels may look identical.
- **Do:** Remove all non-data visual elements (axes, grids, titles, tick marks, margins) from waveform images. These waste pixels and confuse the VLM.
- **Do:** Use CLIP embeddings specifically for the retrieval step -- they capture visual similarity in the same space the VLM processes, yielding better matches than signal-domain features (DTW, spectral distance).
- **Avoid:** Using ViT-only architectures for this task. Research shows highly global attention mechanisms underperform on stacked waveforms -- CNN-based visual encoders or hybrid architectures work better.
- **Avoid:** Selecting few-shot examples randomly. The retrieval step is not optional -- random selection degrades balanced accuracy by 10%+ compared to RAICL retrieval.
- **Avoid:** Cramming too many channels into one image. If you have 64+ channels, split into logical groups (e.g., frontal, temporal, parietal) and render separate images per group, then query each or tile them into a grid.

## Error Handling

- **VLM refuses to classify medical data:** Some VLMs have safety filters for medical content. Frame the prompt as "analyzing a waveform plot" rather than "diagnosing a patient." Emphasize this is a research/educational context, not clinical decision-making.
- **Low retrieval quality (all examples look similar):** This occurs when the embedding space doesn't capture task-relevant features. Try fine-tuning CLIP on domain-specific image pairs, or switch to a domain-adapted encoder. As a fallback, use signal-domain features (power spectral density, wavelet coefficients) for retrieval while keeping images for the VLM.
- **VLM outputs unparseable responses:** Enforce strict output formatting in the prompt (e.g., "Your final line must be exactly: CLASSIFICATION: [LABEL]"). Implement regex-based extraction with a fallback that re-queries with a simplified prompt asking only for the label.
- **Class imbalance in retrieval pool:** Ensure equal numbers of examples per class in the support set. If one class has far fewer labeled examples, oversample its medoids or use data augmentation (time-shifting, amplitude scaling) before rendering.
- **High API costs from large image payloads:** Downscale images to the VLM's native resolution (224x224 for most). Reduce the number of few-shot examples from M=2 to M=1 per class if budget is tight -- representativeness selection alone still outperforms random.

## Limitations

- **Latency and cost:** Each classification requires a VLM API call with multiple images. This is not suitable for real-time monitoring (e.g., live seizure detection) but works well for batch retrospective analysis.
- **Ceiling on spatial resolution:** Stacking many channels (>20) onto a 224x224 image compresses each waveform into very few vertical pixels, losing fine morphological detail. For high-density arrays (64-256 channels), spatial decomposition or channel selection is necessary.
- **No learned feature extraction:** The approach relies entirely on the VLM's pretrained visual features. Domain-specific patterns that don't resemble anything in the VLM's training data (e.g., highly specialized biomarkers) may not be recognized.
- **Prompt sensitivity:** Classification accuracy is sensitive to the quality of diagnostic criteria in the prompt. Poorly written domain descriptions degrade performance significantly. Always have a domain expert review the prompt text.
- **Subject-specific baseline required:** The representativeness selection step requires non-task data from each test subject. If no baseline data is available for a subject, fall back to population-level centroids (expect a modest accuracy drop).

## Reference

**Paper:** [RAICL: Retrieval-Augmented In-Context Learning for Vision-Language-Model Based EEG Seizure Detection](https://arxiv.org/abs/2601.17844v1) (Li et al., 2026). Key sections: Section III-B for the two-stage retrieval formulation (representativeness + similarity), Section III-A for the waveform rendering specification, and Table II for performance benchmarks showing RAICL matching or exceeding trained signal-processing baselines.