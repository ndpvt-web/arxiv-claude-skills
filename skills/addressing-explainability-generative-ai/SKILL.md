---
name: "addressing-explainability-generative-ai"
description: "Explain generative AI outputs using the gSMILE perturbation-based attribution framework. Builds local surrogate models from controlled input perturbations and Wasserstein distance to produce token-level or word-level importance scores for LLM and diffusion model outputs. Triggers: 'explain why the model generated this', 'token attribution for prompt', 'which words in my prompt matter most', 'interpret generative model output', 'build explainability for my LLM pipeline', 'debug prompt influence on generation'"
---

# gSMILE: Perturbation-Based Explainability for Generative AI

This skill enables Claude to implement the gSMILE (generative Statistical Model-agnostic Interpretability with Local Explanations) framework for explaining why generative models — LLMs and text-to-image diffusion models — produce specific outputs. The core technique treats any generative model as a black box, systematically perturbs input tokens, measures output distribution shifts via Wasserstein distance, and fits a weighted local linear surrogate whose coefficients directly yield per-token importance scores. The result is actionable attribution heatmaps that answer "which parts of my prompt drove this output?"

## When to Use

- When the user asks "why did the model generate this response?" and needs a quantitative, reproducible answer beyond attention visualization
- When building an explainability layer around an LLM API (OpenAI, Anthropic, local models) to audit prompt sensitivity
- When the user wants to debug a text-to-image pipeline by identifying which instruction words drive visual changes
- When implementing fairness or bias audits that require token-level attribution scores across demographic prompt variants
- When the user needs model-agnostic explainability that works without access to model internals (weights, gradients, attention)
- When evaluating prompt engineering choices by quantifying each token's contribution to the generated output

## Key Technique

**gSMILE** extends the SMILE interpretability method from classification to generative settings. Traditional explainability methods like LIME perturb inputs and measure changes in class probabilities — a scalar target. Generative models produce high-dimensional outputs (token sequences, images), so gSMILE replaces scalar probability shifts with **Wasserstein distance** between output distributions. Given an original prompt `x` and a perturbed variant `x̂ⱼ`, the output-level semantic shift `Δ(x, x̂ⱼ) = W(π(y|x), π(y|x̂ⱼ))` captures how much the full output distribution moved. This is the key innovation: measuring distributional shift rather than point predictions.

The perturbations are weighted by proximity to the original input using a **Gaussian kernel**: `wⱼ = exp(-δⱼ² / σ²)`, where `δⱼ` is the input-space distance between original and perturbed prompts. This ensures that the surrogate model prioritizes behavior near the original operating point. A **weighted linear regression** is then fit: `hθ(zⱼ) ≈ Δ(x, x̂ⱼ)`, where `zⱼ` is a binary feature vector indicating token presence/absence. The resulting coefficients `θ` are the attribution scores — positive means the token pushes the output in its current direction, negative means it suppresses it, and magnitude indicates strength.

This approach is **model-agnostic** (requires only API access), **local** (explains one prompt at a time), and **statistically grounded** (Lipschitz smoothness assumptions justify the linear approximation in a local neighborhood). It works for any generative model that accepts text input and produces a scorable output.

## Step-by-Step Workflow

1. **Define the explanation target.** Identify the specific prompt `x` and the generative model to explain. Record the original output `y₀ = model(x)` as the baseline. For LLMs, store the full token probability distribution or the output text; for image models, store the generated image embedding.

2. **Tokenize and build the feature space.** Split the prompt into `N` interpretable units (tokens, words, or phrases). Create a binary feature vector template `z ∈ {0,1}^N` where `zᵢ = 1` means token `i` is present.

3. **Generate J perturbations.** For each perturbation `j = 1..J` (typically J = 100–500), create a masked variant `x̂ⱼ` by randomly dropping or replacing a subset of tokens. Record each perturbation's binary feature vector `zⱼ`. Use a dropout rate of 10–30% per perturbation to balance signal and locality.

4. **Collect perturbed outputs.** Pass each `x̂ⱼ` through the generative model to obtain output `ŷⱼ`. For LLMs, capture the output token probabilities or full generated text. For image models, capture the generated image or its CLIP embedding.

5. **Compute output-level distances.** For each perturbation, calculate `Δⱼ = W(y₀, ŷⱼ)` using Wasserstein distance. For text: use token probability distribution divergence or embedding cosine distance. For images: use CLIP embedding L2 distance or LPIPS perceptual distance.

6. **Compute input-level distances and Gaussian weights.** Calculate `δⱼ` as the Hamming distance (or cosine distance) between the original feature vector and `zⱼ`. Apply the Gaussian kernel: `wⱼ = exp(-δⱼ² / σ²)`. Tune `σ` so that roughly 60–80% of perturbations receive meaningful weight.

