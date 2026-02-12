---
name: "thinking-broad-acting-fast"
description: "Build production search relevance systems using Multi-Perspective Chain-of-Thought distillation into lightweight student models. Use when: 'build a query-product relevance model', 'distill LLM reasoning into BERT', 'multi-perspective CoT for search', 'latent reasoning distillation', 'e-commerce relevance classifier', 'train a fast relevance model from LLM teacher'."
---

# Thinking Broad, Acting Fast: Multi-Perspective CoT Distillation for Search Relevance

This skill enables Claude to build end-to-end search relevance systems that distill rich multi-perspective reasoning from a large teacher LLM into a lightweight student model (BERT-class) that retains structured reasoning capability at inference time. The core technique -- Multi-Perspective Chain-of-Thought (MPCoT) combined with Latent Reasoning Knowledge Distillation (LRKD) -- produces student models that run 300x faster than the teacher while capturing diverse reasoning signals across user intent, attribute matching, and business rules.

## When to Use

- When building a query-product or query-document relevance classifier for search or advertising
- When distilling an LLM's chain-of-thought reasoning into a BERT-sized model for real-time serving
- When a single-perspective relevance model struggles with ambiguous, long-tail, or multi-attribute queries
- When you need to generate diverse training rationales from multiple analytical angles using an LLM
- When implementing knowledge distillation that preserves reasoning structure beyond simple label matching
- When deploying a relevance model under strict latency constraints (<150ms) while retaining reasoning quality

## Key Technique

**Multi-Perspective CoT (MPCoT):** Instead of prompting an LLM for a single chain-of-thought, generate three independent rationales per query-product pair: (1) a *User Intent* perspective that simulates a shopper's intuitive thought process, focusing on use-case scenarios and functional needs; (2) a *Structured Analysis* perspective that performs systematic attribute-by-attribute comparison (brand, category, model, specifications); and (3) a *Business Rule* perspective that applies domain heuristics for common ambiguities (accessories vs. main products, compatible substitutes, brand-specific constraints). Each perspective catches different error types -- intent handles category mismatches, structured analysis catches attribute mismatches, and business rules resolve accessory disambiguation. The teacher is first fine-tuned via SFT on all correct rationales, then hardened via DPO on conflict samples where perspectives disagree, teaching the model to prefer the correct reasoning path in context.

**Latent Reasoning Knowledge Distillation (LRKD):** Rather than discarding CoT at distillation time (using rationales only as auxiliary training signal), LRKD equips the student model with a lightweight *latent reasoning extractor* that persists at inference. The student backbone (e.g., BERT) encodes the query-product pair into token representations. The extractor -- implemented as an MLP, Poly-Encoder, or Graph Attention Network (GAT) -- maps these representations into a latent reasoning vector. During training, this vector is aligned (via MSE loss) with a frozen text embedding of the teacher's CoT rationale. At inference, the extractor runs without any text generation, producing a compact reasoning vector that is concatenated with the [CLS] token for final classification. This adds <1M parameters and <17ms latency while delivering +2.4% accuracy and +3.9% F1 over the BERT baseline.

## Step-by-Step Workflow

1. **Define the relevance taxonomy.** Establish the label set for your domain. For e-commerce: `Exact`, `Substitute`, `Complement`, `Irrelevant`. For fine-grained: add `Brand Mismatch`, `Category Mismatch`, `Attribute Mismatch`, `Accessory Mismatch`. Each label must have a clear definition with examples.

2. **Design three perspective prompts.** Write few-shot prompt templates for each perspective:
   - *User Intent*: "As a shopper searching for '{query}', would product '{title}' meet your needs? Think through what a user likely wants and why."
   - *Structured Analysis*: "Compare the query '{query}' against the product '{title}' attribute by attribute: category, brand, model, key specifications. List matches and mismatches."
   - *Business Rules*: "Apply these rules: (a) accessories for X are not X itself, (b) compatible substitutes require functional equivalence, (c) brand-specific queries require brand match. Evaluate '{query}' vs '{title}'."

