---
name: "alienlm-alienization-api-boundary-privacy"
description: "Implement AlienLM-style API-boundary privacy layers that protect sensitive text sent to black-box LLM APIs using vocabulary-scale bijective token remapping. Use when: 'add privacy layer to LLM API calls', 'protect prompts sent to external API', 'alienize text for API privacy', 'build token-level encryption for LLM pipeline', 'implement bijective vocabulary mapping', 'privacy-preserving LLM deployment'."
---

# AlienLM: Vocabulary-Scale Bijection for API-Boundary Privacy

This skill enables Claude to design and implement **AlienLM-style privacy layers** that protect sensitive prompts, outputs, and fine-tuning data transmitted to black-box LLM APIs. The core technique is a vocabulary-scale bijection — a one-to-one token-ID remapping that translates plaintext into an "Alien Language" before it crosses the API boundary, then losslessly recovers the original text client-side using the inverse mapping. Combined with Alien Adaptation Training (AAT), the target model learns to operate directly on alienized inputs, retaining over 81% of plaintext performance while exposing fewer than 0.22% of tokens to recovery attacks.

## When to Use

- When the user needs to **protect sensitive prompts or data** sent to external LLM APIs (medical records, legal documents, PII-laden queries)
- When building a **client-side privacy proxy** that sits between an application and a third-party LLM endpoint
- When implementing **token-level obfuscation** that preserves model performance (not encryption, but a practical privacy layer)
- When designing a **fine-tuning pipeline** where training data must not be readable by the API provider
- When the user asks about **privacy-preserving LLM deployment** under API-only access constraints
- When evaluating or comparing **prompt-protection strategies** (character substitution vs. token bijection vs. differential privacy)
- When implementing **multi-tenant key rotation** for different users or sessions sharing an LLM API

## Key Technique

**Vocabulary-Scale Bijection.** AlienLM defines a bijection `f: I → I` over the set of non-special token IDs in the model's vocabulary. Special tokens (BOS, EOS, PAD, etc.) are preserved unchanged. For each non-special token ID `i_k`, the bijection maps it to a different token ID `f(i_k)`, creating an "alien vocabulary" where every token is swapped with another. Because `f` is bijective, the inverse `f⁻¹` exists and recovery is lossless: `D(E(x)) = x`. The bijection is seeded by a secret key stored client-side — the API provider never sees the mapping. Critically, this operates at the **token-ID level**, not the character level. Character-level substitutions (like ROT13) break subword tokenizer boundaries and produce out-of-distribution token sequences, degrading performance below 25%. Token-ID bijection preserves the model's familiar subword structure.

**Bijection Optimization.** Not all bijections are equal. A random permutation works but leaves performance on the table. AlienLM optimizes the bijection by (1) maximizing the surface-form edit distance between original and mapped tokens (so alienized text looks nothing like plaintext) while (2) minimizing the embedding-space distance between paired tokens (so the model can transfer learned representations). This is solved greedily: partition token IDs into `k` buckets by seed, retrieve `k` nearest neighbors in embedding space for each token, score candidates on edit-distance vs. embedding-similarity, and greedily assign symmetric pairs. A proxy model's embeddings (e.g., Qwen for LLaMA) suffice — cross-model alignment is strong enough that proxy-based bijection performs within ~1.75 accuracy points of using the target model's own embeddings.

**Alien Adaptation Training (AAT).** After constructing the bijection, the target model is fine-tuned exclusively on alienized data. Both inputs and outputs in the training set are passed through the encoder `E` before upload to the fine-tuning API. The training objective is standard causal language modeling loss over alienized token sequences. The API provider sees only alien text during training and inference — no plaintext ever crosses the boundary. AAT uses ~300K instruction-tuning examples plus optional domain-specific data and completes in approximately 12 hours on 4×A100-equivalent compute via commercial fine-tuning APIs.

## Step-by-Step Workflow

