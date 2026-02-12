---
name: "evaluating-retrievalaugmented-generation-variants"
description: |
  Build production-grade natural language to SQL/API pipelines using RAG variant selection
  (Standard RAG, Self-RAG, CoRAG). Implements iterative query decomposition, hybrid documentation
  retrieval, and dynamic task classification for enterprise NL interfaces.
  Trigger phrases:
  - "Build a natural language to SQL interface with RAG"
  - "Generate API calls from user questions using retrieval augmented generation"
  - "Set up a hybrid SQL and REST API generation pipeline"
  - "Implement CoRAG for enterprise query generation"
  - "Create a text-to-SQL system with document retrieval"
  - "Design a retrieval pipeline that handles both database queries and API calls"
---

# RAG-Based Natural Language to SQL & API Call Generation

This skill enables Claude to design and implement retrieval-augmented generation pipelines that translate natural language requests into executable SQL queries and REST API calls. Based on a systematic evaluation of three RAG variants — Standard RAG, Self-RAG, and CoRAG — across 18 configurations on enterprise banking data, the skill encodes which retrieval strategy to select based on documentation heterogeneity, how to structure hybrid document stores, and how to build dynamic task classifiers that route user intent to the correct output modality (SQL vs. API).

## When to Use

- When the user needs a natural language interface that generates SQL queries against a known schema
- When the user wants to translate plain-English requests into REST API calls (POST/PUT/PATCH/DELETE)
- When building a system that must handle both read operations (SQL SELECT) and write/mutation operations (API calls) from a single input
- When designing a RAG pipeline and choosing between top-k retrieval, relevance filtering, or iterative decomposition
- When the user has heterogeneous documentation (database schemas mixed with API specs) and needs robust retrieval across both
- When optimizing retrieval for domain-specific enterprise systems (ERP, banking, CRM) where zero-shot LLM accuracy is near 0%

## Key Technique

The paper's central finding is that **retrieval policy design determines success** in NL-to-code generation for enterprise systems. Without retrieval, even GPT-5 achieves 0% exact match on domain-specific SQL and API generation. Three RAG variants were evaluated:

**Standard RAG** embeds the user query, retrieves the top-5 most similar documentation chunks from a vector store, concatenates them with task-specific instructions, and makes a single LLM call. This is the simplest approach and works adequately when documentation is homogeneous (all DB schemas or all API specs). **Self-RAG** adds a post-retrieval relevance filter: each retrieved chunk is individually scored, and only chunks exceeding a relevance threshold (e.g., >= 0.2) proceed to the prompt. This reduces noise but can over-filter in hybrid contexts. **CoRAG (Chain-of-Retrieval)** implements iterative query decomposition: the LLM generates a sub-query via reasoning, retrieves chunks for that sub-query, then decides via aggregation reasoning whether it has sufficient information or needs another retrieval round. This loop continues until the LLM signals completion.

The critical insight is that **CoRAG outperforms the other variants specifically under documentation heterogeneity** — when the vector store contains mixed database schemas, API endpoint specs, and business object descriptions simultaneously. CoRAG achieved 10.29% exact match vs. 7.45% for Standard RAG on combined tasks (p < 0.05), driven by its ability to decompose ambiguous queries into targeted sub-queries that pull the right document types. For single-modality tasks (SQL-only or API-only), the simpler Standard RAG is often sufficient.

## Step-by-Step Workflow

1. **Chunk your documentation into semantic units.** Each database table's complete schema (name, description, columns with types) becomes one chunk. Each API endpoint path with all HTTP methods, parameters, and response schemas becomes one chunk. Add business-object-level descriptions as separate chunks that bridge both modalities. Use fixed-length chunking with ~10% overlap only for chunks exceeding your model's context budget (e.g., 8,000 tokens).

2. **Embed chunks into a vector store with separate collections.** Create three ChromaDB (or equivalent) collections: `db_schemas`, `api_endpoints`, and `business_objects`. Use a consistent embedding model (e.g., `text-embedding-3-small`, 1536 dimensions) with squared Euclidean distance similarity. This separation enables controlled retrieval depending on whether you want DB-only, API-only, or hybrid context.

3. **Select the RAG variant based on your documentation profile.** If your system handles only SQL generation with homogeneous schema docs, use Standard RAG (top-5 retrieval). If documentation is noisy or large, use Self-RAG with a relevance threshold of 0.2 to filter irrelevant chunks. If the system must handle both SQL and API calls from a single hybrid document store, use CoRAG with iterative sub-query decomposition.

4. **Build the dynamic task classifier.** Before generating code, classify whether the user request requires data retrieval (SQL SELECT) or data modification (REST API POST/PUT/PATCH/DELETE). Implement this as a preliminary LLM call or rule-based classifier. The paper found classification accuracy exceeds 95% because linguistic signals (e.g., "show me" vs. "update the") reliably distinguish intent.

5. **Construct task-specific prompt templates with strict constraints.** For SQL tasks: restrict output to SELECT statements only, include the retrieved schema chunks, enforce column/table name fidelity, and require a fixed reference date for temporal queries. For API tasks: restrict to mutation methods (POST/PUT/PATCH/DELETE), include endpoint specs with parameter schemas, and require valid JSON body payloads.

