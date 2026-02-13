---
name: "towards-sample-efficient-stable-reinforcement"
description: |
  Build RL-enhanced LLM recommendation systems using the RISER framework (Reinforced Item Space Exploration for Recommendation). Applies reinforcement learning to LLM-based sequential recommendation with high sample efficiency and training stability. Converts failed rollouts into preference learning signal, prevents redundant exploration, and constrains token-level updates via prefix-tree masking.

  Trigger phrases:
  - "Build an RL-based recommendation system with an LLM"
  - "Improve sample efficiency for LLM recommendation training"
  - "Apply reinforcement learning to sequential recommendation"
  - "Convert failed RL rollouts into training signal"
  - "Stabilize GRPO training for recommendation"
  - "Implement preference optimization for item prediction"
---

# RISER: Reinforced Item Space Exploration for Recommendation

This skill enables Claude to design and implement RL-enhanced LLM recommendation pipelines following the RISER framework. RISER bypasses Chain-of-Thought reasoning (which adds latency without benefiting recommendation tasks) and instead applies reinforcement learning directly to explore the item space. Its core innovations are: (1) transforming zero-signal failed rollouts into pairwise preference data via SimPO, (2) deduplicating rollouts through oversampling to encourage broader exploration, and (3) applying a certainty-aware loss mask built from a prefix tree over the item vocabulary to prevent unstable gradient updates on deterministic tokens.

## When to Use

- When the user wants to train an LLM (e.g., Qwen2, LLaMA) to predict next items in a sequential recommendation setting
- When building an RL fine-tuning loop on top of an SFT-trained recommendation model and most rollouts produce no reward signal (the "sparse reward" problem)
- When the user needs to stabilize GRPO or PPO training for generative recommendation where the action space is the full item catalog (10,000+ items)
- When converting a recommendation task from supervised next-item prediction to an RL exploration problem
- When the user asks how to reuse failed/incorrect model outputs as training signal instead of discarding them
- When implementing token-level regularization for generative models that output structured item identifiers (titles, IDs)

## Key Technique

**The core problem:** In sequential recommendation, an LLM generates item predictions given a user's interaction history. Standard RL (e.g., GRPO) samples multiple rollouts per prompt and computes advantages from reward differences. But when the item catalog is large, most or all rollouts miss the ground-truth item, yielding zero advantage across the entire group. These "non-learnable" trajectories are wasted compute under standard GRPO.

**RISER's solution has three pillars:**

1. **Failed-Rollout Recycling via SimPO.** When all G rollouts for a prompt fail (none hit the ground-truth), RISER constructs G pairwise preference pairs: each pairs the ground-truth item (as the preferred response) against one of the incorrect rollout outputs (as the dispreferred response). These pairs are optimized with Simple Preference Optimization (SimPO), which uses the average log-likelihood of the sequence as an implicit reward. This converts 100% of training data into usable signal.

2. **Redundant Rollout Prevention.** LLMs in recommendation mode tend to repeatedly generate the same popular items. RISER oversamples m > n completions, deduplicates them, and fills the final batch of n with unique items first, drawing from duplicates only if needed. This forces the policy to explore more of the item space per training step.

3. **Certainty-Aware Token Masking (KL-Cov + Prefix Tree).** Not all tokens in an item name carry equal decision-making weight. Given the prefix "Harry Pot", the next token "ter" is deterministic -- updating the model on it is noise. RISER builds a prefix tree from the item vocabulary and identifies branching points (where multiple valid continuations exist). Tokens at branching points receive full gradient weight; deterministic continuation tokens receive a decay factor d in [0, 1). This same mask also targets the KL divergence penalty in GRPO to outlier tokens specifically (high confidence + high advantage), preventing catastrophic policy shifts.

## Step-by-Step Workflow

1. **Format the dataset as sequential recommendation prompts.** For each user, construct a chronological interaction history `h = [item_1, item_2, ..., item_k]` and ground-truth next item `y = item_{k+1}`. Convert to a text prompt: `"User has interacted with: [item_1], [item_2], ... [item_k]. Predict the next item the user will interact with."` Split data 8:1:1 into train/val/test chronologically.

2. **Run supervised fine-tuning (SFT) on the training set.** Fine-tune the base LLM on `(prompt, ground_truth_item)` pairs using standard causal language modeling loss. This gives the model a baseline ability to generate valid item names from the catalog.

3. **Build the item vocabulary prefix tree.** Tokenize every item name/title in the catalog. Construct a trie (prefix tree) where each node represents a token and edges represent valid continuations. For each position in a generated sequence, this tree tells you whether the current prefix has one valid continuation (deterministic) or multiple (branching point).

