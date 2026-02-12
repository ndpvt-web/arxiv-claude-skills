---
name: "halt-hallucination-assessment-log-probs"
description: >
  Detect LLM hallucinations using token log-probabilities as time series with the HALT method.
  Implements a lightweight GRU-based detector that processes top-k log-probs and entropy features
  to flag unreliable LLM outputs without needing model internals. Use this skill when the user says:
  "detect hallucinations in LLM output", "check if this generation is hallucinated",
  "build a hallucination detector from log-probs", "implement HALT for hallucination detection",
  "flag unreliable model outputs using token probabilities", "add confidence scoring to LLM pipeline".
---

# HALT: Hallucination Assessment via Log-probs as Time Series

This skill enables Claude to implement the HALT hallucination detection framework, which treats top-20 token log-probabilities from LLM generations as a time series and classifies outputs as hallucinated or faithful using a compact GRU model with entropy-derived features. HALT is 30x smaller (~5M parameters) and 60x faster than encoder-based detectors like Lettuce, while achieving superior macro-F1 across 10 task categories. It works with any LLM that exposes token-level log-probabilities via API, making it directly applicable to proprietary models.

## When to Use

- When the user wants to build a hallucination detection pipeline for LLM outputs in production
- When the user needs to add confidence/reliability scoring to an existing LLM application
- When the user asks to implement a lightweight model that flags suspect generations without access to model internals
- When the user wants to process log-probabilities from OpenAI, Anthropic, or open-source model APIs into hallucination signals
- When the user is building a guardrail system that must operate at low latency (sub-millisecond per token)
- When the user needs to benchmark hallucination detection across diverse task types (QA, summarization, code generation, chat, reasoning)
- When the user wants to extract and engineer entropy features from token probability distributions
- When the user asks about the HUB (Hallucination detection Unified Benchmark) evaluation framework

## Key Technique

HALT treats LLM generation as a stochastic process where each decoding step emits a probability distribution over tokens. At each timestep t, the top-20 token log-probabilities are extracted as a vector `l_t in R^20`, forming a matrix `L in R^{T x 20}` for a response of T tokens. The core insight is that hallucinated text exhibits distinct temporal patterns in these log-probability sequences -- sudden entropy spikes, rank inversions where the selected token is not the highest-probability choice, and sharp certainty transitions between consecutive steps all serve as signatures of unreliable generation.

Five engineered features are concatenated with the raw log-probs at each timestep: (1) **Average Log-Probability** measuring distribution sharpness, (2) **Rank Proxy** counting how many alternatives outrank the selected token, (3) **Overall Entropy** capturing uncertainty across the top-k, (4) **Alternatives-Only Entropy** measuring dispersion among competitor tokens, and (5) **Temporal Decision Delta** capturing abrupt certainty shifts between consecutive steps. These features are computed from a renormalized truncated distribution using log-sum-exp for numerical stability.

A bidirectional GRU (5 layers, hidden dim 256, ~5M parameters) processes this enriched time series. Instead of standard last-hidden-state pooling, HALT uses top-q salient-timestep pooling -- averaging the hidden states at timesteps with the largest L2 norms -- to focus on the most informative moments. A linear classification head produces a single logit trained with binary cross-entropy. Critically, HALT learns model-specific calibration bias: a detector trained on LLaMA outputs generalizes across tasks for LLaMA but does not transfer to Qwen, confirming that each model family requires its own HALT instance.

## Step-by-Step Workflow

### 1. Extract top-k log-probabilities from the target LLM

Use the LLM's API or a serving framework like vLLM to obtain the top-20 token log-probabilities at each decoding step. For OpenAI-compatible APIs, set `logprobs=True` and `top_logprobs=20`. Store as a `T x 20` matrix where row t contains `[log p_t(1), ..., log p_t(20)]` with the selected token always at index 0.

```python
import numpy as np

def extract_logprob_matrix(api_response) -> np.ndarray:
    """Extract T x 20 log-probability matrix from API response."""
    token_logprobs = api_response["choices"][0]["logprobs"]["content"]
    matrix = []
    for token_info in token_logprobs:
        top_lps = [token_info["logprob"]]  # selected token first
        for alt in token_info["top_logprobs"][:19]:
            top_lps.append(alt["logprob"])
        # Pad if fewer than 20 returned
        while len(top_lps) < 20:
            top_lps.append(-100.0)
        matrix.append(top_lps)
    return np.array(matrix, dtype=np.float32)
```

