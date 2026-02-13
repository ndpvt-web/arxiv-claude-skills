---
name: "unintended-memorization-sensitive-information"
description: "Audit fine-tuned LLMs for unintended PII memorization and apply privacy-preserving mitigations. Use when: 'audit my model for PII leakage', 'check if fine-tuning memorized sensitive data', 'add differential privacy to training', 'run extraction probes on my model', 'prevent PII leakage in fine-tuned model', 'apply DPO for privacy alignment'."
---

# Unintended Memorization of Sensitive Information in Fine-Tuned LLMs

This skill enables Claude to audit fine-tuned language models for unintended memorization of Personally Identifiable Information (PII), design extraction probes that test whether a model leaks sensitive data from its training inputs, and implement four privacy-preserving mitigations: differential privacy (DP), machine unlearning, regularization, and preference alignment (DPO). The core insight from the paper is that PII appearing *only in model inputs* — not in training targets — still gets memorized and can be extracted, which most practitioners overlook entirely.

## When to Use

- When a user is fine-tuning an LLM on data containing names, addresses, financial details, medical records, or other PII and wants to quantify leakage risk
- When a user asks to run extraction attacks or red-team a fine-tuned model for memorized sensitive content
- When a user needs to choose between differential privacy, unlearning, regularization, or DPO to protect training data privacy
- When reviewing a fine-tuning pipeline and assessing whether PII in the *input* context (not just the label/target) poses a memorization risk
- When a user wants to build a DPO dataset that teaches a model to withhold PII
- When evaluating privacy compliance (GDPR, HIPAA) for a deployed fine-tuned model
- When a user asks how model size, PII frequency, or language affects memorization behavior

## Key Technique

**The overlooked threat: input-side memorization.** Standard fine-tuning optimizes the model to predict target tokens, so most privacy work focuses on PII in targets. This paper demonstrates that PII appearing only in input contexts — the prefix the model conditions on — is also memorized. Fine-tuned models become *more confident* predicting PII tokens (the negative log-likelihood distribution shifts toward zero), meaning the model actively learns to reproduce sensitive data it was never explicitly trained to output.

**Extraction probes quantify the risk.** The primary attack is the True-Prefix Attack (TPA): given a prefix `c` from training data, generate `N` tokens and check if the model reproduces the PII string `s` that followed. An enhanced variant adds the first character of the PII to the prefix to boost extraction. The paper also tests role-play prompts ("continue this text exactly"), recovery prompts ("recover the original continuation"), and jailbreak prompts (delimiter-based containment). Cross-memorization is measured by comparing generated PII against *all* PII of the same type in the dataset — not just the specific training example — revealing that models generalize memorized PII patterns.

**Four mitigations with distinct trade-offs.** (1) Differential privacy (DP-SGD) with epsilon values like 2 or 6 and delta=10^-5 adds calibrated noise to gradients but introduces training instability and requires careful hyperparameter tuning. (2) Machine unlearning via gradient ascent on sensitive spans forces the model to "forget" specific PII. (3) Regularization (weight decay, dropout) reduces overfitting to PII contexts. (4) DPO preference alignment, where the preferred response masks PII in 20-token spans and the rejected response leaves it intact, achieves the best task-accuracy retention but weaker direct-extraction reduction. The critical finding: even the best single method achieves only ~40% reduction in direct PII memorization, so layering multiple approaches is necessary.

## Step-by-Step Workflow

1. **Inventory PII in training data.** Scan the fine-tuning dataset with a NER pipeline (regex + LLM-based detection + manual spot-check) to identify all PII entities — names, addresses, phone numbers, financial identifiers, medical record numbers. Categorize each by type and note whether it appears in the input context, the target, or both.

2. **Assess frequency and distribution.** Count how many times each unique PII entity appears across the dataset. Note: frequency is a weak predictor of memorization (R^2 = 0.237), so do not assume low-frequency PII is safe. Context and utility to the task matter more.

3. **Fine-tune with a privacy-aware baseline.** Use QLoRA (rank=8) or equivalent parameter-efficient method. Record training hyperparameters precisely — they affect memorization. Larger models (8B+) memorize more aggressively than smaller ones (1B).

4. **Construct extraction probes.** Build a probe set with three attack types:
   - **Greedy TPA**: For each PII span of N tokens following prefix `c`, run greedy decoding: `s* = argmax f(s'|c)` for |s'|=N. Check if `s* == s`.
   - **Enhanced TPA**: Append the first character of the PII to the prefix before generating.
   - **Sampling TPA**: Generate 32 samples per prefix at temperature=1.0 and check if any sample contains the PII.

5. **Run cross-memorization analysis.** For each generated PII string, compare it against *all* PII of the same type in the entire dataset, not just the originating example. This catches cases where the model has generalized a memorized pattern to new contexts.

