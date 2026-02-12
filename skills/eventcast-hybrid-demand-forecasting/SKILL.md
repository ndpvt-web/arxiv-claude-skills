---
name: eventcast-hybrid-demand-forecasting
description: >
  Build hybrid demand forecasting systems that fuse LLM-generated event knowledge with time-series models
  using a dual-tower architecture. The LLM handles event reasoning (holidays, promotions, campaigns) while
  a separate tower handles historical demand patterns, combined via learned fusion for accurate forecasts
  during volatile event-driven periods.
  Trigger phrases: "demand forecasting with events", "predict sales during promotions",
  "forecast demand for flash sales", "event-aware time series", "e-commerce demand prediction",
  "LLM-augmented forecasting"
---

# EventCast: Hybrid Demand Forecasting with LLM-Based Event Knowledge

This skill enables Claude to build demand forecasting pipelines that combine LLM-driven event reasoning with classical time-series prediction. The core insight from EventCast is that LLMs should NOT be used to predict numbers directly -- instead, they excel at interpreting unstructured business context (campaigns, holidays, seller incentives) into structured event summaries that a lightweight neural forecaster can consume. This dual-tower approach keeps forecasts grounded in historical data while injecting forward-looking event semantics, achieving dramatic accuracy gains during volatile periods like flash sales and holiday campaigns.

## When to Use

- When the user needs to forecast demand, sales, or order volume for an e-commerce platform that runs promotions, flash sales, or holiday campaigns
- When building a time-series forecasting system that must account for future known events (marketing campaigns, policy changes, holiday schedules)
- When existing demand forecasts degrade during promotional or holiday periods and the user wants event-aware corrections
- When the user has unstructured business data (campaign briefs, holiday calendars, seller incentive plans) and wants to incorporate it into quantitative forecasts
- When building a forecasting pipeline that must work across multiple countries or regions with different cultural calendars (e.g., Ramadan, Diwali, Chinese New Year)
- When the user asks to combine LLM reasoning with numerical prediction without letting the LLM hallucinate numbers

## Key Technique

**Separation of concerns: LLM for reasoning, neural network for numbers.** EventCast rejects the approach of feeding time-series data into an LLM and asking it to predict future values. Instead, it uses the LLM exclusively as an event interpreter. Unstructured business data -- campaign descriptions, holiday schedules, seller incentive structures -- is fed to a frozen LLM via parameterized prompts. The LLM outputs structured textual summaries capturing event type, intensity, cultural context, and expected demand direction. These summaries are then tokenized and embedded as features for a lightweight forecasting model.

**Dual-tower fusion architecture.** The forecasting model has two towers. The Historical Tower uses an inverted transformer (iTransformer-style) where each input variable is treated as a token, and multi-head self-attention captures cross-variable interactions over a lookback window (typically 14 days). The Event Tower takes the LLM-generated summary, tokenizes it, applies learnable token + positional embeddings (randomly initialized, not pretrained), and aggregates via summation into a fixed-dimensional vector. Both towers project into a shared 1024-dimensional latent space, fuse via element-wise addition, then pass through a residual feed-forward network with dual forecasting heads -- one for trend, one for event-driven patterns -- combined as `y = lambda * y_trend + (1-lambda) * y_event` with lambda=0.4.

**Why this works:** Events like "Ramadan week 3 + free shipping + 50% seller co-fund" create demand patterns that pure time-series models cannot anticipate from historical data alone. The LLM bridges this gap by leveraging world knowledge (e.g., understanding that pre-iftar shopping spikes on Thursday-Friday, or that Eid al-Fitr suppresses demand). The structured summary format ensures this knowledge is consumed deterministically by the forecaster, avoiding LLM hallucination on actual numbers.

## Step-by-Step Workflow

1. **Inventory the event data sources.** Identify all available unstructured business data: campaign calendars, promotional rules, holiday schedules, logistics incentive tables, seller co-funding agreements. Map each source to a database table or API endpoint. Classify events into categories: promotional campaigns, religious/cultural holidays, public holidays, logistics changes, policy interventions.

