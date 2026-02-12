---
name: "raca-representation-aware-coverage-criteria"
description: "Evaluate and improve LLM safety test suites using representation-aware coverage criteria. Implements the RACA framework for measuring jailbreak test quality, prioritizing attack prompts, and sampling effective adversarial inputs. Use when asked to: 'assess safety test coverage', 'prioritize jailbreak prompts', 'evaluate attack test suite quality', 'filter redundant safety tests', 'measure LLM safety coverage', 'sample effective adversarial prompts'."
---

# RACA: Representation-Aware Coverage Criteria for LLM Safety Testing

This skill enables Claude to apply the RACA framework for systematically evaluating and improving the quality of LLM safety test suites. Rather than treating safety testing as ad-hoc prompt collection, RACA uses representation engineering to identify safety-critical directions in a model's activation space, then measures how thoroughly a test suite exercises those directions across six complementary coverage sub-criteria. This turns safety testing from "do we have enough prompts?" into "do our prompts cover the safety-relevant concept space?"

## When to Use

- When the user has a collection of jailbreak or adversarial prompts and wants to measure how comprehensive they are
- When prioritizing which prompts from a large candidate pool are most valuable for safety evaluation
- When building a safety test suite and needing to identify gaps in coverage of safety-critical concepts
- When comparing two test sets to determine which provides better safety evaluation coverage
- When sampling a small but effective subset of attack prompts from a larger corpus
- When implementing automated safety testing pipelines that need quality metrics beyond simple count or success rate
- When auditing whether a red-teaming exercise has adequately probed a model's safety boundaries

## Key Technique

RACA's core insight is that LLM safety behavior is governed by a low-dimensional subspace of the model's internal representations, not by the full neuron-level activation space. Traditional neural network coverage criteria (neuron coverage, k-multisection coverage) fail on LLMs because they operate on raw neurons -- too high-dimensional, too noisy, and unable to distinguish safety-relevant activations from irrelevant ones. RACA solves this by using **representation engineering**: extracting a small calibration set's activations from middle layers, then applying PCA to find the principal directions along which harmful vs. safe inputs diverge. These directions become "safety concepts" -- a compressed, interpretable basis for measuring test coverage.

The framework computes **conceptual activation scores** for each test input by projecting its hidden-state activations onto these safety concept directions: `f_j(x) = v_j^T(h(x) - mu)`, where `v_j` is the j-th principal component, `h(x)` is the layer activation, and `mu` is the calibration mean. This score quantifies how strongly a test input activates each safety concept. Coverage is then measured across two dimensions: **individual concept coverage** (do test inputs activate each safety feature sufficiently?) and **compositional concept coverage** (do test inputs explore combinations and boundaries between safety concepts?).

The six sub-criteria work together: Safety Feature Coverage (SFC) checks breadth; Top-K Feature Coverage (TKFC) checks dominance; Feature Intensity Coverage (FIC) checks depth across intensity ranges; Semantic Cluster Coverage (SCC) checks distribution across concept clusters; Pairwise Concept Coverage (PCC) checks feature co-activation patterns; and Cluster Boundary Coverage (CBC) checks exploration of ambiguous regions between clusters. A test suite scoring well across all six criteria is genuinely comprehensive.

## Step-by-Step Workflow

1. **Construct the calibration set.** Curate 20-100 representative jailbreak prompts covering the target safety categories (e.g., violence, fraud, hate speech, self-harm). These must be expert-verified as genuinely harmful requests. Balance across categories. Even 20 well-chosen prompts yield acceptable calibration.

2. **Extract hidden-state activations.** Run each calibration prompt through the target LLM and capture activations from middle layers (layers 15-18 for 7B models, roughly the 47-56% depth range). Average across the selected layers to get one activation vector per prompt. Also extract activations for a matched set of benign prompts.

