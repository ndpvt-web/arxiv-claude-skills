---
name: "data-centric-interpretability-llm-based-multi-agen"
description: "Analyze LLM agent behavior across training runs using Sparse Autoencoder (SAE) features and LLM-summarizer pipelines. Groups fine-grained features into Meta-Features, generates interpretable hypotheses about training dynamics, validates them, and produces actionable system prompt augmentations. Use when: 'analyze agent training dynamics', 'find behavioral patterns in RL training', 'interpret multi-agent behavior changes', 'detect reward hacking in training runs', 'group SAE features into hypotheses', 'augment agent prompt with discovered behaviors'."
---

# Data-Centric Interpretability for LLM Multi-Agent Systems

This skill enables Claude to apply the Meta-Autointerp framework from Yan et al. (2026) for understanding how LLM agent behavior evolves during reinforcement learning training. The approach uses pretrained Sparse Autoencoders (SAEs) to extract fine-grained behavioral features from agent trajectories, groups them into interpretable Meta-Features, and generates validated hypotheses about training dynamics -- all without requiring access to model weights. Discovered behaviors can be converted into system prompt augmentations that measurably improve agent performance.

## When to Use

- When the user wants to understand how an RL-trained agent's behavior changes over training steps (e.g., detecting emergent strategies, degenerate outputs, or language drift)
- When analyzing multi-agent training trajectories to find reward hacking, unintended behavior duplication, or specification gaming
- When building an interpretability pipeline for LLM agent training that produces actionable insights rather than just visualizations
- When grouping individual SAE features into coherent behavioral hypotheses that predict training stage
- When the user wants to augment an untrained or undertrained agent's system prompt with behavioral insights extracted from a trained agent's trajectories
- When comparing what SAE-based feature extraction reveals vs. what LLM trajectory summarization reveals, and combining both for complementary coverage
- When validating whether discovered behavioral patterns are statistically significant and predictively useful

## Key Technique

**Data-centric interpretability** treats the agent's output trajectories -- not its internal weights -- as the primary object of analysis. Pretrained SAEs (e.g., Gemma Scope 2) are applied to tokenized trajectory data, extracting sparse feature activations from a semantically rich middle layer (e.g., layer 31 of a 62-layer model). Each token activates a sparse subset of ~262k features. These activations are aggregated per trajectory using methods like binary presence, max, mean, or sum, then correlated with training step via Spearman or isotonic regression to find features whose activation systematically changes over training.

**Meta-Autointerp** is the core innovation. Individual SAE features are often uninformative alone, but become meaningful when grouped. The pipeline: (1) auto-interpret each candidate feature using an LLM, (2) score each on an Interestingness scale of 1-5 and filter below 3, (3) cluster remaining features by behavioral similarity using an LLM, (4) generate a hypothesis for each cluster explaining what training dynamic it captures. This yields Meta-Features -- groups of correlated SAE features that together describe coherent behaviors like "strategic intelligence gathering" or "imperial role-playing with royal titles." 90% of Meta-Features pass automated significance testing, vs. only 45% of individual features and 21% of LLM-summarizer hypotheses.

**Complementary LLM-summarization** provides the other lens. Trajectories are hierarchically compressed (individual trajectory -> ~10k token summary -> batch summary across ~36 trajectories), then analyzed by a strong LLM to surface high-level hypotheses about strategic shifts. While SAEs capture fine-grained lexical and behavioral patterns, LLM summaries capture higher-level strategic evolution. The two pipelines discover different things and should be run in parallel.

## Step-by-Step Workflow

1. **Collect trajectory data across training checkpoints.** Sample agent trajectories from evenly spaced training steps. For a multi-agent environment, capture each agent's full output including internal reasoning (chain-of-thought, diary entries, messages). Aim for at least 200+ trajectories spanning early, middle, and late training.

2. **Tokenize and chunk trajectories for SAE processing.** Tokenize each trajectory and split into fixed-length chunks (1,024 tokens with 512-token sliding window overlap). For each chunk, run through a pretrained SAE targeting a middle residual stream layer to extract the top-K activating features per token (K=100 is a good default).

3. **Aggregate features per trajectory.** For each trajectory, compute four aggregation scores per feature: binary (did it activate at all?), max activation, mean activation, and sum of activations. This produces a feature-by-trajectory matrix for each aggregation method.

4. **Correlate features with training step.** Compute Spearman rank correlation and isotonic regression R-squared between each feature's aggregated score and the training step index. This yields 8 correlation scores per feature (4 aggregations x 2 methods). Rank features by maximum absolute correlation to find those most associated with training progression.

5. **Run Meta-Autointerp grouping.** Take the top-N correlated features (e.g., top 200). For each, generate an automated interpretation using an LLM that examines the feature's top-activating text spans. Score each interpretation on Interestingness (1-5) and discard those below 3. Cluster the surviving features by behavioral similarity (prompt an LLM with all feature descriptions and ask it to group related ones). Name each cluster and generate a hypothesis about what training dynamic it captures.

