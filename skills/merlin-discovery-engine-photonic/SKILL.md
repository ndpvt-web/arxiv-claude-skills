---
name: "merlin-discovery-engine-photonic"
description: "Build and benchmark photonic quantum machine learning models using the MerLin framework. Integrates linear optical circuit simulation into PyTorch and scikit-learn workflows for differentiable training, kernel methods, and hybrid classical-quantum architectures. Use when: 'set up a photonic QML experiment', 'build a quantum convolutional network with MerLin', 'benchmark quantum vs classical kernels', 'train a hybrid photonic neural network', 'reproduce a photonic QML paper', 'simulate linear optical circuits in PyTorch'."
---

# MerLin: Photonic & Hybrid Quantum Machine Learning Discovery Engine

This skill enables Claude to help users build, train, and benchmark photonic quantum machine learning models using the MerLin framework. MerLin wraps optimized strong linear optical simulation (SLOS) inside standard PyTorch `nn.Module` layers and scikit-learn estimators, so photonic quantum circuits become composable, differentiable building blocks in familiar ML pipelines. The core value is systematic empirical comparison: instead of evaluating one quantum model in isolation, MerLin lets you sweep across architectures (kernels, reservoirs, CNNs, RNNs, GANs), encoding strategies (angle vs amplitude), detector models (threshold vs photon-number-resolving), and datasets on a single codebase.

## When to Use

- When the user wants to build a photonic quantum neural network layer and train it end-to-end with PyTorch
- When the user asks to compare quantum kernel methods against classical baselines on a classification task
- When the user wants to reproduce or extend one of the 18 benchmark photonic QML experiments from the MerLin paper
- When the user needs to encode classical data into photonic quantum states (angle encoding or amplitude encoding)
- When the user asks about hybrid classical-quantum architectures combining conventional layers with photonic circuits
- When the user wants hardware-aware simulation that accounts for realistic detector models and photonic processor constraints
- When the user asks to benchmark quantum reservoir computing, quantum self-supervised learning, or photonic QGANs
- When the user wants to set up a differentiable photonic circuit with trainable beamsplitters and phase shifters

## Key Technique

MerLin's simulation backbone is Strong Linear Optical Simulation (SLOS), which computes the exact quantum state after evolution through a parametrized interferometer in Fock space. The interferometer is decomposed via the Clements scheme into O(m^2) beamsplitter-phase-shifter pairs for m optical modes. Transition amplitudes between Fock states are computed from matrix permanents of submatrices of the unitary. The key optimization: MerLin precomputes a sparse computation graph of layer-by-layer Fock state transitions once per input configuration, then during training only updates the unitary coefficients while reusing the graph structure. This is compiled via TorchScript for GPU-compatible speed. Practical limit is roughly n <= 20 photons on standard hardware, with time complexity O(n * C(m+n-1, n)) where m is modes and n is photons.

The framework exposes this simulation through `QuantumLayer`, a `torch.nn.Module` that accepts configuration for: (1) measurement strategy (full photon-number distribution, per-mode expectations, or raw amplitudes), (2) computation space (full Fock space or restricted encodings), (3) detector model (photon-number-resolving or threshold), and (4) data encoding method. For scikit-learn workflows, `FidelityKernel` computes quantum kernel matrices k(x1, x2) = |<phi(x1)|phi(x2)>|^2 and plugs directly into SVM or kernel ridge regression pipelines. A `QuantumBridge` abstraction handles qubit-to-Fock-space mappings for hybrid workflows. Hardware execution routes through `MerlinProcessor` for cloud-accessible photonic QPUs (e.g., Quandela).

What makes MerLin distinct from other QML frameworks is the benchmarking-first design. It ships with 18 reproduced experiments spanning kernel methods, reservoir computing, photonic QCNNs, quantum RNNs, quantum self-supervised learning, photonic QGANs, quantum transfer learning, data reuploading, distributed QNNs, and quantum knowledge distillation. Each reproduction is a modular, reusable experiment that can be directly extended with new datasets or architectural variants.

## Step-by-Step Workflow

1. **Install MerLin and dependencies.** Clone `github.com/merlinquantum/merlin`, install with pip. Core dependencies are PyTorch, NumPy, and scikit-learn. Optional: Perceval for backend interop, Quandela cloud access for hardware runs.

