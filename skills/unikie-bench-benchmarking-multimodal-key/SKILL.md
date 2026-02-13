---
name: "unikie-bench-benchmarking-multimodal-key"
description: "Extract structured key information from document images using schema-guided prompting for LMMs. Builds KIE pipelines that output normalized JSON from receipts, invoices, forms, medical records, tax documents, and more. Trigger phrases: 'extract fields from this document image', 'build a document KIE pipeline', 'parse invoice fields into JSON', 'extract key information from scanned documents', 'structured data extraction from document photos', 'benchmark document extraction accuracy'"
---

# Schema-Guided Key Information Extraction from Document Images

This skill enables Claude to build robust pipelines for extracting structured key information from document images (receipts, invoices, forms, tax records, medical documents, labels) using schema-guided prompting with Large Multimodal Models. The core technique, drawn from the UNIKIE-BENCH benchmark, formulates extraction as a single-pass structured prediction problem: given a document image and a JSON schema defining the target fields, the model fills in all values simultaneously rather than asking field-by-field questions. This produces more accurate, layout-aware extractions across diverse document types.

## When to Use

- When the user asks to extract specific fields (vendor name, total, date, line items) from a document image like a receipt or invoice
- When building an automated document processing pipeline that must output structured JSON from scanned or photographed documents
- When the user needs to define a reusable extraction schema for a category of documents (e.g., "all our purchase orders have these 12 fields")
- When evaluating how well an extraction system performs across document types using field-level F1 scoring
- When handling multilingual documents (English and Chinese) that require fullwidth-to-halfwidth normalization
- When extracting from documents with complex layouts: nested tables, multi-column forms, or documents with both header-level and line-item-level fields

## Key Technique: Schema-Guided Structured Prediction

Traditional document extraction treats each field as an independent question ("What is the total?", "What is the date?"). UNIKIE-BENCH demonstrates that **schema-guided structured prediction**--providing the full target schema upfront and asking the model to fill all fields in one pass--yields better results because it lets the model reason about field relationships and layout structure holistically.

The approach defines an extraction schema as a pair `(F, R)` where `F` is the set of target fields and `R` captures relationships among them (e.g., line items are a list of objects each containing description, quantity, and price). The schema is rendered as a JSON template with empty values, and the model is instructed to populate it from the document image. This is critically different from QA-style extraction: the model sees the complete structure it must fill, which anchors its attention to the right document regions.

The benchmark identifies two operating modes: **constrained-category KIE** where the schema is predefined per document scenario (e.g., all receipts share one schema), and **open-category KIE** where the schema is derived per-document for ad-hoc extraction. Both modes use the same prompting pattern but differ in how schemas are sourced. Top models achieve ~82% field-level F1 on constrained tasks but degrade significantly on long-tail fields, complex layouts, and open-category extraction--so robust normalization and error handling are essential.

## Step-by-Step Workflow

1. **Classify the document type and choose the extraction mode.** Determine whether this is a known document category (constrained: receipt, invoice, tax form) with a reusable schema, or an ad-hoc document (open: extract whatever key information is present). This determines whether you define the schema upfront or derive it from the document.

2. **Define the JSON extraction schema.** Create a JSON object with all target field names as keys and empty strings (or empty arrays for repeating groups) as values. For nested structures like line items, use an array containing a template object. Example for a receipt:
   ```json
   {
     "store_name": "",
     "store_address": "",
     "date": "",
     "total": "",
     "tax": "",
     "items": [{"description": "", "quantity": "", "unit_price": "", "amount": ""}]
   }
   ```

3. **Preprocess the document image.** Convert to RGB, resize if the image exceeds ~1.6 megapixels (to stay within model context limits), encode as JPEG at quality 95, then base64-encode for API transmission. Preserve aspect ratio during resizing.

4. **Construct the schema-guided prompt.** Use this template pattern:
   ```
   You are an information extraction expert. Extract key information from the
   document image and fill in the following JSON schema. Rules:
   - Output ONLY valid JSON matching the schema structure.
   - What you see is what you get: extract text exactly as it appears in the image.
   - Output language must be consistent with the document image.
   - For list fields, repeat the template element for each instance found.
   - If a field is not present in the document, use an empty string "".

   Schema:
   {schema_json}
   ```

5. **Send the image and prompt to the multimodal model.** Use a vision-capable model API (OpenAI-compatible, Gemini, Claude, etc.), sending the image as a base64 data URI alongside the text prompt. Set temperature to 0 for deterministic extraction.

6. **Parse the structured JSON response.** Strip any markdown code fences (` ```json ... ``` `) and `<think>` tags from the response. Attempt `json.loads()` first; if it fails, apply a JSON repair library (e.g., `json_repair`) as fallback. Log parse failures for debugging.

7. **Normalize extracted values.** Apply these normalization steps to both predictions and ground truth before comparison:
   - Convert fullwidth Unicode characters to halfwidth (e.g., `\uff10`-`\uff19` to `0`-`9`, fullwidth spaces to ASCII spaces)
   - Strip leading/trailing whitespace from all string values
   - Remove trailing periods
   - Normalize currency symbols and special characters (e.g., fullwidth `\u00a5` to standard yen sign)

