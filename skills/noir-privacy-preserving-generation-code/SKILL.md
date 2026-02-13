---
name: "noir-privacy-preserving-generation-code"
description: "Design and implement privacy-preserving code generation systems using the NOIR split-architecture pattern: client-side encoder/decoder with cloud LLM enrichment, local differential privacy on token embeddings, and randomized tokenization. Use when asked to 'protect code prompts from cloud providers', 'private code generation', 'split LLM for privacy', 'NOIR architecture', 'privacy-preserving LLM deployment', or 'defend against embedding reconstruction attacks'."
---

# NOIR: Privacy-Preserving Code Generation with Split LLM Architecture

This skill enables Claude to design, implement, and advise on privacy-preserving code generation systems based on the NOIR framework. NOIR splits an open-source LLM into a lightweight client-side encoder (first attention block), a cloud-hosted middle section, and a client-side decoder (last four attention blocks). The client perturbs token embeddings with Adaptive Randomized Response (local differential privacy) and uses a randomized tokenizer (LTokenizer) before sending embeddings to the cloud, ensuring the cloud cannot reconstruct the user's prompts or generated code — even under reconstruction and frequency analysis attacks.

## When to Use

- When the user wants to deploy an LLM for code generation while keeping prompts and outputs confidential from the cloud provider
- When designing a system where proprietary code must not be visible to a hosting service (e.g., enterprise code completion behind a privacy boundary)
- When implementing differential privacy at the token embedding level for any LLM split-inference architecture
- When building defenses against embedding inversion attacks (Vec2Text, BiSR) on LLM intermediary representations
- When the user asks how to run a large code model with only a single GPU on the client and offload heavy computation to an untrusted cloud
- When evaluating the privacy-utility tradeoff for code generation under formal differential privacy guarantees

## Key Technique

**Split Architecture with Privacy Perturbation.** NOIR partitions an open-source LLM (e.g., Qwen2.5-Coder-32B, CodeLlama-7B, Llama3-8B) into three segments. The client retains the first attention block as the encoder and the last four attention blocks as the decoder. The remaining middle blocks run on the cloud. The client encodes prompts locally, sends perturbed embeddings to the cloud for enrichment, receives enriched embeddings back, and decodes them locally to generate code. This reduces client-side GPU cost by roughly 10x compared to hosting the full model.

