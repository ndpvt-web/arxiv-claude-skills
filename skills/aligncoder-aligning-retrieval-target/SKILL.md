---
name: "aligncoder-aligning-retrieval-target"
description: |
  Repository-level code completion using AlignCoder's query enhancement and aligned retrieval technique.
  Generates candidate completions to build an enhanced query that bridges the semantic gap between
  unfinished code and the target completion, then retrieves precisely relevant cross-file context.

  Use this skill when the user says:
  - "Complete this code using context from other files in the repo"
  - "Help me finish this function using the rest of the codebase"
  - "Use cross-file context to complete this code"
  - "I need repo-aware code completion for this partial implementation"
  - "Retrieve relevant code from the repository to help complete this"
  - "What would go here based on how similar code works in this project?"
---

# AlignCoder: Aligned Retrieval for Repository-Level Code Completion

This skill enables Claude to perform high-quality repository-level code completion by applying AlignCoder's core insight: when completing code, the initial unfinished code (the "query") is semantically distant from the target completion, making naive retrieval of cross-file context ineffective. AlignCoder solves this by first generating multiple candidate completions to construct an *enhanced query* that contains key tokens related to the target, then using that enriched signal to retrieve precisely relevant code snippets from the repository. This two-phase retrieve-then-refine approach produces dramatically better completions than single-pass retrieval.

## When to Use

- When the user asks to complete a function, method, or code block that depends on types, APIs, or patterns defined in other files in the repository
- When completing code that calls methods from imported classes whose signatures are elsewhere in the codebase
- When the user has partial code and wants the completion to be consistent with the repository's existing conventions and APIs
- When naive autocompletion would fail because the completion requires knowledge of cross-file dependencies (e.g., method signatures, class hierarchies, utility functions)
- When the user asks "what should go here?" and the answer depends on how similar code works elsewhere in the project
- When completing API calls, configuration patterns, or domain-specific idioms that are established elsewhere in the codebase

## Key Technique

**The Alignment Problem.** Traditional RAG-based code completion retrieves cross-file snippets using the unfinished code as the search query. But the unfinished code is semantically closer to the *prefix* of the target, not the *completion itself*. For example, if you need to complete `generator.`, the unfinished code mentions `generator` but not `get_accept_token` — the actual method to call. This semantic gap causes retrieval to return snippets similar to the context rather than snippets relevant to the completion.

**Query Enhancement via Candidate Completions.** AlignCoder addresses this by generating multiple candidate completions (typically 4) using a coarse initial retrieval pass. These candidates — even if individually imperfect — collectively contain key tokens that overlap with the true target. By concatenating these candidates with the original unfinished code, AlignCoder constructs an *enhanced query* that is much closer to the target in embedding space. This enhanced query is then used for a second, fine-grained retrieval pass that fetches genuinely relevant cross-file context (e.g., class definitions, method signatures, related implementations).

**Two-Type Codebase Construction.** The repository is indexed into two snippet types: (1) *base code snippets* created by splitting files at blank lines and aggregating into fixed-length chunks, and (2) *dependency snippets* extracted by parsing import statements and collecting the signatures (not full implementations) of imported functions, classes, and methods. This dual representation ensures retrieval can surface both usage patterns and API definitions.

## Step-by-Step Workflow

1. **Parse the unfinished code and identify the completion point.** Determine exactly where the cursor is, what the surrounding context looks like, and what imports/dependencies are declared at the top of the current file.

2. **Extract dependency information from imports.** Parse the import statements in the current file to identify which modules, classes, and functions are referenced. Use this to prioritize which files in the repository are likely relevant.

3. **Build the retrieval codebase from the repository.** Index the repo into two snippet pools:
   - *Base snippets*: Split each file at blank lines, then aggregate consecutive blocks into chunks of roughly 20 lines. These capture usage patterns and idioms.
   - *Dependency snippets*: For each imported symbol, extract its signature (function signature, class definition with method signatures). These capture API surfaces without implementation noise.

4. **Perform coarse-grained retrieval using lexical matching (BM25).** Use the unfinished code as a query to retrieve the top-k most lexically similar snippets from the combined snippet pool. This initial pass provides rough context.