4. **Sample an RL training set from the validation split.** Draw ~10,000 prompts for RL training and ~3,000 disjoint prompts for RL validation. Keep these separate from the SFT training data to avoid overfitting.

5. **Generate rollouts with oversampling and deduplication.** For each prompt, sample m completions from the current policy (m > G, where G is the target group size). Extract unique completions into a set, then fill G slots prioritizing unique items. This is the deduplicated rollout batch.

6. **Partition rollouts into successful and failed groups.** A rollout is "successful" if the generated item matches the ground-truth item `y`. Split each prompt's G rollouts into `O_pass` (contains ground-truth) and `O_fail` (does not).

7. **Compute modified GRPO loss on successful groups.** For prompts where at least one rollout hit ground-truth, compute group-normalized advantages. Apply the certainty-aware mask: for each token, check the prefix tree -- if the token is at a branching point, weight = 1.0; if deterministic, weight = d. Apply KL-Cov penalty only on outlier tokens (those with both high model confidence and high advantage magnitude).

8. **Construct preference pairs and compute SimPO loss on failed groups.** For prompts where zero rollouts hit ground-truth, create G pairs of `(y_preferred=ground_truth, y_dispreferred=rollout_i)`. Compute the SimPO loss using masked average log-likelihood as the implicit reward, applying the same prefix-tree mask to down-weight deterministic tokens.

9. **Combine losses and update the policy.** Sum the GRPO loss (from successful groups) and SimPO loss (from failed groups) with appropriate weighting. Update model parameters. Repeat from step 5 for multiple iterations.

10. **Evaluate on the held-out test set using all-ranking protocol.** Generate predictions for each test prompt and compute NDCG@N and HR@N (N in {5, 10, 20}) by ranking the predicted item against the full catalog.

## Concrete Examples

**Example 1: E-Commerce Next-Purchase Prediction**

User: "I want to train a Qwen2-1.5B model to predict the next video game a user will buy on Amazon, using RL to improve over my SFT baseline."

Approach:
1. Load the Amazon Games dataset, apply 5-core filtering (users and items with >= 5 interactions), truncate sequences to the last 10 items.
2. Format each sample as:
   ```
   Input: "User purchased: Zelda BOTW, Mario Odyssey, ..., Hades. What will they buy next?"
   Target: "Celeste"
   ```
3. SFT fine-tune Qwen2-1.5B on the training split for 3 epochs.
4. Build prefix tree from all game titles tokenized with the Qwen2 tokenizer.
5. Run RISER RL loop: G=6 rollouts per prompt, oversample m=10, decay factor d=0.1, SimPO beta=2.0, margin gamma=0.5.
6. After 3 RL iterations, evaluate on test set.

Output:
```
Metric        | SFT Baseline | + RISER RL
HR@5          | 0.042        | 0.061
HR@10         | 0.068        | 0.089
NDCG@5        | 0.029        | 0.044
NDCG@10       | 0.041        | 0.058
```

**Example 2: Implementing the Prefix-Tree Token Mask**

User: "How do I build the certainty-aware loss mask for my item catalog?"

Approach:
1. Tokenize all item names in the catalog.
2. Build a trie where each path from root to leaf is a tokenized item name.
3. At inference/training time, for each generated token at position t, look up the prefix `tokens[0:t]` in the trie and count children of that node.
4. If children > 1 (branching), mask value = 1.0. If children == 1 (deterministic), mask value = d.

```python
from collections import defaultdict

class PrefixTree:
    def __init__(self):
        self.children = defaultdict(PrefixTree)
        self.is_terminal = False

    def insert(self, token_ids: list[int]):
        node = self
        for tid in token_ids:
            node = node.children[tid]
        node.is_terminal = True

    def get_branching_mask(self, token_ids: list[int], decay: float = 0.1) -> list[float]:
        """Returns per-token mask: 1.0 at branching points, decay at deterministic points."""
        mask = []
        node = self
        for tid in token_ids:
            if tid not in node.children:
                mask.append(1.0)  # Unknown prefix, treat as branching
                break
            num_children = len(node.children)
            mask.append(1.0 if num_children > 1 else decay)
            node = node.children[tid]
        return mask

# Build tree from catalog
tree = PrefixTree()
for item_name in catalog:
    token_ids = tokenizer.encode(item_name, add_special_tokens=False)
    tree.insert(token_ids)

# During training, for a generated sequence:
mask = tree.get_branching_mask(generated_token_ids, decay=0.1)
# Apply mask to per-token loss: loss_t *= mask[t]
```

**Example 3: Converting Failed Rollouts to SimPO Preference Pairs**

