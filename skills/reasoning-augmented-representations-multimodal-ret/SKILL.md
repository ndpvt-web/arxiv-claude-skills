---
name: "reasoning-augmented-representations-multimodal-ret"
description: "Decouple reasoning from embedding compression in multimodal retrieval pipelines by enriching queries and corpus entries with explicit semantic context before encoding. Use when: 'build a multimodal search system', 'improve image-text retrieval accuracy', 'fix retrieval for ambiguous queries', 'add reasoning to my embedding pipeline', 'my CLIP search returns wrong results for complex queries', 'enhance retrieval with VLM captions'."
---

# Reasoning-Augmented Representations for Multimodal Retrieval

This skill teaches Claude to apply the Reasoning-Augmented Representations (RAR) technique from Zhang et al. (2026) to build and improve multimodal retrieval systems. The core insight: embedding models fail on queries requiring latent reasoning (ambiguous references, compositional constraints, implicit visual semantics) because a single encoding pass must simultaneously *reason about intent* and *compress into a vector*. RAR fixes this by externalizing reasoning as a preprocessing step -- using a Vision-Language Model to densify the semantic content of both queries and corpus entries before they ever reach the embedding model.

## When to Use

- When building an image-text search system and retrieval quality degrades on compositional or ambiguous queries (e.g., "find a photo like this but with a red background")
- When a user's CLIP/SigLIP-based retrieval returns results matching surface features (pose, color) instead of semantic intent
- When designing a retrieval pipeline over a corpus where images carry information not captured in metadata (diagrams, product photos, medical scans)
- When queries contain deictic references like "this animal" or "the building shown here" that need resolution before embedding
- When building RAG systems that retrieve multimodal documents and need to handle underspecified natural language questions
- When fine-tuning a retrieval model and looking for a data-centric augmentation strategy that avoids architecture changes

## Key Technique

**The Problem: Reasoning-Compression Bottleneck.** Standard multimodal retrievers encode queries and documents into dense vectors with a single forward pass. When the query is ambiguous ("What species is this?") or the document's key evidence is purely visual (an unlabeled photo of a Giant Panda), the encoder must implicitly *reason* about what matters and *compress* it into the same fixed-dimensional vector. This dual burden produces brittle embeddings that latch onto spurious surface features -- matching pose similarity when the user asked for semantic similarity, or matching background textures instead of foreground objects.

**The Solution: Externalize Reasoning, Then Embed.** RAR decouples reasoning from compression by using a VLM (e.g., Qwen-VL) to preprocess both sides of the retrieval problem. For corpus entries, it generates dense captions that "unsilence" visual evidence -- converting implicit image content into explicit, keyword-rich text. For queries, it resolves ambiguous references, completes implicit intent, and distills modification instructions into concise constraint sets. The embedding model then operates on semantically dense inputs where the reasoning has already been done.

**Critical: Training Alignment.** Simply augmenting at inference time causes *performance degradation* due to distribution shift -- models trained on sparse text cannot process dense captions. The retriever must be fine-tuned on the enhanced representations using contrastive learning. This alignment step is non-negotiable; without it, the augmented data acts as noise rather than signal.

## Step-by-Step Workflow

1. **Classify your corpus entries** into three categories: text-only (no enhancement needed), image-only (needs full dense captioning), and image-text pairs (needs caption appended as supplementary context).

2. **Generate dense corpus captions** using a VLM. For each image, prompt the VLM with explicit instructions: identify the main object/entity/scene, list specific details (colors, materials, text/logos), state named entities when recognizable, cap at ~100 words, and avoid aesthetic or subjective language. Append results as `"\nVisual Context: {caption}"` for image-text pairs.

3. **Classify incoming queries** into three types: text-only (pass through unchanged), image-only (generate a 50-word dense caption), or image-text (further subdivide into QA-style vs. modification-style).

4. **Enhance QA-style queries** by resolving deictic references ("this animal" -> "Giant Panda"), completing implicit intent, adding 3-5 words of visual description maximum, and deleting filler adjectives. The VLM receives both image and text.

5. **Enhance modification-style queries** by extracting target constraint keywords. Delete filler verbs (is, has, make, change, show), preserve all descriptive adjectives and nouns, and critically *exclude the reference image from VLM input* to prevent contextual contamination -- the model should distill the instruction text alone.

6. **Build training pairs** by aligning enhanced queries `q̃` with enhanced corpus entries `d̃`. Use the original ground-truth relevance labels; the enhancement changes representation, not relevance judgments.

7. **Fine-tune the retriever** on enhanced pairs using contrastive loss. Use the `<emb>` token approach (append a special embedding token after task instructions) or your framework's equivalent pooling strategy. This alignment step bridges the distribution gap between sparse originals and dense augmented representations.