### 2. Renormalize into a truncated probability distribution

Apply log-sum-exp normalization for numerical stability. For each timestep t, compute `p_tilde_t(i) = exp(l_t(i) - m_t) / sum_j exp(l_t(j) - m_t)` where `m_t = max(l_t)`.

```python
def renormalize_logprobs(logprob_matrix: np.ndarray) -> np.ndarray:
    """Renormalize top-k log-probs into truncated probability distribution."""
    m = logprob_matrix.max(axis=1, keepdims=True)
    shifted = logprob_matrix - m
    exp_shifted = np.exp(shifted)
    return exp_shifted / exp_shifted.sum(axis=1, keepdims=True)
```

### 3. Compute the five entropy-based features per timestep

Engineer features that capture distribution shape, token ranking, and temporal dynamics:

```python
def compute_entropy_features(logprob_matrix: np.ndarray,
                              p_tilde: np.ndarray) -> np.ndarray:
    """Compute 5 entropy features for each timestep. Returns T x 5 matrix."""
    T, k = logprob_matrix.shape
    features = np.zeros((T, 5), dtype=np.float32)

    # 1. Average Log-Probability
    features[:, 0] = logprob_matrix.mean(axis=1)

    # 2. Rank Proxy: count alternatives outranking selected token
    selected = logprob_matrix[:, 0:1]
    features[:, 1] = 1.0 + (logprob_matrix[:, 1:] > selected).sum(axis=1)

    # 3. Overall Entropy: H = -sum p_tilde * log(p_tilde)
    log_p = np.log(p_tilde + 1e-12)
    features[:, 2] = -(p_tilde * log_p).sum(axis=1)

    # 4. Alternatives-Only Entropy (exclude index 0)
    p_alts = p_tilde[:, 1:]
    p_alts_norm = p_alts / (p_alts.sum(axis=1, keepdims=True) + 1e-12)
    log_p_alts = np.log(p_alts_norm + 1e-12)
    features[:, 3] = -(p_alts_norm * log_p_alts).sum(axis=1)

    # 5. Temporal Decision Delta: change in decision entropy
    h_dec = -(p_tilde[:, 0] * np.log(p_tilde[:, 0] + 1e-12))
    delta_h = np.zeros(T, dtype=np.float32)
    delta_h[1:] = h_dec[1:] - h_dec[:-1]
    features[:, 4] = delta_h

    return features
```

### 4. Concatenate raw log-probs with features to form the input tensor

Combine the `T x 20` raw log-probability matrix with the `T x 5` feature matrix to produce a `T x 25` input tensor for the GRU.

```python
def build_halt_input(logprob_matrix: np.ndarray) -> np.ndarray:
    """Build complete HALT input tensor: T x 25."""
    p_tilde = renormalize_logprobs(logprob_matrix)
    features = compute_entropy_features(logprob_matrix, p_tilde)
    return np.concatenate([logprob_matrix, features], axis=1)
```

### 5. Build the GRU classifier architecture

Implement the HALT model: LayerNorm + 2-layer MLP projection (to 128 dims), bidirectional GRU (5 layers, hidden 256, dropout 0.4), top-q salient-timestep pooling, and a linear classification head.

```python
import torch
import torch.nn as nn

class HALT(nn.Module):
    def __init__(self, input_dim=25, proj_dim=128, hidden_dim=256,
                 num_layers=5, dropout=0.4, top_q=8):
        super().__init__()
        self.top_q = top_q
        self.proj = nn.Sequential(
            nn.LayerNorm(input_dim),
            nn.Linear(input_dim, proj_dim),
            nn.GELU(),
            nn.Linear(proj_dim, proj_dim),
            nn.GELU(),
        )
        self.gru = nn.GRU(
            input_size=proj_dim, hidden_size=hidden_dim,
            num_layers=num_layers, batch_first=True,
            bidirectional=True, dropout=dropout,
        )
        self.classifier = nn.Linear(hidden_dim * 2, 1)

    def forward(self, x, lengths=None):
        # x: (B, T, 25)
        h = self.proj(x)                    # (B, T, 128)
        out, _ = self.gru(h)                # (B, T, 512)
        # Top-q salient-timestep pooling
        norms = out.norm(dim=-1)             # (B, T)
        _, top_idx = norms.topk(min(self.top_q, out.size(1)), dim=1)
        pooled = torch.stack([
            out[b, top_idx[b]].mean(dim=0) for b in range(out.size(0))
        ])
        return self.classifier(pooled).squeeze(-1)
```

