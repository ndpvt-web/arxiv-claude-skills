---
name: "diverge-diversity-enhanced-rag-open-ended"
description: "Diversity-enhanced RAG for open-ended queries with multiple valid answers. Uses reflection-guided generation and memory-augmented iterative refinement to produce diverse, high-quality responses instead of collapsing to a single dominant answer. Triggers: 'give me diverse perspectives on', 'explore different viewpoints', 'brainstorm multiple approaches to', 'what are the different ways to', 'open-ended search with diversity', 'DIVERGE-style RAG search'"
---

# DIVERGE: Diversity-Enhanced RAG for Open-Ended Information Seeking

This skill enables Claude to implement the DIVERGE agentic RAG framework, which generates multiple diverse yet high-quality responses to open-ended queries instead of collapsing to a single dominant answer. Standard RAG pipelines assume one correct answer per query; DIVERGE breaks this assumption through reflection-guided viewpoint generation, diversity-aware retrieval re-ranking, and a memory buffer that tracks explored perspectives across iterations. Use this when building search systems, recommendation engines, brainstorming tools, or any pipeline where query answers are legitimately plural.

## When to Use

- When the user asks to build a RAG pipeline that handles open-ended questions with multiple valid answers (e.g., "What are good date night restaurants?" or "How can I improve my sleep?")
- When implementing a search or Q&A system that should surface diverse perspectives rather than N near-duplicate results
- When the user wants to brainstorm or ideate by retrieving and synthesizing multiple distinct viewpoints from a corpus
- When building a recommendation system where variety matters (travel suggestions, product alternatives, career paths)
- When the user notices their RAG system keeps returning the same angle regardless of how many documents are retrieved
- When implementing fair and inclusive information access where minority viewpoints should not be suppressed by majority consensus

## Key Technique

**The core insight:** Standard RAG systems underutilize retrieved context diversity. Even when retrieval returns documents covering multiple perspectives, the LLM generation step collapses them into a single dominant response. Simply increasing retrieval diversity does not fix this — the bottleneck is in generation, not retrieval.

**DIVERGE solves this with three interlocking mechanisms.** First, *reflection-guided viewpoint generation*: after producing an initial response, the system extracts the underlying viewpoints (e.g., "budget-focused", "health-focused", "convenience-focused"), then explicitly reflects on which perspectives remain unexplored and generates a new viewpoint to target. Second, *viewpoint-aware diversity retrieval*: each iteration retrieves documents using a modified Maximal Marginal Relevance (MMR) score that penalizes similarity to previously retrieved contexts across all prior iterations, not just the current one. The scoring formula balances relevance to the viewpoint-conditioned query against cross-iteration and within-iteration diversity. Third, *memory-augmented iterative refinement*: a lightweight buffer stores `(query, documents, viewpoint, answer)` tuples from each iteration, giving the system full awareness of what has already been explored.

**The diversity-quality trade-off is measured** using semantic diversity (pairwise cosine distance between response embeddings), viewpoint diversity (unique atomic claims across responses), and quality (factual accuracy, evidence support, consistency, relevance). The unified score UQ^D is the harmonic mean of normalized quality and diversity, providing a single metric for the trade-off.

## Step-by-Step Workflow

1. **Classify the query as open-ended or closed.** Check whether the query has a single factual answer ("What year was Python created?") or multiple valid answers ("What's the best way to learn Python?"). Only apply DIVERGE to open-ended queries — closed queries should use standard RAG.

2. **Perform initial retrieval and generation.** Run a standard retrieval step against your corpus (vector search, BM25, or hybrid). Generate the first response using the retrieved documents. This becomes iteration t=1.

3. **Extract viewpoints from the initial response.** Prompt the LLM to decompose the response into its underlying viewpoints — the distinct perspectives, assumptions, or angles it covers. Output these as a structured list (e.g., `["cost-effectiveness", "ease of use", "community support"]`).

4. **Initialize the memory buffer.** Store the first iteration as a tuple: `{query, retrieved_docs, viewpoints, answer}`. This buffer persists across all iterations and is the system's "explored territory" map.

5. **Reflect to identify an unexplored viewpoint.** Prompt the LLM with the memory buffer contents and ask: "Given the query and the viewpoints already explored, identify a new meaningful perspective that has not been covered. Avoid restating existing viewpoints at a higher abstraction level." The reflection must produce a concrete, novel viewpoint.

6. **Construct a viewpoint-conditioned query.** Rewrite the original query to emphasize the new viewpoint. For example, if the original query is "How can I improve my sleep?" and the new viewpoint is "environmental factors", the conditioned query becomes "How can I improve my sleep by changing my environment?"