3. **Generate MPCoT training data.** For each query-product pair in your seed dataset, prompt the teacher LLM (e.g., Qwen3-14B, Llama 3, GPT-4) with all three perspective templates. Filter outputs: keep only rationales whose predicted label matches ground truth. Aggregate all passing rationales into a unified SFT dataset.

4. **Fine-tune the teacher with SFT.** Apply LoRA fine-tuning on the base LLM using the aggregated MPCoT dataset. Use standard next-token prediction loss. Recommended: 3 epochs, learning rate 5e-5, LoRA rank 16.

5. **Harden the teacher with DPO.** Identify *conflict samples* where at least one perspective produced an incorrect rationale. Construct preference pairs: the incorrect rationale as `rejected`, a correct rationale (from pass@5 sampling) as `chosen`. Cross-perspective pairing is key -- pair a wrong business-rule rationale with a correct intent rationale to teach flexible perspective weighting. Train with DPO loss for 3 epochs at learning rate 5e-6.

6. **Generate distillation targets.** Run the hardened teacher over the full training set to produce final CoT rationales. Embed each rationale using a frozen sentence encoder (e.g., BGE-M3) to obtain target embedding vectors `e_cot`.

7. **Build the student model with latent reasoning extractor.** Architecture:
   - Backbone: BERT-multilingual-base (or domain BERT) encoding `[CLS] query [SEP] product_title [SEP]` into token representations `H`.
   - Extractor: A 2-layer MLP (best balance of quality and speed) or GAT (best quality) that maps `H` to a latent reasoning vector `r` of dimension `d`.
   - Classification head: Linear layer over `concat([CLS], r)` projecting to `C` relevance classes.

8. **Train the student with combined loss.** Minimize `L = L_cls + lambda * L_guide` where:
   - `L_cls`: cross-entropy on relevance labels
   - `L_guide`: MSE between student's latent vector `r` and teacher's CoT embedding `e_cot`
   - `lambda = 0.1` (paper default; tune on validation set)
   Train for 5 epochs, learning rate 2e-6, max sequence length 128.

9. **Validate with probing analysis.** After training, verify the extractor learned meaningful reasoning by probing: extract `r` vectors for test samples, train a small linear probe to predict reasoning-relevant features (mismatch type, key attribute). The latent vector should substantially outperform raw [CLS] on non-trivial cases.

10. **Deploy with the extractor retained.** At inference, the full pipeline runs: BERT encode -> extractor -> concat with [CLS] -> classify. No text generation occurs. Expected latency: 130-150ms per batch on A100 (vs. ~47,000ms for the teacher).

## Concrete Examples

**Example 1: Building an E-Commerce Relevance Pipeline**

User: "I need to build a query-product relevance model for our marketplace. We have 100K labeled pairs. The current BERT model struggles with ambiguous queries like 'apple charger' matching to phone cases."

Approach:
1. Define labels: Exact, Substitute, Complement, Irrelevant
2. Write three perspective prompts targeting the specific ambiguity patterns in the data
3. Generate MPCoT data using an LLM (e.g., Qwen3-14B via vLLM):

```python
PERSPECTIVES = {
    "user_intent": """You are a shopper. Given query: "{query}" and product: "{title}".
Think step-by-step: What is the shopper likely looking for? Does this product fulfill that need?
Provide your reasoning, then classify as: Exact, Substitute, Complement, or Irrelevant.""",

    "structured_analysis": """Systematically compare query "{query}" with product "{title}".
Step 1: Identify query category vs product category.
Step 2: Compare brand, model, key attributes.
Step 3: Note matches and mismatches.
Classify as: Exact, Substitute, Complement, or Irrelevant.""",

    "business_rules": """Apply these e-commerce rules to query "{query}" and product "{title}":
Rule 1: Accessories for X are not X (e.g., "iPhone case" is not "iPhone").
Rule 2: Brand-specific queries require brand match ("Nike shoes" must be Nike).
Rule 3: Substitutes must be functionally equivalent in the primary use case.
Classify as: Exact, Substitute, Complement, or Irrelevant."""
}

# Generate rationales, filter for label correctness
for perspective, template in PERSPECTIVES.items():
    prompt = template.format(query=query, title=title)
    response = teacher_model.generate(prompt)
    predicted_label = extract_label(response)
    if predicted_label == ground_truth_label:
        sft_data.append({"input": prompt, "output": response, "perspective": perspective})
```

