---
name: "automated-customization-enterprise-code"
description: "Customize LLMs for enterprise code repositories using semantic scopes -- automatically partition codebases into meaningful units (function bodies, conditionals, loops, logging patterns) and generate fine-tuning data pairs for repository-specific code completion. Use when: 'customize a model for our codebase', 'generate fine-tuning data from our repo', 'improve code completion for private code', 'extract semantic scopes from source files', 'build training pairs for code LLM', 'set up RAG for enterprise code completion'."
---

This skill enables Claude to apply the Semantic Scopes technique for automated LLM customization to enterprise code repositories. Rather than treating a codebase as flat text, the approach identifies semantically meaningful code units -- function bodies, loop bodies, conditional blocks, logging patterns, function calls -- and uses their boundaries to generate high-quality fine-tuning data pairs and RAG contexts. The result: moderately sized models fine-tuned with this method outperform much larger uncustomized models on repository-specific code completion.

## When to Use

- When the user wants to fine-tune a code LLM (Granite, Llama, Qwen, CodeLlama, StarCoder) on a private enterprise repository
- When the user asks to generate training data pairs from a codebase for supervised fine-tuning
- When the user needs to set up retrieval-augmented generation (RAG) for code completion against a proprietary repo
- When the user wants to identify and extract semantic scopes (function bodies, conditionals, loops) from source files
- When the user asks to improve code completion accuracy for internal APIs, logging conventions, or domain-specific patterns
- When the user wants to evaluate customized vs. uncustomized model performance using Levenshtein distance metrics
- When the user has a C/C++, Java, Python, or similar codebase and needs structured scope extraction

## Key Technique

**Semantic scopes** are recursive, logically cohesive units of code whose boundaries typically coincide with syntactic scopes in languages that use block delimiters. In Java and C/C++, scope candidates include: function/method definitions, `for` loop bodies, `while` loop bodies, `if`/`else` conditional bodies, and code between matching brackets or parentheses. The insight is that these scopes represent the natural units developers write and read -- completing an entire scope (not just the next token) is what developers actually need from code assistants.

The pipeline has three phases: **Data Selection** (collect files by language, store metadata), **Scope Identification** (parse each file to extract scope candidates with byte-offset boundaries, then filter by size: 50-1,000 bytes for the scope, minimum 200 bytes of preceding context), and **Pair Generation** (create query-label pairs where the query is the code prefix up to the scope start, and the label is the complete scope text terminated with an end-of-text token). To improve robustness, the system generates additional pairs starting at random byte offsets within the preceding context, not only at the exact scope boundary.

For RAG-based customization, code prefixes are embedded using a sentence transformer (e.g., `all-MiniLM-L6-v2`), stored in a vector index, and the top-k nearest neighbors (typically 3-5) are prepended as context before the completion query. Fine-tuning uses standard supervised training with the Adam optimizer. A practical finding: offloading the optimizer to CPU in 32-bit float arithmetic reduces gradient norm spikes during training.

## Step-by-Step Workflow

1. **Inventory the repository.** Recursively scan the target codebase and classify files by programming language. Store file paths, sizes, and language tags in a JSON manifest. Exclude vendored code, generated files, and test fixtures unless explicitly requested.

2. **Parse files and extract scope candidates.** For each source file, use a parser (tree-sitter recommended) to identify all syntactic blocks: function/method bodies, `for` bodies, `while` bodies, `if` bodies, `else` bodies, and any nested scoped blocks. Record each scope's start byte offset, end byte offset, scope type, and parent scope.

3. **Filter scopes by size constraints.** Retain only scopes between 50 and 1,000 bytes in length. Require at least 200 bytes of preceding code in the same file before the scope start. Discard trivially small scopes (single-line returns, empty blocks) and overly large ones (entire class definitions spanning thousands of lines).

4. **Generate primary training pairs.** For each retained scope, create a pair: the **query** is the file content from the start (or a context window of ~2,048 tokens) up to the scope's start byte; the **label** is the scope content plus an end-of-text token. This teaches the model to complete the exact scope when positioned at its boundary.

5. **Generate augmented training pairs.** For each primary pair, create 1-3 additional pairs by selecting random byte offsets within the 200+ bytes preceding the scope start. The label remains the content from that offset through the end of the scope. This prevents the model from only learning to complete at exact scope boundaries.

