---
name: "greprag-empirical-study-optimization"
description: "Lightweight, index-free repository-level code retrieval using ripgrep for context-aware code completion. Uses LLM-generated ripgrep commands with identifier-weighted re-ranking and structure-aware deduplication. Trigger phrases: 'find relevant code across the repo', 'retrieve cross-file context for completion', 'grep the codebase for related code', 'use ripgrep to find dependencies', 'get repo context for this function', 'search the project for related definitions'"
---

# GrepRAG: Index-Free Lexical Retrieval for Repository-Level Code Completion

This skill enables Claude to retrieve relevant cross-file context from a repository using only ripgrep — no embeddings, no vector databases, no pre-built indexes. By generating targeted ripgrep commands from the code under completion, then applying identifier-weighted BM25 re-ranking and structure-aware deduplication, Claude can surface the exact definitions, usages, and patterns needed to complete code accurately. This approach matches or outperforms graph-based and semantic retrieval systems while running 35x faster on large repositories.

## When to Use

- When completing code that depends on classes, methods, or variables defined in other files across the repository
- When the user asks to "find all related code" or "get context" for a function they're writing
- When implementing a method that needs to match an interface, base class, or protocol defined elsewhere
- When completing a function call where argument types or return signatures are defined in another module
- When the user is working in a large repo and needs relevant context without building or maintaining an index
- When filling in code that references configuration objects, enums, or constants from other files
- When the user asks to "search the codebase" for patterns related to code they're writing

## Key Technique

**Core Insight:** Simple lexical search with ripgrep, when guided by LLM intent analysis, retrieves code fragments that are more lexically precise and spatially closer to the completion site than sophisticated graph-based or embedding-based retrieval. The key is generating *intent-aware* search queries — not generic keyword searches, but targeted identifier lookups that follow the same patterns a developer would use when navigating unfamiliar code.

**The GrepRAG Pipeline:** Given local context around a cursor position, the LLM analyzes lexical features and latent dependencies to generate ~10 ripgrep commands targeting four categories: class names (36% of queries — retrieving type definitions and member structures), method names (41% — locating definitions and call sites), variable names (18% — surfacing type annotations and assignments), and other identifiers (5% — finding similar code patterns). About 23.5% of generated commands use fuzzy matching with wildcards (e.g., `rg "class.*ConfigModel"`). The retrieved snippets are then ranked and deduplicated before being added to the completion prompt.

**Post-Processing That Matters:** Raw ripgrep results suffer from two problems: (1) keyword ambiguity where common identifiers like `init`, `config`, or `run` produce floods of irrelevant matches, and (2) context fragmentation where overlapping snippets from the same file waste the context budget. GrepRAG addresses these with BM25 re-ranking (which uses inverse document frequency to suppress common terms and amplify rare, meaningful identifiers) and structure-aware deduplication (which merges adjacent or overlapping line ranges from the same file into coherent blocks). In ablation studies, deduplication alone provides a +3.32% EM gain — substantially more than re-ranking alone (+0.51%) — because reconstructing logical code flow eliminates redundancy and restores semantic continuity.

## Step-by-Step Workflow

1. **Extract local context.** Identify the code immediately surrounding the completion site — the current function, its imports, class context, and any visible type annotations or variable bindings within the same file. This is your `C_local`.

2. **Analyze intent and identify search targets.** From `C_local`, determine what cross-file information is needed. Classify each dependency into one of four categories:
   - *Class names*: When you see `obj.method()` or `class Foo(Bar)`, you need the definition of the type
   - *Method names*: When completing arguments or a method body, you need the method's signature and call sites
   - *Variable names*: When a variable's type or origin is unclear, you need its definition and assignments
   - *String/pattern identifiers*: When matching conventions or config keys, you need similar usage patterns

3. **Generate ripgrep commands.** Produce ~10 ripgrep commands targeting the identified dependencies. Use these patterns:
   - Exact identifier: `rg -n "class DataProcessor" --type py`
   - Fuzzy/wildcard: `rg -n "def process.*batch" --type py`
   - Definition lookup: `rg -n "def calculate_loss" --type py`
   - Usage lookup: `rg -n "calculate_loss\(" --type py`
   - Import tracing: `rg -n "from.*models.*import" --type py`
   Use `--type` to restrict to the relevant language. Use `-n` for line numbers. Use `-C 5` or similar context flags to capture surrounding lines.

4. **Execute commands and collect snippets.** Run each ripgrep command against the repository root. For each match, capture the file path, line numbers, and surrounding context (typically 5-15 lines around each match). Exclude the current file's completion site from results.

