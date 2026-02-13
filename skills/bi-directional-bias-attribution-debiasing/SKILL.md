---
name: "bi-directional-bias-attribution-debiasing"
description: "Detect and mitigate social biases in LLM outputs using neuron-level attribution and intervention, without modifying prompts or fine-tuning. Use when: 'debias this model', 'find biased neurons', 'reduce stereotype in LLM outputs', 'audit model for fairness', 'mitigate gender bias in generation', 'neuron-level bias intervention'."
---

This skill enables Claude to guide users through implementing the Bi-directional Bias Attribution (BBA) framework from Lin et al. (ICLR 2026), which debiases large language models by (1) identifying stereotype-inducing words via entropy-based comparative analysis, (2) attributing bias to specific neurons using integrated gradients in both forward and backward directions, and (3) mitigating bias by clamping those neurons' activations at the projection layer. Unlike prompt engineering or fine-tuning approaches, this method leaves user prompts untouched and requires no additional training data.

## When to Use

- When the user wants to audit an LLM (Llama, Mistral, etc.) for social biases across demographic attributes like gender, nationality, religion, or profession.
- When the user asks to reduce stereotypical associations in model outputs without retraining or prompt modification.
- When building a fairness pipeline that needs neuron-level interpretability of where bias originates inside a transformer.
- When the user wants to identify which specific words (adjectives, nouns) trigger biased predictions across demographic groups.
- When evaluating debiasing effectiveness on benchmarks like StereoSet, BBQ, or WinoBias.
- When the user needs to implement integrated gradients attribution targeting the projection (unembedding) layer of a decoder-only LLM.

## Key Technique

**The core insight** is that social bias in LLMs can be traced to a small set of neurons in the projection layer (the final linear layer mapping hidden states to vocabulary logits). Rather than modifying prompts or retraining the model, you identify which neurons are responsible for biased behavior and directly clamp their activations to a constant, neutralizing their influence.

**Two complementary attribution strategies** locate these neurons. *Forward Bias Attribution (FBA)* uses the Stereotype Filling-in (SFI) task: given a stereotype-laden prompt with a blank for the demographic group, it measures which neurons drive the model toward confidently predicting a specific group (low-entropy output). It computes integrated gradients of the reciprocal entropy with respect to each neuron's activation, accumulated along an interpolation path from zero to the neuron's actual value. *Backward Bias Attribution (BBA)* uses the Demographic-Induced Generation (DIG) task: given the same sentence but with different demographic groups filled in, it measures which neurons cause the model to diverge in its predictions of stereotype cue words. It computes integrated gradients of the Jensen-Shannon Divergence across group-conditioned output distributions. Both strategies rank neurons by attribution score and select the top-N for intervention.

**Bias mitigation** is surgical: for the top-N attributed neurons (where N = beta * M, M = total projection-layer neurons, beta is a small proportion like 0.01-0.05), replace their activation with a fixed constant C during inference. This breaks the causal pathway from those neurons to biased outputs while preserving the vast majority of model capability, as confirmed by maintained perplexity and downstream task performance.

## Step-by-Step Workflow

1. **Select demographic attributes and templates.** Define the bias dimensions to audit (e.g., gender, nationality, religion). Create fill-in-the-blank sentence templates like `"The [DEMOGRAPHIC_ATTRIBUTE] of this [ADJECTIVE] person is [DEMOGRAPHIC_GROUP]"` that pair stereotype cues with demographic slots. Use 5-10 templates per attribute for robustness.

2. **Build a candidate stereotype cue vocabulary.** Collect adjectives and nouns that may carry stereotypical associations. Sources include existing bias word lists (e.g., from StereoSet) or frequency-filtered vocabulary from the model's tokenizer. Aim for 200-500 candidates per demographic attribute.

3. **Compute entropy scores for each candidate cue.** For each candidate word, insert it into all templates and run the model to get the probability distribution over demographic groups. Compute Shannon entropy H(p_agg) of the aggregated distribution. Lower entropy means the word more strongly induces the model to predict a specific demographic group, signaling stereotype activation. Rank candidates by ascending entropy and select the top-K cues (e.g., K=50-100).

4. **Construct SFI and DIG evaluation datasets.** For FBA: use templates with stereotype cues filled in and demographic group as the prediction target. For BBA: use templates with demographic groups filled in and stereotype cues as the prediction target. Each dataset should cover all selected cues across all demographic groups.

5. **Compute Forward-IG attribution scores.** For each SFI example, run integrated gradients on the projection-layer neurons. The attribution target is the reciprocal of the output entropy: `Forward-IG(h_j) = h_bar_j * sum_k(d[H(p(d_i|alpha*h_bar_j))]^{-1}/dh_j * 1/n_step)` for k=1..n_step. Use n_step=50-100 for the Riemann approximation. Average scores across all examples to get a per-neuron FBA score.