7. **Retrieve with diversity-aware re-ranking.** Retrieve candidate documents using the conditioned query, then re-rank using the modified MMR formula: `score(d) = alpha * Rel(d, q_t) - beta * max_similarity_to_memory - (1-alpha) * max_similarity_within_batch`. This ensures new documents are relevant to the target viewpoint while being distinct from all previously retrieved contexts.

8. **Generate a viewpoint-conditioned response.** Prompt the LLM to answer the original query from the specific viewpoint, grounded in the newly retrieved documents. Include a refinement instruction: "Ensure the response addresses the original question while focusing on the [viewpoint] perspective."

9. **Update the memory buffer and repeat.** Append the new iteration's tuple to memory. Return to step 5 and repeat for K iterations total (typically K=5-10 depending on the desired diversity breadth and compute budget).

10. **Aggregate and deduplicate the final response set.** Collect all K responses. Optionally compute pairwise semantic similarity and merge near-duplicates. Present the responses organized by viewpoint, or synthesize them into a single comprehensive answer that explicitly labels each perspective.

## Concrete Examples

**Example 1: Building a diverse travel recommendation RAG**

User: "I'm building a travel Q&A chatbot. When someone asks 'Where should I travel in Southeast Asia?', my RAG keeps returning the same Bangkok/Bali recommendations. Help me implement DIVERGE to get diverse suggestions."

Approach:
1. Classify as open-ended (multiple valid destinations, no single correct answer)
2. Initial retrieval + generation produces: "Bangkok for street food, Bali for beaches"
3. Extract viewpoints: `["popular tourist destinations", "food-focused travel", "beach vacations"]`
4. Store in memory buffer
5. Reflect: "Unexplored viewpoints: budget backpacking, cultural immersion, adventure/nature, off-the-beaten-path"
6. Conditioned query for iteration 2: "Where should I travel in Southeast Asia for cultural immersion experiences?"
7. Diversity-aware retrieval pulls documents about Luang Prabang, Yogyakarta, Hoi An — penalizing Bangkok/Bali docs
8. Generate: "For deep cultural immersion, consider Luang Prabang in Laos..."
9. Repeat for "adventure/nature" (retrieves Borneo, Komodo, northern Vietnam), "budget backpacking" (retrieves Cambodia, Myanmar), etc.

Output (5 iterations):
```
Viewpoint 1 — Popular highlights: Bangkok, Bali, Singapore
Viewpoint 2 — Cultural immersion: Luang Prabang, Yogyakarta, Hoi An
Viewpoint 3 — Adventure & nature: Borneo rainforest, Komodo, Ha Giang Loop
Viewpoint 4 — Budget backpacking: Cambodia, Myanmar, rural Philippines
Viewpoint 5 — Wellness & retreat: Ubud, Chiang Mai, Koh Phangan
```

**Example 2: Diverse code architecture suggestions**

User: "I want to use DIVERGE to give developers multiple architectural options when they ask 'How should I structure my backend API?'"

Approach:
1. Open-ended: multiple valid architectures exist
2. Initial generation: "Use a REST API with Express.js and MVC pattern"
3. Extract viewpoints: `["REST paradigm", "Node.js ecosystem", "MVC pattern"]`
4. Reflect: unexplored — GraphQL, microservices vs monolith, serverless, event-driven
5. Iteration 2 conditioned on "event-driven architecture": retrieves docs on CQRS, message queues
6. Iteration 3 conditioned on "serverless": retrieves docs on AWS Lambda, API Gateway patterns
7. Iteration 4 conditioned on "type-safe API design": retrieves docs on tRPC, gRPC

Output:
```
Option A — Classic REST + MVC: Express.js with controllers/services/models separation
Option B — Event-driven CQRS: Commands and queries separated, message bus for async ops
Option C — Serverless functions: AWS Lambda per endpoint, API Gateway routing
Option D — Type-safe RPC: tRPC or gRPC with generated clients, no REST overhead
Each option includes: when to use it, trade-offs, and example folder structure.
```

**Example 3: Implementing the diversity-aware retrieval re-ranker**

User: "Show me how to implement the DIVERGE MMR re-ranking step in Python."

