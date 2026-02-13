---
name: "fine-r1-make-multi-modal-excel"
description: >
  Implement Fine-R1's structured chain-of-thought reasoning pipeline for fine-grained visual recognition (FGVR) tasks.
  Builds multi-stage reasoning systems that decompose visual classification into Visual Analysis, Candidate Proposal,
  Comparative Discrimination, and Final Prediction — enabling MLLMs to distinguish visually similar sub-categories
  (bird species, car models, aircraft variants, flower types) with minimal training data.
  Trigger phrases: "fine-grained visual recognition", "FGVR pipeline", "visual chain-of-thought",
  "distinguish similar categories", "species identification system", "Fine-R1 reasoning",
  "discriminative visual attributes", "few-shot visual classification"
---

# Fine-R1: Structured Chain-of-Thought for Fine-Grained Visual Recognition

This skill enables Claude to build systems that apply Fine-R1's four-stage chain-of-thought reasoning framework to fine-grained visual recognition tasks. Instead of relying on end-to-end classification, the approach decomposes visual identification into structured reasoning stages — visual analysis, candidate proposal, inter-class comparison, and final prediction — dramatically improving accuracy on tasks where sub-categories share high visual similarity (e.g., distinguishing a Laysan Albatross from a Black-footed Albatross). The method achieves state-of-the-art results with as few as 4 training examples per category.

## When to Use

- When the user needs to build a system that classifies images into fine-grained sub-categories (bird species, car models, aircraft types, dog breeds, flower varieties, etc.)
- When the user wants to add explainable reasoning to a visual classification pipeline rather than opaque softmax outputs
- When the user asks to fine-tune an MLLM (Qwen2.5-VL, LLaVA, InternVL) for domain-specific visual recognition with limited labeled data
- When the user needs a few-shot visual classification system that generalizes to unseen sub-categories
- When the user wants to implement reinforcement learning (DAPO/GRPO) on top of a vision-language model for discriminative tasks
- When the user asks to build a triplet-based training pipeline that exploits intra-class and inter-class visual differences

## Key Technique

**Structured CoT for Visual Discrimination.** Fine-R1's core insight is that MLLMs already encode fine-grained sub-category knowledge but fail to *deploy* it effectively. The fix is not better embeddings but better reasoning. The method structures MLLM output into four explicit stages: (1) **Visual Analysis** — describe discriminative attributes visible in the image (plumage pattern, wing shape, bill color); (2) **Candidate Sub-categories** — propose up to four plausible categories based on the analysis; (3) **Comparison** — systematically compare candidates against observed attributes, eliminating mismatches; (4) **Final Prediction** — output the selected sub-category with confidence. This decomposition forces the model to reason through discriminative features rather than pattern-match.

**Two-Stage Training: CoT SFT + TAPO.** Stage 1 (Cold Start) constructs a small, high-quality dataset (~404 samples) of structured reasoning chains using a stronger teacher model, then fine-tunes the target MLLM via supervised learning. Stage 2 (Triplet Augmented Policy Optimization) applies reinforcement learning with two augmentation strategies: *intra-class augmentation* mixes rollout trajectories from images of the same category to build robustness against within-class variance, while *inter-class augmentation* maximizes KL divergence between output distributions conditioned on similar vs. different categories, sharpening the model's discriminative boundaries.

**Information Bottleneck for Attribute Extraction.** To generate training data, the method uses an information bottleneck strategy: generate multiple diverse descriptions of each image emphasizing different attributes, then filter to retain only features that are discriminative for the target taxonomy. This avoids the common failure mode where CoT reasoning fixates on irrelevant visual details.

## Step-by-Step Workflow

1. **Define the taxonomy and gather reference images.** Specify the hierarchical category structure (e.g., Bird > Warbler > Yellow Warbler). Collect at minimum 4 reference images per sub-category — these serve as the few-shot training set.

2. **Build the structured CoT prompt template.** Create a system prompt that instructs the MLLM to respond in four labeled sections:
   ```
   <Visual Analysis> Describe the key discriminative features visible in this image...
   <Candidate Sub-categories> Based on the analysis, list up to 4 possible sub-categories...
   <Comparison> For each candidate, compare expected attributes against observed features...
   <Final Prediction> The image shows: [sub-category name]
   ```

3. **Generate seed reasoning chains with a teacher model.** Use a stronger MLLM (e.g., Qwen2.5-VL-32B or GPT-4o) to produce structured CoT responses for each reference image. Provide the ground-truth label and the CoT template. Filter outputs to retain only chains where the final prediction is correct and the reasoning references genuinely discriminative attributes.

4. **Apply information bottleneck filtering.** For each generated chain, verify that the cited visual attributes are (a) actually visible in the image and (b) discriminative within the taxonomy. Remove chains that cite genus-level features common to all candidates (e.g., "has feathers" for bird classification) or hallucinate attributes not present in the image.

