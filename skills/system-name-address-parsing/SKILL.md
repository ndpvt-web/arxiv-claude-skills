---
name: "system-name-address-parsing"
description: "Parse unstructured person names and addresses into a structured 17-field schema using prompt-driven extraction with layered validation. Use when the user asks to 'parse addresses', 'extract name fields', 'structure address data', 'normalize mailing records', 'split name and address into fields', or 'clean up contact data'."
---

# Structured Name and Address Parsing with LLM-Driven Extraction

This skill enables Claude to parse free-text person names and addresses into a consistent 17-field schema using a four-stage pipeline: input normalization, structured prompting, constrained decoding, and rule-based validation. Based on the Tarannum et al. (2026) framework, the approach achieves 99.8% exact-row accuracy without fine-tuning by decoupling generative parsing from deterministic validation. Each stage is independently testable, making errors traceable and corrections systematic.

## When to Use

- When the user provides raw name-and-address strings (from CSVs, databases, forms, OCR output) and needs them split into discrete fields
- When building an ETL pipeline that ingests unstructured contact records into a normalized relational schema
- When cleaning messy mailing list data that mixes formats, abbreviations, and multilingual entries
- When the user needs to validate parsed address components against postal standards (USPS state codes, ZIP formats, directional sets)
- When deduplicating contact records that first require consistent field-level representations
- When implementing a batch address parser in Python, Node.js, or any language that calls an LLM API

## Key Technique

The core insight is **stage decoupling**: normalization removes surface variability before the LLM ever sees the text, the prompt constrains the output format tightly, decoding parameters limit creative drift, and a post-hoc validation layer enforces correctness independently of the model. This separation means each stage can be debugged and improved in isolation. The LLM handles the hard semantic work (disambiguating "Washington" as a name vs. city vs. state) while deterministic rules handle everything that can be checked mechanically.

The 17-field schema covers: `record_id`, `prefix_title`, `first_name`, `middle_name`, `last_name`, `suffix`, `street_number`, `pre_directional`, `street_name`, `street_type`, `post_directional`, `unit_type`, `unit_number`, `city`, `state`, `postal_code`, `country`. Pipe-delimited output is used instead of CSV to avoid collisions with commas in names and addresses. Each record gets a confidence score derived from rule-pass counts, edit-distance agreement on a second parse, and token rarity -- enabling downstream systems to route only the 4% of low-confidence records to human review.

Constrained decoding is critical: temperature is clamped to 0.30, top-p to 0.90, and stop sequences terminate output exactly at record boundaries. A structural pre-check rejects any output row that doesn't have exactly 17 pipe-separated fields before validation even begins. One retry with tighter stops is allowed per failed batch.

## Step-by-Step Workflow