8. **Evaluate extraction quality using field-level F1.** Flatten nested JSON structures into `(key_path, value)` tuples. Compute true positives (exact match on both key path and normalized value), false positives (predicted tuples not in ground truth), and false negatives (ground truth tuples not in predictions). Calculate micro-averaged F1 = 2*TP / (2*TP + FP + FN).

9. **Iterate on the schema for problematic fields.** UNIKIE-BENCH findings show that long-tail fields (rare keys) and complex layouts cause the most errors. Add field descriptions or examples to the schema for ambiguous fields. Split overly complex schemas into sub-schemas if accuracy drops.

10. **Build a reusable pipeline.** Wrap steps 3-7 into a function that takes `(image_path, schema)` and returns structured JSON. Add retry logic (up to 3 attempts on parse failure) and concurrent processing for batch extraction.

## Concrete Examples

**Example 1: Receipt Information Extraction**

User: "Extract structured data from this restaurant receipt image."

Approach:
1. Define a receipt schema with fields: store_name, address, date, time, items (list of name/qty/price), subtotal, tax, total, payment_method.
2. Preprocess the image (resize if needed, base64 encode).
3. Send schema-guided prompt with the image to a vision model.
4. Parse and normalize the JSON response.

Schema sent:
```json
{
  "store_name": "",
  "address": "",
  "date": "",
  "time": "",
  "items": [{"name": "", "quantity": "", "price": ""}],
  "subtotal": "",
  "tax": "",
  "total": "",
  "payment_method": ""
}
```

Output returned:
```json
{
  "store_name": "Golden Dragon Restaurant",
  "address": "123 Main St, Suite 4, New York, NY 10001",
  "date": "2025-11-15",
  "time": "19:32",
  "items": [
    {"name": "Kung Pao Chicken", "quantity": "1", "price": "14.99"},
    {"name": "Fried Rice", "quantity": "2", "price": "8.99"},
    {"name": "Spring Rolls", "quantity": "1", "price": "6.50"}
  ],
  "subtotal": "39.47",
  "tax": "3.50",
  "total": "42.97",
  "payment_method": "Visa ending 4532"
}
```

**Example 2: Batch Invoice Processing Pipeline**

User: "Build a pipeline to extract vendor, invoice number, date, line items, and total from a folder of invoice images."

Approach:
1. Define the invoice schema.
2. Write a Python function using asyncio for concurrent processing.
3. Process each image through the schema-guided extraction.
4. Collect results into a structured dataset.

```python
import asyncio
import base64
import json
from pathlib import Path
from openai import AsyncOpenAI

INVOICE_SCHEMA = {
    "vendor_name": "",
    "vendor_address": "",
    "invoice_number": "",
    "invoice_date": "",
    "due_date": "",
    "line_items": [{"description": "", "quantity": "", "unit_price": "", "amount": ""}],
    "subtotal": "",
    "tax": "",
    "total_due": "",
    "currency": ""
}

PROMPT_TEMPLATE = """You are an information extraction expert. Extract key information from the document image and fill in the following JSON schema.
Rules:
- Output ONLY valid JSON matching the schema structure.
- Extract text exactly as it appears in the image.
- Output language must match the document.
- Repeat list template elements for each instance found.
- Use empty string "" for fields not present.

Schema:
{schema}"""

async def extract_from_image(client, image_path: Path, schema: dict, model: str = "gpt-4o") -> dict:
    img_bytes = image_path.read_bytes()
    b64 = base64.b64encode(img_bytes).decode()
    prompt = PROMPT_TEMPLATE.format(schema=json.dumps(schema, indent=2))

    response = await client.chat.completions.create(
        model=model,
        temperature=0,
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": prompt},
                {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}}
            ]
        }]
    )

    text = response.choices[0].message.content
    # Strip markdown fences
    if "```json" in text:
        text = text.split("```json")[1].split("```")[0]
    return json.loads(text)

async def process_folder(folder: str, schema: dict, concurrency: int = 8):
    client = AsyncOpenAI()
    sem = asyncio.Semaphore(concurrency)
    images = list(Path(folder).glob("*.jpg")) + list(Path(folder).glob("*.png"))

    async def bounded_extract(img):
        async with sem:
            try:
                result = await extract_from_image(client, img, schema)
                return {"file": img.name, "status": "ok", "data": result}
            except Exception as e:
                return {"file": img.name, "status": "error", "error": str(e)}

    return await asyncio.gather(*[bounded_extract(img) for img in images])
```

**Example 3: Evaluating Extraction Accuracy**

User: "I have ground truth labels and model predictions for my document extraction. How do I compute field-level F1?"

Approach:
1. Flatten both prediction and ground truth JSON into (key_path, value) tuples.
2. Normalize all values.
3. Compute TP, FP, FN via set operations.

```python
import unicodedata

