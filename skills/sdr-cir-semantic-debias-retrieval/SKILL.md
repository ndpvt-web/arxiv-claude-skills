---
name: "sdr-cir-semantic-debias-retrieval"
description: "Build training-free composed image retrieval systems that combine a reference image with modification text to find target images, using semantic debiasing to suppress reference-image noise. Use when: 'build a composed image retrieval system', 'find similar images with text modifications', 'debias image search results', 'zero-shot image retrieval with CLIP', 'retrieve images matching a reference plus text edit', 'semantic ranking with bias correction'."
---

# SDR-CIR: Semantic Debias Retrieval for Composed Image Retrieval

This skill enables Claude to implement training-free, zero-shot Composed Image Retrieval (CIR) systems based on the SDR-CIR framework. CIR takes a reference image and a modification text (e.g., "make the background snowy") and retrieves a target image from a database that matches the modified intent. The core technique uses Selective Chain-of-Thought prompting to generate target descriptions via an MLLM, then applies a two-step Anchor-Debias ranking formula that fuses reference image features while explicitly penalizing semantic bias inherited from the reference image. This eliminates the need for task-specific training while achieving state-of-the-art retrieval accuracy.

## When to Use

- When the user wants to build an image retrieval system that accepts both an image and text as a combined query (composed image retrieval)
- When implementing zero-shot or training-free image search that must generalize without fine-tuning on retrieval triplets
- When a retrieval system returns results too similar to the reference image, ignoring the modification text (semantic bias problem)
- When the user asks to re-rank CLIP-based image search results to better reflect user intent expressed in natural language
- When building fashion search, visual product discovery, or creative asset retrieval where users describe changes to a reference image
- When the user needs a debiasing or re-ranking layer on top of existing CLIP embeddings to improve retrieval quality

## Key Technique

**The Semantic Bias Problem.** Standard zero-shot CIR methods use an MLLM to describe the target image based on the reference image and modification text, then retrieve using CLIP text-image similarity. The problem: the generated description inherits visual details from the reference image that are irrelevant or contradictory to the modification. For example, if the reference shows a red car and the text says "change the color to blue," the description may still emphasize features of the red car, biasing retrieval toward the original rather than the target.

**Selective CoT + Anchor-Debias Ranking.** SDR-CIR solves this in two stages. First, Selective Chain-of-Thought prompting guides the MLLM to parse the modification text into explicit targets (directly stated changes) and implicit targets (unstated but logically required changes), then selectively extracts only the visual content relevant to those targets. This reduces noise at the source. Second, the Anchor step fuses the reference image embedding with the generated description embedding (`Fq = (1-alpha)*Fd + alpha*Fr`) to recover any visual cues the description missed. The Debias step then computes a penalty term representing the reference image's redundant semantic contribution (`Si = Sd - Sm`, where Sd is description-candidate similarity and Sm is modification-text-candidate similarity) and subtracts it from the final score: `Sf = (1+beta)*Sq - beta*Si`. This explicitly suppresses candidates that match the reference's visual noise rather than the intended modification.

**Why it works.** The penalty term `Si = Sd - Sm` isolates the portion of the description's similarity to each candidate that comes from the reference image rather than the modification text. Subtracting this penalizes candidates that are semantically close to the reference's unmodified features. The parameters alpha and beta are dataset-tunable but stable across a wide range (alpha: 0.05-0.2, beta: 0.35-0.5).

## Step-by-Step Workflow

1. **Encode the reference image.** Use a CLIP image encoder (ViT-L/14 or ViT-G/14 via OpenCLIP) to produce the reference image feature vector `Fr`. Normalize to unit length.

2. **Generate a target description via Selective CoT.** Prompt an MLLM (GPT-4, Qwen2.5-VL, or similar) with a four-stage Chain-of-Thought: (a) understand the reference image, (b) parse the modification text into explicit and implicit targets, (c) selectively extract only visual content relevant to those targets from the reference, (d) compose a natural-language description of the target image integrating the modifications.