4. Fine-tune teacher with SFT on aggregated rationales, then DPO on conflict samples
5. Build student with LRKD:

```python
class RelevanceStudentModel(nn.Module):
    def __init__(self, num_classes=4, hidden_dim=768):
        super().__init__()
        self.backbone = AutoModel.from_pretrained("bert-base-multilingual-cased")
        # Latent Reasoning Extractor (MLP variant)
        self.extractor = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.GELU(),
            nn.Linear(hidden_dim, hidden_dim)
        )
        # Classification head over [CLS] + latent reasoning vector
        self.classifier = nn.Linear(hidden_dim * 2, num_classes)

    def forward(self, input_ids, attention_mask, cot_embedding=None):
        outputs = self.backbone(input_ids, attention_mask=attention_mask)
        cls_repr = outputs.last_hidden_state[:, 0]  # [CLS]
        token_reprs = outputs.last_hidden_state      # All tokens
        # Extractor: transform tokens, then masked mean pool
        extracted = self.extractor(token_reprs)
        mask = attention_mask.unsqueeze(-1).float()
        reasoning_vector = (extracted * mask).sum(1) / mask.sum(1)
        # Classify from combined representation
        combined = torch.cat([cls_repr, reasoning_vector], dim=-1)
        logits = self.classifier(combined)
        # Compute losses
        loss_cls = F.cross_entropy(logits, labels)
        if cot_embedding is not None:
            loss_guide = F.mse_loss(reasoning_vector, cot_embedding)
            loss = loss_cls + 0.1 * loss_guide
        else:
            loss = loss_cls
        return logits, loss
```

Output: A student model at ~169M params serving at <150ms with +2-4% accuracy over vanilla BERT.

**Example 2: Distilling Reasoning for Document Retrieval Relevance**

User: "We have a legal document search system. Our reranker LLM is too slow for production. Can we distill its reasoning into something faster?"

Approach:
1. Adapt the three perspectives for legal domain:
   - *User Intent*: What legal concept is the searcher investigating?
   - *Structured Analysis*: Compare query terms against document sections, statutes cited, jurisdiction
   - *Domain Rules*: Jurisdictional relevance, recency of precedent, hierarchical court authority
2. Generate MPCoT rationales from the reranker LLM on the labeled query-document pairs
3. Embed rationales with a sentence encoder (BGE-M3 or domain-specific legal encoder)
4. Train a legal-BERT student with the latent reasoning extractor and combined loss
5. Deploy the student as the reranker; reserve the teacher for periodic data refresh

Output: Reranker latency drops from seconds to milliseconds while retaining multi-faceted legal reasoning signals.

**Example 3: Adding DPO Hardening to an Existing CoT Pipeline**

User: "We already generate single-perspective CoT for our relevance model. How do we add multi-perspective reasoning and DPO?"

Approach:
1. Keep the existing perspective as Perspective 1
2. Design two additional perspectives that cover orthogonal failure modes:
   - Analyze your current model's top errors to identify what reasoning angles are missing
   - Common additions: attribute-level matching (catches specification mismatches) and business heuristics (catches accessory/substitute confusion)
3. Generate new rationales for the full training set with all three perspectives
4. Identify conflict samples (where perspectives disagree on label)
5. Construct DPO pairs from conflicts:

```python
# For each conflict sample, pair wrong rationale with correct one
for sample in conflict_samples:
    wrong_rationales = [r for r in sample.rationales if r.predicted != sample.ground_truth]
    correct_rationales = [r for r in sample.rationales if r.predicted == sample.ground_truth]
    # If no correct rationale exists, sample from teacher with pass@5
    if not correct_rationales:
        correct_rationales = teacher.generate(sample.prompt, n=5, filter_correct=True)
    for wrong in wrong_rationales:
        for correct in correct_rationales:
            dpo_pairs.append({"chosen": correct.text, "rejected": wrong.text, "input": sample.input})
```

