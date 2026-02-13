---
name: "thinktank-me-multi-expert-framework-middle"
description: "Build multi-expert forecasting systems where specialized LLM agents collaborate through routing and aggregation to predict complex events. Use when asked to: 'build a multi-expert prediction system', 'create specialized agents that collaborate on forecasting', 'implement expert routing for predictions', 'aggregate predictions from multiple specialized models', 'design a think-tank style multi-agent architecture', 'forecast geopolitical or temporal events with multiple experts'."
---

# ThinkTank-ME: Multi-Expert Collaborative Forecasting Framework

This skill enables Claude to design and implement multi-expert forecasting systems inspired by the ThinkTank-ME architecture. Instead of relying on a single model to make complex predictions, this approach trains or prompts multiple domain-specialized experts, routes queries to the most relevant experts via a learned router, and aggregates their predictions using weighted ensemble methods. The technique is broadly applicable to any domain where predictions benefit from diverse specialized perspectives -- geopolitical forecasting, financial analysis, medical diagnosis, supply chain risk, or any temporal event prediction task.

## When to Use

- When the user needs to predict outcomes in a domain with multiple interacting sub-domains (e.g., geopolitics, markets, healthcare)
- When building a system that routes questions to specialized models or prompts based on query characteristics
- When implementing Mixture-of-Experts (MoE) style architectures at the application level using LLMs
- When the user asks to aggregate predictions from multiple models or agents with confidence weighting
- When designing a benchmark or evaluation pipeline for multi-expert forecasting systems
- When constructing temporal event sequences from raw event data for prediction tasks
- When a single-model approach underperforms because the domain requires diverse regional or topical expertise

## Key Technique

ThinkTank-ME decomposes a monolithic forecasting problem into three cooperating components: **specialized experts**, a **routing model**, and an **aggregation strategy**. Each expert is fine-tuned (or prompt-specialized) on a subset of the data corresponding to its domain of expertise -- for example, country-specific geopolitical event histories. This forces each expert to develop deep knowledge of its narrow domain rather than shallow knowledge of everything.

The routing model acts as a learned dispatcher. Given a new query with its historical context, the router scores each expert's likely relevance and selects which experts to consult. The router is trained using supervision signals derived from which experts historically produced correct predictions for similar queries. This is more principled than consulting all experts equally, since irrelevant experts introduce noise.

Three aggregation strategies are available in increasing sophistication: (1) **Expert Routing** -- select a single best expert via the router (fast, but fragile); (2) **Wisdom Aggregation** -- majority voting or confidence-weighted voting across all experts; (3) **Elite Ensemble** -- the router ranks experts, selects the top-k, and aggregates only their predictions with confidence weighting using `S_agg(o) = sum(c_j * I(o_j = o))` where `c_j` is the confidence score and `I` is the indicator function. The Elite Ensemble achieves the best accuracy by balancing specialization with robustness.

## Step-by-Step Workflow

1. **Define the domain decomposition.** Identify the natural sub-domains that partition your prediction space. For geopolitics this is countries; for finance it might be sectors; for medicine it might be organ systems. Each sub-domain becomes one expert. Aim for 10-50 experts depending on data availability.

2. **Construct temporal event sequences.** Format raw event data into structured quadruples: `(subject, relation, object, timestamp)`. Group semantically related events using clustering (BERTopic with HDBSCAN works well -- use temporal weighting by concatenating UMAP embeddings with normalized time features, min cluster size ~20). Split oversized clusters (>100 atomic events or >30-day span).

3. **Partition data by expert domain.** Split the event dataset so each expert receives only events relevant to its specialty. Apply entity validation (filter entities appearing <5 times, validate ambiguous entities with a secondary LLM pass). Deduplicate on `(subject, relation, object, date)` tuples.

4. **Create expert system prompts.** Each expert gets a prompt establishing its specialization:
   ```
   You are a domain expert specializing in [DOMAIN_NAME]. Given a sequence
   of historical events, predict the missing entity in the target event.
   Respond with only the predicted entity name, no explanation.

   [Historical Events]:
   - (Subject_1, Relation_1, Object_1, Time_1)
   - (Subject_2, Relation_2, Object_2, Time_2)
   ...

   [Target]: (Subject_t, Relation_t, ???, Time_t)
   ```

5. **Fine-tune or prompt-specialize each expert.** For fine-tuning, use parameter-efficient methods (LoRA/QLoRA) on the domain-specific data subset. For prompt-only approaches, pack relevant domain context into each expert's system prompt. Store predictions in structured format: `{expert_id, query_id, prediction, confidence}`.

