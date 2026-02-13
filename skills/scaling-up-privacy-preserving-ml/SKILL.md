---
name: "scaling-up-privacy-preserving-ml"
description: "Design and implement privacy-preserving LLM inference systems using CKKS fully homomorphic encryption with unbalanced chunked prefill. Use when: 'encrypt LLM inference', 'private inference with FHE', 'CKKS homomorphic encryption for transformers', 'confidential LLM queries', 'privacy-preserving language model', 'homomorphic matrix multiplication for neural networks'."
---

# Privacy-Preserving LLM Inference with CKKS Homomorphic Encryption

This skill enables Claude to design, architect, and implement systems for running large language model inference on encrypted data using the CKKS fully homomorphic encryption scheme. The core technique is an **unbalanced chunked prefill framework** that partitions input tokens into a large public context and a small encrypted private segment, then routes computation through three distinct pipelines (plaintext-plaintext, plaintext-ciphertext, ciphertext-ciphertext) with tailored algorithms for each. This approach, drawn from Park et al. (2026), achieves practical Llama-2-7B inference on 4096 tokens (128 encrypted) in 85 seconds on consumer GPUs.

## When to Use

- When the user wants to build a system where an LLM processes queries without seeing the user's sensitive input in plaintext
- When designing a confidential inference API where the server never decrypts the client's private prompt tokens
- When implementing CKKS homomorphic encryption operations for neural network layers (matrix multiplications, attention, non-linear activations)
- When the user needs to approximate non-linear functions (SiLU, softmax, RMSNorm) with low-degree polynomials for FHE evaluation
- When mitigating activation outliers in transformer models to enable fixed-point encrypted computation without retraining
- When architecting a GPU-accelerated FHE pipeline for transformer inference with bootstrapping budget management
- When the user asks how to split a prompt into public context and private query for partial encryption

## Key Technique

### Unbalanced Chunked Prefill

Standard FHE-based LLM inference encrypts all tokens, making cost scale linearly with input length and limiting practical systems to ~128 tokens. The key insight is that most real-world prompts contain a benign context (system prompt, retrieved documents, few-shot examples) and a small sensitive query. The **unbalanced chunked prefill** splits inference into three phases:

1. **Public prefill** (plaintext-plaintext): Process the large context (e.g., 3968 tokens) entirely in cleartext, generating a KV cache. This runs at normal LLM speed.
2. **Private prefill** (mixed): Process the encrypted tokens (e.g., 128) using plaintext-ciphertext (PC) operations for linear layers (QKV projections, FFN) and ciphertext-ciphertext (CC) operations for attention between encrypted queries. The PC path is designed to be "almost bootstrap-free" by keeping multiplicative depth shallow. CC operations are confined to the small encrypted block, drastically reducing cost.
3. **Decode** (mixed): Autoregressive generation where each new encrypted token attends to the full (mostly plaintext) KV cache via matrix-vector PC operations.

### Outlier Mitigation Without Retraining

Transformer activations contain large outlier values (magnitudes >2000) that force high-degree polynomial approximations in FHE, increasing depth and cost. Two techniques compress activation ranges by ~99.7%:

- **Token prepending**: Precompute KV cache entries from special tokens and prepend them. This acts as an "attention sink" that absorbs outlier energy, reducing RMSNorm input magnitudes from ~2244 to ~7.65.
- **Adaptive space rotation**: Apply learned orthogonal transformations to hidden states, spreading concentrated outliers across dimensions. These rotations fuse into adjacent weight matrices at zero inference cost.

Together, these enable 12-bit fixed-point precision with no accuracy degradation (perplexity matches FP16 baseline at 5.49 for Llama-2-7B).

### Optimized Homomorphic Algorithms

- **Baby-step giant-step (BSGS) matrix multiplication**: Reduces rotation count from O(d) to O(sqrt(d)) for PC matrix-matrix products.
- **Depth-1 PCMM**: A modified algorithm that drops from 3 multiplicative levels to 1 by exploiting free plaintext rotations and fusing encoding conversions.
- **Slim polynomial evaluation**: For sparsely-packed ciphertexts, decomposes high-degree polynomials via sum-of-squares into a binary tree structure, achieving O(log d) key-switching operations instead of O(d). Unused SIMD slots evaluate two half-degree polynomials in parallel.

