---
name: "gutenocr-grounded-vision-language-front-end"
description: "Build grounded OCR pipelines using GutenOCR's prompt-based interface for reading, detection, and spatial grounding on documents. Use when: 'extract text with bounding boxes from a PDF', 'find where a phrase appears in a scanned document', 'build a document OCR pipeline with spatial coordinates', 'detect text regions in business documents', 'locate specific content in a scientific paper image', 'read text from a cropped region of a document'."
---

# GutenOCR: Grounded Vision-Language Front-End for Documents

This skill enables Claude to design and implement document OCR pipelines that go beyond plain text extraction — producing spatially grounded outputs with bounding boxes, handling conditional location queries ("where is X?"), and performing region-constrained reading. The technique comes from GutenOCR, which fine-tunes Qwen2.5-VL (3B/7B) into a single-checkpoint model exposing reading, detection, and grounding through a unified prompt interface. Claude uses this skill to help users set up, prompt, and post-process GutenOCR models for real document intelligence workflows.

## When to Use

- When the user needs to extract text from document images **with spatial coordinates** (bounding boxes per line or paragraph)
- When building a pipeline to **locate specific phrases** within scanned documents (e.g., "find where the total amount appears on this invoice")
- When the user wants to **read text from a specific region** of a document image given a bounding box
- When designing a document processing system that needs **both OCR and layout detection** from a single model
- When the user asks to process business documents (invoices, IDs, claims) or scientific articles with structured output
- When implementing a search-over-documents feature that requires text-to-coordinate mapping
- When evaluating or comparing OCR systems using grounded metrics (CER with IoU matching)

## Key Technique

GutenOCR collapses what traditionally requires separate OCR, layout detection, and text grounding models into a single vision-language model checkpoint. By fine-tuning Qwen2.5-VL on business documents, scientific articles, and synthetic grounding data through a four-stage curriculum, the model learns to respond to natural-language prompts that select among four task modes: **full-page reading** (plain text or layout-preserving text2d), **full-page detection** (bounding box arrays without recognition), **conditional detection** ("where is X?" queries returning boxes for matching text), and **localized reading** (transcribing only within a specified bounding box region).

The critical design choice is the **prompt-based task routing**. Instead of separate model heads or APIs, the task is encoded entirely in the text prompt. The model outputs structured JSON with `{"text": "...", "bbox": [x1, y1, x2, y2]}` objects for grounded tasks, plain text for reading tasks, and coordinate arrays for detection-only tasks. Bounding boxes use pixel-space axis-aligned rectangles `[x1, y1, x2, y2]` relative to the original image dimensions. Training uses systematic prompt variation (mixing "this/that/the provided" with "document/image/scan") so the model handles natural phrasing at inference time.

The 3B variant excels at localized reading and pure detection (more efficient for high-throughput pipelines), while the 7B variant is stronger at global page reading and conditional detection. Both models were trained on 8x H100 GPUs with images rasterized at 72 DPI without cropping or tiling. The four curriculum stages progress from short-context synthetic data to long-form scientific articles, with Stage 3a (multi-source, 2-8K tokens) being the sweet spot before over-specialization sets in.

## Step-by-Step Workflow

1. **Install the model and dependencies.** Clone the GutenOCR repository from GitHub. Install `transformers`, `qwen-vl-utils`, and `torch`. Download the model weights from Hugging Face (`gutenocr-3b` for throughput-sensitive pipelines or `gutenocr-7b` for best global reading quality).

2. **Rasterize document inputs to images at 72 DPI.** Convert PDFs to images using `pdf2image` or `PyMuPDF` at 72 DPI — this matches training resolution. Do not crop or tile; feed full pages. Record the original image dimensions for bounding box interpretation.

3. **Select the task mode and construct the prompt.** Choose from the four task families based on the user's goal:
   - **Full-page reading**: `"Read all text in this document and return a single plain-text transcription"` — or request `text2d` format for layout-preserving output with whitespace gaps.
   - **Full-page detection**: `"Detect all lines in the image and return a JSON array of [x1, y1, x2, y2] boxes"` — returns coordinates only, no text.
   - **Conditional detection**: `"Given the query string '<phrase>', return bounding boxes for all matching lines in this document"` — for locating specific content.
   - **Localized reading**: `"Given the bounding box [x1, y1, x2, y2], read only the text inside that region"` — for region-constrained extraction.

4. **Run inference with the vision-language model.** Pass the image and constructed prompt to the model using the Qwen2.5-VL inference pipeline. Set `max_new_tokens` appropriately (2K for short documents, up to 16K for multi-page scientific articles).

