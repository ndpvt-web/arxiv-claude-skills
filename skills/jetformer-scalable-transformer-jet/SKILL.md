---
name: "jetformer-scalable-transformer-jet"
description: "Build, train, compress, and deploy JetFormer encoder-only Transformers for particle jet tagging -- from offline analysis to FPGA triggers. Use when: 'build a jet tagger', 'deploy transformer on FPGA', 'compress a transformer for edge hardware', 'particle physics classification', 'hardware-aware hyperparameter search', 'quantize and prune a transformer model'."
---

# JetFormer: Scalable Transformer for Jet Tagging -- Offline to FPGA

This skill enables Claude to implement, train, compress, and deploy JetFormer, an encoder-only Transformer architecture designed for particle jet classification at the LHC. JetFormer processes variable-length sets of particle features using self-attention (no explicit pairwise interaction inputs), achieves state-of-the-art accuracy on JetClass and HLS4ML benchmarks, and can be aggressively compressed via structured pruning and quantization for sub-microsecond FPGA inference. The core workflow spans the full pipeline: data preparation, model definition, multi-objective hyperparameter optimization, training, compression, and hardware synthesis via the Allo/HLS4ML toolchain.

## When to Use

- When the user needs to classify particle jets (quark/gluon, top, W/Z/Higgs tagging) from LHC collision data
- When building an encoder-only Transformer that must operate on variable-length, unordered sets of features (not just NLP sequences)
- When deploying a Transformer model to FPGA hardware with strict latency budgets (sub-microsecond)
- When performing multi-objective hyperparameter optimization balancing accuracy against FLOPs, latency, or model size
- When applying structured pruning and low-bit quantization to compress a Transformer while preserving accuracy
- When the user needs a hardware-aware ML pipeline that bridges PyTorch training with HLS/FPGA synthesis
- When comparing Transformer-based classifiers against DeepSets, Interaction Networks, or MLPs on point-cloud-like scientific data

## Key Technique

**Architecture.** JetFormer is a BERT-style encoder-only Transformer operating on sets. Each particle in a jet is represented as a feature vector (3 kinematic features: pT, eta, phi; or 16 extended features). A linear embedding layer projects each particle's features into a d-dimensional space. A learnable `[CLS]` token is prepended to the sequence. The embedded tokens pass through N stacked Transformer encoder blocks (multi-head self-attention + feed-forward network with GELU activation, layer normalization, and dropout). The `[CLS]` token output is fed to a classification head (linear layers) that produces jet-type logits. Crucially, no positional encoding is used -- the input is treated as an unordered set, which is physically appropriate since particle ordering within a jet is arbitrary. No explicit pairwise interaction features are computed, reducing FLOPs by ~37% compared to interaction-rich models like ParT while staying within 0.7% accuracy.

**Hardware-Aware Optimization.** The paper introduces a multi-objective hyperparameter optimization (HPO) pipeline using Optuna with hypervolume-based Pareto search. The search space covers: number of encoder layers (1-6), embedding dimension (16-128), number of attention heads (1-8), feed-forward hidden dimension, dropout rate, and learning rate. Two competing objectives are optimized simultaneously: maximize classification accuracy and minimize a hardware cost proxy (FLOPs, estimated latency, or parameter count). This yields a Pareto front of model variants ranging from JetFormer-large (high accuracy, offline use) to JetFormer-tiny (compact, FPGA-deployable).

**Compression and Deployment.** Selected Pareto-optimal models are further compressed through (1) structured pruning -- removing entire attention heads and feed-forward neurons ranked by importance scores, then fine-tuning -- and (2) post-training quantization (PTQ) and quantization-aware training (QAT) down to 8-bit or lower fixed-point representations. The compressed model is translated to synthesizable hardware via the Allo framework, which generates HLS C++ code targeting Xilinx FPGAs with pipelining and parallelization directives for sub-microsecond inference.

## Step-by-Step Workflow

1. **Set up the environment.** Clone `https://github.com/walkieq/JetFormer` and install dependencies from `environment_3090.yml` (GPU training) or `environment_allo.yml` (FPGA synthesis). Key dependencies: PyTorch, Optuna, scikit-learn, h5py, and the Allo framework for HLS.

2. **Prepare the dataset.** For quick benchmarking, use the HLS4ML HLC Jets dataset (auto-downloaded by the training script). For the 150-particle dataset, download from Zenodo and place files in `transformer/data/`. For full-scale evaluation, clone the `particle_transformer` repo and follow its JetClass data preparation instructions.

