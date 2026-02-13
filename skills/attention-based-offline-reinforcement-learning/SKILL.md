---
name: "attention-based-offline-reinforcement-learning"
description: |
  Build interpretable offline reinforcement learning pipelines with attention-based encoders,
  patient/entity stratification via clustering, synthetic data augmentation (VAE + diffusion),
  ensemble conservative policies, and LLM-powered rationale generation. Designed for sequential
  decision-making on logged data where safety, interpretability, and class imbalance matter.

  Trigger phrases:
  - "Build an offline RL agent with attention"
  - "Create an interpretable treatment recommendation system"
  - "Train a conservative policy from logged data with explanations"
  - "Cluster entities into risk groups and train a safe RL policy"
  - "Augment underrepresented trajectories for offline RL"
  - "Generate natural language rationales for RL decisions"
---

# Attention-Based Offline RL with Clustering and Rationale Generation

This skill enables Claude to build end-to-end offline reinforcement learning systems that combine four synergistic components: (1) HDBSCAN clustering with UMAP for entity stratification into risk tiers, (2) VAE and diffusion model pipelines that synthesize underrepresented trajectories to fix class imbalance, (3) Advantage Weighted Regression (AWR) with a lightweight self-attention encoder and a three-model ensemble (AWR + XGBoost + TabNet) for conservative action selection, and (4) a retrieval-augmented LLM module that generates natural-language justifications grounded in domain knowledge. The technique is drawn from Kumar et al. (2026), evaluated on MIMIC-III/eICU clinical data, and generalizes to any domain where you must learn a safe, interpretable policy from fixed historical logs.

## When to Use

- When the user needs to train an RL agent on a static, historical dataset (no environment interaction) and wants interpretable outputs.
- When building a clinical decision support system for treatment recommendations (e.g., sepsis, ventilator management, dosing).
- When a dataset has severe class imbalance in action labels (e.g., rare treatments) and needs synthetic augmentation before policy learning.
- When the user wants conservative, safety-aware action selection using an ensemble of heterogeneous models rather than a single network.
- When attention-weighted feature importance must be surfaced to explain why the policy chose a particular action.
- When the user asks for natural-language rationales (via LLM + RAG) attached to each policy recommendation.
- When stratifying entities (patients, users, devices) into risk groups before applying differentiated RL policies per group.

## Key Technique

**Offline RL with AWR and Attention.** Unlike online RL, offline RL learns entirely from a fixed dataset of (state, action, reward, next_state) tuples. Advantage Weighted Regression (AWR) avoids the instability of importance sampling by weighting the behavioral policy's actions proportionally to their exponentiated advantage: `w_A = exp(A / beta)`, where `A = Q(z, a) - V(z)` and `beta` is a temperature controlling conservatism. The value function uses expectile regression (`tau`-weighted squared loss) for implicit quantile estimation, which is more stable than standard Bellman backup in offline settings. A lightweight self-attention block encodes the raw state vector `s` into a latent `z = Attn(s)`, and the resulting attention weights directly indicate which input features drove each decision—providing built-in interpretability without post-hoc explanation methods.

**Clustering + Augmentation + Ensemble.** Before training, HDBSCAN (with UMAP dimensionality reduction; `min_cluster_size=30`, `min_samples=30`) partitions entities into risk tiers validated by chi-square tests. A VAE and a diffusion model then generate synthetic trajectories specifically for underrepresented action classes, using the standard ELBO loss and denoising score matching respectively. At inference, a three-model ensemble—AWR-attention policy, XGBoost, and TabNet—votes conservatively: an action is recommended only when ensemble member probability exceeds threshold `omega` and beats competing actions. This prevents the RL agent from recommending risky out-of-distribution actions unsupported by the tree-based models.

**LLM Rationale Generation.** A locally-deployable multi-modal LLM (e.g., LLaMA 3.2-Vision) receives the patient state, chosen action, and top-k retrieved domain knowledge chunks (via ANN search with a NOMIC encoder over a vector database) and produces a natural-language justification. This closes the interpretability loop: attention weights show *which* features mattered, and the LLM explains *why* in clinical language.

