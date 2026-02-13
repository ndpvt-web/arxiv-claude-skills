---
name: "sar-rag-atr-visual-question"
description: |
  Build image retrieval-augmented generation (ImageRAG) pipelines for visual recognition tasks.
  Combines vision embeddings, vector databases, and multimodal LLMs to classify or describe
  images by retrieving similar labeled exemplars before generation.

  Use this skill when:
  - "Build an image RAG pipeline for visual classification"
  - "Create a retrieval-augmented system for SAR target recognition"
  - "Set up a vector database of image embeddings for similarity search"
  - "Add a visual memory bank to improve image classification accuracy"
  - "Implement ImageRAG for visual question answering"
  - "Retrieve similar images from a database to help an MLLM identify objects"
---

# SAR-RAG: Image Retrieval-Augmented Generation for Visual Recognition

This skill teaches Claude to implement **ImageRAG** pipelines following the SAR-RAG architecture (Ramirez et al., 2026). The core idea: instead of asking a multimodal LLM to classify an image cold, first retrieve visually similar images with known labels from a vector database, then feed those exemplars alongside the query image into the MLLM. This "visual memory bank" pattern dramatically improves classification accuracy and enables dimensional regression on objects that would otherwise be ambiguous -- particularly for domain-specific imagery like SAR (synthetic aperture radar), medical scans, satellite photos, or industrial inspection.

## When to Use

- When the user wants to classify images using an MLLM but accuracy on fine-grained categories is insufficient without reference examples
- When building a system that must distinguish visually similar objects (e.g., vehicle types in radar imagery, cell types in microscopy, defect categories in manufacturing)
- When the user asks to create a visual memory bank or image exemplar library for few-shot visual reasoning
- When implementing retrieval-augmented generation where the "documents" are images rather than text
- When the user needs to combine vector similarity search with multimodal LLM prompting
- When building an agentic tool that searches an image database as part of its reasoning loop

## Key Technique

**The SAR-RAG architecture has three stages: Embed, Retrieve, Generate.**

In the **Embed** stage, a vision encoder (such as CLIP, SigLIP, or DINOv2) converts each reference image into a dense vector embedding. These embeddings, along with metadata (true label, measurements, source info), are stored in a vector database (FAISS, ChromaDB, Qdrant, or Pinecone). This constitutes the "ATR memory bank" -- a searchable library of labeled exemplars.

In the **Retrieve** stage, a query image is embedded using the same encoder, and cosine similarity search finds the top-k most similar reference images. The key insight from SAR-RAG is that semantic similarity in embedding space correlates with target category, so retrieved neighbors are likely to share the same class as the query. The retrieval step returns both the reference images and their associated metadata (known labels, physical dimensions, viewing angles).

In the **Generate** stage, the query image and the retrieved exemplars (with their labels) are composed into a structured prompt for a multimodal LLM. The MLLM performs visual comparison: "Here is the query image. Here are K similar reference images with known types [T1, T2, ...]. Based on visual similarity, identify the query target." This transforms zero-shot classification into a visually-grounded few-shot task, yielding measurably higher accuracy for both categorical classification and numeric attribute regression.

## Step-by-Step Workflow

### 1. Prepare the reference dataset

Organize labeled images into a structured directory or manifest file. Each entry needs: image path/bytes, ground-truth label, and any numeric attributes (dimensions, angle, frequency, etc.).

```python
import json

# Example manifest structure
manifest = [
    {
        "image_path": "data/references/bmp2_001.png",
        "label": "BMP-2",
        "attributes": {"length_m": 6.74, "width_m": 3.15, "depression_angle": 15}
    },
    # ... hundreds to thousands of entries
]
with open("data/reference_manifest.json", "w") as f:
    json.dump(manifest, f)
```

### 2. Select and load a vision encoder for embedding

Choose an encoder whose embedding space captures the visual distinctions that matter for your domain. CLIP is a strong general-purpose default; domain-fine-tuned models perform better for specialized imagery.

```python
import torch
from transformers import CLIPModel, CLIPProcessor

model_name = "openai/clip-vit-large-patch14"
model = CLIPModel.from_pretrained(model_name)
processor = CLIPProcessor.from_pretrained(model_name)

def embed_image(image_path: str) -> list[float]:
    from PIL import Image
    image = Image.open(image_path).convert("RGB")
    inputs = processor(images=image, return_tensors="pt")
    with torch.no_grad():
        embedding = model.get_image_features(**inputs)
    # L2-normalize for cosine similarity
    embedding = embedding / embedding.norm(dim=-1, keepdim=True)
    return embedding.squeeze().tolist()
```