5. **Generate multiple candidate completions (n=4).** Using the coarse-retrieved context concatenated with the unfinished code, generate 4 diverse candidate completions. Use higher temperature (0.8) and nucleus sampling (top-p 0.95) to ensure diversity — the goal is to cover different plausible completions so that collectively they contain key tokens from the true target.

6. **Construct the enhanced query.** Concatenate the original unfinished code with all 4 candidate completions. This enhanced query now contains tokens that are semantically aligned with the target completion, bridging the gap that made initial retrieval imprecise.

7. **Perform fine-grained retrieval using the enhanced query.** Search the snippet pool again using the enhanced query with semantic similarity (embedding-based). The candidate completion tokens steer retrieval toward snippets that define the APIs, patterns, or types actually needed for the completion.

8. **Assemble the final prompt.** Combine the fine-grained retrieved snippets (as cross-file context) with the original unfinished code. Place retrieved snippets before the unfinished code so the model sees the relevant definitions first.

9. **Generate the final completion.** Produce the target completion from this enriched prompt. The model now has both the unfinished code and precisely relevant cross-file context.

10. **Validate the completion against repository conventions.** Check that the generated code uses correct method names, argument orders, and types as defined in the retrieved snippets. Fix any inconsistencies.

## Concrete Examples

**Example 1: Completing a method call on an imported class**

```
User: Complete this code — I know the ExLlamaGenerator class is defined
elsewhere in the repo but I don't remember the exact method names.

# Current file: inference.py
from model.generator import ExLlamaGenerator

def run_inference(gen: ExLlamaGenerator, input_ids):
    gen.settings.temperature = 0.7
    token = gen.  # <-- complete here
```

Approach:
1. Parse imports: `ExLlamaGenerator` comes from `model/generator.py`
2. Extract dependency snippet: read `model/generator.py`, extract class
   signature and method signatures for `ExLlamaGenerator`
3. Coarse retrieval (BM25): find snippets in repo mentioning `ExLlamaGenerator`
4. Generate 4 candidates: `gen.get_accept_token(input_ids)`,
   `gen.generate_token(input_ids)`, `gen.sample_token(input_ids)`,
   `gen.get_next_token(input_ids)`
5. Enhanced query now contains tokens like `get_accept_token`, `generate_token`
6. Fine-grained retrieval: finds the actual class definition showing
   `def get_accept_token(self, input_ids) -> torch.Tensor`
7. Final completion: `gen.get_accept_token(input_ids)`

Output:
```python
token = gen.get_accept_token(input_ids)
```

**Example 2: Completing a configuration pattern used elsewhere in the project**

```
User: I'm adding a new endpoint to this FastAPI app. Complete the route
handler based on how other endpoints are structured in this repo.

# Current file: api/routes/users.py
from api.deps import get_db, get_current_user
from api.schemas.user import UserResponse

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    # <-- complete the rest of this function signature and body
```

Approach:
1. Parse imports: `get_db`, `get_current_user` from `api/deps.py`,
   `UserResponse` from `api/schemas/user.py`
2. Extract dependency snippets from `api/deps.py` (get function signatures)
   and `api/schemas/user.py` (get UserResponse fields)
3. Coarse retrieval: find other route handlers in `api/routes/` directory
4. Generate 4 candidate completions — they mention patterns like
   `db: Session = Depends(get_db)`, `current_user = Depends(get_current_user)`,
   `raise HTTPException(status_code=404)`
5. Enhanced query surfaces the `Depends()` injection pattern used across the repo
6. Fine-grained retrieval: pulls in another handler like `get_item()` that shows
   the exact dependency injection pattern and error handling convention
7. Complete using the project's established pattern

Output:
```python
@router.get("/users/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: Session = Depends(get_db),
    current_user: User = Depends(get_current_user),
):
    user = db.query(UserModel).filter(UserModel.id == user_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

**Example 3: Completing a test that follows the repo's testing patterns**

```
User: Add a test for the new `calculate_discount` function. Follow the
patterns used in the existing test files.