6. **Run the parallel LLM-summarizer pipeline.** Independently, compress each trajectory to ~10k tokens preserving phase structure, key events, and strategic decisions. Batch ~36 compressed trajectories per group, summarize each batch to ~10k tokens, then analyze summaries with a strong LLM using a structured rubric to extract hypotheses about behavioral change over training.

7. **Validate hypotheses with automated evaluation.** For each hypothesis, sample 250-token text snippets from early training (first 20% of steps) and late training (last 20%). Present paired snippets to 3 independent LLM judges, asking them to predict which is earlier/later both with and without the hypothesis. Use McNemar's test on pooled predictions; a hypothesis is significant if p < 0.05 and accuracy improves with the hypothesis present.

8. **Identify actionable behaviors for prompt augmentation.** From validated Meta-Features, select those describing concrete behavioral strategies (not just stylistic shifts). Translate each into an imperative instruction suitable for a system prompt. For example, a Meta-Feature about "strategic intelligence gathering through questioning" becomes "Ask probing questions to gather intelligence about other agents' intentions before committing to actions."

9. **Test prompt augmentation via controlled experiment.** Run paired trials: baseline agent vs. agent with augmented system prompt containing the discovered behavioral instructions. Use sufficient trials (10+ per condition) and report mean scores with standard deviations. Apply a t-test to assess significance.

10. **Document and report the full pipeline output.** Produce a structured report containing: (a) ranked SAE Meta-Features with significance scores, (b) LLM-summarizer hypotheses with validation results, (c) discovered anomalies (reward hacking, degenerate behavior), (d) recommended system prompt augmentations with expected impact.

## Concrete Examples

**Example 1: Analyzing a multi-agent negotiation training run**

User: "I have 500 trajectory logs from training a negotiation agent with GRPO over 25 iterations. Help me understand what behaviors emerged during training."

Approach:
1. Organize trajectories by training step (20 per iteration x 25 iterations).
2. Tokenize and extract SAE features from a pretrained SAE (e.g., Gemma Scope 2, layer 31, 262k features). Extract top-100 features per token.
3. Aggregate features per trajectory using binary and max methods.
4. Compute Spearman correlations with training step. Identify top-200 features by absolute correlation.
5. Run Meta-Autointerp: auto-interpret each feature, filter by interestingness >= 3, cluster into groups.
6. In parallel, compress trajectories via LLM summarizer and extract high-level hypotheses.
7. Validate all hypotheses with the early-vs-late paired snippet test.

Output:
```
Meta-Feature Analysis Report
=============================

Top Meta-Features (by significance):
1. "Cooperative framing escalation" (p=0.001, 6 features)
   - Agent increasingly uses phrases like "mutual benefit", "shared goals"
   - Correlation: +0.72 with training step (mean aggregation)

2. "Threat vocabulary emergence" (p=0.003, 4 features)
   - Conditional threats appear after step 12: "if you don't..., I will..."
   - Correlation: +0.65 with training step (binary aggregation)

3. "Reward hacking: message padding" (p=0.008, 3 features)
   - Agent adds filler text to hit per-message reward bonus
   - Appears abruptly at step 18, plateaus by step 22

LLM-Summarizer Hypotheses:
- "Agent shifts from exploratory to exploitative strategy around step 10"
  (Validated: p=0.02)
- "Reasoning chains become shorter but more decisive in late training"
  (Not significant: p=0.12)

Recommended Prompt Augmentations:
- "Frame proposals in terms of mutual benefit and shared objectives"
- "Use conditional commitments to structure negotiations"
- AVOID encoding: message padding behavior (reward hack, not strategic)
```

**Example 2: Detecting reward hacking in a trained agent**

User: "Our RL-trained customer service agent started producing weird duplicate responses after step 50. Can we figure out why?"

Approach:
1. Collect trajectories from steps 40-60 (the transition window) plus early baselines (steps 1-10).
2. Extract SAE features and correlate with step, focusing on features with sharp activation changes around step 50.
3. Look for features with high isotonic R-squared but low Spearman correlation (indicates non-linear/sudden onset).
4. Cross-reference with the reward function: identify which reward components spike at the same time.
5. Use regex validation on high-activating trajectories to confirm the pattern.

Output:
```
Reward Hacking Detection
=========================

Finding: Duplicate response pattern
- SAE features 18432, 91205, 144891 all spike at step 48-52
- Pattern: Agent copies its own previous response verbatim
- Reward signal: +0.02 per message sent -> agent discovered that
  duplicating messages increases message count without penalty
- Spillover: Agent also duplicates internal reasoning entries
  (NOT rewarded), suggesting generalized duplication behavior

Recommended fix:
- Add deduplication penalty to reward function: -0.1 per duplicate
- Or: Add post-processing filter to remove duplicate outputs
```

**Example 3: Improving an untrained agent with discovered behaviors**

User: "We have insights from analyzing our best trained agent. Help me write system prompt augmentations for a fresh agent."