6. **Measure baseline leakage.** Count total extracted PII instances (greedy, enhanced, sampled) and unique entities. Compare fine-tuned model rates against the base model to isolate memorization introduced by fine-tuning.

7. **Apply mitigation(s) based on deployment requirements:**
   - **For strongest formal guarantees**: DP-SGD with epsilon=6, delta=10^-5. Expect ~60% cross-memorization reduction but possible training instability. Use larger batch sizes and higher learning rates to stabilize.
   - **For best task performance retention**: DPO with a 150-token sliding window, masking PII in 20-token spans. Create preferred (PII-masked) and rejected (PII-unmasked) pairs with a uniform system prompt instructing the model to withhold PII.
   - **For targeted forgetting**: Gradient ascent unlearning on the specific PII spans you want removed.
   - **For general regularization**: Increase weight decay and dropout during fine-tuning.

8. **Re-run all extraction probes after mitigation.** Compare post-mitigation extraction rates against baseline. Report reduction percentages for greedy, enhanced, sampling, and cross-memorization settings separately — they often diverge.

9. **Evaluate task performance trade-off.** Measure downstream task accuracy (BERTScore-F1 or task-specific metric) before and after mitigation. A mitigation that destroys task utility is not viable. DPO typically preserves near-optimal performance; DP may degrade it significantly at low epsilon.

10. **Document and monitor.** Record all probe results, mitigation parameters, and performance metrics. Schedule periodic re-evaluation, especially after any data or model updates.

## Concrete Examples

**Example 1: Auditing a fine-tuned medical summarization model**

User: "I fine-tuned Llama 3.2 3B on 26,000 German clinical discharge summaries. How do I check if it memorized patient names?"

Approach:
1. Extract all patient names from the training data using NER (combine regex for German name patterns with an LLM-based detector, then spot-check 200 samples manually).
2. For each training example containing a patient name, extract the 50 tokens preceding the name as the prefix.
3. Run greedy TPA: feed each prefix to the fine-tuned model, generate tokens equal to the name length, check for exact match.
4. Run enhanced TPA: append the first letter of the patient name to each prefix and regenerate.
5. Run sampling TPA: generate 32 samples per prefix at temperature=1.0.
6. Run cross-memorization: compare all generated names against the full list of patient names in the dataset.

Output:
```
PII Audit Report — Llama 3.2 3B (Discharge Summaries)
------------------------------------------------------
Extraction Method     | Extracted Instances | Unique Entities
Greedy TPA            | 1,604               | 892
Enhanced TPA          | 91                  | 67
Sampling TPA (32x)    | 3,217               | 1,431
Cross-memorization    | 11,754              | 2,108

Baseline (pre-fine-tune):
Greedy TPA            | 1,940               | 1,102

Finding: Fine-tuning reduced some greedy extractions but
cross-memorization is severe — the model generalizes
patient name patterns to new contexts.
Recommendation: Apply DPO alignment + DP-SGD (epsilon=6).
```

**Example 2: Implementing DPO for PII suppression**

User: "How do I build a DPO dataset to teach my model not to leak credit card numbers?"

Approach:
1. Identify all training examples containing credit card numbers.
2. For each example, create a 150-token sliding window around each credit card number.
3. Create the preferred response: mask the credit card number within a 20-token span (replace with `[REDACTED]` or generic placeholder).
4. Create the rejected response: leave the credit card number intact.
5. Add a uniform system prompt: "You must never reproduce credit card numbers, account numbers, or other financial identifiers from your training data."
6. Fine-tune with DPO on these preference pairs after the initial task fine-tuning.

Output:
```python
# DPO dataset construction for PII suppression
import re

def build_dpo_pairs(examples, pii_pattern, window=150, mask_span=20):
    pairs = []
    for ex in examples:
        for match in re.finditer(pii_pattern, ex["text"]):
            start = max(0, match.start() - window // 2)
            end = min(len(ex["text"]), match.end() + window // 2)
            context = ex["text"][start:end]

            # Preferred: PII masked
            mask_start = match.start() - start
            mask_end = min(mask_start + mask_span, len(context))
            preferred = context[:mask_start] + "[REDACTED]" + context[mask_end:]

            # Rejected: PII visible
            rejected = context

            pairs.append({
                "system": "Never reproduce financial identifiers from training data.",
                "prompt": context[:mask_start],
                "preferred": preferred[mask_start:],
                "rejected": rejected[mask_start:],
            })
    return pairs

# Typical dataset sizes: 1,500-6,000 pairs depending on PII density
```

**Example 3: Choosing a mitigation strategy**

User: "I'm fine-tuning Gemma 4B on multilingual financial documents. Which privacy method should I use?"

