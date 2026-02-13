---
name: "context-augmented-code-generation-programming-know"
description: |
  Enhance code generation with Programming Knowledge Graph (PKG) retrieval, tree pruning, and re-ranking.
  Uses fine-grained knowledge graph nodes (function blocks, code snippets, documentation paths) to retrieve
  precisely relevant context, then prunes and re-ranks results to minimize hallucination.

  Trigger phrases:
  - "Generate code using knowledge graph context"
  - "Help me solve this coding problem with relevant examples"
  - "Retrieve similar code patterns for this function"
  - "Augment my code generation with external knowledge"
  - "Find relevant code context and generate a solution"
  - "Use PKG-style retrieval for this programming task"
---

# Context-Augmented Code Generation Using Programming Knowledge Graphs

This skill enables Claude to apply the Programming Knowledge Graph (PKG) technique from Seddik et al. (2026) to real code generation tasks. Instead of treating external code knowledge as flat documents, Claude structures relevant knowledge into a hierarchical graph of fine-grained nodes (function names, implementation blocks, code fragments, documentation paths), retrieves the most relevant nodes via semantic similarity, prunes irrelevant branches, and re-ranks candidate solutions to suppress hallucination. The result is more accurate, contextually grounded code generation, especially for complex problems where naive LLM generation fails.

## When to Use

- When the user provides a complex coding problem and you have access to a codebase, library docs, or code examples that could inform the solution
- When generating code that depends on APIs, patterns, or idioms found elsewhere in the project
- When the user asks you to find similar code patterns in a repository and use them to generate a new function
- When a straightforward generation attempt produces hallucinated API calls, incorrect method signatures, or wrong patterns
- When solving algorithmic problems where retrieving known solution fragments (sorting routines, graph traversals, DP patterns) would improve correctness
- When the user wants you to search documentation or code examples before generating, rather than relying purely on parametric knowledge

## Key Technique

**Programming Knowledge Graphs (PKGs)** decompose code and text into fine-grained graph nodes rather than chunking documents at arbitrary boundaries. For code, an AST-based parser (FunctionAnalyzer) extracts a hierarchy: function name nodes -> implementation nodes -> constituent code block nodes. For text/documentation, JSON-like structures are converted into directed acyclic graphs of (path, value) pairs. Each node gets a semantic embedding (the paper uses VoyageCode2). This granularity means retrieval can return a single relevant code block instead of an entire file, dramatically improving precision.

**Tree Pruning** addresses the problem of retrieving too much context. After identifying candidate nodes via cosine similarity, the system models retrieved subgraphs as DAGs and iteratively removes branches. Each pruned variant is re-embedded and scored against the query; the variant maximizing similarity is selected. This eliminates tangential code that would confuse the generator.

**Re-Ranking with Non-RAG Integration** is the hallucination defense. Candidate solutions go through a three-stage pipeline: (1) syntactic validation via AST parsing to eliminate malformed code, (2) runtime execution to remove candidates that crash, and (3) semantic similarity scoring against the original query. Critically, baseline (non-RAG) solutions are included in the candidate pool, so if retrieval introduces noise, the system can fall back to the model's own knowledge. This yielded up to 20% pass@1 gains on HumanEval and 34% improvement over baselines on MBPP while causing minimal regression on problems already solvable without retrieval.

## Step-by-Step Workflow

1. **Decompose the problem statement** into a clear natural language query and identify what type of code artifact is needed (function, class, module, algorithm). Extract key concepts: data structures involved, algorithms referenced, APIs mentioned.

2. **Build a local knowledge graph from available context.** Scan the user's codebase, imported libraries, or provided documentation. For each code file, extract functions and classes using AST parsing. For each function, create three node levels: function signature, full implementation, and individual code blocks (loops, conditionals, expressions). For documentation, extract hierarchical (path, value) pairs.

3. **Embed the query and all graph nodes** using a consistent semantic representation. In practice, generate a concise textual description of each node (e.g., "function `merge_sort` that recursively splits and merges a list") and compute similarity against the user's problem statement. Prioritize nodes whose descriptions share key terms and semantic meaning with the query.

4. **Retrieve the top-k most relevant nodes** using cosine similarity between the query embedding and node embeddings. Use three retrieval granularities in parallel:
   - **Block-wise**: individual code blocks (highest granularity, best for specific patterns)
   - **Function-wise**: complete function implementations (good for holistic examples)
   - **Path-value**: documentation entries (good for API usage and parameter details)

