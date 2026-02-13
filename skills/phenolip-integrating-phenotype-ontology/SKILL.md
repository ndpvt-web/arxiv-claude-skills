---
name: "phenolip-integrating-phenotype-ontology"
description: "Build phenotype-aware medical vision-language models by integrating ontology knowledge graphs into CLIP-style pretraining. Use when: 'build a medical image classifier using phenotype ontology', 'create a phenotype-centric knowledge graph from PubMed', 'distill ontology structure into a vision-language model', 'implement contrastive pretraining with hierarchical medical knowledge', 'set up cross-modal retrieval for clinical phenotypes', 'construct a multimodal biomedical knowledge graph with HPO terms'."
---

# PhenoLIP: Ontology-Guided Medical Vision-Language Pretraining

This skill enables Claude to help build medical vision-language systems that integrate structured phenotype ontology knowledge (e.g., HPO — Human Phenotype Ontology) into CLIP-style contrastive pretraining. The core technique from PhenoLIP is a two-stage pipeline: (1) learn a knowledge-enhanced phenotype embedding space from ontology text using contrastive learning on PubMedBERT, then (2) distill that structured knowledge into a multimodal VLM via a teacher-guided objective alongside standard image-text contrastive loss. This produces models that understand medical phenotype hierarchies, synonyms, and definitions — not just surface-level image-caption co-occurrence.

## When to Use

- When the user wants to build a medical image classifier that leverages phenotype ontology structure (HPO, OMIM, SNOMED-CT) rather than flat labels
- When constructing a multimodal knowledge graph linking medical images from PubMed/PMC-OA to ontology phenotype terms
- When implementing two-stage pretraining: ontology-aware text encoder followed by vision-language alignment with knowledge distillation
- When designing a cross-modal retrieval system for phenotype-to-image or image-to-phenotype search
- When improving a BiomedCLIP or general medical VLM with structured clinical knowledge priors
- When building an evaluation benchmark for phenotype recognition from medical images
- When the user needs to encode hierarchical medical relationships (is-a, synonym, definition) into embedding spaces

## Key Technique

**The problem with standard medical VLMs:** CLIP-style models trained on image-caption pairs learn coarse associations. They miss the rich relational structure in medical ontologies — that "Arachnodactyly" is-a child of "Long fingers," which is-a child of "Abnormality of finger," and that each has synonyms and precise definitions. Standard contrastive objectives treat captions as flat strings and cannot encode these hierarchical priors.

**PhenoLIP's two-stage solution:** Stage 1 trains a knowledge encoder (initialized from PubMedBERT) using InfoNCE contrastive loss over ontology attributes. For each phenotype, its name, definition, synonyms, and hierarchical relational descriptions (e.g., "[Arachnodactyly] is a child phenotype of [Long fingers]") are treated as positive pairs within a batch, pulling them together in embedding space. The loss is: `L_k = -(1/2B) * sum( log( exp(sim(z_i, z_i+) / tau_1) / sum_k( exp(sim(z_i, z_k) / tau_1) ) ) )`. This produces a phenotype embedding space where ontologically related terms cluster together.

**Stage 2** runs standard bidirectional image-text contrastive loss (`L_m`) using a ViT-B vision encoder and PubMedBERT text encoder, but adds a knowledge distillation term (`L_kd`). The frozen Stage 1 encoder acts as a teacher: for each caption `c_i`, the teacher produces embedding `k_i = Phi_k(c_i)` and the student text encoder produces `t_i = Phi_t(c_i)`. The distillation loss treats `(t_i, k_i)` as positive pairs, regularizing the student's text representations to stay aligned with the structured ontology space. The total loss is `L = L_m + lambda * L_kd`. This ensures the final VLM inherits phenotype hierarchy knowledge without requiring ontology annotations at inference time.

## Step-by-Step Workflow

### Phase A: Knowledge Graph Construction

1. **Extract phenotype terms from a medical ontology.** Parse HPO (or equivalent OBO/OWL ontology) to extract all phenotype nodes, their definitions, synonyms, and hierarchical edges. Filter to terminal nodes (leaf phenotypes) for image linking. Store as a structured graph with fields: `{id, name, definition, synonyms: [], parents: [], children: []}`.

