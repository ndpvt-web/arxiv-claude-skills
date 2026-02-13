---
name: "data-free-privacy-preserving-inversion-selective"
description: |
  Remove memorized PII from language models without needing original training data using Data-Free Selective Unlearning (DFSU).
  Combines model inversion to synthesize pseudo-PII, token-level privacy masking, and contrastive LoRA-based unlearning.
  Use when: "remove PII from a fine-tuned model", "unlearn sensitive data without training set",
  "privacy-preserving model cleanup", "erase memorized personal information from LLM",
  "data-free machine unlearning", "selectively forget PII in language model"
---

# Data-Free Privacy-Preserving LLM Unlearning via Model Inversion and Selective Unlearning

This skill enables Claude to guide users through removing memorized personally identifiable information (PII) from large language models **without access to the original training data**. The technique, called Data-Free Selective Unlearning (DFSU), works in three stages: (1) invert the model's knowledge to synthesize pseudo-PII samples, (2) construct token-level masks that isolate sensitive entities, and (3) apply a contrastive mask loss within a LoRA subspace to surgically erase PII while preserving general model capability. This addresses a critical real-world gap where training data is proprietary or deleted but the model still leaks sensitive information.

## When to Use

- When a user needs to remove memorized PII (names, emails, SSNs, phone numbers, medical records) from a fine-tuned LLM but no longer has access to the training data
- When building a privacy compliance pipeline (GDPR right-to-erasure, CCPA) for deployed language models
- When auditing a model for PII leakage and then applying targeted remediation
- When the user asks how to do machine unlearning without a forget set
- When designing a post-training privacy scrubbing system that must preserve model utility (perplexity, accuracy)
- When implementing selective forgetting that targets specific entity types rather than blanket degradation

## Key Technique

**The core problem**: Standard machine unlearning requires the exact data you want the model to forget. In practice, that data is often gone -- deleted for compliance, held by a third party, or simply never retained. Gradient ascent on the full model destroys utility. DFSU solves both problems.

**Model inversion for pseudo-PII synthesis**: Rather than needing real PII, DFSU trains an inverter model (e.g., Flan-T5-Large) that takes the target LLM's output logit distributions and reconstructs the likely input text. The procedure: create syntactic templates containing PII slots, fill those slots with random substitute entities from a public disjoint pool, feed these candidates through the target model to collect logits, then decode via the inverter. The result is synthetic samples that mirror the model's memorized PII patterns without ever touching real training data. A few-shot LLM prompt then annotates these reconstructions to produce token-level entity labels.

**Contrastive mask loss in LoRA subspace**: The token-level mask M splits each sample into privacy-sensitive tokens (M=1) and benign tokens (M=0). Two loss streams are computed: a privacy loss L_priv (cross-entropy over masked tokens) and a utility loss L_gen (cross-entropy over unmasked tokens). The training objective is J(phi) = alpha * L_gen - beta * L_priv -- the model is trained to *increase* loss on PII tokens (forgetting them) while *decreasing* loss on non-PII tokens (retaining fluency). All updates are confined to LoRA adapters (rank 4, alpha 32, MLP-only by default), leaving the base model frozen. This constrains the unlearning to a low-rank subspace, preventing catastrophic forgetting.

## Step-by-Step Workflow

1. **Audit the target model for PII memorization**: Probe the model with known PII-context prefixes (e.g., "My name is", "Email:", "SSN:") and measure extraction rates. Compute Exact Reconstruction Rate (ERR) and Entity-Level Hit Rate (E-Hit) to quantify the problem's severity.

2. **Prepare an entity substitution pool**: Collect a public, disjoint set of fake PII entities -- synthetic names, fabricated emails, random phone numbers, dummy SSNs. These must not overlap with any real PII. Libraries like `Faker` in Python work well here.

3. **Build syntactic PII templates**: Create sentence templates with entity slots that match the types of PII you want to erase (e.g., `"Patient [NAME] can be reached at [PHONE] or [EMAIL]"`). Fill slots with entities from the substitution pool to create candidate inputs.

4. **Train the model inverter**: Use Flan-T5-Large (or similar seq2seq model) as the inverter. Feed each candidate through the target LLM to collect output logits. Compute soft embeddings as weighted sums of the LLM's word embeddings using the logit distribution, apply a learnable projection, and train the inverter with cross-entropy loss. Train for 30 epochs, batch size 256, learning rate 5e-4.