## Step-by-Step Workflow

1. **Classify input tokens as public or private.** Define the boundary: system prompt, retrieved context, and few-shot examples are public; the user's sensitive query (last N tokens, typically 64-128) is private. Implement a `TokenPartitioner` that splits token IDs into `public_tokens` and `private_tokens` arrays.

2. **Set up CKKS encryption parameters.** Choose ring degree N = 2^16 (65536 real slots in conjugate-invariant mode), scaling factor delta targeting 12-bit precision, and a ciphertext modulus chain sized for the required multiplicative depth. Precompute relinearization and rotation keys. Use a library like HEaaN, SEAL, OpenFHE, or Lattigo.

3. **Run public prefill in cleartext.** Execute standard transformer forward passes on `public_tokens` using the unmodified model weights. Cache the resulting K and V tensors for all layers. This is a normal PyTorch/CUDA operation with no FHE overhead.

4. **Apply outlier mitigation to model weights (one-time preprocessing).** (a) Compute token-prepend KV entries from special tokens and store as static cache. (b) Compute per-layer orthogonal rotation matrices via SVD or Hadamard-based methods on calibration data. Fuse rotations into adjacent weight matrices: `W_new = R @ W_original @ R^T`. Verify perplexity is preserved.

5. **Encrypt private tokens client-side.** Encode the private token embeddings into CKKS plaintext slots, then encrypt. Pack multiple values per ciphertext using row-major SIMD packing. Ship ciphertexts to the server along with a session ID referencing the public KV cache.

6. **Execute private prefill with mixed computation.** For each transformer layer:
   - **Linear projections (PC)**: Multiply plaintext weight matrices by encrypted activation vectors using depth-1 BSGS PCMM. This consumes 1 multiplicative level per projection.
   - **Attention (CC for encrypted-encrypted, PC for encrypted-plaintext)**: Compute Q_enc @ K_enc^T as CC matmul (expensive, confined to 128x128 block). Compute Q_enc @ K_plain^T as PC matmul (cheaper, covers 128x3968 block). Concatenate scores.
   - **Softmax approximation**: Use degree-15 Chebyshev polynomial for the main exponential, plus a degree-128 slim polynomial on an auxiliary track for normalization. The slim evaluation exploits unused SIMD slots.
   - **RMSNorm**: Approximate inverse-square-root with degree-7 Chebyshev polynomial on range [-2, 2] (feasible due to outlier mitigation).
   - **SiLU activation**: Degree-9 Chebyshev approximation on [-6, 6].
   - **Bootstrap** only where depth budget is exhausted (primarily after softmax and CC attention).

7. **Execute autoregressive decode loop.** For each output token: compute encrypted Q from the last hidden state, attend to the full KV cache (PC matrix-vector operations against plaintext K/V plus the small encrypted K/V block), apply FFN, and extract the next-token logits ciphertext. Send the encrypted logits to the client for decryption and sampling.

8. **Parallelize across GPUs.** Distribute PCMM and CCMM kernels across GPUs using ring-allreduce for partial-sum aggregation. Run slim polynomial evaluations (narrow ciphertexts) on a single GPU and broadcast results. Stream rotation key data rather than preloading to manage GPU memory.

9. **Decrypt and return results client-side.** The client decrypts the output logits ciphertext, applies temperature/top-p sampling, and obtains the generated token. For multi-token generation, the client re-encrypts the chosen token embedding and sends it back for the next decode step.

10. **Validate end-to-end correctness.** Compare FHE inference outputs against plaintext baseline using perplexity on standard benchmarks (WikiText-2, etc.). Acceptable threshold: perplexity difference < 0.1. Run precision simulations at each layer to track cumulative fixed-point error.

## Concrete Examples

**Example 1: Confidential Medical Query Summarization**

User: "I want to build a system where a hospital sends patient notes to a cloud LLM for summarization, but the patient's actual symptoms and diagnosis in the query must stay encrypted."