### 3. Build the vector database (memory bank)

Embed all reference images and insert them into a vector store with their metadata. Use FAISS for local/offline workloads, ChromaDB for easy prototyping, or a managed service for production scale.

```python
import chromadb

client = chromadb.PersistentClient(path="./sar_rag_db")
collection = client.get_or_create_collection(
    name="reference_targets",
    metadata={"hnsw:space": "cosine"}
)

for i, entry in enumerate(manifest):
    embedding = embed_image(entry["image_path"])
    collection.add(
        ids=[f"ref_{i}"],
        embeddings=[embedding],
        metadatas=[{
            "label": entry["label"],
            "length_m": entry["attributes"]["length_m"],
            "width_m": entry["attributes"]["width_m"],
            "depression_angle": entry["attributes"]["depression_angle"],
            "image_path": entry["image_path"]
        }]
    )
```

### 4. Implement the retrieval function

Given a query image, embed it and perform top-k similarity search. Return both metadata and the actual reference images for the MLLM prompt.

```python
def retrieve_similar(query_image_path: str, k: int = 5) -> list[dict]:
    query_embedding = embed_image(query_image_path)
    results = collection.query(
        query_embeddings=[query_embedding],
        n_results=k,
        include=["metadatas", "distances"]
    )
    retrieved = []
    for meta, dist in zip(results["metadatas"][0], results["distances"][0]):
        retrieved.append({
            **meta,
            "similarity": 1 - dist  # ChromaDB returns distance, convert to similarity
        })
    return retrieved
```

### 5. Compose the augmented MLLM prompt

Build a structured prompt that presents the query image alongside retrieved exemplars. The prompt must clearly separate the reference context from the query, and instruct the MLLM to reason by visual comparison.

```python
def build_rag_prompt(query_path: str, retrieved: list[dict], task: str = "classify") -> dict:
    reference_descriptions = []
    reference_images = []
    for i, ref in enumerate(retrieved):
        reference_descriptions.append(
            f"Reference {i+1}: Label={ref['label']}, "
            f"Length={ref['length_m']}m, Width={ref['width_m']}m, "
            f"Similarity={ref['similarity']:.3f}"
        )
        reference_images.append(ref["image_path"])

    context_block = "\n".join(reference_descriptions)

    if task == "classify":
        instruction = (
            "You are an image recognition expert. I will show you a query image "
            "and several reference images with known labels.\n\n"
            f"REFERENCE CONTEXT:\n{context_block}\n\n"
            "The reference images are shown in order above. "
            "Compare the query image visually to the references. "
            "Based on shape, structure, and similarity scores, "
            "predict the most likely label for the query image. "
            "Explain your reasoning, then state your prediction."
        )
    elif task == "regress":
        instruction = (
            "You are a measurement estimation expert. I will show you a query image "
            "and reference images with known physical dimensions.\n\n"
            f"REFERENCE CONTEXT:\n{context_block}\n\n"
            "Using the known dimensions of the most similar references, "
            "estimate the length and width of the object in the query image."
        )

    return {"instruction": instruction, "image_paths": reference_images + [query_path]}
```

### 6. Call the multimodal LLM with images and prompt

Send the composed prompt with all images (references + query) to the MLLM. Use GPT-4o, Claude with vision, Gemini, or LLaVA depending on your setup.

```python
import base64, httpx

def encode_image_b64(path: str) -> str:
    with open(path, "rb") as f:
        return base64.standard_b64encode(f.read()).decode("utf-8")

def call_mllm(prompt_data: dict) -> str:
    content = [{"type": "text", "text": prompt_data["instruction"]}]
    for img_path in prompt_data["image_paths"]:
        content.append({
            "type": "image_url",
            "image_url": {"url": f"data:image/png;base64,{encode_image_b64(img_path)}"}
        })

    # Example using OpenAI-compatible API
    response = httpx.post(
        "https://api.openai.com/v1/chat/completions",
        headers={"Authorization": f"Bearer {API_KEY}"},
        json={
            "model": "gpt-4o",
            "messages": [{"role": "user", "content": content}],
            "max_tokens": 1024
        }
    )
    return response.json()["choices"][0]["message"]["content"]
```

### 7. Parse the MLLM output into structured predictions

Extract the classification label or regression values from the MLLM response.

