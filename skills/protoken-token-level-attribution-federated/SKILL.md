---
name: "protoken-token-level-attribution-federated"
description: "Implement ProToken-style token-level attribution to trace which federated learning client(s) contributed to each generated token in a federated LLM. Use when: 'trace token provenance in federated model', 'attribute LLM outputs to FL clients', 'debug federated model generations', 'identify malicious client in federated LLM', 'build reward allocation for federated training', 'add provenance tracking to federated pipeline'."
---

# ProToken: Token-Level Attribution for Federated LLMs

This skill enables Claude to implement token-level provenance attribution systems for federated Large Language Models, based on the ProToken methodology. The core technique decomposes a federated global model's output into per-client contributions by replaying each client's layer computations on shared inputs and measuring alignment with gradient-based relevance signals. This lets you answer "which client's data drove this specific token?" for every token in an autoregressive generation — enabling debugging, malicious client detection, fair reward allocation, and trust verification without violating FL privacy constraints.

## When to Use

- When building a federated learning pipeline for LLMs and needing to trace which client influenced each generated token
- When debugging unexpected or toxic outputs from a federated model and needing to identify the responsible training client
- When implementing fair compensation or reward allocation across FL participants based on actual contribution to model outputs
- When adding provenance metadata to a federated LLM inference service for auditability or compliance
- When detecting data poisoning or adversarial clients in a federated training setup by analyzing attribution patterns
- When evaluating client contribution quality across domains (medical, financial, code, math) in a multi-client federation

## Key Technique

ProToken exploits two structural properties of transformer-based LLMs. First, later transformer blocks concentrate task-specific knowledge — the final N decoder blocks contain the signal most relevant to predicting the next token, while earlier blocks encode more general representations. By restricting attribution analysis to only the self-attention output projection and final MLP layers within the last N blocks, the method reduces the parameter space from millions to a tractable subset without significant accuracy loss.

Second, not all neural activations within a layer matter equally for a given token prediction. ProToken uses gradient-based relevance weighting: it backpropagates from the predicted token's logit to compute `g^l_xj = d(logit_xj)/d(h^l_G)`, the gradient of the output logit with respect to each layer's hidden state. Dimensions with large gradient magnitudes are the ones that actually drive the prediction; dimensions with near-zero gradients are irrelevant noise. This gradient acts as a relevance filter.

The attribution score for client `i` at layer `l` for token `xj` is then the inner product `P^l_{i,xj} = <h^l_i, g^l_xj>`, where `h^l_i` is what that layer would have produced using client i's weights (instead of the global weights) on the same input. Scores are summed across selected layers and tokens, then softmax-normalized into a probability distribution over clients. The argmax identifies the most responsible client for any given token or sequence. This achieves 98% attribution accuracy across Gemma, Llama, Qwen, and SmolLM architectures.

## Step-by-Step Workflow

1. **Set up the federated training infrastructure.** Use Flower (or equivalent FL framework) with HuggingFace Transformers. Train a global model via FedAvg across K clients, each fine-tuning with LoRA or full parameters on their domain-specific dataset. Store each client's final model checkpoint — you need per-client weights at inference time.

2. **Select target layers for attribution.** Identify the last N transformer blocks in your architecture (start with N=3-4 for models under 1B parameters). Within each block, target two specific sublayers: the self-attention output projection (`attn.o_proj`) and the final MLP layer (`mlp.down_proj` or equivalent). This drastically reduces the parameter tracking surface.

3. **Instrument the global model forward pass.** During inference, hook into the selected layers to capture: (a) the input tensor to each selected layer (`x^l_input`), and (b) the global model's hidden state output (`h^l_G`). Use PyTorch forward hooks to capture these without modifying the model architecture.

4. **Generate tokens autoregressively.** Run standard greedy or sampled decoding with the global model. For each generated token `xj`, record which token was selected and its logit value. Accumulate the captured layer inputs and hidden states per token.

5. **Compute gradient relevance vectors.** For each generated token `xj`, backpropagate from `logit_xj` through the model to each selected layer to obtain `g^l_xj = d(logit_xj)/d(h^l_G)`. Use `torch.autograd.grad` with `retain_graph=True` across layers. These gradients tell you which activation dimensions actually influenced the token choice.

