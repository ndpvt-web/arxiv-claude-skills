---
name: "pand-prompt-aware-neighborhood-distillation"
description: "Implement PAND (Prompt-Aware Neighborhood Distillation) for distilling Vision-Language Models into lightweight networks for fine-grained visual classification. Use when: 'distill CLIP into ResNet for fine-grained classification', 'knowledge distillation for bird/car/aircraft recognition', 'compress VLM into lightweight student model', 'fine-grained image classification with small model', 'prompt-aware distillation from CLIP', 'train lightweight classifier using CLIP knowledge'."
---

# PAND: Prompt-Aware Neighborhood Distillation

This skill enables Claude to implement and configure the PAND framework — a two-stage knowledge distillation pipeline that transfers fine-grained visual classification ability from large Vision-Language Models (like CLIP) into lightweight student networks (ResNet-18, MobileNet-V2). PAND decouples the distillation into (1) learning adaptive text prompts that serve as semantic anchors via CoOp-style context optimization, and (2) a neighborhood-aware structural distillation loss that preserves the teacher's local decision geometry around each sample. This yields state-of-the-art lightweight FGVC accuracy — 76.09% on CUB-200 with ResNet-18, a +3.4% improvement over prior best.

## When to Use

- When the user wants to distill CLIP or another VLM into a small CNN (ResNet-18, MobileNet-V2) for fine-grained classification tasks like bird species, car models, aircraft variants, or flower types.
- When building a deployment-ready fine-grained classifier that must run on edge devices or with low latency, but needs VLM-level discriminative power.
- When the user asks how to use learned prompts (CoOp) as fixed supervision targets for training a separate student network.
- When implementing knowledge distillation that preserves local neighborhood structure rather than only global feature alignment.
- When the user needs to set up a two-stage training pipeline: prompt learning followed by structural distillation.
- When choosing loss functions and hyperparameters for VLM-to-CNN distillation on FGVC benchmarks.

## Key Technique

**Stage 1 — Prompt-Aware Semantic Calibration (PSC):** Instead of hand-crafting text prompts like `"a photo of a {class}"`, PAND learns continuous context vectors using Context Optimization (CoOp). Each class prompt is parameterized as `p_c = [v_1, v_2, ..., v_16, w_c]`, where `v_i` are learnable tokens and `w_c` is the frozen class name embedding. A symmetric cross-entropy loss aligns CLIP image features with these learned text features. After training (200 epochs, SGD, lr=0.002), the optimized text features `F_txt = [f_txt_1, ..., f_txt_C]` are frozen and exported as **adaptive semantic anchors** — fixed targets that encode task-specific, fine-grained distinctions far richer than static templates.

**Stage 2 — Neighborhood-Aware Structural Distillation (NSD):** The student network trains against a composite loss. The base loss `L_base = λ_cls * L_cls + λ_vis * L_vis + λ_txt * L_txt` combines standard cross-entropy with visual and textual alignment terms using the frozen semantic anchors. The key innovation is the **Neighborhood Logit Relation Distillation (NLRD)** loss: for each sample, PAND identifies the Top-K non-ground-truth classes with the highest teacher logits (the "confusing neighbors"), computes logit margins `Δ_ij = z_yi - z_j` for both teacher and student, normalizes them into distributions via softmax, and minimizes their Jensen-Shannon divergence. This forces the student to replicate the teacher's local confusion patterns — critical for fine-grained tasks where classes are visually similar.

**Why it works:** Global alignment losses treat all classes equally, but fine-grained classification hinges on distinguishing near-duplicates. By explicitly distilling the teacher's ranking of the K most confusable classes for each sample, PAND preserves the decision boundary geometry that matters most. Ablations confirm NSD contributes +3.24% alone vs. +0.85% for PSC alone on CUB-200.

## Step-by-Step Workflow

1. **Select teacher and student architectures.** Use a CLIP model (e.g., ViT-B/16) as the frozen teacher. Choose a lightweight student: ResNet-18 (76.09% on CUB-200) or MobileNet-V2 (76.52% on CUB-200). Install dependencies: `torch`, `pytorch-lightning`, `open-clip-torch`, `hydra-core`.

2. **Prepare dataset configuration.** Create a YAML config specifying `data_root`, `class_num`, class names list, and a `prompt_tmpl` string. Use domain-appropriate templates:
   - CUB-200: `"a photo of a {}, a type of bird."`
   - Stanford Cars: `"a photo of a {}, a type of car."`
   - FGVC-Aircraft: `"a photo of a {}, a type of aircraft."`
   - Oxford Flowers: `"a photo of a {}, a type of flower."`

3. **Run Stage-PSC: Learn adaptive semantic anchors.** Train CoOp context vectors (16 tokens) with SGD (lr=0.002, momentum=0.9, no weight decay) for 200 epochs, batch size 128. Freeze both CLIP image and text encoders — only the context vectors are optimized. Use symmetric cross-entropy between CLIP image features and the prompted text features.

4. **Export learned text features.** After PSC converges, extract and save the calibrated text features as `learned_text_features.pt` — a tensor of shape `[C, D]` where C is the number of classes and D is the CLIP embedding dimension.

