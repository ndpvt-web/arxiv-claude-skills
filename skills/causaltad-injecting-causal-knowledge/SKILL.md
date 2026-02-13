---
name: "causaltad-injecting-causal-knowledge"
description: "Detect anomalies in tabular data by injecting causal column relationships into LLM-based detection pipelines. Reorders and reweights columns based on causal graphs before serializing rows to text for fine-tuned language models. Trigger phrases: 'detect anomalies in tabular data', 'causal anomaly detection', 'tabular fraud detection with LLM', 'causal column ordering for anomaly detection', 'reorder columns by causal graph', 'CausalTAD'."
---

# CausalTAD: Causal Knowledge Injection for Tabular Anomaly Detection

This skill enables Claude to implement the CausalTAD pipeline — a method that discovers causal relationships between columns in tabular data, reorders columns to respect causal structure, assigns importance weights per column, and serializes the result into text for LLM-based anomaly detection. The core insight is that column order in text-serialized tabular data is not arbitrary: ordering columns along causal flows (causes before effects) lets autoregressive LLMs model conditional dependencies more naturally, producing sharper anomaly scores than random orderings.

## When to Use

- When the user wants to detect anomalies (fraud, intrusions, outliers) in structured tabular datasets using LLMs
- When the user asks how to convert tabular data to text for a language model and wants a principled column ordering
- When the user has a fraud detection or anomaly detection task on CSV/DataFrame data and wants state-of-the-art results
- When the user asks about causal discovery on tabular features and how to exploit causal structure for downstream ML
- When the user wants to fine-tune a small language model (SmolLM, GPT-2) on tabular data for anomaly scoring
- When the user needs to reweight feature contributions in an anomaly detection pipeline based on causal importance

## Key Technique

**Problem with standard approaches.** Methods like AnoLLM serialize each row of a table into text (e.g., `"age is 35, income is 50000, flagged is yes"`) and fine-tune an autoregressive LLM to model the joint distribution of normal data. At inference, rows with low log-likelihood are flagged as anomalies. However, these methods randomly permute column order during serialization — the LLM sees `"income is 50000, age is 35, flagged is yes"` one time and a different order the next. This ignores that `age` causally influences `income`, which causally influences `flagged`. An autoregressive model benefits from seeing causes before their effects, since P(effect | cause) is easier to model than the reverse.

**CausalTAD's solution.** The method has three stages: (1) **Causal discovery** — run the PC algorithm (or GES/FCI) from the `causallearn` library on the training data to produce a directed acyclic graph (DAG) over columns. (2) **Causal ordering** — solve a linear ordering problem on the DAG to produce a topological sort that places parent columns before child columns. When the DAG has multiple valid orderings, the method selects orderings that maximize alignment with edge directions. This deterministic ordering replaces random shuffling. (3) **Causal reweighting** — assign each column a weight proportional to its causal centrality (e.g., number of descendants or edges in the DAG). During inference, per-column log-likelihood contributions are scaled by `weight + 1.0`, so causally important columns contribute more to the anomaly score.

**Why it works.** Autoregressive LLMs decompose the joint probability as a product of conditionals: P(col1) × P(col2|col1) × P(col3|col1,col2) × ... . When col1 is a cause of col2, the conditional P(col2|col1) captures a real generative mechanism. In anomalous rows, these conditional probabilities drop sharply, making anomalies easier to detect. Random ordering mixes causes and effects, diluting the signal.

## Step-by-Step Workflow

### 1. Load and inspect the tabular dataset

Read the CSV/DataFrame. Identify column names, data types (numerical vs. categorical), and the label column (if any). Separate training data (normal samples only) from evaluation data (normal + anomalous).

```python
import pandas as pd
df = pd.read_csv("transactions.csv")
train_df = df[df["label"] == 0].drop(columns=["label"])
eval_df = df.copy()
```

### 2. Bin numerical columns into categorical buckets

Autoregressive LLMs work on tokens, so continuous values must be discretized. Use quantile binning (recommended) or equal-width binning into 10-20 buckets.

```python
from sklearn.preprocessing import KBinsDiscretizer
binner = KBinsDiscretizer(n_bins=15, encode="ordinal", strategy="quantile")
num_cols = train_df.select_dtypes(include="number").columns.tolist()
train_df[num_cols] = binner.fit_transform(train_df[num_cols])
eval_df[num_cols] = binner.transform(eval_df[num_cols])
# Convert bin indices to descriptive labels like "bin_0", "bin_1", ...
for col in num_cols:
    train_df[col] = train_df[col].astype(int).apply(lambda x: f"bin_{x}")
    eval_df[col] = eval_df[col].astype(int).apply(lambda x: f"bin_{x}")
```

### 3. Run causal discovery to produce a DAG over columns