**Adaptive Randomized Response (ARR) on Embeddings.** Each token embedding has m features (e.g., 4096 for a 7B model). ARR perturbs each feature independently: with probability p_i it keeps the original value; with probability q_{i,k} it replaces it with the corresponding feature from another token's embedding, favoring values that are already similar. The per-feature privacy budget is epsilon/m, composing to a total budget epsilon. At epsilon=13 with temperature h=0.25, the probability of reconstructing a ground-truth token from a 200-token prompt is bounded below 5.5x10^-11. This achieves epsilon-Indistinguishability (IND): for any two tokens t, t' in vocabulary V, Pr[M(e_t)=O] <= e^epsilon * Pr[M(e_t')=O], making tokens statistically indistinguishable to the cloud.

**Randomized Tokenizer (LTokenizer).** Before sending embeddings, LTokenizer performs a data-independent uniform random permutation of all token indices. Each token and its (already perturbed) embedding are assigned to a random position in the vocabulary. The cloud sees only the permuted index, so even if it intercepts gradient one-hot vectors during fine-tuning, its probability of recovering the true token index is 1/|V| (pure random guess). This defeats frequency analysis attacks: in experiments, attackers recovered zero tokens from actual client prompts or code — only common instruction-template tokens like "Write", "a", "Python" were identifiable.

## Step-by-Step Workflow

1. **Select and partition the base LLM.** Choose an open-source code model (Qwen2.5-Coder-32B-Instruct recommended for best results; CodeLlama-7B or Llama3-8B for lighter deployments). Assign the first attention block to the client encoder, the last four attention blocks to the client decoder, and all remaining middle blocks to the cloud.

2. **Implement the client encoder.** Load only the embedding layer and first transformer block on the client GPU. Given an input prompt, tokenize it locally, produce token embeddings, and run them through this single block to get encoded representations.

3. **Apply Adaptive Randomized Response (ARR) to embeddings.** For each of the m features in each token embedding, apply the randomized response mechanism: keep the original value with probability p_i = e^(epsilon/m) / (e^(epsilon/m) + |V| - 1), or replace it with another token's feature value with probability proportional to similarity. Use epsilon=13 and temperature h=0.25 as the baseline configuration.

4. **Apply LTokenizer permutation.** Generate a fixed random permutation of the full vocabulary (once per session or per deployment). Remap each perturbed token embedding to its permuted index before transmission. Store the permutation map client-side for decoding.

5. **Send perturbed embeddings to the cloud.** Transmit only the randomized, permuted embeddings over the network. The cloud feeds these into the middle transformer blocks and returns enriched embeddings. No raw tokens, no plaintext prompts cross the boundary.

6. **Decode locally on the client.** Receive enriched embeddings from the cloud. Run them through the last four attention blocks (decoder) on the client GPU. Apply the inverse LTokenizer permutation to recover actual token logits. Use standard autoregressive sampling to generate code tokens.

7. **Fine-tune with Split Tuning (STuning) if needed.** To adapt the model to a domain: compute loss from final-layer logits against target one-hot vectors on the client. Backpropagate through the decoder, send gradients for the last cloud layer back to the cloud. The cloud updates its LoRA parameters and returns gradients for the encoder. This keeps training data private while allowing cloud-side model improvement.

8. **Validate privacy guarantees.** Compute the theoretical reconstruction bound: for a prompt of n tokens at budget epsilon, the probability of full reconstruction is at most (e^epsilon / (e^epsilon + |V| - 1))^n. For epsilon=13 and |V|=92416 (CodeQwen), this is effectively zero for any prompt longer than a few tokens. Run a Vec2Text or BiSR attack simulation against your perturbed embeddings to empirically verify meaningless reconstruction.

9. **Benchmark code quality.** Evaluate on standard benchmarks to confirm acceptable utility. Target thresholds: MBPP Pass@1 >= 75%, HumanEval Pass@1 >= 76%, BigCodeBench Pass@1 >= 37%. If quality drops too far, increase epsilon (less privacy, more utility) or increase training data for STuning.

10. **Deploy as a service or VS Code extension.** Expose the cloud middle blocks as an API endpoint. Package the client encoder, decoder, ARR module, and LTokenizer as a local application or IDE extension. The client needs only a single GPU; the cloud handles the compute-intensive middle layers.

## Concrete Examples

**Example 1: Designing a NOIR deployment for enterprise code completion**

User: "We have proprietary Python codebases and want to use a 32B code model for completion, but we can't let the cloud provider see our code. Design a system."

Approach:
1. Select Qwen2.5-Coder-32B-Instruct as the base model
2. Split: client gets block 0 (encoder) + blocks 60-63 (decoder, last 4 of 64 total); cloud gets blocks 1-59
3. Client-side: ~3B parameters, fits on a single A10 or RTX 4090
4. Apply ARR with epsilon=13, h=0.25 on 4096-dim embeddings before cloud transmission
5. Generate a session-fixed random permutation over the 151,936-token vocabulary for LTokenizer
6. Cloud API accepts perturbed embedding tensors, returns enriched tensors — never sees tokens

Output architecture:
```
Client (single GPU)                    Cloud (multi-GPU cluster)
+-----------------------+              +-------------------------+
| Tokenizer             |              |                         |
| Embedding Layer       |              |  Transformer Blocks     |
| Block 0 (Encoder)     |              |  1 through 59           |
| ARR(epsilon=13, h=.25)|--perturbed-->|                         |
| LTokenizer permute    | embeddings   |  Returns enriched       |
|                       |<--enriched---|  embeddings             |
| LTokenizer unpermute  |  embeddings  |                         |
| Blocks 60-63 (Decoder)|              +-------------------------+
| Token sampling        |
+-----------------------+
```

Expected performance: HumanEval Pass@1 ~77%, BigCodeBench Pass@1 ~38.7% (1.77% drop from non-private baseline).

---

**Example 2: Implementing the ARR perturbation mechanism**

User: "Show me how to implement the Adaptive Randomized Response for token embeddings."

Approach:
1. Take a token embedding vector of dimension m
2. For each feature dimension, compute keep probability and replacement distribution
3. Apply the randomized response independently per feature

Output:
```python
import torch
import torch.nn.functional as F

def adaptive_randomized_response(
    embedding: torch.Tensor,       # shape: (seq_len, m)
    all_embeddings: torch.Tensor,  # shape: (vocab_size, m) - full embedding table
    epsilon: float = 13.0,
    temperature: float = 0.25,
) -> torch.Tensor:
    """Apply ARR to each feature of each token embedding."""
    seq_len, m = embedding.shape
    vocab_size = all_embeddings.shape[0]
    eps_per_dim = epsilon / m

    # Keep probability per dimension
    p = torch.exp(torch.tensor(eps_per_dim)) / (
        torch.exp(torch.tensor(eps_per_dim)) + vocab_size - 1
    )

    perturbed = embedding.clone()
    for i in range(m):
        # For each feature dimension, decide keep or replace
        keep_mask = torch.rand(seq_len) < p  # shape: (seq_len,)

        if (~keep_mask).any():
            # Compute similarity-based replacement distribution
            # for tokens that are being replaced
            orig_vals = embedding[~keep_mask, i]  # (num_replace,)
            candidate_vals = all_embeddings[:, i]  # (vocab_size,)

            # Similarity: closer values get higher probability
            distances = (orig_vals.unsqueeze(1) - candidate_vals.unsqueeze(0)).abs()
            logits = -distances / temperature
            replacement_probs = F.softmax(logits, dim=1)

            # Sample replacements
            indices = torch.multinomial(replacement_probs, num_samples=1).squeeze(1)
            perturbed[~keep_mask, i] = candidate_vals[indices]

    return perturbed
```

---

**Example 3: Evaluating privacy strength against reconstruction attacks**

User: "How do I verify that my NOIR deployment actually resists embedding inversion attacks?"

Approach:
1. Collect a set of test prompts with known content
2. Run them through the encoder + ARR + LTokenizer pipeline
3. Apply Vec2Text or BiSR attack models against the perturbed embeddings
4. Measure reconstruction quality with BLEU and ROUGE metrics

Output:
```python
def evaluate_privacy(
    test_prompts: list[str],
    encoder,
    arr_module,
    ltokenizer,
    attack_model,  # Vec2Text or BiSR reconstruction model
    tokenizer,
):
    """Measure attack success rate on NOIR-protected embeddings."""
    results = {"bleu_scores": [], "rouge_scores": [], "attack_success": 0}

    for prompt in test_prompts:
        # Client-side: encode and protect
        tokens = tokenizer.encode(prompt)
        raw_embeddings = encoder(tokens)
        perturbed = arr_module.perturb(raw_embeddings)
        permuted = ltokenizer.permute(perturbed)

        # Attacker's view: try to reconstruct from permuted embeddings
        reconstructed = attack_model.reconstruct(permuted)

        # Measure reconstruction quality
        bleu = compute_bleu(prompt, reconstructed)
        rouge = compute_rouge(prompt, reconstructed)
        results["bleu_scores"].append(bleu)
        results["rouge_scores"].append(rouge)

        # NOIR threat model: attack succeeds if BLEU >= 20 or ROUGE >= 0.4
        if bleu >= 20 or rouge >= 0.4:
            results["attack_success"] += 1

    total = len(test_prompts)
    print(f"Attack success rate: {results['attack_success']}/{total}")
    print(f"Mean BLEU: {sum(results['bleu_scores'])/total:.2f}")
    print(f"Mean ROUGE: {sum(results['rouge_scores'])/total:.2f}")
    # At epsilon=13, expect: attack success = 0, BLEU < 5, ROUGE < 0.1
    return results
```

At epsilon=13, reconstructed outputs are typically meaningless fragments like `{'<s>', '_', 'a', '.', 'to'}` — no semantic content is recoverable.

## Best Practices

- **Do** use epsilon=13 with h=0.25 as your starting point — this is the sweet spot identified across all benchmarks, providing strong privacy with less than 1.2% utility drop on HumanEval
- **Do** generate the LTokenizer permutation once per deployment and store it securely client-side — it must remain secret from the cloud but consistent across inference calls for the same session
- **Do** keep the encoder minimal (single attention block) and decoder at four blocks — this ratio was validated across multiple model families and scales
- **Do** use STuning with domain-specific data if your use case diverges from general coding — 376k training samples at epsilon=27 can push MBPP from ~65% to ~76%
- **Avoid** sending raw token IDs, attention masks, or any unperturbed metadata to the cloud — even position encodings should be computed client-side to prevent leakage
- **Avoid** using epsilon < 5 in production unless you can tolerate significant quality degradation — at epsilon=1, Pass@1 drops substantially (though Pass@10 remains viable at ~50%)
- **Avoid** reusing the same LTokenizer permutation across different clients — if the cloud collates multiple permutations, it may narrow down token identities through intersection attacks

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Generated code quality drops sharply | Epsilon too low (too much noise) | Increase epsilon from 1 toward 13; verify h=0.25 |
| Cloud returns NaN or unstable enriched embeddings | Perturbed embeddings fall outside the distribution the middle blocks expect | Apply gradient clipping during STuning; ensure ARR replacement candidates come from the same model's embedding table |
| LTokenizer permutation mismatch between encode/decode | Permutation map corrupted or not persisted | Store permutation as a deterministic seed-based generation; regenerate from seed on restart |
| Client GPU OOM with decoder blocks | Model too large for available VRAM | Use 7B model instead of 32B; or quantize client-side blocks to 4-bit (encoder/decoder are small enough that quantization loss is minimal) |
| Frequency analysis reveals instruction template tokens | Fixed instruction prefixes like "Write a Python function" repeat across prompts | Randomize instruction templates or apply ARR to instruction tokens as well, not just the user-specific content |

## Limitations

- **Honest-but-curious threat model only.** NOIR does not protect against a malicious cloud that actively manipulates model weights or returns adversarial enriched embeddings. If the cloud modifies the middle blocks to act as a side channel, privacy guarantees break.
- **Open-source LLMs required.** The architecture requires splitting the model, which is only possible with open-weight models. Proprietary APIs (GPT-4, Claude) cannot be used as the cloud component.
- **Client still needs a GPU.** While the cost is ~10x lower than hosting the full model, the client must run the encoder and decoder blocks locally. CPU-only inference is impractical for the decoder's four transformer blocks at 7B+ scale.
- **No protection for output semantics.** NOIR protects raw tokens and embeddings, but if the cloud can observe side channels (timing, embedding tensor shapes, sequence lengths), it may infer partial information about the task category.
- **Adaptive attacks excluded.** The threat model does not cover an attacker who crafts specific inputs to probe the privacy mechanism. An adversary who can trigger chosen-plaintext queries against the system is out of scope.
- **Training data distribution matters.** STuning effectiveness depends on the fine-tuning data being representative. If the domain-specific code is highly unusual (e.g., esoteric DSLs), the enriched embeddings from the cloud may not capture sufficient signal, and utility will degrade regardless of epsilon.

## Reference

**Paper:** [NOIR: Privacy-Preserving Generation of Code with Open-Source LLMs](https://arxiv.org/abs/2601.16354v1) (Nguyen et al., USENIX Security 2026)
**Key sections to read:** Section 3 for the formal IND definition and ARR mechanism; Section 4 for the LTokenizer construction; Section 6 for benchmark results and attack evaluations. Look specifically at Table 2 for the epsilon-utility tradeoff curves and Table 4 for attack success rates across different threat scenarios.