5. **Configure Stage-NSD losses.** Set up the composite loss with these components:
   - `L_cls`: Standard cross-entropy on ground-truth labels.
   - `L_vis`: KL divergence between student visual logits and teacher visual logits (temperature-scaled).
   - `L_txt`: Alignment loss between student features and the frozen semantic anchors from Stage-PSC.
   - `L_NSD`: NLRD loss with K=3 nearest confusing classes, λ_NSD=0.5.

6. **Train the student network (Stage-NSD).** Use AdamW optimizer (lr=1e-4, weight decay=1e-4) with cosine annealing for 300 epochs, batch size 128 (scale linearly with GPUs). The teacher and semantic anchors remain frozen throughout.

7. **Implement the NLRD loss computation.** For each sample in the batch: (a) get teacher logits and identify Top-K classes by teacher confidence excluding ground truth, (b) compute logit margins for teacher and student: `Δ_ij = z_yi - z_j` for each neighbor j, (c) normalize margins to probability distributions via softmax, (d) compute JS divergence between teacher and student distributions, (e) average across samples.

8. **Tune λ_NSD via validation.** Sweep λ_NSD in [0.1, 0.3, 0.5, 0.7, 1.0]. Optimal is typically 0.5 — performance degrades when NSD loss dominates (>0.5) because it can suppress the base classification signal.

9. **Evaluate and visualize.** Test on held-out set. Use t-SNE on student features to verify tighter intra-class clustering and clearer inter-class separation compared to baseline distillation.

10. **Export the student model.** The final artifact is a standalone lightweight network (ResNet-18 or MobileNet-V2) with no dependency on CLIP at inference time — only the student CNN is deployed.

## Concrete Examples

**Example 1: Distilling CLIP into ResNet-18 for bird species classification**

User: "I want to train a lightweight ResNet-18 to classify 200 bird species from CUB-200, using knowledge from CLIP. Help me set up the PAND pipeline."

Approach:
1. Set up the dataset config for CUB-200 with 200 classes and template `"a photo of a {}, a type of bird."`.
2. Implement Stage-PSC with 16 learnable context tokens:

```python
import torch
import torch.nn as nn
from open_clip import create_model_and_transforms, tokenize

class CoOpPromptLearner(nn.Module):
    def __init__(self, clip_model, class_names, n_ctx=16):
        super().__init__()
        self.n_ctx = n_ctx
        # Initialize learnable context vectors from CLIP's token embedding
        ctx_dim = clip_model.token_embedding.embedding_dim
        self.ctx = nn.Parameter(torch.randn(n_ctx, ctx_dim) * 0.02)
        # Tokenize class names and store embeddings (frozen)
        self.class_tokens = self._prepare_class_tokens(clip_model, class_names)

    def forward(self, clip_text_encoder):
        # Concatenate [learnable_ctx | class_name_embedding] for each class
        prompts = torch.cat([self.ctx.unsqueeze(0).expand(len(self.class_tokens), -1, -1),
                             self.class_tokens], dim=1)
        return clip_text_encoder(prompts)  # [C, D] text features
```

3. Train PSC for 200 epochs with symmetric cross-entropy, then save `learned_text_features.pt`.
4. Implement Stage-NSD with the NLRD loss:

```python
class NLRDLoss(nn.Module):
    def __init__(self, k=3, temperature=1.0):
        super().__init__()
        self.k = k
        self.temperature = temperature

    def forward(self, student_logits, teacher_logits, targets):
        bs = student_logits.size(0)
        loss = 0.0
        for i in range(bs):
            yi = targets[i]
            t_logits = teacher_logits[i].clone()
            t_logits[yi] = -float('inf')  # exclude ground truth
            _, topk_idx = t_logits.topk(self.k)  # Top-K confusing classes

            # Compute logit margins
            t_margins = teacher_logits[i, yi] - teacher_logits[i, topk_idx]
            s_margins = student_logits[i, yi] - student_logits[i, topk_idx]

            # Normalize to distributions
            t_dist = torch.softmax(t_margins / self.temperature, dim=0)
            s_dist = torch.softmax(s_margins / self.temperature, dim=0)

            # Jensen-Shannon divergence
            m = 0.5 * (t_dist + s_dist)
            js = 0.5 * (t_dist * (t_dist / m).log()).sum() + \
                 0.5 * (s_dist * (s_dist / m).log()).sum()
            loss += js
        return loss / bs
```

5. Train ResNet-18 student for 300 epochs with `L_total = L_base + 0.5 * L_NSD`.

Output: A standalone ResNet-18 achieving ~76% accuracy on CUB-200, deployable without CLIP.

**Example 2: Adapting PAND for a custom fine-grained product classification task**

User: "I have 150 classes of sneaker models. I want to distill CLIP into MobileNet-V2 for a mobile app. How do I adapt PAND?"

