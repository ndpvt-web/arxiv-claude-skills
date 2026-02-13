---
name: "spectral-guardrails-agents-wild"
description: "Implement training-free hallucination detection for LLM agent tool calls using spectral analysis of attention topology. Computes graph Laplacian eigenvalues from attention matrices to detect when a model's reasoning becomes incoherent. Triggers: 'detect tool use hallucinations', 'spectral guardrail for agent', 'attention topology analysis', 'hallucination detection without training data', 'agent safety guardrail', 'spectral analysis of attention heads'"
---

# Spectral Guardrails: Detecting Tool Use Hallucinations via Attention Topology

This skill enables Claude to implement a **training-free hallucination detection system** for LLM agents that use tools. The core technique treats transformer attention matrices as weighted graphs, computes their graph Laplacian eigenvalues, and extracts four spectral diagnostics (entropy, Fiedler value, smoothness, high-frequency energy ratio) that reveal when a model's internal reasoning has become incoherent. A single spectral feature at the right layer can catch 98% of hallucinated tool calls with zero labeled training data.

## When to Use

- When building an autonomous agent framework and needing a runtime guardrail to catch hallucinated tool calls (fabricated API parameters, wrong function names, impossible arguments)
- When deploying open-weight models (Llama, Mistral, Qwen) with tool use and needing a safety layer that requires no fine-tuning or labeled data
- When the user asks to monitor attention patterns during inference to flag unreliable generations
- When implementing a fallback/retry mechanism that triggers on detected hallucinations before the tool call reaches an external service
- When evaluating whether a model's tool use output is trustworthy in a specific domain (finance, code generation, general Q&A) without building a classifier
- When comparing hallucination detection across different model architectures or sizes

## Key Technique

**Attention as a dynamic graph.** During each forward pass, every attention head produces an N-by-N matrix (where N is the sequence length). The method symmetrizes each head's attention matrix as `W = 0.5 * (A + A^T)`, aggregates across heads using attention-mass weighting (`alpha_h = total_mass_h / sum_all_masses`), and constructs the combinatorial graph Laplacian `L = D - W` (where D is the degree matrix). The eigendecomposition of L yields eigenvalues that encode the topology of information flow in that layer.

**Four spectral diagnostics detect hallucination.** (1) **Spectral Entropy** measures how uniformly signal energy spreads across eigenmodes — high entropy means dispersed, noisy reasoning. (2) **Fiedler Value** (the second-smallest eigenvalue, lambda_2) measures graph connectivity — low values mean fragmented attention. (3) **Smoothness** measures whether connected tokens have similar hidden representations — values near 1 mean coherent flow, near 0 means chaotic. (4) **High-Frequency Energy Ratio (HFER)** measures how much signal energy sits in the upper half of the spectrum — high values mean incoherent, noisy representations.

**The "thermodynamic state change" insight.** Hallucination is not a subtle token-level error. It manifests as a catastrophic shift in the attention spectrum: energy disperses from coherent low-frequency modes into noise. This is why a single threshold on a single layer's smoothness or entropy can catch nearly all hallucinations. The paper calls this the "Loud Liar" phenomenon for Llama 3.1 8B — its failures are spectrally catastrophic and easy to detect. Mistral 7B is quieter but still detectable (AUC 0.900). This means the guardrail is architecture-aware: you must calibrate per model, but the principle is universal.

## Step-by-Step Workflow

1. **Hook into the model's forward pass** to extract attention matrices `A[layer][head]` (shape: `[num_heads, seq_len, seq_len]`) and hidden states `X[layer]` (shape: `[seq_len, hidden_dim]`) at each layer. In HuggingFace Transformers, pass `output_attentions=True` and `output_hidden_states=True` to the model call.

2. **Select target layers** based on the model architecture. Use these empirically validated starting points:
   - Llama 3.1 8B: Layer 26 (Smoothness), Layers 0/1 (Fiedler), Layers 20/22/29 (HFER)
   - Mistral 7B: Layer 3 (Entropy)
   - Qwen 2.5 0.5B: Layer 7 (Smoothness)
   For other models, start with layers in the final third and the first few layers.

