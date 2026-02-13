---
name: "spd-faith-bench-diagnosing-improving"
description: "Diagnose and improve faithfulness of chain-of-thought reasoning in multimodal LLM pipelines using the SPD-Faith Bench methodology and SAGE framework. Detects perceptual blindness, perception-reasoning dissociation, and post-hoc rationalization in vision-language systems. Trigger phrases: 'check if my model reasoning is faithful', 'diagnose CoT faithfulness', 'improve visual reasoning faithfulness', 'detect hallucinated reasoning', 'audit multimodal chain-of-thought', 'fix perception-reasoning gap'"
---

# SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal LLMs

This skill enables Claude to diagnose whether a multimodal LLM's chain-of-thought (CoT) reasoning actually reflects its visual perception, and to apply the train-free SAGE intervention framework to improve faithfulness. Based on the SPD-Faith Bench paper, the core insight is that models frequently "see" the right visual evidence but then generate reasoning that contradicts or ignores it -- a failure mode called perception-reasoning dissociation. This skill provides concrete metrics, diagnostic patterns, and a three-stage inference-time fix (See-Think-Generate) that requires no retraining.

## When to Use

- When a user is building or evaluating a vision-language pipeline and wants to verify that the model's CoT actually reflects what it perceives in the image, not just linguistic priors
- When debugging a multimodal system where the final answer is sometimes correct but the intermediate reasoning steps reference non-existent or incorrect visual features
- When a user reports that their MLLM gives contradictory answers to semantically equivalent questions (e.g., "Are these the same?" vs. "Are these different?" on the same image pair)
- When implementing inference-time interventions to steer a multimodal model toward more visually grounded reasoning without retraining
- When designing evaluation benchmarks for multimodal faithfulness that go beyond simple accuracy metrics
- When auditing a deployed MLLM for post-hoc rationalization -- generating plausible-sounding but fabricated justifications

## Key Technique

**The Problem: Faithfulness vs. Correctness.** Standard evaluation checks whether a model's final answer is correct. But a model can produce the right answer for the wrong reasons -- guessing, relying on language priors, or generating a reasoning trace that sounds plausible but does not correspond to actual visual processing. SPD-Faith Bench isolates this by using "Spot the Difference" image pairs where the model must explicitly compare two nearly identical images, making it impossible to answer from language alone. The benchmark measures three hierarchical levels: global perception (did it detect a change?), faithful perception (did it correctly identify the category and type of change?), and faithful reasoning (does its CoT semantically align with the actual visual evidence without internal contradictions?).

**Two Systematic Failure Modes.** The paper identifies (1) *perceptual blindness* -- attention maps show negligible activation on relevant image regions, causing random guessing, and (2) *perception-reasoning dissociation* -- the model attends to the correct regions but then generates explanations that conflict with what it perceived. Root cause analysis reveals that visual attention decays through the network's layers (initial suppression by system prompt tokens, then progressive decay), and that FFN layers in deeper blocks override visual context with parametric priors stored in weights.

**SAGE: A Train-Free Fix.** The SAGE framework (See-Think-Generate) intervenes at inference time across three stages. *See* amplifies visual token attention in shallow layers and adaptively restores it in deep layers. *Think* monitors the divergence between the multi-head attention output (which carries visual signal) and the FFN output (which carries parametric priors), dynamically suppressing FFN contribution when it overrides visual context. *Generate* constructs a visual evidence mask from attention maps and uses contrastive decoding -- comparing logits from the full input versus a masked input -- to boost tokens that depend on actual visual evidence. This produces CoT that is demonstrably grounded in perception rather than confabulated.

## Step-by-Step Workflow

1. **Define the faithfulness evaluation scope.** Identify the specific multimodal task (image comparison, VQA, visual reasoning) and determine which faithfulness dimensions matter: global perception, faithful perception, or faithful reasoning. For most audits, all three are needed.

2. **Construct diagnostic probes using paired or symmetric queries.** Create test cases where the same visual content is queried in semantically equivalent but differently phrased ways (e.g., "Is there a difference?" and "Are these identical?"). Contradictory responses to symmetric queries indicate unfaithful reasoning. Calculate the Contradiction Rate (CR): the fraction of cases where the model gives inconsistent binary judgments.

