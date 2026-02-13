---
name: "beyond-superficial-unlearning-sharpness-aware"
description: "Implement sharpness-aware robust erasure (SARE) for hallucination removal in multimodal LLMs. Uses Targeted-SAM to flatten the loss landscape so unlearning is geometrically stable against relearning. Triggers: 'unlearn hallucinations from MLLM', 'robust hallucination erasure', 'sharpness-aware unlearning', 'prevent hallucination resurgence after fine-tuning', 'SARE training loop', 'flatten loss landscape for unlearning'"
---

# Sharpness-Aware Robust Erasure (SARE) for Hallucination Unlearning

This skill enables Claude to implement the SARE framework from "Beyond Superficial Unlearning" (arXiv:2601.16527), which removes object hallucinations from multimodal LLMs in a way that resists resurgence during subsequent fine-tuning. The core insight: standard unlearning traps models in sharp minima where hallucinations trivially resurface. SARE solves this by casting unlearning as a min-max optimization problem, using Targeted Sharpness-Aware Minimization (Targeted-SAM) to flatten the loss landscape around hallucinated concepts so that suppression persists under weight perturbations.

## When to Use

- When the user wants to remove object hallucinations from a vision-language model (LLaVA, InstructBLIP, MiniGPT-4) and needs the fix to survive further training
- When implementing a machine unlearning pipeline where standard gradient ascent on a forget set causes catastrophic resurgence after relearning
- When the user asks to build a training loop that uses sharpness-aware minimization for targeted concept erasure
- When evaluating hallucination robustness against relearning attacks, LoRA fine-tuning, or adversarial prompts
- When constructing CLIP-filtered forget/retain datasets for fine-grained subsentence-level unlearning
- When the user needs to reproduce or extend the SARE results on CHAIR, POPE, or similar hallucination benchmarks

## Key Technique

**The Problem with Standard Unlearning.** Methods like gradient ascent (GA) or negative preference optimization (NPO) suppress hallucinations by pushing the model away from hallucinated outputs. But this creates a sharp minimum: the model's loss landscape has a narrow valley where hallucinations are suppressed, surrounded by regions where they persist. A small parameter shift from continued training pushes the model out of that valley, and hallucinations return. Empirically, just 140 relearning samples can undo standard erasure.

**Targeted-SAM: Flattening the Landscape.** SARE addresses this with a two-step gradient computation inspired by Sharpness-Aware Minimization (SAM). At each training step, it first computes a worst-case perturbation `epsilon* = rho * grad(L_neg) / ||grad(L_neg)||` that maximally reactivates hallucinations within a radius `rho`. Then it computes the actual update gradient at these *perturbed* weights, not the original ones. This forces the optimizer to find flat regions where hallucinations stay suppressed even under weight shifts. The final gradient aggregates three signals: (1) hallucination erasure at perturbed weights (`lambda_1 * grad L_neg(theta + epsilon*)`), (2) visual grounding preservation at current weights (`grad L_pos(theta)`), and (3) sentence-level coherence retention (`lambda_2 * grad L_sent(theta)`).

**Data Curation via CLIP Filtering.** The forget set, retain set, and sentence-level set are split at the subsentence level using CLIP alignment scores. Objects with score below threshold T1 are hallucinated (forget set D_neg). Objects above T0 are grounded (retain set D_pos). Full sentences above T2 form the coherence set D_sent. This granularity lets SARE target individual hallucinated objects without damaging correctly grounded descriptions.

## Step-by-Step Workflow

1. **Prepare the base model and data.** Load the multimodal LLM (e.g., LLaVA-v1.5-7B) and a caption dataset with image-response pairs. Ensure CLIP (ViT-L/14) is available for scoring.

2. **Score objects with CLIP alignment.** For each generated caption, extract mentioned objects. Compute CLIP similarity `S(o)` between each object phrase and the image. Objects with `S(o) < T1` go into D_neg (hallucinated). Objects with `S(o) > T0` go into D_pos (grounded). Full sentences with `S(y) > T2` go into D_sent.