2. **Design parameterized LLM prompts for event summarization.** Build a prompt template that includes: country/region context, target date, day-of-week, calendar events with relative positioning (e.g., "3rd day of Ramadan"), promotional intensity on a numeric scale (1-12), free-shipping status, minimum order thresholds, seller incentive types (rebate, subsidy, co-fund ratios). End the prompt with: "Considering the combination of all factors, what is the expected overall demand trend?" Instruct the LLM to return structured output in a consistent format with semicolon-separated fields inside `<result>` tags.

3. **Run the LLM event extraction pipeline.** For each (region, date) pair in the forecast horizon, populate the prompt template with relevant event data and call the LLM (use a frozen model -- no fine-tuning needed). Parse the structured output into fields: holiday_status, holiday_scope (state/national), campaign_level, free_shipping, logistics_threshold, seller_incentives, demand_trend. Cache results since the same event context applies to all SKUs in a region-date.

4. **Prepare historical demand features.** Extract a multivariate time-series matrix `X` of shape `(T, d)` where T is the lookback window (14 days recommended) and d is the number of demand-related variables (order volume, GMV, units sold, etc.). Transpose to `(d, T)` for the inverted transformer -- each variable becomes a token with T-length sequence.

5. **Build the Historical Tower.** Implement a multi-head self-attention encoder operating on the transposed input. Use scaled dot-product attention: `Attention(Q,K,V) = softmax(QK^T / sqrt(d_k)) V`. Project the encoder output to the shared latent dimension (1024) via a learned projection matrix.

6. **Build the Event Tower.** Tokenize the LLM-generated event summary using a standard tokenizer (e.g., ChatGLM tokenizer or any subword tokenizer). Apply randomly initialized token embeddings and positional embeddings: `E_i = TokenEmbed(s_i) + PosEmbed(i)`. Aggregate all token embeddings via summation: `h_sem = sum(E_i)`. Project to the same 1024-dimensional shared space.

7. **Implement the fusion layer and dual forecasting heads.** Add the two tower outputs element-wise: `h_aligned = h_hist + h_sem`. Pass through a residual feed-forward block with BatchNorm and LeakyReLU. Split into two forecasting heads: a trend head (captures base demand) and an event head (captures event-driven deviations). Combine outputs: `y = 0.4 * y_trend + 0.6 * y_event`.

8. **Train on historical data with known events.** Use region-date aligned pairs of (historical demand sequences, event summaries, actual demand) as training data. Train jointly with MSE loss. The event embeddings are learned end-to-end -- the model learns which tokens in the event summary matter for demand.

9. **Deploy with a two-stage inference pipeline.** Stage 1 (offline, daily): Run the LLM event summarization for all region-date pairs in the forecast horizon (T+1 to T+4). Stage 2 (online, per-request): Feed the cached event embeddings + live historical features through the dual-tower model. Target inference latency: <20 seconds for a full regional forecast.

10. **Monitor and recalibrate.** Track MAE/MSE separately for event-driven periods vs. normal periods. If event-period accuracy degrades, inspect the LLM summaries for that event type -- the prompt template may need new fields for novel event categories. Retrain the forecasting model monthly with fresh event-demand pairs.

## Concrete Examples

**Example 1: E-commerce flash sale forecasting pipeline**

User: "We run flash sales every month on our e-commerce platform. Our current ARIMA model fails badly during these periods. Build me a forecasting system that accounts for upcoming promotions."

Approach:
1. Query the promotions database for upcoming flash sale metadata: discount depth, category scope, duration, free-shipping eligibility, seller participation rate.
2. Build an LLM prompt template:
   ```
   You are analyzing demand impact for an e-commerce platform.
   Country: {country_code} | Region: {region_id}
   Date: {target_date} ({day_of_week})
   Holiday status: {holiday_name or "none"}, day {day_number} of {total_days}
   Promotion: {promo_type}, discount level {1-12}, categories: {categories}
   Free shipping: {yes/no}, minimum order: ${threshold}
   Seller incentives: {incentive_description}

   Based on these factors, summarize the expected demand conditions.
   Format your response as: <result>holiday_status;holiday_scope;campaign_level;
   free_shipping;logistics_threshold;seller_incentives;demand_trend</result>
   ```