```python
import numpy as np
from typing import List, Dict

def diverge_mmr_rerank(
    candidates: List[Dict],        # [{id, embedding, text}, ...]
    query_embedding: np.ndarray,    # embedding of viewpoint-conditioned query
    memory_embeddings: List[np.ndarray],  # embeddings from all prior iterations
    alpha: float = 0.6,            # relevance vs within-batch diversity
    beta: float = 0.3,             # cross-iteration diversity penalty
    top_k: int = 5
) -> List[Dict]:
    """Re-rank candidates using DIVERGE's diversity-aware MMR."""
    selected = []
    remaining = list(range(len(candidates)))

    for _ in range(min(top_k, len(candidates))):
        best_score, best_idx = -float('inf'), -1
        for i in remaining:
            emb = candidates[i]['embedding']

            # Relevance to viewpoint-conditioned query
            relevance = np.dot(emb, query_embedding) / (
                np.linalg.norm(emb) * np.linalg.norm(query_embedding)
            )

            # Cross-iteration diversity: penalize similarity to memory
            cross_div = 0.0
            if memory_embeddings:
                sims = [np.dot(emb, m) / (np.linalg.norm(emb) * np.linalg.norm(m))
                        for m in memory_embeddings]
                cross_div = max(sims)

            # Within-iteration diversity: penalize similarity to already selected
            within_div = 0.0
            if selected:
                sel_embs = [candidates[s]['embedding'] for s in selected]
                within_sims = [np.dot(emb, s) / (np.linalg.norm(emb) * np.linalg.norm(s))
                               for s in sel_embs]
                within_div = max(within_sims)

            score = alpha * relevance - beta * cross_div - (1 - alpha) * within_div
            if score > best_score:
                best_score, best_idx = score, i

        selected.append(best_idx)
        remaining.remove(best_idx)

    return [candidates[i] for i in selected]
```

## Best Practices

- **Do:** Extract viewpoints as concrete, distinct perspectives — not vague abstractions. "Budget-conscious approach" is good; "another way to think about it" is not.
- **Do:** Tune the alpha/beta parameters in the MMR re-ranker based on your domain. Higher beta (0.3-0.5) for creative tasks where diversity matters most; lower beta (0.1-0.2) for factual domains where relevance must dominate.
- **Do:** Include a refinement check in each iteration that validates the response still answers the original query, not just the conditioned sub-query.
- **Do:** Set K (iteration count) based on the breadth of the topic. Simple questions may need K=3-4; broad topics like "how to start a business" may warrant K=8-10.
- **Avoid:** Applying DIVERGE to factual, single-answer queries. "What is the capital of France?" does not benefit from diverse viewpoints — it adds hallucination risk.
- **Avoid:** Letting viewpoint reflection produce increasingly abstract or meta perspectives. If iteration 5 produces "a philosophical lens" when the query is about restaurant recommendations, the reflection prompt needs stronger grounding constraints.

## Error Handling

- **Viewpoint saturation:** If reflection cannot produce a genuinely new viewpoint, stop early rather than force repetitive iterations. Detect this by checking semantic similarity between the new viewpoint and all prior ones — if cosine similarity > 0.85, terminate.
- **Quality degradation in later iterations:** Monitor response quality scores across iterations. If quality drops below a threshold (e.g., the response becomes speculative or unsupported), discard that iteration and retry with a different viewpoint.
- **Retrieval returns no diverse documents:** If the corpus is narrow, the diversity-aware retrieval may struggle. Fall back to standard MMR or increase the candidate pool size before re-ranking.
- **Memory buffer grows too large:** For very high K values, summarize older memory entries rather than passing raw tuples. Keep the most recent 3-4 iterations in full detail and compress earlier ones to viewpoint labels only.

## Limitations

- DIVERGE adds latency proportional to K iterations — each iteration requires a retrieval step and an LLM generation. For real-time applications, consider running iterations in parallel where the corpus allows independent retrieval.
- The framework assumes the underlying corpus actually contains diverse perspectives. If your document collection is homogeneous (e.g., all from one source), DIVERGE cannot manufacture diversity that isn't there.
- Viewpoint extraction quality depends on the LLM's ability to identify implicit perspectives. For highly technical or niche domains, you may need domain-specific viewpoint taxonomies rather than open-ended extraction.
- The approach is designed for open-ended information seeking. Applying it to structured data queries, code generation with a single correct output, or mathematical problems will not yield meaningful benefits.

## Reference

[DIVERGE: Diversity-Enhanced RAG for Open-Ended Information Seeking](https://arxiv.org/abs/2602.00238v1) — Hu, Tandon, Arora (2026). Focus on Section 3 (the DIVERGE framework architecture), Section 4.2 (diversity-aware MMR formula), and Section 5 (the UQ^D unified diversity-quality metric).