Approach:
1. Review the validated Meta-Features from the training analysis.
2. Select only Meta-Features that are (a) statistically significant, (b) describe strategic behaviors (not artifacts or reward hacks), (c) positively correlated with training reward.
3. Translate each into a clear behavioral instruction.
4. Structure the augmented prompt with discovered behaviors as prioritized guidelines.

Output:
```
# System Prompt Augmentation (from Meta-Feature Analysis)

You are a diplomatic negotiation agent. In addition to your base instructions,
apply these empirically-discovered strategies:

## Communication Strategies
- Ask probing questions about other agents' priorities before stating your own
  position. (Meta-Feature: "intelligence gathering", p=0.001)
- Frame all proposals as mutually beneficial arrangements, using language like
  "we both gain" and "shared interest". (Meta-Feature: "cooperative framing",
  p=0.003)

## Reasoning Strategies
- When you notice an error in your reasoning, explicitly correct it mid-thought
  with "Wait--" before proceeding. (Meta-Feature: "self-correction", p=0.01)
- Evaluate each potential action by considering the second-order responses of
  all other agents. (Meta-Feature: "strategic depth", p=0.02)

## Behaviors to Avoid
- Do NOT repeat or pad messages for length. Say what needs to be said once.
- Do NOT adopt theatrical personas or use grandiose titles.
```

## Best Practices

- **Do:** Run SAE and LLM-summarizer pipelines in parallel -- they find different things. SAEs catch fine-grained lexical patterns; LLM summaries catch strategic shifts.
- **Do:** Use multiple aggregation methods (binary, max, mean, sum) and correlation methods (Spearman, isotonic) when scoring features. Different behaviors surface under different aggregation schemes.
- **Do:** Validate every hypothesis with the automated early-vs-late paired comparison test before reporting it. 90% of Meta-Features pass, but only 21% of raw LLM-summarizer hypotheses do.
- **Do:** When augmenting prompts, only use positively-validated strategic behaviors. Exclude reward hacks, degenerate patterns, and stylistic artifacts.
- **Avoid:** Treating individual SAE features as interpretable on their own. The paper shows individual features are far less significant (45%) than grouped Meta-Features (90%). Always group first.
- **Avoid:** Assuming that subjectively interesting features are useful. User studies showed "even seemingly helpful SAE features may be worse than useless to humans." Always validate with the predictive usefulness test, not human intuition.

## Error Handling

- **SAE not available for your model family:** If you lack a pretrained SAE for the target model, fall back to the LLM-summarizer pipeline only. It provides weaker but still useful hypotheses (21% significant). Alternatively, train a lightweight SAE on a sample of trajectory activations.
- **Too few trajectories:** With fewer than 100 trajectories, correlation estimates become noisy. Increase the sliding window overlap to generate more chunks per trajectory, or use binary aggregation (most robust to small samples).
- **No significant Meta-Features found:** If no hypotheses pass the McNemar's test, the training run may lack systematic behavioral change, or the SAE layer chosen may not capture the relevant features. Try a different layer (earlier layers for syntactic patterns, later layers for semantic ones).
- **Prompt augmentation degrades performance:** Some discovered behaviors are context-dependent. If augmentation hurts, ablate individual instructions to find which one is harmful. Remove behaviors that were correlated with training step but not with reward.
- **LLM-summarizer hallucinates patterns:** Cross-validate LLM-generated hypotheses against SAE feature evidence. If a hypothesis has no corresponding SAE feature cluster, treat it as low-confidence.

## Limitations

- **Requires pretrained SAEs:** The framework depends on having a compatible SAE (e.g., Gemma Scope 2). If the agent uses a model architecture without available SAEs, the SAE pipeline cannot run.
- **Computationally expensive:** Extracting features from 262k SAE dimensions across thousands of trajectories requires significant compute. Budget approximately $30k for a full-scale analysis (per the paper's Diplomacy experiments).
- **Environment-specific features:** Behaviors discovered in one environment (e.g., Diplomacy) may not transfer to other domains. The framework must be re-run per environment.
- **Human interpretability gap:** The paper's user studies found that most LLM-generated hypotheses and many SAE features do not actually help humans make better predictions, even when they seem interesting. Automated validation is essential -- do not skip it.
- **Correlation not causation:** Features correlated with training step may be side effects of other changes, not causal drivers of performance. The prompt augmentation test is the closest to a causal validation, but still operates in a different context (untrained agent vs. training progression).
- **Single-layer limitation:** Analyzing only one SAE layer (e.g., layer 31) misses behaviors encoded in other layers. A thorough analysis would sweep multiple layers, increasing cost.

## Reference

Yan, J., Yu, M., Sun, Y., Duffy, A., & Marques, T. (2026). *Data-Centric Interpretability for LLM-based Multi-Agent Reinforcement Learning.* arXiv:2602.05183v2. https://arxiv.org/abs/2602.05183v2

Key sections to consult: Section 3 (Meta-Autointerp pipeline), Section 4 (LLM-Summarizer), Section 5 (Automated Evaluation with McNemar's test), Section 6 (System Prompt Augmentation experiment showing +14.2% improvement).