```python
import re

def parse_classification(response: str, valid_labels: list[str]) -> str:
    # Check for explicit label mentions, preferring the last one (conclusion)
    for label in valid_labels:
        if label.lower() in response.lower().split("prediction")[-1]:
            return label
    # Fallback: find most-mentioned label
    counts = {l: response.lower().count(l.lower()) for l in valid_labels}
    return max(counts, key=counts.get)

def parse_regression(response: str) -> dict:
    length_match = re.search(r"length[:\s]*(\d+\.?\d*)\s*m", response, re.IGNORECASE)
    width_match = re.search(r"width[:\s]*(\d+\.?\d*)\s*m", response, re.IGNORECASE)
    return {
        "length_m": float(length_match.group(1)) if length_match else None,
        "width_m": float(width_match.group(1)) if width_match else None,
    }
```

### 8. Run the full SAR-RAG pipeline end-to-end

```python
def sar_rag_predict(query_image: str, task: str = "classify", k: int = 5) -> dict:
    retrieved = retrieve_similar(query_image, k=k)
    prompt_data = build_rag_prompt(query_image, retrieved, task=task)
    response = call_mllm(prompt_data)

    if task == "classify":
        valid_labels = list(set(r["label"] for r in retrieved))
        prediction = parse_classification(response, valid_labels)
    else:
        prediction = parse_regression(response)

    return {
        "prediction": prediction,
        "retrieved_labels": [r["label"] for r in retrieved],
        "retrieved_similarities": [r["similarity"] for r in retrieved],
        "raw_response": response
    }
```

### 9. Evaluate with proper metrics

Measure search quality (precision@k, recall@k), classification accuracy, and regression error (MAE/RMSE) to confirm the RAG pipeline improves over the MLLM baseline.

```python
from sklearn.metrics import accuracy_score, mean_absolute_error
import numpy as np

def evaluate_retrieval(results: list[dict]) -> dict:
    precisions = []
    for r in results:
        correct = sum(1 for rl in r["retrieved_labels"] if rl == r["true_label"])
        precisions.append(correct / len(r["retrieved_labels"]))
    return {"precision_at_k": np.mean(precisions)}

def evaluate_classification(results: list[dict]) -> dict:
    y_true = [r["true_label"] for r in results]
    y_pred = [r["prediction"] for r in results]
    return {"accuracy": accuracy_score(y_true, y_pred)}
```

## Concrete Examples

**Example 1: SAR Vehicle Classification with MSTAR Dataset**

User: "I have the MSTAR SAR dataset with 10 vehicle classes. Build me a system that classifies test images by retrieving similar training examples."

Approach:
1. Load MSTAR training split images and labels (BMP-2, BTR-70, T-72, BTR-60, 2S1, BRDM-2, D7, T-62, ZIL-131, ZSU-234)
2. Embed all training images with CLIP-ViT-L/14 into a ChromaDB collection
3. For each test image, retrieve top-5 nearest neighbors from the training set
4. Build a prompt showing the 5 reference images with labels + the query image
5. Ask the MLLM: "Based on visual comparison with these references, classify the query vehicle"
6. Parse the predicted label and compute accuracy across the test set

Output:
```
Baseline MLLM (no retrieval):  62.3% accuracy
SAR-RAG (k=5 retrieval):       78.1% accuracy
Retrieval precision@5:          0.73
```

**Example 2: Medical Image Triage with Exemplar Retrieval**

User: "I have a database of labeled X-ray images. Help me build a system where new X-rays get classified by finding similar past cases."

Approach:
1. Embed the labeled X-ray archive using a medical vision encoder (BiomedCLIP or PubMedCLIP)
2. Store embeddings in FAISS with metadata: diagnosis, severity, patient demographics
3. For a new X-ray, retrieve top-3 most similar cases
4. Prompt the MLLM: "Here are 3 similar prior cases with confirmed diagnoses [A, B, C]. Compare them with this new image. What is the most likely diagnosis?"
5. Return structured output: predicted diagnosis, confidence reasoning, reference case IDs

Output:
```json
{
  "prediction": "pneumonia",
  "confidence_reasoning": "Query shows bilateral opacities consistent with Ref 1 (pneumonia, similarity 0.91) and Ref 2 (pneumonia, similarity 0.87). Ref 3 is pleural effusion (similarity 0.82) but pattern differs.",
  "reference_cases": ["case_4521", "case_3087", "case_6193"]
}
```

**Example 3: Industrial Defect Classification with Dimension Estimation**

User: "We photograph manufactured parts and need to classify defect types and estimate defect dimensions. We have a database of past inspections."