3. **Define model hyperparameters.** Configure the JetFormer architecture through CLI arguments to `train.py`:
   ```bash
   python3 train.py \
     --num_particles 30 \
     --num_feats 16 \
     --num_transformers 3 \
     --embbed_dim 64 \
     --num_heads 2 \
     --batch_size 256 \
     --dropout 0.0 \
     --save
   ```
   Key choices: `num_particles` sets the max sequence length (zero-padded for shorter jets), `num_feats` selects 3 (kinematic only) or 16 (extended), `num_transformers` is the encoder depth, `embbed_dim` is the embedding/hidden dimension, `num_heads` is the number of attention heads.

4. **Train the model.** Training uses AdamW optimizer with cosine-annealing learning rate schedule and early stopping on validation loss. Monitor training via the saved logs. The training script handles train/val/test splits, data normalization, and checkpoint saving.

5. **Run multi-objective HPO.** Use the scripts in `transformer/hpo/` to launch an Optuna study with multiple objectives (accuracy vs. FLOPs or latency). Configure the search space bounds and the number of trials. Analyze the resulting Pareto front to select candidate models for different deployment scenarios (offline vs. trigger).

6. **Apply structured pruning.** Using the scripts in `transformer/compress/`, rank attention heads and feed-forward neurons by importance (e.g., Taylor expansion or magnitude-based scores). Remove the lowest-ranked structures at a target sparsity ratio (e.g., 50-70%). Fine-tune the pruned model for a few epochs to recover accuracy.

7. **Quantize the model.** Apply post-training quantization (PTQ) to convert weights and activations from FP32 to INT8 or lower fixed-point. For tighter compression, use quantization-aware training (QAT) where fake-quantization nodes are inserted during fine-tuning. Evaluate the accuracy-latency tradeoff at each precision level.

8. **Estimate hardware cost.** Use the latency and model-size estimation tools in `transformer/compress/` to predict on-FPGA resource utilization (LUTs, DSPs, BRAM) and inference latency before synthesis.

9. **Synthesize for FPGA.** Use the scripts in `transformer/hls/` (e.g., `particle_transformer.py`) to generate HLS C++ via the Allo framework. Configure pipelining depth and parallelization factors to meet the target latency. Run Vivado HLS to produce the bitstream.

10. **Validate end-to-end.** Compare the FPGA inference outputs against the PyTorch reference model on a held-out test set to confirm that compression and synthesis introduced no significant accuracy regression (target: <1% drop).

## Concrete Examples

**Example 1: Train a baseline JetFormer on the 30-particle dataset**

User: "Train a jet tagger on the HLS4ML 30-particle dataset with 16 features."

Approach:
1. Clone the JetFormer repo and set up the conda environment from `environment_3090.yml`.
2. Run training with default hyperparameters:
   ```bash
   cd transformer
   python3 train.py --num_particles 30 --num_feats 16 \
     --num_transformers 3 --embbed_dim 64 --num_heads 2 \
     --batch_size 256 --dropout 0.0 --save
   ```
3. Monitor training logs for convergence; early stopping triggers when validation loss plateaus.
4. Evaluate on the test split -- expect ~76-78% 5-class accuracy (quark, gluon, W, Z, top), outperforming DeepSets and Interaction Networks by 3-4%.

Output:
```
Epoch 120/200 | Train Loss: 0.612 | Val Loss: 0.638 | Val Acc: 77.2%
Early stopping triggered at epoch 142.
Test Accuracy: 77.4% | FLOPs: 1.2M | Parameters: 45K
Model saved to checkpoints/jetformer_30p_16f.pt
```

**Example 2: Multi-objective HPO to find FPGA-deployable variants**

User: "Find the smallest JetFormer that still gets >74% accuracy on the 30-particle dataset, optimized for FPGA latency."

Approach:
1. Configure an Optuna study in `transformer/hpo/` with two objectives: maximize accuracy, minimize estimated FLOPs.
2. Define the search space:
   - `num_transformers`: [1, 2, 3]
   - `embbed_dim`: [16, 32, 64]
   - `num_heads`: [1, 2, 4]
   - `dropout`: [0.0, 0.1, 0.2]
   - `learning_rate`: [1e-4, 1e-2] (log-uniform)
3. Run 100 trials with early stopping per trial.
4. Extract the Pareto front and filter for models exceeding 74% accuracy.
5. Select the smallest (lowest-FLOP) model meeting the accuracy threshold.

Output:
```
Pareto front (top 5 by efficiency):
  Trial 42: Acc=74.8%, FLOPs=210K, Layers=1, Dim=32, Heads=2  <-- JetFormer-tiny
  Trial 67: Acc=75.9%, FLOPs=480K, Layers=2, Dim=32, Heads=2
  Trial 23: Acc=76.5%, FLOPs=620K, Layers=2, Dim=48, Heads=2
  Trial 91: Acc=77.1%, FLOPs=950K, Layers=3, Dim=64, Heads=2
  Trial 12: Acc=77.4%, FLOPs=1.2M, Layers=3, Dim=64, Heads=2  <-- JetFormer-base

Selected: Trial 42 (JetFormer-tiny) -- 74.8% accuracy, 210K FLOPs
```