6. **Collect training data for the router.** Run all experts on a held-out training sample. For each query, record which experts predicted correctly. Build router training pairs: `(query_context) -> (list of correct experts)`.

7. **Train the routing model.** Fine-tune a model (or use a prompted classifier) that takes query context and outputs expert relevance scores. The router should rank all experts, not just pick one, to support top-k selection.

8. **Implement the aggregation layer.** Build the Elite Ensemble: for each query, the router ranks experts, selects top-k (start with k=3-5), collects their predictions and confidence scores, and applies weighted voting:
   ```python
   def elite_ensemble(predictions, confidences, k=5):
       top_k_indices = sorted(range(len(confidences)),
                              key=lambda i: confidences[i], reverse=True)[:k]
       scores = {}
       for i in top_k_indices:
           pred = predictions[i]
           scores[pred] = scores.get(pred, 0.0) + confidences[i]
       return max(scores, key=scores.get)
   ```

9. **Evaluate with micro and macro accuracy.** Micro-average weights all queries equally (favors high-volume domains). Macro-average weights all domains equally (ensures rare domains are not ignored). Report both. ThinkTank-ME showed ~8-10% improvement over single-model baselines using Elite Ensemble.

10. **Partition test data temporally.** Ensure test events occur after training events chronologically. Account for the knowledge cutoff of your base model to prevent data leakage. Events before the cutoff can appear in training; events after must be test-only.

## Concrete Examples

**Example 1: Geopolitical Event Forecasting System**

User: "Build a multi-expert system that predicts which country will be the target of diplomatic actions in the Middle East based on recent event history."

Approach:
1. Parse POLECAT-format event data into `(actor, action_type, target, date)` quadruples
2. Partition events by actor country -- create 35 experts (17 ME + 18 G20 nations)
3. Fine-tune Llama-3.1-8B with LoRA for each country subset
4. Generate router training data by running all experts on validation set
5. Train router to predict which expert(s) are most accurate for a given event context
6. Deploy Elite Ensemble with top-5 expert selection and weighted voting

Output structure:
```json
{
  "query": {
    "subject": "Iran",
    "relation": "Diplomatic_Criticism",
    "object": "???",
    "timestamp": "2024-03-15",
    "context": ["(Iran, Military_Posture, Israel, 2024-03-10)", "..."]
  },
  "expert_predictions": [
    {"expert": "israel_expert", "prediction": "Israel", "confidence": 0.82},
    {"expert": "usa_expert", "prediction": "United States", "confidence": 0.65},
    {"expert": "saudi_expert", "prediction": "Saudi Arabia", "confidence": 0.41}
  ],
  "router_ranking": ["israel_expert", "usa_expert", "saudi_expert"],
  "final_prediction": "Israel",
  "aggregated_confidence": 0.82
}
```

**Example 2: Financial Sector Prediction System**

User: "I want to predict which companies will be affected by supply chain disruptions using a multi-expert approach."

Approach:
1. Define experts by industry sector (semiconductors, automotive, pharma, energy, etc.)
2. Structure supply chain events as `(company, disruption_type, affected_entity, date)`
3. Prompt-specialize each expert with sector-specific context (no fine-tuning needed for prototyping):
   ```
   You are an expert in the semiconductor supply chain. Given recent
   disruption events, predict which entity is most likely affected next.
   ```
4. Build a router that examines the disruption type and originating company to select relevant sector experts
5. Aggregate top-3 expert predictions with confidence weighting
6. Evaluate with both micro-accuracy (all events) and macro-accuracy (per-sector)

Output:
```
Router selected: [semiconductor_expert (0.91), automotive_expert (0.73), electronics_expert (0.68)]
Predictions: TSMC (0.91), Toyota (0.73), Samsung (0.68)
Elite Ensemble result: TSMC (confidence: 0.91)
```

**Example 3: Prompt-Only Multi-Expert Prototype (No Fine-Tuning)**

User: "I want to quickly prototype a multi-expert forecasting system using just prompts, no training."

Approach:
1. Define 5-10 expert personas via system prompts, each with a distinct analytical lens:
   ```python
   experts = {
       "military_analyst": "You analyze events through military and security dynamics...",
       "economic_analyst": "You analyze events through trade and economic dependencies...",
       "diplomatic_analyst": "You analyze events through diplomatic relations and treaties...",
       "cultural_analyst": "You analyze events through cultural and religious factors...",
       "historical_analyst": "You analyze events through historical precedent and patterns..."
   }
   ```