6. **Build the fine-tuning dataset.** Combine all pairs, shuffle, and split into train/validation sets (e.g., 95/5). Target 200,000-300,000 pairs for repositories of 10,000+ files. Export in JSONL format with fields: `prefix`, `completion`, `scope_type`, `file_path`, `language`.

7. **Configure and run fine-tuning.** Use the Hugging Face Trainer with Adam optimizer (offload to CPU for 32-bit stability). Train for 3-10 epochs with a learning rate around 1e-5. Monitor gradient norms -- spikes indicate the need for CPU optimizer offloading or gradient clipping.

8. **Build the RAG index (optional, complementary).** Embed all training pair prefixes using `all-MiniLM-L6-v2` or a code-specific embedding model. Store embeddings in a vector database (FAISS, Chroma, or similar). At inference time, embed the current code prefix, retrieve the top 3-5 nearest neighbors, and prepend them as context before the completion query.

9. **Create evaluation test cases.** Manually curate 15-40 test cases per scope category (`func_body`, `if_body`, `for_body`, `else_body`, `logging`, `func_call`). For each, record the prefix and the ground-truth scope completion.

10. **Evaluate with Levenshtein distance.** Compute two metrics for each prediction: **Full** (edit distance between the complete prediction and ground truth) and **Opt** (minimum edit distance across all prefixes of the prediction, which forgives over-generation). Compare base model vs. fine-tuned model vs. RAG-augmented model across all scope categories.

## Concrete Examples

**Example 1: Generating fine-tuning data from a Java repository**

User: "I have a Java microservices repo at `./services/` with about 15K files. Help me generate fine-tuning data for code completion."

Approach:
1. Scan `./services/` recursively, collecting all `.java` files into a manifest
2. Parse each file with tree-sitter-java to extract method bodies, if/else blocks, for/while loops, and try/catch blocks
3. Filter: keep scopes 50-1,000 bytes with 200+ bytes preceding context
4. For each scope, generate the primary prefix-completion pair plus 2 random-offset augmented pairs
5. Export as JSONL

Output (sample pair from the JSONL):
```json
{
  "prefix": "package com.acme.orders;\n\nimport com.acme.logging.AcmeLogger;\nimport com.acme.models.Order;\n\npublic class OrderService {\n    private final AcmeLogger logger;\n    \n    public OrderStatus processOrder(Order order) {\n        logger.info(\"Processing order\", order.getId());\n        ",
  "completion": "if (order.getItems().isEmpty()) {\n            logger.warn(\"Empty order received\", order.getId());\n            return OrderStatus.REJECTED;\n        }<|endoftext|>",
  "scope_type": "if_body",
  "file_path": "services/order-service/src/main/java/com/acme/orders/OrderService.java",
  "language": "java"
}
```

**Example 2: Setting up RAG for C++ code completion**

User: "Our C++ codebase uses custom logging macros like `PD_LOG_BEGIN`/`PD_LOG_END`. Models always get these wrong. Can we fix this with RAG?"

Approach:
1. Extract all semantic scopes from the C++ repo, prioritizing `logging`-category scopes containing the custom macros
2. Generate prefix-completion pairs focused on logging patterns
3. Embed all prefixes with `all-MiniLM-L6-v2` and index in FAISS
4. At inference time, when the developer's cursor is inside or near a logging block, retrieve the 5 nearest logging-scope examples
5. Prepend retrieved examples as few-shot context before the completion query

Output (RAG-augmented prompt sent to the model):
```
// Retrieved context example 1:
PD_LOG_BEGIN(logger, INFO)
  << "Connection established: " << conn.getId()
  << " endpoint=" << conn.getEndpoint()
PD_LOG_END

// Retrieved context example 2:
PD_LOG_BEGIN(logger, WARN)
  << "Retry attempt " << retryCount
  << " for request=" << req.getId()
PD_LOG_END

// Current code to complete:
void handleTimeout(const Request& req, int elapsed) {
    if (elapsed > threshold) {
        PD_LOG_BEGIN(logger, ERROR)
```

**Example 3: Evaluating fine-tuned model improvement**

User: "I fine-tuned Granite-8B on our repo for 5 epochs. How do I measure if it actually improved?"