6. Fine-tune teacher with DPO on these pairs (3 epochs, lr=5e-6)

Output: Teacher accuracy improves ~2% from DPO hardening; downstream student inherits the improvement.

## Best Practices

- **Do:** Use all three perspectives during data generation even if one seems redundant -- empirical analysis shows each catches distinct error types that the others miss (intent: category errors; structured: attribute errors; rules: accessory confusion).
- **Do:** Keep the latent reasoning extractor at inference time. The whole point of LRKD over standard distillation is that the extractor continues to contribute reasoning signal during serving.
- **Do:** Use a frozen, high-quality sentence encoder (BGE-M3, E5-large) for embedding teacher CoTs. The embedding quality directly determines the guidance signal quality.
- **Do:** Start with the MLP extractor variant for prototyping (simplest, +1M params, strong results), then try GAT if you need maximum accuracy.
- **Avoid:** Using lambda > 0.3 for the guidance loss weight. Too much guidance loss overwhelms classification signal. The paper finds lambda=0.1 optimal.
- **Avoid:** Including rationales with incorrect predicted labels in SFT data. Consistency filtering (keeping only label-correct outputs) is essential for teacher quality.
- **Avoid:** Skipping the DPO stage. Conflict samples are where the model learns to arbitrate between perspectives, and DPO provides a +1-2% accuracy lift that propagates to the student.

## Error Handling

- **Teacher generates low-quality rationales:** If fewer than 50% of rationales pass consistency filtering, the few-shot examples in your perspective prompts need improvement. Iterate on prompt quality before scaling data generation.
- **Student guidance loss doesn't decrease:** Check that CoT embeddings are normalized and that the sentence encoder is truly frozen. A learning rate that is too high for the student backbone can also destabilize the extractor.
- **Conflict samples are too few for DPO:** If perspectives rarely disagree, the task may be too easy for the teacher or the perspectives are too similar. Redesign one perspective to be more aggressive or conservative to introduce productive disagreement.
- **Latency exceeds budget after adding extractor:** Switch from GAT to MLP extractor (saves ~5ms) or reduce extractor hidden dimension. The Poly-Encoder variant adds only ~30K parameters with minimal latency impact.
- **Student overfits on small datasets:** Reduce lambda to 0.05 to lean more on classification signal, or augment training data with teacher-generated labels on unlabeled query-product pairs.

## Limitations

- Requires a labeled seed dataset with at least 10K-50K query-product pairs to generate meaningful MPCoT data. Cold-start scenarios with no labels need a separate bootstrapping strategy.
- The three perspectives (intent, structured, business rules) are designed for e-commerce and search relevance. Adapting to other domains (e.g., medical, legal) requires redesigning the perspectives around that domain's failure modes.
- LRKD assumes the teacher's CoT rationales can be meaningfully compressed into a single embedding vector. For tasks requiring very long or multi-step reasoning chains, a single vector may be a lossy bottleneck.
- The approach optimizes classification accuracy, not ranking metrics (NDCG, MRR). If your task is pointwise relevance classification this works directly; for listwise ranking, additional adaptation is needed.
- DPO requires conflict samples where perspectives disagree. If your domain has little ambiguity, the DPO stage adds minimal value and can be skipped.

## Reference

[Thinking Broad, Acting Fast: Latent Reasoning Distillation from Multi-Perspective Chain-of-Thought for E-Commerce Relevance](https://arxiv.org/abs/2601.21611v1) (WWW 2026 Industry Track). Key sections: Section 3.1 for MPCoT perspective design, Section 3.2 for SFT+DPO teacher pipeline, Section 3.3 for LRKD student architecture and loss formulation, Section 4.3 for ablation results showing each component's contribution.