Approach:
1. Partition the prompt: the summarization instruction and format template (public, ~500 tokens) go as plaintext; the patient's clinical note (private, ~100 tokens) is CKKS-encrypted client-side.
2. Server runs public prefill on the instruction template, caching KV pairs.
3. Server runs private prefill on the encrypted clinical note using PC linear layers and CC attention within the encrypted block.
4. Server generates a summary token-by-token in decode mode. Each output logit ciphertext is sent to the hospital client.
5. Hospital decrypts logits locally, samples tokens, and reconstructs the summary.

Output architecture:
```
Client (Hospital)                    Server (Cloud)
─────────────────                    ──────────────
Encrypt(patient_note) ──────────>    Public prefill(template)
                                     Private prefill(enc_note)
                      <────────────  Enc(logits_0)
Decrypt, sample tok_0
Encrypt(tok_0_embed)  ──────────>    Decode step 1
                      <────────────  Enc(logits_1)
... repeat for N tokens ...
Final summary (plaintext, client-side only)
```

**Example 2: Implementing PC Matrix Multiplication with BSGS**

User: "Show me how to implement the baby-step giant-step plaintext-ciphertext matrix multiplication for a 4096x4096 weight matrix times an encrypted activation matrix."

Approach:
1. Choose BSGS parameters: for d=4096, set baby-step b=64, giant-step g=64 (so b*g=4096).
2. Precompute 64 diagonal-encoded plaintext vectors from the weight matrix for each baby-step index.
3. For each baby-step i in [0, 63]: rotate the ciphertext by i positions and multiply by the corresponding plaintext diagonal.
4. Accumulate the b inner products, then for each giant-step j in [0, 63]: rotate the accumulated result by j*b positions.
5. Sum all giant-step results to get the final encrypted matrix product.

Pseudocode output:
```python
def bsgs_pcmm(weight_plain, activation_ct, b=64, g=64):
    """
    Plaintext-ciphertext matrix multiplication using BSGS.
    weight_plain: d x d plaintext weight matrix
    activation_ct: CKKS ciphertext encoding d x k activation matrix
    Returns: encrypted result of weight @ activation
    """
    d = b * g  # 4096
    diagonals = extract_diagonals(weight_plain, d)  # d diagonal vectors

    result_ct = zero_ciphertext()
    for j in range(g):
        inner_sum = zero_ciphertext()
        for i in range(b):
            diag_idx = j * b + i
            pt_diag = encode_plaintext(diagonals[diag_idx])
            rotated = rotate(activation_ct, i)       # 1 rotation
            inner_sum += pt_diag * rotated            # pt-ct multiply (free depth)
        result_ct += rotate(inner_sum, j * b)         # 1 rotation

    return result_ct  # Total: 64 + 64 = 128 rotations instead of 4096
```

**Example 3: Outlier Mitigation via Token Prepending**

User: "My encrypted transformer has activation values exceeding 2000, which blows up my polynomial approximations. How do I fix this without retraining?"

Approach:
1. Run the model on a calibration dataset and record per-layer activation statistics, identifying outlier magnitudes.
2. Construct a set of 4-8 special prepend tokens (e.g., BOS + padding tokens) and compute their KV cache entries for every layer using a single forward pass.
3. At inference time, prepend these static KV entries to the cache before any user tokens. The attention mechanism now has "sink" positions that absorb disproportionate attention weight, preventing outlier accumulation.
4. Measure post-prepend activation ranges: expect reduction from ~2244 to ~7.65 for RMSNorm inputs.
5. Additionally apply per-layer orthogonal rotations: compute rotation matrix R from SVD of activation covariance on calibration data. Replace W_q with R @ W_q, W_k with R @ W_k, etc., and apply R^T to the output projection. This spreads residual outliers evenly.
6. Verify with perplexity evaluation that accuracy is unchanged.

Output metrics:
```
Before mitigation:
  RMSNorm input range: [-2244, 2244]
  Required polynomial degree: 31+ (depth 5+)
  Bootstrapping budget: exceeded at layer 12

After mitigation:
  RMSNorm input range: [-7.65, 7.65]  (maps to [-2, 2] after scaling)
  Required polynomial degree: 7 (depth 3)
  Bootstrapping budget: sufficient through all 32 layers
  Perplexity change: 5.49 -> 5.49 (none)
```

## Best Practices

