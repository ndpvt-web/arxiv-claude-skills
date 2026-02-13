---
name: "evaluation-oncotimia-system-supporting"
description: "Build RAG pipelines that transform unstructured clinical or domain-specific documents into structured form records using a multi-layer data lake, hybrid relational+vector storage, and rule-driven adaptive forms. Trigger phrases: 'build a clinical document extraction pipeline', 'convert unstructured reports to structured forms', 'RAG pipeline for medical records', 'automate form completion from documents', 'extract structured data from clinical notes', 'build a tumor board automation system'."
---

# Clinical Document-to-Structured-Form RAG Pipeline (ONCOTIMIA Architecture)

This skill enables Claude to design and implement end-to-end pipelines that ingest heterogeneous unstructured documents (clinical notes, radiology reports, pathology results, PDFs, DOCX files), store them in a multi-layer data lake with hybrid relational and vector storage, and use retrieval-augmented generation with rule-driven adaptive form logic to automatically populate structured records. The architecture follows the ONCOTIMIA system's proven approach of achieving 80% field-level accuracy on complex oncology forms with sub-30-second latency.

## When to Use

- When the user needs to extract structured fields from unstructured clinical documents (discharge summaries, pathology reports, radiology reads)
- When building a RAG pipeline that must fill out conditional, multi-block forms where later sections depend on earlier answers
- When designing a data lake with Landing/Staging/Refined layers for document ingestion and transformation
- When the user wants to evaluate multiple LLMs on a structured extraction task with accuracy and latency benchmarks
- When converting free-text domain reports (legal, insurance, medical) into standardized structured records
- When implementing hybrid PostgreSQL + vector database (Qdrant) storage for both structured queries and semantic retrieval
- When the user asks to automate documentation workflows that currently require manual expert review

## Key Technique

The ONCOTIMIA architecture solves the problem of transforming messy, heterogeneous documents into precise structured records through three interlocking components. First, a **three-layer data lake** (Landing, Staging, Refined) preserves raw documents for auditability, applies ETL normalization in the middle tier, and produces query-optimized representations at the top. Documents flow through format validation, content extraction via LangChain loaders (PyPDFLoader, Docx2txtLoader), text cleaning/tokenization/lemmatization, and then split into two paths: unstructured narrative text gets embedded via the Nomic model and stored in Qdrant, while structured fields go into PostgreSQL.

Second, the **RAG pipeline** works per-form-block rather than per-form. For each logical block of the target form, the system constructs a query from the block's field schema, retrieves the most relevant text fragments from the vector store, and injects them into a structured prompt template containing explicit field definitions, data types, and completion instructions. This block-level granularity is critical -- it ensures the LLM receives focused, relevant context rather than the entire patient record, and each generated field maintains traceability back to its source fragments.

Third, the **rule-driven adaptive form model** implements conditional logic where Block 1 (core variables) determines which downstream blocks activate. For example, if "previous neoplasia" is true, Block 2 activates; if "treatment refusal" is true, Block 3 activates. This prevents the LLM from hallucinating answers to irrelevant sections and reduces token waste. The combination of block-level RAG with rule-based flow control is what distinguishes this from naive "dump everything into the prompt" approaches.

## Step-by-Step Workflow

1. **Define the target form schema as a JSON structure** with logical blocks, where each block contains field definitions (name, data type, allowed values, description) and activation rules referencing fields in earlier blocks. Model conditional transitions explicitly -- e.g., `block_2.activated_when: "block_1.previous_neoplasia == true"`.

2. **Build the three-layer data lake ingestion pipeline**:
   - Landing layer: Accept raw files (PDF, DOCX, TXT, JSON) and store them unchanged with metadata (source, timestamp, format, hash).
   - Staging layer: Run format-specific extractors (PyPDFLoader for PDFs, Docx2txtLoader for Word docs) via LangChain, then apply text cleaning, tokenization, and normalization.
   - Refined layer: Split output into structured fields (to PostgreSQL) and narrative text chunks (to vector store).

3. **Configure the vector store** (Qdrant or pgvector) with Nomic embeddings (or another open-source embedding model). Segment clinical narratives into semantically coherent chunks -- split on section headers or paragraph boundaries rather than fixed token counts. Store each chunk with metadata linking it back to the source document and section.

4. **Configure the relational store** (PostgreSQL) with tables for patient demographics, coded diagnoses, staging data, and structured lab values. This serves as the analytical source of truth for fields that come from structured sources.

5. **Implement per-block RAG retrieval**: For each form block, generate a retrieval query from the block's field names and descriptions. Retrieve the top-k most relevant chunks from the vector store. Combine retrieved chunks with any structured data from PostgreSQL that relates to the block.

6. **Construct block-level prompt templates** containing: (a) the target fields with their data types and allowed values, (b) retrieved context fragments, (c) explicit instructions to return only the requested fields in a specified JSON format, and (d) instructions to return `null` for fields with insufficient evidence rather than guessing.

