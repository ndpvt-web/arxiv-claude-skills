---
name: "cost-aware-selection-text-classification"
description: |
  Guides cost-aware model selection for text classification pipelines, applying multi-objective
  trade-off analysis (F1 vs cost vs latency) to choose between fine-tuned encoders (BERT/RoBERTa/DistilBERT)
  and LLM prompting (GPT-4o/Claude). Uses Pareto frontier analysis and a parameterized utility function
  to recommend the right model for a given deployment regime.
  Trigger phrases:
  - "Which model should I use for text classification?"
  - "Is GPT-4o overkill for my classification task?"
  - "Help me pick a cost-effective NLP model"
  - "Compare BERT vs LLM for classification cost"
  - "Optimize my text classification pipeline for production"
  - "Build a cost-aware NLP system"
---

# Cost-Aware Model Selection for Text Classification

This skill enables Claude to act as a production NLP architect who recommends the right text classification model by jointly optimizing predictive quality (macro F1), inference cost (USD per million requests), and latency (p50/p95 milliseconds). Rather than defaulting to the largest available LLM, Claude applies the multi-objective framework from Valdes Gonzalez (2026) — using Pareto frontier projections and a parameterized utility function — to match a model to the user's actual deployment constraints: interactive latency budgets, batch throughput targets, or monthly cost ceilings.

## When to Use

- When a user asks which model to use for a text classification task (sentiment, topic, intent, spam, etc.) and hasn't considered cost or latency.
- When a user is building a production NLP pipeline and needs to justify model choice against operational budgets.
- When a user is using GPT-4o or Claude API for classification and wants to know if a fine-tuned encoder would be cheaper/faster.
- When a user needs to classify text at scale (>100K requests/month) and cost is a real constraint.
- When a user asks about zero-shot vs few-shot vs fine-tuned trade-offs for a fixed-label-space problem.
- When designing a hybrid architecture where LLMs handle ambiguous cases and encoders handle the bulk.

## Key Technique

The core insight is that model selection for text classification should be treated as a **multi-objective optimization problem** across three axes: F1 score, inference cost, and latency. The paper demonstrates that fine-tuned BERT-family encoders (BERT, RoBERTa, DistilBERT) achieve F1 scores within 0-3 points of GPT-4o and Claude Sonnet 4.5 on standard benchmarks (IMDB, SST-2, AG News, DBPedia) while costing **30-250x less** and running **2-10x faster**. For example, DistilBERT classifies SST-2 at $5.19/1M requests with 98ms p50 latency; GPT-4o zero-shot costs $192.48/1M with 377ms p50 — and Claude zero-shot costs $326.67/1M with 1394ms p50.

The decision framework uses a **parameterized utility function**: `U(F1, Cost, Latency) = F1 / Cost * exp(-Latency_p50 / τ)`, where τ is a latency tolerance parameter. Three deployment regimes are defined: **latency-sensitive** (τ=250ms, interactive UIs), **balanced** (τ=500ms, standard production APIs), and **latency-tolerant** (τ=1000ms, batch/async jobs). Under all three regimes, fine-tuned encoders dominate the Pareto frontier for classification tasks with fixed label spaces. LLMs only become competitive when the label space is evolving, training data is unavailable, or the task requires open-ended reasoning beyond pattern matching.

Few-shot prompting deserves special scrutiny: it typically improves F1 by only 0.5-3.5 points over zero-shot while doubling or quadrupling cost (token counts increase 2-4x). The paper concludes few-shot is "a budgeted choice: justified under explicit cost-latency constraints rather than adopted as default."

## Step-by-Step Workflow

1. **Characterize the classification task.** Determine: Is the label space fixed or evolving? How many classes? Is labeled training data available (or obtainable)? What is the expected request volume per month? This determines whether fine-tuning is feasible.

2. **Define deployment constraints.** Ask the user for their deployment regime:
   - **Latency-sensitive** (τ=250ms): Real-time UI, chatbots, autocomplete — requires p50 < 250ms.
   - **Balanced** (τ=500ms): Standard API endpoints, webhook processors — p50 < 500ms acceptable.
   - **Latency-tolerant** (τ=1000ms+): Batch pipelines, async queues, offline analytics — latency is secondary.

3. **Estimate cost at scale.** Calculate monthly cost using these reference points (per 1M requests):
   - DistilBERT: $5-$13 (varies by input length; SST-2 shortest, IMDB longest)
   - BERT/RoBERTa: $8-$33
   - GPT-4o zero-shot: $190-$850
   - GPT-4o few-shot: $390-$1,900
   - Claude zero-shot: $330-$1,175
   - Claude few-shot: $600-$2,700
   Multiply by expected monthly volume / 1M.

