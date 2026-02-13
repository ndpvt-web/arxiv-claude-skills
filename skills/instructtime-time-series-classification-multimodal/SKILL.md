---
name: "instructtime-time-series-classification-multimodal"
description: "Reformulate time series classification as a multimodal generative task using LLMs. Discretizes time series into tokens, extracts statistical and visual implicit features, and constructs structured prompts so a language model generates class labels as text. Use when: 'classify this time series with an LLM', 'use language models for sensor data classification', 'multimodal time series classification', 'convert time series to text for classification', 'generative approach to ECG/EEG/HAR classification', 'InstructTime time series framework'."
---

# InstructTime++: Time Series Classification via Multimodal Language Modeling

This skill enables Claude to help users implement the InstructTime++ framework, which reformulates time series classification as a multimodal generative task. Instead of mapping sequences directly to one-hot labels (the standard discriminative approach), InstructTime++ treats continuous time series, contextual metadata, statistical features, and visual descriptions as multimodal inputs to a language model that generates class labels as natural language text. This approach captures semantic relationships between classes and leverages contextual knowledge that discriminative models discard.

## When to Use

- When the user wants to classify time series data (EEG, ECG, HAR, fault detection, audio) using a language model instead of traditional classifiers like CNNs or transformers
- When the user asks how to convert continuous sensor signals into discrete tokens suitable for LLM input
- When the user needs to enrich time series classification with contextual features (domain descriptions, statistical summaries, visual pattern descriptions)
- When the user wants to build a generative classification pipeline where class labels are produced as text rather than logit vectors
- When the user asks about bridging the modality gap between numerical time series and language model embeddings
- When the user is working on a multi-domain time series benchmark and wants a unified framework that transfers across domains

## Key Technique

**Core Insight: Classification as Text Generation.** Traditional time series classifiers output a probability distribution over fixed class indices. InstructTime++ instead frames classification as conditional text generation: given a structured prompt containing the time series (as discrete tokens), domain instructions, and feature descriptions, the LLM autoregressively generates the class name as a string. This lets the model exploit semantic similarity between class labels (e.g., "walking" and "running" share meaning that indices 3 and 5 do not) and incorporate rich contextual priors.

**Time Series Discretization via Vector Quantization.** Raw continuous signals cannot be fed directly into a language model's vocabulary. InstructTime++ segments the time series into non-overlapping patches using 1D convolution, then maps each patch to its nearest entry in a learned codebook via a Vector-Quantized (VQ) network with a Temporal Convolutional Network (TCN) backbone. The training loss combines reconstruction error, codebook commitment loss, and embedding regularization. The resulting discrete temporal tokens are projected into the LLM's embedding space through a multi-layer perceptron (MLP) alignment layer, ensuring they occupy the same representational space as text tokens.

**Implicit Feature Enhancement.** Language models lack the inductive biases (translation invariance, locality) that specialized time series architectures have. To compensate, InstructTime++ mines two categories of implicit features and translates them to natural language: (1) **Statistical features** -- mean, variance, skewness, kurtosis, sample entropy, approximate entropy, linear trend, and periodicity -- computed directly from raw signals; (2) **Visual features** -- the 1D signal is rendered as a 2D plot and processed by a vision-language model to produce textual descriptions of global trends, local shape patterns, and fine-grained fluctuations. These text descriptions are concatenated into the prompt, giving the LLM explicit access to patterns it would otherwise struggle to infer from token sequences alone.

## Step-by-Step Workflow

1. **Segment the raw time series into patches.** Use 1D convolution with a fixed kernel size (matching the patch length) and stride equal to the kernel size to produce non-overlapping patch embeddings. For a series of length T with patch size P, this yields T/P patches.

2. **Train or load a VQ codebook for discretization.** Implement a VQ-VAE style module: a TCN encoder maps patches to latent vectors, each latent is snapped to its nearest codebook entry (using squared Euclidean distance), and a TCN decoder reconstructs the original patch. Train with: `L = L_recon + beta * L_commit + L_embed_reg`. Use codebook sizes of 256-1024 depending on signal complexity.

3. **Compute statistical implicit features from the raw signal.** For each input time series, extract:
   - Distributional: mean, variance, skewness, kurtosis
   - Complexity: sample entropy, approximate entropy
   - Temporal dynamics: linear trend slope (via linear regression), dominant periodicity (via autocorrelation or FFT)

   Format as natural language: `"The signal has mean 0.32, variance 1.47, skewness -0.21, kurtosis 2.8. Sample entropy is 1.52, indicating moderate complexity. The signal exhibits a slight upward trend (slope 0.003) with a dominant period of approximately 50 timesteps."`

