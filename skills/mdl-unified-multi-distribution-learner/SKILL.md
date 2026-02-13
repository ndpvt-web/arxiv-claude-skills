---
name: "mdl-unified-multi-distribution-learner"
description: "Design and implement MDL (Multi-Distribution Learner) architectures for industrial recommendation systems that jointly handle multiple scenarios and tasks through tokenization and attention. Use when asked to: 'build a multi-scenario recommendation model', 'implement multi-task learning for recommendations', 'design a unified ranking model across scenarios', 'add scenario-aware attention to a recommender', 'implement tokenized feature interaction for CTR prediction', 'scale a recommendation model across multiple business domains'."
---

# MDL: Unified Multi-Distribution Learner for Recommendation Systems

This skill enables Claude to design, implement, and advise on Multi-Distribution Learning (MDL) architectures for large-scale recommendation systems. MDL treats scenario information (e.g., different product surfaces) and task information (e.g., click, like, share prediction) as learnable tokens that "prompt" a shared deep network -- analogous to how prompt tokens steer LLMs. This replaces traditional gating or tower-splitting approaches (MMoE, PLE, STAR) with a unified tokenize-and-interact framework that achieves stronger feature utilization and better joint modeling of scenarios and tasks.

## When to Use

- When the user needs to build a ranking/recommendation model that serves multiple scenarios (e.g., homepage feed, search results, detail page) from a single architecture
- When implementing multi-task learning for recommendation (predicting click, conversion, like, share, etc. simultaneously) and existing shared-bottom or mixture-of-experts approaches plateau
- When the user wants to replace scenario-specific towers or gating networks with a more parameter-efficient attention-based design
- When scaling a recommendation model to billions of parameters and needing scenario/task signals to interact deeply with features rather than only at the output layer
- When designing a feature interaction module that goes beyond DCN or DeepFM by treating grouped features as tokens with self-attention
- When migrating from separate per-scenario models to a unified model and needing a principled architecture

## Key Technique

**The core insight** is that scenario IDs and task IDs should not be shallow signals (gate inputs, tower selectors, or bias terms). Instead, MDL converts them into dense token representations that participate in the same attention-based interaction pipeline as feature tokens. This "Tokenize-and-Interact" philosophy is borrowed from how LLM prompts steer model behavior: scenario and task tokens act as persistent prompts that activate relevant subspaces of the model's parameters at every layer.

**Three stacked mechanisms** form each MDL block: (1) **Feature Token Self-Attention** -- grouped features are projected into fixed-dimension tokens and interact via self-attention (with TokenMixing + per-token FFN), capturing high-order feature crosses without explicit cross-network design. (2) **Domain-Feature Cross-Attention** -- task tokens query against feature token keys/values, allowing each task to selectively attend to the features most relevant to its prediction objective. Scenario tokens undergo an analogous process. (3) **Domain-Fused Aggregation** -- for each instance, the active scenario tokens (instance-specific + global) are mean-pooled and added into task token representations, fusing scenario context into task predictions without separate scenario-specific heads.

**Stacking L layers** of these blocks means scenario/task information penetrates the feature interaction pipeline bottom-up, rather than being injected only at the final prediction layer. This is what enables MDL to outperform shallow multi-scenario/multi-task methods, especially as model size grows (demonstrated at ~0.5B parameters on Douyin Search).

## Step-by-Step Workflow

1. **Group raw features into semantic clusters.** Organize the feature set (user profile, item attributes, context, cross features) into Nf groups based on domain knowledge. Each group will become one feature token. Example: group `[user_age, user_gender, user_city]` into a "user_demographics" token.

2. **Define scenario and task vocabularies.** Enumerate all scenarios (e.g., `single_column_feed`, `double_column_feed`, `in_search`) and all tasks (e.g., `click`, `like`, `share`, `complete_view`). Each gets a learnable embedding plus a transformation network.

3. **Implement the tokenization module.** For each feature group j, concatenate its embeddings and project to a fixed dimension: `t_j = Linear(concat(e_j))`, producing `T_f in R^{Nf x d}`. For scenario tokens, feed scenario-specific embeddings through per-scenario FFNs: `t_s = ReLU(FFN(e_implicit || e_specific))`. Add one global scenario token. Task tokens follow the same pattern.

4. **Build the Feature Token Self-Interaction layer.** Implement a transformer-style block: `T_f = PerTokenFFN(LayerNorm(TokenMixing(T_f) + T_f))`. TokenMixing can be standard multi-head self-attention or a lightweight mixer (e.g., parameterized matrix mixing across token positions). This replaces explicit feature cross networks.

5. **Build the Domain-Feature Cross-Attention layer.** Task tokens attend to feature tokens via standard cross-attention: `T_t = softmax(Q_t @ K_f^T / sqrt(d)) @ V_f`, where Q comes from task tokens, K/V from feature tokens. Apply the same for scenario tokens attending to feature tokens.