4. **Benchmark F1 on the user's domain.** If the user has a dataset, recommend fine-tuning a BERT-family encoder as the baseline. Use AdamW with weight decay, up to 4 epochs, selecting the checkpoint that maximizes `F1_val - |F1_val - F1_train|` (the paper's generalization-aware metric). Run 3 seeds. Simultaneously, test GPT-4o zero-shot with temperature=0.0 and top_p=1.0 for a prompt-based ceiling.

5. **Construct the Pareto frontier.** Plot each candidate model on three 2D projections: (F1 vs Cost), (F1 vs Latency), (Cost vs Latency). A model is Pareto-optimal if no other model simultaneously beats it on all three metrics. Discard dominated models.

6. **Compute the utility score.** For each Pareto-optimal model, compute `U = F1 / Cost * exp(-Latency_p50 / τ)` using the user's τ. Rank by U. The highest-U model is the recommended default.

7. **Design the hybrid architecture (if applicable).** If LLMs score higher F1 on a meaningful subset of ambiguous inputs, recommend a two-tier system: route the bulk through the encoder, and escalate low-confidence predictions (below a calibrated threshold) to the LLM for a second opinion. This captures LLM quality on the hard tail while keeping average cost near encoder levels.

8. **Specify the deployment stack.** For encoders, recommend containerized inference on Google Cloud Run (or equivalent) with the model in an OCI image stored in Artifact Registry. For LLMs, use the provider's API with structured output parsing. Include retry/fallback logic.

9. **Set up monitoring.** Track per-model F1 on a held-out production sample, p50/p95 latency, and cumulative cost. Set alerts if F1 drops (data drift) or cost exceeds the monthly budget.

10. **Document the decision.** Produce a model selection report showing the Pareto chart, utility scores, and the rationale for the chosen model — so the team can revisit when costs or requirements change.

## Concrete Examples

**Example 1: Sentiment analysis API for a SaaS product**

User: "I need to classify customer feedback as positive/negative/neutral. We get about 2M messages per month. Currently using GPT-4o with few-shot prompting and it's costing us a fortune."

Approach:
1. This is a fixed 3-class sentiment task with high volume — a textbook case for fine-tuned encoders.
2. Deployment regime: Balanced (τ=500ms) — the API serves a dashboard, not real-time chat.
3. Cost estimate at current setup: GPT-4o few-shot on short text ≈ $387/1M × 2 = **$775/month**.
4. Proposed alternative: Fine-tune DistilBERT on 5K labeled examples from their own feedback data.
5. Projected cost: DistilBERT ≈ $5.19/1M × 2 = **$10.38/month** — a 75x reduction.
6. Expected F1: On SST-2 (comparable task), DistilBERT achieves 93.49% vs GPT-4o few-shot at 90.45%. The encoder is actually *better* here.
7. Latency: DistilBERT p50=98ms vs GPT-4o p50=377ms — 3.8x faster.

Output:
```
RECOMMENDATION: Fine-tune DistilBERT
- F1: ~93.5% (vs 90.5% GPT-4o few-shot)
- Cost: $10/month (vs $775/month) — 75x savings
- Latency: 98ms p50 (vs 377ms) — 3.8x faster
- Utility score (τ=500): 14.8 (vs 0.19 for GPT-4o FS)
- Action: Collect 5K labeled samples, fine-tune with AdamW, deploy on Cloud Run
```

**Example 2: Topic classification at startup scale**

User: "We're building a news aggregator and need to classify articles into 4 categories (World, Sports, Business, Sci/Tech). We have no labeled data yet. Volume is low — maybe 10K articles/month to start."

Approach:
1. Fixed 4-class topic task. No labeled data, but low volume.
2. Deployment regime: Latency-tolerant (τ=1000ms) — batch processing overnight.
3. Short-term: Start with GPT-4o zero-shot. At 10K/month, cost = $276/1M × 0.01 = **$2.76/month** — negligible.
4. GPT-4o zero-shot F1 on AG News (same task): 87.93%. Acceptable for MVP.
5. Medium-term: Use GPT-4o to auto-label 5-10K articles, then fine-tune RoBERTa (F1=94.63% on AG News). Transition when volume hits 100K+/month.
6. At 100K/month: RoBERTa = $1/month vs GPT-4o = $27.60/month.

Output:
```
PHASE 1 (NOW, <50K/month):
  Model: GPT-4o zero-shot
  Cost: ~$3-14/month — acceptable at this scale
  F1: ~88% — good enough for MVP

PHASE 2 (WHEN volume >100K/month):
  Model: Fine-tuned RoBERTa
  Cost: $1/month (vs $28+ for GPT-4o) — 28x savings
  F1: ~94.6% — 7 point improvement over LLM
  Bootstrap: Use Phase 1 GPT-4o outputs as training labels
```