3. **Construct subsentence training samples.** For D_neg, create (image, hallucinated-object-phrase) pairs targeted for erasure. For D_pos, create (image, grounded-object-phrase) pairs for retention. For D_sent, keep full (image, sentence) pairs for coherence.

4. **Define the three loss components.** Implement `L_neg` as inverted cross-entropy (gradient ascent direction) on D_neg, `L_pos` as standard cross-entropy on D_pos, and `L_sent` as standard cross-entropy on D_sent.

5. **Implement the Targeted-SAM inner step.** At each iteration, compute `grad_neg = gradient(L_neg, theta)`. Calculate the perturbation: `epsilon = rho * grad_neg / grad_neg.norm()`. This is the worst-case direction that reactivates hallucinations.

6. **Compute the perturbed gradient.** Temporarily shift parameters to `theta + epsilon`. Compute `grad_neg_perturbed = gradient(L_neg, theta + epsilon)`. Restore parameters to `theta`.

7. **Aggregate the final gradient.** Combine: `g_final = lambda_1 * grad_neg_perturbed + grad_pos + lambda_2 * grad_sent`. Apply optimizer step with this gradient.

8. **Train for exactly 1 epoch.** SARE is sensitive to overtraining--more than 1 epoch risks grounding collapse. Monitor CHAIR_S and POPE during training.

9. **Evaluate hallucination suppression.** Run CHAIR (sentence-level and instance-level), POPE (accuracy, F1, yes-ratio), BLEU, and perplexity on the validation set.

10. **Test robustness against relearning.** Fine-tune the unlearned model on a small set of hallucination-containing samples (e.g., 140 samples). Measure whether CHAIR scores degrade. A robust model should maintain suppression within ~1% of pre-relearning scores.

## Concrete Examples

**Example 1: Implementing the SARE training loop in PyTorch**

User: "I have a LLaVA model that hallucinates objects. Help me implement the SARE training loop to robustly remove hallucinations."

Approach:
1. Set up CLIP scoring to partition data into D_neg, D_pos, D_sent
2. Implement the three loss functions
3. Code the Targeted-SAM two-step gradient
4. Wire up the training loop with single-epoch constraint

Output:
```python
import torch
from torch.nn import CrossEntropyLoss

class SARETrainer:
    def __init__(self, model, optimizer, rho=0.05, lambda_1=1.0, lambda_2=0.5):
        self.model = model
        self.optimizer = optimizer
        self.rho = rho
        self.lambda_1 = lambda_1
        self.lambda_2 = lambda_2
        self.ce_loss = CrossEntropyLoss()

    def compute_loss(self, batch, loss_type="pos"):
        """Compute cross-entropy loss. For 'neg', returns negated loss (gradient ascent)."""
        outputs = self.model(batch["images"], batch["input_ids"])
        logits = outputs.logits[:, :-1].contiguous()
        labels = batch["labels"][:, 1:].contiguous()
        loss = self.ce_loss(logits.view(-1, logits.size(-1)), labels.view(-1))
        return -loss if loss_type == "neg" else loss

    def targeted_sam_step(self, neg_batch, pos_batch, sent_batch):
        # Step 1: Compute gradient of L_neg at current weights
        self.optimizer.zero_grad()
        loss_neg = self.compute_loss(neg_batch, loss_type="neg")
        loss_neg.backward()

        # Collect grad_neg and compute perturbation direction
        grad_neg = {}
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                grad_neg[name] = param.grad.clone()

        grad_norm = torch.sqrt(
            sum(g.pow(2).sum() for g in grad_neg.values())
        )

        # Step 2: Apply perturbation epsilon = rho * grad_neg / ||grad_neg||
        with torch.no_grad():
            for name, param in self.model.named_parameters():
                if name in grad_neg:
                    param.add_(self.rho * grad_neg[name] / (grad_norm + 1e-12))

        # Step 3: Compute gradient of L_neg at perturbed weights
        self.optimizer.zero_grad()
        loss_neg_perturbed = self.compute_loss(neg_batch, loss_type="neg")
        loss_neg_perturbed.backward()

        grad_neg_perturbed = {}
        for name, param in self.model.named_parameters():
            if param.grad is not None:
                grad_neg_perturbed[name] = param.grad.clone()

        # Restore original weights
        with torch.no_grad():
            for name, param in self.model.named_parameters():
                if name in grad_neg:
                    param.sub_(self.rho * grad_neg[name] / (grad_norm + 1e-12))

        # Step 4: Compute gradients for retain and sentence losses
        self.optimizer.zero_grad()
        loss_pos = self.compute_loss(pos_batch, loss_type="pos")
        loss_sent = self.compute_loss(sent_batch, loss_type="pos")
        (loss_pos + self.lambda_2 * loss_sent).backward()

        # Step 5: Aggregate final gradient
        with torch.no_grad():
            for name, param in self.model.named_parameters():
                if param.grad is not None and name in grad_neg_perturbed:
                    param.grad.add_(self.lambda_1 * grad_neg_perturbed[name])

        self.optimizer.step()
        return {
            "loss_neg": loss_neg.item(),
            "loss_pos": loss_pos.item(),
            "loss_sent": loss_sent.item(),
        }
```