1. **Normalize input text.** Apply NFKC Unicode normalization, strip zero-width characters, collapse consecutive whitespace, and remove stray punctuation. Expand common abbreviations (Ave to Avenue, Blvd to Boulevard, St to Street). Standardize directionals to the canonical set: N, NE, E, SE, S, SW, W, NW. Normalize ordinal suffixes (1st to 1). Unify unit symbols (# to Unit).

2. **Separate records from free-text blocks.** Use postal code patterns (`\b\d{5}(-\d{4})?\b` for US ZIP), state lexica, and unit markers (Apt, Ste, Unit) as boundary signals to split concatenated blocks into individual records. Assign each record a stable `record_id`.

3. **Assemble the structured prompt.** Use this exact scaffold:
   ```
   Role: "You are an expert name-and-address parser."
   Task: "For each input line, output one pipe-delimited row with exactly
          17 fields in this order: record_id|prefix_title|first_name|
          middle_name|last_name|suffix|street_number|pre_directional|
          street_name|street_type|post_directional|unit_type|unit_number|
          city|state|postal_code|country. Use empty strings for unknown
          fields. Do not output headers or explanations."
   Rules: "Canonicalize street types to long form. Directionals must be
           one of {N,NE,E,SE,S,SW,W,NW}. State = USPS two-letter code.
           ZIP = 5 or 9 digits. Do not invent data."
   ```
   Include 1-2 worked examples in the prompt, then append the batch of input records.

4. **Batch records for inference.** Group normalized records into batches of 16. Number each record within the batch to ensure prompt-output alignment. Set decoding parameters: `temperature=0.30`, `top_p=0.90`, `top_k=50`, `max_tokens=1500`. Define stop sequences at record boundaries.

5. **Parse and structurally validate the raw output.** Split each output line on `|`. Reject any row that does not have exactly 17 fields. On structural failure, retry the batch once with tighter stop sequences. If the retry also fails, flag the batch for manual review.

6. **Apply canonicalization rules.** Map state fields to USPS two-letter codes. Normalize directionals (N.E. to NE). Convert street types to canonical long forms (Av to Avenue). Ensure postal codes are 5 or 9 digits.

7. **Run cross-field validation.** Check ZIP-state consistency (e.g., flag 02139 paired with CA). Verify that `street_name` is non-empty when `street_number` is present. Confirm `unit_number` is present when `unit_type` is set. Attach a clear failure reason to every rejected record.

8. **Compute confidence scores.** Combine: (a) count of validation rules passed, (b) edit-distance agreement if a second parse is run, (c) token rarity penalty for unusual forms. Bucket into High (90-100%), Medium (70-89%), Low (<70%).

9. **Export structured data.** Write valid records as JSON Lines or CSV with stable field ordering. Write a metadata sidecar file containing batch ID, timestamp, decoder settings, and validation summary. Route low-confidence records to a separate review queue.

10. **Generate a parsing report.** Summarize total records processed, pass/fail counts, field-level error distribution, and confidence histogram. This report enables the user to assess quality before downstream consumption.

## Concrete Examples

**Example 1: Single US address with name**

User: "Parse this into structured fields: Mr. John A. Smith Jr., 123-1/2 NE Main St Apt 4B, Phoenix AZ 85004"

Approach:
1. Normalize: collapse spaces, expand "St" to "Street"
2. Assemble prompt with the record numbered as `1.`
3. Parse LLM output into 17 fields

Output:
```
{
  "record_id": "rec_001",
  "prefix_title": "Mr.",
  "first_name": "John",
  "middle_name": "A.",
  "last_name": "Smith",
  "suffix": "Jr.",
  "street_number": "123-1/2",
  "pre_directional": "NE",
  "street_name": "Main",
  "street_type": "Street",
  "post_directional": "",
  "unit_type": "Apt",
  "unit_number": "4B",
  "city": "Phoenix",
  "state": "AZ",
  "postal_code": "85004",
  "country": ""
}
```

**Example 2: Batch of messy CSV data**

User: "I have a CSV with 500 rows of free-text addresses. Clean them up into a normalized table."

Approach:
1. Read the CSV. Apply NFKC normalization and abbreviation expansion to every row.
2. Split into batches of 16. Assemble the prompt scaffold for each batch.
3. Call the LLM for each batch with constrained decoding parameters.
4. Validate all 500 outputs: schema check (17 fields), canonicalization, cross-field rules.
5. Export a clean CSV with 17 columns plus a `confidence` column.
6. Write a separate `review_queue.csv` containing the ~4% low-confidence records.
7. Print a summary: "487 records parsed at high confidence, 13 flagged for review."

Output (first 2 rows of clean CSV):
```csv
record_id,prefix_title,first_name,middle_name,last_name,suffix,street_number,pre_directional,street_name,street_type,post_directional,unit_type,unit_number,city,state,postal_code,country,confidence
rec_001,,Maria,,Garcia,,456,,Oak,Avenue,,Apt,12,San Antonio,TX,78201,US,96
rec_002,Dr.,James,R.,Wilson,III,789,SW,Park,Boulevard,,,Portland,OR,97205,US,99
```

**Example 3: Building a Python parsing function**

User: "Write me a Python function that uses an LLM to parse name/address strings."

Approach:
1. Implement `normalize_input()` with NFKC, whitespace collapse, abbreviation expansion.
2. Implement `build_prompt(records)` using the 17-field scaffold with examples.
3. Implement `parse_batch(prompt)` calling the LLM with constrained parameters.
4. Implement `validate_record(fields)` checking field count, canonicalization, cross-field rules.
5. Implement `compute_confidence(record, validation_results)` returning a 0-100 score.
6. Wire them into `parse_addresses(raw_texts) -> list[dict]`.

Output (key function):
```python
import unicodedata
import re

SCHEMA_FIELDS = [
    "record_id", "prefix_title", "first_name", "middle_name", "last_name",
    "suffix", "street_number", "pre_directional", "street_name", "street_type",
    "post_directional", "unit_type", "unit_number", "city", "state",
    "postal_code", "country"
]

STREET_TYPE_MAP = {
    "st": "Street", "ave": "Avenue", "blvd": "Boulevard", "dr": "Drive",
    "ln": "Lane", "rd": "Road", "ct": "Court", "pl": "Place", "cir": "Circle",
    "pkwy": "Parkway", "ter": "Terrace", "way": "Way",
}

DIRECTIONALS = {"N", "NE", "E", "SE", "S", "SW", "W", "NW"}

def normalize_input(text: str) -> str:
    text = unicodedata.normalize("NFKC", text)
    text = re.sub(r"[\u200b-\u200f\u2028-\u202f]", "", text)  # zero-width
    text = re.sub(r"\s+", " ", text).strip()
    for abbr, full in STREET_TYPE_MAP.items():
        text = re.sub(rf"\b{abbr}\.?\b", full, text, flags=re.IGNORECASE)
    text = re.sub(r"\b([NSEW])\.([EW])\.\b", r"\1\2", text)  # N.E. -> NE
    text = re.sub(r"#\s*", "Unit ", text)
    return text

def validate_record(fields: dict) -> list[str]:
    errors = []
    if len(fields) != 17:
        errors.append(f"Expected 17 fields, got {len(fields)}")
    if fields.get("state") and len(fields["state"]) != 2:
        errors.append(f"State not 2-letter USPS code: {fields['state']}")
    if fields.get("postal_code") and not re.match(r"^\d{5}(-\d{4})?$", fields["postal_code"]):
        errors.append(f"Invalid ZIP format: {fields['postal_code']}")
    if fields.get("pre_directional") and fields["pre_directional"] not in DIRECTIONALS:
        errors.append(f"Invalid directional: {fields['pre_directional']}")
    if fields.get("street_number") and not fields.get("street_name"):
        errors.append("street_number present but street_name missing")
    if fields.get("unit_type") and not fields.get("unit_number"):
        errors.append("unit_type present but unit_number missing")
    return errors
```

## Best Practices

- **Do:** Always normalize input before sending to the LLM. Surface variability (extra spaces, abbreviations, Unicode artifacts) accounts for a large share of parsing errors. Cleaning it first lets the model focus on semantic disambiguation.
- **Do:** Use pipe delimiters (`|`) in the prompt output format rather than commas or tabs. Names and addresses frequently contain commas; pipes virtually never appear in address data.
- **Do:** Include 1-2 worked examples in every prompt. The examples anchor the model to the exact field order and formatting conventions. Pin instructions verbatim -- only template the input block and examples.
- **Do:** Validate independently of the model. Never trust LLM output without a deterministic check. The validation layer catches the ~0.2% of rows the model gets wrong.
- **Avoid:** Setting temperature above 0.30 for this task. Higher temperatures introduce creative variation that manifests as inconsistent field assignment and hallucinated data.
- **Avoid:** Batches larger than 16 records per prompt. Larger batches increase the chance of the model losing track of record boundaries and misaligning numbered outputs.

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| Wrong field count | Model merged or split fields | Retry batch once with tighter stop sequences; if still wrong, flag for review |
| ZIP-state mismatch | Ambiguous or incorrect input | Flag the record as low confidence; include both values for human review |
| Empty required field (e.g., `last_name`) | Input genuinely lacks the data | Accept empty string but lower confidence score; do not hallucinate a value |
| Non-ASCII in ZIP field | Unicode normalization missed an artifact | Re-run NFKC normalization on the specific field; reject if still non-digit |
| Batch timeout / API error | Network or rate limit | Retry with exponential backoff; log the failed batch ID for auditability |
| Model outputs headers or explanations | Prompt not strict enough | Add explicit negative instruction: "Do not output headers, explanations, or markdown formatting" |

## Limitations

- **US-centric schema.** The 17-field schema is designed around US postal conventions. International addresses with different structures (e.g., Japanese addresses with district/block/building ordering, or UK postcodes) may not map cleanly and require schema adaptation.
- **No fine-tuning.** The approach relies entirely on prompt engineering. Extremely domain-specific formats (military APO/FPO addresses, PO Boxes with unusual notation) may need additional few-shot examples in the prompt.
- **Confidence scoring is heuristic.** The three-signal confidence score (rule passes, edit-distance, token rarity) is lightweight and practical but not a calibrated probability. Do not treat it as a ground-truth reliability measure.
- **Batch size tradeoff.** Fixed 16-record batches balance throughput and accuracy but are not optimal for all input distributions. Very short records waste tokens; very long records may need smaller batches.
- **Single-pass extraction.** The system parses each record once (with one retry on failure). It does not perform iterative refinement or multi-model consensus, which could improve accuracy on the hardest cases at higher cost.

## Reference

Tarannum, A., Mohammed, M.A., Cakmak, M.C., Al Mandalawi, S., & Talburt, J. (2026). *A System for Name and Address Parsing with Large Language Models*. arXiv:2601.18014v1. https://arxiv.org/abs/2601.18014v1

Key takeaway: Section 3 (System Design) details the four-stage pipeline and the exact prompt scaffold; Table 5 lists the cross-field validation rules; Table 8 shows confidence calibration results proving that the three-tier bucketing reliably separates accurate from uncertain records.