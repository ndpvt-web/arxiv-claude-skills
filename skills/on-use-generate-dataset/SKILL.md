---
name: "on-use-generate-dataset"
description: "Generate diverse, validated datasets of neural network implementations using LLM-driven combinatorial design. Use when: 'generate neural network dataset', 'create NN benchmark', 'build dataset of PyTorch models', 'generate diverse architectures for testing', 'create neural network test suite', 'benchmark NN verification tools'."
---

This skill enables Claude to systematically generate large, diverse datasets of neural network implementations in PyTorch by applying a combinatorial design methodology from Daoudi & Cabot (2026). Rather than writing networks ad-hoc, Claude constructs a design space of architecture types, tasks, input modalities, scales, and complexity levels, then generates one valid network per combination -- each validated through AST-based static analysis and symbolic tracing to guarantee correctness.

## When to Use

- When the user needs a benchmark dataset of neural networks to test NN analysis tools, code verifiers, or refactoring pipelines
- When the user asks to generate many diverse PyTorch model definitions covering different architectures (MLP, CNN, RNN variants)
- When the user wants to stress-test a model migration tool by feeding it architecturally varied networks
- When the user needs validated, self-contained PyTorch model classes for testing static analysis or linting tools
- When the user is building a dataset for meta-learning or neural architecture search and needs structurally diverse seed models
- When the user asks to create a test suite of neural networks spanning multiple input types (tabular, time series, text, image)

## Key Technique

The core insight is **combinatorial coverage over a structured design space**. Instead of randomly prompting an LLM to "write some neural networks," the method defines seven architecture families (MLP, CNN-1D, CNN-2D, CNN-3D, RNN-Simple, RNN-LSTM, RNN-GRU), four task types (binary classification, multiclass classification, regression, representation learning), four input modalities (tabular, time series, text, image) each with scale variants, and four complexity tiers. Each valid combination of these dimensions produces exactly one generation prompt. This eliminates redundancy and guarantees coverage.

Each generated network must be a **self-contained PyTorch `nn.Module`** with a `forward` method, no external dependencies, no helper functions, and fixed hyperparameters. The prompt enforces these constraints explicitly. Complexity is controlled by thresholds on the "characterizing layers" (CLs) -- the layers that define the architecture family (e.g., `nn.Linear` for MLPs, `nn.Conv2d` for CNN-2D). Width thresholds cap neuron/channel counts; depth thresholds cap CL count. Four tiers emerge: low-width/low-depth, low-width/high-depth, high-width/low-depth, high-width/high-depth.

After generation, each network passes through a **two-stage validation pipeline**. First, AST parsing extracts all layers and their execution order, checking that (a) at least one CL exists, (b) the output layer matches the task (e.g., single output for regression, sigmoid for binary classification), (c) the first CL's input dimensions match the specified input shape, and (d) depth/width fall within complexity thresholds. Second, symbolic tracing with `torch.fx` or manual shape propagation confirms that a dummy input of the correct shape flows through the network without dimension mismatches. Networks failing validation are regenerated with the same prompt.

## Step-by-Step Workflow

1. **Define the design space.** Enumerate the architecture families relevant to the user's goal. Full space: MLP, CNN-1D, CNN-2D, CNN-3D, RNN-Simple, RNN-LSTM, RNN-GRU. Map each to its characterizing layer type (`nn.Linear`, `nn.Conv1d`, `nn.Conv2d`, `nn.Conv3d`, `nn.RNN`, `nn.LSTM`, `nn.GRU`).

2. **Define task types.** Select from: binary classification (1 output + sigmoid), multiclass classification (N outputs + softmax/log_softmax), regression (1 output, no activation), representation learning (embedding output, no task-specific head).

3. **Define input modalities and scales.** For each modality, specify concrete input shapes:
   - Tabular: `(batch, F)` where F < 50 or F > 2000
   - Time series: `(batch, seq_len, features)` with varying seq_len and feature counts
   - Text: `(batch, seq_len)` with vocab size < 1k or > 100k, requiring an `nn.Embedding` layer
   - Image: `(batch, C, H, W)` with resolution < 64 or > 1024

4. **Define complexity tiers.** Set width and depth thresholds for each architecture's characterizing layers:
   - MLP/RNN: width threshold = `min(128, upper_bound)`, depth threshold = 4 (RNN: 2)
   - CNN: width threshold = `min(8, W//8)` for channels, depth threshold = 4
   - Tiers: (low-w, low-d), (low-w, high-d), (high-w, low-d), (high-w, high-d)

