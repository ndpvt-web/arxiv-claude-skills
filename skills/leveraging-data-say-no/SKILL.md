---
name: "leveraging-data-say-no"
description: "Implement memory-augmented selective prediction for vision-language models using retrieval-based confidence scoring and contrastive normalization. Use when: 'add abstain/reject option to VLM predictions', 'build confidence scoring for image captioning', 'implement selective prediction with CLIP', 'calibrate VLM confidence using retrieval', 'filter unreliable model outputs with memory augmentation', 'add know-when-to-say-no to a vision pipeline'."
---

# Memory-Augmented Plug-and-Play Selective Prediction (MA-PaPSP)

This skill enables Claude to implement selective prediction systems for vision-language models (VLMs) — giving any VLM the ability to abstain from making predictions when confidence is low. The core technique, from the ICLR 2026 paper "Leveraging Data to Say No," uses a retrieval dataset of image-text pairs combined with contrastive normalization to produce well-calibrated confidence scores without retraining the underlying model. It works across captioning, image-text matching, and fine-grained classification, and requires only an external CLIP-family encoder plus a retrieval corpus.

## When to Use

- When the user wants to add a reject/abstain option to a vision-language model so it can skip low-confidence outputs
- When building a selective captioning pipeline that should only emit captions the model is confident about
- When the user needs calibrated confidence scores from CLIP-based similarity (raw cosine similarity is poorly calibrated)
- When implementing quality-gated inference where unreliable predictions are filtered before reaching downstream consumers
- When the user asks to reduce hallucination or error rates by letting a model "say no" to hard examples
- When building retrieval-augmented confidence estimation for any image-text task without fine-tuning

## Key Technique

**The Problem:** VLMs produce predictions for every input, even when those predictions are wrong. Standard confidence measures (softmax probabilities) only work for closed-vocabulary tasks. For open-set tasks like captioning, there is no probability distribution to inspect. Naive CLIP cosine similarity between image and predicted text suffers from two flaws: (1) high variance in embeddings makes scores noisy, and (2) raw similarity magnitudes are poorly calibrated across different regions of embedding space, so a fixed threshold cannot reliably separate correct from incorrect predictions.

**The Solution (MA-PaPSP):** Instead of using raw CLIP embeddings, retrieve K nearest-neighbor image-text pairs from a reference corpus and compute weighted-average *proxy embeddings*. These averaged representations are more stable than single-sample embeddings, reducing variance. Then, instead of using raw cosine similarity as the confidence score, apply *contrastive normalization*: generate hard negatives by replacing key nouns in the predicted text with semantically different alternatives (via WordNet or an LLM), compute similarity for each negative, and normalize the positive score against these negatives using a softmax-style denominator. This produces scores with uniform magnitude across the embedding space, enabling a single threshold to work globally.

**Why It Works:** The proxy embedding step averages out noise — with K=15 neighbors, the effective embedding converges toward the true semantic region. Contrastive normalization converts absolute similarity (which varies by image complexity, caption length, etc.) into a relative measure: "how much better does this prediction match than plausible alternatives?" This relative score is inherently better calibrated. The combination yields 15-30% AURC improvements over raw CLIP scoring, and a 16M-parameter scoring model with MA-PaPSP outperforms a 1B-parameter model without it.

## Step-by-Step Workflow

1. **Choose the SP-VLM (Selective Prediction VLM) encoder.** Use a CLIP-family model as the external scoring backbone. SigLIP B/16 or SO-400M are strong defaults. This encoder is separate from the prediction model — it only scores predictions.

2. **Prepare the retrieval corpus.** Assemble a dataset of (image, text) pairs. In-domain data (e.g., COCO train split for captioning tasks) works best, but out-of-domain corpora like CC3M or CC12M also provide gains. Store CLIP image embeddings for all retrieval samples in a FAISS index for fast nearest-neighbor lookup.

3. **Encode the query image.** For each test input image x, compute its CLIP image embedding `phi_img(x)`.

4. **Retrieve K nearest neighbors.** Query the FAISS index with `phi_img(x)` to find the K=15 closest retrieval images. Return both their image and text embeddings from the retrieval corpus.

5. **Compute proxy embeddings.** For both image and text modalities, compute a weighted average of the K retrieved neighbors' embeddings, where weights are proportional to cosine similarity with the query:

   ```python
   # weights[i] = cos_sim(query_embed, neighbor_i_embed)
   weights = F.cosine_similarity(query_embed, neighbor_embeds, dim=-1)
   weights = F.softmax(weights, dim=0)
   proxy_embed = (weights.unsqueeze(-1) * neighbor_embeds).sum(dim=0)
   ```

6. **Generate hard negatives for contrastive normalization.** Take the VLM's predicted text `f(x)` and produce 5-10 negative variants by replacing nouns with WordNet hypernyms/siblings (fast, rule-based) or by prompting an LLM to produce semantically distinct alternatives. Encode each negative with the CLIP text encoder.