5. **Run CoT Supervised Fine-Tuning (Stage 1).** Fine-tune the target MLLM on the curated reasoning chains. Use full-parameter or LoRA fine-tuning for 5-10 epochs. This establishes the model's ability to produce structured four-stage reasoning output.

6. **Construct triplet training sets for TAPO (Stage 2).** For each anchor image, pair it with a positive (same sub-category, different image) and a negative (visually similar but different sub-category). Ensure negatives are hard negatives — categories that share the most visual overlap with the anchor.

7. **Implement Triplet Augmented Policy Optimization.** Using DAPO (or GRPO) as the base RL algorithm:
   - Sample rollout trajectories for anchor and positive images.
   - **Intra-class augmentation**: Mix trajectories from anchor and positive into a shared batch, training the policy to be invariant to within-class variation.
   - **Inter-class augmentation**: Compute KL divergence between output distributions for anchor vs. negative, adding a reward bonus proportional to this divergence to encourage discriminative outputs.

8. **Evaluate on seen and unseen categories.** Split categories 60/40 into seen (used in training) and unseen (held out). Report accuracy on both splits. Fine-R1 should show strong generalization to unseen categories — if unseen accuracy drops sharply, increase the diversity of reasoning chains in Step 3.

9. **Deploy with structured output parsing.** In production, parse the model's four-stage output to extract the final prediction programmatically. Optionally surface the Comparison stage to users as an explanation for the classification decision.

10. **Iterate on failure cases.** Examine misclassifications where the Visual Analysis stage correctly identifies discriminative features but the Comparison stage draws wrong conclusions. These indicate the model needs more contrastive examples between the confused sub-categories in the TAPO training set.

## Concrete Examples

**Example 1: Building a Bird Species Identifier**

User: "I want to build a bird species classifier that can distinguish between 200 CUB species using Qwen2.5-VL-7B with minimal training data."

Approach:
1. Download CUB-200-2011 dataset. Sample 4 images per species for training.
2. Create the CoT prompt template with bird-specific attribute vocabulary (bill shape, plumage color, wing bars, tail shape, habitat cues).
3. Generate reasoning chains using Qwen2.5-VL-32B as teacher:
   ```
   <Visual Analysis> The bird has a bright yellow body with black wings
   featuring white wing bars. The bill is short and conical, typical of
   seed-eating passerines. Black streaking is visible on the flanks.
   <Candidate Sub-categories> 1. Yellow Warbler 2. American Goldfinch
   3. Evening Grosbeak 4. Wilson's Warbler
   <Comparison> Yellow Warbler: expects reddish breast streaks — observed
   black flank streaks, partial match. American Goldfinch: expects black
   cap and wings with white wing bars — matches wing pattern but no black
   cap visible, likely female/non-breeding. Evening Grosbeak: expects
   massive bill — bill here is small, eliminated. Wilson's Warbler:
   expects olive-green back — back is yellow, eliminated.
   <Final Prediction> American Goldfinch (female/non-breeding plumage)
   ```
4. Fine-tune Qwen2.5-VL-7B on 404 curated chains (5 epochs, lr=1e-5).
5. Run TAPO with goldfinch vs. warbler as hard negative pairs.
6. Evaluate: expect ~91% on seen species, ~85% on unseen species.

Output: A fine-tuned model that outputs structured reasoning explaining why it chose each species, not just a label.

**Example 2: Implementing the TAPO Training Loop**

User: "Show me how to implement the triplet augmented policy optimization from Fine-R1."

Approach:
1. Start from a DAPO/GRPO implementation (e.g., OpenRLHF or TRL library).
2. Modify the rollout sampler to generate trajectories for triplets:

```python
def sample_triplet_rollouts(model, anchor_img, positive_img, negative_img, prompt_template, n_samples=8):
    """Generate rollout trajectories for a triplet of images."""
    anchor_trajectories = [model.generate(prompt_template.format(image=anchor_img)) for _ in range(n_samples)]
    positive_trajectories = [model.generate(prompt_template.format(image=positive_img)) for _ in range(n_samples)]
    negative_trajectories = [model.generate(prompt_template.format(image=negative_img)) for _ in range(n_samples)]
    return anchor_trajectories, positive_trajectories, negative_trajectories

def compute_tapo_reward(anchor_trajs, pos_trajs, neg_trajs, ground_truth_label):
    """Compute reward with intra-class and inter-class augmentation."""
    rewards = []
    for traj in anchor_trajs + pos_trajs:
        # Base reward: correctness of final prediction
        prediction = extract_final_prediction(traj)
        base_reward = 1.0 if prediction == ground_truth_label else 0.0

        # Format reward: structured CoT present and well-formed
        format_reward = 0.5 if has_valid_cot_structure(traj) else 0.0

        rewards.append(base_reward + format_reward)

    # Inter-class augmentation: bonus for divergence from negative
    anchor_dist = compute_output_distribution(anchor_trajs)
    neg_dist = compute_output_distribution(neg_trajs)
    kl_bonus = kl_divergence(anchor_dist, neg_dist) * 0.1

    rewards = [r + kl_bonus for r in rewards]
    return rewards
```