6. **Replay client layer computations.** For each client `i` and each selected layer `l`, pass the captured global layer input (`x^l_input`) through client i's version of that layer to produce `h^l_i`. This simulates "what would this layer have computed using client i's weights?" without running a full forward pass per client.

7. **Compute per-client attribution scores.** For each client `i`, selected layer `l`, and token `xj`, compute the inner product `P^l_{i,xj} = dot(h^l_i, g^l_xj)`. Sum across all selected layers: `P_{i,xj} = sum_l P^l_{i,xj}`. Optionally aggregate across all generated tokens: `P_{i,y} = sum_j P_{i,xj}`.

8. **Normalize into probabilities.** Apply softmax over clients: `P_i = exp(P_{i,y}) / sum_k exp(P_{k,y})`. The result is a probability distribution over K clients for each token (or the full sequence). The argmax identifies the dominant contributor.

9. **Build the provenance report.** Emit a per-token attribution map: for each generated token, output the token text, the client probability distribution, and the top contributing client. Aggregate into sequence-level summaries for dashboards or alerting.

10. **Integrate into your serving pipeline.** Wrap the attribution logic as a post-generation middleware. Store provenance metadata alongside generated responses in your database. Set up alerting when a single client dominates unexpectedly or when known-untrusted clients show high attribution on sensitive outputs.

## Concrete Examples

**Example 1: Debugging a federated medical LLM generating incorrect drug interactions**

```
User: "Our federated medical LLM gave a dangerous drug interaction answer.
We have 6 hospital clients. Which client's data caused this?"

Approach:
1. Load the global model and all 6 client checkpoints (fine-tuned LoRA adapters)
2. Reproduce the problematic prompt and generation
3. Hook the last 3 transformer blocks (attn.o_proj + mlp.down_proj = 6 layers)
4. For the generated sequence "...combining warfarin with aspirin is safe...",
   compute attribution per token
5. Run gradient backprop from each token logit to hooked layers
6. Replay each client's layer weights on the global inputs
7. Compute inner products and softmax-normalize

Output (per-token attribution):
  Token        | Client 1 | Client 2 | Client 3 | Client 4 | Client 5 | Client 6
  "combining"  |   0.12   |   0.08   |   0.41   |   0.15   |   0.11   |   0.13
  "warfarin"   |   0.05   |   0.03   |   0.72   |   0.08   |   0.06   |   0.06
  "with"       |   0.16   |   0.17   |   0.18   |   0.16   |   0.17   |   0.16
  "aspirin"    |   0.04   |   0.02   |   0.68   |   0.10   |   0.09   |   0.07
  "is"         |   0.14   |   0.15   |   0.22   |   0.16   |   0.17   |   0.16
  "safe"       |   0.03   |   0.04   |   0.75   |   0.07   |   0.05   |   0.06

Conclusion: Client 3 dominates attribution for the medically critical tokens
("warfarin", "aspirin", "safe"). Investigate Client 3's training data for
incorrect drug safety labels.
```

**Example 2: Fair reward allocation in a coding federation**

```
User: "We have 4 companies contributing code data to a federated code LLM.
We need to allocate API revenue based on actual contribution to outputs."

Approach:
1. Sample N representative generation requests from production logs
2. For each request, run ProToken attribution across all generated tokens
3. Aggregate client attribution probabilities across the full sample
4. Weight by request value (e.g., token count, customer tier)

Implementation (Python pseudocode):
  client_revenue = {c: 0.0 for c in clients}
  for request in sampled_requests:
      tokens, attributions = run_protoken(global_model, clients, request.prompt)
      for j, token in enumerate(tokens):
          dominant_client = argmax(attributions[j])
          client_revenue[dominant_client] += request.value_per_token

Output:
  Revenue allocation for Q4 (sampled from 10,000 requests):
    Company A (systems code):   31.2%  ($15,600)
    Company B (web frameworks): 28.7%  ($14,350)
    Company C (ML libraries):   24.1%  ($12,050)
    Company D (DevOps scripts): 16.0%  ($8,000)
```

**Example 3: Detecting a poisoning attack in federated training**