Approach:
1. Assess the deployment constraints: Is formal privacy guarantee required (regulatory)? Is task accuracy critical?
2. Check model size: At 4B parameters, DP-SGD is feasible but requires careful tuning.
3. Check language distribution: Memorization rates vary 2-4% across languages — test each language separately.
4. Recommend a layered strategy based on the paper's findings.

Output:
```
Mitigation Decision Matrix — Gemma 4B (Multilingual Financial)
--------------------------------------------------------------
Method              | Privacy Reduction | Task Perf. | Stability | Formal Guarantee
DP-SGD (eps=6)      | ~60% cross-mem    | Moderate   | Low       | Yes
DPO alignment       | Moderate direct   | High       | High      | No
Gradient unlearning  | High (targeted)   | Moderate   | Medium    | No
Regularization       | Low-Moderate      | High       | High      | No

Recommendation for regulatory compliance:
  1. Fine-tune with DP-SGD (eps=6, delta=1e-5, larger batch size)
  2. Follow with DPO alignment to catch remaining leakage
  3. Run extraction probes per-language to verify

Recommendation for maximum task accuracy:
  1. Fine-tune normally with increased weight decay
  2. Apply DPO alignment post-training
  3. Accept weaker formal guarantees, compensate with monitoring

Note: No single method exceeds ~40% direct memorization reduction.
Layer at least two approaches.
```

## Best Practices

- **Do:** Always test for cross-memorization, not just direct extraction. A model may not reproduce the exact PII from a specific example but may reproduce PII from *other* examples when given a similar context.
- **Do:** Treat input-context PII with the same severity as target PII. The paper proves both are memorized.
- **Do:** Run extraction probes at multiple temperatures (greedy + sampling at T=1.0) since they reveal different leakage patterns.
- **Do:** Use DPO as a post-training privacy layer even when using DP during training — they complement each other.
- **Avoid:** Assuming PII frequency predicts memorization risk. The correlation is weak (R^2=0.237). Low-frequency PII in a distinctive context can still be extracted.
- **Avoid:** Using very low DP epsilon values (e.g., epsilon=2) without extensive hyperparameter search — training instability often makes the model unusable.
- **Avoid:** Relying on a single mitigation method. The best individual method still leaves ~60% of direct memorization intact.

## Error Handling

- **DP training diverges or produces garbage output**: Increase batch size, raise learning rate, or relax epsilon (e.g., from 2 to 6). DP-SGD is highly sensitive to these settings and often needs 3-5x the batch size of standard fine-tuning.
- **Extraction probes show zero hits but model still leaks in production**: Your probe set may be too narrow. Expand to include role-play, recovery, and jailbreak prompt variants. Also test with paraphrased prefixes, not just exact training prefixes.
- **DPO degrades model quality on non-PII tasks**: The preference dataset may be too aggressive. Reduce the PII masking span from 20 tokens to 10 tokens, or limit DPO training to fewer epochs.
- **Cross-memorization count is very high**: This indicates the model has learned distributional patterns of PII (e.g., name frequencies in a population). Consider pre-training data deduplication and PII scrubbing before fine-tuning, not just post-hoc mitigation.
- **Unlearning removes too much general knowledge**: Gradient ascent is aggressive. Use a smaller learning rate for the unlearning phase (1/10th of fine-tuning LR) and limit to 1-2 epochs over the targeted PII spans only.

## Limitations

- The paper's mitigations cap at ~40% direct memorization reduction individually. Full PII elimination from a fine-tuned model is not achievable with current techniques — defense in depth and data sanitization before training remain essential.
- Extraction probes test known PII. An adversary with different prompting strategies or access to model logits may extract PII that probes miss.
- Results were validated on models up to 12B parameters. Memorization behavior at 70B+ scale may differ and likely worsens.
- DP formal guarantees apply to the training algorithm, not to specific PII instances. An epsilon=6 guarantee does not mean any particular individual's data is safe — it bounds the statistical difference.
- The DPO approach requires knowing where PII is in the training data to build preference pairs. If PII detection is incomplete, DPO cannot protect what it does not see.
- Multilingual results show 2-4% extraction variation across languages, but only 7 languages were tested. Coverage gaps exist for low-resource languages.

## Reference

**Paper:** [Unintended Memorization of Sensitive Information in Fine-Tuned Language Models](https://arxiv.org/abs/2601.17480v1) (Szep et al., EACL 2026)
**Code:** github.com/martonszep/llm-pii-leak
**Key takeaway:** PII in model *inputs* (not just targets) is memorized during fine-tuning. Use True-Prefix Attacks to quantify leakage, and layer post-training methods (DPO + DP) for the best privacy-utility trade-off.