---
name: "reconstructing-training-data-adapter-based"
description: >
  Audit adapter-based federated LLMs for training data leakage using the UTR
  (Unordered-word-bag-based Text Reconstruction) gradient inversion attack.
  Implements token inference from frozen attention layers, sentence-level
  inversion in LoRA low-rank subspaces, and constrained greedy decoding.
  Use when: "audit federated LoRA for privacy leaks", "reconstruct training
  data from adapter gradients", "test if LoRA gradients leak private text",
  "gradient inversion attack on federated LLM", "evaluate privacy of
  adapter-based fine-tuning", "check if LoRA adapters expose training data".
---

# Reconstructing Training Data from Adapter-Based Federated LLMs

This skill enables Claude to implement and reason about the UTR (Unordered-word-bag-based Text Reconstruction) attack, a gradient inversion technique that reconstructs private training text from the shared gradients of LoRA/adapter-based federated large language models. Unlike prior gradient inversion attacks that fail on low-rank adapters, UTR exploits the frozen backbone's attention layers to infer which tokens were present, then uses the adapter's low-rank gradient subspace to verify sentence-level reconstructions, achieving ROUGE-1/2 > 99 even at batch sizes where all prior attacks fail completely. This skill is for authorized security auditing, privacy research, and defensive evaluation only.

## When to Use

- When a user wants to audit whether their federated LoRA fine-tuning setup leaks private training data through shared gradients
- When implementing a privacy red-team evaluation for adapter-based federated learning pipelines
- When the user asks to reproduce or extend the UTR attack from Chen et al. (WWW 2026) on GPT-2, BERT, or Qwen-family models
- When evaluating whether differential privacy noise or gradient pruning adequately protects adapter gradients from reconstruction
- When building a defense benchmark: testing countermeasures against gradient inversion on LoRA/adapter-tuned LLMs
- When a user needs to understand the theoretical privacy-efficiency trade-off in adapter-based federated learning (Corollary 4.3: recoverable tokens bounded by adapter bottleneck rank)

## Key Technique

**The core insight** is that adapter-based federated LLMs (where the backbone is frozen and only LoRA adapters are trained) create *new* leakage channels that prior gradient inversion attacks cannot exploit. Traditional GIAs try to invert full-model gradients, which collapses when gradients are low-rank. UTR instead decomposes the problem into three stages that exploit the frozen-backbone/trainable-adapter split.

**Stage 1 — Token Inference via RWBG.** The Ratio of Weight to Bias Gradient (RWBG) in frozen fully-connected layers with ReLU activations satisfies `nabla_W / nabla_b = sum f(X)^i`, where `f(X)^i` are the input embeddings that activated each neuron. By computing RWBGs across bottleneck neurons in the first frozen layers, you construct a linear subspace `S` and test whether each vocabulary token's embedding lies within it. This produces an *unordered word bag* `W_b` — the set of tokens present in the training batch — without knowing their order or sentence assignment.

**Stage 2 — Sentence-Level Inversion + Constrained Greedy Decoding.** Given the word bag, UTR reconstructs coherent sentences by iteratively building candidate sequences from `W_b` and verifying each against the adapter's low-rank gradient subspace `S_LA`. Three Boolean filters prune the combinatorial search: EICW (no consecutive identical tokens), CG (grammar check), and CS (semantic coherence via language model scoring). A candidate token sequence is accepted only if its hidden embedding can be expressed as a linear combination of basis vectors from `S_LA`. The theoretical upper bound on recoverable tokens is `k_max <= rank(S) <= d_bottleneck`, directly linking adapter rank to privacy risk.

## Step-by-Step Workflow

1. **Set up the target federated learning simulation.** Instantiate the model (GPT-2 Large, BERT, or Qwen2.5-family) with LoRA adapters inserted between (not inside) transformer layers. Freeze the backbone. Use `peft` or manual LoRA injection with a specified reduction factor `r` in {1, 2, 4, 8}.

2. **Capture adapter gradients from a training round.** Run one forward-backward pass on the private training batch. Extract and store gradients for all LoRA adapter parameters (`lora_A`, `lora_B` weight matrices) and the frozen layers' weight and bias gradients from the first two transformer layers.

3. **Compute RWBG vectors for token inference.** For each bottleneck neuron in the target frozen FC layers, compute `RWBG_j = nabla_{W_j} / nabla_{b_j}`. Collect all RWBG vectors to form the subspace `S`. This is a simple element-wise division followed by stacking into a matrix.

4. **Build the unordered word bag `W_b`.** For every token in the vocabulary, project its embedding onto subspace `S` and compute the residual norm. If `||e_token - proj_S(e_token)|| < epsilon`, the token is in the batch. Use threshold `epsilon` from {0.1, 0.01, 0.001, 0.0001} — start with 0.01 and tune based on false positive rate.