Approach:
1. Create dataset config with your 150 sneaker classes and template `"a photo of {}, a type of sneaker."`.
2. Run Stage-PSC exactly as standard — the CoOp context tokens adapt automatically to your domain since they are learned from your training data. Use n_ctx=16, 200 epochs.
3. For Stage-NSD, keep K=3 neighbors and λ_NSD=0.5 as defaults. If your classes have high visual similarity (e.g., color variants), consider K=5 to capture more confusing neighbors.
4. Use MobileNet-V2 as student with the same training recipe (AdamW, lr=1e-4, 300 epochs).
5. If your dataset is small (<5k images), reduce batch size to 64 and increase epochs to 400 to compensate.

Output: MobileNet-V2 model (~3.4M params) suitable for mobile deployment with fine-grained sneaker discrimination.

**Example 3: Implementing only the NLRD loss as a drop-in for existing distillation**

User: "I already have a KD pipeline. I just want to add the neighborhood-aware loss from PAND."

Approach:
1. Keep your existing teacher-student setup and base distillation loss unchanged.
2. Add the NLRD module from Example 1 to your loss computation.
3. Integrate it as an additive term:

```python
# In your training loop
base_loss = your_existing_kd_loss(student_out, teacher_out, targets)
nlrd_loss = nlrd_criterion(student_out['logits'], teacher_out['logits'], targets)
total_loss = base_loss + 0.5 * nlrd_loss
```

4. The NLRD loss is architecture-agnostic — it only requires logits from teacher and student, plus ground-truth labels. No changes needed to your model code.
5. Start with K=3 and λ_NSD=0.5, then tune on your validation set.

Output: Improved distillation accuracy with minimal code changes, especially on fine-grained or long-tail tasks.

## Best Practices

- **Do:** Use domain-specific prompt templates in Stage-PSC. `"a photo of a {}, a type of bird."` outperforms generic `"a photo of a {}."` because it constrains the semantic space to the relevant domain.
- **Do:** Freeze the CLIP encoders completely during Stage-PSC — only the 16 context tokens should be learnable. Unfreezing CLIP causes overfitting on small FGVC datasets.
- **Do:** Keep K small (3-5) for the neighborhood set. Fine-grained tasks have few truly confusing classes per sample; large K dilutes the signal with irrelevant classes.
- **Do:** Export and freeze semantic anchors between stages. Stage-NSD must not backpropagate into the text features — they are fixed supervision targets.
- **Avoid:** Setting λ_NSD above 0.5. The neighborhood loss can dominate and suppress the classification signal, degrading accuracy (demonstrated in sensitivity analysis).
- **Avoid:** Using NLRD on coarse-grained tasks (e.g., ImageNet-1K top-level categories). The method is specifically designed for fine-grained scenarios where inter-class confusion is concentrated among a few similar classes. On coarse tasks, standard KD suffices.

## Error Handling

- **Stage-PSC fails to converge:** Verify CLIP model compatibility with `open-clip-torch`. Ensure context vectors are initialized with small random values (std=0.02), not zeros. Check that the class name tokenization matches CLIP's vocabulary.
- **NLRD loss is NaN:** This occurs when softmax inputs are extremely large. Add numerical stability: clamp logit margins to [-50, 50] before softmax, and add epsilon (1e-8) to distributions before log computation in JS divergence.
- **Student underfits with NSD:** If accuracy drops below the baseline when adding NLRD, reduce λ_NSD to 0.1-0.3 and verify the teacher logits are producing meaningful rankings (teacher accuracy should be high).
- **GPU memory issues in Stage-NSD:** The NLRD loss iterates per-sample. Vectorize using `torch.gather` and batch operations instead of Python loops for production use. Batch size 128 on 4 GPUs (32 per GPU) is the recommended baseline.
- **Domain mismatch with CLIP:** If your fine-grained domain is far from CLIP's pretraining data (e.g., medical histology), PSC may not produce strong anchors. Consider using a domain-adapted CLIP (e.g., BiomedCLIP) as the teacher.

## Limitations

- **Requires a VLM teacher.** PAND is built around CLIP-style models. It cannot be applied to teacher-student pairs that lack a text encoder or vision-language alignment.
- **Two-stage training overhead.** The PSC stage adds ~200 epochs of CoOp training before the main distillation begins. For rapid experimentation, this overhead may be significant.
- **Fine-grained specific.** The NLRD loss is most effective when classes are visually similar and confusion is localized. On coarse classification tasks, the neighborhood structure provides minimal benefit over standard KD.
- **Fixed K across all samples.** Some samples may have more confusing neighbors than others. A per-sample adaptive K could improve results but is not implemented in the current framework.
- **Benchmark scope.** Published results cover CUB-200, Stanford Cars, FGVC-Aircraft, and Oxford Pets/Flowers. Performance on larger-scale fine-grained datasets (iNaturalist, Products-10K) is unvalidated.

## Reference

[PAND: Prompt-Aware Neighborhood Distillation for Lightweight Fine-Grained Visual Classification](https://arxiv.org/abs/2602.07768v1) — Luo et al., 2026. Focus on Section 3 (Method) for the PSC and NSD formulations, Table 1 for benchmark comparisons, and Table 2 for ablation results showing NSD's dominant contribution. Code: [github.com/LLLVTA/PAND](https://github.com/LLLVTA/PAND).