8. **Deploy with inference-time enhancement** by running the same VLM preprocessing on incoming queries before encoding. Corpus entries are pre-enhanced and cached; only query enhancement adds latency at serving time.

9. **Validate with ablation** by measuring retrieval quality (Recall@K) across four configurations: baseline (no enhancement), corpus-only enhancement, query-only enhancement, and full enhancement. Expect corpus enhancement to primarily boost knowledge-intensive queries and query enhancement to primarily boost compositional/entity-centric queries.

## Concrete Examples

**Example 1: Building a Product Image Search with Ambiguous Queries**

User: "My e-commerce image search returns wrong products when users upload a photo and ask 'find something like this but in blue'. How do I fix this?"

Approach:
1. Classify the task: modification-style query (image + text instruction)
2. For each product image in the corpus, generate a dense caption:
   ```python
   corpus_prompt = """Describe this product image in under 100 words.
   Identify: product type, material, color, brand/logos, shape, size cues.
   State named entities if recognizable. No aesthetic language."""

   caption = vlm.generate(image=product_img, prompt=corpus_prompt)
   enhanced_entry = f"{original_metadata}\nVisual Context: {caption}"
   ```
3. For the modification query, extract constraints from text only (no image):
   ```python
   query_prompt = """Extract the target attributes from this modification request.
   Delete filler verbs (find, make, change, show).
   Keep ALL descriptive adjectives and nouns as keywords.
   Input: 'find something like this but in blue'"""
   # Output: "blue"

   enhanced_query = f"{original_query_text} | Constraints: {constraints}"
   ```
4. Fine-tune retriever on (enhanced_query, enhanced_corpus) pairs with contrastive loss
5. At serving time, run constraint extraction on user queries before embedding

Output: The retriever now matches on the semantic constraint "blue" against corpus captions containing explicit color descriptions, instead of matching pose/shape features from the reference image.

**Example 2: Building a Visual QA Retrieval System Over Wikipedia**

User: "I'm building a system where users upload a photo and ask questions like 'What lake is this?' and I need to retrieve the right Wikipedia article. Current retrieval is terrible."

Approach:
1. Classify: QA-style query (image + question with implicit reference)
2. Enhance Wikipedia corpus entries that contain images:
   ```python
   for entry in wikipedia_corpus:
       if entry.has_image:
           caption = vlm.generate(
               image=entry.image,
               prompt="Dense caption, 100 words max. Identify: location, "
                      "geographic features, surrounding landmarks, text/signs."
           )
           entry.text = f"{entry.text}\nVisual Context: {caption}"
   ```
3. Enhance the query by resolving "this" to explicit visual description:
   ```python
   query_prompt = """The user uploaded an image and asks: 'What lake is this?'
   Resolve 'this' to a specific visual description.
   Add up to 5 words of discriminative visual detail.
   Output the rewritten query only."""
   # Input image: photo of a lake surrounded by green mountains
   # Output: "What lake is surrounded by green forested mountains with
   #          a small island and calm reflective water"
   ```
4. Train retriever on enhanced (query, article) pairs
5. The shared visual context ("surrounded by green mountains") now bridges query and corpus

Output: Retrieval accuracy on entity-centric visual questions improves significantly (the paper reports +2.84% Recall@1 on InfoSeek benchmarks).

**Example 3: Implementing the Enhancement Pipeline in Python**

User: "Give me the code skeleton for the RAR enhancement pipeline."