- **Do** separate public and private tokens at the prompt level rather than encrypting everything. The cost savings are enormous: a 4096-token fully-encrypted inference is infeasible, but 3968 public + 128 encrypted is practical.
- **Do** apply outlier mitigation (token prepending + rotation) before choosing polynomial approximation degrees. This is the single highest-impact optimization, reducing required polynomial degrees by 4-10x.
- **Do** use conjugate-invariant CKKS mode to get N real slots instead of N/2 complex slots, doubling throughput for real-valued activations.
- **Do** minimize multiplicative depth in PC operations by using depth-1 PCMM variants. Reserve bootstrapping budget for the unavoidable CC operations and softmax normalization.
- **Avoid** precomputing and storing all rotated plaintext matrices when moduli are large; instead compute rotation intermediates on-the-fly to prevent GPU memory exhaustion.
- **Avoid** using generic polynomial evaluation for the softmax auxiliary track; use slim polynomial decomposition (sum-of-squares binary tree) to exploit sparse SIMD packing and cut key-switching operations from O(d) to O(log d).
- **Avoid** encrypting more tokens than necessary. Each additional encrypted token increases CC attention cost quadratically. Keep the encrypted window as small as the privacy requirement allows.

## Error Handling

- **Precision overflow**: If accumulated fixed-point error causes ciphertext noise to exceed the decryption threshold, the output will be garbage. Monitor noise budget at each layer during development using the FHE library's noise estimation tools. Insert additional bootstrapping operations at layers where the budget is critically low.
- **Polynomial approximation out-of-range**: If an activation value falls outside the Chebyshev approximation domain (e.g., >6 for SiLU), the polynomial output diverges catastrophically. Always verify post-mitigation activation ranges on representative data before deploying. Add clamping logic in the plaintext preprocessing if needed.
- **Key generation failures**: CKKS rotation keys for large ring degrees consume significant memory (tens of GB). If key generation OOMs, generate rotation keys lazily for only the rotation indices actually used by the BSGS decomposition, not all possible rotations.
- **GPU memory exhaustion**: A single ciphertext at ring degree 2^16 with large modulus occupies ~500MB. Pipeline ciphertext computation across layers, freeing intermediate results eagerly. Use CPU-GPU streaming for rotation keys.
- **Accuracy regression after rotation fusion**: If perplexity degrades after fusing orthogonal rotations into weights, the rotation matrices may not be truly orthogonal due to numerical errors. Re-orthogonalize using QR decomposition and verify R @ R^T = I within floating-point tolerance.

## Limitations

- **Encrypted token count is limited.** The system is designed for 64-128 encrypted tokens. Scaling to thousands of encrypted tokens would require CC attention over much larger blocks, making costs prohibitive with current hardware.
- **No encrypted system prompts.** The architecture assumes the context/system prompt is public. If the entire prompt must be private, this approach does not apply and fully-encrypted (much slower) methods are needed.
- **CKKS is approximate.** Unlike exact FHE schemes (BFV/BGV), CKKS introduces small numerical errors at each operation. This is acceptable for ML inference but unsuitable for applications requiring exact computation.
- **Single next-token output.** The server returns encrypted logits for one token at a time. Speculative decoding or parallel token generation in the encrypted domain is an open problem.
- **Hardware requirements are substantial.** Even with all optimizations, 8 high-end GPUs are needed for practical latency. Single-GPU inference of Llama-2-7B scale models under FHE is not yet feasible at interactive speeds.
- **No fine-tuning under encryption.** This approach covers inference only. Training or fine-tuning on encrypted data requires different techniques (e.g., secure multi-party computation).
- **Library ecosystem is immature.** Production-grade CKKS GPU libraries (HEaaN, OpenFHE-GPU) are limited in availability. Many implementations will require custom CUDA kernels.

## Reference

Park, J., Park, S., Park, J.H., Ahn, J.H., & Cheon, J.H. (2026). *Scaling up Privacy-Preserving ML: A CKKS Implementation of Llama-2-7B*. arXiv:2601.18511v1. https://arxiv.org/abs/2601.18511v1

Key sections to study: Section 3 (unbalanced chunked prefill framework and computation type taxonomy), Section 4 (homomorphic matrix multiplication algorithms including depth-1 PCMM and BSGS), Section 5 (slim polynomial evaluation), and Section 6 (outlier mitigation via token prepending and adaptive space rotation).