5. **Prune irrelevant branches** from the retrieved subgraph. For each retrieved node, examine its parent and sibling nodes. Remove branches where the combined embedding diverges from the query. Re-score the pruned subgraph to confirm it remains relevant. The goal is a minimal, focused context window.

6. **Format the retrieved context** into a structured prompt. Place the most relevant code blocks closest to the generation point (Fill-in-the-Middle style). Include function signatures and docstrings as prefix context, and place the user's specific requirements as a suffix. This sandwich structure guides the model to generate code consistent with both the examples and the requirements.

7. **Generate multiple candidate solutions.** Produce at least 2-3 candidates: one using the PKG-augmented context, one without any retrieved context (baseline), and optionally one with a different retrieval granularity. This ensures the re-ranker has diverse candidates to choose from.

8. **Apply the three-stage re-ranking filter:**
   - **Stage 1 (Syntactic):** Parse each candidate with `ast.parse()`. Discard any candidate that raises a `SyntaxError`.
   - **Stage 2 (Runtime):** If test cases are available, execute each candidate. Discard those that raise unhandled exceptions or fail assertions.
   - **Stage 3 (Semantic):** Score surviving candidates by semantic similarity to the original query and by alignment with the retrieved context. Select the highest-scoring candidate.

9. **Return the top-ranked solution** with an explanation of which retrieved context informed it. If the baseline (non-RAG) solution won re-ranking, note that retrieval did not improve on the model's own knowledge for this particular problem.

10. **Iterate if needed.** If no candidate passes all three stages, relax the retrieval scope (retrieve more nodes or use a coarser granularity), regenerate candidates, and re-rank again.

## Concrete Examples

**Example 1: Generating a function using codebase patterns**

User: "Write a function to validate and normalize email addresses, following the patterns used in our `utils/validators.py` module."

Approach:
1. Parse `utils/validators.py` with AST to extract all validator functions as graph nodes (e.g., `validate_phone`, `validate_url`, `normalize_string`).
2. Break each function into block-level nodes: input validation blocks, regex patterns, normalization steps, return statements.
3. Query: "validate and normalize email address." Retrieve the most similar blocks — likely the regex pattern from `validate_url`, the normalization logic from `normalize_string`, and the error-handling pattern from `validate_phone`.
4. Prune: drop the URL-specific regex details, keep the structural pattern (compile regex, match, raise on failure, return normalized).
5. Generate two candidates: one using retrieved context, one baseline.
6. Re-rank: both parse correctly, test with sample emails, select the one with correct RFC 5322 handling.

Output:
```python
import re

def validate_and_normalize_email(email: str) -> str:
    """Validate email format and return normalized version."""
    if not isinstance(email, str):
        raise TypeError(f"Expected str, got {type(email).__name__}")
    email = email.strip().lower()
    pattern = re.compile(
        r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    )
    if not pattern.match(email):
        raise ValueError(f"Invalid email address: {email}")
    return email
```

**Example 2: Solving an algorithmic problem with retrieved code fragments**

User: "Implement a function that finds the longest increasing subsequence in an array. I know there's an O(n log n) approach."

Approach:
1. Build nodes from known algorithm patterns: binary search implementations, dynamic programming templates, patience sorting descriptions.
2. Query: "longest increasing subsequence O(n log n)." Top retrieved nodes: a binary search block for insertion point finding, a DP template for subsequence tracking, documentation on the patience sorting analogy.
3. Prune: remove the full patience sorting explanation (too verbose), keep the binary search block and the tails-array update pattern.
4. Format context: place the binary search utility as prefix, the problem statement as the generation target.
5. Generate candidates with and without context.
6. Re-rank: run against test cases `[10, 9, 2, 5, 3, 7, 101, 18]` -> expected length 4. Select the passing candidate.

