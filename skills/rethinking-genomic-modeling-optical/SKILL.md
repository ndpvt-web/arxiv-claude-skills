---
name: "rethinking-genomic-modeling-optical"
description: "Implement OpticalDNA-style pipelines that render DNA sequences as 2D visual layouts and process them with OCR-capable vision-language models instead of sequential tokenizers. Use when: 'render DNA as an image', 'compress long genomic sequences', 'build a visual genomic encoder', 'OCR for DNA', 'reduce tokens for genome analysis', 'vision-based genomic modeling'."
---

# OpticalDNA: Vision-Based Genomic Modeling via OCR-Style Document Understanding

This skill enables Claude to help users build genomic analysis pipelines that treat DNA sequences as rendered 2D documents rather than 1D token streams. Based on the OpticalDNA framework, the core idea is to rasterize nucleotide sequences (A/C/G/T) onto fixed-resolution image pages using monospace fonts, then process them with a frozen vision encoder (SAM-Conv-CLIP-L) fused into a compact set of visual tokens, decoded by an autoregressive language model. This approach achieves ~20x token compression over sequential baselines and handles sequences up to 450 kilobases while training only 256k parameters for downstream tasks.

## When to Use

- When a user needs to process long genomic sequences (>10kb) and hits context-length or token-budget limits with standard nucleotide tokenizers
- When building a pipeline that renders DNA into images for visual model consumption (e.g., combining PIL/Pillow rendering with a vision-language model)
- When implementing any of the four OpticalDNA genomic primitives: sequence reading/transcription, region grounding, subsequence retrieval, or masked span completion
- When the user wants to compress genomic input for classification tasks (eQTL prediction, subspecies classification, phenotype regression) using frozen visual representations
- When designing a multi-page document representation for very long sequences (100k-450k bases) with per-nucleotide bounding box annotations
- When exploring OCR-inspired approaches to bioinformatics problems where spatial layout carries structural meaning

## Key Technique

**Why vision instead of language?** Traditional genomic foundation models (DNABERT, HyenaDNA, Evo) tokenize DNA as 1D sequences of nucleotides or k-mers. But genomic semantics are sparse and discontinuous — regulatory elements, exons, and functional motifs are scattered across vast stretches of low-information background. Sequential models waste computation reading through this background. OpticalDNA sidesteps this by rendering the sequence into a 2D visual layout and letting a vision encoder spatially attend to relevant regions, compressing the entire document into a fixed-size set of 100 visual tokens regardless of input length.

**The rendering pipeline.** Each DNA sequence is written character-by-character (uppercase A/C/G/T/N only) in a monospace font (size 14, line spacing 1.6) onto 640x640 pixel pages, left-to-right then top-to-bottom, yielding ~1,800 nucleotides per page. Every rendered character gets a bounding box annotation linking it to its global sequence index and pixel coordinates. For a 450kb sequence this produces ~250 pages. A frozen SAM-Conv-CLIP-L backbone extracts 16x16 patch features from each page, a learned projector aligns them to the decoder width, and a multi-head self-attention fusion module (20 heads, 1 layer) compresses all per-page tokens into exactly L=100 document-level visual tokens.

**Prompt-conditioned training over genomic primitives.** Rather than a single pretraining objective, OpticalDNA defines six prompted tasks that teach the model layout-aware genomic reasoning: (T1) free-form DNA transcription from the rendered image, (T2) transcription with bounding box grounding, (T3) region-of-interest recognition within user-specified coordinates, (T4) masked span completion given bounding boxes, (T5) subsequence retrieval with localization, and (T6) chromosome-level classification. The decoder (DeepSeek-3B MoE, 570M activated params) is fine-tuned with LoRA while the vision backbone stays frozen, and downstream tasks require only a 256k-parameter linear probe on the fused visual tokens.

## Step-by-Step Workflow

1. **Parse the input DNA sequence.** Read the FASTA/FASTQ file or raw string. Validate that it contains only A, C, G, T, N characters (case-insensitive). Uppercase all characters. Record total sequence length to determine how many pages are needed.