**Example 3: Compress and deploy JetFormer-tiny to FPGA**

User: "Take the JetFormer-tiny checkpoint and deploy it to an FPGA with sub-microsecond latency."

Approach:
1. Load the JetFormer-tiny checkpoint (1 layer, dim=32, 2 heads).
2. Apply structured pruning at 50% sparsity on the feed-forward layer, then fine-tune for 20 epochs.
3. Apply post-training quantization to INT8 fixed-point.
4. Estimate latency using `transformer/compress/` tools -- confirm <1 us at target clock frequency.
5. Generate HLS C++ via `transformer/hls/particle_transformer.py` using the Allo framework.
6. Run Vivado HLS synthesis targeting a Xilinx Kintex UltraScale FPGA.
7. Validate: compare FPGA output logits against PyTorch reference on 10K test jets.

Output:
```
Post-pruning accuracy: 74.3% (from 74.8%, -0.5%)
Post-quantization accuracy: 74.1% (from 74.3%, -0.2%)
Estimated FPGA latency: 0.78 us @ 200 MHz
Resource utilization: 12% LUTs, 8% DSPs, 5% BRAM
Bitstream generated: jetformer_tiny_int8.bit
Validation: max logit difference vs PyTorch = 0.003 (pass)
```

## Best Practices

- **Do:** Treat the particle set as unordered -- omit positional encodings. Particle ordering in jets is physically meaningless, and adding positional encoding degrades generalization.
- **Do:** Start HPO with a coarse grid on encoder depth and embedding dimension first, then refine with Optuna's TPE sampler. These two hyperparameters dominate both accuracy and hardware cost.
- **Do:** Use structured pruning (removing entire heads/neurons) rather than unstructured (individual weight zeroing). Structured pruning translates directly to real FLOP and latency savings on FPGAs.
- **Do:** Validate accuracy after each compression step (pruning, then quantization) independently before stacking them. Compounding errors from both steps simultaneously makes debugging impossible.
- **Avoid:** Using more than 4 encoder layers for FPGA deployment -- the latency scales linearly with depth, and diminishing accuracy returns set in after 2-3 layers for the 30-particle dataset.
- **Avoid:** Skipping the fine-tuning step after structured pruning. Without fine-tuning, pruned models lose 3-5% accuracy that is easily recoverable with 10-20 epochs of retraining.

## Error Handling

- **Out-of-memory during training:** Reduce `batch_size` or `embbed_dim`. For the 150-particle dataset with 16 features, batch size 128 with dim 64 fits on a 24GB GPU. Gradient accumulation can compensate for smaller batches.
- **HPO trials failing early:** Set a minimum epoch count (e.g., 20) before allowing early stopping in HPO trials. Very small models may have noisy validation curves that trigger premature stopping.
- **Quantization accuracy collapse:** If PTQ drops accuracy by >2%, switch to QAT with a low learning rate (1e-5) for 10-20 epochs. Some attention layers are more sensitive to quantization -- try mixed-precision where attention stays FP16 and feed-forward layers are INT8.
- **HLS synthesis timing failure:** If the design fails timing at the target clock frequency, reduce parallelization factor or increase pipeline depth in the Allo configuration. Alternatively, revisit the model size -- a slightly smaller embedding dimension (32 -> 24) can resolve critical path issues.
- **Zero-padded particles corrupting attention:** Ensure the attention mask correctly masks out zero-padded particle positions. Without masking, the model attends to padding tokens and accuracy drops by 2-5%.

## Limitations

- JetFormer's accuracy ceiling is ~0.7% below interaction-rich models like ParT on JetClass. If maximum offline accuracy is the only goal and compute is unconstrained, ParT may be preferable.
- The FPGA deployment pipeline depends on the Allo framework and Vivado HLS, which require specific toolchain versions and FPGA board support. Not all FPGA families are supported out of the box.
- The 150-particle dataset must be manually downloaded from Zenodo -- there is no automated download path in the current codebase.
- Quantization below INT8 (e.g., INT4 or binary) is not validated in the paper and may require significant architecture modifications.
- The current codebase targets classification (jet tagging). Adapting JetFormer for regression tasks (e.g., jet energy calibration) requires modifying the classification head and loss function.

## Reference

- **Paper:** [JetFormer: A Scalable and Efficient Transformer for Jet Tagging from Offline Analysis to FPGA Triggers](https://arxiv.org/abs/2601.17215v1) -- Focus on Section 3 (architecture), Section 4 (HPO pipeline), and Section 5 (compression and FPGA deployment) for implementation details.
- **Code:** [https://github.com/walkieq/JetFormer](https://github.com/walkieq/JetFormer) -- Training, compression, and HLS synthesis scripts.