---
name: "bridging-lexical-ambiguity-vision"
description: "Build Visual Word Sense Disambiguation (VWSD) systems that resolve lexical ambiguity using CLIP, diffusion models, and LLM reasoning. Use when: 'disambiguate this word visually', 'pick the right image for this word sense', 'build a VWSD pipeline', 'resolve ambiguous words with images', 'rank candidate images by word meaning', 'multimodal word sense disambiguation'."
---

# Visual Word Sense Disambiguation (VWSD) Pipeline Builder

This skill enables Claude to design and implement Visual Word Sense Disambiguation systems — pipelines that resolve lexical ambiguity by selecting or ranking images that match the intended meaning of a polysemous word given minimal textual context. The core technique combines CLIP-based contrastive alignment, diffusion-based image generation for synthetic sense augmentation, and LLM reasoning for context expansion, following the progression from feature-based to contrastive embedding methods surveyed across 2016–2025 research.

## When to Use

- When a user needs to build a system that selects the correct image from candidates given an ambiguous word and short context (e.g., "bank" + "river bank" → pick the nature image, not the building)
- When implementing an image search or retrieval system that must handle polysemous queries (e.g., "crane" returning construction equipment vs. bird photos depending on context)
- When designing a multilingual content system where the same word in different languages maps to different visual concepts
- When building accessibility tools that pair images with text and must resolve which sense of a word the image should depict
- When fine-tuning CLIP or similar vision-language models for sense-aware image-text matching
- When creating evaluation benchmarks or datasets for multimodal disambiguation tasks
- When augmenting a RAG pipeline with visual grounding to ensure retrieved images match the intended word sense

## Key Technique

**VWSD task definition:** Given a target ambiguous word `w`, a short context phrase `c` (often just 2–3 words like "andromeda galaxy"), and a set of 10 candidate images `{I_1, ..., I_10}`, the system must rank images so the one depicting the intended sense of `w` in context `c` is ranked first. Evaluation uses Mean Reciprocal Rank (MRR) and Hit Rate@1 (HR@1).

**Three-pillar approach from the literature:** (1) **CLIP contrastive alignment** encodes both the text context and each candidate image into a shared embedding space, then ranks images by cosine similarity to the text embedding. Fine-tuning CLIP on sense-annotated image-text pairs yields 6–8% MRR gains over zero-shot baselines. (2) **Diffusion-based augmentation** uses text-to-image models (e.g., Stable Diffusion) to generate synthetic reference images for each word sense from the context phrase, then compares candidates against these generated references in CLIP space — effectively creating a "visual definition" of the intended sense. (3) **LLM context expansion** uses an LLM to enrich the sparse context phrase into a detailed description of the intended sense (e.g., "andromeda galaxy" → "the Andromeda spiral galaxy, Messier 31, a large barred spiral galaxy visible as a faint smudge in the night sky"), which dramatically improves CLIP retrieval quality.

**What makes this better than text-only WSD:** Traditional WSD relies on lexical resources like WordNet and surrounding sentence context. VWSD operates with minimal text (often just a target word + one modifier) and leverages visual grounding — the fact that different senses of a word look fundamentally different in image space. This makes it uniquely suited for image retrieval, visual QA, and cross-modal search where textual context is sparse.

## Step-by-Step Workflow

1. **Parse the disambiguation request.** Extract the ambiguous target word, any available context phrase, and the candidate image set (URLs, file paths, or base64). If the user provides only a word, prompt for context or candidate images.

2. **Expand context with LLM reasoning.** Use the LLM to generate a detailed sense description from the sparse context. Prompt pattern: `"The word '{word}' in the context '{context}' refers to: [generate a 2-3 sentence visual description of what this sense looks like]"`. Generate descriptions for the top 2–3 most likely senses if the context is highly ambiguous.

3. **Encode text representations.** Pass both the original context phrase and the expanded LLM description through CLIP's text encoder (or OpenAI's API if using `openai.Embedding`). Concatenate or average the embeddings to form the query vector `q_text`.

4. **Encode candidate images.** Pass each candidate image through CLIP's image encoder to produce image embeddings `{e_1, ..., e_n}`. If using an API, batch the requests. Normalize all embeddings to unit vectors.

5. **Generate synthetic reference images (optional but powerful).** Use a diffusion model (Stable Diffusion, DALL-E) to generate 1–3 reference images from the expanded sense description. Encode these with CLIP's image encoder to get reference embeddings `{r_1, ..., r_k}`. Average them into a single reference vector `q_visual`.