# Current file: tests/test_pricing.py
from pricing.discounts import calculate_discount
from tests.fixtures import sample_order, premium_customer

class TestCalculateDiscount:
    def test_premium_customer_discount(self):
        # <-- complete here
```

Approach:
1. Parse imports to find `calculate_discount` signature and `sample_order`/
   `premium_customer` fixture definitions
2. Extract dependency snippets: signature of `calculate_discount`, fields of
   fixture objects
3. Coarse retrieval: find other test classes in `tests/` directory
4. Generate 4 candidates that try different assertion styles, fixture usage
5. Enhanced query contains tokens like `assert`, fixture field names,
   expected discount values — aligning with the test pattern
6. Fine-grained retrieval: pulls in `TestCalculateShipping` which shows
   exact assertion pattern: arrange fixtures, call function, assert result
7. Complete following the established pattern

Output:
```python
def test_premium_customer_discount(self):
    order = sample_order(subtotal=100.00)
    customer = premium_customer()
    discount = calculate_discount(order, customer)
    assert discount.percentage == 15.0
    assert discount.amount == 15.00
```

## Best Practices

- **Do:** Generate diverse candidates by using moderate temperature (0.7-0.8). The point is coverage of possible tokens, not individual accuracy.
- **Do:** Prioritize dependency snippets (signatures and type definitions) over base snippets when the completion involves calling external APIs. Signatures are more informative per token than full implementations.
- **Do:** Limit candidate count to 4. Research shows diminishing returns beyond this — additional candidates introduce noise tokens that degrade retrieval quality.
- **Do:** Place retrieved cross-file context *before* the unfinished code in the prompt, so the model processes definitions before the completion point.
- **Avoid:** Using only the raw unfinished code for retrieval. The whole point of the technique is that unfinished code is a poor query for finding what the *completion* needs.
- **Avoid:** Retrieving excessively long snippets. Keep snippets to ~20 lines. Dense, focused context (signatures, short usage examples) outperforms large code dumps.
- **Avoid:** Treating all candidate completions as correct. They are query enhancement material, not answers. The final completion should be generated fresh with proper retrieved context.

## Error Handling

- **No relevant cross-file context found:** Fall back to completing with only the current file's context. Some completions are self-contained and don't need cross-file information.
- **All candidate completions are identical:** Increase temperature or reduce top-p to encourage diversity. Identical candidates provide no query enhancement benefit.
- **Candidate completions are wildly wrong:** This usually means the coarse retrieval failed. Widen the initial retrieval window (increase k) or fall back to import-based dependency extraction only.
- **Retrieved snippets contradict each other:** Prefer dependency snippets (authoritative definitions) over base snippets (usage examples that may be outdated). When in doubt, trace the import chain to the source of truth.
- **Completion point is ambiguous:** Ask the user to clarify exactly where the cursor is and what scope the completion should cover (single expression, full function body, etc.).

## Limitations

- **Works best for completions that depend on cross-file context.** For purely local completions (e.g., arithmetic, string formatting), the two-phase retrieval adds overhead without benefit.
- **Candidate quality matters.** If the initial coarse retrieval returns completely irrelevant context, the candidates will be poor and the enhanced query won't contain useful tokens. The technique assumes at least some signal in the first retrieval pass.
- **Tested primarily on Python and Java.** The approach is language-agnostic in principle, but the evaluation evidence is strongest for these two languages.
- **Computational cost.** Generating 4 candidate completions plus two retrieval passes is more expensive than a single completion. Use this workflow for cases where cross-file accuracy matters, not for trivial completions.
- **Repository must be accessible.** This technique requires reading and indexing the repository's files. It doesn't apply when the user provides only an isolated code snippet with no repository context.

## Reference

[AlignCoder: Aligning Retrieval with Target Intent for Repository-Level Code Completion](https://arxiv.org/abs/2601.19697v1) — Jiang et al., ASE 2025. Key sections: Section 3 (Approach) for the query enhancement mechanism and AlignRetriever training; Section 4.4 for ablation studies showing the contribution of each component; Figure 2 for the end-to-end pipeline diagram.