7. **Fit the weighted linear surrogate.** Solve the weighted least squares problem: `θ* = argmin_θ Σⱼ wⱼ · (hθ(zⱼ) - Δⱼ)²`, where `hθ(z) = θ · z + θ₀`. Use `sklearn.linear_model.LinearRegression` with sample weights, or `numpy.linalg.lstsq` on the weight-scaled system.

8. **Extract and normalize attribution scores.** The coefficient vector `θ*` contains per-token importance. Normalize to `[0, 1]` range for heatmap visualization. Higher absolute values indicate stronger influence. Sign indicates direction: positive means the token's presence increases output divergence from a null baseline.

9. **Visualize as attribution heatmaps.** Map normalized scores back to the original prompt tokens. Render as a color-coded heatmap (red = high importance, blue = low) using matplotlib, HTML spans with background colors, or a terminal-based display.

10. **Validate with fidelity and stability metrics.** Compute weighted MSE between surrogate predictions and actual distances (fidelity). Compute Jaccard similarity of top-K attributed tokens across repeated runs (stability). Flag explanations with fidelity R² < 0.7 as unreliable.

## Concrete Examples

**Example 1: Explaining an LLM response to a factual question**

User: "I prompted GPT with 'What is the capital of France and why is it historically significant?' and got a long answer. Which parts of my prompt drove the response?"

Approach:
1. Tokenize prompt into: `["What", "is", "the", "capital", "of", "France", "and", "why", "is", "it", "historically", "significant", "?"]`
2. Generate 200 perturbations by randomly masking 2–4 tokens each
3. Query the LLM API for each perturbed prompt, collect output texts
4. Compute sentence-transformer cosine distance between original and perturbed outputs
5. Fit weighted linear model with Gaussian kernel (σ = 0.5 * sqrt(N))

Output:
```
Token Attribution Scores (normalized 0–1):
  What              ██░░░░░░░░  0.21
  is                ░░░░░░░░░░  0.03
  the               ░░░░░░░░░░  0.02
  capital           ████████░░  0.78
  of                ░░░░░░░░░░  0.04
  France            ██████████  0.95
  and               █░░░░░░░░░  0.08
  why               ██████░░░░  0.61
  is                ░░░░░░░░░░  0.02
  it                ░░░░░░░░░░  0.03
  historically      ████████░░  0.82
  significant       ██████░░░░  0.58
  ?                 ░░░░░░░░░░  0.01

Surrogate fidelity R²: 0.87
Top drivers: "France" (0.95), "historically" (0.82), "capital" (0.78)
```

**Example 2: Debugging a text-to-image prompt**

User: "My Stable Diffusion prompt 'A serene Japanese garden at sunset with cherry blossoms and a stone bridge' keeps generating images without the bridge. Help me understand which words matter."

Approach:
1. Tokenize into 12 interpretable words
2. Generate 300 perturbations, dropping 1–3 words each
3. Generate images for each perturbation via the diffusion API
4. Compute CLIP embedding distances between original and perturbed images
5. Fit surrogate with Gaussian weighting

Output:
```
Token Attribution Scores:
  A                 ░░░░░░░░░░  0.01
  serene            ███░░░░░░░  0.29
  Japanese          ████████░░  0.81
  garden            █████████░  0.88
  at                ░░░░░░░░░░  0.02
  sunset            ██████░░░░  0.63
  with              ░░░░░░░░░░  0.03
  cherry            ██████░░░░  0.59
  blossoms          █████░░░░░  0.52
  and               ░░░░░░░░░░  0.01
  a                 ░░░░░░░░░░  0.01
  stone             █░░░░░░░░░  0.11
  bridge            ██░░░░░░░░  0.14

Insight: "stone" (0.11) and "bridge" (0.14) have very low attribution,
meaning the model largely ignores them. "Japanese garden" dominates.
Recommendation: Move "stone bridge" to the beginning of the prompt
or increase its weight with prompt syntax like "(stone bridge:1.5)".
```

**Example 3: Implementing gSMILE as a Python module**

User: "Build me a reusable explainability module for my LLM API wrapper."