6. **Build the Domain-Fused Aggregation module.** For each training instance, select the scenario tokens corresponding to its active scenario(s) plus the global scenario token. Mean-pool them and add to the task token: `t_task += mean(t_scenario_active, t_scenario_global)`. This fuses scenario context into task predictions without separate heads.

7. **Stack L MDL blocks.** Each block runs all three mechanisms. Feature tokens are updated by self-interaction; scenario and task tokens are updated by cross-attention with features; task tokens additionally receive scenario fusion. Typical depth: 3-6 layers depending on model budget.

8. **Add per-task prediction heads.** After L layers, each task token is passed to a simple logits layer (1-2 linear layers + sigmoid) to produce the task-specific prediction: `y_hat_n = sigmoid(Linear(t_task_n))`.

9. **Define the joint training loss.** Sum cross-entropy losses across all scenarios and tasks: `L = sum_k sum_n sum_i CE(y_i^{n,k}, y_hat_i^{n,k})`, where k indexes scenarios and n indexes tasks. Weight individual losses if task imbalance is severe.

10. **Train with mixed optimizer strategy.** Use RMSProp (or Adam) for dense parameters (attention weights, FFNs) and Adagrad for sparse embedding parameters. Batch size ~2048. Distribute across GPUs for large-scale training.

## Concrete Examples

**Example 1: Multi-scenario CTR model for an e-commerce platform**

User: "We have three product surfaces -- homepage, search results, and category pages -- each predicting click and purchase. Currently we train separate models per surface. Design a unified model."

Approach:
1. Define 3 scenario tokens (homepage, search, category) + 1 global scenario token, and 2 task tokens (click, purchase).
2. Group features: user_profile (5 features -> token), item_attrs (8 features -> token), query_match (3 features -> token, zero-padded for non-search), context (4 features -> token). Total: 4 feature tokens.
3. Project each group to d=128 via Linear layers.
4. Stack 4 MDL blocks: each block has feature self-attention (4 tokens, 4 heads), domain-feature cross-attention (task tokens query feature tokens), and domain-fused aggregation (active scenario + global -> task tokens).
5. Two sigmoid heads on final task tokens for click and purchase.

Output architecture (PyTorch pseudocode):
```python
class MDLBlock(nn.Module):
    def __init__(self, d_model=128, n_heads=4):
        super().__init__()
        self.feature_self_attn = nn.MultiheadAttention(d_model, n_heads, batch_first=True)
        self.feature_ffn = PerTokenFFN(d_model)
        self.feature_ln = nn.LayerNorm(d_model)

        self.task_cross_attn = nn.MultiheadAttention(d_model, n_heads, batch_first=True)
        self.task_ffn = nn.Linear(d_model, d_model)
        self.task_ln = nn.LayerNorm(d_model)

        self.scenario_cross_attn = nn.MultiheadAttention(d_model, n_heads, batch_first=True)
        self.scenario_ffn = nn.Linear(d_model, d_model)
        self.scenario_ln = nn.LayerNorm(d_model)

    def forward(self, feat_tokens, task_tokens, scenario_tokens, active_scenario_mask):
        # 1. Feature self-interaction
        feat_res = feat_tokens
        feat_tokens = self.feature_ln(self.feature_self_attn(
            feat_tokens, feat_tokens, feat_tokens)[0] + feat_res)
        feat_tokens = self.feature_ffn(feat_tokens)

        # 2. Task-feature cross-attention
        task_res = task_tokens
        task_tokens = self.task_ln(self.task_cross_attn(
            task_tokens, feat_tokens, feat_tokens)[0] + task_res)
        task_tokens = self.task_ffn(task_tokens)

        # 3. Scenario-feature cross-attention
        scen_res = scenario_tokens
        scenario_tokens = self.scenario_ln(self.scenario_cross_attn(
            scenario_tokens, feat_tokens, feat_tokens)[0] + scen_res)
        scenario_tokens = self.scenario_ffn(scenario_tokens)

        # 4. Domain-fused aggregation: fuse scenario into task
        # active_scenario_mask: [batch, num_scenarios+1] binary
        active = scenario_tokens * active_scenario_mask.unsqueeze(-1)
        scen_avg = active.sum(dim=1, keepdim=True) / active_scenario_mask.sum(
            dim=1, keepdim=True).unsqueeze(-1)
        task_tokens = task_tokens + scen_avg

        return feat_tokens, task_tokens, scenario_tokens
```

**Example 2: Adding MDL-style domain tokens to an existing DeepFM model**

User: "We already have a DeepFM model. How do we retrofit MDL's scenario/task tokenization without rewriting everything?"

Approach:
1. Keep the existing DeepFM feature interaction for the FM component.
2. Replace the Deep component's MLP with MDL blocks: tokenize the Deep component's intermediate representations into groups.
3. Add scenario and task token embeddings as new parameters.
4. Insert 2 MDL blocks between the feature interaction output and the final prediction layer.
5. The FM output and the MDL task-token output are summed before the sigmoid.