6. **Compute similarity scores.** For each candidate image `I_j`, compute: `score_j = α * cos(q_text, e_j) + (1 - α) * cos(q_visual, e_j)`, where `α` controls the text-vs-visual weight (default `α = 0.6`). If no synthetic references were generated, use `α = 1.0`.

7. **Rank candidates and select.** Sort candidates by descending score. Return the top-ranked image as the predicted sense match, along with the full ranking and confidence scores.

8. **Validate with sense inventory (when available).** If a sense inventory (WordNet synsets, BabelNet IDs) is accessible, map the selected image back to a formal sense ID. Cross-check the LLM's expanded description against the synset gloss for consistency.

9. **Handle multilingual inputs.** For non-English contexts, use multilingual CLIP (e.g., `M-CLIP` or `XLM-R` backed models) or translate the context to English before encoding. Preserve the original language sense distinctions — some polysemy is language-specific.

10. **Evaluate and iterate.** Compute MRR and HR@1 on a held-out set. If MRR < 0.75, consider fine-tuning CLIP on domain-specific sense-image pairs using contrastive loss with hard negatives (images of wrong senses of the same word).

## Concrete Examples

**Example 1: Image search disambiguation**

```
User: I'm building an image search engine. When a user searches "bass",
I need to show fish photos if they're in a fishing context and instrument
photos if they're in a music context. How do I do this?

Approach:
1. Extract query word ("bass") and context signals (search history,
   category filter, or co-occurring terms like "lake" or "guitar").
2. Expand context with LLM:
   - Fishing context → "A largemouth bass fish, freshwater species with
     greenish body and dark lateral stripe, often shown being caught
     or swimming underwater"
   - Music context → "A bass guitar or upright bass, stringed musical
     instrument with a long neck, typically dark wood or electric body"
3. Encode both descriptions with CLIP text encoder.
4. For each image in your index, compute cosine similarity against
   the appropriate sense embedding.
5. Re-rank search results by similarity to the detected sense.

Implementation (Python):
```

```python
import torch
from transformers import CLIPModel, CLIPProcessor

model = CLIPModel.from_pretrained("openai/clip-vit-large-patch14")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-large-patch14")

def disambiguate_and_rank(query_word, context_hint, candidate_images):
    """Rank candidate images by relevance to the intended word sense."""

    # Step 1: LLM context expansion (use your preferred LLM)
    expanded = expand_context_with_llm(query_word, context_hint)
    # e.g., "bass in fishing context" → detailed visual description

    # Step 2: Encode text query (original + expanded)
    text_inputs = processor(
        text=[f"{context_hint} {query_word}", expanded],
        return_tensors="pt", padding=True, truncation=True
    )
    text_embeds = model.get_text_features(**text_inputs)
    text_embeds = text_embeds / text_embeds.norm(dim=-1, keepdim=True)
    query_vec = text_embeds.mean(dim=0, keepdim=True)

    # Step 3: Encode candidate images
    image_inputs = processor(images=candidate_images, return_tensors="pt")
    image_embeds = model.get_image_features(**image_inputs)
    image_embeds = image_embeds / image_embeds.norm(dim=-1, keepdim=True)

    # Step 4: Rank by cosine similarity
    scores = (query_vec @ image_embeds.T).squeeze()
    ranked_indices = scores.argsort(descending=True).tolist()
    return ranked_indices, scores[ranked_indices].tolist()
```

**Example 2: Multilingual visual disambiguation**

```
User: I have a dataset with Farsi words that are ambiguous. I need to
match each word+context to the correct image from 10 candidates.
The dataset follows the SemEval-2023 Task 1 format.

Approach:
1. Load the dataset: each row has (target_word, context, 10 image paths,
   gold_image_index).
2. Use multilingual CLIP (M-CLIP or NLLB-CLIP) to handle Farsi text.
3. For each instance:
   a. Translate context to English as a fallback signal.
   b. Encode both Farsi original and English translation with M-CLIP.
   c. Average the two text embeddings.
   d. Encode all 10 candidate images.
   e. Rank by cosine similarity.
4. Compute MRR: for each instance, MRR_i = 1 / rank_of_gold_image.
   Final MRR = mean(MRR_i).

Output:
  Instance 1: word="شیر" context="شیر جنگل" → ranked gold at position 1 (MRR=1.0)
  Instance 2: word="شیر" context="شیر آب"   → ranked gold at position 1 (MRR=1.0)
  Instance 3: word="سیب" context="سیب زمینی" → ranked gold at position 2 (MRR=0.5)
  ...
  Overall MRR: 0.82 | HR@1: 0.74
```

**Example 3: Diffusion-augmented disambiguation**