```python
import numpy as np
from sklearn.linear_model import LinearRegression
from sentence_transformers import SentenceTransformer

class GSMILEExplainer:
    def __init__(self, model_fn, n_perturbations=200, dropout_rate=0.2, sigma=None):
        """
        model_fn: callable that takes a string prompt and returns output text
        """
        self.model_fn = model_fn
        self.n_perturbations = n_perturbations
        self.dropout_rate = dropout_rate
        self.sigma = sigma
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')

    def explain(self, prompt: str) -> dict:
        tokens = prompt.split()
        n = len(tokens)
        sigma = self.sigma or (0.5 * np.sqrt(n))

        # Step 1: Get baseline output
        baseline_output = self.model_fn(prompt)
        baseline_emb = self.embedder.encode([baseline_output])[0]

        # Step 2: Generate perturbations
        Z = np.ones((self.n_perturbations, n))  # feature vectors
        deltas = np.zeros(self.n_perturbations)  # output distances
        input_dists = np.zeros(self.n_perturbations)

        for j in range(self.n_perturbations):
            mask = np.random.random(n) > self.dropout_rate
            Z[j] = mask.astype(float)
            perturbed_tokens = [t for t, m in zip(tokens, mask) if m]
            perturbed_prompt = " ".join(perturbed_tokens) if perturbed_tokens else tokens[0]

            perturbed_output = self.model_fn(perturbed_prompt)
            perturbed_emb = self.embedder.encode([perturbed_output])[0]

            deltas[j] = np.linalg.norm(baseline_emb - perturbed_emb)
            input_dists[j] = np.sum(1 - mask) / n  # normalized Hamming

        # Step 3: Gaussian kernel weights
        weights = np.exp(-(input_dists ** 2) / (sigma ** 2))

        # Step 4: Fit weighted linear surrogate
        reg = LinearRegression()
        reg.fit(Z, deltas, sample_weight=weights)

        # Step 5: Extract attributions
        raw_scores = np.abs(reg.coef_)
        max_score = raw_scores.max() if raw_scores.max() > 0 else 1.0
        normalized = raw_scores / max_score

        return {
            "tokens": tokens,
            "attributions": normalized.tolist(),
            "coefficients": reg.coef_.tolist(),
            "fidelity_r2": reg.score(Z, deltas, sample_weight=weights),
            "top_tokens": sorted(
                zip(tokens, normalized), key=lambda x: -x[1]
            )[:5],
        }
```

## Best Practices

**Do:**
- Use at least 150–300 perturbations for stable attribution scores. Fewer leads to noisy, unreliable coefficients.
- Tune the kernel width `σ` relative to feature dimensionality. A good starting point is `σ = 0.5 * sqrt(N)` where N is the token count.
- Validate every explanation with fidelity (R² of surrogate) before presenting it. Discard results with R² below 0.7.
- Cache model outputs aggressively — the perturbation step is the computational bottleneck, and many perturbations may produce similar outputs.
- Use semantic distance (embeddings) rather than string-level metrics like edit distance for measuring output shifts. Embedding distances capture meaning changes that surface metrics miss.

**Avoid:**
- Do not perturb more than 30% of tokens per sample — extreme perturbations push outside the local neighborhood where the linear assumption holds.
- Do not use this method for single-token prompts or prompts shorter than 5 tokens — insufficient perturbation space produces degenerate surrogates.
- Do not interpret small coefficient differences as meaningful. Report confidence by running the explanation 3–5 times and averaging; treat tokens with high variance across runs as unreliable.
- Do not confuse correlation with causation — gSMILE measures associative influence under perturbation, not true causal effect. Correlated tokens (e.g., "cherry" and "blossoms") may share attribution.

## Error Handling

| Problem | Symptom | Solution |
|---------|---------|----------|
| Low fidelity (R² < 0.5) | Surrogate doesn't match model behavior | Increase perturbation count, reduce dropout rate, or narrow σ |
| All attributions near-equal | Flat coefficient vector | The prompt may be redundant or the model is insensitive — try coarser perturbation (drop whole phrases) |
| API rate limits during perturbation | Timeouts or 429 errors | Implement exponential backoff and batch perturbations; reduce J |
| Unstable scores across runs | High variance in top-K tokens | Increase J to 400+, average over 3–5 independent runs |
| Degenerate outputs from heavy masking | Model returns empty or nonsensical text | Cap dropout at 20%, ensure at least 70% of tokens survive each perturbation |
| Memory issues with image embeddings | OOM on large batches | Process perturbations in chunks of 20–50, store embeddings to disk |

## Limitations

- **Computational cost scales linearly with perturbation count and model inference cost.** For expensive models (GPT-4, large diffusion models), 300 perturbations means 300 API calls per explanation. Budget accordingly.
- **The linear surrogate assumption breaks down for highly non-linear prompt interactions.** If two tokens interact strongly (e.g., "not" + "good"), the linear model cannot capture the interaction without explicit cross-features.
- **Token-level granularity may be too fine or too coarse.** For some applications, phrase-level or sentence-level perturbation is more appropriate. The framework supports this by changing the tokenization unit.
- **Model-agnostic means model-uninformed.** Methods with gradient access (integrated gradients, attention rollout) can be more efficient when model internals are available. gSMILE is best suited for black-box API scenarios.
- **Wasserstein distance on high-dimensional output spaces can be noisy.** Embedding-based proxies (cosine distance on sentence embeddings, CLIP distance on images) are practical substitutes but introduce their own approximation error.

## Reference

[Addressing Explainability of Generative AI using SMILE](https://arxiv.org/abs/2602.01206v1) — Zeinab Dehghani, 2026. Introduces the gSMILE framework for model-agnostic explainability of generative models via contrastive perturbation, Wasserstein distance, and weighted linear surrogates. Focus on Sections 3–5 for the mathematical framework, perturbation strategies, and evaluation metrics.