Key change -- the minimal retrofit:
```python
# Before: deep_out = MLP(concat_features)  # single vector
# After:
feat_tokens = tokenize_features(concat_features, group_indices)  # [B, Nf, d]
scenario_tokens = self.scenario_embed(scenario_ids)               # [B, Ns+1, d]
task_tokens = self.task_embed(task_ids)                           # [B, Nt, d]

for block in self.mdl_blocks:
    feat_tokens, task_tokens, scenario_tokens = block(
        feat_tokens, task_tokens, scenario_tokens, scenario_mask)

deep_out = task_tokens[:, task_idx, :]  # select relevant task token
logit = fm_out + self.head(deep_out)
```

**Example 3: Designing feature token groups for a video recommendation system**

User: "I have 200+ features for video recommendation. How should I group them into tokens?"

Approach:
1. Group by semantic domain, not arbitrarily. Recommended grouping:
   - **User profile token**: age, gender, location, account_age, device_type (5 features)
   - **User behavior token**: watch_history_emb, search_history_emb, like_history_emb (3 sequence embeddings)
   - **Item static token**: video_id, author_id, category, duration, upload_time (5 features)
   - **Item content token**: title_emb, thumbnail_emb, tag_embs (3 embeddings)
   - **Engagement stats token**: historical_ctr, historical_like_rate, avg_watch_pct (3 features)
   - **Context token**: time_of_day, day_of_week, request_source, network_type (4 features)
   - **Cross token**: user-item_similarity, query-item_relevance (2 features, if applicable)
2. Each group is concatenated and projected to d=128. Total: 7 feature tokens.
3. This keeps self-attention cost manageable (7x7 attention matrix) while preserving semantic coherence within each token.

## Best Practices

- **Do:** Group features by semantic meaning, not by data type. A "user demographics" token is more meaningful than a "all categorical features" token. The paper uses domain knowledge for grouping.
- **Do:** Always include a global scenario token alongside instance-specific scenario tokens. The global token captures cross-scenario shared patterns and prevents scenario tokens from over-specializing.
- **Do:** Use per-token FFNs (separate FFN parameters per token position) rather than a shared FFN across all tokens. This gives each feature group its own transformation capacity.
- **Do:** Scale the number of MDL layers with model size. For models under 100M parameters, 2-3 layers suffice. At 500M+, use 4-6 layers where the deeper scenario/task penetration shows the most gain.
- **Avoid:** Treating scenario/task tokens as one-hot inputs to the first layer only. The entire point of MDL is that these tokens interact at every layer. Injecting them only at input or output collapses the method to a standard shared-bottom approach.
- **Avoid:** Using too many feature token groups (>20). Self-attention cost is quadratic in the number of tokens. Keep Nf between 5-15 for practical training speed. Merge small related feature sets.

## Error Handling

- **Sparse scenario data:** If some scenarios have very few training instances, the corresponding scenario tokens may underfit. Mitigate by ensuring the global scenario token is always included in fusion and by upsampling rare scenarios in training batches.
- **Task conflict:** When tasks have opposing objectives (e.g., maximize clicks vs. maximize watch time), the domain-fused aggregation can create gradient conflicts. Use per-task loss weighting or gradient surgery techniques (PCGrad) alongside MDL.
- **Feature group mismatch:** If a feature group is missing for certain scenarios (e.g., query features absent in feed scenarios), zero-pad the group's token and consider adding a binary "feature-present" indicator to the token. Do not drop the token entirely, as self-attention positions are fixed.
- **Training instability at scale:** With 500M+ parameters and multi-head attention, gradients can be unstable. Use the mixed optimizer strategy (RMSProp for dense, Adagrad for sparse), apply LayerNorm before attention, and initialize attention weights with small variance (0.01).

## Limitations

- MDL is designed for large-scale industrial recommendation with hundreds of millions of training instances. For small datasets (<100K instances), the tokenization and multi-layer attention add unnecessary complexity -- a standard shared-bottom MLP will likely suffice.
- The framework assumes discrete, enumerable scenarios and tasks. It does not natively handle continuous scenario descriptors or dynamically emerging scenarios without retraining the scenario token embeddings.
- Feature grouping requires domain knowledge and manual design. There is no automated grouping method in the paper; poor grouping choices can degrade self-attention effectiveness.
- Online serving latency increases with the number of MDL layers and attention heads. The paper reports deployment at Douyin's scale but does not detail latency-accuracy tradeoffs. Budget attention compute carefully for real-time serving (<50ms).
- The architecture assumes all tasks share the same feature space. If tasks require fundamentally different feature sets, the cross-attention may waste capacity attending to irrelevant tokens.

## Reference

[MDL: A Unified Multi-Distribution Learner in Large-scale Industrial Recommendation through Tokenization](https://arxiv.org/abs/2602.07520v2) -- Focus on Section 3 (method) for the three interaction mechanisms and Section 4.3 (ablation) to understand the contribution of each component. The key figures are Figure 2 (architecture overview) and Table 2 (ablation results showing scenario tokens matter more than task tokens).