Approach:
1. Embed past inspection images with CLIP, store in Qdrant with labels (crack, scratch, pit, void) and measured dimensions (length_mm, width_mm)
2. For new inspection image, retrieve top-5 similar defects
3. Use `task="classify"` prompt to identify defect type
4. Use `task="regress"` prompt to estimate physical dimensions from reference comparisons
5. Return both classification and dimension estimates

Output:
```json
{
  "defect_type": "crack",
  "estimated_length_mm": 12.3,
  "estimated_width_mm": 0.8,
  "similar_cases": [
    {"id": "insp_892", "type": "crack", "similarity": 0.94},
    {"id": "insp_1247", "type": "crack", "similarity": 0.91}
  ]
}
```

## Best Practices

**Do:**
- Use the same vision encoder for both indexing reference images and embedding queries. Mixing encoders produces incompatible embedding spaces.
- L2-normalize embeddings before storing and querying so that dot product equals cosine similarity, which simplifies distance computation.
- Include metadata beyond just labels -- viewing angle, acquisition parameters, and physical measurements all add value to the retrieval context.
- Experiment with k values (3, 5, 10). Too few exemplars may miss the correct class; too many dilute the prompt with irrelevant references and hit MLLM token/image limits.
- Present reference images and their labels in a structured, numbered format so the MLLM can explicitly reason about which reference is most similar.

**Avoid:**
- Do not embed images at low resolution if fine-grained visual distinctions matter. Ensure preprocessing matches the encoder's expected input size (typically 224x224 or 336x336).
- Do not skip the retrieval evaluation step. If precision@k is low, the MLLM receives misleading references and accuracy will degrade rather than improve.
- Do not send more images than the MLLM context window supports. Most MLLMs handle 5-10 images well; beyond that, quality degrades.
- Do not assume the MLLM will always follow the retrieved consensus. Parse outputs defensively and validate against the known label set.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Retrieved exemplars all have low similarity (<0.5) | Query is out-of-distribution or encoder is poor fit | Fall back to zero-shot MLLM classification; flag for human review |
| MLLM returns a label not in the valid set | Free-form generation drift | Constrain parsing to valid labels only; use structured output modes if available |
| Vector DB query is slow (>1s for single query) | Index not optimized or dataset too large for brute-force | Switch to approximate nearest neighbor (HNSW index in FAISS/ChromaDB) |
| Embedding dimension mismatch | Encoder changed or mixed encoders | Verify encoder consistency; rebuild index if encoder changes |
| MLLM refuses to classify images | Content policy rejection (e.g., military imagery) | Use a self-hosted MLLM (LLaVA, InternVL) with no content filtering |
| Classification accuracy worse with RAG than baseline | Retrieved neighbors are from wrong classes | Increase k, improve encoder (fine-tune on domain), or filter by similarity threshold |

## Limitations

- **Token/image budget**: MLLMs have finite context windows. With k=5 reference images + 1 query, you consume 6 image slots per request, which limits batch processing and adds cost.
- **Embedding quality ceiling**: If the vision encoder cannot distinguish between target classes in embedding space, retrieval will return wrong-class neighbors and hurt rather than help. Domain-specific fine-tuning of the encoder may be necessary.
- **Not a replacement for specialized classifiers**: For well-defined classification tasks with abundant training data, a fine-tuned CNN or ViT will likely outperform ImageRAG at lower inference cost. ImageRAG excels when training data is limited, classes are evolving, or interpretability (showing the reference cases) is required.
- **Latency**: The pipeline involves embedding + vector search + MLLM inference, making it slower than a single-model classifier. Not suitable for hard real-time requirements.
- **Image-only retrieval**: The current approach retrieves by visual similarity only. It does not incorporate text metadata in the similarity search (though metadata is passed to the MLLM). Hybrid retrieval combining visual and textual embeddings could improve results.

## Reference

**Paper**: Ramirez, D.F., Overman, T., Jaskie, K., Marvin, J., & Spanias, A. (2026). *SAR-RAG: ATR Visual Question Answering by Semantic Search, Retrieval, and MLLM Generation*. arXiv:2602.04712v1. Submitted to 2026 IEEE Radar Conference.

**What to look for**: The paper's evaluation on the MSTAR SAR dataset demonstrates that adding a retrieval memory bank to an MLLM baseline improves categorical classification accuracy, search precision, and numeric dimension regression. Focus on Section III (method architecture), Section IV (experimental setup and metrics), and the ablation comparing baseline MLLM vs. SAR-RAG.