**Example 2: CLIP-based dataset partitioning**

User: "How do I split my caption data into forget, retain, and sentence sets using CLIP scores?"

Approach:
1. Extract per-object phrases from captions via noun-phrase parsing
2. Score each object against its source image using CLIP
3. Partition based on thresholds T0, T1, T2

Output:
```python
import clip
import spacy

nlp = spacy.load("en_core_web_sm")

def extract_objects(caption):
    """Extract noun phrases as object mentions."""
    doc = nlp(caption)
    return [chunk.text for chunk in doc.noun_chunks]

def score_objects(image, objects, clip_model, preprocess, device):
    """Compute CLIP alignment scores for each object against the image."""
    image_input = preprocess(image).unsqueeze(0).to(device)
    text_inputs = clip.tokenize(objects).to(device)
    with torch.no_grad():
        img_feat = clip_model.encode_image(image_input)
        txt_feat = clip_model.encode_text(text_inputs)
        img_feat /= img_feat.norm(dim=-1, keepdim=True)
        txt_feat /= txt_feat.norm(dim=-1, keepdim=True)
        scores = (img_feat @ txt_feat.T).squeeze(0)
    return scores.cpu().tolist()

def partition_data(dataset, clip_model, preprocess, device,
                   t0=0.25, t1=0.18, t2=0.22):
    """Split into D_neg (hallucinated), D_pos (grounded), D_sent (coherent)."""
    d_neg, d_pos, d_sent = [], [], []
    for sample in dataset:
        image, caption = sample["image"], sample["caption"]
        objects = extract_objects(caption)
        if not objects:
            continue
        scores = score_objects(image, objects, clip_model, preprocess, device)

        for obj, score in zip(objects, scores):
            if score < t1:
                d_neg.append({"image": image, "text": obj})
            elif score > t0:
                d_pos.append({"image": image, "text": obj})

        # Sentence-level: average CLIP score for full caption
        sent_score = score_objects(image, [caption], clip_model, preprocess, device)[0]
        if sent_score > t2:
            d_sent.append({"image": image, "text": caption})

    return d_neg, d_pos, d_sent
```

**Example 3: Robustness evaluation after relearning**

User: "I applied SARE. How do I test whether hallucination suppression survives relearning?"

Approach:
1. Save the SARE-unlearned checkpoint
2. Fine-tune it on a small hallucination-containing set
3. Compare CHAIR before and after relearning

