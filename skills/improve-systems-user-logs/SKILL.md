---
name: "improve-systems-user-logs"
description: "Build feedback-driven LLM optimization pipelines using the UNO framework. Distill user interaction logs into structured rules and preference pairs, cluster queries by semantic similarity, quantify cognitive gaps, and construct adaptive experience modules. Use when: 'optimize LLM from user feedback', 'build a feedback loop for my AI system', 'distill user logs into training data', 'cluster user queries for personalization', 'create preference pairs from chat logs', 'improve model responses using interaction history'."
---

# Improve LLM Systems with User Logs (UNO Framework)

This skill enables Claude to implement **UNO (User log-driveN Optimization)**, a framework that transforms raw user interaction logs into structured learning signals for LLM systems. Rather than treating logs as flat retrieval corpora (like RAG) or unstructured memory stores, UNO distills logs into semi-structured rules and preference pairs, clusters them by query semantics and feedback type, measures the cognitive gap between the model's existing knowledge and what the logs reveal, and then selectively builds training modules only where the model actually needs improvement. This approach outperforms both RAG and memory-based baselines on long-horizon personalization and adaptation tasks.

## When to Use

- When building a feedback pipeline that turns user chat logs, corrections, or thumbs-up/down signals into actionable training data for an LLM
- When a user wants to cluster diverse user queries to build specialized model behaviors per query type
- When creating DPO (Direct Preference Optimization) preference pairs from real user interactions rather than synthetic data
- When implementing continuous learning for a deployed LLM system that receives ongoing user feedback
- When a user needs to filter noisy user feedback and only train on signals that represent genuine knowledge gaps
- When building an adaptive system that decides per-cluster whether lightweight rule injection suffices or full fine-tuning is needed

## Key Technique

**Log Distillation into Rules and Preference Pairs.** UNO processes each user interaction triplet — (query, model_response, user_feedback) — through two parallel extraction paths. First, a "no-feedback" path analyzes the query alone and extracts up to 5 rules about how to answer it well. Second, a "with-feedback" path analyzes the full triplet and distills the user's feedback into structured JSON rules capturing what the model should have done differently. The delta between these two rule sets reveals what the user taught the system. Revised answers are then generated using these rules (via multiple sampling seeds for diversity), creating preference pairs where revised answers are preferred over originals — graded by an LLM judge on accuracy, completeness, clarity, and relevance (1-10 scale).

**Query-and-Feedback-Driven Clustering.** Rather than treating all logs uniformly, UNO embeds queries using a lightweight model (e.g., Qwen3-Embedding-0.6B) and clusters them by semantic similarity with a configurable distance threshold. Each cluster represents a coherent category of user needs. This is critical because different query types may require different adaptation strategies — some clusters may need full DPO+SFT fine-tuning while others are already well-served by the base model.

**Cognitive Gap Assessment and Adaptive Module Construction.** For each cluster, UNO evaluates whether the model's existing knowledge (primary experience) already handles the cluster well by measuring win rates against baseline outputs and BLEU score ratios. Clusters where the model passes an epsilon threshold (e.g., 0.53 win rate) skip expensive fine-tuning entirely. Clusters that fail receive a **Reflective Experience Module** — additional DPO+SFT training on that cluster's preference pairs. This selective approach avoids overfitting on already-learned patterns and focuses compute on genuine knowledge deficits.

## Step-by-Step Workflow

1. **Structure the raw logs.** Parse user interaction logs into triplets of `(query, model_response, user_feedback)`. Feedback can be explicit (corrections, ratings, follow-up instructions) or implicit (user rephrased their question, abandoned the conversation, accepted the answer). Normalize each triplet into a consistent schema with fields: `query`, `old_answer`, `feedback`, `language`.

2. **Extract rules without feedback context.** For each query, generate a prompt asking the LLM: "Given this user query, extract up to 5 JSON-formatted rules or suggestions that would help a model answer it well." This captures the model's prior knowledge about how to handle the query type. Save as `without_feedback_rules.json`.

3. **Extract rules with feedback context.** For each full triplet (query + response + feedback), generate a prompt: "Analyze this conversation where a user provided feedback on the model's response. Distill the feedback into structured rules in JSON format." This captures what the user actually wanted. Save as `with_feedback_rules.json`.

4. **Generate revised answers to build preference pairs.** Using the extracted rules, generate multiple revised answers per query using different strategies — rules-only, rules+original-answer, rules+feedback+original-answer. Sample each strategy multiple times (varying random seeds) to create diverse candidates. Each revised answer paired with the original forms a preference pair.

