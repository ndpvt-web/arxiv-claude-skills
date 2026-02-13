---
name: "zero2text-zero-training-cross-domain-inversion"
description: "Implement embedding inversion attacks that reconstruct original text from vector embeddings without training data, using recursive online alignment with ridge regression. Use when: 'audit my RAG pipeline for embedding inversion leaks', 'test if embeddings in my vector DB can be reversed', 'build a zero-shot embedding-to-text decoder', 'red-team my embedding API for privacy', 'assess differential privacy defenses on my embedding service', 'recover text from OpenAI embeddings without training data'."
---

# Zero2Text: Training-Free Embedding Inversion via Recursive Online Alignment

This skill enables Claude to implement and apply the Zero2Text framework for reconstructing original text from embedding vectors without any training data or in-domain examples. The technique uses recursive online alignment -- combining LLM token-level generation with a dynamically-updated ridge regression projector -- to iteratively decode text from black-box embedding APIs. This is a security auditing tool for testing whether vector databases and RAG systems leak private text through their embeddings.

## When to Use

- When the user asks to **red-team or audit a RAG pipeline** for embedding inversion vulnerabilities
- When the user wants to **test whether their vector database embeddings can be reversed** into original text
- When the user needs to **evaluate differential privacy defenses** (Laplace, Purkayastha mechanisms) on embedding APIs
- When the user asks to **build a black-box embedding-to-text recovery system** without access to model weights or training data
- When the user wants to **benchmark embedding privacy** across providers (OpenAI, open-source models)
- When the user is working on a **CTF challenge or security research** involving embedding inversion

## Key Technique

Zero2Text solves the embedding inversion problem -- given only a target embedding vector `e_v` from a black-box API, reconstruct the original text -- without any offline training data. Previous approaches either required thousands of expensive optimization queries (Vec2Text) or assumed access to in-domain text-embedding pairs for training an inversion model (ALGEN). Zero2Text needs neither.

The core mechanism is **recursive online alignment**. At each token position, an LLM (e.g., Qwen3-0.6B) proposes ~1000 diverse candidate next-tokens based on its language prior. These candidates are embedded locally (using a lightweight model like `all-mpnet-base-v2`) and projected into the victim's embedding space via a ridge regression matrix `W^t`. A small subset of the best candidates is queried against the actual victim API, and the returned embeddings update `W^t` in closed form: `W^t = (E^t'E^t + lambda*I)^{-1} E^t' E_tilde^t`. This means the projector improves with every token generated, making later tokens cheaper (fewer queries needed) and more accurate.

The scoring combines LLM logits (ensuring grammatical fluency) with projected embedding similarity (ensuring semantic fidelity), weighted by a confidence term that reflects how reliable the current projector is. Beam search (width 10) maintains the top candidate sequences across up to 32 token positions. The result: text recovery that achieves 1.8x higher ROUGE-L and 6.4x higher BLEU-2 than baselines on MS MARCO against OpenAI embeddings, using ~6x fewer API queries.

## Step-by-Step Workflow

1. **Define the threat model.** Identify the victim embedding API (e.g., OpenAI `text-embedding-3-large`), confirm you have black-box query access to it, and obtain the target embedding vector `e_v` you want to invert. You do NOT need model weights, gradients, or any training data.

2. **Set up the local embedding model.** Install `sentence-transformers` and load `all-mpnet-base-v2` (768-dim) as the attacker's local embedder. This model is used to embed candidate tokens cheaply without querying the victim API.

3. **Initialize the LLM generator.** Load a small LLM (Qwen3-0.6B or similar) for token-level candidate generation. Configure it to output logits over the vocabulary, restricted to ASCII tokens. Apply a logit penalty of `-5.0` to non-alphabetic tokens on the first iteration to bootstrap coherent text.

4. **Generate diverse candidates at each token position.** From the LLM's logit distribution over the current context, select `K_S = 1000` candidate tokens, enforcing pairwise cosine similarity below `T_hw = 0.9` in the local embedding space to ensure diversity.

5. **Project candidates into the victim's space.** Embed each candidate locally, then multiply by the current ridge regression matrix `W^t` to estimate where each candidate falls in the victim's embedding space. Rank candidates by cosine similarity to `e_v`.

6. **Query the victim API for top candidates.** Send the top `K_A * gamma^(t-1)` candidates (start with `K_A = 50`, `gamma = 0.8`) to the victim embedding API. Collect the returned embeddings `E_tilde^t`. This is where the budget is spent -- the exponential decay means later tokens cost fewer queries.