6. **Implement CoRAG's iterative retrieval loop (for hybrid systems).** The loop: (a) LLM generates a reasoning step and a targeted sub-query; (b) retrieve top-5 chunks for that sub-query from all collections; (c) LLM performs aggregation reasoning — does it have enough context?; (d) if not, loop back to (a) with accumulated context; (e) if yes, generate the final SQL or API call. Cap iterations (3-5) to prevent runaway loops.

7. **Add self-validation to the generation prompt.** Instruct the LLM to verify its output before returning: check that SQL references only columns present in retrieved schemas, check that API calls use valid endpoint paths and required parameters, and flag any assumptions made about missing information.

8. **Validate generated outputs against execution environments.** Parse SQL with a SQL parser (e.g., `sqlparse`) and execute against a mock/shadow database. Validate API calls by sending to a mock server (e.g., Postman mock) and checking for status 200/201/204. Log failures for retrieval tuning.

9. **Evaluate with layered metrics.** Track exact match accuracy (binary, order-invariant), component match accuracy (partial credit for correct sub-parts like correct table but wrong WHERE clause), execution accuracy (does it run?), and endpoint retrieval accuracy (correct path regardless of parameters). Use paired two-tailed t-tests to compare configurations.

10. **Iterate on chunk quality, not just retrieval strategy.** When accuracy plateaus, improve documentation chunks: add column descriptions, annotate business rules in schema comments, enrich API endpoint descriptions with usage examples. Better chunks yield more gains than retrieval algorithm tweaks.

## Concrete Examples

**Example 1: Building a text-to-SQL pipeline for a product database**

User: "I have a PostgreSQL database with tables for products, orders, and customers. I want users to ask questions in English and get SQL queries back."

Approach:
1. Export each table's schema as a JSON chunk: `{"table": "orders", "columns": [{"name": "order_id", "type": "INTEGER", "description": "Primary key"}, ...], "description": "Customer purchase orders with line items"}`
2. Embed all schema chunks into a ChromaDB `db_schemas` collection
3. Since this is SQL-only with homogeneous docs, use Standard RAG with top-5 retrieval
4. Build the prompt template:

```python
SYSTEM_PROMPT = """You are a SQL query generator. Given the user's question and the
database schema context below, generate a single SELECT statement.

Rules:
- Use ONLY tables and columns present in the provided schema context
- Reference date for relative time expressions: {current_date}
- Output the SQL query inside a ```sql code block
- Do NOT use INSERT, UPDATE, DELETE, or DDL statements