3. **Symmetrize and aggregate attention heads** for each target layer. For each head h, compute `W_h = 0.5 * (A_h + A_h.T)`. Compute attention mass per head as `s_h = W_h.sum()`. Aggregate: `W_bar = sum(alpha_h * W_h)` where `alpha_h = s_h / sum(s_all)`.

4. **Construct the graph Laplacian**: compute degree matrix `D = diag(W_bar.sum(axis=1))`, then `L = D - W_bar`. Verify L is symmetric positive semidefinite.

5. **Compute eigendecomposition** of L: `eigenvalues, eigenvectors = np.linalg.eigh(L)`. Sort eigenvalues ascending. For sequences under 200 tokens, full decomposition adds ~10-50ms. For longer sequences, use partial decomposition (`scipy.sparse.linalg.eigsh` with k smallest/largest).

6. **Extract the four spectral diagnostics**:
   - **Fiedler Value**: `lambda_2 = eigenvalues[1]`
   - **Smoothness**: `S = 1 - trace(X.T @ L @ X) / (eigenvalues[-1] * frobenius_norm(X)^2)`
   - **Spectral Entropy**: Compute Graph Fourier Transform `X_hat = U.T @ X`, then mode energies `p_m = norm(X_hat[m])^2 / sum(norms^2)`, then `SE = -sum(p_m * log(p_m))`
   - **HFER**: `sum(energies[N//2:]) / sum(energies)`

7. **Apply threshold detection**. For maximum recall (safety-critical), use a single feature:
   - Llama L26 Smoothness <= threshold -> hallucination (98.2% recall)
   - Mistral L3 Entropy >= threshold -> hallucination (94.7% recall)
   For balanced precision/recall, combine 3-5 features with OR logic (any feature triggers = hallucination).

8. **Calibrate thresholds** on a small unlabeled or lightly labeled holdout set. Generate ~100-200 tool calls, manually label a subset, and find the threshold that maximizes your target metric (recall for safety, F1 for balance). The paper uses T=0.3 sampling temperature and N=1000 for robust calibration.

9. **Wire the detector into the agent loop**: after the model generates a tool call but before executing it, run the spectral check. If flagged: retry with lower temperature, request human review, or fall back to a safer model.

10. **Monitor and log** spectral features over time to detect distribution shift, domain-specific degradation (hallucination rates can jump 2.8x between domains), and to refine thresholds in production.

## Concrete Examples

**Example 1: Adding a spectral guardrail to a LangChain agent**

User: "I'm building a LangChain agent with Llama 3.1 8B that calls external APIs. Add a hallucination guardrail that catches bad tool calls before they execute."

Approach:
1. Create a custom callback handler that intercepts the model's forward pass
2. Extract attention from layer 26 and compute smoothness
3. Insert a check between tool call generation and tool execution

Output:
```python
import numpy as np
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

class SpectralGuardrail:
    """Training-free hallucination detector using attention spectral analysis."""

    def __init__(self, model, target_layer=26, threshold=None):
        self.model = model
        self.target_layer = target_layer
        self.threshold = threshold  # Calibrate on your data
        self._attentions = {}
        self._hidden_states = {}
        self._register_hooks()

    def _register_hooks(self):
        layer = self.model.model.layers[self.target_layer]
        layer.self_attn.register_forward_hook(self._capture_attention)

    def _capture_attention(self, module, input, output):
        # output[1] contains attention weights when output_attentions=True
        self._attentions[self.target_layer] = output[1].detach().cpu()

    def compute_smoothness(self, attn_matrix, hidden_states):
        """Compute smoothness from attention topology at one layer."""
        # attn_matrix shape: [num_heads, seq_len, seq_len]
        num_heads, N, _ = attn_matrix.shape

        # Symmetrize and aggregate across heads
        W_bar = np.zeros((N, N))
        masses = []
        for h in range(num_heads):
            A_h = attn_matrix[h]
            W_h = 0.5 * (A_h + A_h.T)
            s_h = W_h.sum()
            masses.append(s_h)
        total_mass = sum(masses)
        for h in range(num_heads):
            A_h = attn_matrix[h]
            W_h = 0.5 * (A_h + A_h.T)
            W_bar += (masses[h] / total_mass) * W_h

        # Graph Laplacian
        D = np.diag(W_bar.sum(axis=1))
        L = D - W_bar
        eigenvalues, _ = np.linalg.eigh(L)
        lambda_max = eigenvalues[-1]

        # Smoothness: S = 1 - tr(X^T L X) / (lambda_max * ||X||_F^2)
        X = hidden_states  # shape: [seq_len, hidden_dim]
        numerator = np.trace(X.T @ L @ X)
        denominator = lambda_max * np.linalg.norm(X, 'fro') ** 2
        smoothness = 1.0 - numerator / denominator
        return smoothness

    def check(self, input_ids, hidden_states):
        """Returns True if generation appears hallucinated."""
        attn = self._attentions[self.target_layer].numpy()[0]  # first batch
        X = hidden_states[self.target_layer].numpy()[0]
        smoothness = self.compute_smoothness(attn, X)
        return smoothness <= self.threshold  # Low smoothness = hallucination
```

