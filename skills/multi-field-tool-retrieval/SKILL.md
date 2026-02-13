---
name: "multi-field-tool-retrieval"
description: "Implement multi-field tool retrieval systems that decompose tool documentation into structured fields (description, parameters, response, examples) and match user queries against each field independently with learned weights. Use when asked to 'build a tool retrieval system', 'improve tool search for an LLM agent', 'index API documentation for retrieval', 'match user queries to tools', 'build a tool recommendation engine', or 'optimize function/API lookup for agents'."
---

# Multi-Field Tool Retrieval (MFTR)

This skill enables Claude to build tool retrieval systems that go beyond naive full-document matching. Instead of treating each tool's documentation as a single blob of text, the MFTR framework decomposes tool docs into four standardized fields — **description**, **parameters**, **response**, and **examples** — then scores queries against each field independently and combines scores with learned weights and a parameter-missing penalty. This approach, from Tang et al. (2026), achieved state-of-the-art results across five benchmarks, improving NDCG@10 by 28–50% over full-document baselines.

## When to Use

- When building a tool/API retrieval layer for an LLM agent that must select from hundreds or thousands of tools
- When indexing a heterogeneous set of API docs (REST APIs, CLI tools, Python functions) where documentation quality varies
- When existing keyword or embedding search over raw tool docs returns poor results due to vocabulary mismatch between user queries and technical docs
- When an agent framework needs to narrow a large tool corpus to a shortlist before passing to the LLM context window
- When designing a retrieval-augmented tool-use pipeline and you need the retrieval stage to consider parameter feasibility, not just functional relevance
- When migrating from a naive "search tool names + descriptions" approach and want structured, multi-signal ranking

## Key Technique

**The core insight** is that tool utility is multi-dimensional. A tool's relevance to a query depends on (1) whether its functionality matches the user's intent, (2) whether the user's inputs satisfy the tool's parameter requirements, (3) whether the tool's output format matches what the user needs, and (4) whether the query aligns with typical use cases. Scoring all four dimensions independently and combining them with learned weights dramatically outperforms matching a query against a concatenated document.

**The pipeline has two phases.** In the offline phase, an LLM standardizes all tool documentation into a four-field schema (description, parameters, response, examples), filling in missing fields and normalizing structure. In the online phase, the user query is decomposed into structured sub-intents aligned to each field, optionally enhanced with pseudo-relevance feedback from a fast BM25 pre-retrieval step. Each field is scored independently using any retriever (BM25, dense encoder, or hybrid), then scores are aggregated via: `S(q,t) = Σ_f w_f · S_f(q,t) + b - P(q,t)`, where `P(q,t)` is an adaptive penalty for parameters the query cannot satisfy. The field weights `w_f` are learned from pairwise ranking loss.

**Why it works better:** Full-document retrieval dilutes signal — a tool with a perfect functional match but incompatible parameters scores high because the description dominates. MFTR catches this because the parameter penalty explicitly down-ranks tools whose required parameters are not satisfiable by the query. The field decomposition also solves vocabulary mismatch: the "examples" field uses natural-language intents close to how users write queries, bridging the gap between casual queries and technical API prose.

## Step-by-Step Workflow

### 1. Standardize Tool Documentation into Four Fields

For each tool in your corpus, produce a JSON object with exactly four fields:

```json
{
  "tool_id": "get_weather",
  "description": "Retrieves current weather conditions for a specified location, including temperature, humidity, and wind speed.",
  "parameters": [
    {"name": "location", "type": "string", "required": true, "meaning": "City name or coordinates"},
    {"name": "units", "type": "enum[metric,imperial]", "required": false, "meaning": "Temperature unit system"}
  ],
  "response": "JSON object with fields: temperature (float), humidity (float), wind_speed (float), condition (string)",
  "examples": [
    "What's the weather like in Tokyo?",
    "Get the current temperature in New York in Fahrenheit",
    "Check humidity levels for my city"
  ]
}
```

