---
name: "accelerating-social-science-research"
description: "Implement the EXPERIGEN agentic framework for automated hypothesis generation and empirical validation on datasets. Uses a Bayesian-optimization-inspired Generator-Experimenter loop to discover statistically significant, novel hypotheses from data. Trigger phrases: 'generate hypotheses from this dataset', 'discover patterns in social data', 'run EXPERIGEN on this data', 'automated hypothesis testing', 'find significant predictors in this dataset', 'data-driven hypothesis discovery'."
---

# EXPERIGEN: Agentic Hypothesis Generation and Experimentation

This skill enables Claude to apply the EXPERIGEN framework — a two-phase agentic system that automates end-to-end scientific discovery on structured datasets. The framework pairs a **Generator** agent (which proposes candidate hypotheses guided by Bayesian optimization principles) with an **Experimenter** agent (which operationalizes and statistically validates each hypothesis). By iterating between exploration of novel hypotheses and local refinement of promising ones, EXPERIGEN discovers 2-4x more statistically significant hypotheses than standard approaches while controlling for spurious findings through confounder analysis and Bonferroni correction.

## When to Use

- When the user has a labeled dataset (CSV, DataFrame, or database) and wants to discover what features predict an outcome variable (e.g., "what makes a Reddit post persuasive?", "what predicts headline engagement?")
- When the user asks to generate and test research hypotheses from data rather than manually specifying features
- When analyzing text, image, or relational datasets for latent predictive patterns (e.g., deception detection, AI-generated content detection, memorability prediction)
- When the user wants to go beyond simple feature importance and needs hypotheses that are interpretable, novel, and statistically grounded
- When the user needs to control for confounders — not just find correlations but validate that effects survive covariate adjustment
- When building a feature bank from discovered hypotheses to train a downstream classifier on held-out data

## Key Technique

EXPERIGEN uses a **Bayesian optimization analogy** to structure hypothesis search. The Generator acts as the acquisition function: it balances **exploitation** (refining hypotheses that already show statistical significance) with **exploration** (proposing semantically distant hypotheses to avoid local optima). Formally, each hypothesis H is scored by an acquisition objective A(H) = s(H) + N(H, H_prev), where s(H) measures plausibility based on prior validated hypotheses, and N(H, H_prev) is an exploration bonus proportional to the semantic distance (via text embeddings) from the existing hypothesis bank. This prevents the system from converging on a narrow cluster of redundant findings.

The **Experimenter** is a ReAct agent with access to a code interpreter and an LLM-based feature extractor. Given a natural-language hypothesis (e.g., "posts that expand the reader's perceived decision space are more persuasive"), it: (1) operationalizes the construct into a computable feature, (2) identifies potential confounders, (3) runs a statistical test with Bonferroni correction (acceptance threshold p < alpha/T across T refinement steps), and (4) returns structured feedback including p-value, effect size, and refinement guidance. This feedback loops back to the Generator, which either refines the hypothesis to address identified confounds or proposes a new seed hypothesis in the next outer iteration.

A critical innovation is the **hypothesis bank** — a curated set of up to K validated hypotheses maintained via semantic diversity. Embeddings (e.g., from text-embedding-3-large) are computed for each hypothesis, and greedy selection maximizes minimum pairwise distance. This bank serves dual purposes: it conditions the Generator to avoid redundancy, and its features are combined into a multivariate logistic regression for downstream prediction on held-out data.

## Step-by-Step Workflow

1. **Construct the dataset description**: Parse the input dataset and build a structured summary containing (a) schema with column names and types, (b) statistical summaries per feature (mean, variance, distribution shape), (c) 5-10 representative observations sampled to illustrate diversity. This description is the Generator's primary input.

2. **Define the prediction target and evaluation split**: Identify the binary or categorical outcome variable. Split data into train/validation/test sets. If the dataset has group structure (e.g., threads, sessions), ensure splits respect group boundaries to prevent leakage.

3. **Initialize the hypothesis bank**: Start with an empty bank H_0. Set hyperparameters: number of outer iterations N (typically 5-10), refinement steps per iteration T (typically 3-5), and bank capacity K (typically 10-20).

4. **Run the Generator (outer loop)**: For each outer iteration i, prompt the Generator with: (a) the dataset description, (b) the current hypothesis bank H_{i-1}, and (c) an instruction to propose a hypothesis that is semantically distant from existing entries while remaining plausible given observed data patterns. The Generator outputs a natural-language hypothesis with a stated expected direction of effect.