5. **Generate the combination matrix.** Compute all valid (architecture, task, input_modality, scale, complexity) tuples. Filter invalid pairings (e.g., CNN-2D cannot process raw tabular data without reshaping; MLP does not naturally handle images at scale). Record expected sample count.

6. **Construct the generation prompt for each combination.** Use a structured prompt template:
   ```
   Generate a complete PyTorch neural network as a single nn.Module class that satisfies:
   - Architecture: {arch_type} using {CL_type} as primary layers
   - Task: {task_type} with appropriate output layer
   - Input: {modality} with shape {input_shape} (scale: {scale_label})
   - Complexity: width {<= or >} {width_threshold}, depth {<= or >} {depth_threshold}
   Requirements: self-contained, no external imports beyond torch/torch.nn,
   no helper functions, all hyperparameters hardcoded, include forward() method.
   ```

7. **Generate each network.** Call the LLM once per combination. Collect the output as a Python string containing a single class definition.

8. **Validate with static analysis.** Parse each generated file's AST to extract:
   - All `nn.*` layer instantiations and their parameter values
   - The `forward()` method's execution order
   - Verify: CL count >= 1, output layer matches task, first CL input matches spec, width/depth within tier

9. **Validate with symbolic tracing.** Create a dummy input tensor of the specified shape, instantiate the model in eval mode, and run a forward pass. Confirm output shape matches task requirements (e.g., `(batch, 1)` for regression, `(batch, num_classes)` for multiclass). Catch any `RuntimeError` for shape mismatches.

10. **Regenerate failures.** For any network failing validation, re-prompt with the same specification (optionally appending the error message as feedback). Repeat up to 3 times before flagging as unresolvable.

## Concrete Examples

**Example 1: Generate a small benchmark for testing a model refactoring tool**

User: "I need 16 diverse PyTorch models to test my automated refactoring tool. Cover different architectures and tasks."

Approach:
1. Select 4 architectures: MLP, CNN-2D, RNN-LSTM, RNN-GRU
2. Select 4 tasks: binary classification, multiclass classification, regression, representation learning
3. Use one input modality per architecture (tabular for MLP, image for CNN-2D, time series for LSTM/GRU)
4. Fix complexity to one tier (low-width, low-depth) for manageable size
5. Generate 4 x 4 = 16 combinations, one network each

Output (one sample -- CNN-2D binary classification):
```python
import torch
import torch.nn as nn

class CNN2D_BinaryClassification(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(3, 8, kernel_size=3, padding=1)
        self.conv2 = nn.Conv2d(8, 8, kernel_size=3, padding=1)
        self.pool = nn.AdaptiveAvgPool2d(1)
        self.bn1 = nn.BatchNorm2d(8)
        self.fc = nn.Linear(8, 1)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        x = self.relu(self.bn1(self.conv1(x)))
        x = self.relu(self.conv2(x))
        x = self.pool(x).flatten(1)
        x = self.sigmoid(self.fc(x))
        return x
```

Validation output:
```
[PASS] Static analysis: 2 CLs (Conv2d), width=8 <= 8, depth=2 <= 4
[PASS] Output layer: Linear(8,1) + Sigmoid -> binary classification
[PASS] Symbolic trace: input (1,3,32,32) -> output (1,1)
```

**Example 2: Full-scale dataset generation for NN verification research**

User: "Generate a comprehensive dataset of neural networks following the Daoudi-Cabot methodology for benchmarking our verification tool."

Approach:
1. Build the full design space: 7 architectures, 4 tasks, 4 modalities with 2-4 scales each, 4 complexity tiers
2. Compute valid combinations: Tabular(32) + TimeSeries(256) + Text(256) + Image(64) = 608
3. Generate all 608 networks with structured prompts
4. Run AST validation on each, flagging non-compliant samples
5. Run symbolic tracing on compliant samples
6. Regenerate any failures (expect ~1-2% failure rate, primarily in RNN variants)
7. Output: directory of 608 `.py` files plus a `metadata.json` with design dimensions per sample