2. **Configure the rendering grid.** Set canvas size to 640x640 pixels, font to a monospace typeface at size 14, line spacing to 1.6x. Calculate characters per line and lines per page (~1,800 nucleotides per page). For a sequence of length S, compute P = ceil(S / 1800) pages.

3. **Render DNA onto image pages.** Using Pillow (PIL), create P blank white canvases. Write nucleotides sequentially left-to-right, top-to-bottom. For each character, record a bounding box tuple (x_min, y_min, x_max, y_max), its global sequence index, and its page number. Store these annotations in a structured format (JSON or a list of dataclass objects).

4. **Extract visual features with a frozen encoder.** Pass each 640x640 page through a SAM-Conv-CLIP-L vision backbone (or a comparable frozen ViT-L/14 if replicating). The Conv stage downsamples 16:1, producing patch-level feature maps. Do not update encoder weights.

5. **Project and fuse multi-page features.** Apply a learned linear projector to align patch features to the decoder's hidden dimension d. Concatenate per-page projected tokens into a single sequence, then pass through a multi-head self-attention fusion layer (20 heads, 1 layer) that compresses everything into exactly L=100 fixed-size document tokens Z.

6. **Define prompt-conditioned task objectives.** Construct training examples for each genomic primitive:
   - *Reading (T1)*: prompt = "Transcribe the DNA sequence"; target = full nucleotide string.
   - *Grounding (T2)*: prompt = "Transcribe with coordinates"; target = sequence + bounding boxes.
   - *ROI Recognition (T3)*: prompt = "Read nucleotides in region [x1,y1,x2,y2]"; target = subsequence in that box.
   - *Completion (T4)*: prompt = "Fill masked region [x1,y1,x2,y2]"; target = original nucleotides in that span.
   - *Retrieval (T5)*: prompt = "Locate all occurrences of ACGTNNACGT"; target = list of coordinate pairs.
   - *Classification (T6)*: prompt = "Classify this document"; target = label (e.g., chromosome ID, tissue type).

7. **Train the decoder with LoRA.** Attach LoRA adapters to the autoregressive decoder (e.g., DeepSeek-3B or any MoE/dense LM). Freeze the vision backbone entirely. Train with teacher forcing on assistant response tokens across all six task types. Use standard cross-entropy loss.

8. **Extract representations for downstream tasks.** After pretraining, take the L=100 fused visual tokens Z as the sequence representation. Attach a linear probe (256k params) or shallow MLP (1.3M-2.3M params) for downstream classification/regression. Fine-tune only this head.

9. **Evaluate with task-specific metrics.** Use Exact Match and Character Similarity for transcription tasks, IoU at 0.99 threshold for grounding tasks, AUROC for classification, and RMSE for regression phenotype tasks.

10. **Optimize for inference on very long sequences.** For sequences >100kb, batch pages through the frozen encoder in parallel (pages are independent), then fuse once. This keeps memory constant at L=100 tokens regardless of input length, enabling 450kb sequences in ~12 minutes versus hours for sequential models.

## Concrete Examples

**Example 1: Rendering a DNA sequence as a visual document**

User: "I have a 50kb DNA sequence from a FASTA file. Help me render it as images for a vision model."

Approach:
1. Read the FASTA file and extract the nucleotide string (50,000 characters).
2. Calculate pages needed: ceil(50000 / 1800) = 28 pages.
3. Render each page as a 640x640 PIL image with monospace font, collecting bounding boxes.

Output:
```python
from PIL import Image, ImageDraw, ImageFont
import json

def render_dna_pages(sequence: str, page_size=640, font_size=14, line_spacing=1.6):
    sequence = sequence.upper().replace("\n", "")
    font = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf", font_size)

    # Measure character cell dimensions
    bbox = font.getbbox("A")
    char_w = bbox[2] - bbox[0]
    char_h = bbox[3] - bbox[1]
    line_h = int(char_h * line_spacing)

    chars_per_line = page_size // char_w
    lines_per_page = page_size // line_h
    chars_per_page = chars_per_line * lines_per_page

    pages, annotations = [], []
    for p in range(0, len(sequence), chars_per_page):
        img = Image.new("RGB", (page_size, page_size), "white")
        draw = ImageDraw.Draw(img)
        page_annot = []
        chunk = sequence[p : p + chars_per_page]

        for i, ch in enumerate(chunk):
            row, col = divmod(i, chars_per_line)
            x = col * char_w
            y = row * line_h
            draw.text((x, y), ch, fill="black", font=font)
            page_annot.append({
                "char": ch,
                "global_idx": p + i,
                "bbox": [x, y, x + char_w, y + char_h],
                "page": len(pages),
            })
        pages.append(img)
        annotations.append(page_annot)

    return pages, annotations

# Usage
with open("sequence.fasta") as f:
    lines = f.readlines()
    seq = "".join(l.strip() for l in lines if not l.startswith(">"))

pages, annots = render_dna_pages(seq)
for i, page in enumerate(pages):
    page.save(f"dna_page_{i:03d}.png")
```

