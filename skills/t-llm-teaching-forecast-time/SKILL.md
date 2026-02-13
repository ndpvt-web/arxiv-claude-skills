---
name: "t-llm-teaching-forecast-time"
description: "Implement temporal distillation pipelines that teach LLMs to forecast time series by training a lightweight trend+frequency teacher and distilling its behavior into a frozen GPT-2/LLaMA backbone with LoRA. Triggers: 'build a time series forecasting pipeline with LLM distillation', 'teach an LLM to forecast', 'temporal distillation for time series', 'T-LLM forecasting setup', 'distill a temporal teacher into GPT-2', 'LLM-based time series prediction with teacher-student training'"
---

# T-LLM: Teaching LLMs to Forecast Time Series via Temporal Distillation

This skill enables Claude to build end-to-end temporal distillation pipelines where a lightweight temporal teacher (combining trend decomposition and frequency-domain analysis) supervises a frozen LLM student (GPT-2 or similar) through multi-level distillation losses. After training, the teacher is discarded entirely — the LLM alone performs inference. This yields LLM-based forecasters that outperform representation-alignment approaches (TimeLLM, CALF, GPT4TS) under full-shot, few-shot, and zero-shot settings while keeping deployment simple.

## When to Use

- When a user asks to build a time series forecasting system that leverages a pretrained LLM backbone (GPT-2, LLaMA) rather than training a transformer from scratch
- When implementing knowledge distillation from a specialized temporal model into a general-purpose language model
- When the user needs a forecaster that works across few-shot or zero-shot transfer scenarios (e.g., train on flu data, deploy on COVID)
- When setting up a training pipeline that combines trend decomposition, FFT-based frequency analysis, and LLM fine-tuning with LoRA
- When the user wants to forecast multivariate time series (electricity, traffic, weather, epidemiological data) using an LLM without heavy inference-time temporal modules
- When comparing or replacing existing LLM-forecasting approaches like GPT4TS, TimeLLM, or CALF

## Key Technique

**Temporal Distillation** differs from prior LLM-forecasting work in a critical way: instead of aligning time series representations to the LLM's embedding space and hoping the LLM generalizes (TimeLLM, CALF), T-LLM explicitly teaches the LLM *how to forecast* by having it imitate a purpose-built temporal teacher during training. The teacher is a lightweight model with two branches: (1) a **trend-seasonal decomposition** branch that applies moving-average filtering to separate trend from seasonal components, then projects each independently; and (2) a **frequency-domain branch** using FFT with a Dominant Spectral Projection (DSP) module that compresses the frequency dimension via a learnable projection matrix sized according to the prediction horizon. These branches fuse via channel-wise gating: `g * F_periodic + (1-g) * H_trend`, letting each channel adaptively balance trend vs. periodic information.

The **student LLM** (typically the first 6 layers of GPT-2) receives time series input through a channel-as-token embedding where each of C channels becomes a token of dimension d_model. Cross-attention with a compact dictionary D of size P (P << vocabulary size) projects temporal features into the LLM's textual embedding space. The LLM backbone stays frozen; only LoRA adapters and the embedding/projection layers are trained. The total loss combines four terms: teacher supervision (L_teach, smooth L1), imitation loss (L_imit, matching student predictions to teacher predictions), feature-level guidance (L_guide, aligning intermediate representations at layers 2 and 3), and direct student supervision (L_stud). Weights are lambda_1=1.0, lambda_2=0.01, lambda_3=1.0. Early stopping on teacher convergence prevents the student from overfitting to imitation.

At **inference**, the teacher is completely removed. The LLM with its trained LoRA adapters, embedding layer, and output projection is the only model deployed. This eliminates the computational overhead of temporal modules at prediction time and simplifies the serving pipeline to a single model forward pass.

## Step-by-Step Workflow

1. **Prepare the dataset in (L, T, C) format.** Load raw time series and normalize per-channel. Structure into samples with lookback window L=96 timesteps and prediction horizon T in {96, 192, 336, 720}. Each sample is X in R^(L x C), Y in R^(T x C). Split into train/val/test (typically 7:1:2).

2. **Build the temporal teacher model.** Implement two parallel branches:
   - **Trend branch**: Apply a moving-average kernel (size 25 is common) to extract trend, compute seasonal as residual, project each through independent linear layers, then concatenate.
   - **Frequency branch**: Compute FFT along the time axis, apply a learnable spectral projection W_spec in R^(d_FFT x d_red) where d_red is set based on prediction horizon (shorter horizon = smaller d_red), then inverse-project back. Fuse both branches with a sigmoid gate per channel.

3. **Build the channel-as-token embedding layer.** Map input X in R^(L x C) to E in R^(C x d_model) via a linear projection. Each channel becomes one token. This lets self-attention capture cross-channel temporal dependencies.