```
User: "We suspect one of our 10 FL clients injected backdoor triggers.
How do we identify which client using ProToken?"

Approach:
1. Prepare a test set containing known trigger patterns and clean inputs
2. Run generation on both sets through the global model
3. Compute ProToken attribution for each generated sequence
4. Compare attribution distributions between trigger and clean inputs
5. A poisoning client will show anomalously high attribution on trigger inputs
   but normal attribution on clean inputs

Output:
  Attribution entropy analysis (lower = more concentrated on one client):
                  | Clean inputs | Trigger inputs | Delta
    Client 1      |    0.11      |     0.09       |  -0.02
    Client 2      |    0.10      |     0.08       |  -0.02
    Client 7      |    0.09      |     0.63       |  +0.54  << ANOMALY
    Client 10     |    0.10      |     0.04       |  -0.06

  Client 7 attribution spikes from 9% to 63% on trigger inputs.
  Flag Client 7 for backdoor investigation.
```

## Best Practices

- **Do:** Store client checkpoints after each FL round, not just the final round. This enables temporal attribution analysis to trace when a client's influence changed.
- **Do:** Start layer selection with the last 3-4 blocks and benchmark attribution accuracy before expanding. For models under 1B parameters, 3 blocks typically suffice. For larger models, experiment with 4-6.
- **Do:** Use `torch.no_grad()` for client layer replay and only enable gradients for the backprop from logit to hidden states. This minimizes memory overhead.
- **Do:** Cache client layer outputs when generating long sequences — the same client weights are reused across tokens, so batch the replay computation.
- **Avoid:** Running attribution on all layers. Early transformer blocks encode general linguistic features shared across all clients, adding noise to attribution without useful signal.
- **Avoid:** Using cosine similarity instead of raw inner products for the attribution score. The gradient magnitudes carry essential relevance information that normalization destroys.
- **Avoid:** Attributing functional tokens (articles, punctuation, common stop words) in isolation. These tokens show diffuse attribution across clients because they are domain-agnostic. Focus analysis on content-bearing tokens.

## Error Handling

- **Out-of-memory on gradient computation:** Reduce the number of selected layers or process tokens in smaller batches. Use gradient checkpointing (`torch.utils.checkpoint`) for the backprop pass.
- **Client weights shape mismatch:** Ensure all clients trained with identical architecture and LoRA rank. Mismatched adapter dimensions will cause layer replay failures — validate shapes before running attribution.
- **Uniform attribution across all clients:** This indicates the selected layers are too early (generic) or the generation is not domain-specific. Move layer selection deeper (closer to the output) or test with more domain-specific prompts.
- **Negative attribution scores:** These are valid and indicate a client's activations oppose the gradient direction (the client's data pushed against this prediction). Include negative scores in softmax normalization — they will produce low probabilities naturally.
- **Numerical instability in softmax:** When attribution scores have large magnitudes, subtract the max score before exponentiating: `P_i = exp(P_{i,y} - max_k(P_{k,y})) / sum_k exp(P_{k,y} - max_k(P_{k,y}))`.

## Limitations

- **Requires access to all client model checkpoints at inference time.** This is feasible when the FL server retains client models but impractical in cross-silo settings where clients delete local models after aggregation.
- **Computational cost scales linearly with K clients and N selected layers.** For federations with hundreds of clients, attribution becomes expensive — consider subsampling suspected clients or using a coarse pre-filter.
- **Attribution accuracy degrades when clients have highly overlapping data.** If two clients train on nearly identical distributions, ProToken cannot distinguish their contributions because their layer outputs will be similar.
- **Only works for transformer-based architectures.** The layer selection and gradient-based relevance assumptions depend on the transformer block structure. Non-transformer models (RNNs, SSMs) would need a different approach.
- **Assumes FedAvg-style aggregation.** The linear decomposition `theta_global = sum(rho_i * theta_i)` may not hold for more complex aggregation strategies like FedProx or scaffold.
- **Does not provide token-level attribution for the prompt**, only for generated (autoregressive) tokens, since prompt tokens are not predictions of the model.

## Reference

**Paper:** [ProToken: Token-Level Attribution for Federated Large Language Models](https://arxiv.org/abs/2601.19672v2) — Gill, Humayun, Anwar, Gulzar (2026). Focus on Section 3 (Methodology) for the attribution formulas (Equations 3-10) and Algorithms 1-2 for the inference-time attribution pipeline.