7. **Update the projection matrix via ridge regression.** Accumulate all (local_embedding, victim_embedding) pairs seen so far into matrices `E^t` and `E_tilde^t`. Compute the closed-form solution: `W^t = (E^t' * E^t + 0.1 * I)^{-1} * E^t' * E_tilde^t`.

8. **Re-score all candidates with the updated projector.** For the non-queried candidates, re-project using the improved `W^t`. Compute the hybrid score: `S = Z(logit_i) + conf_t * Z(cos(e_i * W^t, e_v))`, where `conf_t` is the mean cosine accuracy of the projector on known pairs.

9. **Run beam search.** Keep the top `K_B = 10` sequences ranked by cumulative score. Feed the best beams back as context for the next token position.

10. **Iterate until termination.** Repeat steps 4-9 for up to `T = 32` tokens or until the LLM generates an EOS token. Return the top-1 beam as the recovered text.

## Concrete Examples

**Example 1: Auditing an OpenAI RAG pipeline for embedding leakage**

```
User: I have a RAG system using OpenAI text-embedding-3-large. I want to test
whether an attacker with API access could recover the original documents from
stored embeddings. Can you build a proof-of-concept?

Approach:
1. Install dependencies: pip install sentence-transformers transformers numpy openai
2. Write a Python script that:
   a. Loads all-mpnet-base-v2 as the local embedder (768-dim)
   b. Loads Qwen3-0.6B (or a local small LLM) for candidate generation
   c. Accepts a target embedding vector (1536-dim for text-embedding-3-large)
   d. Implements the recursive online alignment loop:
      - Generate 1000 diverse token candidates per position
      - Project via W^t, query top candidates against OpenAI API
      - Update W^t with ridge regression (lambda=0.1)
      - Hybrid-score with confidence-weighted cosine + logit z-scores
      - Beam search with width 10
3. Run against 10-20 sample embeddings from the vector DB
4. Measure ROUGE-L and BLEU-2 between recovered and original text
5. Report: "X% of documents had ROUGE-L > 0.25, indicating meaningful leakage"

Output (example recovered text):
Original:  "The patient presented with acute respiratory distress and was admitted to the ICU"
Recovered: "The patient presented with acute respiratory failure and was admitted to intensive care"
ROUGE-L: 0.71 | BLEU-2: 0.42 | Cosine: 0.89
```

**Example 2: Evaluating differential privacy defenses on embeddings**

```
User: We're adding noise to our embeddings before storing them. Can you test
whether the Laplace mechanism with epsilon/d=0.5 actually protects against
inversion?

Approach:
1. Implement the Laplace mechanism: add Laplace(0, d/epsilon) noise to each
   dimension of the embedding before storage (d = embedding dimension)
2. Run Zero2Text against both clean and noised embeddings
3. Sweep epsilon/d over {0.25, 0.5, 1.0, 2.0, 4.0}
4. Compare ROUGE-L scores at each privacy level
5. Key finding from the paper: at epsilon/d=0.25 (strongest defense),
   Zero2Text still achieves ROUGE-L ~13.75, which is competitive with
   undefended baseline methods (~14.19 for ALGEN)

Output:
| epsilon/d | ROUGE-L (clean) | ROUGE-L (noised) | % Degradation |
|-----------|-----------------|-------------------|---------------|
| 4.0       | 26.08           | 23.41             | 10.2%         |
| 1.0       | 26.08           | 19.87             | 23.8%         |
| 0.5       | 26.08           | 16.52             | 36.6%         |
| 0.25      | 26.08           | 13.75             | 47.3%         |

Conclusion: Even at epsilon/d=0.25, recovered text retains semantic meaning.
Differential privacy alone is insufficient -- consider access controls, rate
limiting, or embedding truncation as complementary defenses.
```

**Example 3: Building the ridge regression projector from scratch**

```
User: I want to understand the core alignment mechanism. Can you implement
just the ridge regression projector that maps between embedding spaces?

Approach:
1. Collect paired embeddings: embed the same N texts with both
   all-mpnet-base-v2 (768-dim -> matrix E) and the victim model (-> matrix E_tilde)
2. Compute W = (E^T @ E + lambda * I)^{-1} @ E^T @ E_tilde
3. Test projection quality on held-out pairs

Implementation:
```python
import numpy as np
from sentence_transformers import SentenceTransformer