3. **Compute centered activation vectors.** For each calibration prompt, subtract the mean activation `mu` across the full calibration set (harmful + benign). This centering isolates the safety-relevant signal from the baseline activation pattern.

4. **Apply PCA to extract safety concepts.** Run PCA on the centered activation vectors. Retain the top-n principal components (default n=64). Each component `v_j` represents a "safety concept direction" -- a linear direction in activation space that captures systematic variation between safe and unsafe inputs.

5. **Score the test suite.** For each test prompt in the suite under evaluation, extract its hidden-state activation `h(x)` from the same layers, then compute conceptual activation scores: `f_j(x) = v_j^T(h(x) - mu)` for each of the n safety concepts. This produces an n-dimensional safety profile per test prompt.

6. **Compute individual concept coverage (Dimension I).**
   - **SFC**: Count how many safety features are activated above threshold `epsilon=5.0` by at least one test case. Report as fraction of total features.
   - **TKFC**: For each test case, find its Top-K (K=10) most activated features. Count how many features appear in at least one test case's Top-K. Report as fraction.
   - **FIC**: Discretize each feature's activation range into K=10 equal-width bins. For each feature, count bins covered by at least one test case. Average across features.

7. **Compute compositional concept coverage (Dimension II).**
   - **SCC**: Apply K-means clustering (M=32 centroids) to the activation score vectors. Count how many clusters contain at least one test case. Report as fraction.
   - **PCC**: For each pair of safety features (i,j), check if any test case activates both above threshold `epsilon=2.5`. Report as fraction of all pairs covered.
   - **CBC**: Identify test cases in low-density regions between cluster centroids (distance to nearest centroid exceeds delta=8.0). Report as fraction of test cases exploring boundaries.

8. **Aggregate and interpret results.** Report all six sub-criteria individually. No single metric suffices: SFC/TKFC show breadth, FIC shows depth, SCC shows distributional coverage, PCC shows combinatorial coverage, CBC shows boundary exploration. Flag any sub-criterion below 0.5 as a coverage gap.

9. **Apply to downstream tasks.** Use the scores for:
   - **Test prioritization**: Rank prompts by marginal coverage gain; greedily select prompts that maximize coverage improvement.
   - **Attack sampling**: From a mixed pool, select prompts with highest activation scores on under-covered safety concepts.
   - **Gap analysis**: Identify which safety concepts or clusters lack coverage and generate targeted prompts for those gaps.

10. **Validate and iterate.** Re-run coverage computation after adding new prompts. Confirm that coverage metrics improve and that added prompts correspond to genuinely novel safety scenarios, not redundant variations.

## Concrete Examples

**Example 1: Evaluating a jailbreak test suite's coverage**

```
User: I have 500 jailbreak prompts for testing Llama-2-7B safety. How do I know
if they're comprehensive enough?

Approach:
1. Select 50 expert-curated calibration prompts spanning violence, fraud,
   hate speech, CSAM, self-harm, and illegal activity categories
2. Extract activations from layers 15-18 of Llama-2-7B for calibration set
3. PCA to get 64 safety concept directions
4. Score all 500 test prompts against these 64 directions
5. Compute all six RACA sub-criteria

Output:
  Coverage Report for test_suite_500.jsonl
  ─────────────────────────────────────────
  Individual Concept Coverage:
    SFC  (Safety Feature Coverage):    0.89  ✓ (58/64 features activated)
    TKFC (Top-K Feature Coverage):     0.73  ✓ (47/64 features dominant)
    FIC  (Feature Intensity Coverage):  0.41  ✗ (shallow intensity range)

  Compositional Concept Coverage:
    SCC  (Semantic Cluster Coverage):   0.81  ✓ (26/32 clusters visited)
    PCC  (Pairwise Concept Coverage):   0.34  ✗ (low co-activation diversity)
    CBC  (Cluster Boundary Coverage):   0.12  ✗ (few boundary-region probes)

  Gaps identified:
  - Features 22, 31, 44-48, 59 never activated (likely under-tested concepts)
  - Clusters 4, 11, 19, 27, 29, 30 unvisited
  - Intensity coverage is shallow: most prompts activate at similar magnitudes
  - Very few prompts explore concept boundaries (ambiguous/novel attacks)

  Recommendation: Add prompts targeting unvisited clusters and boundary
  regions. Vary attack intensity (subtle vs. overt harmful requests).
```