2. **Define the photonic circuit architecture.** Choose the number of optical modes (m) and photon count (n) based on the problem. Use `CircuitBuilder` to compose layers: call `add_angle_encoding()` to insert phase shifters whose phases map to classical input features, then add parametrized interferometer layers (trainable beamsplitters + phase shifters) between encodings.

3. **Select data encoding strategy.** For tabular or low-dimensional data, use **angle encoding** which maps features to phase shifts and produces Fourier-like models where trainable parameters control linear combinations of fixed frequency features. For high-dimensional data that can be normalized, use **amplitude encoding** which maps a vector x directly into quantum state amplitudes |x> = sum(xi|i>) with ||x||^2 = 1.

4. **Configure the QuantumLayer.** Instantiate `QuantumLayer` as a `torch.nn.Module` specifying: measurement strategy (e.g., per-mode photon expectations for regression, full distribution for classification), detector model (threshold detectors for binary outputs, PNR for richer statistics), and computation space.

5. **Build the hybrid model.** Compose the `QuantumLayer` with classical PyTorch layers (e.g., `nn.Linear` for pre-processing, `nn.Softmax` for post-processing). For kernel methods, instantiate `FidelityKernel` and pass it to `sklearn.svm.SVC` or `sklearn.kernel_ridge.KernelRidge`.

6. **Set up training.** Use standard PyTorch optimizers (Adam recommended) and loss functions. The SLOS backend supports automatic differentiation through the quantum circuit, so `loss.backward()` propagates gradients to beamsplitter and phase shifter parameters. Set learning rates in the 1e-3 to 1e-2 range for quantum parameters.

7. **Run benchmarks against baselines.** Train equivalent classical models (MLPs, CNNs, SVMs with RBF kernels) on the same data splits. Record test accuracy, parameter count, and training time. Use MerLin's reproduced experiments as reference points for expected quantum model performance.

8. **Perform ablation studies.** Vary circuit depth, number of modes, photon count, encoding strategy, and detector model. Compare full Fock space vs restricted encodings. Test data reuploading (re-encoding inputs at multiple circuit layers) vs single-encoding architectures.

9. **Add hardware-aware constraints (optional).** Switch to `MerlinProcessor` to simulate with realistic shot-based sampling, latency, and limited parallelism. Test with threshold detectors to match hardware capabilities. Profile the gap between ideal simulation and hardware-constrained results.

10. **Package results for reproducibility.** Export the experiment as a modular MerLin experiment following the conventions in `merlinquantum/reproduced_papers`. Include dataset loading, model definition, training loop, and evaluation metrics in a self-contained script.

## Concrete Examples

**Example 1: Photonic Quantum Kernel Classification**

User: "I want to classify the Iris dataset using a photonic quantum kernel with MerLin and compare it to a classical RBF-SVM."

Approach:
1. Load Iris dataset, standardize features, split train/test
2. Build a 4-mode photonic circuit with angle encoding (one feature per mode)
3. Compute the quantum kernel matrix using FidelityKernel
4. Train SVM with quantum kernel and with RBF kernel
5. Compare test accuracies

Output:
```python
import numpy as np
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.svm import SVC
from sklearn.metrics import accuracy_score

# MerLin imports
from merlin.kernels import FidelityKernel
from merlin.circuits import CircuitBuilder

# Data preparation
X, y = load_iris(return_X_y=True)
X = StandardScaler().fit_transform(X)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)

# Build photonic circuit for kernel: 4 modes, 2 photons, angle encoding
circuit = CircuitBuilder(n_modes=4, n_photons=2)
circuit.add_angle_encoding(modes=[0, 1, 2, 3])
circuit.add_interferometer(depth=3)

# Quantum kernel
qkernel = FidelityKernel(circuit=circuit)
K_train = qkernel.compute(X_train)
K_test = qkernel.compute(X_test, X_train)

# Quantum SVM
q_svm = SVC(kernel="precomputed")
q_svm.fit(K_train, y_train)
q_acc = accuracy_score(y_test, q_svm.predict(K_test))

# Classical baseline
c_svm = SVC(kernel="rbf")
c_svm.fit(X_train, y_train)
c_acc = accuracy_score(y_test, c_svm.predict(X_test))

print(f"Quantum kernel accuracy: {q_acc:.3f}")
print(f"Classical RBF accuracy:  {c_acc:.3f}")
```