Approach:
1. Create a test set with 15-40 manually curated examples per scope category: `func_body`, `if_body`, `for_body`, `else_body`, `logging`, `func_call`
2. Run both the base model and fine-tuned model on each test case with greedy decoding
3. Compute Full Levenshtein distance (prediction vs. ground truth) and Opt distance (best-matching prefix of prediction vs. ground truth)
4. Aggregate by scope category and compare

Output (evaluation summary table):
```
Category    | N   | Base (Opt) | FT (Opt) | Improvement
------------|-----|------------|----------|------------
func_body   |  38 |       156  |       72 |     53.8%
if_body     |  30 |       112  |       48 |     57.1%
for_body    |  16 |        89  |       41 |     53.9%
else_body   |  20 |        95  |       52 |     45.3%
logging     |  36 |       347  |      129 |     62.8%
func_call   |  80 |        84  |       42 |     50.0%
```

## Best Practices

- **Do:** Use tree-sitter or a language-aware parser for scope extraction. Regex-based approaches miss nested scopes and produce incorrect boundaries.
- **Do:** Include the `scope_type` label in your training metadata so you can evaluate per-category performance and diagnose which patterns the model struggles with.
- **Do:** Generate augmented pairs at random offsets, not just at exact scope boundaries. This makes the fine-tuned model robust to cursor positions that don't align perfectly with scope starts.
- **Do:** Combine RAG and fine-tuning for best results. Fine-tuning teaches general repository patterns; RAG injects specific relevant examples at inference time.
- **Avoid:** Including generated code, vendored dependencies, or test fixtures in training data unless the goal is specifically to complete test code. These pollute the signal from production code patterns.
- **Avoid:** Training on scopes shorter than 50 bytes (trivially simple, low signal) or longer than 1,000 bytes (too much to predict accurately, introduces noise during training).
- **Avoid:** Using a single global Levenshtein score for evaluation. Always break down by scope category -- a model might excel at function bodies but fail on logging patterns, and the aggregate score hides this.

## Error Handling

- **Parser failures on malformed code:** Some enterprise repos contain preprocessor-heavy C/C++ or incomplete files. Skip files that fail to parse and log them for manual review. Do not let a single parse error halt the entire pipeline.
- **Gradient norm spikes during fine-tuning:** If loss spikes or gradients explode, switch the optimizer to CPU-offloaded 32-bit mode. This is a known issue when fine-tuning code models on heterogeneous repo data.
- **Embedding model mismatch for RAG:** If retrieved neighbors are semantically irrelevant (e.g., syntactically similar but functionally unrelated code), try a code-specific embedding model like `codebert-base` or `unixcoder-base` instead of general-purpose sentence transformers.
- **Insufficient training pairs:** Repos under 5,000 files may produce fewer than 50,000 scope pairs. In this case, increase the augmented pairs per scope from 2 to 5, or relax the minimum preceding-context requirement from 200 to 100 bytes.
- **Evaluation test set too small:** Categories with fewer than 10 test cases produce unreliable metrics. Prioritize curating at least 15 examples per scope type before drawing conclusions.

## Limitations

- **Language coverage:** The technique is demonstrated on Java and C/C++ where syntactic scopes map cleanly to semantic boundaries. For languages like Python (indentation-based) or Lisp (parenthesis-heavy), the scope extraction logic needs significant adaptation.
- **Scope size ceiling:** The 1,000-byte upper limit means this approach targets short-to-medium completions (roughly 20-40 lines). It is not designed for generating entire classes or multi-function modules.
- **Static analysis only:** Semantic scopes are identified via syntactic structure. The approach does not capture runtime semantics, data flow, or cross-file dependencies. A function that calls another function in a different file won't have that dependency reflected in its scope.
- **Manual test curation:** Automated evaluation is limited to Levenshtein distance. Meaningful assessment of whether completions are *correct* (compiles, passes tests, matches intent) still requires human review.
- **Training data scale:** The approach requires repositories with at least several thousand files to generate enough diverse training pairs. Small projects or single-file scripts won't benefit.

## Reference

[Finkler et al., 2026. "Automated Customization of LLMs for Enterprise Code Repositories Using Semantic Scopes." arXiv:2602.05780](https://arxiv.org/abs/2602.05780v1) -- Focus on Section 3 (Semantic Scope definition and extraction), Section 4 (training pair generation pipeline), and Table 2 (per-category evaluation results showing fine-tuning improvements).