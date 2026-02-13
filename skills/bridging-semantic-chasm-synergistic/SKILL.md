---
name: "bridging-semantic-chasm-synergistic"
description: |
  Build multi-agent systems for cross-modal alignment and OOD recognition using SynerNet-style
  synergistic agent architectures. Four specialized agents (visual, linguistic, embedding, coordinator)
  collaborate via structured message-passing to handle out-of-distribution inputs in vision-language tasks.
  Use when: "build a multi-agent VLM pipeline", "handle OOD detection with agents",
  "cross-modal alignment for new concepts", "few-shot recognition system with agent coordination",
  "fix VLM failures on unseen categories", "semantic context exchange between agents".
---

# Synergistic Multi-Agent Framework for Cross-Modal OOD Perception

This skill teaches Claude to design and implement multi-agent systems inspired by the SynerNet framework (arXiv:2602.00340), where four specialized computational agents — visual perception, linguistic context, nominal embedding, and global coordination — collaborate through structured message-passing to resolve cross-modal alignment failures in Vision-Language Models encountering out-of-distribution concepts. The core insight is that no single module can bridge the "semantic chasm" between vision and language for novel concepts; instead, specialized agents must exchange intermediate representations and adapt cooperatively, each compensating for the others' blind spots.

## When to Use

- When the user needs a multi-agent pipeline that handles **out-of-distribution inputs** in a vision-language system (e.g., classifying images of categories never seen during training)
- When building a **few-shot or zero-shot recognition** system where CLIP or similar VLMs degrade on novel concepts
- When designing **cross-modal alignment** systems where visual and textual encoders produce misaligned embeddings for rare or new categories
- When the user asks to implement **structured message-passing** between specialized processing agents (not generic chat agents, but computational units with state)
- When improving an existing VLM pipeline by adding **adaptive difficulty estimation** — routing hard samples through robust encoding paths
- When implementing **context exchange** data augmentation to expand training diversity for few-shot learning

## Key Technique

**The Cross-Modal Alignment Degeneration Problem.** Pre-trained VLMs like CLIP learn strong vision-language alignment on their training distribution, but this alignment degenerates for out-of-distribution concepts. A photo of a rare bird species gets mapped to a visual embedding that no longer sits close to its textual description. Single-model fine-tuning approaches (prompt tuning, adapter layers) cannot fully resolve this because the degeneration affects multiple processing stages simultaneously — visual encoding, text encoding, and the shared embedding space all need coordinated correction.

**The SynerNet Solution: Four Cooperative Agents.** Instead of monolithic fine-tuning, SynerNet decomposes the problem into four specialized agents that communicate via a structured message-propagation protocol `μ = (source, destination, payload)`. The **Visual Perception Unit** applies difficulty-adaptive encoding — standard encoding for easy samples, robust residual-stabilized encoding for hard ones, with a learned difficulty estimator selecting the path. The **Linguistic Context Unit** fuses textual and visual features so text encodings become visually grounded via a dual-layer context integration module `G_ctx`. The **Nominal Embedding Unit** learns dedicated vector representations for novel concepts and performs context-exchange augmentation — permuting semantic contexts across concept templates to force the model to distinguish essential features from contextual noise. The **Global Coordinator** governs the system with dynamic temperature scaling for contrastive loss and learned loss-weight balancing between contrastive and classification objectives.

**Why This Architecture Works.** The message-passing protocol creates bidirectional information flow: visual features inform text encoding (grounding), text features guide visual attention (semantic steering), and the coordinator dynamically adjusts the training signal based on aggregate system state. Ablation studies show the nominal embedding unit contributes the most (-4.1% on removal), followed by visual perception (-3.8%) and context exchange (-3.7%), confirming each agent's non-redundant role.

## Step-by-Step Workflow

1. **Define the four agent interfaces.** Create an abstract base class `Agent` with signature `(input, state) -> (output, new_state)`. Each agent maintains its own memory state (`S_k`) and produces typed outputs. Define message types as dataclasses: `Message(source: str, destination: str, payload: Tensor | dict)`.