**Example 2: Hybrid Photonic-Classical CNN for Image Classification**

User: "Build a hybrid model that uses a photonic quantum convolutional layer followed by classical dense layers for MNIST binary classification (0 vs 1)."

Approach:
1. Filter MNIST to digits 0 and 1, downsample images to 4x4
2. Create a quantum convolutional layer scanning 2x2 patches through a 4-mode photonic circuit
3. Stack with classical fully-connected layers for final classification
4. Train end-to-end with Adam optimizer

Output:
```python
import torch
import torch.nn as nn
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

from merlin.layers import QuantumLayer
from merlin.circuits import CircuitBuilder

class HybridPhotoniCNN(nn.Module):
    def __init__(self):
        super().__init__()
        # Photonic convolutional layer: 4 modes (2x2 patch), 2 photons
        circuit = CircuitBuilder(n_modes=4, n_photons=2)
        circuit.add_angle_encoding(modes=[0, 1, 2, 3])
        circuit.add_interferometer(depth=4)

        self.q_conv = QuantumLayer(
            circuit=circuit,
            measurement="expectations",  # per-mode photon expectations
            detector="pnr"               # photon-number-resolving
        )
        # Output: 4 expectation values per patch position
        # For 4x4 image with 2x2 stride: 2x2 = 4 patch positions -> 16 features
        self.fc1 = nn.Linear(16, 8)
        self.fc2 = nn.Linear(8, 1)

    def forward(self, x):
        batch_size = x.shape[0]
        # Extract 2x2 patches with stride 2 from 4x4 images
        patches = x.unfold(2, 2, 2).unfold(3, 2, 2)  # (B, 1, 2, 2, 2, 2)
        patches = patches.contiguous().view(batch_size, 4, 4)  # (B, 4 patches, 4 pixels)

        # Process each patch through photonic circuit
        q_out = torch.stack([self.q_conv(patches[:, i]) for i in range(4)], dim=1)
        q_out = q_out.view(batch_size, -1)  # (B, 16)

        out = torch.relu(self.fc1(q_out))
        return torch.sigmoid(self.fc2(out)).squeeze()

# Training
model = HybridPhotoniCNN()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-2)
criterion = nn.BCELoss()

# ... standard PyTorch training loop over filtered MNIST ...
```

Expected result: test accuracy around 98% on MNIST 0-vs-1, comparable to the MerLin paper's reproduced photonic QCNN benchmark (98.8 +/- 1.0%).

**Example 3: Quantum Reservoir Computing for Time Series**

User: "Use MerLin to set up a quantum optical reservoir for a simple time series prediction task."

Approach:
1. Generate a synthetic time series (e.g., Mackey-Glass)
2. Build a fixed (non-trainable) photonic interferometer as the reservoir
3. Feed time-windowed inputs via angle encoding, collect measurement statistics as reservoir features
4. Train a linear readout layer on the reservoir features

Output:
```python
import numpy as np
import torch
from sklearn.linear_model import Ridge

from merlin.layers import QuantumLayer
from merlin.circuits import CircuitBuilder

# Fixed photonic reservoir: 8 modes, 3 photons, random but fixed interferometer
circuit = CircuitBuilder(n_modes=8, n_photons=3)
circuit.add_angle_encoding(modes=list(range(8)))
circuit.add_interferometer(depth=6, trainable=False)  # fixed random unitaries

reservoir = QuantumLayer(
    circuit=circuit,
    measurement="distribution",  # full photon-number distribution
    detector="pnr"
)

# Generate reservoir features from input windows
def extract_features(time_series, window_size=8):
    features = []
    with torch.no_grad():
        for i in range(len(time_series) - window_size):
            window = torch.tensor(time_series[i:i+window_size], dtype=torch.float32)
            feat = reservoir(window.unsqueeze(0))  # (1, D)
            features.append(feat.squeeze().numpy())
    return np.array(features)

# X_reservoir shape: (num_windows, fock_distribution_size)
# Train linear readout with Ridge regression
readout = Ridge(alpha=1.0)
readout.fit(X_train_features, y_train)
predictions = readout.predict(X_test_features)
```