## Step-by-Step Workflow

1. **Define the state, action, and reward spaces.** Specify the observation vector (e.g., 30 features: vitals, labs, treatment history), discrete action set (e.g., 4 actions: no treatment, fluids, vasopressors, combined), and a composite reward function (e.g., `r_t = -I{mortality_48h} + 0.3*I{MAP>65} + 0.3*I{SpO2>94} + 0.2*I{lactate<2}`). Normalize features with z-score; handle missing values via forward-fill then median imputation.

2. **Build the clustering-based stratification module.** Apply UMAP to reduce the feature space, then run HDBSCAN (`min_cluster_size=30, min_samples=30, epsilon=0.01`). Validate clusters with chi-square tests (p < 0.001) on outcome rates. Aggregate clusters into risk tiers by outcome thresholds (e.g., low: <=40% mortality, intermediate: 40-75%, high: >75%). Tag each trajectory with its tier.

3. **Identify class imbalance in the action distribution.** Compute action frequencies per risk tier. Flag any action class that constitutes less than ~15% of samples as underrepresented.

4. **Train generative models for synthetic augmentation.** For each underrepresented action class: (a) Train a VAE with encoder `f_phi: (s, a) -> N(mu, sigma^2)` and decoder `g_psi: z, a, r, d -> s'`, using ELBO loss with KL weight `beta`. (b) Train a diffusion model with linear noise schedule to generate full transition tuples. Generate synthetic trajectories until the minority class reaches a target proportion (e.g., parity or 2x oversampling).

5. **Implement the attention encoder.** Build a single-block self-attention module: project the `d`-dimensional state into query, key, value matrices, compute scaled dot-product attention, and output latent `z`. Store attention weights for later visualization.

6. **Train the AWR offline RL agent.** Initialize value network `V_psi`, Q-network `Q_theta`, and policy network `pi_phi` (all taking `z` as input). For each batch: (a) Compute value targets `y_V = r + gamma*(1-d)*V_psi'(z')` using a soft-updated target network. (b) Update V with expectile loss: `L_V = E[w(delta)*delta^2]` where `w = tau if delta > 0 else 1-tau`. (c) Update Q with MSE: `L_Q = E[(Q(z,a) - y_V)^2]`. (d) Compute advantage `A = Q(z,a) - V(z)`. (e) Update policy: `L_pi = -E[exp(A/beta) * log pi(a|z)]`.

7. **Train XGBoost and TabNet classifiers on the same augmented dataset.** Use the same state features as input, action labels as targets. Tune hyperparameters via cross-validation.

8. **Implement the conservative ensemble voting rule.** At inference, collect predicted action probabilities from AWR policy, XGBoost, and TabNet. Apply threshold `omega` (e.g., 0.6): recommend an action only if at least one ensemble member's probability exceeds `omega` and is the argmax. If XGBoost or TabNet strongly favors an action above threshold, prefer it over the RL policy (tree models are less prone to out-of-distribution extrapolation). Otherwise, default to the AWR policy's argmax.

9. **Build the RAG-powered rationale generator.** Index domain knowledge (clinical guidelines, textbook passages) into a vector store using a NOMIC or similar text encoder. At decision time, retrieve top-k chunks relevant to the current state. Construct a prompt: `"Patient state: {features}. Recommended action: {action}. Relevant guidelines: {retrieved_text}. Explain the clinical rationale."` Call a local LLM (e.g., LLaMA 3.2) with `temperature=4.7, top_k=100, repeat_penalty=1.1` to generate the rationale.

10. **Evaluate and visualize.** Report per-action precision/recall, overall accuracy, and average reward. Visualize attention heatmaps (line thickness proportional to attention weight) per risk tier. Compare against ablations: remove attention, remove augmentation, remove ensemble to quantify each component's contribution.

## Concrete Examples

**Example 1: Clinical Sepsis Treatment Recommender**