**Example 2: Multi-feature detector for balanced precision and recall**

User: "I need fewer false positives than single-feature detection. Implement a multi-feature spectral detector for Llama 3.1 8B targeting finance domain tool calls."

Approach:
1. Extract features from three layers: L30 Fiedler, L1 Entropy, L25 HFER
2. Apply calibrated thresholds to each feature independently
3. Flag as hallucination only if at least 2 of 3 features trigger (majority vote)

Output:
```python
class MultiFeatureGuardrail:
    """Balanced hallucination detection using 3 spectral features."""

    FEATURE_CONFIG = {
        'fiedler_L30': {'layer': 30, 'metric': 'fiedler', 'direction': 'below'},
        'entropy_L1': {'layer': 1, 'metric': 'entropy', 'direction': 'above'},
        'hfer_L25': {'layer': 25, 'metric': 'hfer', 'direction': 'above'},
    }

    def __init__(self, thresholds: dict, vote_threshold: int = 2):
        self.thresholds = thresholds  # {'fiedler_L30': 0.05, 'entropy_L1': 3.2, ...}
        self.vote_threshold = vote_threshold

    def compute_all_features(self, attentions, hidden_states):
        results = {}
        for name, cfg in self.FEATURE_CONFIG.items():
            attn = attentions[cfg['layer']]
            X = hidden_states[cfg['layer']]
            W_bar, L, eigenvalues, eigenvectors = self._spectral_decompose(attn)

            if cfg['metric'] == 'fiedler':
                results[name] = eigenvalues[1]
            elif cfg['metric'] == 'entropy':
                results[name] = self._spectral_entropy(eigenvectors, X)
            elif cfg['metric'] == 'hfer':
                results[name] = self._hfer(eigenvectors, X)
        return results

    def check(self, attentions, hidden_states):
        features = self.compute_all_features(attentions, hidden_states)
        votes = 0
        for name, value in features.items():
            cfg = self.FEATURE_CONFIG[name]
            thresh = self.thresholds[name]
            if cfg['direction'] == 'above' and value >= thresh:
                votes += 1
            elif cfg['direction'] == 'below' and value <= thresh:
                votes += 1
        is_hallucination = votes >= self.vote_threshold
        return is_hallucination, features  # Return features for logging

    # ... _spectral_decompose, _spectral_entropy, _hfer methods as above
```

This achieves ~86% recall with ~81% precision on the finance domain.

**Example 3: Comparing spectral signatures across models**

User: "I want to evaluate whether Llama or Mistral is safer for my agent's tool use. Compare their hallucination spectral profiles."

Approach:
1. Generate a matched evaluation set: N=1000 tool calls per model, T=0.3, same domain
2. Extract spectral features from both models at their optimal layers
3. Compute AUC, recall, precision for each model's best single-feature detector
4. Visualize the spectral distributions for hallucinated vs correct calls