5. **Run the Experimenter (inner loop)**: For each proposed hypothesis, execute the ReAct evaluation cycle:
   - **Operationalize**: Convert the hypothesis into a computable feature (regex pattern, text classifier call, image feature extraction, or aggregation over relational structure).
   - **Identify confounders**: Determine covariates that could explain away the effect (e.g., text length, session identity, group membership).
   - **Test statistically**: Run logistic or linear regression with the feature and covariates. Apply Bonferroni correction: accept only if p < alpha/T.
   - **Report**: Return p-value, effect size (odds ratio or Cohen's d), and specific refinement guidance if the hypothesis fails.

6. **Refine or reject**: If the Experimenter returns a non-significant result with actionable feedback (e.g., "effect disappears when controlling for post length"), feed the feedback back to the Generator for up to T-1 additional refinement attempts. The Generator adds qualifiers, changes the operationalization, or narrows the claim scope.

7. **Update the hypothesis bank**: If a hypothesis passes significance testing, compute its text embedding and add it to the bank. If the bank exceeds capacity K, run greedy diversity selection: keep the subset of K hypotheses that maximizes minimum pairwise cosine distance.

8. **Build the combined predictor**: After all outer iterations complete, extract the feature corresponding to each validated hypothesis in the bank. Train a multivariate logistic regression on the training set using all hypothesis-derived features. Evaluate on the held-out test set and report accuracy, AUC, and per-feature coefficients.

9. **Rank and present findings**: Sort validated hypotheses by effect size and statistical significance. For each, present: the natural-language claim, the operationalization method, the p-value, the effect size, and any confounders controlled for. Flag hypotheses where the effect was initially confounded but survived after covariate adjustment.

10. **Suggest real-world validation**: For the top hypotheses, propose concrete A/B test designs or field experiments that could validate the finding outside the observed dataset. Specify the treatment, control, randomization unit, and minimum detectable effect size.

## Concrete Examples

**Example 1: Discovering predictors of persuasive arguments**

```
User: I have a CSV of Reddit ChangeMyView posts with columns [post_text,
is_persuasive, author_karma, thread_id, reply_position]. Find what makes
arguments persuasive.

Approach:
1. Parse the CSV, note the binary target `is_persuasive`, and that
   `thread_id` creates group structure (split by thread, not by row).
2. Build dataset description: 4,200 rows, text lengths range 20-800 words,
   ~30% persuasive. Sample 8 representative examples.
3. Initialize empty hypothesis bank, set N=6 outer iterations, T=4
   refinement steps, K=15 bank capacity.
4. Generator proposes: "Arguments that explicitly acknowledge the
   original poster's position before presenting a counter-argument
   are more persuasive."
5. Experimenter operationalizes: use LLM to classify whether post
   contains explicit acknowledgment (binary feature). Controls for
   reply_position and post length. Result: p=0.003, OR=1.8. Passes
   threshold (0.05/4 = 0.0125). Added to bank.
6. Next iteration, Generator proposes: "Posts that expand the reader's
   perceived decision space (offering alternatives rather than binary
   refutation) are more persuasive." Distant from bank entry #1.
7. Experimenter operationalizes via LLM feature extractor. Controls
   for length, position, acknowledgment. Result: p=0.0008, OR=2.3.
   Added to bank.
8. After 6 iterations, bank contains 9 validated hypotheses. Combined
   logistic regression achieves 71% accuracy (vs. 62% baseline).

Output (per hypothesis):
| # | Hypothesis | Feature Type | p-value | Effect Size (OR) | Confounders Controlled |
|---|-----------|-------------|---------|-------------------|----------------------|
| 1 | Acknowledging OP's position | LLM classifier | 0.003 | 1.8 | length, position |
| 2 | Expanding decision space | LLM classifier | 0.0008 | 2.3 | length, position, H1 |
| ... | ... | ... | ... | ... | ... |
Combined model accuracy: 71.2% (test set)
```

**Example 2: Detecting AI-generated text**

```
User: I have 5,000 labeled human vs. AI-generated essays. What
linguistic features distinguish them?

Approach:
1. Parse dataset: columns [text, label(human/AI), topic, word_count].
   Target: label. Standard random split (no group structure).
2. Generator proposes: "AI-generated text uses fewer discourse markers
   (however, moreover, nevertheless) per sentence than human text."
3. Experimenter: regex count of discourse markers / sentence count.
   Controls for topic and word_count. Result: p=0.02, OR=0.7.
   Passes (0.05/3=0.017 — does NOT pass). Refine.
4. Generator refines: "AI-generated text uses discourse markers more
   uniformly across paragraphs (lower variance in marker density)."
5. Experimenter: compute per-paragraph marker density, take std dev.
   Controls for topic, length. Result: p=0.001, OR=0.5. Passes.
6. Bank accumulates features like: marker variance, hedging frequency,
   sentence-initial pronoun patterns, paragraph length entropy.

Output:
Discovered 12 significant features across 8 iterations.
Top 3 by effect size:
  1. Paragraph length entropy (AI text more uniform): OR=0.4, p<0.001
  2. Hedging language density: OR=0.6, p=0.002
  3. Discourse marker variance: OR=0.5, p=0.001
Combined classifier: 83% accuracy (vs. 74% with standard features)
```

**Example 3: Multimodal — image memorability prediction**

```
User: I have 8,000 image pairs labeled by which is more memorable,
plus image files. What visual properties predict memorability?

Approach:
1. Dataset has [image_path_a, image_path_b, more_memorable_label].
   Pairwise structure requires careful operationalization.
2. Generator proposes: "Images containing human faces in unexpected
   contexts (e.g., non-portrait settings) are more memorable."
3. Experimenter: use vision model to detect faces and classify
   context as portrait/non-portrait. Compute binary feature per
   image. Run Bradley-Terry model on pairs. p=0.004, effect=1.6x.
4. Generator proposes: "Images with higher color contrast between
   foreground subject and background are more memorable."
5. Experimenter: segment foreground/background, compute mean LAB
   color distance. p=0.01, effect=1.3x. Passes after controlling
   for image brightness and complexity.

Output:
5 validated visual memorability hypotheses discovered.
Only method to surface significant memorability predictors in
this dataset (baselines found 0-1 significant features).
```

## Best Practices

- **Do:** Always apply Bonferroni correction within each refinement chain. If you allow T=4 refinement attempts, your per-test threshold is alpha/4, not alpha. This prevents inflated false discovery rates from multiple testing.
- **Do:** Explicitly identify and control for confounders before declaring significance. EXPERIGEN's key advantage over naive feature mining is confounder awareness — a correlation that vanishes after controlling for text length or group membership is not a discovery.
- **Do:** Maintain semantic diversity in the hypothesis bank using embedding-based distance. Greedy max-min selection prevents the bank from filling with minor variations of the same insight.
- **Do:** Operationalize hypotheses reproducibly — use deterministic regex/computation where possible, and set fixed random seeds for LLM-based feature extraction to ensure reproducibility.
- **Avoid:** Treating the Generator's output as ground truth. Every hypothesis must pass through the Experimenter's statistical validation. The Generator's role is to propose, not to conclude.
- **Avoid:** Running the framework on tiny datasets (< 200 observations). Statistical tests lose power and Bonferroni correction becomes prohibitively conservative with small samples. Recommend at least 500 observations for reliable discovery.

## Error Handling

- **Hypothesis is too vague to operationalize**: If the Experimenter cannot convert a hypothesis into a computable feature (e.g., "posts with good vibes are more persuasive"), return feedback requesting a more specific, measurable construct. The Generator should reformulate with concrete indicators.
- **All refinement steps fail**: After T unsuccessful refinements, abandon the hypothesis and move to the next outer iteration. Do not force significance — this is the system working correctly by filtering out weak hypotheses.
- **Confounder eliminates effect**: This is a success, not a failure. Report the confounding relationship (e.g., "the effect of X is fully explained by Y") as a finding in itself. Update the hypothesis bank context so the Generator avoids similar confounded proposals.
- **Feature extraction is computationally expensive**: For LLM-based features on large datasets, subsample for initial significance testing (e.g., 1,000 rows), then validate on the full dataset only for hypotheses that pass the subsample test.
- **Hypothesis bank reaches capacity**: Apply greedy diversity selection — embed all candidates, iteratively select the one maximizing minimum distance to the already-selected set, until K hypotheses remain.

## Limitations

- Requires a labeled dataset with a clear prediction target. Unsupervised discovery (finding interesting patterns without a target variable) is not supported by this framework.
- LLM-based feature extraction introduces non-determinism. Even with fixed seeds, different model versions may operationalize the same hypothesis differently, affecting reproducibility across time.
- The framework assumes hypotheses can be tested via observational data with covariate adjustment. It cannot establish causation — only controlled experiments (like the A/B test described in the paper) can do that. Always frame findings as predictive associations, not causal claims.
- Computationally intensive: each outer iteration involves multiple LLM calls (Generator + Experimenter + potential feature extraction). Budget approximately 10-20 LLM calls per outer iteration.
- Expert validation showed 88% of hypotheses were rated novel and 70% impactful, but this means roughly 12-30% may be obvious or not actionable. Human review of the final hypothesis bank is still recommended before acting on findings.

## Reference

**Paper:** [Accelerating Social Science Research via Agentic Hypothesization and Experimentation](https://arxiv.org/abs/2602.07983v1) — Sen Gupta et al., 2026. Look for: Section 3 (the acquisition objective formula and Generator-Experimenter loop), Section 4 (Experimenter's ReAct evaluation pipeline), Section 11.1 (full prompts), and Section 6 (the A/B test methodology showing 344% effect size on real-world conversion).