User: "All 6 of my rollouts missed the ground truth. How do I still learn from them?"

Approach:
1. You have ground-truth item `y = "The Great Gatsby"` and 6 failed rollouts: `["1984", "Dune", "Dune", "Fahrenheit 451", "1984", "Brave New World"]`.
2. After deduplication in future batches, construct 6 preference pairs:
   ```
   Pair 1: preferred="The Great Gatsby", dispreferred="1984"
   Pair 2: preferred="The Great Gatsby", dispreferred="Dune"
   Pair 3: preferred="The Great Gatsby", dispreferred="Dune"
   Pair 4: preferred="The Great Gatsby", dispreferred="Fahrenheit 451"
   Pair 5: preferred="The Great Gatsby", dispreferred="1984"
   Pair 6: preferred="The Great Gatsby", dispreferred="Brave New World"
   ```
3. Compute SimPO loss for each pair:
   ```python
   def simpo_loss(model, prompt, y_preferred, y_dispreferred, beta, gamma, mask_fn):
       logp_pref = masked_avg_logprob(model, prompt, y_preferred, mask_fn)
       logp_disp = masked_avg_logprob(model, prompt, y_dispreferred, mask_fn)
       return -torch.log(torch.sigmoid(beta * (logp_pref - logp_disp) - gamma))
   ```
4. Average the 6 pair losses and add to the total RL objective.

## Best Practices

- **Do:** Always apply 5-core filtering to your dataset before training. Users and items with fewer than 5 interactions create noise that destabilizes RL training.
- **Do:** Oversample rollouts by at least 1.5-2x the target group size G to ensure sufficient diversity after deduplication. If G=6, sample m=10-12.
- **Do:** Set the decay factor d low (0.05-0.2) for catalogs with long item names where most tokens are deterministic. Higher d (0.3-0.5) is acceptable for short item identifiers.
- **Do:** Use the SFT checkpoint as the reference model for KL divergence in GRPO. Do not update the reference model during RL training.
- **Avoid:** Using Chain-of-Thought prompting for sequential recommendation. It adds latency without improving accuracy because user behavioral data lacks explicit reasoning traces.
- **Avoid:** Running RL training on the same data used for SFT. Sample a separate RL training set from the validation split to prevent memorization.
- **Avoid:** Large KL penalty coefficients (beta_KL > 0.1). The certainty-aware masking already constrains updates; excessive global KL penalty stifles exploration.

## Error Handling

- **All rollouts are identical after oversampling:** Increase sampling temperature or increase the oversample ratio m/n. If the model has collapsed to a single output, revert to a recent SFT or earlier RL checkpoint.
- **SimPO loss diverges:** The margin gamma may be too small, causing the sigmoid to saturate. Increase gamma (e.g., from 0.5 to 1.0) or reduce beta_SimPO.
- **Ground-truth item not in the prefix tree:** This indicates a tokenization mismatch between the catalog and the training data. Re-tokenize the catalog with the exact same tokenizer and settings used during generation.
- **GRPO advantage is NaN:** Occurs when all rollouts in a successful group have identical rewards. Add a small epsilon (1e-8) to the standard deviation in advantage normalization.
- **Training reward plateaus early:** The RL training set may be too small or too easy. Sample more prompts (up to 20,000) and ensure they include difficult cases where the SFT model already fails.

## Limitations

- **Requires a pre-trained SFT baseline.** RISER is a fine-tuning framework, not a from-scratch training method. The SFT model must already be capable of generating valid item names.
- **Scales with catalog size.** The prefix tree and oversampling overhead grow with the number of items. For catalogs with 100K+ items, prefix tree construction and lookup may need optimization (e.g., batch lookups, GPU-side trie).
- **Binary reward only.** RISER uses a hit/miss reward. It does not natively support graded relevance, diversity objectives, or multi-objective optimization. Extending to richer reward signals requires modifying the advantage computation.
- **Item-name-dependent.** The prefix tree masking assumes items are represented as tokenized text (names/titles). If items use opaque numeric IDs, the masking mechanism provides no benefit since every token is a branching point.
- **Sequential recommendation only.** The framework is designed for next-item prediction from a chronological history. It does not directly apply to session-based, context-aware, or multi-modal recommendation.

## Reference

**Paper:** [Towards Sample-Efficient and Stable Reinforcement Learning for LLM-based Recommendation](https://arxiv.org/abs/2602.00632v1) (Ding et al., 2026)

Key sections to study: Section 4 (RISER framework -- the three-pillar design), Section 4.3 (certainty-aware loss mask and prefix tree construction), Appendix E (full algorithm pseudocode), and Table 2 (ablation results showing each component's contribution).