def build_projector(texts, victim_embed_fn, lambda_reg=0.1):
    """Build ridge regression projector from text samples."""
    local_model = SentenceTransformer('all-mpnet-base-v2')
    E_local = local_model.encode(texts)           # (N, 768)
    E_victim = victim_embed_fn(texts)              # (N, d_victim)

    # Ridge regression closed-form
    EtE = E_local.T @ E_local                     # (768, 768)
    reg = lambda_reg * np.eye(EtE.shape[0])
    W = np.linalg.solve(EtE + reg, E_local.T @ E_victim)  # (768, d_victim)
    return W

def project_and_score(local_emb, W, target_emb):
    """Project local embedding and compute cosine similarity to target."""
    projected = local_emb @ W
    cos_sim = np.dot(projected, target_emb) / (
        np.linalg.norm(projected) * np.linalg.norm(target_emb)
    )
    return cos_sim
```

Key insight: W improves as more pairs accumulate during the attack.
With 50 pairs, projection cosine accuracy is ~0.6. By 500 pairs
(after ~15 tokens), it exceeds 0.85.
```

## Best Practices

**Do:**
- Start with a small `K_A` (50) and let the exponential decay (`gamma=0.8`) reduce queries naturally -- the projector gets better over time, compensating for fewer queries
- Enforce candidate diversity (`T_hw=0.9` cosine threshold) to prevent the beam from collapsing to trivial variations of the same token
- Use z-score normalization on both logit and cosine components of the hybrid score to prevent one signal from dominating
- Accumulate ALL (local, victim) embedding pairs across all token positions -- the ridge regression benefits from every observation
- Rate-limit your victim API queries and track total cost; a typical 32-token inversion uses ~13.9k tokens of API queries

**Avoid:**
- Do not skip the LLM logit component in scoring -- without it, recovered text is semantically close but grammatically broken
- Do not use a fixed confidence weight; the dynamic `conf_t` is essential because the projector is unreliable in early iterations
- Do not apply this technique against systems without explicit authorization -- this is a security auditing tool
- Do not assume differential privacy alone will defend against this attack; the paper shows it degrades but does not prevent meaningful recovery

## Error Handling

- **Victim API rate limits:** Implement exponential backoff. The decay factor `gamma=0.8` already reduces queries per token; if hitting limits, reduce `K_A` from 50 to 30 or increase `gamma` to 0.85.
- **Dimension mismatch:** If the victim embedding dimension is unknown, query a single known text first to determine it. The projector `W` adapts to any target dimension.
- **Degenerate projector in early iterations:** When `t=1`, the projector has few samples. The framework handles this via the confidence weight `conf_1 = 0.7 * mean_cosine`, which down-weights unreliable projections. If results are poor, increase `K_A` for the first 2-3 iterations.
- **Beam collapse:** If all 10 beams converge to near-identical prefixes, increase the diversity threshold `T_hw` from 0.9 to 0.95, or increase beam width to 15.
- **Out-of-vocabulary tokens:** Restrict the LLM vocabulary to ASCII tokens and apply the `-5.0` logit penalty for non-alphabetic tokens only at `t=1`. For later positions, the LLM context provides sufficient constraint.

## Limitations

- **Requires embedding API query access.** This is a black-box attack but still needs the ability to send arbitrary text to the victim's embedding endpoint and receive vectors back. Systems that restrict API access or enforce authentication inherently limit exposure.
- **Token budget is non-trivial.** Each inversion costs ~13.9k embedding API tokens. Inverting an entire vector database of 100k documents would be expensive and likely trigger rate limits or anomaly detection.
- **Short text bias.** The method generates up to 32 tokens (~1-2 sentences). Longer documents are only partially recovered -- typically the most semantically dominant sentence.
- **Semantic vs. lexical recovery.** The method recovers meaning more reliably than exact wording. Synonyms and paraphrases are common (e.g., "ICU" vs. "intensive care"). ROUGE-L of ~26 on MS MARCO reflects this gap.
- **Attacker LLM quality matters.** Using a weaker or larger LLM changes the cost-quality tradeoff. Qwen3-0.6B balances speed and quality; smaller models degrade substantially.
- **Not applicable to non-text embeddings.** This technique is designed for textual embeddings. Image or multimodal embeddings require different approaches.

## Reference

**Paper:** [Zero2Text: Zero-Training Cross-Domain Inversion Attacks on Textual Embeddings](https://arxiv.org/abs/2602.01757v2) (Kim et al., 2026)
**Key sections to read:** Section 3 (Method) for the recursive online alignment algorithm and ridge regression formulation; Section 4.4 for defense evaluation showing differential privacy limitations; Table 1 for cross-domain benchmark results.