```python
from enum import Enum
from dataclasses import dataclass

class EntryType(Enum):
    TEXT_ONLY = "text_only"
    IMAGE_ONLY = "image_only"
    IMAGE_TEXT = "image_text"

class QueryType(Enum):
    TEXT_ONLY = "text_only"
    IMAGE_ONLY = "image_only"
    QA_STYLE = "qa_style"
    MODIFICATION = "modification"

@dataclass
class EnhancedEntry:
    original_text: str | None
    visual_context: str | None
    entry_type: EntryType

CORPUS_CAPTION_PROMPT = (
    "Describe this image in under {word_limit} words. "
    "Identify the main object, entity, or scene. "
    "List: colors, materials, text/logos, named entities. "
    "No aesthetic or subjective language."
)

QA_RESOLVE_PROMPT = (
    "Rewrite this question by resolving ambiguous references "
    "(e.g., 'this', 'the one shown') to explicit visual descriptions. "
    "Add at most 5 words of visual detail. Delete filler adjectives. "
    "Question: {query_text}"
)

MODIFICATION_EXTRACT_PROMPT = (
    "Extract target attribute keywords from this instruction. "
    "Delete filler verbs (is, has, make, change, show, find). "
    "Preserve ALL descriptive adjectives and nouns. "
    "Instruction: {query_text}"
)

def enhance_corpus_entry(entry, vlm) -> EnhancedEntry:
    if entry.entry_type == EntryType.TEXT_ONLY:
        return EnhancedEntry(entry.text, None, entry.entry_type)

    caption = vlm.generate(
        image=entry.image,
        prompt=CORPUS_CAPTION_PROMPT.format(word_limit=100)
    )

    if entry.entry_type == EntryType.IMAGE_ONLY:
        return EnhancedEntry(caption, caption, EntryType.IMAGE_ONLY)

    # IMAGE_TEXT: append visual context to original text
    combined = f"{entry.text}\nVisual Context: {caption}"
    return EnhancedEntry(combined, caption, EntryType.IMAGE_TEXT)

def enhance_query(query, vlm) -> str:
    if query.query_type == QueryType.TEXT_ONLY:
        return query.text

    if query.query_type == QueryType.IMAGE_ONLY:
        return vlm.generate(
            image=query.image,
            prompt=CORPUS_CAPTION_PROMPT.format(word_limit=50)
        )

    if query.query_type == QueryType.QA_STYLE:
        return vlm.generate(
            image=query.image,  # include image for reference resolution
            prompt=QA_RESOLVE_PROMPT.format(query_text=query.text)
        )

    if query.query_type == QueryType.MODIFICATION:
        # CRITICAL: exclude image to prevent contextual contamination
        return vlm.generate(
            prompt=MODIFICATION_EXTRACT_PROMPT.format(query_text=query.text)
        )
```

## Best Practices

- **Do:** Always classify entries/queries into types before enhancement. The three corpus categories and four query categories require different prompts and different VLM inputs.
- **Do:** Cap caption length strictly (100 words corpus, 50 words query). Verbose captions introduce noise that dilutes embedding signal.
- **Do:** Exclude the reference image when enhancing modification-style queries. Including it causes the VLM to describe the reference instead of extracting the desired change, contaminating the query embedding.
- **Do:** Fine-tune your retriever on enhanced data. Inference-time-only enhancement consistently degrades performance due to distribution shift.
- **Avoid:** Applying enhancement to text-only entries or queries. These already contain explicit semantics; captioning adds nothing and may inject hallucinated content.
- **Avoid:** Using subjective or aesthetic language in caption prompts ("beautiful", "stunning"). Dense captions must be factual and keyword-rich to support lexical matching in the embedding space.

## Error Handling

- **Distribution shift regression:** If Recall@K drops after adding enhancements, you are likely applying enhanced representations to a model trained on unenhanced data. Fine-tune on the enhanced pairs before evaluating.
- **VLM hallucination in captions:** If the VLM generates incorrect entity names or fabricated details, add a verification step: cross-check generated named entities against metadata or a knowledge base. Prefer visual descriptors over uncertain entity names.
- **Contaminated modification queries:** If modification-style retrieval returns items similar to the reference image instead of the modified version, verify that the reference image is excluded from VLM input during query enhancement.
- **Latency at serving time:** Query enhancement adds one VLM call per query. Mitigate by using a smaller/quantized VLM for inference-time enhancement, caching frequent query patterns, or batching enhancement calls.
- **Uneven gains across query types:** Corpus enhancement primarily helps knowledge-intensive lookups; query enhancement primarily helps compositional queries. If only one category improves, check that both sides of the pipeline are active.

## Limitations

- Requires access to a capable VLM (7B+ parameters) for caption generation, adding infrastructure cost and latency.
- Enhancement quality is bottlenecked by VLM accuracy -- domains with specialized visual content (medical imaging, satellite imagery) may need domain-specific VLMs.
- The technique addresses *data-induced* brittleness specifically; it does not fix fundamental embedding capacity limitations or architectural constraints.
- Text-only retrieval tasks see no benefit; this approach targets the multimodal gap where visual evidence is implicit or queries are underspecified.
- Fine-tuning on enhanced data means maintaining two pipelines (enhancement + training); inference-only deployment is explicitly shown to hurt performance.

## Reference

Zhang, J., Rajan, A.S., Han, B., Lee, S., & Ganguly, S. (2026). *Reasoning-Augmented Representations for Multimodal Retrieval.* arXiv:2602.07125v1. Key sections: Section 3 (Enhancement Framework) for prompt templates and category taxonomy; Table 3 for the critical distribution-shift ablation showing why training alignment is mandatory; Figures 3-4 for qualitative before/after retrieval examples.