Output:
```python
import bisect

def length_of_lis(nums: list[int]) -> int:
    """Find length of longest increasing subsequence in O(n log n)."""
    tails = []
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

**Example 3: API-dependent code generation with documentation retrieval**

User: "Write a function to upload a file to S3 with server-side encryption, using the patterns from our existing `storage/` module."

Approach:
1. Parse `storage/` module files. Extract function nodes for existing upload/download operations and documentation nodes for AWS SDK configuration.
2. Retrieve path-value nodes: `storage.config.encryption -> "AES256"`, `storage.upload.retry_policy -> exponential_backoff`, and function nodes for `upload_object()` and `configure_client()`.
3. Prune: drop download-related branches and unrelated storage backends.
4. Generate candidate using the project's established patterns (client initialization, error handling, retry logic) with the addition of `ServerSideEncryption` parameter.
5. Re-rank: AST-validate, verify against mocked S3 client if available, score semantic alignment.

Output:
```python
def upload_file_encrypted(
    bucket: str,
    key: str,
    file_path: str,
    encryption: str = "AES256",
) -> dict:
    """Upload file to S3 with server-side encryption."""
    client = get_s3_client()  # reuses project's client factory
    with open(file_path, "rb") as f:
        response = client.put_object(
            Bucket=bucket,
            Key=key,
            Body=f,
            ServerSideEncryption=encryption,
        )
    return response
```

## Best Practices

- **Do:** Decompose code into the finest practical granularity (individual blocks, not whole files). The paper's key insight is that block-level retrieval (individual loops, conditionals, expressions) outperforms function-level or document-level retrieval for precision.
- **Do:** Always include a non-RAG baseline candidate in your re-ranking pool. The paper shows this prevents regression — retrieval sometimes introduces noise, and the baseline acts as a safety net.
- **Do:** Apply AST validation as the first filter before any runtime checks. It is cheap and eliminates obviously broken candidates immediately.
- **Do:** Use the Fill-in-the-Middle prompt format when inserting retrieved context — place relevant code as prefix, the generation target in the middle, and constraints as suffix.
- **Avoid:** Stuffing the entire retrieved subgraph into the prompt without pruning. The paper demonstrates that pruning irrelevant branches is essential; excess context degrades generation quality.
- **Avoid:** Relying on a single retrieval granularity. Use block-wise for specific patterns, function-wise for structural examples, and path-value for documentation. Different problems benefit from different granularities.
- **Avoid:** Skipping the runtime validation stage when test cases are available. Syntactic correctness does not guarantee functional correctness — execution-based filtering is the strongest signal.

## Error Handling

| Problem | Cause | Resolution |
|---|---|---|
| Retrieved context is semantically distant from the query | Knowledge graph lacks relevant nodes for this domain | Fall back to baseline (non-RAG) generation; the re-ranker will naturally prefer the non-augmented candidate |
| All candidates fail AST validation | Malformed retrieved context corrupting generation | Re-generate without retrieved context; investigate whether the source code nodes contain syntax errors |
| Re-ranking selects a wrong candidate | Test cases are insufficient or missing | Add edge-case tests; if no tests exist, rely on semantic scoring and flag the solution as needing manual review |
| Retrieval returns too many nodes, exceeding context window | Pruning threshold too permissive | Tighten the similarity threshold; limit to top-3 nodes per granularity level |
| Retrieved code uses deprecated APIs or incompatible versions | Source knowledge graph is stale | Validate retrieved code against current dependency versions before including it in the prompt |

## Limitations

- **Requires a knowledge base to retrieve from.** This technique shines when there is a codebase, documentation set, or curated code corpus to build the graph from. For purely novel problems with no analogous prior code, the approach reduces to standard generation.
- **Graph construction has upfront cost.** The paper reports 301 minutes and 12.5 GB for 143K code samples on Neo4j. For ad-hoc tasks, a lightweight in-memory approximation (AST-parsed function index with embeddings) is more practical than a full graph database.
- **Re-ranking requires test cases for maximum effectiveness.** The runtime validation stage (Stage 2) depends on executable test cases. Without them, the pipeline falls back to syntactic + semantic filtering only, which is weaker.
- **Granularity sweet spot varies by task.** Block-level retrieval is best for pattern matching; function-level is better for understanding overall structure. There is no universal setting — the user or system must choose or try multiple.
- **Not designed for multi-file generation.** The PKG approach targets single-function or single-class generation. For large-scale code scaffolding across many files, other techniques (agentic planning, template-based generation) are more appropriate.

## Reference

**Paper:** Seddik et al., "Context-Augmented Code Generation Using Programming Knowledge Graphs," arXiv:2601.20810v1 (2026).
https://arxiv.org/abs/2601.20810v1

**Key takeaway:** Fine-grained graph-structured retrieval (block-level AST nodes + pruning) combined with a three-stage re-ranker that includes non-RAG baselines yields up to 20% pass@1 improvement on HumanEval and 34% on MBPP, while avoiding regression on problems the model already solves correctly.

**Replication package:** https://github.com/iamshahd/ProgrammingKnowledgeGraph