**Example 2: Prioritizing prompts from a large candidate pool**

```
User: I scraped 10,000 jailbreak prompts from online sources. I can only test
200 due to budget constraints. Which 200 should I pick?

Approach:
1. Compute RACA safety concept directions using calibration set
2. Score all 10,000 candidates
3. Greedy selection: start empty, iteratively add the prompt that maximizes
   marginal coverage gain across all six sub-criteria
4. Stop at 200 prompts

Output:
  Prioritized subset: prioritized_200.jsonl

  Coverage comparison:
                    Random-200    RACA-Prioritized-200
  SFC                 0.61             0.92
  TKFC                0.42             0.81
  FIC                 0.28             0.67
  SCC                 0.47             0.88
  PCC                 0.15             0.52
  CBC                 0.06             0.31

  The RACA-prioritized set covers 50-200% more of the safety concept space
  than a random sample of equal size. Prompts selected span 29/32 clusters
  versus 15/32 for random selection.
```

**Example 3: Implementing RACA scoring in a safety testing pipeline**

```
User: I want to add RACA coverage metrics to our CI safety testing pipeline.

Approach:
1. Pre-compute and cache calibration artifacts (PCA components, mean vector,
   cluster centroids) as a one-time setup step
2. In the pipeline, load cached artifacts and score new test prompts
3. Compute coverage, fail the pipeline if any sub-criterion drops below threshold

Output (Python pseudocode):

  import numpy as np
  from sklearn.decomposition import PCA
  from sklearn.cluster import KMeans

  class RACAEvaluator:
      def __init__(self, model, layers=[15,16,17,18], n_components=64):
          self.model = model
          self.layers = layers
          self.n_components = n_components

      def calibrate(self, harmful_prompts, benign_prompts):
          """Stage 1: Extract safety concept directions."""
          activations = []
          for prompt in harmful_prompts + benign_prompts:
              h = self._extract_avg_activation(prompt)
              activations.append(h)
          activations = np.array(activations)
          self.mu = activations.mean(axis=0)
          centered = activations - self.mu
          self.pca = PCA(n_components=self.n_components).fit(centered)
          self.components = self.pca.components_  # (n_components, hidden_dim)
          # Fit clusters on calibration scores
          cal_scores = centered @ self.components.T
          self.kmeans = KMeans(n_clusters=32).fit(cal_scores)

      def score(self, test_prompts):
          """Stage 2: Compute conceptual activation scores."""
          scores = []
          for prompt in test_prompts:
              h = self._extract_avg_activation(prompt)
              s = self.components @ (h - self.mu)  # (n_components,)
              scores.append(s)
          return np.array(scores)

      def compute_coverage(self, scores, eps_sfc=5.0, eps_pcc=2.5,
                           k_topk=10, n_bins=10, delta_cbc=8.0):
          """Stage 3: Compute six RACA sub-criteria."""
          n_tests, n_feat = scores.shape
          # SFC
          sfc = np.mean(np.any(scores > eps_sfc, axis=0))
          # TKFC
          topk_mask = np.zeros_like(scores, dtype=bool)
          for i in range(n_tests):
              topk_idx = np.argsort(scores[i])[-k_topk:]
              topk_mask[i, topk_idx] = True
          tkfc = np.mean(np.any(topk_mask, axis=0))
          # FIC
          fic_scores = []
          for j in range(n_feat):
              col = scores[:, j]
              bins = np.linspace(col.min(), col.max(), n_bins + 1)
              covered = len(set(np.digitize(col, bins)))
              fic_scores.append(covered / n_bins)
          fic = np.mean(fic_scores)
          # SCC
          labels = self.kmeans.predict(scores)
          scc = len(set(labels)) / self.kmeans.n_clusters
          # PCC
          active = scores > eps_pcc
          pairs_covered = 0
          total_pairs = n_feat * (n_feat - 1) // 2
          for i in range(n_feat):
              for j in range(i+1, n_feat):
                  if np.any(active[:, i] & active[:, j]):
                      pairs_covered += 1
          pcc = pairs_covered / total_pairs
          # CBC
          dists = self.kmeans.transform(scores).min(axis=1)
          cbc = np.mean(dists > delta_cbc)
          return {"SFC": sfc, "TKFC": tkfc, "FIC": fic,
                  "SCC": scc, "PCC": pcc, "CBC": cbc}
```