Output structure:
```
dataset/
  tabular/
    mlp_binary_small_loww_lowd.py
    mlp_binary_small_loww_highd.py
    ...
  timeseries/
    rnn_simple_multiclass_short_highw_lowd.py
    lstm_regression_long_loww_highd.py
    ...
  text/
    gru_representation_smallvocab_highw_highd.py
    ...
  image/
    cnn2d_binary_lowres_loww_lowd.py
    ...
  metadata.json
  validation_report.json
```

**Example 3: Generating networks with validation feedback loop**

User: "Generate 4 RNN-LSTM networks for time series regression at different complexity levels, and validate each one."

Approach:
1. Fix: architecture=RNN-LSTM, task=regression, input=time_series(batch, 50, 10)
2. Four complexity tiers: (width<=128, depth<=2), (width<=128, depth>2), (width>128, depth<=2), (width>128, depth>2)
3. Generate, validate, regenerate on failure

Output for one failed-then-fixed sample:
```
Attempt 1 - LSTM high-width high-depth:
[FAIL] Static analysis: Linear layer before first LSTM disrupts sequential structure
  -> Regenerating with error context appended to prompt

Attempt 2:
[PASS] Static analysis: 3 CLs (LSTM), width=256 > 128, depth=3 > 2
[PASS] Output layer: Linear(256, 1), no activation -> regression
[PASS] Symbolic trace: input (1, 50, 10) -> output (1, 1)
```

## Best Practices

- **Do:** Always include the task-specific output constraint in the prompt (sigmoid for binary, softmax for multiclass, raw for regression). This is the most common source of non-compliance if omitted.
- **Do:** Validate with both static analysis AND symbolic tracing. AST checks catch structural violations; tracing catches dimension mismatches that look correct on paper.
- **Do:** Use `model.eval()` and `torch.no_grad()` during symbolic tracing to avoid batch norm and dropout issues with single-sample dummy inputs.
- **Do:** Name generated files and classes systematically using the design dimensions (e.g., `cnn2d_multiclass_highres_hw_hd.py`) so the dataset is self-documenting.
- **Avoid:** Generating networks with external data loading, training loops, or imports beyond `torch` and `torch.nn`. Each sample should be a pure architecture definition.
- **Avoid:** Allowing the LLM to add comments, docstrings, or helper functions. These introduce variability that complicates automated analysis. Enforce "no comments, no helpers" in the prompt.

## Error Handling

- **AST parse failure:** The generated code has syntax errors. Regenerate with the same prompt. If persistent, simplify the complexity tier.
- **Missing characterizing layer:** The LLM substituted a different layer type (e.g., used Linear instead of Conv2d for a CNN). Regenerate with a more explicit prompt specifying "you MUST use {CL_type} layers."
- **Dimension mismatch at forward pass:** Most common in RNN variants where a Linear projection precedes the recurrent layer. Append the traceback to the regeneration prompt as feedback.
- **Output shape mismatch:** The network produces wrong output dimensions for the task. Check that the prompt explicitly specifies expected output shape.
- **Complexity tier violation:** Width or depth exceeds thresholds. This is subtle -- the LLM may add extra layers. AST validation catches it; regenerate with stricter prompt language ("use EXACTLY N layers").

## Limitations

- The combinatorial space explodes if you add too many dimensions. The paper's 608 samples use a curated set of valid combinations; adding new modalities or architectures requires careful filtering of invalid pairings.
- Generated networks are structurally valid but not trained or tuned. They are useful for testing analysis tools, not for evaluating model performance.
- RNN variants (LSTM, GRU) have the highest failure rate (~1-2%) due to the complexity of sequence handling. Budget extra regeneration attempts for these.
- The method assumes PyTorch. Adapting to TensorFlow/Keras requires rewriting the prompt templates and validation pipeline.
- Representation learning networks lack a task-specific output head, which may confuse analysis tools expecting classification/regression structure.
- Validation via symbolic tracing requires instantiating the model, which can fail for very large networks on memory-constrained systems.

## Reference

Daoudi, N. & Cabot, J. (2026). *On the use of LLMs to generate a dataset of Neural Networks*. arXiv:2602.04388v1. https://arxiv.org/abs/2602.04388v1

Key takeaway: the paper's Table 1 (architecture taxonomy with CL types and thresholds) and Table 2 (combinatorial distribution across modalities) are the essential references for reproducing the design space. The prompt template in Listing 1 provides the exact constraint structure for generation.