```
User: My CLIP-only approach gets MRR of 0.71 on ambiguous words where
both senses are visually similar (e.g., "trunk" as tree trunk vs. car trunk).
How can I improve it?

Approach:
1. Identify hard cases: instances where top-2 candidates have similarity
   scores within 0.05 of each other.
2. For these hard cases, generate synthetic reference images:
   a. Use the LLM-expanded description as a Stable Diffusion prompt.
   b. Generate 3 images per sense at different seeds.
   c. Encode generated images with CLIP → average into reference vector.
3. Re-score candidates using the combined text + visual reference:
   score = 0.5 * cos(text_embed, candidate) + 0.5 * cos(ref_visual, candidate)
4. This two-signal approach breaks ties that pure text matching cannot.

Expected improvement: 4-6% MRR gain on hard disambiguation cases,
based on the literature showing diffusion augmentation is most effective
when text-only similarity is insufficient to distinguish senses.
```

## Best Practices

- **Do:** Always expand sparse context before encoding. A 2-word phrase like "crane bird" encodes poorly in CLIP; a full sentence description like "a tall wading bird with long legs and neck, often standing in marshland" produces dramatically better embeddings.
- **Do:** Use hard negative mining when fine-tuning — pair each sense's correct image with images of the *wrong* sense of the *same* word. This teaches the model the specific distinctions that matter.
- **Do:** Weight text similarity higher than visual reference similarity (`α ≥ 0.6`) unless you have high-quality generated references. Poor diffusion outputs add noise.
- **Do:** Cache CLIP embeddings for your image corpus. Re-encoding images per query is unnecessary since image embeddings are static.
- **Avoid:** Relying on zero-shot CLIP alone for polysemous words with visually similar senses. The literature consistently shows 6–8% MRR degradation vs. fine-tuned or augmented approaches.
- **Avoid:** Using English-only CLIP for non-English inputs without translation. Multilingual CLIP variants or translate-then-encode pipelines are essential for Farsi, Italian, and other languages in VWSD benchmarks.

## Error Handling

- **No candidate images match any sense:** If all cosine similarities are below a threshold (e.g., < 0.15), flag the instance as "low confidence" and return the ranking with a warning. The context may be too sparse or the word may not be in CLIP's training vocabulary.
- **LLM generates wrong sense expansion:** Validate the expanded description by checking if it mentions the target word's intended domain. If the context is "bank river" but the LLM describes a financial institution, regenerate with a more constrained prompt: `"The word 'bank' in 'bank river' refers to the physical geographic feature: ..."`.
- **CLIP embedding collapse:** If multiple candidate images produce near-identical embeddings (similarity > 0.98 between candidates), the images may be too similar for CLIP to distinguish. Fall back to pixel-level comparison or a domain-specific classifier.
- **Multilingual encoding failure:** If the multilingual CLIP model returns near-random similarities for a language, fall back to translating the context to English via a translation API before encoding.
- **Out-of-vocabulary target words:** Rare or domain-specific words may not be well-represented in CLIP's text encoder. Generate a dictionary-style definition and use that as the text input instead of the raw word.

## Limitations

- **Sparse context is the fundamental bottleneck.** VWSD operates with minimal text (often 2–3 words). When context is a single word with no modifier, even augmented systems struggle — there is simply not enough signal to determine intended sense.
- **Bias toward dominant senses.** CLIP and diffusion models are trained on web data where common senses (e.g., "apple" as the company) dominate. Rare senses (e.g., "apple" as the fruit variety Malus sieversii) are systematically under-represented and harder to retrieve.
- **Multilingual coverage gaps.** Most VWSD research and datasets cover English, with limited support for Farsi and Italian. Low-resource languages lack sense-annotated visual data entirely.
- **Computational cost.** Running CLIP encoding + diffusion generation + LLM expansion per query is expensive. For real-time applications, pre-compute as much as possible and reserve diffusion augmentation for hard cases only.
- **Evaluation brittleness.** MRR on 10-candidate sets can be misleading — a system might rank the gold image second every time (MRR = 0.5) while being practically useful. Consider task-specific metrics alongside MRR.

## Reference

**Paper:** Nilukshi & Sumanathilaka, "Bridging Lexical Ambiguity and Vision: A Mini Review on Visual Word Sense Disambiguation" (2026). [arXiv:2602.01193v1](https://arxiv.org/abs/2602.01193v1). Look for: Table 1 comparing method categories (feature-based, graph-based, contrastive), Table 2 with quantitative MRR results across systems, and the analysis of CLIP + diffusion + LLM convergence as the state-of-the-art pipeline.