2. **Implement the Visual Perception Unit.** Build two encoding paths: standard (`encode(x)`) and robust (`encode(x)/norm + beta * encode(x).detach()`). Add a difficulty estimator — a small MLP that takes batch-averaged features and outputs a sigmoid score. Route samples with `delta > threshold` through the robust path.

3. **Implement the Linguistic Context Unit.** Wrap the text encoder with a context integration module: a two-layer MLP `G_ctx` that takes concatenated `[text_embedding; visual_context]` and outputs a fused representation. Blend with original text encoding using a learnable `lambda` parameter: `output = lambda * text_enc + (1 - lambda) * G_ctx([text_enc; visual_ctx])`.

4. **Implement the Nominal Embedding Unit.** For each novel concept `c`, learn a set of embedding vectors `V_c = {v_1, ..., v_n}`. Generate prompts by filling templates like `"a photo of {V_c}, a type of {category}"`. Implement context-exchange augmentation: for each concept, also generate cross-template prompts using templates from other concepts to force invariance to contextual phrasing.

5. **Implement the Global Coordinator.** Create learnable parameters for: temperature `kappa` (clamped to [0.5, 2.0]), contrastive loss weight `w_con` (clamped to [0.5, 2.0]), and classification loss weight `w_cls` (clamped to [0.1, 1.0]). Normalize weights so they sum to 1. Compute total loss as `w_con * L_con + w_cls * L_cls`.

6. **Wire up the message-passing protocol.** Define a message bus or simple dispatcher. In each forward pass: (a) Visual Unit encodes images and sends features to Linguistic Unit and Coordinator; (b) Linguistic Unit receives visual context, produces grounded text embeddings, sends to Coordinator; (c) Nominal Embedding Unit generates concept prompts, sends to Linguistic Unit for encoding; (d) Coordinator computes loss with dynamic temperature and broadcasts gradient signals.

7. **Implement contrastive + classification training.** Use symmetric InfoNCE loss with the coordinator's learned temperature: `L_con = -(1/2N) * sum(log_softmax(similarity/kappa))` for both image-to-text and text-to-image directions. Add auxiliary classification loss on visual features with a linear head.

8. **Set up the training loop with AdamW and cosine annealing.** Freeze the backbone VLM (e.g., OpenCLIP). Only train: the difficulty estimator MLP, the context integration MLP, the nominal embeddings, and the coordinator's temperature/weight parameters. Grid-search learning rates in [1e-5, 1e-3]. Run 3 seeds for variance estimation.

9. **Evaluate in K-shot protocol.** For each K in {1, 2, 4, 8, 16}, sample K examples per novel class. Test on a composite set of both OOD and seen-distribution classes (generalized few-shot setting). Report harmonic mean of OOD and seen accuracy to capture the balance.

10. **Deploy with difficulty-adaptive inference.** At inference time, the difficulty estimator routes samples: easy samples use the fast standard path, hard samples use the robust path. The coordinator's learned temperature remains fixed. No message-passing overhead at inference — the trained parameters are baked into each module.

## Concrete Examples

**Example 1: Few-Shot Bird Species Classifier**

User: "I have a CLIP model that works well on common birds but fails on rare species from my field survey. I have 4 photos of each rare species. Build me a pipeline that adapts CLIP to recognize these new species without degrading performance on common ones."

Approach:
1. Freeze the CLIP backbone. Extract image features for all 4 shots of each rare species.
2. Initialize the Nominal Embedding Unit: learn 4 embedding vectors per rare species via `nn.Embedding`, generate prompts like `"a photo of [V_sparrow], a type of bird in forest habitat"`.
3. Enable context exchange: also generate `"a photo of [V_sparrow], a type of bird in wetland habitat"` using templates from other species to build invariance.
4. Train the Linguistic Context Unit's `G_ctx` MLP to fuse visual features from the 4-shot examples with text embeddings, producing visually-grounded text representations.
5. Set up the Global Coordinator with learned temperature (init=1.0) and symmetric contrastive loss over the 4-shot training pairs.
6. Train for 50 epochs with AdamW, lr=5e-4, cosine annealing.
7. At inference, compute similarity between new images and all class text embeddings (common + rare). The difficulty estimator routes ambiguous bird images through robust encoding.