```
User: Build an offline RL system that recommends sepsis treatments
      from MIMIC-III data with explanations for each decision.

Approach:
1. Extract 30-feature observation vectors (HR, MAP, SpO2, lactate,
   creatinine, platelets, WBC, etc.) from MIMIC-III. Define 4 actions:
   {no_treatment, fluids, vasopressors, combined}. Reward:
   r = -1*(death_48h) + 0.3*(MAP>65) + 0.3*(SpO2>94) + 0.2*(lactate<2).

2. Run UMAP + HDBSCAN to cluster ~28K ICU stays into risk tiers.
   Validate with chi-square (p < 0.001).

3. Augment fluids-only and vasopressor-only trajectories (minority
   classes) with VAE + diffusion until balanced.

4. Train AWR agent with attention encoder (d=30 -> z=64).
   Train XGBoost and TabNet on same data.

5. Ensemble vote with omega=0.6. For each recommendation, retrieve
   top-5 sepsis guideline chunks and generate LLM rationale.

Output (per patient timestep):
  State: HR=112, MAP=58, SpO2=91%, Lactate=4.1
  Risk tier: High
  Recommended action: Vasopressors
  Ensemble agreement: AWR=vasopressors(0.72), XGBoost=vasopressors(0.68),
                      TabNet=vasopressors(0.65)
  Attention top features: MAP (0.31), Lactate (0.27), SpO2 (0.19)
  Rationale: "Vasopressor therapy is indicated due to persistent
  hypotension (MAP 58 mmHg, below 65 mmHg threshold) combined with
  elevated lactate (4.1 mmol/L), suggesting tissue hypoperfusion
  consistent with septic shock. Surviving Sepsis Campaign guidelines
  recommend vasopressors as first-line when MAP target is not achieved
  with fluid resuscitation alone."
```

**Example 2: E-Commerce Inventory Restocking Agent**

```
User: I have historical warehouse data with states (stock levels,
      demand forecasts, lead times), actions (restock levels 0-3),
      and rewards (profit - waste). Build an offline RL agent that
      recommends restocking decisions with explanations.

Approach:
1. Define state space: 20 features (current stock per SKU category,
   rolling demand mean/variance, supplier lead time, seasonality
   indicator). Actions: {no_restock, small, medium, large}.
   Reward: profit_margin - spoilage_cost - stockout_penalty.

2. Cluster SKUs by demand pattern with HDBSCAN. Tiers: stable demand,
   seasonal spikes, unpredictable.

3. "Large restock" is rare in logs (8% of actions). Augment with
   VAE to reach 20% representation.

4. Train AWR + attention, XGBoost, TabNet. Ensemble threshold omega=0.55.

5. Index supply chain best-practice docs into vector store. Generate
   rationale per recommendation via LLM.

Output:
  SKU cluster: Seasonal-spike (Tier 2)
  State: Stock=120 units, Demand_forecast=340, Lead_time=5 days
  Recommended action: Large restock
  Attention: Demand_forecast (0.35), Lead_time (0.22), Stock (0.20)
  Rationale: "A large restock is recommended because forecasted demand
  (340 units) significantly exceeds current stock (120 units) with a
  5-day lead time. This SKU cluster exhibits seasonal spikes, and
  historical data shows stockout penalties dominate spoilage costs
  in this tier."
```

**Example 3: Adding Augmentation to an Existing Offline RL Pipeline**

```
User: My offline RL agent performs poorly on rare actions. How do I
      augment the dataset?

Approach:
1. Identify minority actions: compute action frequency histogram.
   Flag actions below 15% frequency.

2. For each minority action, filter trajectories containing that action.

3. Train a VAE:
   - Encoder: state + action -> mu, log_sigma (latent_dim=32)
   - Decoder: z + action + reward + done -> next_state
   - Loss: MSE_reconstruction + beta * KL_divergence (beta=0.5)

4. Train a diffusion model (T=1000 steps, linear beta schedule
   1e-4 to 0.02):
   - Forward: gradually add Gaussian noise to transition tuples
   - Reverse: neural net predicts noise at each step
   - Loss: E[||epsilon - epsilon_theta(x_t, t)||^2]

5. Sample synthetic transitions. Validate: check that synthetic
   state distributions overlap real distributions (KS test p > 0.05).
   Append to training dataset.

Result: Minority action accuracy improves from ~55% to ~75% while
majority action accuracy remains stable.
```