7. **Compute the contrastive confidence score.** Calculate cosine similarity between the proxy image embedding and the proxy text embedding for the positive prediction, then normalize against negatives:

   ```python
   # s_pos = cos_sim(proxy_img, proxy_txt_positive)
   # s_neg_k = cos_sim(proxy_img, proxy_txt_negative_k) for each negative
   tau = 0.07  # temperature
   score = exp(s_pos / tau) / (exp(s_pos / tau) + sum(exp(s_neg / tau) for s_neg in s_negs))
   ```

8. **Apply the accept/reject threshold.** Set a coverage target (e.g., accept 80% of predictions) and find the score threshold that achieves it on a calibration set. At inference, accept predictions with score >= threshold, reject the rest.

9. **Evaluate using AURC (Area Under Risk-Coverage Curve).** Sweep across all thresholds, compute risk (error rate on accepted samples) at each coverage level, and integrate. Lower AURC means the system correctly rejects hard examples first.

10. **Optimize retrieval overhead.** Use FAISS IVF indexes for large corpora. The rule-based negative generation (WordNet substitution) adds only ~5ms per sample vs. ~40ms for LLM-based generation, with comparable performance.

## Concrete Examples

**Example 1: Selective Image Captioning Pipeline**

User: "I have a captioning model (BLIP-2) generating captions for product images. Some captions are wrong or hallucinated. Build a confidence filter that rejects bad captions."

Approach:
1. Load SigLIP as the scoring encoder, separate from BLIP-2
2. Build a FAISS index from the product image training set (images + ground-truth descriptions)
3. For each product image, generate a caption with BLIP-2
4. Retrieve K=15 nearest product images from the FAISS index
5. Compute proxy embeddings by weighted-averaging the neighbors
6. Generate negatives: replace key nouns in the caption ("red leather handbag" -> "blue nylon backpack", "wooden picture frame", etc.)
7. Compute contrastive score; reject captions below threshold

```python
import torch
import faiss
import open_clip
from nltk.corpus import wordnet as wn

class MAPaPSP:
    def __init__(self, clip_model_name="ViT-B-16-SigLIP", retrieval_embeddings=None,
                 retrieval_texts=None, k=15, tau=0.07):
        self.model, _, self.preprocess = open_clip.create_model_and_transforms(clip_model_name)
        self.tokenizer = open_clip.get_tokenizer(clip_model_name)
        self.k = k
        self.tau = tau
        # Build FAISS index from precomputed retrieval image embeddings
        self.index = faiss.IndexFlatIP(retrieval_embeddings.shape[1])
        self.index.add(retrieval_embeddings)
        self.ret_img_embeds = torch.tensor(retrieval_embeddings)
        self.ret_txt_embeds = retrieval_texts  # precomputed text embeddings

    def get_proxy_embedding(self, query_embed, neighbor_indices):
        neighbor_embeds = self.ret_img_embeds[neighbor_indices]
        weights = torch.cosine_similarity(query_embed.unsqueeze(0), neighbor_embeds, dim=-1)
        weights = torch.softmax(weights, dim=0)
        return (weights.unsqueeze(-1) * neighbor_embeds).sum(dim=0)

    def generate_negatives(self, caption, n=8):
        """Replace nouns with WordNet alternatives."""
        import nltk
        tokens = nltk.pos_tag(nltk.word_tokenize(caption))
        nouns = [w for w, p in tokens if p.startswith("NN")]
        negatives = []
        for noun in nouns[:3]:
            for syn in wn.synsets(noun, pos=wn.NOUN)[:3]:
                for lemma in syn.lemmas()[:2]:
                    if lemma.name() != noun:
                        negatives.append(caption.replace(noun, lemma.name().replace("_", " ")))
        return negatives[:n]

    def score(self, image, caption):
        img_embed = self.model.encode_image(self.preprocess(image).unsqueeze(0))
        img_embed = img_embed / img_embed.norm(dim=-1, keepdim=True)
        # Retrieve neighbors
        _, indices = self.index.search(img_embed.detach().numpy(), self.k)
        proxy_img = self.get_proxy_embedding(img_embed.squeeze(), indices[0])
        # Encode positive and negatives
        pos_embed = self.model.encode_text(self.tokenizer(caption))
        negatives = self.generate_negatives(caption)
        neg_embeds = self.model.encode_text(self.tokenizer(negatives))
        # Contrastive score
        s_pos = torch.cosine_similarity(proxy_img.unsqueeze(0), pos_embed, dim=-1)
        s_negs = torch.cosine_similarity(proxy_img.unsqueeze(0), neg_embeds, dim=-1)
        score = torch.exp(s_pos / self.tau) / (
            torch.exp(s_pos / self.tau) + torch.exp(s_negs / self.tau).sum()
        )
        return score.item()
```