4. **Generate visual implicit features via image captioning.** Render the time series as a line plot (matplotlib or similar), then pass the image to a vision-language model (e.g., LLaVA, Qwen-VL, or GPT-4V) with a prompt like: `"Describe the temporal patterns in this signal, including global trends, local shapes, and notable fluctuations."` Collect the textual description.

5. **Construct the multimodal prompt.** Assemble the following components in order:
   ```
   [Domain Instruction]: "You are classifying {domain} signals. The possible classes are: {class_1}, {class_2}, ..., {class_n}. Based on the following features and signal, output the class name."
   [Explicit Context]: "{any available metadata, e.g., sampling rate, sensor placement}"
   [Statistical Features]: "{text from step 3}"
   [Visual Features]: "{text from step 4}"
   [Temporal Tokens]: "<ts_start> {token_1} {token_2} ... {token_k} <ts_end>"
   [Query]: "The class label for this signal is:"
   ```

6. **Project discrete tokens into the LLM embedding space.** Pass the VQ codebook indices through the MLP alignment layer (hidden sizes: 64 -> 128 -> 256 -> 512 -> 768 for a 768-dim LLM). The projected embeddings replace the `{token_i}` placeholders in the prompt's embedding sequence.

7. **Pre-train with cross-domain autoregressive generation.** Before fine-tuning on target data, pre-train the alignment layer and LLM jointly on multiple time series domains using next-token prediction. This improves cross-modal alignment and domain generalizability. Use learning rate 5e-5, batch size 16, Adam optimizer with weight decay 1e-5, cosine annealing with 5% warmup.

8. **Fine-tune on the target classification task.** Train on the target domain's labeled data with the generative classification objective (cross-entropy on the generated label tokens). Use learning rate 5e-4, batch size 4, AdamW with weight decay 0.01, gradient clipping at norm 1.0, early stopping with patience 10, max 30 epochs, max sequence length 2048.

9. **Run inference via constrained decoding.** At test time, feed the constructed prompt through the model and decode autoregressively. Constrain the output vocabulary to only the valid class names to avoid hallucinated labels. The predicted class is the generated text string.

10. **Evaluate with accuracy and macro F1.** Compare the generated text labels against ground truth. Use both accuracy (for balanced datasets) and macro F1 (for imbalanced ones) as primary metrics.

## Concrete Examples

**Example 1: Human Activity Recognition from Accelerometer Data**

User: "I have a HAR dataset with 6 classes (walking, jogging, upstairs, downstairs, sitting, standing) from smartphone accelerometers. Help me set up the InstructTime++ pipeline."

Approach:
1. Segment each triaxial accelerometer signal (treat each axis independently or concatenate) into patches of size 16 with stride 16
2. Train VQ codebook with 512 entries on the training split; TCN encoder/decoder with 4 layers
3. Compute statistical features per sample:
   ```
   "Mean acceleration: x=0.12g, y=9.78g, z=0.34g. Variance: x=0.85, y=0.22, z=1.03.
    Skewness: x=-0.15, y=0.02, z=0.41. Sample entropy: 1.23 (moderate regularity).
    Dominant period: ~48 samples (~0.96s at 50Hz), consistent with periodic motion."
   ```
4. Plot the signal and caption with VLM:
   ```
   "The signal shows a strongly periodic pattern with sharp peaks every ~1 second,
    consistent amplitude, and minimal drift. Local shape shows asymmetric oscillation
    with faster rises than falls."
   ```
5. Construct prompt:
   ```
   You are classifying human activity from accelerometer signals. The possible classes
   are: walking, jogging, upstairs, downstairs, sitting, standing.

   Statistical features: [from step 3]
   Visual description: [from step 4]
   Signal: <ts_start> tok_42 tok_187 tok_311 ... tok_95 <ts_end>
   The activity class for this signal is:
   ```
6. Model generates: `"walking"`

Output: Classification label "walking" with full prompt trace for interpretability.

**Example 2: ECG Arrhythmia Detection**

User: "I want to classify ECG signals into 5 arrhythmia types using a language model approach."

Approach:
1. Segment single-lead ECG (e.g., 187 timesteps) into patches of size 11, yielding 17 patches
2. Train VQ with 256 codebook entries (ECG morphology is constrained)
3. Extract statistical features:
   ```
   "Mean amplitude: 0.48mV. Variance: 0.31. Kurtosis: 5.2 (leptokurtic, sharp peaks).
    Approximate entropy: 0.89 (relatively regular). Linear trend: -0.001 (stable baseline).
    Dominant period: 72 samples, corresponding to ~80 BPM heart rate."
   ```