3. **Measure global perception metrics.** Compute Difference Sensitivity (DS) -- binary accuracy on detecting whether a change exists -- and Difference Quantity Recall (DQR) -- accuracy of counting how many differences are present. These establish whether the model perceives changes at all before evaluating reasoning quality.

4. **Evaluate faithful perception with category-level and type-level F1.** Extract the model's claimed change categories (e.g., "object removed", "color changed", "position shifted") from its CoT and compare against ground truth. Category-Level F1 (CF1) measures coarse-grained accuracy; Type-Level F1 (TF1) measures fine-grained accuracy (e.g., "red car removed" not just "object removed").

5. **Score reasoning faithfulness via two-phase DRF evaluation.** First, perform global content matching: does the generated CoT mention the actual regions and changes present in the image? Second, categorize errors: are unfaithful statements fabrications (mentioning non-existent changes), omissions (missing real changes), or contradictions (internally inconsistent claims)? The Difference Reasoning Faithfulness (DRF) score is the semantic alignment between CoT content and visual ground truth.

6. **Diagnose the failure mode using attention analysis.** If available, extract attention maps from the model's vision-language layers. For perceptual blindness: attention in relevant image regions will be near zero. For perception-reasoning dissociation: attention will be high in correct regions but the generated text will not match. If attention maps are unavailable, use the symmetric query contradiction pattern as a proxy -- high CR with high DS strongly suggests dissociation.

7. **Apply SAGE Stage 1 (See): Dynamic Visual Routing.** In shallow layers (before layer `l_s`, typically the first 30-40% of layers), scale visual token attention by factor `(1 + alpha)` where `alpha = 0.1`. In deeper layers, compute the attention decay rate per layer and apply adaptive amplification inversely proportional to decay. This prevents visual signal from being drowned out by text tokens.

8. **Apply SAGE Stage 2 (Think): Information Flow Rectification.** At each transformer block in the mid-to-deep layers, compute the KL divergence between the residual stream after the MHA sublayer and after the FFN sublayer. When divergence exceeds a threshold, scale down the FFN contribution by coefficient `beta = 0.9`. This prevents parametric priors from overwriting visual context.

9. **Apply SAGE Stage 3 (Generate): Visual-Anchored Contrastive Decoding.** Construct a binary visual evidence mask from the intersection of top-k attention regions and token activation regions. Run two forward passes: the main path with the original input and an auxiliary path with the masked (evidence-removed) input. Compute final logits as `L_final = L_main + eta * ReLU(L_main - L_aux)` where `eta = 0.5`. This amplifies tokens whose generation depends on actual visual evidence.

10. **Re-evaluate and iterate.** Run the same diagnostic probes from steps 2-5 after applying SAGE. Compare CR, CF1, TF1, and DRF before and after intervention. A successful intervention should decrease CR and increase DRF without significantly degrading task accuracy.

## Concrete Examples

**Example 1: Diagnosing Contradiction in a Visual QA System**

User: "My multimodal model gives contradictory answers when I ask the same question in different ways about image differences. How do I measure and fix this?"

Approach:
1. Create symmetric query pairs for each test image:
   - Query A: "Are these two images the same?"
   - Query B: "Is there any difference between these images?"
2. Run both queries on each image pair, collecting binary judgments.
3. Compute Contradiction Rate:
   ```python
   contradictions = 0
   total = 0
   for pair in image_pairs:
       answer_a = model.query(pair, "Are these two images the same?")
       answer_b = model.query(pair, "Is there any difference between these images?")
       binary_a = parse_binary(answer_a)  # True = same
       binary_b = parse_binary(answer_b)  # True = different
       if binary_a == binary_b:  # Both say "same" or both say "different" => contradiction
           contradictions += 1
       total += 1
   CR = contradictions / total
   ```
4. Stratify by difficulty (number of objects in scene). Expect CR to rise with visual complexity.
5. If CR > 15%, the model has systematic faithfulness issues.