Output:
```python
class SynerNetBirdClassifier:
    def __init__(self, clip_model, rare_species, k_shot=4):
        self.visual_unit = VisualPerceptionUnit(clip_model.visual, beta=0.1)
        self.linguistic_unit = LinguisticContextUnit(clip_model.text, hidden_dim=512)
        self.nominal_unit = NominalEmbeddingUnit(
            species=rare_species, n_vectors=4,
            templates=BIRD_TEMPLATES, cross_exchange=True
        )
        self.coordinator = GlobalCoordinator(
            kappa_init=1.0, w_con_init=1.0, w_cls_init=0.5
        )

    def forward(self, images, labels):
        # Visual agent encodes with difficulty routing
        vis_feats, difficulty = self.visual_unit(images)
        # Nominal agent generates concept prompts
        prompts = self.nominal_unit.generate(labels)
        # Linguistic agent fuses text + visual context
        text_feats = self.linguistic_unit(prompts, visual_ctx=vis_feats.mean(0))
        # Coordinator computes dynamic-temperature contrastive loss
        loss = self.coordinator.compute_loss(vis_feats, text_feats, labels)
        return loss
```

**Example 2: Zero-Shot Architectural Style Recognition**

User: "I need to classify building photos into architectural styles (Gothic, Brutalist, Deconstructivist, etc.) without any training examples. My CLIP model gives only 24% accuracy on uncommon styles."

Approach:
1. Use the Nominal Embedding Unit in zero-shot mode: instead of learned embeddings, generate rich descriptive prompts per style using templates that capture defining visual attributes.
2. For each style, create 8+ prompt variants via context exchange: `"a building in {style} style with {attribute_A}"`, `"a {style} structure featuring {attribute_B}"`.
3. Pass all prompts through the Linguistic Context Unit. Since there are no training images, use the text encoder's own attention features as a proxy for visual context (self-grounding).
4. Average the multiple prompt embeddings per style to create robust class prototypes.
5. At inference, encode the query image through the Visual Unit's robust path (unseen styles are inherently "hard"), then compute cosine similarity against all style prototypes.

Output:
```python
# Zero-shot setup: no training images needed
styles = ["Gothic", "Brutalist", "Deconstructivist", "Art Deco", ...]
templates = [
    "a building in {style} style with {attr}",
    "a {style} structure featuring {attr}",
    "architecture example of {style}, showing {attr}",
]
# Context exchange: cross attribute sets across styles
augmented_prompts = nominal_unit.cross_template_generate(styles, templates, attributes)
# Encode all prompts, average per style -> class prototypes
prototypes = linguistic_unit.encode_and_average(augmented_prompts, per_class=True)
# Inference: robust encoding for all queries (OOD assumption)
query_feat = visual_unit.encode_robust(query_image)
prediction = cosine_similarity(query_feat, prototypes).argmax()
```

**Example 3: Multi-Agent Code Architecture for Custom Domain**

User: "I want to build a SynerNet-style multi-agent system for matching product images to textual descriptions in an e-commerce catalog, handling new product categories."

Approach:
1. Define four agent classes inheriting from `BaseAgent(input, state) -> (output, state)`.
2. Implement message bus: `MessageBus.send(Message(src, dst, payload))` with typed routing.
3. Visual Unit: product image encoder with difficulty estimation (blurry/unusual product photos get robust encoding).
4. Linguistic Unit: product description encoder with context integration — fuse visual features from product images with text features from descriptions.
5. Nominal Unit: for each new product category, learn embeddings from the few available catalog entries; apply context exchange across category templates.
6. Coordinator: dynamic temperature contrastive learning + product matching loss.
7. Training: freeze backbone, train only the four agents' lightweight parameters.