Use the PC algorithm from `causallearn` with a suitable independence test (Fisher's Z for continuous data, chi-square for categorical). The output is an adjacency matrix representing directed edges.

```python
from causallearn.search.ConstraintBased.PC import pc
import numpy as np

# Encode categoricals numerically for causal discovery
from sklearn.preprocessing import LabelEncoder
encoded = train_df.copy()
for col in encoded.columns:
    if encoded[col].dtype == object:
        encoded[col] = LabelEncoder().fit_transform(encoded[col])

result = pc(encoded.values, alpha=0.05, indep_test="fisherz")
adjacency = result.G.graph  # shape: (n_cols, n_cols)
col_names = train_df.columns.tolist()
```

### 4. Extract the causal ordering via topological sort

Convert the adjacency matrix to a directed graph and compute a topological ordering. This places cause columns before effect columns. If the graph has cycles (due to statistical noise), break them by removing the weakest edges.

```python
import networkx as nx

G = nx.DiGraph()
for i, src in enumerate(col_names):
    for j, tgt in enumerate(col_names):
        if adjacency[i][j] == -1 and adjacency[j][i] == 1:
            G.add_edge(src, tgt)  # src -> tgt

# Break cycles if any
while not nx.is_directed_acyclic_graph(G):
    cycle = nx.find_cycle(G)
    G.remove_edge(*cycle[-1][:2])  # Remove last edge in cycle

causal_order = list(nx.topological_sort(G))
# Add any columns not in the graph (isolated nodes) at the end
remaining = [c for c in col_names if c not in causal_order]
causal_order.extend(remaining)
```

### 5. Compute per-column causal weights

Assign each column a weight reflecting its causal importance. A practical measure is the number of descendants (direct + transitive) in the DAG, normalized to [0, 1].

```python
weights = {}
for col in causal_order:
    if col in G:
        desc_count = len(nx.descendants(G, col))
    else:
        desc_count = 0
    weights[col] = desc_count

max_w = max(weights.values()) if max(weights.values()) > 0 else 1
weights = {col: round(w / max_w, 4) for col, w in weights.items()}
# Save for inference
import json
with open("causal_weights.json", "w") as f:
    json.dump({"order": causal_order, "weights": weights}, f)
```

### 6. Serialize each row to text using causal column order

Convert each row to the format `" col_name is value , col_name is value , ... "` with columns in the causal order determined in step 4.

```python
def serialize_row(row, column_order):
    parts = [f" {col} is {row[col]} " for col in column_order]
    return ",".join(parts)

train_texts = train_df.apply(lambda r: serialize_row(r, causal_order), axis=1).tolist()
```

### 7. Fine-tune a small autoregressive LLM on normal data

Use a model like SmolLM-135M or GPT-2. Fine-tune with standard causal language modeling loss (next-token prediction). LoRA is recommended for efficiency.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments, Trainer
from peft import get_peft_model, LoraConfig

model_name = "HuggingFaceTB/SmolLM-135M"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

lora_config = LoraConfig(r=8, lora_alpha=16, target_modules=["q_proj", "v_proj"])
model = get_peft_model(model, lora_config)

# Tokenize training texts
encodings = tokenizer(train_texts, truncation=True, max_length=512,
                      padding=True, return_tensors="pt")

# Standard causal LM training
training_args = TrainingArguments(
    output_dir="./causaltad_model", num_train_epochs=5,
    per_device_train_batch_size=32, learning_rate=5e-4,
    bf16=True, logging_steps=50
)
# Use HuggingFace Trainer with DataCollatorForLanguageModeling
```

### 8. Score evaluation rows with weighted log-likelihood

For each evaluation row, compute per-column negative log-likelihood from the fine-tuned model. Multiply each column's contribution by its causal weight + 1.0. Higher total score = more anomalous.

```python
import torch

def score_row(text, column_order, weights, model, tokenizer):
    inputs = tokenizer(text, return_tensors="pt")
    with torch.no_grad():
        outputs = model(**inputs, labels=inputs["input_ids"])
    # Per-token NLL
    logits = outputs.logits[:, :-1, :]
    labels = inputs["input_ids"][:, 1:]
    nll = torch.nn.functional.cross_entropy(
        logits.reshape(-1, logits.size(-1)), labels.reshape(-1), reduction="none"
    )
    # Map tokens back to columns and apply weights
    total_score = nll.sum().item()  # Simplified; production code maps token spans to columns
    return total_score

eval_texts = eval_df.drop(columns=["label"]).apply(
    lambda r: serialize_row(r, causal_order), axis=1
).tolist()
scores = [score_row(t, causal_order, weights, model, tokenizer) for t in eval_texts]
```

### 9. Evaluate with AUROC and flag anomalies

Compare scores against ground truth labels using AUROC. Optionally set a threshold at a chosen percentile of training scores.

```python
from sklearn.metrics import roc_auc_score
auroc = roc_auc_score(eval_df["label"], scores)
print(f"AUROC: {auroc:.4f}")
```

## Concrete Examples

**Example 1: Credit Card Fraud Detection**

User: "I have a credit card transaction dataset with columns like amount, merchant_category, time_of_day, distance_from_home, and is_fraud. Help me detect fraud using CausalTAD."

Approach:
1. Load the data, separate normal transactions for training
2. Bin `amount`, `time_of_day`, `distance_from_home` into 15 quantile bins
3. Run PC algorithm — discover that `time_of_day → merchant_category → amount` and `distance_from_home → amount`
4. Topological order: `[time_of_day, distance_from_home, merchant_category, amount]`
5. Weights: `time_of_day: 0.75, distance_from_home: 0.5, merchant_category: 0.5, amount: 0.0`
6. Serialize: `" time_of_day is bin_3 , distance_from_home is bin_12 , merchant_category is grocery , amount is bin_7 "`
7. Fine-tune SmolLM-135M, score eval rows, achieve AUROC ~0.93

**Example 2: Network Intrusion Detection**

User: "I have network flow data with src_port, dst_port, protocol, packet_size, duration, flag_count. Can I use causal ordering to improve anomaly detection?"

Approach:
1. Causal discovery finds: `protocol → dst_port`, `protocol → packet_size`, `dst_port → duration`, `flag_count ← duration`
2. Topological order: `[protocol, src_port, dst_port, packet_size, duration, flag_count]`
3. `protocol` gets highest weight (most descendants), `flag_count` gets lowest
4. Serialize rows in this order, fine-tune, and score — causal ordering lets the model learn that certain protocol+port combinations have predictable packet sizes, making deviations more detectable

**Example 3: Manufacturing Quality Control**

User: "My factory sensor data has temperature, pressure, vibration, speed, and defect_rate. How do I set up CausalTAD?"

Approach:
1. PC algorithm discovers: `speed → vibration`, `temperature → pressure`, `vibration → defect_rate`, `pressure → defect_rate`
2. Topological order: `[speed, temperature, vibration, pressure, defect_rate]`
3. `speed` and `temperature` get high weights as root causes; `defect_rate` gets zero (leaf node)
4. The model learns normal conditional distributions — e.g., normal vibration given speed — and flags rows where vibration is abnormally high for the given speed

## Best Practices

**Do:**
- Train causal discovery only on normal (non-anomalous) data — anomalous rows can introduce spurious causal edges
- Use quantile binning over equal-width binning for numerical columns — it preserves distributional information better
- Experiment with multiple causal discovery algorithms (PC, GES, FCI) and compare the resulting graphs for consistency
- Use the causal ordering deterministically at both training and inference time — do not shuffle columns once the order is set

**Avoid:**
- Do not include the label column in causal discovery — it would create trivial edges from all features to the label
- Do not use causal ordering on datasets with fewer than 3 columns — there is not enough structure for causal relationships to matter
- Do not skip the binning step for numerical features — raw continuous values tokenize poorly and produce uninformative tokens
- Do not assume the causal graph is always a DAG — statistical tests can produce cycles; always check and break them before topological sorting

## Error Handling

- **Causal discovery produces an empty graph (no edges):** Fall back to domain-knowledge ordering or correlation-based ordering (order by mutual information with other columns). The reweighting step assigns uniform weights.
- **Causal graph has cycles:** Use `nx.find_cycle()` to identify cycles, then remove the edge with the highest p-value (weakest statistical evidence). Repeat until acyclic.
- **Too many columns (>50):** PC algorithm becomes slow and unreliable with many variables. Pre-select the top 20-30 columns by variance or domain relevance before running causal discovery.
- **Very small datasets (<200 rows):** Causal discovery will be unreliable. Use domain knowledge to manually specify the causal graph, or skip causal ordering and fall back to AnoLLM with random column permutations.
- **Tokenization overflow:** Long serialized rows can exceed model context length. Truncate individual column values (not columns themselves) to fit within the model's max sequence length (512 tokens is typical).

## Limitations

- The method requires a meaningful causal structure between columns. Datasets where columns are independent (e.g., one-hot encoded features from the same categorical) will not benefit from causal ordering.
- Causal discovery algorithms assume causal sufficiency (no unobserved confounders) for PC/GES. If hidden confounders exist, the discovered graph may be inaccurate. FCI handles latent confounders but produces a less precise partial ancestral graph.
- Fine-tuning an LLM requires GPU resources. For very large datasets (>1M rows), the training cost may not justify the improvement over classical anomaly detectors like Isolation Forest.
- The text serialization format is verbose — each row becomes a sentence of ~100-300 tokens. This limits scalability to wide tables (many columns) due to context length constraints.
- The method is designed for unsupervised/semi-supervised anomaly detection (training on normal data only). It is not a supervised classifier and will not directly predict anomaly types.

## Reference

**Paper:** [CausalTAD: Injecting Causal Knowledge into Large Language Models for Tabular Anomaly Detection](https://arxiv.org/abs/2602.07798v1) — Wang et al., 2026. Look for Section 3 (Method) which details the causal discovery, linear ordering formulation, and reweighting strategy, and Section 4 (Experiments) for results across 30+ datasets.

**Code:** [github.com/350234/CausalTAD](https://github.com/350234/CausalTAD) — Three-stage pipeline: `graph_gen/` for causal discovery, `use_graph/` for ordering and weights, `AnoLLM/` for LLM fine-tuning and scoring.