def fullwidth_to_halfwidth(text: str) -> str:
    result = []
    for char in text:
        code = ord(char)
        if 0xFF01 <= code <= 0xFF5E:  # Fullwidth ASCII variants
            result.append(chr(code - 0xFEE0))
        elif code == 0x3000:  # Fullwidth space
            result.append(' ')
        else:
            result.append(char)
    return ''.join(result)

def normalize(text: str) -> str:
    text = fullwidth_to_halfwidth(str(text))
    return text.strip().rstrip('.')

def flatten_dict(d, prefix=""):
    tuples = []
    if isinstance(d, dict):
        for k, v in sorted(d.items()):
            path = f"{prefix}.{k}" if prefix else k
            if isinstance(v, (dict, list)):
                tuples.extend(flatten_dict(v, path))
            else:
                normed = normalize(str(v))
                if normed:
                    tuples.append((path, normed))
    elif isinstance(d, list):
        for i, item in enumerate(d):
            tuples.extend(flatten_dict(item, f"{prefix}[{i}]"))
    return tuples

def compute_field_f1(prediction: dict, ground_truth: dict) -> dict:
    pred_tuples = set(flatten_dict(prediction))
    gt_tuples = set(flatten_dict(ground_truth))

    tp = len(pred_tuples & gt_tuples)
    fp = len(pred_tuples - gt_tuples)
    fn = len(gt_tuples - pred_tuples)

    precision = tp / (tp + fp) if (tp + fp) > 0 else 0.0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0.0
    f1 = 2 * precision * recall / (precision + recall) if (precision + recall) > 0 else 0.0

    return {"precision": precision, "recall": recall, "f1": f1, "tp": tp, "fp": fp, "fn": fn}
```

## Best Practices

**Do:**
- Always provide the complete schema in a single prompt rather than asking field-by-field questions. The model produces more accurate extractions when it can see the full structure and reason about field relationships.
- Normalize both predictions and ground truth identically before comparison. Fullwidth/halfwidth mismatches and trailing whitespace are the most common sources of false negatives.
- Use empty string `""` as the convention for absent fields rather than `null` or omitting the key. This keeps the output structure consistent and parseable.
- Include a template element inside list fields (e.g., `"items": [{"name": "", "qty": ""}]`) so the model knows the expected structure for repeating groups.

**Avoid:**
- Do not ask the model to explain or justify its extractions. The prompt should demand JSON-only output. Explanatory text surrounding the JSON causes parsing failures.
- Do not use schemas with more than ~30 top-level fields in a single pass. UNIKIE-BENCH shows accuracy degrades with schema complexity. Split into sub-schemas if needed.
- Do not skip image preprocessing. Oversized images consume unnecessary tokens and can get silently truncated, causing missed fields in lower regions of the document.
- Do not assume field order in the output matches the schema. Always access fields by key name, never by position.

## Error Handling

- **JSON parse failures**: Strip markdown fences and think tags first. Fall back to a JSON repair library (`json_repair` in Python, or regex-based brace balancing). If repair fails after 3 retries, log the raw response and return an empty schema with all fields set to `""`.
- **Missing fields in output**: The model may omit fields it cannot find. Post-process the output to ensure all schema keys exist, filling missing ones with `""`.
- **Hallucinated values**: The model may infer values not visually present (e.g., computing a total). Cross-validate numeric fields (do line items sum to subtotal?) and flag inconsistencies.
- **Image quality issues**: Low-resolution or skewed images degrade extraction. Recommend minimum 150 DPI. For skewed documents, apply deskewing (e.g., OpenCV) before extraction.
- **Encoding issues with multilingual documents**: Chinese, Japanese, and Korean documents frequently contain fullwidth characters. Always apply fullwidth-to-halfwidth normalization before string comparison.

## Limitations

- Field-level F1 requires exact character-level matches after normalization. Semantically equivalent but textually different values (e.g., "Nov 15, 2025" vs "2025-11-15") will be scored as mismatches. Date and number normalization can mitigate this but requires domain-specific rules.
- Open-category extraction (no predefined schema) is significantly harder--even the best models achieve ~15% lower F1 than constrained extraction. For ad-hoc documents, expect to iterate on the schema.
- Complex nested layouts (tables within tables, multi-page documents) remain a persistent challenge. Consider splitting multi-page documents into individual pages and merging results.
- The single-pass approach works best when the schema has fewer than ~30 fields. Beyond that, attention diffusion causes accuracy drops on long-tail fields.
- Handwritten documents have substantially lower extraction accuracy than printed ones. Set expectations accordingly and consider OCR preprocessing for heavily handwritten content.

## Reference

**Paper**: [UNIKIE-BENCH: Benchmarking Large Multimodal Models for Key Information Extraction in Visual Documents](https://arxiv.org/abs/2602.07038v1) -- Look for: the schema-guided prompting template (Appendix A.6), field-level F1 computation methodology, per-scenario performance breakdowns showing which document types and field categories are hardest, and the constrained vs. open-category evaluation protocol.