5. **Parse the structured output.** For grounded tasks, parse the JSON output into a list of `{"text": str, "bbox": [int, int, int, int]}` objects. For detection-only tasks, parse the `[[x1,y1,x2,y2], ...]` array. For plain reading, use the raw text string. Validate that all bounding boxes satisfy `x1 < x2` and `y1 < y2`.

6. **Post-process bounding boxes for downstream use.** Convert pixel-space coordinates to normalized `[0,1]` ranges if needed by dividing by image width/height. Map coordinates back to original PDF space if the document was rasterized from PDF. Merge overlapping boxes if paragraph-level grouping is needed.

7. **Build the downstream application layer.** Use the grounded output for: search indexing (text + coordinates), visual annotation overlays, form field extraction (localized reading on detected regions), or document comparison (align boxes across versions).

8. **Evaluate using the grounded OCR protocol.** Compute CER/WER for text quality, F1@0.5 for detection accuracy (IoU >= 0.5), mCER@0.5 for end-to-end grounded recognition, and the composite score that averages all metrics on a [0,1] scale.

## Concrete Examples

**Example 1: Invoice field extraction with spatial grounding**

User: "I have scanned invoices and need to extract all text with bounding boxes, then find where the total amount appears."

Approach:
1. Rasterize the invoice image at 72 DPI
2. Run full-page grounded reading to get all text with line-level bounding boxes
3. Run a conditional detection query for the total amount value
4. Return both the full extraction and the located region

```python
from transformers import Qwen2_5_VLForConditionalGeneration, AutoProcessor
from qwen_vl_utils import process_vision_info
from PIL import Image
import json

model_name = "hunter-heidenreich/gutenocr-7b"
model = Qwen2_5_VLForConditionalGeneration.from_pretrained(model_name, torch_dtype="auto", device_map="auto")
processor = AutoProcessor.from_pretrained(model_name)

image = Image.open("invoice_scan.png")

# Step 1: Full-page grounded reading (lines with bounding boxes)
messages = [{"role": "user", "content": [
    {"type": "image", "image": image},
    {"type": "text", "text": "Read all text in this document. Return each line as a JSON object with 'text' and 'bbox' keys."}
]}]
inputs = processor.apply_chat_template(messages, return_tensors="pt", add_generation_prompt=True)
output = model.generate(**inputs, max_new_tokens=4096)
grounded_lines = json.loads(processor.batch_decode(output, skip_special_tokens=True)[0])

# Step 2: Conditional detection — locate the total
messages_locate = [{"role": "user", "content": [
    {"type": "image", "image": image},
    {"type": "text", "text": "Given the query string '$1,234.56', return bounding boxes for all matching lines in this document."}
]}]
inputs_locate = processor.apply_chat_template(messages_locate, return_tensors="pt", add_generation_prompt=True)
output_locate = model.generate(**inputs_locate, max_new_tokens=512)
total_boxes = json.loads(processor.batch_decode(output_locate, skip_special_tokens=True)[0])

print(f"Extracted {len(grounded_lines)} lines with coordinates")
print(f"Total amount located at: {total_boxes}")
```

Output:
```json
[
  {"text": "INVOICE #2024-0891", "bbox": [42, 15, 310, 38]},
  {"text": "Date: January 15, 2024", "bbox": [42, 45, 245, 65]},
  {"text": "Total: $1,234.56", "bbox": [320, 410, 520, 435]}
]
```

**Example 2: Region-constrained reading for form fields**

User: "I know the bounding box of a form field from a previous detection step. I need to read just the text inside that box."

Approach:
1. Take the known bounding box coordinates from the upstream detector
2. Use GutenOCR's localized reading mode to transcribe only that region
3. Return clean text without any surrounding content

```python
# Coordinates from a prior detection step (e.g., a name field on a form)
field_bbox = [85, 203, 340, 228]

messages = [{"role": "user", "content": [
    {"type": "image", "image": Image.open("application_form.png")},
    {"type": "text", "text": f"Given the bounding box {field_bbox}, read only the text inside that region of this document."}
]}]
inputs = processor.apply_chat_template(messages, return_tensors="pt", add_generation_prompt=True)
output = model.generate(**inputs, max_new_tokens=256)
field_text = processor.batch_decode(output, skip_special_tokens=True)[0]

print(f"Field content: '{field_text}'")
# Output: Field content: 'Jane A. Morrison'
```

**Example 3: Building a searchable document index**

User: "I want to make a batch of scanned scientific papers searchable — extract text with coordinates so users can click search results and jump to the right spot."

Approach:
1. Convert each PDF page to a 72 DPI image
2. Run grounded paragraph-level reading on each page
3. Store text + bounding boxes + page number in a search index
4. At query time, return matching text with page and coordinate metadata