4. VLM caption from rendered ECG plot:
   ```
   "The ECG shows regular QRS complexes with narrow morphology. P-waves are visible
    before each QRS. The ST segment appears slightly elevated. No premature beats visible.
    R-R intervals are consistent."
   ```
5. Assemble prompt with domain instruction mentioning ECG-specific classes (Normal, AFib, AFlutter, VTach, SVTach)
6. Fine-tune Qwen3-0.6B (or similar small LLM) for 30 epochs with early stopping

Output: Model generates class name e.g., `"Normal"` or `"AFib"`.

**Example 3: Implementing Just the Feature Enhancement Module**

User: "I already have a time series classifier but want to add InstructTime++'s implicit feature extraction as additional input features."

Approach:
1. Skip VQ discretization -- keep existing model architecture
2. Compute the 8 statistical features per sample and concatenate as a feature vector, or format as text if using a text-aware model
3. Render time series to PNG, run through a VLM to get textual description
4. Encode the textual descriptions using a sentence transformer (e.g., all-MiniLM-L6-v2) to get a fixed-size embedding
5. Concatenate the statistical feature vector and sentence embedding with the existing model's penultimate layer features
6. Fine-tune the classification head on the augmented feature set

Output: Improved classification accuracy from the richer feature representation, without changing the core model architecture.

## Best Practices

- **Do** use a VQ codebook sized appropriately for the signal's complexity: 256 for signals with limited morphological variety (ECG), 512-1024 for complex signals (EEG, audio)
- **Do** include all candidate class names in the prompt instruction, so the LLM can leverage semantic priors about label relationships
- **Do** use constrained decoding at inference time to restrict generation to valid class names only -- unconstrained generation risks producing labels not in the label set
- **Do** pre-train the alignment layer across multiple domains before fine-tuning on the target domain; this substantially improves cross-modal alignment
- **Avoid** using very large LLM backbones; the paper found Qwen3-0.6B outperformed 1.7B and 4B variants on most datasets, suggesting small-to-medium models are sufficient and more efficient
- **Avoid** skipping the implicit feature extraction; statistical and visual features compensate for the LLM's lack of time series inductive biases and consistently improve results over using raw tokens alone

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| VQ codebook collapse (few entries used) | Learning rate too high or codebook too large | Use EMA codebook updates, reduce codebook size, or add codebook reset for unused entries |
| Generated label not in class set | Unconstrained decoding | Implement constrained decoding by masking logits for tokens outside valid class name prefixes |
| Poor cross-modal alignment | Insufficient pre-training | Increase pre-training data diversity (more domains) or train longer; verify MLP hidden dimensions match LLM embedding size |
| Statistical features dominated by outliers | Noisy raw signal | Apply basic preprocessing (bandpass filter, z-score normalization) before computing statistics |
| VLM captions are generic/unhelpful | Poor plot rendering or weak VLM prompt | Use domain-specific plot formatting (e.g., ECG grid lines, labeled axes with units) and more specific VLM prompts ("Describe the shape of each heartbeat cycle") |
| Sequence length exceeds LLM context window | Too many temporal tokens | Increase patch size to reduce token count, or truncate/downsample the signal |

## Limitations

- **Inference latency**: Autoregressive text generation is slower than a single forward pass through a discriminative classifier. Not suitable for real-time classification at sub-millisecond latency requirements.
- **Requires a VLM for visual features**: The image captioning step adds a dependency on a vision-language model (and its GPU memory). For resource-constrained settings, skip visual features and use only statistical features.
- **Label semantics assumption**: The generative approach benefits most when class names carry semantic meaning (e.g., "walking", "atrial fibrillation"). For datasets with arbitrary numeric labels (Class 0, Class 1), the semantic advantage diminishes.
- **Codebook training data**: The VQ discretization module needs sufficient training data to learn a meaningful codebook. Very small datasets (< 100 samples) may produce poor discretizations.
- **Language model ceiling**: While the approach is strong on standard benchmarks (87-99% accuracy on HAR, fault detection), it may underperform specialized architectures on domains requiring very fine-grained temporal resolution that discretization smooths over.

## Reference

**Paper**: [InstructTime++: Time Series Classification with Multimodal Language Modeling via Implicit Feature Enhancement](https://arxiv.org/abs/2601.14968v1) -- Cheng et al., 2026. Key sections: Section 3 (framework architecture), Section 3.3 (implicit feature modeling), Section 4 (experiments with Qwen3-0.6B on 7 benchmark datasets). Look for Table 2 (main results) and the ablation studies showing the contribution of each implicit feature type.