**Example 2: Building the fusion module for multi-page token compression**

User: "I have per-page CLIP features for 28 DNA pages. How do I fuse them into 100 fixed tokens?"

Approach:
1. Concatenate all per-page token sequences along the sequence dimension.
2. Pass through a learned projector to match decoder hidden size.
3. Apply multi-head self-attention with 100 learnable query tokens to produce fixed-size output.

Output:
```python
import torch
import torch.nn as nn

class DNADocumentFusion(nn.Module):
    def __init__(self, clip_dim=768, decoder_dim=2048, n_heads=20, n_output_tokens=100):
        super().__init__()
        self.projector = nn.Linear(clip_dim, decoder_dim)
        self.query_tokens = nn.Parameter(torch.randn(n_output_tokens, decoder_dim))
        self.cross_attn = nn.MultiheadAttention(
            embed_dim=decoder_dim, num_heads=n_heads, batch_first=True
        )
        self.norm = nn.LayerNorm(decoder_dim)

    def forward(self, page_features: list[torch.Tensor]) -> torch.Tensor:
        """
        page_features: list of P tensors, each [N_patches, clip_dim]
        Returns: [1, 100, decoder_dim] fused document tokens
        """
        # Concatenate all page tokens
        all_tokens = torch.cat(page_features, dim=0)  # [total_patches, clip_dim]
        projected = self.projector(all_tokens).unsqueeze(0)  # [1, total_patches, decoder_dim]

        # Cross-attend from learned queries to projected page tokens
        queries = self.query_tokens.unsqueeze(0)  # [1, 100, decoder_dim]
        fused, _ = self.cross_attn(queries, projected, projected)
        return self.norm(fused)  # [1, 100, decoder_dim]

# Usage: 28 pages, each with 1600 patches (640/16 = 40, 40*40=1600)
page_feats = [torch.randn(1600, 768) for _ in range(28)]
fusion = DNADocumentFusion()
doc_tokens = fusion(page_feats)  # [1, 100, 2048]
print(f"Compressed {28*1600} patch tokens -> {doc_tokens.shape[1]} document tokens")
# Output: Compressed 44800 patch tokens -> 100 document tokens
```

**Example 3: Downstream linear probe for eQTL classification**

User: "I have OpticalDNA embeddings (100 tokens x 2048 dims) for 10,000 genomic samples. How do I train a classifier for eQTL prediction?"

Approach:
1. Mean-pool the 100 tokens into a single 2048-dim vector per sample.
2. Attach a linear layer (2048 -> 1) with sigmoid for binary classification.
3. Train with BCE loss; this is the 256k-parameter linear probe setup.

