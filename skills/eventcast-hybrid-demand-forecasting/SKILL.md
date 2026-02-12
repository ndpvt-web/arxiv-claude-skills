---
name: "eventcast-hybrid-demand-forecasting"
description: "Build hybrid demand forecasting systems that fuse LLM-extracted event knowledge with time-series models using a dual-tower architecture. Use when asked to: 'forecast demand for flash sales', 'build a demand prediction pipeline with event awareness', 'integrate promotional calendars into forecasting', 'predict sales spikes from campaigns or holidays', 'combine LLM reasoning with time-series forecasting', 'create an event-driven demand model'."
---

# EventCast: Hybrid Demand Forecasting with LLM-Based Event Knowledge

This skill enables Claude to build modular demand forecasting systems that treat LLMs as event reasoning engines rather than numerical predictors. Based on the EventCast framework, the approach uses a dual-tower architecture: one tower encodes historical demand via an inverted transformer, while a second tower encodes structured event summaries produced by an LLM from unstructured business data (campaigns, holidays, seller incentives). The two towers fuse in a shared embedding space via additive alignment, producing forecasts that capture both baseline trends and event-driven demand shifts. This architecture achieved up to 57% MAE reduction over industrial baselines during promotional periods across 4 countries and 160 regions.

## When to Use

- When the user needs to forecast demand for products affected by promotions, flash sales, holidays, or policy changes
- When building a forecasting pipeline that must ingest unstructured business calendars, campaign briefs, or seller incentive descriptions
- When the user wants to augment an existing time-series model (ARIMA, Prophet, transformer-based) with contextual event features
- When asked to handle demand spikes that purely statistical models miss because the causal event is not in the historical data
- When designing an LLM-augmented forecasting system where the LLM should reason about events, not predict numbers directly
- When the user needs explainable forecasts that attribute demand changes to specific events

## Key Technique

**LLMs for reasoning, not prediction.** EventCast's core insight is that LLMs are poor at direct numerical forecasting but excellent at interpreting unstructured business context. The framework constrains the LLM to a single task: read raw campaign descriptions, holiday schedules, and incentive rules, then produce a structured textual summary describing the expected demand impact. This summary captures cultural nuances (e.g., Ramadan timing varies by country), event interactions (e.g., a flash sale overlapping a public holiday), and intensity signals (e.g., free-shipping thresholds). The LLM never sees demand numbers.