## Best Practices

- **Do:** Start with small circuits (4-6 modes, 2-3 photons) for prototyping. SLOS complexity scales combinatorially, and small circuits train in seconds while large ones can take hours.
- **Do:** Use angle encoding as the default for structured/tabular data. It produces well-understood Fourier-like models and gives clear insight into what frequency components the quantum model can learn.
- **Do:** Always benchmark against a classical baseline of comparable parameter count. MerLin's value is in honest comparison, not quantum hype.
- **Do:** Leverage the 18 reproduced experiments as starting templates. Clone `merlinquantum/reproduced_papers` and modify an existing experiment rather than writing from scratch.
- **Avoid:** Jumping to amplitude encoding without verifying your data can be L2-normalized meaningfully. Amplitude encoding requires ||x||^2 = 1, which distorts data where magnitude carries information.
- **Avoid:** Using full Fock space computation when restricted encodings suffice. If your task only uses single-photon states, restrict the computation space to avoid exponential overhead.
- **Avoid:** Treating photonic quantum layers as drop-in replacements for classical layers in large models. Quantum layers are most effective as specialized feature extractors in small-to-medium circuits, composed with classical pre/post-processing.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| `OutOfMemoryError` during forward pass | Too many modes/photons; Fock space grows as C(m+n-1, n) | Reduce photon count or mode count. For 8 modes + 5 photons, Fock space has 792 states; 12 modes + 5 photons has 4368. |
| Gradients are NaN or explode | Learning rate too high for quantum parameters or deep circuit | Lower learning rate to 1e-3 or below. Clip gradients. Reduce interferometer depth. |
| Kernel matrix is not positive semi-definite | Numerical precision issues with permanent computation | Add small diagonal regularization: K += epsilon * I with epsilon ~1e-8. |
| Training loss plateaus immediately | Barren plateau in quantum circuit optimization | Use shallower circuits, local cost functions, or layer-wise pre-training. Data reuploading can also help. |
| Hardware results differ from simulation | Shot noise, detector imperfections, optical losses | Use `MerlinProcessor` with realistic shot counts (>1000) during development. Compare distribution statistics, not individual shots. |

## Limitations

- **Simulation scale ceiling:** SLOS is exact but scales combinatorially. Circuits beyond ~20 photons or ~15 modes with high photon counts become intractable on standard hardware. This limits the size of quantum models that can be prototyped.
- **No proven quantum advantage for ML:** The paper explicitly positions MerLin as a discovery engine, not a proof of advantage. Current photonic QML models match but rarely exceed classical baselines on standard benchmarks. The value is in systematic exploration, not guaranteed speedups.
- **Angle encoding frequency limitations:** With angle encoding, the circuit's expressivity is bounded by the frequency spectrum fixed at encoding time. Trainable parameters only control linear combinations of these frequencies, which can limit approximation power on complex functions.
- **Hardware gap:** While MerLin supports hardware-aware simulation, actual photonic QPU access is limited to specific providers (Quandela). Results on simulators may not transfer directly due to optical losses, crosstalk, and calibration drift.
- **Not a general quantum computing framework:** MerLin is specifically designed for linear optical (photonic) quantum computing. It does not simulate superconducting, trapped-ion, or other qubit architectures natively, though `QuantumBridge` enables some qubit-to-photonic mappings.

## Reference

**Paper:** Notton, Stott, Schoeb, Walsh, Leboucher. "MerLin: A Discovery Engine for Photonic and Hybrid Quantum Machine Learning." arXiv:2602.11092v1, 2026. ([arxiv.org/abs/2602.11092v1](https://arxiv.org/abs/2602.11092v1))

Look for: Table of 18 reproduced experiments (Section IV), SLOS optimization via precomputed sparse computation graphs (Section III), and the benchmarking methodology comparing quantum vs classical across architectures (Section V).

**Code:** [github.com/merlinquantum/merlin](https://github.com/merlinquantum/merlin) | [github.com/merlinquantum/reproduced_papers](https://github.com/merlinquantum/reproduced_papers)