5. **Score preference pairs with an LLM judge.** Evaluate each revised answer against the extracted rules using a judge prompt that scores on accuracy, completeness, clarity, and relevance (1-10 scale). Filter out pairs where the revised answer scores lower than the original — these represent noisy or unhelpful feedback.

6. **Cluster queries by semantic similarity.** Embed all queries using a sentence embedding model and cluster them with a distance-based algorithm (e.g., agglomerative clustering). Each cluster should represent a coherent query category. Store cluster assignments in `cluster_results.json` with metadata including per-item novelty scores.

7. **Assess cognitive gap per cluster.** For each cluster, run the base model on a validation split and compare outputs against the preference-pair-revised answers. Compute win rates (how often the revised answer is preferred) and BLEU ratios. Clusters where the base model already achieves win rate > epsilon (default 0.53) are marked as "primary experience sufficient."

8. **Build Primary Experience Module.** For all clusters, construct a lightweight rule-injection module — essentially a retrieval layer that, at inference time, fetches the most relevant distilled rules for the incoming query's cluster and prepends them to the prompt. This is the baseline adaptation.

9. **Build Reflective Experience Module for failing clusters.** For clusters below the epsilon threshold, run DPO+SFT training on the cluster's filtered preference pairs. Use a weighted combination (e.g., `sft_weight=0.5`, `sigmoid_weight=0.5`) to balance supervised fine-tuning loss with the DPO preference loss. Train for multiple epochs and select the best checkpoint per cluster based on validation win rate.

10. **Deploy with per-cluster routing.** At inference time, embed the incoming query, predict its cluster assignment, and route it to the appropriate module — either the primary experience (rule injection) or the reflective experience (fine-tuned LoRA adapter for that cluster). Generate the response using the selected module.

## Concrete Examples

**Example 1: Building a feedback pipeline for a customer support bot**

```
User: "Our support bot gets corrections from agents when it gives wrong answers.
I have 10K interaction logs with agent feedback. How do I use these to improve
the bot?"

Approach:
1. Parse the logs into triplets: (customer_query, bot_response, agent_correction).
   Normalize agent corrections as the "feedback" field.

2. Run rule extraction on each triplet:
   - Without feedback: "For the query 'How do I reset my password?', rules are:
     [{"rule": "Mention the Settings > Security path"}, {"rule": "Include 2FA
     recovery steps"}]"
   - With feedback: "Agent corrected: bot said 'click Profile' but correct path
     is 'Settings > Security'. Rules: [{"rule": "Password reset is under Settings,
     not Profile"}, {"rule": "Always mention backup email recovery option"}]"

3. Generate revised answers using the with-feedback rules, sample 3 variants.

4. Judge scores: Original answer scores 4/10, revised answers score 8/10, 7/10,
   9/10. Keep all three as preferred over original.

5. Cluster the 10K queries — yields clusters like "password-reset" (1.2K),
   "billing" (2.1K), "technical-troubleshooting" (3.5K), etc.

6. Cognitive gap check: "password-reset" cluster → base model win rate 0.41
   (below 0.53) → needs Reflective Module. "billing" cluster → win rate 0.67
   → Primary Experience (rule injection) suffices.

7. Fine-tune a LoRA adapter on the password-reset preference pairs.
   Deploy with cluster routing.

Output structure:
  data/
    without_feedback_rules.json   # 10K entries
    with_feedback_rules.json      # 10K entries
    preference_pairs.json         # ~25K pairs (3 revisions each, filtered)
    cluster_results.json          # cluster assignments
  models/
    cluster_0_lora/               # reflective module for password-reset
    cluster_3_lora/               # reflective module for tech-troubleshooting
  config/
    cluster_routing.json          # maps cluster_id → module type
```

**Example 2: Distilling implicit feedback from conversation abandonment**

```
User: "Users often rephrase their question when the model gives a bad answer.
I want to treat rephrased queries as implicit negative feedback. How do I
structure this?"

Approach:
1. Detect rephrase patterns: consecutive queries from the same session where
   cosine similarity > 0.7 between query embeddings. The first query + response
   is the "failed" interaction; the rephrase is implicit feedback.

2. Construct triplets:
   query: "What's the return policy for electronics?"
   old_answer: "Our return policy allows 30-day returns."  (too vague)
   feedback: "User rephrased as 'Can I return an opened laptop after 15 days?'"

3. Extract rules with feedback context:
   [{"rule": "Specify return windows per product category"},
    {"rule": "Address opened vs sealed item distinctions"},
    {"rule": "Include restocking fee information"}]

4. Generate revised answer incorporating these rules, judge it, build
   preference pair.

Output: Same pipeline as explicit feedback, but the "feedback" field is
synthesized from the rephrase pattern rather than direct user correction.
```