**Example 2: Quality-Gated Image-Text Matching**

User: "I'm matching product images to catalog descriptions. I need a confidence score to flag uncertain matches for human review."

Approach:
1. Use the same MA-PaPSP pipeline but skip caption generation — the text is given
2. Build retrieval index from the catalog's training image-text pairs
3. For each (image, description) pair, compute contrastive score
4. Route low-confidence matches to human reviewers, auto-approve high-confidence ones

Output:
```
Image: product_0412.jpg | Description: "Sterling silver hoop earrings, 25mm"
  Raw CLIP similarity: 0.31 (uncalibrated — hard to threshold)
  MA-PaPSP score: 0.89 -> ACCEPT (above 0.65 threshold)

Image: product_0873.jpg | Description: "Organic cotton throw blanket, grey"
  Raw CLIP similarity: 0.28 (looks similar to above, but wrong match)
  MA-PaPSP score: 0.23 -> REJECT (contrastive norm reveals ambiguity)
```

**Example 3: Selective Fine-Grained Classification**

User: "My bird species classifier sometimes confuses similar species. Add a confidence gate so uncertain predictions go to an expert."

Approach:
1. Build retrieval index from training images of the bird classification dataset
2. After the classifier predicts a species, encode (image, predicted_species_name) with CLIP
3. Retrieve neighbors, compute proxy embeddings, generate negatives from other species names
4. Contrastive score naturally captures inter-class confusion — similar species produce high negative scores, lowering confidence appropriately

## Best Practices

- **Do:** Use in-domain retrieval data when available. COCO train for COCO captioning, CUB train for bird classification. In-domain retrieval consistently outperforms out-of-domain by 5-15%.
- **Do:** Set K=15 as the default neighborhood size. K=1 severely underperforms; K=15 provides the best variance-performance tradeoff across tasks.
- **Do:** Use the rule-based WordNet negative generation for production systems. It adds only ~5ms per sample and matches LLM-generated negatives in quality for this task.
- **Do:** Normalize all embeddings to unit length before computing cosine similarity and building the FAISS index.
- **Avoid:** Using raw CLIP cosine similarity as a confidence score without contrastive normalization. It is poorly calibrated — the same score means different things in different regions of embedding space.
- **Avoid:** Using a very small retrieval corpus (<1K pairs). The proxy embedding averaging needs sufficient density in embedding space to be effective. At minimum, use several thousand retrieval pairs.
- **Avoid:** Generating negatives that are too similar to the positive (e.g., only changing adjectives). The contrastive signal requires semantically distinct alternatives — swap nouns, not modifiers.

## Error Handling

- **FAISS index is empty or too small:** Fall back to standard PaPSP (direct CLIP cosine similarity) without proxy embeddings. Log a warning that confidence scores will be less calibrated.
- **WordNet fails to find substitutions for a noun:** Use a fallback list of generic nouns ("object", "thing", "item") or skip contrastive normalization for that sample and use the proxy embedding score alone.
- **CLIP model mismatch:** The same CLIP model must encode both the retrieval corpus and the test inputs. Mixing encoders (e.g., indexing with OpenCLIP, querying with SigLIP) produces meaningless similarity scores.
- **Out-of-memory on large retrieval sets:** Use FAISS IVF or HNSW indexes instead of flat indexes. With CC12M (12M pairs), IVF with nprobe=32 maintains retrieval quality while fitting in 8GB GPU memory.
- **No nouns detected in caption:** This happens with very short or unusual captions. Fall back to random word replacement or use the proxy score without contrastive normalization.

## Limitations

- **Training-free but not data-free:** Requires a retrieval corpus of image-text pairs. If no relevant corpus exists for your domain, the proxy embeddings may not help.
- **Latency overhead:** The full pipeline adds 17-58ms per sample depending on negative generation strategy. For real-time applications processing thousands of images per second, this may be prohibitive.
- **Dependent on CLIP-family encoder quality:** If the CLIP model has weak coverage of your domain (e.g., medical images, satellite imagery), the confidence scores will be unreliable regardless of retrieval augmentation.
- **Open-vocabulary assumption:** The contrastive normalization assumes you can generate meaningful negatives. For highly specialized or technical captions, generic WordNet substitution may produce poor negatives.
- **Does not improve the underlying model:** This only tells you *when* the model is wrong — it does not fix the predictions. You still need a separate strategy for handling rejected samples (human review, alternative model, etc.).

## Reference

[Leveraging Data to Say No: Memory Augmented Plug-and-Play Selective Prediction](https://arxiv.org/abs/2601.22570) (Sarkar et al., ICLR 2026). Focus on Sections 3.2 (proxy embeddings) and 3.3 (contrastive normalization) for the core algorithmic contributions, and Table 1 for the taxonomy of selective prediction tasks. Code: [github.com/kingston-aditya/MA-PaPSP](https://github.com/kingston-aditya/MA-PaPSP).