Output:
```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset
from sklearn.metrics import roc_auc_score

class EQTLProbe(nn.Module):
    def __init__(self, dim=2048):
        super().__init__()
        self.pool = nn.AdaptiveAvgPool1d(1)
        self.head = nn.Linear(dim, 1)  # 2048 + 1 = 2049 params per output

    def forward(self, doc_tokens):
        # doc_tokens: [B, 100, 2048]
        pooled = doc_tokens.mean(dim=1)  # [B, 2048]
        return self.head(pooled).squeeze(-1)  # [B]

# Freeze everything except the probe
probe = EQTLProbe()
optimizer = torch.optim.Adam(probe.parameters(), lr=1e-3)
criterion = nn.BCEWithLogitsLoss()

# embeddings: [N, 100, 2048], labels: [N]
for epoch in range(50):
    probe.train()
    for batch_emb, batch_lbl in dataloader:
        logits = probe(batch_emb)
        loss = criterion(logits, batch_lbl.float())
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()

    # Evaluate
    probe.eval()
    with torch.no_grad():
        all_preds, all_labels = [], []
        for batch_emb, batch_lbl in val_loader:
            all_preds.append(torch.sigmoid(probe(batch_emb)))
            all_labels.append(batch_lbl)
        auroc = roc_auc_score(
            torch.cat(all_labels).numpy(), torch.cat(all_preds).numpy()
        )
    print(f"Epoch {epoch}: AUROC = {auroc:.4f}")
# Expected: AUROC ~0.85 matching OpticalDNA paper results
```

## Best Practices

- **Do:** Use a strictly monospace font (DejaVu Sans Mono, Courier New) so every nucleotide occupies identical pixel space — this is critical for accurate bounding box annotations and consistent patch alignment.
- **Do:** Keep the vision encoder frozen during pretraining and downstream fine-tuning. The compression power comes from the fusion module and projector, not from updating CLIP/SAM weights.
- **Do:** Train on all six genomic primitives simultaneously during pretraining. The multi-task objective is what teaches the model layout-aware reasoning, not any single task alone.
- **Do:** Use L=100 fused tokens as your default. The paper shows this is the sweet spot for compression-vs-fidelity; going lower degrades transcription accuracy.
- **Avoid:** Adding color coding, coordinate annotations, or auxiliary markers to the rendered images. OpticalDNA uses plain black-on-white uppercase characters only — embellishments add noise without information.
- **Avoid:** Processing pages sequentially through the encoder when you can batch them. Pages are independent until the fusion step, so parallel encoding is both correct and much faster.

## Error Handling

- **Font rendering inconsistency:** Different systems may render the same monospace font at slightly different pixel widths. Always measure `font.getbbox("A")` at runtime rather than hardcoding character dimensions. Assert that char_w is uniform across A/C/G/T/N.
- **Sequence contains non-ACGTN characters:** Strip or replace ambiguous IUPAC codes (R, Y, S, W, etc.) with N before rendering. Log the count of replacements as a warning.
- **Page count exceeds memory:** For sequences >450kb producing 250+ pages, process encoder forward passes in mini-batches of pages (e.g., 16 at a time) and accumulate patch features before fusion.
- **Bounding box misalignment:** If IoU between predicted and ground-truth boxes falls below 0.95 during validation, check that font size and line spacing exactly match training configuration (14px, 1.6x). Even 1px drift compounds across pages.
- **Decoder hallucination on long sequences:** If the decoder generates nucleotides not present in the rendered region, the fusion module may be under-trained. Increase pretraining steps on T1 (reading) and T3 (ROI recognition) tasks.

## Limitations

- Requires a pretrained vision encoder (CLIP-L or SAM) as the backbone — this is not a from-scratch genomic model, so users need access to these large vision models.
- The 640x640 rendering resolution and font size are fixed hyperparameters tuned for the paper's setup. Changing resolution requires re-deriving characters-per-page and retraining the fusion module.
- Amino acid or protein sequences are not directly supported — the rendering and training objectives are designed for 5-character DNA alphabet only.
- For sequences shorter than ~2kb, the overhead of rendering and visual encoding may outweigh the compression benefit compared to direct tokenization approaches like BPE on nucleotides.
- The approach has been validated on reference-quality genomic assemblies. Noisy long-read sequencing data with high error rates may degrade rendering fidelity and downstream accuracy.
- Multi-page fusion via self-attention has O(P^2) cost in the number of pages, which becomes a bottleneck beyond ~500 pages (~900kb sequences).

## Reference

**Paper:** [Rethinking Genomic Modeling Through Optical Character Recognition](https://arxiv.org/abs/2602.02014v1) (Xiang et al., 2026). Focus on Section 3 (method) for the rendering pipeline and fusion architecture, Section 4.1 for the six task definitions, and Tables 1-3 for benchmark comparisons showing 20x token compression with 256k trainable parameters.