2. **Crawl and filter medical images.** Query PubMed Central Open Access with phenotype term keywords. For each retrieved figure, apply a subfigure detector (DAB-DETR or similar) to segment compound figures into individual panels. Use DINOv2 embeddings + K-means clustering (k~400) to remove non-medical images (charts, diagrams, logos). Use a VLM (e.g., Qwen2.5-VL) to align subfigures with their captions via bounding box overlays.

3. **Refine captions and link to phenotypes.** Run an LLM over raw captions to remove noise (numeric references, journal metadata, formatting artifacts). Map cleaned captions to phenotype IDs using keyword matching against ontology terms and synonyms. Store as triples: `(image_path, cleaned_caption, phenotype_id)`.

### Phase B: Two-Stage Pretraining

4. **Train the knowledge encoder (Stage 1).** Initialize a text encoder from PubMedBERT. For each phenotype, construct positive pairs from its attributes: `(name, definition)`, `(name, synonym)`, `(name, "X is a child phenotype of Y")`. Train with InfoNCE contrastive loss using in-batch negatives. Use temperature `tau_1` (typically 0.07). Train until ontologically related phenotypes cluster tightly in embedding space.

5. **Set up the multimodal pretraining (Stage 2).** Initialize a ViT-B image encoder (from BiomedCLIP or similar pretrained medical ViT). Initialize the text encoder with the Stage 1 knowledge encoder weights. Freeze the Stage 1 encoder as the teacher.

6. **Train with combined contrastive + distillation loss.** For each batch of `(image, caption)` pairs from PhenoKG: compute bidirectional image-text contrastive loss `L_m` with temperature `tau_2`; compute knowledge distillation loss `L_kd` between student text encoder outputs and frozen teacher outputs; optimize `L = L_m + lambda * L_kd`. The distillation term ensures the text encoder retains ontology structure while learning visual alignment.

7. **Validate on held-out phenotype retrieval.** During training, periodically evaluate on a held-out set using image-to-phenotype (I2P) and phenotype-to-image (P2I) retrieval metrics (Recall@10, Recall@50). Monitor for collapse — if I2P recall drops while I2T stays flat, increase the distillation weight `lambda`.

### Phase C: Evaluation & Deployment

8. **Build an evaluation benchmark.** Collect image-caption pairs covering diverse phenotypes. Have clinical experts verify each pair. Apply strict document-level splits (by PMCID/FigureID) to prevent data leakage. Evaluate with zero-shot classification accuracy and cross-modal retrieval recall.

9. **Deploy for downstream tasks.** For zero-shot phenotype classification: encode candidate phenotype names with the text encoder, encode the query image with the vision encoder, rank by cosine similarity. For retrieval: build a FAISS index over image/text embeddings, query with phenotype descriptions or images.

10. **Fine-tune with linear probing when labeled data exists.** Freeze the pretrained encoders and train a linear classifier on top for specific downstream tasks (dermatology, radiology, pathology). The ontology-aware embeddings provide strong features even at low data ratios (1-10% labels).

## Concrete Examples

**Example 1: Building an Ontology-Aware Knowledge Encoder**

User: "I have HPO phenotype data with definitions and hierarchy. Help me train a phenotype embedding model that captures ontology structure."

Approach:
1. Parse the HPO OBO file into a list of phenotypes with attributes
2. Generate positive pair training data from ontology relationships
3. Implement InfoNCE contrastive training on PubMedBERT