5. **Synthesize pseudo-PII via inversion**: Run the entity-swapped candidates through the target model, extract logits, and decode them with the trained inverter. The reconstructed text approximates what the model has memorized.

6. **Construct token-level privacy masks**: Use few-shot prompting with an LLM (or a dedicated NER model) to annotate each pseudo-PII sample, marking start and end positions of privacy-sensitive entities. Build binary mask M where M[t]=1 for PII tokens, M[t]=0 otherwise.

7. **Configure LoRA adapters on the target model**: Attach LoRA adapters with rank=4, alpha=32, targeting MLP layers. Freeze all base model parameters. Only the LoRA parameters phi will be updated during unlearning.

8. **Run contrastive selective unlearning**: Optimize J(phi) = alpha * L_gen - beta * L_priv using AdamW with cosine schedule. Use learning rate 5e-5 to 1e-4, effective batch size 16, for 10 epochs. Set alpha and beta to 1.0 as defaults; increase beta for more aggressive PII erasure.

9. **Evaluate privacy and utility**: Re-run the PII extraction probes from step 1. Measure ERR (target: 0%), FRS (target: <5%), E-Hit (target: <1%). Measure utility via perplexity on a held-out corpus (target: within 5% of original) or downstream task accuracy.

10. **Merge and deploy**: If metrics pass, merge the LoRA adapter into the base model weights for inference efficiency. Re-audit periodically as new PII categories are identified.

## Concrete Examples

**Example 1: Removing memorized patient names from a medical LLM**

User: "We fine-tuned Pythia-410M on clinical notes. The training data was deleted per policy, but the model still autocompletes patient names when given context prefixes. How do I remove this PII?"

Approach:
1. Probe the model with prefixes like `"Patient ID 4821, name:"` and record completions
2. Generate 500 fake patient records using `Faker` with realistic medical templates
3. Train a Flan-T5-Large inverter on the Pythia-410M logits (30 epochs, lr=5e-4)
4. Synthesize pseudo-PII by inverting the model's outputs on the fake records
5. Annotate with few-shot NER to mask PERSON, PHONE, MRN entity tokens
6. Attach LoRA (r=4, alpha=32) to Pythia-410M MLP layers
7. Train with contrastive mask loss for 10 epochs (lr=5e-5, beta=1.0)
8. Verify ERR drops to 0% and perplexity stays under 9.0

Output:
```
Pre-unlearning:  ERR=20.4%, FRS=46.5%, PPL=8.51
Post-unlearning: ERR=0.0%,  FRS=3.9%,  PPL=8.83
Status: PII effectively removed. Utility degradation: +3.8% perplexity (acceptable).
```

**Example 2: GDPR erasure request on a customer-service chatbot**

User: "A user exercised their GDPR right-to-erasure. We deleted their data from our databases, but our fine-tuned 1.4B model might still recall their email and address. We don't have the original training corpus anymore."

Approach:
1. Construct targeted probes using known context patterns (e.g., order confirmations, support tickets) with placeholder entities
2. Build substitution pool of 200 synthetic customer profiles (fake names, addresses, emails)
3. Train inverter on target model logits using the synthetic profiles as candidates
4. Invert and annotate: reconstruct memorized patterns, mask EMAIL, ADDRESS, PERSON tokens
5. Apply LoRA unlearning with beta=1.5 (higher weight for aggressive PII removal given compliance requirement)
6. Validate E-Hit < 1% and downstream accuracy within 3% of baseline

Output:
```
Pre-unlearning:  E-Hit=37.8%, Task Accuracy=79.9%
Post-unlearning: E-Hit=0.4%,  Task Accuracy=77.1%
GDPR compliance: Target PII no longer extractable via beam search (K=40).
```

**Example 3: Setting up a reusable privacy scrubbing pipeline**

User: "I want to build an automated pipeline that periodically scrubs PII from our production model. Give me the architecture."