Output:
```
Faithfulness Diagnosis Report
=============================
Total image pairs evaluated: 500
Contradiction Rate (CR): 24.6%
  - Easy scenes (< 5 objects):  12.3%
  - Medium scenes (5-15):       26.1%
  - Hard scenes (> 15):         35.4%

Diagnosis: Perception-reasoning dissociation detected.
CR rises sharply with scene complexity, indicating the model
perceives changes but generates inconsistent rationalizations.
Recommended: Apply SAGE framework (See-Think-Generate) at inference.
```

**Example 2: Implementing the SAGE Contrastive Decoding Stage**

User: "I want to implement the visual-anchored contrastive decoding from SAGE for my LLaVA-based model."

Approach:
1. Extract attention maps from the vision-language cross-attention layers.
2. Compute a visual evidence mask from top-k attended regions.
3. Run contrastive decoding with main and auxiliary (masked) paths.

```python
import torch
import torch.nn.functional as F

def sage_contrastive_decode(model, input_ids, pixel_values, attention_mask,
                            eta=0.5, top_k_ratio=0.3):
    """
    SAGE Stage 3: Visual-Anchored Contrastive Decoding.
    Amplifies tokens whose generation depends on actual visual evidence.
    """
    # Main path: full visual input
    with torch.no_grad():
        main_outputs = model(
            input_ids=input_ids,
            pixel_values=pixel_values,
            attention_mask=attention_mask,
            output_attentions=True
        )
        logits_main = main_outputs.logits[:, -1, :]

        # Extract cross-attention to visual tokens from last layer
        cross_attn = main_outputs.attentions[-1]  # [B, H, seq, seq]
        visual_token_range = model.get_visual_token_range()
        visual_attn = cross_attn[:, :, -1, visual_token_range].mean(dim=1)

        # Build evidence mask: zero out top-k attended visual regions
        k = int(top_k_ratio * visual_attn.shape[-1])
        topk_indices = visual_attn.topk(k, dim=-1).indices
        masked_pixel_values = pixel_values.clone()
        # Map attention indices back to pixel space and mask
        mask = build_spatial_mask(topk_indices, pixel_values.shape)
        masked_pixel_values = masked_pixel_values * (1 - mask)

    # Auxiliary path: masked visual input (evidence removed)
    with torch.no_grad():
        aux_outputs = model(
            input_ids=input_ids,
            pixel_values=masked_pixel_values,
            attention_mask=attention_mask
        )
        logits_aux = aux_outputs.logits[:, -1, :]

    # Contrastive decoding: amplify evidence-dependent tokens
    diff = logits_main - logits_aux
    logits_final = logits_main + eta * F.relu(diff)

    return logits_final
```

Output: Tokens that the model generates only because of visual evidence get amplified. Tokens the model would generate regardless (from language priors) are unchanged. This steers CoT toward visually grounded statements.

**Example 3: Building a Faithfulness Evaluation Pipeline**

User: "I need a comprehensive evaluation of my multimodal model's reasoning faithfulness, not just accuracy."

Approach:
1. Prepare a test set with ground-truth annotations for each image pair (change categories, types, locations).
2. Run the model and extract structured CoT.
3. Compute the full metric hierarchy.

```python
def evaluate_faithfulness(model, test_set):
    results = {"DS": [], "DQR": [], "CF1": [], "TF1": [], "CR": [], "DRF": []}

    for sample in test_set:
        # Global Perception
        pred_has_diff = model.detect_difference(sample.img_a, sample.img_b)
        results["DS"].append(pred_has_diff == sample.has_difference)

        pred_count = model.count_differences(sample.img_a, sample.img_b)
        results["DQR"].append(pred_count == sample.num_differences)

        # Faithful Perception
        pred_categories = model.extract_categories_from_cot(sample)
        cf1 = f1_score(sample.gt_categories, pred_categories, level="category")
        tf1 = f1_score(sample.gt_categories, pred_categories, level="type")
        results["CF1"].append(cf1)
        results["TF1"].append(tf1)

        # Faithful Reasoning
        cr = compute_contradiction_rate(model, sample)
        results["CR"].append(cr)

        drf = compute_drf(
            generated_cot=model.get_cot(sample),
            ground_truth=sample.gt_reasoning,
            phase1="global_content_match",
            phase2="error_categorization"  # fabrication, omission, contradiction
        )
        results["DRF"].append(drf)

    return {k: sum(v)/len(v) for k, v in results.items()}
```