Output:
```python
from sklearn.metrics import roc_auc_score, precision_recall_curve

def compare_models(llama_features, mistral_features, llama_labels, mistral_labels):
    """Compare hallucination detectability across models."""
    # Llama L26 Smoothness (lower = hallucination)
    llama_scores = -llama_features['smoothness_L26']  # negate so higher = hallucination
    llama_auc = roc_auc_score(llama_labels, llama_scores)

    # Mistral L3 Entropy (higher = hallucination)
    mistral_scores = mistral_features['entropy_L3']
    mistral_auc = roc_auc_score(mistral_labels, mistral_scores)

    print(f"Llama 3.1 8B  - AUC: {llama_auc:.3f} (Loud Liar: spectrally catastrophic failures)")
    print(f"Mistral 7B    - AUC: {mistral_auc:.3f} (Quieter but better discrimination)")
    # Expected: Llama ~0.845, Mistral ~0.900
```

## Best Practices

- **Do:** Start with single-feature detection at the paper's recommended layers (Llama L26 Smoothness, Mistral L3 Entropy) before tuning multi-feature configurations. Single features are simpler to debug and already achieve >94% recall.
- **Do:** Always calibrate thresholds on data from your specific domain. Hallucination rates vary dramatically across domains (e.g., 21.7% general vs 61.3% finance for Llama), and so do optimal thresholds.
- **Do:** Log all spectral features alongside tool call results in production. This creates a free dataset for future threshold refinement and lets you detect distribution shift.
- **Do:** Use low sampling temperature (T=0.3) during calibration to match the paper's evaluation conditions. Higher temperatures change the spectral landscape.
- **Avoid:** Assuming thresholds transfer across models. Each architecture has a different "spectral fingerprint" for hallucination. Llama's failures are loud; Mistral's are subtle.
- **Avoid:** Running full eigendecomposition on sequences longer than ~500 tokens without switching to partial decomposition (`scipy.sparse.linalg.eigsh`). The O(N^3) cost becomes significant.
- **Avoid:** Using this as the sole safety mechanism. Spectral guardrails catch catastrophic hallucinations (attention topology collapse) but may miss subtle factual errors where the model's reasoning structure remains coherent.

## Error Handling

- **Eigendecomposition fails or produces NaN**: This can happen if the Laplacian has numerical issues from very sparse attention. Add a small epsilon to the diagonal (`L += eps * I`) before decomposition. If attention is all zeros for a head, exclude that head from aggregation.
- **Feature values outside expected range**: Smoothness should be in [0, 1], Fiedler >= 0, HFER in [0, 1], Entropy >= 0. Values outside these ranges indicate a bug in attention extraction or aggregation. Verify the attention matrices sum to 1 along the correct axis.
- **High false positive rate in production**: The single-feature maximum-recall configuration trades precision for recall. Switch to multi-feature majority voting (Example 2) or raise the detection threshold to reduce false positives.
- **Model architecture doesn't expose attention weights**: Some quantized or optimized inference runtimes (e.g., vLLM with certain backends) don't return attention matrices. You'll need to either use a runtime that supports `output_attentions=True` or register forward hooks on the attention module directly.
- **Domain shift degrades detection**: If moving to a new domain, re-run calibration with ~200 examples. The spectral features are robust across domains but thresholds shift.

## Limitations

- **Requires access to attention matrices**: This method cannot be applied to black-box API models (OpenAI, Anthropic, Google). It requires open-weight models where you control inference.
- **Validated on 7B-8B scale**: The paper tests Llama 3.1 8B, Mistral 7B, and Qwen 2.5 0.5B. Behavior at 70B+ scale is unknown and the optimal layers will differ.
- **Catches structural hallucination, not factual errors**: If the model confidently generates a wrong but structurally plausible tool call (e.g., a real API endpoint with the wrong parameter value), spectral features may not flag it because the attention topology remains coherent.
- **Latency overhead for long sequences**: Full eigendecomposition is O(N^3). For tool calls (typically <200 tokens), this is negligible (~10-50ms). For long-context scenarios, partial decomposition or sampling-based approximations are needed.
- **Threshold calibration requires some labeled data**: While the method itself is training-free, finding optimal thresholds benefits from at least a small labeled validation set (50-200 examples).

## Reference

[Spectral Guardrails for Agents in the Wild: Detecting Tool Use Hallucinations via Attention Topology](https://arxiv.org/abs/2602.08082v1) (Noel, 2026). Focus on Section 3 for the four spectral diagnostics, Section 4 for cross-model "Loud Liar" results, and the appendix tables for per-layer feature performance across all model/domain combinations.