Use an LLM to extract and normalize from raw docs. Mark all parameters as required unless the source explicitly says optional. Generate 3–5 example user intents per tool if none exist.

### 2. Build Per-Field Indexes

Create a separate search index for each of the four fields. For BM25, this means four inverted indexes. For dense retrieval, encode each field's text separately and store four vector collections. Do not concatenate fields into a single document.

```python
from collections import defaultdict

field_indexes = {}
for field in ["description", "parameters", "response", "examples"]:
    field_indexes[field] = build_index(
        documents=[tool[field] for tool in tools],
        doc_ids=[tool["tool_id"] for tool in tools]
    )
```

For the `parameters` field, serialize the parameter list into a structured string: `"location (string, required): City name or coordinates; units (enum, optional): Temperature unit system"`.

### 3. Decompose the User Query

Transform the raw user query into structured sub-queries aligned to each field:

- **intent** → matches against `examples` field (natural language intent)
- **tool_description** → matches against `description` field (functional summary)
- **expected_output** → matches against `response` field (what format/data the user needs)
- **extracted_args** → matches against `parameters` field (arguments the user provided or implied)

Use an LLM with this prompt structure:

```
Given the user query, extract:
1. intent: What the user wants to accomplish (plain language)
2. tool_description: What kind of tool would solve this (functional description)
3. expected_output: What output format/data the user expects
4. extracted_args: Any arguments, values, or constraints mentioned
   (for each: name, value, inferred type)

Query: "Convert this 1080p MP4 video to a 720p GIF under 5MB"
```

### 4. (Optional) Apply Pseudo-Relevance Feedback

Run BM25 over the `description` index with the raw query. Take the top-K (K=20) results. Feed the full documentation of the top-3 tools back to the LLM and ask it to rewrite the decomposed query using terminology from the tool repository. This closes vocabulary gaps between user language and API jargon.

### 5. Score Each Field Independently

For each candidate tool, compute a relevance score per field:

```python
def score_tool(query_fields, tool, retrievers):
    scores = {}
    for field in ["description", "response", "examples"]:
        # Take max across query sub-intents for multi-intent queries
        scores[field] = max(
            retrievers[field].similarity(qf, tool[field])
            for qf in query_fields[field]
        )
    # Parameter scoring: average per-parameter match
    param_scores = []
    for param in tool["parameters"]:
        param_scores.append(
            retrievers["parameters"].similarity(
                query_fields["extracted_args"], param
            )
        )
    scores["parameters"] = mean(param_scores) if param_scores else 0.0
    return scores
```

### 6. Compute Parameter Missing Penalty

For each tool parameter, apply a sigmoid-gated penalty that activates when the query cannot provide a required argument:

```python
import math

def param_penalty(param_scores, tool_params, alpha=15, tau=0.5, w_req=1.0, w_opt=0.3):
    penalty = 0.0
    for score, param in zip(param_scores, tool_params):
        gate = 1.0 / (1.0 + math.exp(alpha * (tau - score)))
        weight = w_req if param["required"] else w_opt
        penalty += gate * weight
    return penalty
```