```python
import torch
import torch.nn.functional as F
from transformers import AutoTokenizer, AutoModel

# Step 1: Parse ontology into training pairs
def build_phenotype_pairs(hpo_data):
    """Generate positive pairs from ontology attributes."""
    pairs = []
    for pheno in hpo_data:
        name = pheno["name"]
        # (name, definition) positive pair
        if pheno.get("definition"):
            pairs.append((name, pheno["definition"]))
        # (name, synonym) positive pairs
        for syn in pheno.get("synonyms", []):
            pairs.append((name, syn))
        # (name, hierarchical relation) positive pairs
        for parent in pheno.get("parents", []):
            rel_text = f"{name} is a child phenotype of {parent}"
            pairs.append((name, rel_text))
    return pairs

# Step 2: InfoNCE contrastive loss
def info_nce_loss(z_anchor, z_positive, temperature=0.07):
    """Symmetric InfoNCE loss with in-batch negatives."""
    z_a = F.normalize(z_anchor, dim=-1)
    z_p = F.normalize(z_positive, dim=-1)
    logits = z_a @ z_p.T / temperature
    labels = torch.arange(len(z_a), device=z_a.device)
    loss = (F.cross_entropy(logits, labels) +
            F.cross_entropy(logits.T, labels)) / 2
    return loss

# Step 3: Training loop sketch
tokenizer = AutoTokenizer.from_pretrained("microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract")
encoder = AutoModel.from_pretrained("microsoft/BiomedNLP-PubMedBERT-base-uncased-abstract")

for epoch in range(num_epochs):
    for batch_anchors, batch_positives in dataloader:
        tok_a = tokenizer(batch_anchors, padding=True, return_tensors="pt")
        tok_p = tokenizer(batch_positives, padding=True, return_tensors="pt")
        z_a = encoder(**tok_a).last_hidden_state[:, 0, :]  # CLS pooling
        z_p = encoder(**tok_p).last_hidden_state[:, 0, :]
        loss = info_nce_loss(z_a, z_p, temperature=0.07)
        loss.backward()
        optimizer.step()
```

Output: A PubMedBERT-based encoder where "Arachnodactyly", "Spider fingers", and "Long slender fingers" cluster together, and nearby in the space to their parent "Abnormality of finger".

---

**Example 2: Adding Knowledge Distillation to Multimodal Pretraining**

User: "I already have a trained phenotype knowledge encoder. How do I distill its structure into a CLIP-style medical VLM?"

Approach:
1. Freeze the knowledge encoder as teacher
2. Initialize student text encoder from teacher weights
3. Train with combined contrastive + distillation loss

```python
class PhenoLIPTrainer:
    def __init__(self, vision_encoder, text_encoder, knowledge_encoder, lambda_kd=1.0):
        self.vision_enc = vision_encoder      # ViT-B, trainable
        self.text_enc = text_encoder           # PubMedBERT, trainable
        self.knowledge_enc = knowledge_encoder # Frozen teacher
        self.knowledge_enc.eval()
        for p in self.knowledge_enc.parameters():
            p.requires_grad = False
        self.lambda_kd = lambda_kd
        self.tau = 0.07

    def compute_loss(self, images, captions):
        # Encode all modalities
        v = F.normalize(self.vision_enc(images), dim=-1)       # [B, D]
        t = F.normalize(self.text_enc(captions), dim=-1)       # [B, D]
        with torch.no_grad():
            k = F.normalize(self.knowledge_enc(captions), dim=-1)  # [B, D]

        # L_m: bidirectional image-text contrastive
        logits_it = v @ t.T / self.tau
        labels = torch.arange(v.size(0), device=v.device)
        l_m = (F.cross_entropy(logits_it, labels) +
               F.cross_entropy(logits_it.T, labels)) / 2

        # L_kd: student text aligned to teacher knowledge
        logits_kd = t @ k.T / self.tau
        l_kd = (F.cross_entropy(logits_kd, labels) +
                F.cross_entropy(logits_kd.T, labels)) / 2

        return l_m + self.lambda_kd * l_kd
```

Output: A VLM where the text encoder retains ontology-structured knowledge (synonym awareness, hierarchical grouping) while also learning visual alignment. Zero-shot phenotype classification improves ~8-9% over BiomedCLIP.

---

**Example 3: Building a Phenotype Image Retrieval System**

User: "I want to search a medical image database by phenotype name and retrieve relevant clinical images."

Approach:
1. Encode all images with the PhenoLIP vision encoder into a FAISS index
2. Encode phenotype query with the text encoder
3. Retrieve top-K images by cosine similarity

```python
import faiss
import numpy as np

# Index construction (offline)
image_embeddings = []
for batch in image_dataloader:
    with torch.no_grad():
        v = F.normalize(vision_encoder(batch), dim=-1)
    image_embeddings.append(v.cpu().numpy())

image_embeddings = np.concatenate(image_embeddings, axis=0)  # [N, D]
index = faiss.IndexFlatIP(image_embeddings.shape[1])  # Inner product = cosine on normalized vectors
index.add(image_embeddings)

# Query (online)
def retrieve_images(phenotype_query, top_k=10):
    """Retrieve images for a phenotype name or description."""
    tok = tokenizer(phenotype_query, return_tensors="pt")
    with torch.no_grad():
        q = F.normalize(text_encoder(**tok).last_hidden_state[:, 0, :], dim=-1)
    scores, indices = index.search(q.cpu().numpy(), top_k)
    return indices[0], scores[0]

# Example: search for a specific phenotype
results, scores = retrieve_images("Arachnodactyly")
# Returns image indices ranked by relevance, with ontology-aware matching
# that also surfaces images tagged with synonyms like "Spider fingers"
```