**Example 3: Selective fine-tuning with cognitive gap filtering**

```
User: "I ran UNO on my data and got 12 clusters. Do I really need to fine-tune
all of them?"

Approach:
1. Run primary experience evaluation on each cluster's validation set.

2. Results:
   Cluster  | Win Rate | BLEU Ratio | Action
   ---------|----------|------------|------------------
   0        | 0.72     | 1.05       | Primary only (skip fine-tuning)
   1        | 0.38     | 0.82       | Build Reflective Module
   2        | 0.55     | 1.01       | Primary only (above epsilon)
   3        | 0.41     | 0.79       | Build Reflective Module
   ...      | ...      | ...        | ...
   11       | 0.61     | 1.03       | Primary only

3. Only clusters 1, 3, and two others need LoRA fine-tuning.
   This reduces training compute by ~67% compared to fine-tuning all clusters.

4. For each fine-tuned cluster, select the best checkpoint across 8 epochs
   by validation win rate. Cluster 1 peaks at epoch 4, cluster 3 at epoch 6.

Output: 4 LoRA adapters instead of 12, with per-cluster checkpoint selection.
```

## Best Practices

**Do:**
- Always extract rules both with and without feedback context — the delta between them is the actual learning signal, not either set alone
- Use multiple revision strategies (rules-only, rules+original-answer, rules+feedback) and multiple random seeds per strategy to generate diverse preference pairs
- Set the epsilon threshold conservatively (0.50-0.55) to avoid fine-tuning on clusters where the base model is already adequate — overfitting on well-handled clusters degrades generalization
- Filter preference pairs by judge score — discard pairs where the revised answer does not clearly improve on the original (judge score delta < 2 points)

**Avoid:**
- Do not treat all user feedback equally — corrections from domain experts carry different weight than casual user complaints. Weight your judge scoring accordingly
- Do not skip the clustering step and fine-tune on all data uniformly — this creates interference between unrelated query types and reduces per-cluster accuracy
- Do not use the same data for both preference pair construction and cognitive gap evaluation — maintain a held-out validation split per cluster
- Do not set `GENERATE_NUM` (judge/revision samples) below 3 — fewer samples produce unreliable preference rankings and unstable training

## Error Handling

- **Empty feedback fields:** When user feedback is absent or trivially short (< 10 characters), fall back to the no-feedback rule extraction path only. Do not fabricate feedback.
- **Degenerate clusters:** If a cluster contains fewer than 20 samples after filtering, merge it with the nearest cluster by centroid distance rather than training a dedicated module on insufficient data.
- **Judge score collapse:** If the LLM judge assigns near-identical scores to original and revised answers across a cluster, the feedback for that cluster is likely noise — skip Reflective Module construction entirely.
- **LoRA training divergence:** If validation win rate drops below 0.3 at any checkpoint, stop training for that cluster and fall back to Primary Experience Module. The feedback signal is unreliable.
- **Cluster assignment ambiguity at inference:** When a query's embedding is equidistant from two cluster centroids, use both clusters' modules and select the higher-confidence response (by generation probability or self-consistency).

## Limitations

- Requires a minimum of ~500 interaction logs per meaningful cluster to produce reliable preference pairs. Below this threshold, the cognitive gap assessment becomes noisy and fine-tuning is unreliable.
- Implicit feedback (rephrases, abandonment) is inherently noisier than explicit corrections — expect 30-40% of implicit-feedback preference pairs to be filtered out by the judge.
- The framework assumes a single LLM system being optimized. Multi-model pipelines (e.g., separate retriever + generator) require adapting the feedback distillation to target the right component.
- Clustering quality depends heavily on the embedding model — domain-specific queries (medical, legal) may need a domain-adapted embedder rather than a general-purpose one.
- Real-time adaptation is not supported — UNO is a batch process. The minimum practical cycle is daily re-clustering and retraining on accumulated logs.

## Reference

**Paper:** [Improve Large Language Model Systems with User Logs](https://arxiv.org/abs/2602.06470v1) — Wang et al., 2026. Focus on Section 3 (UNO Framework) for the distillation-clustering-gap pipeline, Section 4 for DPO+SFT training configuration, and Table 2 for comparison against RAG/memory baselines. **Code:** [github.com/bebr2/UNO](https://github.com/bebr2/UNO) — reference `prompts.py` for rule extraction templates and `main.py` for the full pipeline orchestration.