5. **Apply BM25 re-ranking.** Score each retrieved snippet against `C_local` using BM25 instead of naive Jaccard similarity. BM25's IDF component naturally suppresses ubiquitous tokens (`self`, `return`, `import`, `__init__`) while amplifying the weight of rare, task-specific identifiers. This is critical for filtering noise from ambiguous keywords.

6. **Select top candidates for deduplication.** Take the top 50% of BM25-ranked snippets as the deduplication candidate pool. This threshold balances coverage against noise — below 50% causes over-aggressive redundancy collapse; above 50% admits low-relevance fragments.

7. **Perform structure-aware deduplication.** Group snippets by file path. Within each file, parse line number ranges and identify overlapping or adjacent snippets (where one snippet's end line is within a few lines of another's start line). Merge these into single coherent blocks that preserve the logical flow of the code. This eliminates duplicate retention of overlapping regions and restores semantic continuity between definitions and their usage sites.

8. **Assemble final context within token budget.** Select the top-K deduplicated blocks, concatenating them into the retrieval context. Enforce a strict token limit (4096 tokens is the paper's default). Prioritize blocks by their BM25 scores. Format each block with its file path and line range as a header.

9. **Construct the completion prompt.** Place the retrieved cross-file context before the local context in the prompt, clearly delineated with file path labels. The LLM then generates the completion with full awareness of the relevant repository context.

10. **Validate the completion.** Check that the generated code references identifiers that actually exist in the retrieved context — method signatures match, class hierarchies are correct, and variable types are consistent with their definitions.

## Concrete Examples

**Example 1: Completing a method that calls a cross-file API**

```
User: "I'm writing a data pipeline and need to complete this method that uses
our DataLoader class from another module."

# Current file: pipeline/processor.py (cursor at line 15)
class BatchProcessor:
    def __init__(self, config: PipelineConfig):
        self.loader = DataLoader(config.data_path)

    def process_batch(self, batch_id: int):
        data = self.loader.  # <-- completion needed here

Approach:
1. Extract C_local: BatchProcessor class, DataLoader reference, PipelineConfig type
2. Identify targets: DataLoader class definition (class-name), PipelineConfig (class-name)
3. Generate ripgrep commands:
   - rg -n "class DataLoader" --type py
   - rg -n "def.*DataLoader" --type py -C 10
   - rg -n "class PipelineConfig" --type py
   - rg -n "data_path" --type py -C 3
   - rg -n "DataLoader\(" --type py
4. Execute and collect: Find DataLoader in data/loader.py with methods
   load_batch(), load_all(), get_schema()
5. BM25 re-rank: DataLoader class definition scores highest (rare identifier match)
6. Deduplicate: Merge DataLoader.__init__ and load_batch snippets from same file
7. Assemble context with DataLoader's full class interface

Output: Complete with `self.loader.load_batch(batch_id)` — matching the
actual method signature found in data/loader.py
```

**Example 2: Implementing an abstract method from a base class**

```
User: "I need to implement the transform method in my custom transformer class."

# Current file: transforms/custom.py
from transforms.base import BaseTransform

class NormalizeTransform(BaseTransform):
    def transform(self, data):  # <-- body completion needed

Approach:
1. Extract C_local: NormalizeTransform inherits BaseTransform, needs transform() body
2. Identify targets: BaseTransform definition (class-name), transform method (method-name),
   other BaseTransform subclasses for usage patterns (class-name + wildcard)
3. Generate ripgrep commands:
   - rg -n "class BaseTransform" --type py -C 15
   - rg -n "def transform.*self.*data" --type py -C 10
   - rg -n "class.*BaseTransform\)" --type py -C 20
   - rg -n "BaseTransform" --type py
5. Execute: Find BaseTransform ABC in transforms/base.py and
   ScaleTransform in transforms/scale.py as a sibling implementation
6. BM25 re-rank: BaseTransform.transform abstract definition and
   ScaleTransform.transform concrete implementation score highest
7. Deduplicate: Merge base class definition block, keep sibling separate

Output: Generate transform() body following the same pattern as ScaleTransform
but applying normalization, matching the expected return type and data format
```

**Example 3: Completing a Java method with cross-file enum and interface dependencies**

```
User: "Help me complete this handler method."

# Current file: src/handlers/OrderHandler.java
public class OrderHandler implements RequestHandler<OrderRequest> {
    @Override
    public Response handle(OrderRequest request) {
        OrderStatus status = // <-- completion needed

Approach:
1. Extract C_local: OrderHandler implements RequestHandler<OrderRequest>,
   needs to work with OrderStatus
2. Identify targets: OrderStatus (class-name), RequestHandler interface (class-name),
   OrderRequest (class-name)
3. Generate ripgrep commands:
   - rg -n "enum OrderStatus" --type java -C 15
   - rg -n "interface RequestHandler" --type java -C 10
   - rg -n "class OrderRequest" --type java -C 10
   - rg -n "OrderStatus\." --type java -C 3
   - rg -n "\.handle\(.*OrderRequest" --type java
4. Execute: Find OrderStatus enum with PENDING, PROCESSING, COMPLETED, FAILED
   and RequestHandler<T> with Response handle(T request) signature
5. BM25 re-rank: OrderStatus enum and existing handler implementations score highest
6. Deduplicate: Merge OrderStatus enum variants into one block

Output: Complete with status assignment using the correct enum values and
following the pattern established by other RequestHandler implementations
```

## Best Practices

**Do:**
- Generate diverse ripgrep commands targeting different dependency types (class, method, variable) rather than repeating similar queries on the same identifier
- Use wildcard patterns (`rg "def process.*batch"`) when the exact identifier name is uncertain — 23.5% of effective queries use fuzzy matching
- Always include `--type` flags to restrict results to the relevant language and `-n` for line numbers needed by the deduplication step
- Prioritize rare, specific identifiers over common ones when selecting search terms — `DataProcessor` is far more useful than `config` or `data`
- Merge adjacent snippets from the same file to reconstruct logical code flow before presenting as context

**Avoid:**
- Searching for high-frequency generic identifiers (`init`, `self`, `config`, `data`, `run`, `test`) as primary search terms — these produce massive noise with low signal
- Treating each ripgrep result as independent context — overlapping snippets from the same file waste your token budget and fragment the code's logical structure
- Using simple Jaccard similarity for ranking — it over-weights common tokens and under-weights the rare identifiers that actually matter for code completion
- Exceeding ~10 ripgrep commands per completion — diminishing returns set in quickly and additional queries mostly retrieve redundant context
- Setting the deduplication candidate pool above or below 50% of ranked results without good reason — this threshold is empirically optimal

## Error Handling

- **No ripgrep results:** If all commands return empty, the identifier may be misspelled or the dependency may be defined dynamically. Fall back to broader wildcard searches (e.g., `rg "class.*Proc" --type py`) or search for import statements that reference the target module.
- **Too many results (>100 matches):** The search term is too generic. Narrow by adding context: combine the identifier with its likely surrounding syntax (e.g., `rg "def calculate_loss\(self" --type py` instead of just `rg "calculate_loss"`).
- **Implicit dependencies not found:** When the code uses inheritance, mixins, or dynamic dispatch, lexical search may miss the relationship. Look for import statements and trace them, or search for the base class name explicitly.
- **Context budget exceeded:** If deduplicated blocks exceed 4096 tokens, drop the lowest-BM25-scoring blocks first. Never truncate a block mid-function — either include the full block or drop it entirely.
- **Wrong language matches:** If the repo is multi-language, always use `--type` flags. For monorepos, also use `--glob` to restrict to relevant directories.

## Limitations

- **Implicit dependencies:** When cross-file relationships rely on inheritance hierarchies, duck typing, or dependency injection without explicit naming, lexical search cannot discover them. If `C_local` contains no textual reference to the needed code, ripgrep has nothing to search for.
- **Dynamic code and metaprogramming:** Code generated at runtime, via decorators that transform signatures, or through metaclasses cannot be found by lexical search.
- **Highly generic identifiers:** In codebases where most classes follow identical naming conventions (e.g., dozens of `*Service`, `*Handler`, `*Controller` classes), lexical precision decreases and noise increases significantly.
- **Non-code context:** Documentation, configuration files, and comments may contain matching identifiers without being useful for completion. Use `--type` flags aggressively.
- **Very large result sets:** On repositories with 500K+ lines, even targeted ripgrep queries can return overwhelming results for common patterns. The 50% deduplication threshold and BM25 re-ranking mitigate this but do not eliminate it.

## Reference

**Paper:** [GrepRAG: An Empirical Study and Optimization of Grep-Like Retrieval for Code Completion](https://arxiv.org/abs/2601.23254v2) — Wang et al., 2026. Focus on Section 4 (the post-processing pipeline: BM25 re-ranking and structure-aware deduplication) and Section 3.3 (analysis of ripgrep command patterns showing the four retrieval categories and wildcard usage rates).