## Best Practices

- **Do:** Use a calibration set balanced across safety categories. Imbalanced calibration skews PCA toward over-represented categories, producing biased safety concepts.
- **Do:** Extract activations from middle layers (47-56% of model depth). Early layers capture syntax, late layers capture output formatting -- neither is ideal for safety semantics.
- **Do:** Cache PCA components and cluster centroids. Recalibration is only needed when the target model changes, not when test suites change.
- **Do:** Examine all six sub-criteria independently. A suite can score high on SFC (breadth) but fail on PCC (combinatorial) or CBC (boundary exploration).
- **Avoid:** Using fewer than 20 calibration prompts. Below this threshold, PCA components become unstable and coverage scores unreliable.
- **Avoid:** Applying RACA scores from one model family to a different model family. Safety representations are model-specific; recalibrate when switching target models.
- **Avoid:** Treating RACA as a binary pass/fail metric. Use it as a diagnostic tool: low scores on specific sub-criteria indicate where to invest in additional test generation.

## Error Handling

- **Calibration set too homogeneous**: If PCA variance is concentrated in 1-2 components (first component explains >80%), the calibration prompts likely lack diversity. Add prompts from under-represented safety categories.
- **All activation scores near zero**: The selected layers may not encode safety-relevant information for this model architecture. Sweep across layers at 10% depth intervals to find the optimal extraction point.
- **Cluster boundary coverage consistently zero**: The delta threshold may be too high for the model's activation scale. Normalize scores to unit variance before computing CBC, or reduce delta proportionally.
- **PCC computation is slow**: Pairwise feature coverage is O(n_features^2 * n_tests). For large feature sets, sample feature pairs or use vectorized matrix operations rather than nested loops.
- **Coverage metrics don't improve with more prompts**: The test suite may be saturated in easy regions. Use CBC and FIC to identify under-explored intensity ranges and boundary areas, then generate targeted prompts.

## Limitations

- RACA requires white-box access to model activations. It cannot be applied to closed-source APIs (GPT-4, Claude) where hidden states are not exposed.
- The framework is validated on 7B-13B parameter models. Scaling behavior to 70B+ models is unvalidated -- middle-layer identification and PCA dimensionality may need adjustment.
- Calibration quality directly bounds coverage quality. If the calibration set misses an entire harm category, RACA will not detect that gap.
- RACA measures test suite coverage of the safety concept space, not attack success rate. High coverage does not guarantee high jailbreak success -- it guarantees thorough probing.
- The six sub-criteria have fixed default thresholds (epsilon, delta, K) tuned on specific models. These may need recalibration for different model families or safety definitions.

## Reference

- **Paper**: [RACA: Representation-Aware Coverage Criteria for LLM Safety Testing](https://arxiv.org/abs/2602.02280v1) (Wei et al., 2026). Focus on Sections 3-4 for the formal definitions of all six sub-criteria and Section 5 for experimental validation of test prioritization and attack sampling applications.