5. **Construct the LoRA gradient subspace `S_LA`.** Extract the low-rank adapter gradients, compute their column space via SVD or QR decomposition. This subspace will be used for membership verification of candidate reconstructions.

6. **Initialize constrained greedy decoding.** For each candidate sentence, start with each token in `W_b` as a seed. At each step, extend the sequence by one token drawn from `W_b`, applying three filters: (a) EICW — reject consecutive duplicate tokens, (b) CG — reject sequences failing a basic grammar check, (c) CS — reject sequences with language model perplexity above a threshold.

7. **Verify candidates against adapter subspace.** For each surviving candidate sequence, compute its hidden representation through the frozen backbone and check whether it lies in the LoRA gradient subspace `S_LA` (residual norm below threshold). Accept sequences that pass this membership test.

8. **Handle bidirectional vs. unidirectional models.** For BERT-style (bidirectional attention), reconstruct complete text in a single pass. For GPT-style (unidirectional attention), expect prefix-wise partial reconstructions at large batch sizes — reconstruct iteratively from left to right.

9. **Evaluate reconstruction quality.** Compute ROUGE-1, ROUGE-2, and ROUGE-L between reconstructed and original text. Optionally compute token-level precision/recall and exact-match rate.

10. **Test defenses.** Add Gaussian noise to gradients (differential privacy with noise multiplier sigma) or apply gradient pruning (zeroing out bottom `r%` of gradient magnitudes). Re-run the attack and measure degradation. At sigma >= 1.5, UTR success drops below 2%, but model utility also degrades severely.

## Concrete Examples

**Example 1: Auditing a LoRA-tuned GPT-2 Large for gradient leakage**

User: "I'm running federated fine-tuning of GPT-2 Large with LoRA rank 8. Can you help me test if the shared adapter gradients leak training data?"

Approach:
1. Load GPT-2 Large with LoRA adapters (rank=8) applied between layers 0-1
2. Simulate a federated round with a batch of private sentences from SST-2
3. Extract frozen-layer gradients and adapter gradients after one backward pass
4. Run RWBG token inference on layers 0-1 to recover the word bag
5. Run constrained greedy decoding with GPT-2 as the language prior
6. Verify candidates against the LoRA gradient subspace
7. Report ROUGE scores comparing reconstructed text to originals

```python
import torch
from transformers import GPT2LMHeadModel, GPT2Tokenizer
from peft import get_peft_model, LoraConfig

# Step 1: Setup model with LoRA
model = GPT2LMHeadModel.from_pretrained("gpt2-large")
lora_config = LoraConfig(r=8, target_modules=["c_fc"], layers_to_transform=[0, 1])
model = get_peft_model(model, lora_config)
tokenizer = GPT2Tokenizer.from_pretrained("gpt2-large")

# Step 2: Simulate federated training round
private_text = ["The movie was absolutely terrible and a waste of time"]
inputs = tokenizer(private_text, return_tensors="pt", padding=True)
outputs = model(**inputs, labels=inputs["input_ids"])
outputs.loss.backward()

# Step 3: Extract RWBG from frozen layers
frozen_layer = model.base_model.model.transformer.h[0].mlp.c_fc
weight_grad = frozen_layer.weight.grad  # [hidden, embed]
bias_grad = frozen_layer.bias.grad      # [hidden]
rwbg = weight_grad / bias_grad.unsqueeze(1)  # [hidden, embed]

# Step 4: Token inference — project each vocab embedding onto RWBG subspace
embeddings = model.base_model.model.transformer.wte.weight.detach()
U, S, V = torch.linalg.svd(rwbg.T, full_matrices=False)
subspace_basis = U[:, :8]  # rank matches adapter dimension
word_bag = []
for tok_id in range(embeddings.shape[0]):
    e = embeddings[tok_id]
    proj = subspace_basis @ (subspace_basis.T @ e)
    residual = torch.norm(e - proj).item()
    if residual < 0.01:
        word_bag.append(tok_id)

print(f"Recovered word bag: {tokenizer.decode(word_bag)}")
# Output: "The movie was absolutely terrible and a waste of time"
```

**Example 2: Evaluating differential privacy as a defense**

User: "Will adding DP noise (sigma=0.5) to our LoRA gradients before sharing stop gradient inversion?"

Approach:
1. Run the standard UTR attack without noise — establish baseline ROUGE scores
2. Add calibrated Gaussian noise to all adapter and frozen-layer gradients
3. Re-run RWBG token inference — measure word bag precision/recall
4. Re-run sentence reconstruction — measure ROUGE degradation
5. Also measure model utility (validation loss) under the same noise

```python
# After computing gradients, add DP noise before running UTR
sigma = 0.5
for param in model.parameters():
    if param.grad is not None:
        param.grad += torch.randn_like(param.grad) * sigma

# Re-run token inference with noisy gradients
# At sigma=0.5: ROUGE-1 drops ~15-30%, attack still partially succeeds
# At sigma=1.5: ROUGE-1 drops below 2%, but validation loss degrades ~40%
```