### 6. Train on labeled hallucination data with binary cross-entropy

Use datasets from HUB (Chat, Data-to-Text, QA splits for training). Label 1 = hallucinated, 0 = faithful. Train one HALT instance per target LLM family.

```python
def train_halt(model, train_loader, val_loader, epochs=20, lr=1e-3):
    optimizer = torch.optim.AdamW(model.parameters(), lr=lr)
    criterion = nn.BCEWithLogitsLoss()
    best_f1 = 0.0
    for epoch in range(epochs):
        model.train()
        for batch_x, batch_y in train_loader:
            logits = model(batch_x)
            loss = criterion(logits, batch_y.float())
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
        # Validate and tune threshold on val set for best macro-F1
        val_f1 = evaluate_macro_f1(model, val_loader)
        if val_f1 > best_f1:
            best_f1 = val_f1
            torch.save(model.state_dict(), "halt_best.pt")
```

### 7. Run inference: score new LLM generations

For each generation, extract log-probs, compute features, and pass through the trained GRU. Apply the tuned decision threshold to classify as hallucinated or faithful.

```python
def detect_hallucination(model, logprob_matrix, threshold=0.5):
    """Returns (is_hallucinated: bool, confidence: float)."""
    model.eval()
    x = build_halt_input(logprob_matrix)
    x_tensor = torch.tensor(x).unsqueeze(0)  # (1, T, 25)
    with torch.no_grad():
        logit = model(x_tensor).item()
    prob = torch.sigmoid(torch.tensor(logit)).item()
    return prob > threshold, prob
```

### 8. Integrate as a guardrail in your LLM pipeline

Wire HALT into your serving stack as a post-generation filter. Flag or regenerate outputs that exceed the hallucination probability threshold.

## Concrete Examples

**Example 1: Adding hallucination detection to a RAG pipeline**

User: "I have a retrieval-augmented generation system. I want to detect when the LLM hallucinates information not in the retrieved context."

Approach:
1. Modify the generation call to request `top_logprobs=20` from the API
2. Extract the `T x 20` log-probability matrix from the response
3. Compute the 5 entropy features and concatenate with raw log-probs
4. Load a pre-trained HALT model matched to your LLM (e.g., HALT-L for LLaMA)
5. Run inference to get a hallucination probability score
6. If score exceeds threshold (tuned on your domain), either flag the response for human review or trigger re-generation with a stricter prompt

Output:
```
Response: "The company was founded in 2015 by Jane Smith..."
HALT score: 0.82 (threshold: 0.55)
Verdict: LIKELY HALLUCINATED - flagged for review
Trigger features: Rank proxy spike at tokens 12-15, entropy delta = +1.3 at "2015"
```

**Example 2: Building a HALT detector from scratch for a custom model**

User: "I'm serving Qwen 2.5-7B with vLLM and need a hallucination detector."

Approach:
1. Generate responses across diverse prompts using vLLM with `--max-logprobs 20`
2. Label responses as hallucinated/faithful (use HUB methodology or manual annotation)
3. Extract log-prob matrices and compute entropy features for each sample
4. Instantiate HALT model (`input_dim=25, hidden_dim=256, num_layers=5`)
5. Train with BCE loss, tune threshold on validation set for macro-F1
6. Deploy alongside vLLM as a sidecar scoring service

Output:
```python
# vLLM integration
from vllm import LLM, SamplingParams

llm = LLM(model="Qwen/Qwen2.5-7B")
params = SamplingParams(max_tokens=512, logprobs=20)
output = llm.generate(prompt, params)

logprob_matrix = extract_logprob_matrix_vllm(output)
is_hallucinated, score = detect_hallucination(halt_model, logprob_matrix)
```

**Example 3: Analyzing entropy patterns to debug hallucination hotspots**

User: "I want to visualize where in a generation the model is most likely hallucinating."

Approach:
1. Extract log-prob matrix and compute all 5 features per token
2. Plot temporal decision delta and overall entropy as line charts over token positions
3. Identify timesteps with high entropy combined with rank proxy > 1 (selected token was not the top choice)
4. Highlight those token spans in the generated text