3. For each (region, date) in the next 4 days, call the LLM with populated prompts.
4. Build a dual-tower PyTorch model:
   ```python
   class EventCastModel(nn.Module):
       def __init__(self, n_vars, lookback, d_model=1024, vocab_size=50000):
           super().__init__()
           # Historical tower: inverted transformer
           self.var_embed = nn.Linear(lookback, d_model)
           self.attn = nn.MultiheadAttention(d_model, nhead=8, batch_first=True)
           self.hist_proj = nn.Linear(d_model, d_model)

           # Event tower: learnable embeddings
           self.token_embed = nn.Embedding(vocab_size, d_model)
           self.pos_embed = nn.Embedding(512, d_model)

           # Fusion + dual heads
           self.fusion_ffn = nn.Sequential(
               nn.Linear(d_model, d_model), nn.BatchNorm1d(d_model),
               nn.LeakyReLU(), nn.Linear(d_model, d_model))
           self.trend_head = nn.Linear(d_model, 1)
           self.event_head = nn.Linear(d_model, 1)

       def forward(self, x_hist, event_tokens):
           # Historical tower: (batch, d_vars, T) -> attention over variables
           h = self.var_embed(x_hist)  # (batch, d_vars, d_model)
           h, _ = self.attn(h, h, h)
           h_hist = self.hist_proj(h.mean(dim=1))  # pool over variables

           # Event tower: token embeddings summed
           positions = torch.arange(event_tokens.size(1), device=event_tokens.device)
           e = self.token_embed(event_tokens) + self.pos_embed(positions)
           h_sem = e.sum(dim=1)  # aggregate tokens

           # Fusion
           h_fused = self.fusion_ffn(h_hist + h_sem) + (h_hist + h_sem)
           y_trend = self.trend_head(h_fused)
           y_event = self.event_head(h_fused)
           return 0.4 * y_trend + 0.6 * y_event
   ```
5. Train on 6+ months of historical (demand, event) pairs. Evaluate separately on event vs. non-event days.

Output: A forecasting service that takes historical demand + upcoming event metadata and produces daily demand predictions per region, with significant accuracy gains during promotional periods.

---

**Example 2: Multi-country holiday demand forecasting**

User: "We operate in Indonesia, Malaysia, Thailand, and the Philippines. Our forecasting breaks during Ramadan, Chinese New Year, and Songkran. How do we handle this?"

Approach:
1. Build a country-aware event database mapping holidays to dates, durations, and cultural significance per region. Include state-level vs. national-level distinction.
2. Design LLM prompts that leverage cultural world knowledge:
   ```
   Country: Indonesia | Region: Jakarta
   Date: 2026-03-15 (Sunday), Ramadan Day 18 of 30
   Context: Pre-Eid shopping period begins. Historically, F&B and
   home decor surge Thu-Fri for iftar preparation. Sahur snack demand
   peaks 3am-6am. Is this a state-level or national-level observance?
   What demand pattern do you expect for this specific day?
   ```
3. The LLM output captures nuances a rule-based system cannot: "Ramadan Day 18 falls on a Sunday; expect moderate increase as weekend iftar gatherings combine with mid-Ramadan fatigue reducing impulse purchases. National-level observance. Campaign level 8."
4. Feed these structured summaries into the EventCast dual-tower model, trained per-country with shared architecture but country-specific embeddings.
5. Key: the same architecture handles Songkran (Thailand water festival with logistics disruptions), CNY (gifting-driven surge), and Eid (biphasic: surge before, drop during) because the LLM contextualizes each event differently.

Output: Per-country, per-region daily demand forecasts that correctly capture culturally-specific demand patterns during overlapping holiday periods.

---

**Example 3: Adding event awareness to an existing forecasting system**

User: "We already have a Prophet/XGBoost demand model. How do we add event knowledge without rebuilding everything?"