3. The intra-class augmentation is achieved by mixing anchor and positive trajectories into the same training batch — the policy gradient treats them as samples from the same category.
4. Train for 1-2 epochs with lr=5e-7, clip ratio 0.2.

**Example 3: Few-Shot Domain Adaptation for Industrial Parts**

User: "I need to classify 50 types of electronic capacitors from inspection camera images. I only have ~10 labeled images per type."

Approach:
1. Define the capacitor taxonomy with discriminative attributes: body shape (cylindrical, rectangular, disc), terminal type (radial, axial, SMD), color bands, size markings, manufacturer stamps.
2. Generate CoT chains using a teacher model, emphasizing attributes that differentiate capacitor types (e.g., "ceramic disc vs. MLCC" requires noting body thickness and terminal geometry).
3. Fine-tune on the 50 * 4 = 200 best chains (SFT stage).
4. Build triplets using confusable pairs (e.g., 0805 vs. 0603 SMD capacitors differing only in size).
5. Run TAPO to sharpen the model's sensitivity to subtle dimensional and marking differences.
6. Deploy with the structured output — the Comparison stage serves double duty as a quality inspection explanation.

Output: A classifier achieving high accuracy on capacitor subtypes with built-in explainability for QA review.

## Best Practices

- **Do:** Use hard negatives in triplet construction — pair categories that humans also confuse. The discriminative power of TAPO comes from contrasting truly similar sub-categories.
- **Do:** Verify that generated reasoning chains cite attributes actually visible in the image. Hallucinated attributes in training data teach the model to fabricate evidence.
- **Do:** Keep the CoT structure rigid (exactly four labeled sections). The model learns the format as a reasoning scaffold — deviations degrade performance.
- **Do:** Start with CoT SFT before TAPO. Skipping Stage 1 means the RL optimization has no structured reasoning baseline to improve upon.
- **Avoid:** Using more than 4 candidate sub-categories in the Comparison stage. The paper found that 4 candidates balances coverage against reasoning depth.
- **Avoid:** Training on easy examples where the sub-category is visually obvious. The method's value is in resolving ambiguous cases — bias training toward hard examples.
- **Avoid:** Fine-tuning on thousands of samples. Fine-R1 specifically shows that 4-shot per category (~400 total samples) outperforms larger datasets because quality and structure matter more than volume.

## Error Handling

- **Model outputs unstructured text instead of four-stage CoT:** The SFT stage was insufficient. Increase SFT epochs (up to 10) or add more format-diverse training examples. Apply a format reward during TAPO that penalizes missing sections.
- **High accuracy on seen categories but poor generalization to unseen:** The model is memorizing category-specific patterns rather than learning discriminative reasoning. Increase inter-class augmentation weight in TAPO and ensure training categories are taxonomically diverse.
- **Reasoning chain cites correct attributes but draws wrong conclusion:** This indicates the Comparison stage needs stronger contrastive signal. Add more hard-negative triplets specifically for the confused category pairs.
- **Teacher model generates incorrect reasoning chains:** Filter aggressively — only keep chains where the final prediction matches ground truth AND the cited attributes are verifiable. A 50% rejection rate is normal.
- **TAPO training diverges or collapses:** Reduce learning rate (try 1e-7), increase clip ratio, or reduce the KL divergence bonus weight. DAPO is sensitive to hyperparameters on small datasets.

## Limitations

- Requires a stronger teacher model (32B+ parameters) to generate seed reasoning chains — if the teacher itself cannot distinguish the sub-categories, the pipeline fails at Step 3.
- The four-stage CoT format adds latency at inference time (3-5x longer generation than direct classification). Not suitable for real-time applications requiring <100ms responses.
- Performance depends on sub-categories being visually distinguishable in principle. If two categories differ only by geographic range or genetic markers not visible in images, structured reasoning cannot help.
- The TAPO stage requires careful triplet mining. Randomly selected negatives provide weak learning signal — you need domain knowledge to identify which category pairs are confusable.
- Currently validated on natural image domains (birds, cars, dogs, flowers, aircraft, pets). Transfer to medical imaging, satellite imagery, or microscopy may require domain-specific attribute vocabularies.

## Reference

**Paper:** [Fine-R1: Make Multi-modal LLMs Excel in Fine-Grained Visual Recognition by Chain-of-Thought Reasoning](https://arxiv.org/abs/2602.07605v2) — He, Geng, Peng (2026). Look for: Section 3 (structured CoT template), Section 4 (TAPO algorithm with intra/inter-class augmentation), and Appendix B (prompt templates) for implementation-critical details.