Output:
```python
import matplotlib.pyplot as plt

features = compute_entropy_features(logprob_matrix,
                                     renormalize_logprobs(logprob_matrix))
fig, axes = plt.subplots(3, 1, figsize=(14, 8), sharex=True)
axes[0].plot(features[:, 2], label="Overall Entropy")
axes[0].set_ylabel("H_overall")
axes[1].plot(features[:, 1], label="Rank Proxy", color="orange")
axes[1].axhline(y=1.0, linestyle="--", color="gray")
axes[1].set_ylabel("Rank Proxy")
axes[2].plot(features[:, 4], label="Temporal Delta H", color="red")
axes[2].set_ylabel("ΔH_dec")
axes[2].set_xlabel("Token position")
plt.tight_layout()
plt.savefig("halt_analysis.png")
```

## Best Practices

- **Do:** Train a separate HALT instance per LLM family. Cross-model transfer degrades significantly (macro-F1 drops ~10-15 points) because calibration biases are model-specific.
- **Do:** Use exactly k=20 top log-probabilities. The paper validates this as the optimal truncation point in ablation studies.
- **Do:** Tune your classification threshold on a held-out validation set using macro-F1, not accuracy. Hallucination datasets are often imbalanced.
- **Do:** Include the temporal decision delta feature. Sharp certainty transitions between consecutive tokens are among the strongest hallucination signals.
- **Avoid:** Using HALT outputs trained on one model to score a different model's generations. The learned calibration bias does not transfer across model families.
- **Avoid:** Applying HALT to models that do not expose top-k log-probabilities. Some APIs only return the selected token's log-prob, which is insufficient -- you need the full top-20 distribution.
- **Avoid:** Expecting sentence-level or span-level hallucination localization from HALT alone. It produces a single sequence-level binary classification. Use the entropy feature visualizations for approximate localization.

## Error Handling

| Issue | Cause | Solution |
|-------|-------|----------|
| API returns fewer than 20 log-probs | API limitation or parameter misconfiguration | Pad missing entries with -100.0 (effectively zero probability after softmax) |
| Variable-length sequences in batches | Different response lengths | Pad sequences to max length in batch, use `lengths` parameter for proper GRU masking |
| Extremely short responses (< 5 tokens) | Terse model output | Top-q pooling with q=8 degrades; reduce q to min(q, T) dynamically |
| NaN in entropy computation | Zero probabilities in truncated distribution | Add epsilon (1e-12) to all probability values before computing log |
| Poor generalization to new task types | Training data doesn't cover task distribution | Retrain including examples from the target task cluster; HALT generalizes across tasks within one model but benefits from diverse training data |
| High false positive rate | Threshold too aggressive | Re-tune threshold on domain-specific validation set; consider task-specific thresholds |

## Limitations

- **Model-specific training required.** Each LLM family needs its own HALT detector because calibration dynamics differ. You cannot reuse a LLaMA-trained HALT for GPT or Claude outputs.
- **Requires top-k log-probs access.** Some proprietary APIs do not expose full top-20 distributions. If your API only provides the single selected token's log-prob, HALT cannot be applied.
- **Sequence-level only.** HALT produces one hallucination score per response. It does not natively identify which specific claims or spans are hallucinated.
- **Training data dependency.** Performance depends on having labeled hallucination data for your LLM. The HUB benchmark provides training data for LLaMA 3.1-8B and Qwen 2.5-7B; other models require new data collection.
- **Not a replacement for factual verification.** HALT detects statistical patterns correlated with hallucination, not factual correctness. High-confidence but factually wrong outputs can evade detection if the model's probability distribution doesn't exhibit hallucination signatures.
- **Greedy/near-greedy decoding assumed.** The method is validated on standard sampling; highly creative or high-temperature generations may shift the distribution patterns HALT relies on.

## Reference

**Paper:** [HALT: Hallucination Assessment via Log-probs as Time series](https://arxiv.org/abs/2602.02888v1) (Shapiro, Taneja, Goel, 2026)

Key sections to read: Section 3 for feature engineering formulas and the five entropy metrics; Section 4 for GRU architecture and top-q pooling; Table 2 for HUB benchmark results across all 10 task categories; Table 4 for cross-task and cross-model transfer results that justify model-specific training.