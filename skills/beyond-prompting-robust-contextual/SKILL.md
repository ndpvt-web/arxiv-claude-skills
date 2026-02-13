---
name: beyond-prompting-robust-contextual
description: >
  Implement logit-space contextual biasing (LOGIC) for speech and language model
  pipelines that need to recognize domain-specific entities — contact names, product
  names, playlist titles, technical jargon — without stuffing them into prompts.
  Builds trie-based entity indexes and applies bias scores at the token-logit level
  during decoding, avoiding prompt bloat and the "lost-in-the-middle" problem.
  Trigger phrases:
  - "bias ASR output toward my entity list"
  - "contextual biasing for speech recognition"
  - "recognize custom entity names without prompt stuffing"
  - "logit-space entity biasing"
  - "trie-based decoding bias for LLM"
  - "reduce entity word error rate in transcription"
---

# LOGIC: Logit-Space Integration for Contextual Biasing

This skill teaches Claude to implement the LOGIC framework — a decoding-layer technique that biases speech or language model outputs toward a user-supplied entity list without injecting those entities into the prompt. Instead of prompting (which scales poorly with large entity catalogs) or post-hoc error correction (which hallucinates entities that were never spoken), LOGIC intercepts the token probability distribution at each decoding step and nudges it toward entity-matching continuations using a trie-indexed shallow fusion approach.

## When to Use

- When the user is building an ASR or speech LLM pipeline and needs to recognize domain-specific names (contacts, products, songs, medical terms) that the base model was not trained on.
- When the user has a large entity catalog (hundreds to thousands of entries) that cannot fit in a prompt context window without degrading quality.
- When the user reports "lost-in-the-middle" issues — the model ignores entities placed in the middle of a long prompt-based bias list.
- When the user wants to add contextual biasing to an existing LLM decoding loop (e.g., Hugging Face `generate`, vLLM, or a custom autoregressive decoder).
- When the user needs to reduce Entity Word Error Rate (Entity WER) without increasing False Alarm Rate — i.e., without hallucinating entities that were never in the audio.
- When the user asks about alternatives to Generative Error Correction (GEC) for fixing entity recognition mistakes in transcripts.

## Key Technique

**The core problem.** Speech LLMs are trained on static corpora and struggle with new or rare entities. The naive fix — prepend a list of entities to the prompt — degrades as the list grows: context windows fill up, latency increases, and models attend poorly to items in the middle of long lists. Post-processing with GEC rewrites transcripts after the fact but tends to "over-correct," inserting entities the speaker never said.

**What LOGIC does differently.** LOGIC operates at the logit layer during autoregressive decoding. It maintains a prefix trie built from the tokenized forms of every entity in the bias list. At each decoding step, the trie tracks which entities are still viable completions given the tokens generated so far. For those viable continuations, LOGIC adds a bias score to the corresponding token logits before the softmax. This is a form of *shallow fusion*: the base model's distribution is interpolated with a sparse bias distribution derived from the trie. The key formula is:

```
logit_final[t] = logit_base[t] + lambda * bias[t]
```

where `bias[t]` is nonzero only for tokens that continue a valid entity prefix in the trie, and `lambda` controls bias strength. Because the trie lookup is O(1) per step (advance one node), the method adds constant overhead regardless of how many entities are in the list — unlike prompting, which scales linearly with entity count.