1. **Identify the vocabulary and special tokens.** Load the target model's tokenizer. Extract the full token-ID set `I` and designate the special-token subset `S` (BOS, EOS, PAD, mask tokens, tool-call delimiters). Only tokens in `I \ S` participate in the bijection.

2. **Generate the bijection seed (secret key).** Generate a cryptographically random seed. Store it securely client-side (e.g., in a secrets manager or environment variable). This seed deterministically controls the bijection — different seeds produce different alien languages with ~1.4% pairwise token overlap.

3. **Build the optimized bijection.** Using the seed, partition non-special token IDs into `k` buckets. For each token in a bucket, retrieve its `k` nearest neighbors by embedding cosine similarity (using a proxy model's LM head if target embeddings are unavailable). Score each candidate pair by: `score = normalized_edit_distance(surface_a, surface_b) - μ * embedding_similarity(a, b)`. Greedily assign the highest-scoring symmetric pairs. This runs in under 20 minutes for a 128K vocabulary.

4. **Implement the encoder `E` and decoder `D`.** The encoder tokenizes plaintext with the model's tokenizer `τ`, remaps each non-special token ID via `f`, then decodes back to a string via `τ⁻¹`. The decoder does the reverse with `f⁻¹`. Both are pure functions — no model inference required. Wrap these as a lightweight client library.

5. **Choose the alienization ratio `ρ`.** The ratio `ρ ∈ [0, 1]` controls what fraction of the vocabulary is permuted. At `ρ = 1.0`, all non-special tokens are remapped (maximum privacy, ~81% performance recovery). At `ρ = 0.6`, ~86% performance recovery with most tokens still alienized. Select based on your privacy-performance tradeoff.

6. **Prepare alienized training data.** Take your instruction-tuning dataset `{(x_i, y_i)}`. Apply the encoder to both inputs and outputs: `{(E(x_i), E(y_i))}`. Upload only the alienized pairs to the fine-tuning API. The provider never receives plaintext.

7. **Run Alien Adaptation Training (AAT).** Fine-tune the target model on the alienized dataset using the provider's standard fine-tuning API. Use standard causal LM loss. Include domain-specific alienized examples if you need strong performance on specialized tasks (e.g., adding alienized math data improves GSM8K from 41.7% to 55.5%).

8. **Deploy the inference pipeline.** At inference time: (a) client alienizes the prompt via `E`, (b) sends alien text to the API, (c) receives alien response, (d) client de-alienizes via `D`. The round trip is lossless. The API provider sees only alien tokens at every stage.

9. **Implement key rotation.** To rotate keys, generate a new seed, rebuild the bijection, re-alienize training data, and re-run AAT. Each key rotation produces an independent alien language. For multi-tenant scenarios, note that per-key AAT outperforms shared multi-seed training.

10. **Validate with recovery attack simulation.** Test your deployment against the three threat tiers: (O1) frequency analysis on alien text, (O2) partial plaintext-alien pair leakage, (O3) adversary with model weights. Verify that token recovery stays below acceptable thresholds (<0.22% for O3-level adversaries with optimized bijections).

## Concrete Examples

**Example 1: Building a privacy proxy for a medical chatbot**

```
User: "I'm building a chatbot that answers patient questions using GPT-4 via API.
Patient messages contain PHI. I need a privacy layer so the API provider
never sees real patient data."

Approach:
1. Load GPT-4's tokenizer (cl100k_base). Identify special tokens
   (e.g., <|endoftext|>, <|im_start|>, <|im_end|>).
2. Generate a random 256-bit seed, store in AWS Secrets Manager.
3. Build optimized bijection over ~100K non-special tokens using
   a proxy model (e.g., Qwen 2.5 7B embeddings for cross-model alignment).
4. Implement E/D as a Python middleware:

   import hashlib, json
   from transformers import AutoTokenizer

   class AlienProxy:
       def __init__(self, seed: bytes, tokenizer_name: str, rho: float = 1.0):
           self.tokenizer = AutoTokenizer.from_pretrained(tokenizer_name)
           self.bijection, self.inverse = self._build_bijection(seed, rho)

       def alienize(self, text: str) -> str:
           ids = self.tokenizer.encode(text, add_special_tokens=False)
           alien_ids = [self.bijection.get(i, i) for i in ids]
           return self.tokenizer.decode(alien_ids)

       def dealienize(self, alien_text: str) -> str:
           ids = self.tokenizer.encode(alien_text, add_special_tokens=False)
           plain_ids = [self.inverse.get(i, i) for i in ids]
           return self.tokenizer.decode(plain_ids)

5. Alienize 300K instruction-tuning examples + 50K medical QA pairs.
   Upload alienized data to fine-tuning API. Run AAT.
6. In production: patient message → alienize → API call → dealienize → display.

Output:
- Patient sends: "I've been having chest pain for 3 days"
- API receives: "omorphagr thixotropydef disjunc kramerwald fost 3 intermitía"
  (alien text — no PHI exposed)
- API responds in alien text
- Client dealienizes to readable medical advice
```

**Example 2: Protecting proprietary code in a coding assistant pipeline**

```
User: "We use an external LLM API for code review. Our source code is
proprietary and we can't let the provider read it. Can we use AlienLM?"

Approach:
1. Load the coding model's tokenizer. Preserve code-structural special
   tokens (indent markers, newlines) in the special-token set S.
2. Set ρ = 0.8 for a balance of privacy and code-task performance.
3. Build bijection, ensuring code-specific tokens (variable names,
   keywords) are remapped while structural tokens stay intact.
4. Prepare alienized training data from open-source code review datasets
   (e.g., CodeReviewer). Alienize both code snippets and review comments.
5. Fine-tune via AAT on the alienized code review data.
6. Deploy: developer submits code → client alienizes → API reviews
   alienized code → client dealienizes review comments.

Key consideration:
- Code structure (indentation, brackets, control flow) is partially
  preserved through special-token exemption, but variable names and
  string literals become unreadable to the provider.
- Include alienized code-specific training data to maintain review quality.
```

**Example 3: Implementing the bijection optimization algorithm**

```
User: "Show me how to build the optimized bijection, not just a random shuffle."

Approach:
1. Load proxy model embeddings (LM head weights), shape [vocab_size, dim].
2. For each non-special token i, compute k=50 nearest neighbors by
   cosine similarity in embedding space.
3. Score each candidate pair (i, j):

   def score_pair(i, j, tokenizer, embeddings, mu=0.5):
       surf_i = tokenizer.decode([i])
       surf_j = tokenizer.decode([j])
       edit_dist = normalized_levenshtein(surf_i, surf_j)
       emb_sim = cosine_similarity(embeddings[i], embeddings[j])
       return edit_dist - mu * (1 - emb_sim)  # maximize edit dist, minimize emb dist

4. Greedy symmetric assignment:

   import numpy as np
   from scipy.spatial.distance import cdist

   def build_optimized_bijection(token_ids, embeddings, tokenizer, mu=0.5, k=50):
       bijection = {}
       available = set(token_ids)
       # Precompute nearest neighbors
       emb_matrix = embeddings[token_ids]
       nn_indices = compute_knn(emb_matrix, k=k)

       for i in token_ids:
           if i in bijection:
               continue
           best_j, best_score = None, -float('inf')
           for j in nn_indices[i]:
               if j not in available or j == i:
                   continue
               s = score_pair(i, j, tokenizer, embeddings, mu)
               if s > best_score:
                   best_j, best_score = j, s
           if best_j is not None:
               bijection[i] = best_j
               bijection[best_j] = i
               available.discard(i)
               available.discard(best_j)
       return bijection

Output: A dictionary mapping each token ID to its optimized partner,
completing in <20 minutes for 128K vocabulary.
```

## Best Practices

- **Do:** Operate the bijection at the **token-ID level**, not the character level. Character-level substitutions (ROT13, Caesar cipher on ASCII) break subword tokenizer boundaries and degrade performance to below 25%.
- **Do:** Use a **proxy model's embeddings** for bijection optimization when you lack access to the target model's weights. Cross-model embedding alignment is strong enough (within ~1.75 accuracy points of using target embeddings directly).
- **Do:** Include **domain-specific alienized data** in AAT when your use case is specialized. Generic instruction-tuning alone leaves domain performance on the table (e.g., math reasoning improves from 41.7% to 55.5% with domain data).
- **Do:** Store the bijection seed as a **secret key** with the same rigor as an encryption key. The seed fully determines the bijection; if it leaks, the privacy guarantee collapses.
- **Avoid:** Alienizing special tokens (BOS, EOS, system delimiters, tool-call markers). These must remain intact for the model's control flow to function correctly.
- **Avoid:** Using a random bijection without optimization. Random permutations (SentinelLM-style) lose 9–40 accuracy points compared to embedding-optimized bijections, with the largest gap on numerical reasoning tasks.
- **Avoid:** Expecting multi-tenant key sharing to match per-key AAT. Training a single model on multiple seeds simultaneously degrades quality. Use dedicated AAT per key for production deployments.

## Error Handling

- **Tokenizer mismatch.** If the client tokenizer version differs from the one used during bijection construction, token IDs will be misaligned and dealienization will produce garbage. Pin the exact tokenizer version and store it alongside the seed.
- **Special token leakage into bijection.** If a special token is accidentally included in the permutable set, model control flow breaks (e.g., EOS gets remapped, generation never terminates). Validate the special-token exclusion list before building the bijection.
- **Vocabulary size mismatch with proxy model.** When the proxy model's vocabulary differs from the target, represent target-only tokens by averaging the proxy embeddings of their subword decomposition. Fail gracefully if a token cannot be decomposed.
- **Performance drop on specific tasks.** If alienized performance drops significantly on a particular benchmark, add domain-specific alienized training data to AAT. The technique shows the most variance on knowledge-intensive tasks (MMLU) and numerical reasoning (GSM8K).
- **Seed loss.** If the bijection seed is lost, all alienized data and model outputs become unrecoverable. Implement seed backup and recovery procedures identical to cryptographic key management.

## Limitations

- **Not encryption.** AlienLM is a privacy layer, not a cryptographic guarantee. An adversary with both model weights and sufficient compute can attempt (though largely fail at) recovery attacks. It reduces plaintext exposure, not eliminates it.
- **Requires fine-tuning access.** AAT needs the API provider to offer fine-tuning. If the provider only exposes inference endpoints (no fine-tuning), AlienLM cannot adapt the model and performance will be near zero on alienized input.
- **Performance ceiling.** Even with optimized bijections, ~19% average performance degradation remains compared to plaintext. Tasks requiring precise factual recall or complex multi-step reasoning are most affected.
- **Key rotation cost.** Rotating the bijection key requires a full AAT re-run (hours of GPU time). This is not suitable for per-request key rotation — treat it as a periodic rotation similar to certificate renewal.
- **Single-model binding.** Each AAT run produces a model adapted to one specific bijection. The alienized model cannot process plaintext, and the original model cannot process alien text. You maintain two endpoints or swap models.
- **Numerical and symbolic tokens.** Digits and punctuation have limited embedding-space neighbors, making bijection optimization less effective for heavily numeric content.

## Reference

**Paper:** [AlienLM: Alienization of Language for API-Boundary Privacy in Black-Box LLMs](https://arxiv.org/abs/2601.22710v1) — Kim & Kang, 2026. Focus on Section 3 (bijection optimization algorithm), Section 4 (AAT procedure), and Table 1 (performance recovery ratios across models and benchmarks).