7. **Execute the adaptive form flow**: Process Block 1 first. Evaluate its output against the rule engine to determine which downstream blocks are activated. Process only activated blocks, skipping irrelevant ones. This is sequential by necessity -- downstream blocks depend on upstream outputs.

8. **Deploy via an LLM abstraction layer** (AWS Bedrock, or a local gateway) that normalizes request/response formats across models, enforces rate limits, logs all interactions for auditability, and supports swapping models without changing application code.

9. **Validate outputs** by running a secondary LLM pass (or rule-based checks) to verify: field values are within allowed ranges, categorical fields match the allowed set, Boolean fields triggered correct downstream block activations, and numerical fields (like PD-L1 percentages) fall within plausible ranges.

10. **Benchmark with a ground-truth evaluation set**: Score each model on field-level accuracy (percentage of correctly completed fields) and end-to-end latency per form. Report mean accuracy and standard deviation across cases to assess stability.

## Concrete Examples

**Example 1: Building a lung cancer tumor board form pipeline**

User: "I have PDFs of clinical notes, pathology reports, and radiology reads for lung cancer patients. I need to automatically fill out a tumor board form with fields like histology type, molecular markers, staging, ECOG status, and treatment history."

Approach:
1. Define the form schema with 7 blocks:
```json
{
  "blocks": [
    {
      "id": "block_1",
      "name": "core_clinical",
      "fields": [
        {"name": "smoking_status", "type": "categorical", "values": ["smoker", "non-smoker", "ex-smoker"]},
        {"name": "ecog", "type": "integer", "min": 0, "max": 5},
        {"name": "histology", "type": "categorical", "values": ["adenocarcinoma", "squamous_cell", "large_cell", "small_cell"]},
        {"name": "molecular_markers", "type": "object", "fields": ["EGFR", "ALK", "KRAS", "BRAF", "ROS1"]},
        {"name": "pdl1_value", "type": "float", "unit": "percent"},
        {"name": "previous_neoplasia", "type": "boolean"},
        {"name": "treatment_refusal", "type": "boolean"},
        {"name": "recurrence", "type": "boolean"}
      ],
      "activation": "always"
    },
    {
      "id": "block_2",
      "name": "previous_neoplasms",
      "activation": "block_1.previous_neoplasia == true",
      "fields": [{"name": "neoplasm_type", "type": "string"}, {"name": "year_diagnosed", "type": "integer"}]
    }
  ]
}
```

2. Ingest documents through the three-layer pipeline:
```python
from langchain.document_loaders import PyPDFLoader, Docx2txtLoader
from qdrant_client import QdrantClient
from sentence_transformers import SentenceTransformer

# Landing: store raw files with metadata
landing_record = {"file_path": path, "sha256": hash_file(path), "ingested_at": datetime.utcnow()}

# Staging: extract and clean text
loader = PyPDFLoader(path) if path.endswith(".pdf") else Docx2txtLoader(path)
documents = loader.load()
cleaned = [clean_and_normalize(doc.page_content) for doc in documents]

# Refined: embed and store
embedder = SentenceTransformer("nomic-ai/nomic-embed-text-v1.5")
chunks = segment_by_section_headers(cleaned)
vectors = embedder.encode(chunks)
qdrant.upsert(collection_name="clinical_docs", points=[
    {"id": i, "vector": v, "payload": {"text": c, "source_file": path, "section": s}}
    for i, (v, c, s) in enumerate(zip(vectors, chunks, sections))
])
```

3. Run block-level RAG and adaptive form completion:
```python
form_result = {}
for block in schema["blocks"]:
    if not evaluate_activation_rule(block["activation"], form_result):
        continue
    query = build_retrieval_query(block["fields"])
    context = qdrant.search(collection_name="clinical_docs", query_vector=embedder.encode(query), limit=5)
    structured_data = pg_query(block["fields"])  # pull any structured fields from PostgreSQL
    prompt = render_block_prompt(block, context, structured_data)
    response = bedrock_client.invoke_model(model_id=selected_model, body=prompt)
    form_result[block["id"]] = parse_and_validate(response, block["fields"])
```

Output: A JSON form with all fields populated, null values for insufficient evidence, and source traceability per field.

**Example 2: Evaluating multiple LLMs for extraction accuracy**

User: "I want to benchmark 4 different LLMs on my document extraction task to pick the best one."

Approach:
1. Prepare a ground-truth dataset -- manually annotated forms for 10+ cases.
2. Run each model through the identical RAG pipeline:
```python
import time

results = {}
for model_id in ["pixtral-large", "qwen3-32b", "gpt-oss-120b", "mistral-large"]:
    model_results = []
    for case in test_cases:
        start = time.time()
        predicted_form = run_pipeline(case["documents"], model_id=model_id)
        latency = time.time() - start
        accuracy = compute_field_accuracy(predicted_form, case["ground_truth"])
        model_results.append({"accuracy": accuracy, "latency_sec": latency})
    results[model_id] = {
        "mean_accuracy": np.mean([r["accuracy"] for r in model_results]),
        "std_accuracy": np.std([r["accuracy"] for r in model_results]),
        "mean_latency": np.mean([r["latency_sec"] for r in model_results]),
    }
```