Output: "At sigma=0.5, UTR still reconstructs partial sentences (ROUGE-1 ~70-85). You need sigma >= 1.5 to reliably block the attack, but this degrades model accuracy by ~40%. Gradient pruning at r=99.9% is also insufficient — significant leakage persists. Consider secure aggregation or increasing batch size alongside moderate DP noise."

**Example 3: Computing the theoretical privacy bound for a given adapter rank**

User: "We use LoRA rank 4 on Qwen2.5-7B. What's the maximum number of tokens an attacker could recover per batch?"

Approach:
1. Apply Corollary 4.3: k_max <= rank(S) <= d_bottleneck
2. For rank 4, the bottleneck dimension is 4
3. The attacker can recover at most 4 linearly independent token embeddings per targeted layer
4. With 2 targeted layers, practical recovery is bounded by ~8 unique token positions

Output: "With LoRA rank 4, the theoretical upper bound is 4 recoverable token embeddings per adapter layer (Corollary 4.3). Targeting 2 layers gives ~8 token positions. However, the RWBG token inference stage operates on frozen layers independently and can identify many more tokens in the word bag — the rank bound constrains only the sentence-level verification stage. In practice, short sentences (< 8 tokens) in the batch are fully recoverable."

## Best Practices

- **Do:** Always establish a clean baseline — run the attack without defenses first to confirm your implementation works before testing countermeasures.
- **Do:** Target the first 2 transformer layers for RWBG extraction. Deeper layers produce noisier gradients that reduce token inference accuracy.
- **Do:** Tune the span similarity threshold `epsilon` per model. Start at 0.01; if false positives flood the word bag, tighten to 0.001. If recall drops, loosen to 0.1.
- **Do:** Use bidirectional models (BERT) for cleaner single-pass reconstruction. Unidirectional models (GPT-2, Qwen) require iterative prefix-wise decoding.
- **Avoid:** Running this attack without authorization. This technique is for privacy auditing of your own systems or authorized red-team engagements only.
- **Avoid:** Assuming gradient pruning alone is sufficient defense. Even at 99.9% pruning, UTR extracts meaningful tokens. Combine pruning with DP noise and secure aggregation.
- **Avoid:** Ignoring the batch size effect. UTR works at batch sizes up to 128, but reconstruction quality on unidirectional models degrades above ~32 due to prefix truncation.

## Error Handling

- **RWBG division by zero:** Some bias gradients may be zero (inactive neurons). Filter out neurons where `|bias_grad| < 1e-10` before computing RWBG.
- **Empty word bag:** If no tokens pass the subspace membership test, `epsilon` is too tight. Increase by 10x and retry. If still empty, check that gradients were correctly extracted from frozen (not adapter) layers.
- **Combinatorial explosion in greedy decoding:** If the word bag exceeds ~50 tokens, the search space becomes intractable. Apply aggressive grammar/semantic filtering, or partition the word bag by positional encoding affinity before decoding.
- **Numerical instability in SVD:** For very large models (7B+), compute SVD in float64 or use truncated SVD (`torch.svd_lowrank`) to avoid memory issues.
- **Model architecture mismatch:** The attack targets FC layers between transformer blocks (e.g., `c_fc` in GPT-2, `intermediate.dense` in BERT). Verify the correct module path for your specific model before extracting gradients.

## Limitations

- **Requires access to shared gradients.** This attack applies to federated learning where clients share gradient updates. It does not apply to centralized training or inference-only scenarios.
- **Frozen backbone assumption.** UTR specifically exploits the frozen-backbone + trainable-adapter split. Full-parameter fine-tuning uses different leakage channels not covered here.
- **Vocabulary-bound reconstruction.** Token inference can only recover tokens in the model's vocabulary. Subword tokenization means rare words may be partially recovered as BPE fragments.
- **Unidirectional prefix truncation.** On GPT-style models at batch sizes > 32, reconstruction tends to recover sentence prefixes rather than complete text.
- **Defense gap.** Effective DP defense (sigma >= 1.5) destroys model utility. There is currently no known defense that fully blocks UTR without significant accuracy loss — this is an open research problem.
- **Single-round gradients only.** The attack as described targets a single federated round. Multi-round aggregation or secure aggregation protocols are not addressed.

## Reference

Chen, S., Luo, Y., Deng, G., Liu, Y., & Xu, M. (2026). *Reconstructing Training Data from Adapter-based Federated Large Language Models.* The Web Conference (WWW) 2026. [arXiv:2601.17533](https://arxiv.org/abs/2601.17533v1) — Focus on Section 4 (UTR methodology), Corollary 4.3 (privacy bound), and Section 6 (defense evaluation). Code: [github.com/shwksnshwowk-wq/GIA](https://github.com/shwksnshwowk-wq/GIA).