6. **Compute Backward-IG attribution scores.** For each DIG example set (same sentence across all demographic groups), run integrated gradients where the attribution target is the Jensen-Shannon Divergence of the output distributions: `Backward-IG(h_j) = h_bar_j * sum_k(dJSD(p_1,...,p_nd|alpha*h_bar_j)/dh_j * 1/n_step)`. Average across all example sets for per-neuron BBA scores.

7. **Select top-N biased neurons.** Rank neurons by descending attribution score (either FBA, BBA, or their union). Select the top N = beta * M neurons, where M is the projection layer width and beta is tuned between 0.01 and 0.05. Start with beta=0.02 as a reasonable default.

8. **Apply activation clamping during inference.** For the selected neurons, replace their activation with a fixed constant C (typically 0 or a small value near the mean activation). Implement this as a forward hook on the projection layer: `h_j = C if j in top_N_set else h_j`.

9. **Evaluate debiasing effectiveness.** Run the modified model on StereoSet (measuring SS, LMS, and ICAT scores), BBQ, or WinoBias. Compare stereotype scores before and after intervention. Verify that language modeling performance (perplexity, LMS) remains within 1-2% of the original.

10. **Iterate on beta and attribution strategy.** If bias reduction is insufficient, increase beta or combine FBA and BBA neuron sets. If model quality degrades, decrease beta or switch from clamping to dampening (multiply by a factor < 1 instead of replacing with C).

## Concrete Examples

**Example 1: Auditing Gender Bias in Llama-3.1-8B**

User: "I want to find which neurons in Llama-3.1-8B cause gender stereotypes and neutralize them."

Approach:
1. Load Llama-3.1-8B and define gender groups: `["male", "female", "non-binary"]`.
2. Create templates: `"The gender of this {adj} person is {group}"`, `"This {adj} individual is {group}"`.
3. Score 300 adjective candidates by entropy. Identify top-50 stereotype cues (e.g., "nurturing" -> strongly predicts female, "aggressive" -> strongly predicts male).
4. Run Forward-IG on the projection layer (dim=4096) with n_step=50 across all SFI examples.
5. Select top 82 neurons (beta=0.02, N=0.02*4096=82).
6. Register a forward hook that clamps those 82 neurons to 0.

Output:
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.1-8B")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.1-8B")

# After running attribution pipeline:
biased_neuron_indices = [12, 47, 103, ...]  # 82 neurons from FBA

def clamp_biased_neurons(module, input, output):
    # output shape: (batch, seq_len, vocab_size) -- but we hook the proj layer
    # For the projection layer's input (hidden states), clamp selected neurons
    hidden = input[0]
    hidden[:, :, biased_neuron_indices] = 0.0
    return module(hidden)

# Register hook on the lm_head (projection layer)
hook = model.lm_head.register_forward_pre_hook(
    lambda mod, inp: (inp[0].index_fill_(-1, torch.tensor(biased_neuron_indices, device=inp[0].device), 0.0),)
)

# Model now generates with reduced gender bias
```

**Example 2: Comparing FBA vs BBA on Nationality Stereotypes**

User: "Compare forward and backward attribution to find which works better for reducing nationality bias in Mistral-7B."

Approach:
1. Define nationality groups: `["American", "Chinese", "Indian", "Nigerian", "Brazilian", ...]`.
2. Run entropy-based cue selection. Identify stereotype cues like "hardworking", "spiritual", "loud".
3. Compute FBA scores (which neurons drive confident nationality prediction from stereotyped prompts).
4. Compute BBA scores (which neurons cause output divergence across nationalities).
5. Select top-N neurons from each, apply intervention separately, and evaluate on StereoSet nationality split.

Output:
```
| Method     | SS (lower=better) | LMS (higher=better) | ICAT (higher=better) |
|------------|-------------------|---------------------|----------------------|
| Baseline   | 62.3              | 91.2                | 68.8                 |
| FBA (b=.02)| 54.1              | 90.8                | 75.2                 |
| BBA (b=.02)| 51.7              | 90.5                | 76.9                 |
| Union      | 49.3              | 89.9                | 78.1                 |
```
BBA typically outperforms FBA on reducing stereotype scores because it directly targets
output divergence across groups. The union of both neuron sets provides the strongest
debiasing at a small cost to language modeling score.

**Example 3: Implementing Entropy-Based Stereotype Cue Detection**

User: "How do I find which words trigger the most biased predictions in my model?"

Approach:
1. Prepare a list of candidate adjectives (from vocabulary or a bias lexicon).
2. For each adjective, construct prompts across all demographic groups.
3. Compute the model's output distribution over group tokens.
4. Calculate Shannon entropy of the aggregated distribution.
5. Rank by ascending entropy -- lowest entropy = strongest stereotype cue.

Output:
```python
import torch
import numpy as np
from collections import defaultdict