Approach:
1. **Ingestion stage**: Maintain a registry of PII entity types to target (configurable per run)
2. **Template generator**: Script that produces entity-swapped candidates from configurable templates using `Faker`
3. **Inverter service**: Pre-trained Flan-T5 inverter, retrained monthly as the production model drifts
4. **Mask annotator**: Few-shot LLM pipeline (or fine-tuned NER) that outputs token-level binary masks
5. **Unlearning worker**: Applies contrastive mask loss with LoRA, parameterized by alpha/beta from config
6. **Evaluation harness**: Automated ERR/FRS/E-Hit/PPL checks with pass/fail gates
7. **Merge and deploy**: On pass, merge LoRA weights and push to model registry

Output:
```
Pipeline: templates -> inverter -> annotator -> unlearner -> evaluator -> deploy
Schedule: Weekly or on-demand (GDPR request triggers immediate run)
Compute: ~2 GPU-hours per scrub cycle on A100 for 1.4B model
Config: alpha=1.0, beta=1.0, LoRA r=4, epochs=10, lr=5e-5
```

## Best Practices

- **Do**: Start with a thorough PII extraction audit before unlearning. You need baseline ERR/FRS/E-Hit numbers to measure success against.
- **Do**: Use a disjoint entity pool for template substitution -- any overlap with real PII defeats the purpose of the data-free approach.
- **Do**: Keep LoRA rank low (r=4) and target MLP layers only. The paper shows this preserves utility better than full-model or attention-targeted updates.
- **Do**: Tune beta (privacy weight) based on your compliance requirements. Higher beta = more aggressive erasure but slightly more utility loss.
- **Avoid**: Applying gradient ascent over all tokens uniformly. This destroys fluency. The token-level mask is what makes selective unlearning work.
- **Avoid**: Skipping the inverter quality check. If the inverter achieves less than F1=30% or BLEU=15% on a validation set, the pseudo-PII will be too noisy and unlearning will target wrong tokens.
- **Avoid**: Running unlearning for too many epochs (>15). Over-unlearning degrades the model on non-PII text. Monitor perplexity each epoch and stop when ERR hits 0%.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| ERR stays high after unlearning | Inverter produced low-quality pseudo-PII | Retrain inverter with more diverse templates; check logit extraction fidelity |
| Perplexity spikes >20% | beta too high or too many epochs | Reduce beta, lower epoch count, or increase alpha to weight utility preservation |
| Mask annotator misses PII entities | Few-shot prompt inadequate | Add more few-shot examples covering the missed entity types; consider a dedicated NER model |
| LoRA training diverges | Learning rate too high for model scale | Use 5e-5 for models >1B; use cosine schedule with warmup |
| Utility drops on downstream task | Unlearning affected task-relevant representations | Switch LoRA target from Full to MLP-only; reduce rank from 4 to 2 |
| Inverter training fails to converge | Logit distribution too flat (high temperature) | Collect logits at temperature=1.0; increase inverter training epochs to 50 |

## Limitations

- **Model scale**: Validated on Pythia models up to 1.4B parameters. Effectiveness on 7B+ models is plausible but unverified -- larger models may require higher LoRA rank or more inversion data.
- **PII type coverage**: Works best for structured PII (names, emails, SSNs, phone numbers). Unstructured sensitive information (e.g., "the patient who lives near the park on 5th street") is harder to mask and may evade token-level annotation.
- **Inverter dependency**: The quality of unlearning is bounded by the inverter's reconstruction quality. Poor inversion means the wrong tokens get targeted.
- **Not a guarantee**: Reduces PII extraction probability dramatically (ERR to 0% in experiments) but cannot provide cryptographic guarantees of erasure. Adversarial extraction with sufficient compute may still recover fragments.
- **Single-pass limitation**: Designed for removing a known set of PII categories. If new categories emerge post-deployment, the pipeline must be re-run.
- **No multi-turn reasoning**: The method operates on individual token sequences. PII spread across multiple conversation turns requires aggregation before processing.

## Reference

**Paper**: [Data-Free Privacy-Preserving for LLMs via Model Inversion and Selective Unlearning](https://arxiv.org/abs/2601.15595v1) (Zhou et al., 2026)
**Key insight**: You do not need the original training data to unlearn PII. A model inverter can reconstruct pseudo-PII from output logits, and a contrastive mask loss confined to LoRA adapters can erase sensitive tokens while preserving everything else. Look for Algorithm 1 (Appendix A) for the complete pseudocode and Tables 1-2 for privacy/utility tradeoff results across Pythia-160M, 410M, and 1.4B.