4. **Build the cross-attention alignment module.** Create a compact dictionary D_hat in R^(P x d_model) (P ~64-128, much smaller than vocab). Use multi-head cross-attention where queries come from temporal embeddings and keys/values come from D_hat. This projects temporal features into the LLM's embedding space.

5. **Load the frozen LLM backbone with LoRA.** Load the first 6 transformer layers of GPT-2 (or equivalent). Freeze all original parameters. Apply LoRA (rank 8-16) to the query and value projection matrices. Add an output linear head mapping d_model to T (the prediction horizon) per channel.

6. **Define the four-part distillation loss.**
   ```python
   loss = L_teach + lambda1 * L_stud + lambda2 * L_imit + lambda3 * L_guide
   # L_teach: SmoothL1(Y_hat_teacher, Y_true)
   # L_stud:  SmoothL1(Y_hat_student, Y_true)
   # L_imit:  MSE(Y_hat_student, Y_hat_teacher.detach())
   # L_guide: sum over layers {2,3} of MSE(student_hidden_k, teacher_hidden_k.detach())
   # lambda1=1.0, lambda2=0.01, lambda3=1.0
   ```

7. **Train jointly with early stopping on the teacher.** Use Adam optimizer (lr=5e-4), batch size 32. Monitor teacher validation loss; when it plateaus, freeze the teacher branch and continue training the student for a few more epochs. This prevents the student from over-imitating a still-improving teacher.

8. **Strip the teacher for deployment.** After training, save only: the embedding layer, cross-attention alignment module, LoRA-adapted LLM layers, and output head. The teacher model, its weights, and the guidance loss computation are discarded entirely.

9. **Run inference as a single forward pass.** Input X -> channel embedding -> cross-attention alignment -> frozen LLM + LoRA -> output head -> Y_hat. No temporal module runs at inference. Quantize or export via ONNX for production serving.

10. **Evaluate across settings.** Test on held-out data using MSE and MAE. For few-shot, train on 10% of data. For zero-shot transfer, train on one domain (e.g., ETTh1) and evaluate on a related domain (e.g., ETTh2) with no fine-tuning.

## Concrete Examples

**Example 1: Building a T-LLM pipeline for electricity demand forecasting**

User: "I have hourly electricity consumption data for 321 meters over 2 years. Build a T-LLM forecasting pipeline that predicts the next 96 hours given the last 96 hours."

Approach:
1. Load the Electricity dataset, normalize each of the 321 channels independently (zero mean, unit variance), split 70/10/20.
2. Create sliding window samples: X shape (96, 321), Y shape (96, 321).
3. Build the temporal teacher with moving-average kernel size 25 and DSP with d_red=32 (appropriate for horizon=96).
4. Build channel-as-token embedding: Linear(96, 768) so each of 321 channels maps to a 768-dim token.
5. Load GPT-2 (first 6 layers, frozen) with LoRA rank=8 on Q/V projections.
6. Define the 4-part loss with lambdas (1.0, 0.01, 1.0).
7. Train for 20 epochs, Adam lr=5e-4, batch=32, early stopping on teacher val loss.
8. Strip teacher, save student checkpoint.

Output:
```
Model: GPT-2 (6 layers) + LoRA rank 8
Parameters trained: ~2.1M (embedding + LoRA + output head)
Parameters frozen: ~81M (GPT-2 backbone)
Teacher parameters (discarded): ~0.8M
Inference: single forward pass, ~4ms per batch on RTX 4090
Expected MSE: ~0.168 (competitive with PatchTST, better than GPT4TS)
```

**Example 2: Zero-shot cross-domain transfer for epidemic forecasting**

User: "I have weekly ILI (influenza-like illness) data. Train on flu seasons and test zero-shot on COVID hospitalization data without any fine-tuning."

Approach:
1. Prepare ILI training data: L=36 weeks lookback, T=12 weeks prediction horizon, C=7 regions.
2. Build temporal teacher with DSP d_red=8 (short horizon needs less spectral capacity).
3. Train the full T-LLM pipeline on ILI data with the 4-part distillation loss.
4. At evaluation, feed COVID hospitalization data (same 7 regions, same temporal format) directly into the trained student LLM — no teacher, no fine-tuning.
5. Evaluate MAE/MSE on COVID test set.

Output:
```
Training domain: ILI (2010-2019 flu seasons)
Evaluation domain: COVID hospitalizations (2020-2021), zero-shot
Transfer mechanism: LLM learned generalizable temporal patterns via distillation
Expected improvement over CALF baseline: ~2.2% lower MSE
Key insight: frequency-domain teacher captures seasonal patterns that transfer across respiratory diseases
```

**Example 3: Few-shot forecasting with limited weather data**