3. **Encode the target description.** Use the CLIP text encoder (matching the image encoder's variant) to produce the description feature vector `Fd`. Normalize to unit length.

4. **Encode the modification text.** Use the same CLIP text encoder to produce the modification text feature vector `Fm`. Normalize to unit length.

5. **Compute the Anchored query feature.** Fuse reference and description features: `Fq = (1 - alpha) * Fd + alpha * Fr`. Use alpha=0.1 as a starting point (tune per domain: 0.05 for general images, 0.15-0.2 for fashion).

6. **Compute candidate similarities.** For every candidate image `It_i` in the database with pre-computed CLIP image feature `F(It_i)`, calculate three cosine similarities: `Sq = cos(Fq, F(It_i))`, `Sd = cos(Fd, F(It_i))`, `Sm = cos(Fm, F(It_i))`.

7. **Compute the debias penalty.** For each candidate: `Si = Sd - Sm`. This isolates the semantic contribution of the reference image to the description's match with each candidate.

8. **Compute the final debiased score.** `Sf = (1 + beta) * Sq - beta * Si`. Use beta=0.4 as a starting point (tune range: 0.3-0.5).

9. **Rank candidates by Sf descending.** Return the top-k results.

10. **Evaluate and tune.** Measure Recall@k on a validation set. Adjust alpha (controls how much reference visual context is preserved) and beta (controls debias strength) independently. Higher alpha helps when descriptions miss visual cues; higher beta helps when results are too similar to the reference.

## Concrete Examples

**Example 1: Fashion Product Retrieval**

```
User: I have a database of 10,000 fashion product images encoded with CLIP ViT-L/14.
A user uploads a photo of a red floral dress and types "make it blue with polka dots."
Build the retrieval pipeline.

Approach:
1. Encode the reference dress image with CLIP ViT-L/14 -> Fr
2. Prompt the MLLM with Selective CoT:
   - Stage 1 (Image Understanding): "A knee-length red dress with floral patterns,
     V-neckline, short sleeves"
   - Stage 2 (Modification Parsing):
     Explicit targets: color -> blue, pattern -> polka dots
     Implicit targets: silhouette, neckline, sleeve length remain unchanged
   - Stage 3 (Selective Extraction): Extract silhouette (knee-length, fitted waist),
     neckline (V-neck), sleeves (short) -- skip red color and floral pattern
   - Stage 4 (Target Description): "A knee-length blue dress with white polka dots,
     V-neckline, short sleeves, fitted waist"
3. Encode description -> Fd, encode "make it blue with polka dots" -> Fm
4. Anchor: Fq = 0.8 * Fd + 0.2 * Fr  (alpha=0.2 for fashion)
5. For each candidate, compute Sf = 1.4 * Sq - 0.4 * (Sd - Sm)
6. Return top-10 results ranked by Sf

Output: Blue polka-dot dresses with similar silhouettes ranked highest.
Without debiasing, red floral dresses would dominate results.
```

**Example 2: General Image Search with Scene Modification**

```
User: Given a photo of a park bench in autumn and the text "same bench but in
winter with snow," retrieve the best match from an image database.

Approach:
1. Encode park bench image -> Fr (CLIP ViT-G/14)
2. Selective CoT prompt to MLLM:
   - Parse explicit targets: season -> winter, ground/trees -> snow-covered
   - Parse implicit targets: bench style, bench position, camera angle preserved
   - Selective extraction: bench design (wooden slat bench, armrests, park setting)
   - Target description: "A wooden park bench with armrests in a snow-covered park,
     bare trees with snow on branches, overcast winter sky, same camera angle"
3. Encode description -> Fd, encode "same bench but in winter with snow" -> Fm
4. Anchor: Fq = 0.95 * Fd + 0.05 * Fr  (alpha=0.05 for general images)
5. Compute Sf = 1.5 * Sq - 0.5 * (Sd - Sm)  (beta=0.5)
6. Rank by Sf

Output: Winter park bench scenes ranked above autumn scenes, despite the
reference image being autumnal. The debias penalty suppresses candidates
matching autumn foliage features inherited from the reference.
```

**Example 3: Implementing the Selective CoT Prompt**

```
User: Show me how to write the Selective CoT prompt for the MLLM.

Prompt Template:
---
You are given a reference image and a modification text. Your task is to
describe the target image that results from applying the modification.

Step 1 - Reference Image Understanding:
Describe the visual content of the reference image in detail.

Step 2 - Modification Text Analysis:
Parse the modification text: "{modification_text}"
- Explicit targets: What attributes are directly stated to change?
- Implicit targets: What attributes must logically change as a consequence?
- Preserved attributes: What should remain unchanged?

Step 3 - Selective Visual Extraction:
From your Step 1 description, extract ONLY the visual details that are
relevant to the preserved attributes and implicit targets. Discard any
details that conflict with the explicit targets.

Step 4 - Target Description:
Compose a single paragraph describing the target image, integrating the
modifications with the preserved visual content. Be specific about colors,
shapes, textures, and spatial relationships.
---

Usage: Send this prompt + reference image to GPT-4 / Qwen2.5-VL / any
vision-language model. Use the Step 4 output as the target description
for CLIP text encoding.
```

## Best Practices

- **Do:** Pre-compute and cache CLIP image embeddings for all database candidates. The SDR-CIR scoring only requires three dot products per candidate at query time, making it efficient for large databases.
- **Do:** Use the same CLIP model variant for both image and text encoding. Mismatched encoders produce incomparable feature spaces.
- **Do:** Normalize all feature vectors to unit length before computing cosine similarity. Unnormalized vectors will break the debias penalty math.
- **Do:** Tune alpha and beta independently on a small validation set. Alpha controls anchor strength (reference image influence), beta controls debias aggressiveness. They affect different failure modes.
- **Avoid:** Setting alpha too high (>0.3). This over-anchors to the reference image and defeats the purpose of the modification text. Start low and increase only if descriptions consistently miss critical visual details.
- **Avoid:** Skipping the Selective CoT and feeding the MLLM a simple "describe the target image" prompt. Without explicit modification parsing, the description inherits far more reference-image bias, making the debias step less effective.
- **Avoid:** Using the debias formula without the anchor step. The anchor step (`Fq`) provides the base signal; the debias step corrects it. Using `Fd` alone as the query loses reference visual context that the description may have omitted.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| MLLM ignores modification text | Description matches reference image exactly | Strengthen Selective CoT prompt; add explicit instruction "You MUST incorporate these changes:" |
| Results too similar to reference | Top results look like the reference, not the target | Increase beta (debias strength) by 0.1 increments |
| Results ignore reference visual style | Top results match text but wrong visual style | Increase alpha (anchor weight) by 0.05 increments |
| CLIP text encoder truncates description | Long descriptions lose trailing detail | Keep target descriptions under 77 tokens; front-load critical modifications |
| Debias penalty produces negative scores | Candidates with very high Si dominate negatively | Clip Sf to a minimum of 0, or use softmax normalization on final scores |
| Modification text is vague | "Make it better" yields poor retrieval | Prompt the user for specific attributes; fall back to pure reference-image retrieval |

## Limitations

- **Requires a pre-existing image database with CLIP embeddings.** This is a retrieval method, not a generation method. It finds existing images; it does not create new ones.
- **CLIP feature space ceiling.** Retrieval quality is bounded by CLIP's ability to distinguish fine-grained visual attributes. Subtle texture or material changes may not be captured.
- **MLLM dependency for description generation.** The quality of Selective CoT output depends on the MLLM's vision capability. Weaker models produce noisier descriptions, increasing the burden on the debias step.
- **Two hyperparameters require tuning.** Alpha and beta are stable within ranges but optimal values vary by domain (fashion vs. general images vs. scene-level queries). A small labeled validation set is needed to tune them.
- **Single-stage retrieval only.** SDR-CIR is a one-stage method. For very large databases (>10M images), consider combining with approximate nearest neighbor search (FAISS/ScaNN) on the anchored query Fq, then applying the debias re-ranking to the top-N shortlist.
- **Does not handle spatial or structural modifications well.** Changes like "move the object to the left" are poorly captured by CLIP's global feature vectors.

## Reference

**Paper:** [SDR-CIR: Semantic Debias Retrieval Framework for Training-Free Zero-Shot Composed Image Retrieval](https://arxiv.org/abs/2602.04451v2) (WWW 2026)
**Key insight:** The debias formula `Sf = (1+beta)*Sq - beta*(Sd - Sm)` isolates and penalizes the reference image's spurious semantic contribution to retrieval scores, enabling training-free CIR that actually follows modification intent rather than returning reference-similar images.