Schema Context:
{retrieved_chunks}
"""
```

5. Retrieval pipeline:

```python
import chromadb

client = chromadb.PersistentClient(path="./vector_store")
collection = client.get_collection("db_schemas")

def generate_sql(user_question: str) -> str:
    results = collection.query(query_texts=[user_question], n_results=5)
    chunks = "\n\n".join(results["documents"][0])
    prompt = SYSTEM_PROMPT.format(
        current_date="2026-02-12",
        retrieved_chunks=chunks
    )
    return call_llm(system=prompt, user=user_question)
```

Output for "How many orders were placed last month?":
```sql
SELECT COUNT(*) FROM orders
WHERE order_date >= '2026-01-01' AND order_date < '2026-02-01';
```

**Example 2: Hybrid SQL + API system with CoRAG**

User: "Build a system where users can ask 'show me all accounts for customer 42' (returns SQL) or 'close account 789' (returns an API call), and it figures out which to use."

Approach:
1. Create three vector collections: `db_schemas` (table definitions), `api_endpoints` (REST specs), `business_objects` (domain descriptions)
2. Implement the task classifier:

```python
CLASSIFIER_PROMPT = """Classify the user request as either RETRIEVAL or MODIFICATION.
- RETRIEVAL: User wants to read/query/view/list/show data -> will generate SQL
- MODIFICATION: User wants to create/update/delete/close/transfer -> will generate API call
Respond with exactly one word: RETRIEVAL or MODIFICATION"""

def classify_task(user_input: str) -> str:
    return call_llm(system=CLASSIFIER_PROMPT, user=user_input).strip()
```

3. Implement CoRAG iterative retrieval for the hybrid store:

```python
def corag_retrieve(user_input: str, max_iterations: int = 4) -> list[str]:
    accumulated_chunks = []
    for i in range(max_iterations):
        sub_query = call_llm(
            system="Generate a focused retrieval sub-query to find documentation "
                   "needed to answer this request. Consider what you still need.",
            user=f"Original request: {user_input}\n"
                 f"Already retrieved:\n{accumulated_chunks}"
        )
        new_chunks = search_all_collections(sub_query, n=5)
        accumulated_chunks.extend(new_chunks)

        has_enough = call_llm(
            system="Do you have sufficient schema/API documentation to generate "
                   "the correct SQL or API call? Answer YES or NO with reasoning.",
            user=f"Request: {user_input}\nContext:\n{accumulated_chunks}"
        )
        if "YES" in has_enough.upper():
            break
    return accumulated_chunks
```

4. Route to the appropriate generator based on classification, using CoRAG-retrieved context

Output for "show me all accounts for customer 42":
```json
{"task_type": "RETRIEVAL", "output": "SELECT * FROM accounts WHERE customer_id = 42;"}
```

Output for "close account 789":
```json
{
  "task_type": "MODIFICATION",
  "output": {
    "method": "PATCH",
    "endpoint": "/api/v1/accounts/789",
    "body": {"status": "CLOSED", "closed_date": "2026-02-12"}
  }
}
```

**Example 3: Adding Self-RAG relevance filtering to reduce noise**

User: "My RAG pipeline retrieves irrelevant schema chunks and the LLM generates queries referencing wrong tables. How do I fix this?"

Approach:
1. Add a relevance scoring step between retrieval and generation:

```python
def self_rag_filter(user_query: str, chunks: list[str], threshold: float = 0.2) -> list[str]:
    relevant = []
    for chunk in chunks:
        score = call_llm(
            system="Rate the relevance of this documentation chunk to the user's "
                   "query on a scale of 0.0 to 1.0. Return only the number.",
            user=f"Query: {user_query}\n\nChunk:\n{chunk}"
        )
        if float(score.strip()) >= threshold:
            relevant.append(chunk)
    return relevant if relevant else chunks[:2]  # fallback to top-2 if all filtered
```

2. Replace the direct top-5 retrieval with: retrieve top-5, then filter with Self-RAG
3. This eliminates noisy chunks that share vocabulary but describe unrelated tables/endpoints

## Best Practices

- **Do:** Structure each documentation chunk as a self-contained semantic unit — one table schema or one API endpoint per chunk. The paper's best results came from chunks that didn't require cross-referencing to be understood.
- **Do:** Include business object descriptions as bridging chunks. These high-level domain descriptions (e.g., "An Account represents a customer's financial instrument") help the retriever connect user intent vocabulary to technical schema/API terminology.
- **Do:** Use CoRAG when your document store mixes schema types (DB + API + prose). Standard RAG degrades under documentation heterogeneity because a single embedding query can't target both schema chunks and API specs effectively.
- **Do:** Enforce strict output constraints in prompts (SELECT-only for SQL, mutation-only for API). This prevents the LLM from generating syntactically valid but semantically wrong outputs.
- **Avoid:** Skipping retrieval entirely, even with powerful models. GPT-5 achieved 0% exact match without retrieval on domain-specific enterprise schemas. The schema/API context is not optional.
- **Avoid:** Using Self-RAG with aggressive thresholds (> 0.5) in hybrid documentation contexts. Over-filtering removes chunks from the less-dominant modality, degrading performance below Standard RAG.

## Error Handling

- **Empty retrieval results:** If the vector store returns no chunks above the similarity threshold, fall back to retrieving the top-3 chunks regardless of score and append a warning to the prompt instructing the LLM to flag uncertainty.
- **Task misclassification:** When the classifier is uncertain (e.g., "get the latest transaction and mark it reviewed" contains both read and write), implement a confidence threshold. If classification confidence is below 0.8, generate both SQL and API outputs and let the user confirm.
- **SQL execution failures:** Parse generated SQL with `sqlparse` before execution. If parsing fails, re-prompt with the error message and the original schema context. Common failures: wrong column names (retrieval missed the right table), unsupported functions (LLM hallucinated a function not in the target DB dialect).
- **API validation failures:** Check that the generated endpoint path exists in your API spec and that required body parameters are present. Missing parameters are the most common API generation error — enrich endpoint chunks with parameter descriptions and required/optional annotations.
- **CoRAG infinite loops:** Always cap iteration count (3-5 rounds). If the LLM never signals "sufficient context," force generation after the cap and add a low-confidence flag to the output.

## Limitations

- Exact match accuracy remains low even with CoRAG (10-15% for complex queries). Execution accuracy is much higher (up to 79%), meaning generated code often works but differs syntactically from the reference. Design evaluation around execution correctness, not string matching.
- The approach assumes documentation exists and is reasonably complete. If your schema has undocumented columns or your API spec is missing endpoints, retrieval cannot compensate.
- CoRAG's iterative retrieval adds latency — each iteration is a full LLM call plus vector search. For latency-sensitive applications, Standard RAG or Self-RAG with pre-computed relevance caches may be preferable.
- The task classifier works well for clear retrieval-vs-modification intent but struggles with compound requests ("show me the account and then close it"). These require request decomposition before classification.
- Results were validated on SAP Transactional Banking with 22 tables and 174 API endpoints. Scaling behavior to schemas with hundreds of tables or thousands of endpoints is not established.

## Reference

[Evaluating Retrieval-Augmented Generation Variants for Natural Language-Based SQL and API Call Generation](https://arxiv.org/abs/2602.07086v1) — Marketsmüller, Martin, Schlippe (2026). Key sections: Section 3 for RAG variant architectures, Section 4 for the evaluation framework and dataset construction, Section 5 for comparative results showing CoRAG's advantage under documentation heterogeneity.