**Example 3: Hybrid architecture for ambiguous content moderation**

User: "We do content moderation — classify posts into 14 categories. Some categories are clear-cut, others are nuanced. We need high accuracy on the ambiguous ones."

Approach:
1. Large label space (14 classes), mix of easy and hard cases — hybrid architecture territory.
2. Fine-tune BERT on the full 14-class task (reference: BERT achieves 99.40% F1 on 14-class DBPedia).
3. Calibrate BERT's softmax confidence. Route predictions with confidence > 0.85 directly to output.
4. Escalate low-confidence predictions (<0.85) to Claude zero-shot for a second opinion.
5. Estimate: If 80% of traffic is high-confidence, the hybrid sees 80% encoder cost + 20% LLM cost.

Output:
```
HYBRID ARCHITECTURE:
  Tier 1 (80% of traffic): Fine-tuned BERT
    - Cost: $10.77/1M × 0.8 = $8.62/1M
    - Latency: 203ms p50
  Tier 2 (20% escalated): Claude zero-shot
    - Cost: $751.89/1M × 0.2 = $150.38/1M
  Blended cost: $159/1M (vs $751.89/1M pure LLM — 4.7x savings)
  Blended F1: Higher than either model alone (encoder handles easy, LLM handles hard)
  Confidence threshold: Tune on validation set to balance cost vs accuracy
```

## Best Practices

- **Do:** Always compute cost-per-classification before choosing a model. A 2% F1 gain that costs 100x more is rarely justified in production.
- **Do:** Use the generalization-aware checkpoint metric `F1_val - |F1_val - F1_train|` when fine-tuning encoders — it selects checkpoints that generalize rather than overfit.
- **Do:** Run LLM baselines at temperature=0.0 with top_p=1.0 for classification — stochastic decoding adds variance without benefit on fixed-label tasks.
- **Do:** Default to DistilBERT as the first encoder to try — it's consistently within 1% F1 of BERT at half the cost and latency.
- **Avoid:** Defaulting to few-shot prompting without checking if zero-shot is sufficient. Few-shot typically adds only 0.5-3.5 F1 points while doubling or quadrupling token cost.
- **Avoid:** Using LLMs for classification tasks with stable label spaces and available training data — this is the most expensive way to solve a well-understood problem.

## Error Handling

- **No labeled data available:** Start with GPT-4o zero-shot, use its predictions to bootstrap a training set, then fine-tune an encoder. Validate the bootstrapped labels with a small human-annotated sample.
- **Encoder F1 significantly below LLM:** The task may require reasoning beyond pattern matching (e.g., sarcasm detection, implicit sentiment). Use the hybrid escalation architecture rather than switching entirely to the LLM.
- **Latency spikes on LLM API:** LLM p95 latency can be 1.5-2x p50 (e.g., Claude SST-2: p50=1394ms, p95=2048ms). Build timeout handling and fallback to the encoder for SLA-sensitive paths.
- **Cost overrun on few-shot:** Token counts scale with the number of examples. Monitor input token counts and set per-request cost caps. If few-shot cost exceeds budget, drop to zero-shot or switch to encoder.
- **Label space changes:** When new classes are added, encoders require retraining. Keep a lightweight LLM zero-shot classifier as a fallback during retraining windows.

## Limitations

- The benchmarks (IMDB, SST-2, AG News, DBPedia) are well-studied English datasets. Results may not transfer directly to low-resource languages, domain-specific jargon, or noisy user-generated text without additional validation.
- The utility function `U = F1 / Cost * exp(-Latency/τ)` is a useful heuristic but may not capture all production concerns (e.g., cold-start latency, rate limits, model update cadence, compliance requirements).
- Fine-tuning encoders requires labeled data (typically 1-10K examples). For truly zero-data scenarios with no budget for labeling, LLM prompting remains the only viable option.
- The cost figures are based on 2026 API pricing. LLM inference costs are dropping rapidly — revisit the analysis quarterly.
- For tasks requiring open-ended reasoning, multi-hop inference, or free-text generation, encoders are not applicable — this framework is strictly for fixed-label classification.

## Reference

**Paper:** Valdes Gonzalez, A. (2026). *Cost-Aware Model Selection for Text Classification: Multi-Objective Trade-offs Between Fine-Tuned Encoders and LLM Prompting in Production.* arXiv:2602.06370v1. [https://arxiv.org/abs/2602.06370v1](https://arxiv.org/abs/2602.06370v1)

**What to look for:** Tables 2-5 for per-model F1/cost/latency numbers across all four benchmarks; Figures 4-6 for Pareto frontier visualizations; Section 4.3 for the utility function formulation and deployment regime analysis; Section 5 for the hybrid architecture recommendations.