3. Report results as a comparison table and select based on accuracy-latency tradeoff.

Output:
```
| Model            | Mean Accuracy | Std Dev | Mean Latency (s) |
|------------------|--------------|---------|-------------------|
| pixtral-large    | 80.0%        | 6.6     | 21.2              |
| qwen3-32b        | 79.3%        | 5.3     | 20.8              |
| gpt-oss-120b     | 79.3%        | 7.1     | 20.5              |
| mistral-large    | 79.0%        | 6.0     | 20.1              |
```

**Example 3: Adapting the pipeline for insurance claim processing**

User: "I want to use this same architecture to extract structured fields from insurance claim documents -- adjuster notes, medical records, and police reports."

Approach:
1. Define a claim form schema with conditional blocks:
   - Block 1: Claim basics (claimant info, incident date, claim type)
   - Block 2: Medical details (activated when claim_type involves injury)
   - Block 3: Vehicle details (activated when claim_type is auto)
   - Block 4: Property details (activated when claim_type is property)
2. Reuse the same Landing/Staging/Refined data lake pattern.
3. Swap the embedding model and retrieval queries to match insurance terminology.
4. Adjust prompt templates to reference insurance field schemas instead of oncology ones.

The architecture is domain-agnostic -- only the form schema, terminology, and validation rules change.

## Best Practices

- **Do:** Process forms block-by-block with separate RAG retrievals per block. This focuses the LLM's context window on relevant information and improves field accuracy compared to dumping all documents into one prompt.
- **Do:** Implement activation rules as explicit code (not LLM-generated decisions) so conditional form flow is deterministic and auditable.
- **Do:** Store raw documents unchanged in the Landing layer -- you will need them for debugging extraction failures and reprocessing when models improve.
- **Do:** Return `null` for fields where the source documents lack sufficient evidence rather than allowing the LLM to guess. An empty field is better than a hallucinated one in clinical/regulatory contexts.
- **Avoid:** Using fixed-token chunking for clinical documents. Section-aware chunking (splitting on headers like "DIAGNOSIS:", "FINDINGS:", "TREATMENT:") preserves semantic coherence and improves retrieval relevance.
- **Avoid:** Evaluating extraction accuracy at the form level only. Field-level accuracy reveals which specific field types the model struggles with (e.g., numerical values vs. categorical vs. free-text), enabling targeted prompt improvements.
- **Avoid:** Processing all form blocks in parallel when downstream blocks depend on upstream outputs. The adaptive flow is inherently sequential -- Block 1 must complete before determining which of Blocks 2-7 activate.

## Error Handling

- **Empty retrieval results**: When the vector store returns no relevant chunks for a block, fall back to structured data from PostgreSQL. If both are empty, set all block fields to `null` and flag the block for manual review.
- **LLM returns malformed JSON**: Wrap the parsing step in a retry loop (2-3 attempts) with an explicit "fix this JSON" re-prompt. If retries fail, log the raw output and mark the block as incomplete.
- **Activation rule evaluation failure**: If a field referenced in an activation rule is `null`, treat the rule as false (block not activated) rather than raising an error. Log a warning for manual review.
- **Latency spikes**: Set per-block timeouts (e.g., 30 seconds). If a model consistently exceeds the timeout, switch to a faster model for that block or reduce the number of retrieved chunks.
- **Embedding model mismatch**: Ensure the same embedding model is used for both indexing and querying. Swapping models without re-indexing produces meaningless similarity scores.
- **Document format failures**: When a PDF is scanned/image-based and PyPDFLoader extracts no text, fall back to an OCR step (Tesseract or cloud OCR) before proceeding to the Staging layer.

## Limitations

- The ONCOTIMIA evaluation used synthetic cases in Spanish -- real-world clinical text contains more noise, abbreviations, and inconsistencies that may reduce accuracy below the reported 80%.
- Rule-driven adaptive forms require upfront schema design by domain experts. The system does not learn form structure automatically.
- The approach assumes documents are already digitized. Handwritten notes or poorly scanned documents require an OCR preprocessing step not covered in the paper's architecture.
- Field-level accuracy was reported in aggregate, not broken down by field type. Some field types (e.g., precise numerical values like PD-L1 percentages) are likely harder to extract than binary or categorical fields.
- The 10-case evaluation set is small. Production deployment requires larger validation sets (50+ cases) to reliably estimate accuracy and detect edge cases.
- The sequential block processing introduces latency proportional to the number of activated blocks. For forms with many blocks, total completion time may become clinically impractical.

## Reference

**Paper**: [Evaluation of Oncotimia: An LLM based system for supporting tumour boards](https://arxiv.org/abs/2601.19899v1) -- Lorenzo et al., 2026. Key sections: Section III (System Architecture) for the three-layer data lake and hybrid storage design; Section IV (Evaluation) for the 6-model benchmark methodology; Table I for per-model accuracy and latency results.