Output:
```
Metric        | Score  | Interpretation
DS            | 0.89   | Good change detection
DQR           | 0.62   | Undercounts differences
CF1           | 0.71   | Moderate category accuracy
TF1           | 0.54   | Weak fine-grained grounding
CR            | 0.22   | 22% contradictions (high)
DRF           | 0.58   | 42% of reasoning is ungrounded

Gap Analysis: DS(0.89) >> DRF(0.58) indicates classic
perception-reasoning dissociation. The model perceives
changes but fabricates or contradicts in its reasoning.
```

## Best Practices

- **Do:** Always measure faithfulness separately from accuracy. A model can be 85% accurate but only 55% faithful -- the gap reveals systematic confabulation.
- **Do:** Use symmetric query pairs as a cheap, model-agnostic faithfulness probe. Contradiction Rate requires no ground truth annotations and catches dissociation reliably.
- **Do:** Stratify evaluation by visual complexity (object density, change subtlety). Faithfulness degrades predictably with difficulty, and this gradient reveals whether failures are perceptual or reasoning-level.
- **Do:** When applying SAGE, tune `alpha`, `beta`, and `eta` on a small validation set. The paper defaults (0.1, 0.9, 0.5) are good starting points but architecture-dependent.
- **Avoid:** Treating correct final answers as evidence of faithful reasoning. The paper shows models with 90%+ detection accuracy can have 40%+ unfaithful reasoning traces.
- **Avoid:** Applying SAGE uniformly across all layers. The shallow-layer amplification and deep-layer rectification target different failure mechanisms and must be applied to the correct layer ranges for the specific architecture.

## Error Handling

- **Attention maps unavailable:** Many API-based models (GPT-4o, Claude) do not expose attention weights. Fall back to the symmetric query contradiction method and DRF scoring via an external judge model (e.g., use a second LLM to compare CoT against visual ground truth).
- **CoT extraction fails:** If the model does not produce structured reasoning steps, prompt it explicitly with "Describe step by step what differences you observe" before evaluation. If free-form, use NLI-based matching rather than exact category extraction.
- **SAGE causes accuracy degradation:** If contrastive decoding reduces final answer accuracy, lower `eta` (try 0.2-0.3) or restrict contrastive decoding to reasoning tokens only, not the final answer token.
- **High CR but low DS:** This indicates perceptual blindness, not dissociation. SAGE Stage 1 (visual routing) is the priority fix; Stages 2-3 will not help if the model never attends to the relevant regions.

## Limitations

- SAGE requires access to model internals (attention maps, residual streams, per-layer activations). It cannot be applied to black-box API models without architectural modifications.
- The SPD-Faith Bench methodology is designed around image-pair comparison tasks. Extending it to single-image reasoning, video understanding, or non-visual modalities requires adapting the probe construction.
- Contradiction Rate is a necessary but not sufficient indicator of unfaithfulness. A model can be consistently wrong without contradicting itself. Always combine CR with DRF for a complete picture.
- The three change types in the original benchmark (color, removal, position) do not cover all visual reasoning scenarios. For domain-specific applications (medical imaging, satellite analysis), construct probes with domain-relevant change categories.
- Contrastive decoding adds latency (two forward passes per generation step). For latency-sensitive deployments, consider applying it only during evaluation or on flagged low-confidence outputs.

## Reference

[SPD-Faith Bench: Diagnosing and Improving Faithfulness in Chain-of-Thought for Multimodal Large Language Models](https://arxiv.org/abs/2602.07833v1) -- Introduces a three-level faithfulness evaluation hierarchy (perception, faithful perception, faithful reasoning) and the train-free SAGE framework for inference-time visual grounding intervention. Focus on Section 4 (SAGE methodology), Table 2 (contradiction rate analysis), and Table 3 (the "seeing but lying" gap between DS and DRF).