Output: A retrieval system where querying "Arachnodactyly" returns relevant clinical images even if their captions only mention "long slender fingers" or "spider-like digits" — because the ontology-aware encoder understands these are the same phenotype.

## Best Practices

- **Do:** Initialize the Stage 1 knowledge encoder from a domain-specific language model (PubMedBERT, SciBERT) rather than general-purpose BERT. Biomedical pretraining matters for phenotype term understanding.
- **Do:** Include all four ontology attribute types as positive pairs — name, definition, synonyms, and hierarchical relations. Ablation in PhenoLIP shows each contributes to downstream performance.
- **Do:** Apply strict document-level splits (by paper ID or figure ID) when constructing train/test sets to prevent data leakage from different subfigures of the same compound figure.
- **Do:** Use the frozen teacher architecture for knowledge distillation rather than joint training. Freezing prevents the knowledge encoder from drifting toward the visual distribution and losing ontology structure.
- **Avoid:** Skipping the image filtering step during knowledge graph construction. Medical literature contains many non-photographic figures (flowcharts, bar charts, schematics) that add noise. DINOv2 clustering + manual review is essential.
- **Avoid:** Setting the distillation weight `lambda` too high. If `L_kd` dominates, the model over-relies on text structure and under-learns visual features. Start with `lambda=1.0` and tune on a validation retrieval task.

## Error Handling

- **Ontology parsing failures:** HPO OBO files can have inconsistent formatting across versions. Validate that every node has at least a name and one parent (except the root). Fall back to the latest stable HPO release if edge cases appear.
- **Subfigure detection errors:** Compound medical figures with overlapping panels cause false splits. If DAB-DETR confidence is below threshold (~0.5), treat the entire figure as a single image rather than splitting.
- **Caption-phenotype mapping ambiguity:** A caption mentioning "blue sclera" could map to multiple HPO terms. Use the most specific (deepest) matching term in the ontology hierarchy. If multiple leaf terms match, create one training pair per phenotype.
- **Embedding collapse during Stage 2:** If the distillation loss drives all text embeddings toward a small region, reduce `lambda_kd` or add gradient clipping. Monitor the uniformity of text embeddings (mean pairwise cosine similarity should stay below 0.3).
- **Out-of-ontology phenotypes at inference:** The model can still process free-text queries not in HPO, but retrieval quality degrades for concepts far from any training phenotype. Log these queries and consider ontology expansion.

## Limitations

- **Ontology dependency:** The approach requires a well-structured phenotype ontology with definitions and hierarchy. Domains without mature ontologies (e.g., emerging diseases) will not benefit from Stage 1 knowledge encoding.
- **English-only:** PhenoKG and PubMedBERT are English-centric. Clinical phenotype descriptions in other languages require separate ontology resources and multilingual encoders.
- **Static knowledge:** The ontology knowledge is frozen at pretraining time. New phenotypes added to HPO after training are not represented in the knowledge encoder without retraining Stage 1.
- **Image scope:** PhenoKG is built from PubMed Central figures, which skew toward published research images. Clinical point-of-care photos (phone images, EHR screenshots) have a significant domain gap that may require additional fine-tuning.
- **Rare phenotype coverage:** Despite 3,000+ phenotypes in PhenoKG, HPO contains ~19,700 total terms. Ultra-rare phenotypes with few published images remain poorly represented.

## Reference

**Paper:** [PhenoLIP: Integrating Phenotype Ontology Knowledge into Medical Vision-Language Pretraining](https://arxiv.org/abs/2602.06184v1) (Liang et al., 2026). Look for: Section 3 (PhenoKG construction pipeline), Section 4.1 (knowledge-enhanced embedding loss `L_k`), Section 4.2 (teacher-guided distillation loss `L_kd`), and Section 5 (PhenoBench evaluation protocol with document-level splits).