Output:
```python
def evaluate_relearning_robustness(model, relearn_data, eval_data,
                                   chair_evaluator, num_relearn_samples=140,
                                   relearn_epochs=3, lr=2e-5):
    """Test whether hallucination suppression survives relearning."""
    # Baseline: evaluate SARE model before relearning
    chair_before = chair_evaluator.evaluate(model, eval_data)
    print(f"CHAIR_S before relearning: {chair_before['chair_s']:.1f}")
    print(f"CHAIR_I before relearning: {chair_before['chair_i']:.1f}")

    # Simulate relearning attack
    relearn_subset = relearn_data[:num_relearn_samples]
    optimizer = torch.optim.AdamW(model.parameters(), lr=lr)
    model.train()
    for epoch in range(relearn_epochs):
        for batch in relearn_subset:
            loss = compute_standard_loss(model, batch)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()

    # Evaluate after relearning
    chair_after = chair_evaluator.evaluate(model, eval_data)
    print(f"CHAIR_S after relearning:  {chair_after['chair_s']:.1f}")
    print(f"CHAIR_I after relearning:  {chair_after['chair_i']:.1f}")

    degradation = chair_after["chair_s"] - chair_before["chair_s"]
    print(f"Degradation: {degradation:+.1f} (target: < 1.0)")
    # SARE typically holds within ~0.2; standard methods degrade by 7-10+
    return {"before": chair_before, "after": chair_after, "delta": degradation}
```

## Best Practices

- **Do:** Train for exactly 1 epoch. The paper shows grounding collapse occurs with extended training. Monitor POPE accuracy as an early-stopping signal.
- **Do:** Use subsentence-level granularity for forget/retain sets. Object-level targeting preserves more useful knowledge than sentence-level erasure.
- **Do:** Set `rho` conservatively (start at 0.05). Larger perturbation radii increase robustness but risk destabilizing the retain-set loss.
- **Do:** Evaluate with both CHAIR (hallucination rate) and POPE (grounding accuracy) together. Reducing hallucinations at the cost of grounding defeats the purpose.
- **Avoid:** Using gradient ascent alone without the SAM perturbation step. Standard GA creates sharp minima that fail under relearning.
- **Avoid:** Setting `lambda_1` too high relative to the retain losses. This aggressively erases hallucinations but degrades generation quality. Balance is critical.

## Error Handling

- **Gradient explosion during perturbation:** If `grad_norm` is near zero (flat region already), the perturbation direction is undefined. Add epsilon to the denominator: `epsilon = rho * grad / (grad_norm + 1e-12)`.
- **CLIP score distribution is bimodal:** If almost all objects score either very high or very low, the thresholds T0/T1 may exclude too many samples. Plot the score histogram and adjust thresholds to capture 10-20% of objects in D_neg.
- **Model collapses to short/generic outputs:** The retain loss (`L_pos + L_sent`) is too weak. Increase `lambda_2` or add more samples to D_sent to anchor generation quality.
- **Hallucinations persist after training:** Check that D_neg actually contains the problematic objects. CLIP scoring can miss abstract hallucinations (e.g., spatial relationships). Consider supplementing with an object detector like DETIC for ground-truth filtering.
- **Out-of-memory on two forward passes:** The Targeted-SAM step requires two forward-backward passes per iteration. Use gradient checkpointing or reduce batch size by half compared to standard fine-tuning.

## Limitations

- SARE is designed for **object hallucinations** (non-existent entities). It does not address attribute errors (wrong color), relational errors (wrong spatial arrangement), or factual errors in text-only generation.
- The CLIP-based filtering assumes hallucinated objects have low visual-text alignment. Abstract or context-dependent hallucinations (e.g., inferring "a person" from shadows) may score deceptively high.
- The method requires two gradient computations per step, roughly doubling training cost compared to standard unlearning. For very large models (70B+), this may require significant infrastructure.
- Robustness is validated against lightweight relearning (~140 samples). Aggressive fine-tuning on thousands of hallucination-containing samples may eventually override the flat minimum.
- Subsentence parsing quality depends on the NLP pipeline. Complex captions with nested clauses may be incorrectly segmented, placing grounded objects in D_neg.

## Reference

**Paper:** "Beyond Superficial Unlearning: Sharpness-Aware Robust Erasure of Hallucinations in Multimodal LLMs" -- arXiv:2601.16527
**Key insight:** Standard unlearning creates sharp minima where hallucinations trivially resurface. Targeted-SAM flattens the loss landscape to make erasure geometrically stable. Focus on Section 3 (Method) for the optimization formulation and Algorithm 1 for the complete training procedure.