Approach:
1. Keep the existing model as the trend predictor (replaces the Historical Tower).
2. Build only the Event Tower as an add-on:
   - Run LLM event summarization for each forecast date
   - Tokenize and embed the summaries using a small learned embedding layer
   - Train a lightweight MLP that maps event embeddings to a demand adjustment factor
3. Combine: `y_final = 0.4 * y_existing_model + 0.6 * y_event_adjustment`
4. Train the event adjustment MLP on residuals from the existing model during event periods.

Output: An event-aware wrapper that improves the existing forecasting model during promotional and holiday periods without replacing it.

## Best Practices

- **Do:** Use the LLM only for event interpretation, never for predicting actual demand numbers. The LLM generates structured text summaries; the neural network produces numbers.
- **Do:** Cache LLM event summaries by (region, date) since they are independent of individual SKU demand. This avoids redundant LLM calls and keeps inference fast.
- **Do:** Include relative event positioning in prompts (e.g., "day 3 of 7-day campaign") rather than just binary event flags. Demand patterns differ dramatically between the start, middle, and end of a promotion.
- **Do:** Use randomly initialized token embeddings for event summaries rather than frozen pretrained embeddings. This lets the model learn which textual features actually correlate with demand shifts.
- **Avoid:** Feeding raw promotional text (e.g., marketing copy) directly to the model. Always pass it through the LLM summarization step first to normalize format and extract demand-relevant signals.
- **Avoid:** Training a single global model without regional or country-level adaptation. Holiday impacts vary dramatically by region (a state holiday in Jakarta has different demand impact than a national holiday across Indonesia).

## Error Handling

- **LLM returns inconsistent format:** Validate the structured output against an expected schema after each call. If fields are missing or malformed, retry once with a stricter prompt. Fall back to a "no event knowledge" baseline (trend-only prediction) if the LLM consistently fails for a given input.
- **Novel event types not in training data:** When a new event category appears (e.g., a government policy change), the LLM can still produce reasonable summaries via world knowledge, but the event embeddings may not generalize. Flag predictions for novel events as low-confidence and route to human review.
- **Overlapping events cause confusion:** When multiple events overlap (38% of days in the paper's dataset), ensure the prompt explicitly lists ALL active events for that date. The LLM should reason about their combined effect rather than processing each independently.
- **Historical data is sparse for a new region:** The Historical Tower needs sufficient lookback data. For cold-start regions, rely more heavily on the Event Tower by adjusting lambda toward 1.0 (more event weight) and use transfer learning from similar regions.
- **Demand trend labels are noisy:** If the LLM's demand_trend field (surge/increase/normal/decrease/drop) doesn't map well to actual outcomes, treat it as one of several input features rather than a hard classification. The model can learn to weight it appropriately during training.

## Limitations

- Requires known future events. EventCast cannot predict demand shifts from unanticipated events (supply chain disruptions, competitor actions, viral social media). It only works when business events are planned and available in databases ahead of time.
- LLM latency makes real-time re-forecasting impractical. The event summarization step should run in batch (daily), not per-request. Intraday demand adjustments need a separate mechanism.
- The dual-tower architecture adds complexity over simpler feature-engineering approaches (e.g., one-hot event flags). For platforms with few, well-understood event types, a simpler approach may suffice. EventCast's value emerges when event combinations are novel or culturally complex.
- Performance gains are concentrated in event-driven periods. During normal (non-event) days, EventCast performs comparably to standard time-series baselines. The ROI depends on what fraction of your demand is event-driven.
- The paper's lambda=0.4 for trend weight was tuned on specific e-commerce data. Different domains (grocery, fashion, electronics) may need different fusion ratios. Treat lambda as a hyperparameter to tune.

## Reference

**Paper:** [EventCast: Hybrid Demand Forecasting in E-Commerce with LLM-Based Event Knowledge](https://arxiv.org/abs/2602.07695v2) (Hu et al., 2026). Look for: the dual-tower architecture diagram (Figure 2), the event summarization prompt template (Section 3.2), and the ablation study separating event-tower vs. historical-tower contributions (Table 3).