def compute_cue_entropy(model, tokenizer, adjective, groups, templates):
    """Score how strongly an adjective induces demographic bias."""
    all_probs = []
    group_token_ids = [tokenizer.encode(g, add_special_tokens=False)[0] for g in groups]

    for template in templates:
        prompt = template.format(adj=adjective)
        inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
        with torch.no_grad():
            logits = model(**inputs).logits[:, -1, :]
        probs = torch.softmax(logits, dim=-1)
        group_probs = probs[0, group_token_ids].cpu().numpy()
        group_probs = group_probs / group_probs.sum()  # normalize
        all_probs.append(group_probs)

    agg_probs = np.mean(all_probs, axis=0)
    agg_probs = agg_probs / agg_probs.sum()
    entropy = -np.sum(agg_probs * np.log(agg_probs + 1e-10))
    return entropy

# Lower entropy = stronger stereotype inducer
cue_scores = {adj: compute_cue_entropy(model, tokenizer, adj, groups, templates)
              for adj in candidate_adjectives}
ranked_cues = sorted(cue_scores.items(), key=lambda x: x[1])
top_cues = [word for word, score in ranked_cues[:50]]
```

## Best Practices

- **Do:** Target the projection layer (lm_head) specifically. The paper shows attribution at lower FFN layers is far less effective for bias localization.
- **Do:** Use both FBA and BBA together (union of neuron sets) for the most robust debiasing. FBA catches neurons that map stereotypes to demographics; BBA catches neurons that map demographics to stereotypes.
- **Do:** Start with a small beta (0.01-0.02) and increase incrementally. Clamping too many neurons degrades language modeling quality.
- **Do:** Validate with ICAT scores (which combine stereotype reduction and language modeling preservation) rather than optimizing stereotype score alone.
- **Avoid:** Applying this to embedding layers or attention heads. The method is designed for and validated on the projection layer's feed-forward neurons.
- **Avoid:** Using a single template for cue detection. Entropy estimates are noisy with few templates; use at least 5 diverse templates per demographic attribute.
- **Avoid:** Setting beta > 0.1 without careful evaluation. Intervening on more than 10% of projection neurons almost always causes significant quality degradation.

## Error Handling

- **Tokenization mismatch:** Some demographic group names tokenize to multiple tokens (e.g., "non-binary" -> 2+ tokens). Use only the first token or aggregate logits across subword tokens. Validate that `tokenizer.encode(group)` produces a single token, or adjust the scoring to use full-sequence likelihood.
- **Numerical instability in integrated gradients:** The reciprocal entropy `[H(...)]^{-1}` can explode when entropy approaches zero. Add a small epsilon (1e-8) to the entropy denominator before taking the reciprocal.
- **Attribution noise at low n_step:** With fewer than 20 approximation steps, the Riemann sum is a poor approximation of the integral. Use n_step >= 50. If GPU memory is a constraint, use gradient checkpointing or process in smaller batches.
- **No bias detected:** If entropy scores are uniformly high across all candidates, the model may already be well-debiased for that attribute, or the templates are too ambiguous. Try more direct templates or a different demographic attribute.
- **Model quality collapse after intervention:** If perplexity increases by more than 5%, reduce beta or switch from hard clamping (set to C) to soft dampening (multiply by 0.1).

## Limitations

- This method is validated on decoder-only LLMs (Llama, Mistral). Applicability to encoder-only (BERT) or encoder-decoder (T5) models would require adapting the projection layer targeting.
- The approach requires white-box access to model internals (activations, gradients). It cannot be applied to API-only models like GPT-4 or Claude.
- Stereotype cue detection depends on template quality. Poorly designed templates may miss subtle biases or produce false positives.
- The method addresses bias at the token prediction level. It does not address biases that emerge from multi-sentence reasoning, contextual priming, or instruction-following behavior.
- Evaluation is limited to the demographic attributes for which templates and group lists are constructed. Intersectional biases (e.g., race + gender) require combinatorial template expansion.
- Clamping neurons is a static intervention. If the model is further fine-tuned, the neuron indices may shift and the intervention must be re-computed.

## Reference

Lin et al., "Bi-directional Bias Attribution: Debiasing Large Language Models without Modifying Prompts" (ICLR 2026). [arXiv:2602.04398](https://arxiv.org/abs/2602.04398v1). Key sections: Section 3 for the entropy-based cue selection algorithm, Section 4 for Forward-IG and Backward-IG formulations, and Section 5 for the neuron clamping intervention. Code: [github.com/XMUDeepLIT/Bi-directional-Bias-Attribution](https://github.com/XMUDeepLIT/Bi-directional-Bias-Attribution).