Output architecture:
```
ProductImage -> [VisualUnit] --vis_feats--> [LinguisticUnit]
                     |                            |
                     |--difficulty--> [Coordinator]|--text_feats-->
                                          |
CatalogText -> [NominalUnit] --prompts--> [LinguisticUnit]
                                          |
                              [Coordinator] <-- all features
                                   |
                              total_loss (contrastive + matching)
```

## Best Practices

- **Do:** Freeze the pre-trained VLM backbone and only train the lightweight agent modules (MLPs, embeddings, temperature). This preserves the backbone's alignment on common concepts while adapting to OOD.
- **Do:** Always implement context exchange augmentation in the Nominal Embedding Unit — ablations show -3.7% without it. Permute templates across concepts, not just within.
- **Do:** Initialize the coordinator's temperature `kappa` at 1.0 and clamp to [0.5, 2.0]. Values outside this range destabilize contrastive learning.
- **Do:** Use the harmonic mean of OOD and seen-class accuracy as your primary metric in generalized few-shot settings. Raw OOD accuracy alone masks degradation on seen classes.
- **Avoid:** Training all four agents from scratch simultaneously. Pre-train the Visual and Linguistic units for a few epochs before enabling the full message-passing protocol — cold-start instability is real.
- **Avoid:** Using this architecture for tasks where the VLM already performs well. The overhead of four agents is only justified when cross-modal alignment has demonstrably degenerated (check: does CLIP accuracy on your OOD classes drop below 50%?).

## Error Handling

- **Difficulty estimator collapses** (all scores near 0 or 1): Add a regularization term `L_diff = -H(delta)` (entropy of difficulty scores) to encourage bimodal distribution. Check batch size is >= 16 for stable batch-averaged features.
- **Context integration module diverges**: Reduce learning rate for `G_ctx` by 10x relative to other parameters. Ensure `lambda` is initialized at 0.7 (favoring original text encoding).
- **Nominal embeddings overfit in 1-shot**: Use stronger context exchange (more template permutations) and add L2 regularization on `V_c` vectors. Consider initializing from the text encoder's word embeddings for the category name.
- **Coordinator temperature drifts to boundary**: If `kappa` hits 0.5 or 2.0 repeatedly, the contrastive loss gradient signal is too strong or too weak. Adjust learning rate for temperature or check for label noise.
- **Message-passing memory overhead**: For large batch sizes, accumulate messages across micro-batches rather than storing all pairwise features. Visual context for the Linguistic Unit can use a running mean instead of full batch features.

## Limitations

- **Backbone-bounded:** Performance is upper-bounded by the pre-trained VLM's representation quality. If the backbone has never seen anything remotely similar to your OOD domain (e.g., medical imaging on a web-trained CLIP), SynerNet's agents cannot compensate for fundamentally missing visual primitives.
- **Object-level only:** The framework handles object-level recognition (bird species, building styles, product categories) but struggles with fine-grained attribute disentanglement (e.g., distinguishing between two nearly identical fabric textures).
- **Computational overhead:** Four agents with message-passing add ~30-40% training overhead versus single-model fine-tuning. At inference, the difficulty estimator adds marginal cost but the rest collapses to standard forward passes.
- **Minimum 1-shot required for full benefit:** While zero-shot mode works via prompt engineering alone, the largest gains (3-5%) require at least 4-shot examples to train the nominal embeddings and context integration module.
- **VISTA-Beyond specific validation:** Published results are on VISTA-Beyond benchmark. Transfer to other OOD benchmarks (e.g., DomainNet, Office-Home) is plausible but unvalidated by the original paper.

## Reference

**Paper:** [Bridging the Semantic Chasm: Synergistic Conceptual Anchoring for Generalized Few-Shot and Zero-Shot OOD Perception](https://arxiv.org/abs/2602.00340v1) — Christoforos et al., 2026. Look for: Table 1 (few-shot results by K-shot), Table 3 (ablation showing each agent's contribution), and Section 3.2 (the message-propagation protocol formalization).