User: "I only have 10% of the Weather dataset available for training. Set up T-LLM to forecast 336 steps ahead."

Approach:
1. Subsample Weather dataset to 10% of training split (keep full val/test). 21 channels, L=96, T=336.
2. Build teacher with DSP d_red=64 (longer horizon needs more spectral capacity).
3. Critical: with limited data, the distillation losses are especially valuable — the teacher's inductive bias (trend + frequency structure) acts as a strong regularizer for the LLM student.
4. Train with same loss weights but increase lambda2 (imitation) to 0.05 to lean more on teacher guidance when data is scarce.
5. Apply early stopping aggressively (patience=3).

Output:
```
Training data: 10% of Weather (~5,200 samples)
Prediction horizon: 336 steps
Channels: 21 meteorological variables
Few-shot advantage: teacher provides structured temporal priors that compensate for limited data
Expected MSE improvement over GPT4TS: ~8-12% with 10% data
```

## Best Practices

- **Do:** Set the DSP reduced dimension (d_red) proportional to prediction horizon — shorter horizons need fewer spectral components, longer horizons need more. Use the paper's horizon-capacity pairs as a starting guide.
- **Do:** Detach teacher outputs before computing imitation and guidance losses. The teacher should supervise the student, not be pulled toward the student's representations.
- **Do:** Use channel-as-token embedding (not patch-based) for multivariate data. This lets the LLM's attention mechanism naturally model cross-channel dependencies.
- **Do:** Freeze the LLM backbone entirely and only train LoRA adapters, embedding layers, and output heads. This preserves the LLM's pretrained representations while keeping training efficient (~2-3% of parameters trainable).
- **Avoid:** Skipping the early stopping on teacher convergence. Without it, the student may overfit to an unstable teacher's intermediate predictions, degrading final performance.
- **Avoid:** Using a very large LoRA rank (>16). The distillation signal is structured enough that low-rank adaptation suffices; higher ranks risk overfitting, especially in few-shot settings.
- **Avoid:** Keeping the teacher at inference time "just in case." The entire point of T-LLM is that the student internalizes temporal reasoning — ensembling with the teacher negates the deployment simplicity benefit.

## Error Handling

- **Teacher diverges during training**: If teacher loss increases after initial convergence, reduce learning rate by half and restart. The teacher is lightweight and trains fast, so this is cheap.
- **Student loss plateaus while imitation loss is still high**: The student is struggling to match the teacher. Check that cross-attention alignment is working (visualize attention weights). Increase dictionary size P or add another attention head.
- **Zero-shot transfer fails badly**: The source and target domains may have fundamentally different temporal dynamics. Verify that the frequency spectra overlap (compute FFT of both and compare dominant frequencies). If they don't overlap, zero-shot won't work — use few-shot instead.
- **Out-of-memory with many channels**: With C > 500 channels (e.g., Traffic with 862), the channel-as-token approach creates long sequences. Use gradient checkpointing on the LLM layers or subsample channels during training.
- **Frequency branch produces artifacts**: If predictions show ringing or oscillation, d_red may be too large, capturing noise frequencies. Reduce d_red or apply a low-pass filter to the FFT output before projection.

## Limitations

- **Requires paired training**: Unlike pure prompting approaches, T-LLM needs a training phase with the teacher. It cannot be applied zero-cost to a new dataset — at minimum, the embedding layers and LoRA adapters must be trained.
- **Fixed prediction horizon**: The output head is a linear projection to T steps. Changing the horizon requires retraining or maintaining multiple output heads.
- **Channel count scaling**: The channel-as-token design means attention cost scales quadratically with the number of channels. Datasets with thousands of channels (e.g., large sensor networks) may need channel grouping or sampling.
- **LLM backbone size**: The paper uses GPT-2 (124M params, 6 layers used). Scaling to larger LLMs (7B+) hasn't been extensively validated and may shift optimal LoRA ranks and distillation weights.
- **Irregular time series**: The method assumes regular sampling intervals. Missing data or irregular timestamps require preprocessing (interpolation, resampling) before T-LLM can be applied.
- **Interpretability**: While the teacher's trend/frequency decomposition is interpretable, once distilled into the LLM, the forecasting logic becomes opaque within the transformer layers.

## Reference

**Paper**: [T-LLM: Teaching Large Language Models to Forecast Time Series via Temporal Distillation](https://arxiv.org/abs/2602.01937v1) (Guo et al., 2026)
**What to look for**: Section 3 for the full framework architecture (teacher branches, distillation losses, cross-attention alignment), Table 1-2 for benchmark results across horizons, Table 3 for few-shot/zero-shot transfer results, and Appendix for hyperparameter sensitivity analysis and DSP horizon-capacity mappings.