**Dual-tower fusion.** The time-series tower uses an inverted transformer that treats each feature dimension's history as a separate token (transposing the input from `[T x d]` to `[d x T]`), enabling cross-variable attention. The event tower tokenizes the LLM's structured output using learnable embeddings (not the LLM's own embeddings) with positional encoding, then sums them into a semantic vector. Both towers project into a shared 1024-dimensional space and fuse via addition: `h_aligned = h_hist + h_sem`. This additive fusion is simple but effective because both towers are trained end-to-end to align their representations.

**Dual prediction heads.** The fused representation feeds into two separate heads: a trend head capturing baseline demand and an event head capturing event-driven deviations. The final forecast is a weighted combination: `y = lambda * y_trend + (1 - lambda) * y_event`, with lambda defaulting to 0.4, biasing toward event-driven corrections when event knowledge is present.

## Step-by-Step Workflow

1. **Inventory your data sources.** Identify three categories: (a) historical demand time-series at your target granularity (daily/weekly, by SKU or region), (b) structured calendar data (holidays, campaigns with start/end dates), and (c) unstructured business text (campaign briefs, seller incentive descriptions, policy announcements). Map each source to a table or API.

2. **Build the event knowledge base.** Create a unified schema with columns: `date`, `region`, `event_type` (campaign/holiday/incentive), `event_name`, `intensity` (1-12 scale), `scope` (national/regional/category-level), `raw_description` (free text), and `duration_days`. Populate from your operational databases. Include a 13-month rolling window for training.

3. **Design the LLM event-extraction prompt.** Construct a parameterized prompt template that provides the LLM with: (a) national context and timezone, (b) temporal properties (day of week, month, quarter), (c) active events with relative positioning (e.g., "Day 3 of 7-day sale"), (d) promotional intensity levels and discount tiers, (e) logistics incentives with eligibility thresholds. Instruct the LLM to reason step-by-step and output semicolon-separated fields inside `<result>...</result>` tags for reliable parsing.

4. **Run LLM extraction in batch.** For each (date, region) pair in your forecast horizon, call the LLM with the populated prompt. Parse the `<result>` block into structured fields: `event_summary`, `expected_impact_direction` (up/down/neutral), `cultural_relevance`, `interaction_effects`. Cache results keyed by (date, region, event_hash) to avoid redundant calls. Use a frozen model (e.g., GPT-4, Claude, or ChatGLM-4) with temperature=0 for deterministic outputs.

5. **Prepare the time-series feature matrix.** For each prediction target, construct an input matrix `X` of shape `[14 x d]` (14-day lookback, d features). Features should include: raw demand, lag-1/7/14 demand, 7-day rolling mean and std, day-of-week encoding, month encoding, and any numerical event features (binary flags for active campaigns). Transpose to `[d x 14]` for the inverted transformer input.

6. **Encode event summaries.** Tokenize each LLM-generated summary field using a standard tokenizer (not the LLM's own). Map tokens through a learnable embedding layer (randomly initialized, trained end-to-end) and add positional embeddings. Aggregate per-field embeddings by summation to produce `h_sem` of dimension `d_model` (1024).

7. **Implement the dual-tower model.** Build the time-series tower as a multi-head self-attention encoder over the transposed feature matrix, projecting the output to `d_model` via a linear layer. Build the event tower as the embedding pipeline from step 6. Fuse via element-wise addition: `h_aligned = h_hist + h_sem`. Pass through a residual feed-forward block with batch normalization and LeakyReLU.

8. **Add dual prediction heads.** Create two linear heads from the fused representation: `y_trend = W_trend @ z + b_trend` and `y_event = W_event @ z + b_event`. Combine as `y_final = 0.4 * y_trend + 0.6 * y_event`. Train with L2 loss on actual demand. Use a weighted sampling strategy that oversamples event-period examples.

9. **Train and validate.** Train on a 13-month rolling window with the most recent month held out. Use NVIDIA GPU hardware; expect ~4 minutes per epoch on ~150M parameters. Evaluate with MAE and MSE, reporting separately for event periods vs. non-event periods to verify the event tower's contribution.

10. **Deploy with weekly retraining.** Set up a pipeline that (a) refreshes the event knowledge base from operational databases, (b) runs LLM extraction for the upcoming forecast horizon, (c) retrains or fine-tunes the model on the updated rolling window, and (d) serves forecasts with inference under 20 seconds. Log the LLM's event summaries alongside predictions for explainability.

## Concrete Examples

**Example 1: E-commerce flash sale forecasting**

User: "I need to forecast daily order volume for our Indonesian marketplace. We have historical orders and a campaign calendar, but our current Prophet model misses demand spikes during flash sales."

Approach:
1. Pull the campaign calendar into the event knowledge base schema: dates, campaign names, discount tiers, free-shipping thresholds
2. Design an LLM prompt for Indonesian market context:

```python
PROMPT_TEMPLATE = """
You are an e-commerce demand analyst for Indonesia.

Date: {date} ({day_of_week}, {month})
Region: {region}
Active campaigns:
{campaign_list}

Upcoming holidays within 7 days:
{holiday_list}

Active seller incentives:
{incentive_list}

Analyze the combined effect of these events on consumer demand.
Consider: Ramadan timing, payday cycles (25th-1st), cultural shopping habits.
Provide step-by-step reasoning, then output your analysis as:
<result>impact_direction;intensity_1_to_12;primary_driver;interaction_effects;cultural_notes</result>
"""
```

3. Batch-extract event summaries for each (date, region) in the 14-day forecast horizon
4. Build the dual-tower model with 14-day lookback, fusing Prophet-style trend features with LLM event embeddings
5. Train on 13 months of data, validate on the latest month's flash sale periods

Output:
```
Forecast for Jakarta, 2025-03-15 (Day 2 of Ramadan Flash Sale):
  Predicted orders: 142,300 (+67% vs. non-event baseline)
  Event attribution: Ramadan Flash Sale (intensity 9/12) + Payday cycle overlap
  Trend component: 85,200 | Event component: 183,500 | Lambda: 0.4
  Confidence: [128,000 - 156,600] (80% interval)
```

**Example 2: Adding event awareness to an existing time-series pipeline**

User: "We already have an LSTM-based demand model. How do I add event knowledge without rewriting everything?"

Approach:
1. Keep the existing LSTM as the time-series tower — extract its final hidden state as `h_hist`
2. Build only the event tower and fusion layer as a wrapper module:

```python
import torch
import torch.nn as nn

class EventTower(nn.Module):
    def __init__(self, vocab_size, d_model=1024, n_fields=5, max_tokens=64):
        super().__init__()
        self.token_embed = nn.Embedding(vocab_size, d_model)
        self.pos_embed = nn.Embedding(max_tokens, d_model)
        self.n_fields = n_fields

    def forward(self, token_ids_per_field):
        # token_ids_per_field: list of [batch, seq_len] tensors, one per field
        h_sem = torch.zeros(token_ids_per_field[0].size(0), 1024,
                           device=token_ids_per_field[0].device)
        for i, tokens in enumerate(token_ids_per_field):
            positions = torch.arange(tokens.size(1), device=tokens.device)
            emb = self.token_embed(tokens) + self.pos_embed(positions)
            h_sem += emb.sum(dim=1)  # aggregate tokens per field, sum across fields
        return h_sem

class EventCastWrapper(nn.Module):
    def __init__(self, existing_lstm, d_lstm, vocab_size, d_model=1024):
        super().__init__()
        self.ts_tower = existing_lstm
        self.ts_proj = nn.Linear(d_lstm, d_model)
        self.event_tower = EventTower(vocab_size, d_model)
        self.ffn = nn.Sequential(
            nn.Linear(d_model, d_model),
            nn.BatchNorm1d(d_model),
            nn.LeakyReLU(),
            nn.Linear(d_model, d_model),
        )
        self.head_trend = nn.Linear(d_model, 1)
        self.head_event = nn.Linear(d_model, 1)

    def forward(self, ts_input, event_token_fields, lam=0.4):
        h_hist = self.ts_proj(self.ts_tower(ts_input))
        h_sem = self.event_tower(event_token_fields)
        h_fused = h_hist + h_sem  # additive fusion
        z = self.ffn(h_fused) + h_fused  # residual connection
        y_trend = self.head_trend(z)
        y_event = self.head_event(z)
        return lam * y_trend + (1 - lam) * y_event
```

3. Freeze the LSTM weights initially, train only the event tower and fusion layers for 5 epochs, then fine-tune everything end-to-end
4. Compare MAE on event vs. non-event periods to validate the event tower adds value

**Example 3: Building the LLM event extraction pipeline**

User: "I have campaign data in a Postgres database and need to extract event features using an LLM. How do I set this up?"

Approach:
1. Query the database for active events within the forecast horizon:

```sql
SELECT date, region, event_name, event_type, description,
       start_date, end_date, discount_pct, free_shipping_threshold
FROM campaign_calendar
WHERE date BETWEEN :forecast_start AND :forecast_end
UNION ALL
SELECT date, region, holiday_name, 'holiday', description,
       date, date, NULL, NULL
FROM holiday_calendar
WHERE date BETWEEN :forecast_start AND :forecast_end
ORDER BY date, region;
```

2. Group events by (date, region) and format into the prompt template
3. Call the LLM in batches with structured output parsing:

```python
import re
from typing import NamedTuple

class EventFeatures(NamedTuple):
    impact_direction: str
    intensity: int
    primary_driver: str
    interaction_effects: str
    cultural_notes: str

def parse_llm_output(response: str) -> EventFeatures:
    match = re.search(r"<result>(.*?)</result>", response, re.DOTALL)
    if not match:
        return EventFeatures("neutral", 1, "none", "none", "none")
    fields = [f.strip() for f in match.group(1).split(";")]
    return EventFeatures(
        impact_direction=fields[0],
        intensity=int(fields[1]),
        primary_driver=fields[2],
        interaction_effects=fields[3] if len(fields) > 3 else "none",
        cultural_notes=fields[4] if len(fields) > 4 else "none",
    )
```

4. Cache parsed features in a `event_features` table keyed by `(date, region, event_hash)` to avoid reprocessing unchanged events

## Best Practices

- **Do** constrain the LLM to reasoning and summarization only — never feed it demand numbers or ask it to predict quantities. The LLM's job is to produce structured text describing *what* is happening, not *how much* demand will change.
- **Do** use deterministic LLM settings (temperature=0) and cache results aggressively. Event descriptions for a given campaign rarely change day-to-day, so cache by event content hash.
- **Do** evaluate event and non-event periods separately. If your event tower does not improve event-period MAE by at least 15-20%, the event extraction pipeline may need prompt tuning or richer input data.
- **Do** use the dual-head design (trend + event) rather than a single head. This separation lets you diagnose whether errors come from baseline trend estimation or event impact estimation.
- **Avoid** using the LLM's own token embeddings as event features. Train a separate learnable embedding layer end-to-end with the forecasting model — this produces task-specific representations that outperform frozen LLM embeddings.
- **Avoid** overcomplicating the fusion mechanism. Additive fusion (`h_hist + h_sem`) in a shared latent space works as well as or better than cross-attention for this task, and trains faster with fewer parameters.

## Error Handling

- **LLM returns unparseable output:** Fall back to a neutral event vector (zeros). The trend head still produces a reasonable baseline forecast. Log the failure for prompt debugging.
- **Missing event data for a region/date:** Use a default "no active events" prompt that still captures calendar features (day of week, month). The model degrades gracefully to time-series-only prediction.
- **Event intensity miscalibrated:** If the LLM consistently over- or under-estimates intensity, add few-shot examples of past events with known outcomes to the prompt template. Three to five calibration examples per market significantly improve alignment.
- **Stale event knowledge base:** If campaign data is not refreshed before extraction, forecasts during new campaigns will miss event signals entirely. Build a staleness check that compares the latest event KB timestamp against the current forecast date and raises an alert if the gap exceeds 24 hours.
- **Lambda imbalance:** If `lambda=0.4` overweights events for your domain (causing over-prediction during minor promotions), tune it on a validation set. Values between 0.3 and 0.6 are typical; lower values favor the event head.

## Limitations

- **Requires structured event calendars.** If your business does not maintain campaign calendars, holiday schedules, or promotion databases, the event extraction pipeline has no input to work with. The approach assumes operational data already exists in some form.
- **LLM latency and cost at scale.** Extracting event summaries for thousands of (date, region) pairs requires batched LLM calls. For very large-scale deployments (10K+ regions), the LLM extraction step may become a bottleneck. Aggressive caching and batch inference mitigate this.
- **Cold start for new markets.** The model needs at least one full seasonal cycle (12+ months) of historical demand paired with event data to learn event-demand relationships. In new regions, the event tower provides minimal lift until sufficient training data accumulates.
- **Does not handle supply-side shocks.** EventCast models demand-side events (promotions, holidays). Supply disruptions (warehouse outages, shipping delays) require separate handling since they affect fulfillment, not demand signals.
- **Single-step forecasting.** The architecture produces point forecasts per (date, region). Multi-horizon or autoregressive multi-step forecasting requires architectural modifications to the prediction heads.

## Reference

**Paper:** [EventCast: Hybrid Demand Forecasting in E-Commerce with LLM-Based Event Knowledge](https://arxiv.org/abs/2602.07695v2) (Hu et al., 2026). Focus on Section 3 for the dual-tower architecture, Section 3.2 for the LLM prompt template design, and Section 4 for ablation studies showing the contribution of each component.