The penalty is high when a required parameter has low alignment score (the query doesn't mention anything matching it), and near-zero when scores are above threshold `tau`.

### 7. Aggregate into Final Score

Combine field scores with learned weights and subtract the parameter penalty:

```python
def final_score(field_scores, penalty, weights, bias):
    # weights = {"description": 0.35, "parameters": 0.25, "response": 0.15, "examples": 0.25}
    score = sum(weights[f] * field_scores[f] for f in field_scores) + bias
    return score - penalty
```

### 8. Learn Weights from Labeled Data (or Set Heuristically)

If you have query-tool relevance labels, train `w_f`, `b`, `alpha`, and `tau` by minimizing pairwise ranking loss:

```
loss = log(1 + exp(-(S(q, t+) - S(q, t-))))
```

Sample negatives from top-64 BM25 results per query. Use Adam optimizer, lr=0.1, batch size 256, 5 epochs with 5-fold cross-validation.

If no labeled data is available, use the heuristic weights: description=0.35, examples=0.25, parameters=0.25, response=0.15, bias=0, alpha=15, tau=0.5.

### 9. Return Ranked Tool List

Sort tools by final score descending. Return the top-N (typically N=5–10) for the LLM's context window.

## Concrete Examples

**Example 1: Building a Tool Retriever for a Coding Agent**

User: "I have 500 Python function docstrings and I need to build a retrieval system so my AI agent can find the right function given a natural language request."

Approach:
1. Parse each docstring into the four-field schema: extract the summary line as `description`, parse `:param` entries into `parameters` with types and required flags, parse `:returns` as `response`, and generate 3 example intents per function using an LLM.
2. Build four FAISS indexes using sentence-transformers embeddings (one per field).
3. At query time, decompose the user's request into sub-queries, score each field, apply parameter penalty, and return top-5 functions.

Output:
```python
# Standardized tool record
{
  "tool_id": "pandas.DataFrame.merge",
  "description": "Merge two DataFrames using database-style join operations on columns or indexes.",
  "parameters": [
    {"name": "right", "type": "DataFrame", "required": true, "meaning": "The other DataFrame to merge with"},
    {"name": "how", "type": "enum[left,right,outer,inner,cross]", "required": false, "meaning": "Type of join"},
    {"name": "on", "type": "str|list", "required": false, "meaning": "Column names to join on"}
  ],
  "response": "A new DataFrame combining columns from both inputs, with rows matched by join keys.",
  "examples": [
    "Join two tables on a common column",
    "Combine customer and order data by customer_id",
    "Left join sales with products"
  ]
}

# Query: "combine my users table with their purchases by user_id"
# Field scores: description=0.82, examples=0.91, parameters=0.74, response=0.68
# Parameter penalty: 0.03 (user_id maps to 'on', right table implied)
# Final: 0.35*0.82 + 0.25*0.91 + 0.25*0.74 + 0.15*0.68 - 0.03 = 0.77
```

**Example 2: Indexing a REST API Collection**

User: "I have 200 REST API endpoints documented in OpenAPI spec. Help me build search so users can find the right endpoint."

Approach:
1. Parse the OpenAPI spec. Map `summary` → description, `parameters` + `requestBody` → parameters, `responses.200` → response. Generate example queries from the operation description.
2. Serialize parameters as structured strings preserving name, type, required, and description.
3. Use hybrid retrieval: BM25 for the `parameters` field (exact parameter names matter) and dense embeddings for the other three fields.
4. Apply pseudo-relevance feedback: for each query, BM25-retrieve top-20 by description, feed top-3 full docs to the LLM for query rewriting.

Output:
```
Query: "upload a profile picture for user 42"

Decomposed:
  intent: "upload a user's profile image"
  tool_description: "endpoint that accepts image file uploads for user profiles"
  expected_output: "confirmation with URL of uploaded image"
  extracted_args: [{name: "user_id", value: "42", type: "integer"},
                   {name: "image", value: null, type: "file"}]

Top-3 results:
  1. PUT /users/{id}/avatar   — score: 0.89
  2. POST /users/{id}/media   — score: 0.71
  3. POST /files/upload        — score: 0.58 (penalized: missing user_id param)
```

**Example 3: Improving Tool Selection in an Existing Agent**

User: "My LangChain agent has 80 tools but often picks the wrong one. Can we improve tool selection?"

Approach:
1. Export all tool schemas. Standardize into four-field format (LangChain tools already have name, description, and args_schema — map these, then generate examples and response fields).
2. Replace the default similarity search with multi-field retrieval: build per-field indexes, implement the scoring pipeline as a custom LangChain retriever.
3. Collect failure cases where the agent picked the wrong tool. Use these as training signal — the correct tool is positive, the incorrectly selected tool is a hard negative. Train field weights on this data.
4. Integrate: the custom retriever returns top-5 tools per query, which are injected into the agent's prompt.

Output:
```python
class MFTRToolRetriever(BaseRetriever):
    def _get_relevant_tools(self, query: str) -> List[Tool]:
        decomposed = self.llm.decompose_query(query)
        candidates = self.bm25_prefilter(query, k=50)
        scored = []
        for tool in candidates:
            field_scores = self.score_fields(decomposed, tool)
            penalty = self.param_penalty(decomposed, tool)
            total = self.aggregate(field_scores, penalty)
            scored.append((tool, total))
        scored.sort(key=lambda x: -x[1])
        return [t for t, s in scored[:5]]
```

## Best Practices

- **Do:** Always standardize tool documentation into the four-field schema before indexing — even if some fields need to be LLM-generated. The structure is what enables multi-signal ranking.
- **Do:** Use the parameter missing penalty for any retrieval scenario where input feasibility matters. This is the single biggest differentiator over naive retrieval.
- **Do:** Generate synthetic example queries for each tool if real usage data isn't available. The `examples` field bridges the vocabulary gap between user language and API terminology.
- **Do:** Start with heuristic weights (desc=0.35, examples=0.25, params=0.25, response=0.15) and only learn weights when you have at least 100 labeled query-tool pairs.
- **Avoid:** Concatenating all four fields back into a single document for retrieval — this defeats the purpose. Each field must be scored independently.
- **Avoid:** Skipping the pseudo-relevance feedback step when your tool corpus uses domain-specific jargon. The BM25 pre-retrieval + LLM rewrite step closes the vocabulary gap at low cost (~1s latency).
- **Avoid:** Setting all parameters as optional to reduce penalties. Accurate required/optional flags are critical for the penalty to function correctly.

## Error Handling

- **Incomplete tool documentation:** When a field is entirely missing (e.g., no response schema), use the LLM to infer it from the description and parameters. If inference is unreliable, set that field's weight to zero for the affected tools rather than hallucinating content.
- **Query decomposition failure:** If the LLM produces a malformed decomposition, fall back to using the raw query against all fields equally (uniform weights, no penalty). Log the failure for later prompt tuning.
- **No parameter overlap:** When a query mentions no arguments at all, skip the parameter penalty entirely (set penalty to 0) to avoid unfairly penalizing tools with many required parameters.
- **Cold start with no training data:** Use heuristic weights and collect implicit feedback (which tool the agent actually uses successfully) to build a training set over time.
- **Retriever latency concerns:** The LLM query decomposition adds ~1s. If this is too slow, pre-compute decompositions for common query patterns or use a smaller model (e.g., GPT-4o-mini or a fine-tuned local model) for decomposition.

## Limitations

- The framework requires an LLM call per query for decomposition, adding latency that may be unacceptable for real-time applications under 100ms.
- Standardizing tool documentation relies on LLM quality — poorly documented tools with no examples or parameter descriptions produce weak field representations even after LLM augmentation.
- The learned weights are dataset-specific. Weights trained on REST API retrieval may not transfer well to CLI tool retrieval or Python function retrieval without retraining.
- For very small tool corpora (under 20 tools), multi-field retrieval adds complexity without meaningful improvement — just pass all tools to the LLM context.
- The approach assumes tools are independent. It does not model tool composition or sequential tool dependencies.

## Reference

Tang, Y., Su, W., Liu, Y., & Ai, Q. (2026). Multi-Field Tool Retrieval. arXiv:2602.05366v1. https://arxiv.org/abs/2602.05366v1

Key sections to study: Section 3 (framework architecture with four-field schema and scoring formulas), Section 4 (query decomposition with pseudo-relevance feedback), and Table 2 (NDCG@10 results showing 28–50% improvement over full-document baselines across five benchmarks).