2. For each query, send the same historical context to all experts
3. Collect structured predictions from each (enforce JSON output)
4. Implement a simple router: keyword-match the query relation type to expert relevance scores
5. Apply weighted majority voting across top-3 experts

```python
import json
from collections import defaultdict

def multi_expert_predict(query, history, experts, llm_client):
    predictions = {}
    for name, system_prompt in experts.items():
        response = llm_client.chat(
            system=system_prompt,
            user=f"[History]: {history}\n[Predict]: {query}"
        )
        pred = json.loads(response)
        predictions[name] = pred  # {"entity": "...", "confidence": 0.X}

    # Simple keyword-based routing
    relation = query["relation"].lower()
    routing_scores = route_by_keywords(relation, experts.keys())

    # Elite ensemble: top-3 by routing score
    top_experts = sorted(routing_scores, key=routing_scores.get, reverse=True)[:3]
    vote_scores = defaultdict(float)
    for expert in top_experts:
        entity = predictions[expert]["entity"]
        conf = predictions[expert]["confidence"]
        vote_scores[entity] += conf * routing_scores[expert]

    return max(vote_scores, key=vote_scores.get)
```

## Best Practices

- **Do:** Decompose the domain along natural fault lines where expertise genuinely differs. Country-level, sector-level, or system-level splits work because experts develop distinct knowledge.
- **Do:** Train the router on actual expert performance data (which experts got which queries right), not on heuristic rules. Learned routing outperforms hand-crafted routing.
- **Do:** Use Elite Ensemble (top-k weighted) over simple majority voting. The router's ranking eliminates noise from irrelevant experts while keeping multiple perspectives.
- **Do:** Report both micro and macro accuracy. Micro rewards volume; macro rewards breadth. A system good at only high-frequency domains is hiding weaknesses.
- **Avoid:** Creating experts with overlapping domains. If two experts cover the same ground, their predictions correlate and aggregation gains diminish. Each expert should have a distinct data partition.
- **Avoid:** Using all experts for every query. This is computationally expensive and introduces noise. The router exists to prune irrelevant experts -- use it.

## Error Handling

- **Expert produces empty or malformed output:** Enforce structured output (JSON mode or constrained decoding with valid token filtering). Fall back to the next-ranked expert if parsing fails.
- **Router selects irrelevant experts:** Monitor router accuracy on validation data. If router quality degrades, fall back to Wisdom Aggregation (all experts, majority vote) as a safety net.
- **Insufficient training data for a domain:** Merge underrepresented domains into a neighboring expert's partition. Filter out domains with fewer than 20 training examples -- they produce unreliable experts.
- **Temporal data leakage:** Always partition train/test chronologically. Verify that your base model's knowledge cutoff predates the test set. Events the model could have seen during pretraining are not valid test data.
- **All experts disagree (no consensus):** When confidence-weighted voting produces a weak winner (e.g., top score < 0.3), flag the prediction as low-confidence and optionally surface all expert predictions to the user for manual review.

## Limitations

- Requires sufficient data per domain partition to train meaningful experts. Domains with sparse data produce weak experts that hurt ensemble quality.
- The router is only as good as its training signal. If experts perform similarly on training data, the router cannot learn meaningful differentiation.
- Prompt-only prototypes (no fine-tuning) lose the deep specialization that makes the approach powerful. They work for prototyping but plateau quickly.
- The approach assumes domain decomposition is known in advance. If the natural partitioning of the prediction space is unclear, expert boundaries become arbitrary and specialization suffers.
- Computational cost scales linearly with the number of experts consulted per query. Elite Ensemble mitigates this with top-k selection, but training all experts still requires N fine-tuning runs.
- Event quadruple format `(S, R, O, T)` does not capture event intensity, credibility, or nested causal relationships. Complex events with nuanced dynamics may be oversimplified.

## Reference

**Paper:** [ThinkTank-ME: A Multi-Expert Framework for Middle East Event Forecasting](https://arxiv.org/abs/2601.17065v1) (Li et al., 2026). Key sections: Section 3 for the three-component architecture (experts, router, aggregation), Section 4 for POLECAT-FOR-ME benchmark construction, and Section 5 for Elite Ensemble results showing 8-10% gains over single-model baselines. Code: [github.com/LuminosityX/ThinkTank-ME](https://github.com/LuminosityX/ThinkTank-ME).