**Robustness against false alarms.** A naive bias would push the model to hallucinate entities in every utterance. LOGIC controls this with two mechanisms: (1) the bias is only applied when the trie has active prefixes matching the current generation, and (2) the bias strength `lambda` is tuned to keep the False Alarm Rate negligible (0.30% in the paper's experiments across 11 locales). An optional confidence threshold can deactivate biasing when the base model's top-token probability already exceeds a threshold, preventing interference with high-confidence predictions.

## Step-by-Step Workflow

1. **Collect and normalize the entity list.** Gather domain-specific entities (contact names, product names, jargon terms) into a flat list. Normalize casing and strip diacritics if the tokenizer is case-sensitive, or preserve them if the model handles mixed case.

2. **Tokenize every entity using the target model's tokenizer.** Each entity becomes a sequence of token IDs. Store these as `List[List[int]]`. This must use the exact tokenizer of the model that will perform decoding — mismatches here silently break biasing.

3. **Build a prefix trie from the tokenized entities.** Each path from root to a leaf represents one entity's full token sequence. Internal nodes store the set of valid next-token IDs. Mark leaf nodes to indicate entity completion.

   ```python
   class TrieNode:
       def __init__(self):
           self.children: dict[int, TrieNode] = {}
           self.is_entity_end: bool = False

   def build_trie(tokenized_entities: list[list[int]]) -> TrieNode:
       root = TrieNode()
       for token_ids in tokenized_entities:
           node = root
           for tid in token_ids:
               if tid not in node.children:
                   node.children[tid] = TrieNode()
               node = node.children[tid]
           node.is_entity_end = True
       return root
   ```

4. **Initialize trie cursors at decoding start.** Maintain a set of "active cursors" — pointers into the trie that track partially matched entity prefixes. At the start of decoding, the only cursor is at the root. After each token is generated, advance cursors that match, spawn new root cursors (to catch entities starting at any position), and prune cursors that have no valid continuation.

5. **At each decoding step, compute the bias vector.** Create a sparse bias vector of size `vocab_size`, initialized to zero. For every active cursor, look at its children in the trie — each child's token ID gets a bias value added. Use a higher bias for cursors deeper in the trie (partial matches that are close to completing an entity) to reinforce commitment.

   ```python
   def compute_bias(cursors: list[TrieNode], vocab_size: int,
                    lambda_val: float, depth_bonus: float = 0.0,
                    depths: list[int] = None) -> torch.Tensor:
       bias = torch.zeros(vocab_size)
       for i, cursor in enumerate(cursors):
           d = depths[i] if depths else 0
           for token_id in cursor.children:
               bias[token_id] += lambda_val + d * depth_bonus
       return bias
   ```

6. **Add the bias to base logits before softmax.** This is the shallow fusion step. The modified logits feed into the normal sampling or beam search procedure.

   ```python
   logits = model.get_next_token_logits(input_ids)
   bias = compute_bias(active_cursors, logits.size(-1), lambda_val=3.0)
   logits = logits + bias
   next_token = torch.argmax(logits, dim=-1)  # or sample
   ```

7. **Update cursors after each generated token.** Advance cursors whose children match the generated token. Always spawn a fresh root cursor so new entity matches can begin at any position. Remove cursors with no matching child (dead branches).

8. **Apply a confidence gate (optional).** If the base model's top token probability exceeds a threshold (e.g., 0.95), skip biasing for that step. This prevents overriding high-confidence predictions and keeps the False Alarm Rate low.

9. **Tune `lambda` on a held-out set.** Start with `lambda=3.0` and sweep `[1.0, 2.0, 3.0, 5.0, 8.0]`. Measure Entity WER (how often entities are recognized correctly) and False Alarm Rate (how often entities are hallucinated). Pick the lambda that maximizes Entity WER reduction while keeping FAR below 1%.

10. **Integrate into the serving loop.** Wrap the bias logic into a `LogitsProcessor` (Hugging Face) or equivalent hook in your decoding framework. The entity trie can be rebuilt per-request (for personalized lists) or cached (for shared catalogs) — trie construction from 10K entities takes <50ms.

## Concrete Examples

**Example 1: Biasing a Speech LLM for Contact Names**

User: "I'm using Whisper with a Hugging Face pipeline for voicemail transcription. My users have contact lists of 200-500 names. The model keeps getting names wrong — 'Call John Malkovich' becomes 'Call John Malcolm Rich'. How do I fix this without stuffing all names into the prompt?"

Approach:
1. Export each user's contact list as a plain text file, one name per line.
2. Tokenize every contact name with the Whisper tokenizer.
3. Build a prefix trie from the tokenized names.
4. Implement a custom `LogitsProcessor` that maintains trie cursors and adds bias at each step.
5. Register the processor with `model.generate(logits_processor=[bias_processor])`.

Output:
```python
from transformers import LogitsProcessor
import torch

class EntityBiasProcessor(LogitsProcessor):
    def __init__(self, trie_root, lambda_val=3.0, confidence_gate=0.95):
        self.trie_root = trie_root
        self.lambda_val = lambda_val
        self.confidence_gate = confidence_gate
        self.cursors = []  # list of (node, depth)

    def __call__(self, input_ids, scores):
        # Confidence gate: skip if top token is very confident
        probs = torch.softmax(scores, dim=-1)
        if probs.max() > self.confidence_gate:
            self._advance_cursors(input_ids[0, -1].item())
            return scores

        # Always allow new entity matches to start
        self.cursors.append((self.trie_root, 0))

        # Build bias vector
        bias = torch.zeros_like(scores)
        for node, depth in self.cursors:
            for token_id in node.children:
                bias[0, token_id] += self.lambda_val + depth * 0.5

        scores = scores + bias
        return scores

    def _advance_cursors(self, last_token_id):
        new_cursors = []
        for node, depth in self.cursors:
            if last_token_id in node.children:
                new_cursors.append((node.children[last_token_id], depth + 1))
        new_cursors.append((self.trie_root, 0))
        self.cursors = new_cursors
```

**Example 2: Product Catalog Biasing for Voice Commerce**

User: "Our voice shopping assistant has 8,000 product names. Prompting with the full catalog is impossible. How can I make the model prefer valid product names during transcription?"

Approach:
1. Tokenize all 8,000 product names with the model's tokenizer.
2. Build a single prefix trie — construction takes ~30ms for this size.
3. At each decoding step, compute bias from the trie and add to logits.
4. Cache the trie since the product catalog changes infrequently (rebuild nightly).
5. Set `lambda=5.0` (higher bias is acceptable because the domain is constrained).

Output:
```python
# Build the trie once and cache it
product_names = load_product_catalog()  # ["AirPods Pro", "Galaxy S24", ...]
tokenized = [tokenizer.encode(name, add_special_tokens=False)
             for name in product_names]
trie_root = build_trie(tokenized)

# Use it per-request
processor = EntityBiasProcessor(trie_root, lambda_val=5.0)
output = model.generate(audio_features, logits_processor=[processor])
transcript = tokenizer.decode(output[0])
```

**Example 3: Multilingual Entity Biasing**

User: "We operate in 6 languages. Entity names often mix scripts — 'iPhone 16 Pro Max' appears the same across locales but surrounding context is in Japanese, German, etc. How do I handle this?"

Approach:
1. Tokenize entities using the multilingual model's tokenizer (e.g., Phi-4-MM). Mixed-script entities will naturally produce different token sequences per tokenizer.
2. Build one trie per locale if entity lists differ, or a single merged trie if the catalog is shared.
3. Apply the same bias logic — the trie is script-agnostic since it operates on token IDs, not characters.
4. Tune `lambda` per locale on held-out data since tokenization granularity varies (CJK tokenizers produce more tokens per entity, which may need lower lambda to avoid over-biasing early tokens).

## Best Practices

- **Do:** Tokenize entities with the exact tokenizer and settings (e.g., `add_special_tokens=False`) that the model uses during decoding. Token ID mismatch is the most common integration bug.
- **Do:** Spawn a new root cursor at every decoding step so entities can be recognized starting at any position in the output, not just at the beginning.
- **Do:** Apply a depth bonus to bias scores — tokens that continue a longer partial match deserve stronger bias since they represent higher commitment to a specific entity.
- **Do:** Tune `lambda` on representative data measuring both Entity WER and False Alarm Rate. The two metrics are in tension; the right `lambda` balances them.
- **Avoid:** Setting `lambda` too high (>10) without a confidence gate. This causes the model to hallucinate entities in unrelated utterances, dramatically increasing False Alarm Rate.
- **Avoid:** Including very short entities (1-2 characters) in the bias list. These match too many token prefixes and pollute the bias vector with noise.

## Error Handling

- **Entity not recognized despite biasing:** Check that the entity's tokenization matches what the model actually produces. Some tokenizers handle whitespace differently (leading space tokens). Try tokenizing with and without a leading space.
- **High False Alarm Rate:** Lower `lambda`, enable the confidence gate, or remove overly generic entities from the list (e.g., common words like "The" or "Go" that happen to be entity names).
- **Latency regression:** Profile the bias computation. For entity lists >50K, consider sharding the trie or limiting active cursors to the top-N by depth. The trie lookup itself is O(1) per step but cursor management can grow if many prefixes are active simultaneously.
- **Beam search interactions:** When using beam search, each beam must maintain its own cursor set. Clone cursors when beams split and discard them when beams are pruned.

## Limitations

- LOGIC requires access to the model's logits at each decoding step. It does not work with black-box APIs (e.g., OpenAI's Whisper API) that only return final transcripts. You need a local model or a serving framework that exposes logit hooks.
- The technique biases toward surface-form matches. It cannot resolve semantic ambiguity — if two entities share the same pronunciation but differ in meaning, LOGIC cannot distinguish them without additional context.
- Entity lists that change per-request (e.g., personalized contact lists) require per-request trie construction. While fast (~5ms for 500 entities), this adds up at high QPS.
- The approach is designed for autoregressive decoding. It does not directly apply to CTC-based or non-autoregressive ASR models without adaptation.
- For very long entity names (>20 tokens), the depth bonus can cause premature commitment — the model locks onto an entity prefix and cannot back out even if the audio diverges.

## Reference

**Paper:** Wang, P. (2026). *Beyond Prompting: Efficient and Robust Contextual Biasing for Speech LLMs via Logit-Space Integration (LOGIC)*. arXiv:2601.15397v2. [https://arxiv.org/abs/2601.15397v2](https://arxiv.org/abs/2601.15397v2)

Look for: the logit interpolation formula, trie-based prefix tracking algorithm, lambda tuning methodology across 11 multilingual locales, and the Entity WER vs. False Alarm Rate tradeoff curves. Note: paper is temporarily withdrawn for institutional approval; check for resubmission.