## Best Practices

- **Do:** Use expectile regression (not standard MSE) for the value function—it provides implicit quantile estimation that stabilizes offline learning by down-weighting overestimated values.
- **Do:** Validate clusters with statistical tests (chi-square, silhouette score) before using them as risk tiers. Arbitrary thresholds without validation lead to meaningless stratification.
- **Do:** Generate augmented data only for genuinely underrepresented classes, and validate that synthetic distributions match real data (KS test, MMD). Over-augmentation can shift the data distribution and degrade the policy.
- **Do:** Use the ensemble as a safety net: if tree-based models (XGBoost, TabNet) disagree strongly with the RL policy, trust the trees—they are less prone to extrapolation on unseen states.
- **Avoid:** Setting AWR temperature `beta` too low (aggressive weighting of high-advantage actions) or too high (uniform weighting that ignores advantages). Start with `beta=1.0` and tune on a held-out validation set.
- **Avoid:** Using attention weights as causal explanations. They indicate *correlation* between features and decisions, not causation. Pair with the LLM rationale module for causal reasoning grounded in domain knowledge.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| HDBSCAN produces too few/many clusters | `min_cluster_size` too large/small | Sweep `min_cluster_size` in [10, 50]; check silhouette scores |
| VAE generates out-of-range states | KL weight `beta` too low; decoder unconstrained | Increase `beta`; clamp decoder outputs to physiological/domain ranges |
| AWR policy collapses to single action | `beta` too small; advantage estimates noisy | Increase `beta`; use larger critic ensemble; add entropy bonus |
| Ensemble always defers to one model | Threshold `omega` too low | Increase `omega` or add minimum agreement count (e.g., 2 of 3 must agree) |
| LLM rationale hallucinates facts | Retrieved context irrelevant or insufficient | Improve retrieval (re-rank, increase k); add faithfulness check against state values |
| Diffusion model generates noisy samples | Too few denoising steps or poor schedule | Increase T; try cosine schedule instead of linear |

## Limitations

- **Offline-only.** The entire pipeline assumes a fixed historical dataset. It cannot explore or collect new data. If the logged data does not cover critical state-action regions, the policy will be unreliable there regardless of augmentation.
- **Discrete actions only (as presented).** The AWR formulation here uses categorical policy outputs. Continuous action spaces require replacing the policy head with a Gaussian or mixture density network and adjusting the advantage weighting.
- **Attention is not causation.** The self-attention encoder highlights correlated features, not causal drivers. For domains requiring causal reasoning, supplement with causal inference methods.
- **LLM rationale quality depends on the knowledge base.** If the vector store lacks relevant domain knowledge, the LLM will produce generic or hallucinated explanations. Curating the knowledge base is essential.
- **Computational cost scales with augmentation.** Training VAE + diffusion + AWR + XGBoost + TabNet is significantly more expensive than a single model. For rapid prototyping, start with AWR + attention alone and add components incrementally.

## Reference

Kumar, P., Saran, V., Patel, D., Kulkarni, N., & Vereshchaka, A. (2026). *Attention-Based Offline Reinforcement Learning and Clustering for Interpretable Sepsis Treatment.* arXiv:2601.14228v1. [https://arxiv.org/abs/2601.14228v1](https://arxiv.org/abs/2601.14228v1)

Key sections to study: Algorithm 1 (HDBSCAN clustering procedure), Algorithm 2 (AWR training loop), Section III-C (ensemble voting rule), and Section III-D (RAG-based rationale generation with LLaMA 3.2-Vision).