```python
import fitz  # PyMuPDF
from pathlib import Path

def index_document(pdf_path: str) -> list[dict]:
    """Extract grounded text from every page for search indexing."""
    doc = fitz.open(pdf_path)
    index_entries = []

    for page_num in range(len(doc)):
        page = doc[page_num]
        pix = page.get_pixmap(dpi=72)
        img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)

        messages = [{"role": "user", "content": [
            {"type": "image", "image": img},
            {"type": "text", "text": "Read all text in this document. Return each paragraph as a JSON object with 'text' and 'bbox' fields."}
        ]}]
        inputs = processor.apply_chat_template(messages, return_tensors="pt", add_generation_prompt=True)
        output = model.generate(**inputs, max_new_tokens=8192)
        paragraphs = json.loads(processor.batch_decode(output, skip_special_tokens=True)[0])

        for para in paragraphs:
            index_entries.append({
                "page": page_num,
                "text": para["text"],
                "bbox": para["bbox"],
                "img_width": pix.width,
                "img_height": pix.height,
            })

    return index_entries

# Build index
entries = index_document("research_paper.pdf")
# Store in Elasticsearch, SQLite FTS, or any text search engine
# Each entry carries page + bbox for click-to-navigate functionality
```

## Best Practices

- **Do:** Rasterize at 72 DPI without cropping or tiling — this matches GutenOCR's training resolution and avoids artifacts from image preprocessing mismatches.
- **Do:** Use the 3B model for high-throughput batch pipelines (localized reading, detection) and the 7B model when global page reading quality matters most.
- **Do:** Vary your prompt phrasing naturally (e.g., "this document", "the attached scan", "the provided image") — the model was trained with prompt variation and handles it robustly.
- **Do:** Validate parsed JSON output and enforce `x1 < x2, y1 < y2` constraints on bounding boxes before passing to downstream systems.
- **Avoid:** Feeding formula-heavy pages (LaTeX-dense math) and expecting high accuracy — the paper documents degraded formula recognition (CDM drops from 0.936 to 0.866 at 3B).
- **Avoid:** Relying on color-guided OCR tasks (e.g., "read only the red text") — fine-tuning causes catastrophic forgetting on color-conditional reading (CER jumps from 0.109 to 0.963 at 7B).
- **Avoid:** Using GutenOCR for visually diverse non-document content (magazines, slides, photos with text) — training data is scoped to business documents and scientific articles.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Garbled or empty JSON output | Input image too low resolution or too large token count | Re-rasterize at exactly 72 DPI; split very long documents into individual pages |
| Bounding boxes exceed image dimensions | Coordinate hallucination on unusual layouts | Clamp all coordinates to `[0, img_width]` and `[0, img_height]`; discard boxes with zero area |
| Conditional detection returns no matches | Query string doesn't exactly match OCR'd text | Run full-page reading first, then fuzzy-match against extracted text to find the actual string before querying |
| Degraded reading order on multi-column pages | Page-level linearization trade-off documented in the paper | Use `text2d` output format which preserves 2D spatial layout with whitespace, then apply column-detection heuristics in post-processing |
| Model outputs plain text instead of JSON | Prompt ambiguity between reading and grounded modes | Be explicit in the prompt: include "return as JSON" and specify "with 'text' and 'bbox' keys" |

## Limitations

- **Formula recognition is degraded.** Fine-tuning trades off LaTeX/math recognition quality versus the base Qwen2.5-VL. If your documents are formula-heavy, consider a dedicated math OCR model for equation regions.
- **Color-conditional reading is broken.** The fine-tuning causes catastrophic forgetting for prompts like "read only the blue text." Do not use GutenOCR for color-segmented extraction.
- **Page linearization order may differ** from traditional top-to-bottom, left-to-right OCR. Multi-column and complex layouts may produce unexpected reading order in plain text mode. Use `text2d` or line-level grounded output and reconstruct order spatially.
- **Training scope is business + scientific documents.** Performance on handwritten text, scene text, slides, magazine layouts, or artistic documents is not validated.
- **72 DPI is the training resolution.** Higher or lower DPI may degrade performance since the model's visual encoder was calibrated to this resolution.
- **Single-page inference only.** Multi-page reasoning must be handled by processing pages independently and aggregating results externally.

## Reference

**Paper:** [GutenOCR: A Grounded Vision-Language Front-End for Documents](https://arxiv.org/abs/2601.14490v2) — Heidenreich, Elliott, Dinica, Getachew (2026). Look for: Table 3 (composite grounded OCR scores), Section 4 (prompt templates and task definitions), Section 5.2 (ablation across curriculum stages), and Section 6 (trade-off analysis on Fox and OmniDocBench).