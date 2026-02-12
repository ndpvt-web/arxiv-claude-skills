---
name: "diverge-diversity-enhanced-rag-open-ended"
description: "Generate diverse, multi-perspective answers for open-ended questions using the DIVERGE agentic RAG framework — reflection-guided viewpoint expansion with memory-augmented iterative refinement. Use when: 'give me multiple perspectives on...', 'brainstorm diverse approaches to...', 'what are different viewpoints on...', 'explore alternative solutions for...', 'generate varied recommendations for...', 'help me think about this from different angles'."
---

# DIVERGE: Diversity-Enhanced RAG for Open-Ended Information Seeking

This skill enables Claude to produce genuinely diverse, high-quality responses to open-ended questions by implementing the DIVERGE framework. Standard RAG and LLM generation systematically collapse toward a dominant perspective — even when retrieved documents contain diverse viewpoints, generations remain homogeneous. DIVERGE breaks this pattern through reflection-guided viewpoint expansion and memory-augmented iterative refinement, producing a set of distinct answers that each explore a different valid perspective while maintaining factual quality.

## When to Use

- When the user asks an open-ended question with multiple plausible answers (e.g., "What are the best approaches to microservice architecture?")
- When brainstorming or ideation requires genuine variety, not surface-level rephrasing (e.g., "Generate diverse startup ideas for the education sector")
- When the user explicitly requests multiple perspectives or alternative viewpoints on a topic
- When building a recommendation system, chatbot, or search feature that must surface diverse results rather than repetitive ones
- When implementing a RAG pipeline and the user wants to avoid the single-answer collapse problem
- When designing a content generation system that needs to produce varied outputs across runs
- When fairness or inclusivity requires representing multiple valid viewpoints rather than defaulting to the most common one

## Key Technique

**The core problem:** Standard RAG pipelines retrieve documents, feed them to an LLM, and generate a single response. Even when you diversify retrieval (e.g., MMR re-ranking, query rewriting), the LLM still collapses to one dominant answer. The paper demonstrates that retrieval diversity alone does not produce generation diversity — the generation step itself must be explicitly diversity-aware.

**DIVERGE's solution** uses three interlocking mechanisms. First, *reflection-guided generation*: after producing an initial response, the system reflects on what viewpoints have already been covered and explicitly identifies a new, underexplored perspective. This new viewpoint becomes a conditioning signal for the next generation. Second, *memory-augmented iteration*: a memory store accumulates (query, documents, viewpoint, answer) tuples across iterations, so each new cycle is aware of everything produced so far and avoids repetition at both the retrieval and generation levels. Third, *iteration-aware diverse retrieval*: a modified MMR scoring function penalizes documents similar to those already in memory, ensuring fresh evidence for each new viewpoint.

**The diversity-quality trade-off** is measured via two complementary metrics: *Semantic Diversity* (average pairwise cosine distance between response embeddings) and *Viewpoint Diversity* (ratio of unique atomic claims to total claims across all responses). Quality is evaluated on factual accuracy, evidence support, internal consistency, and relevance. The unified score is a harmonic mean of normalized diversity and quality, preventing either from being sacrificed. In experiments, DIVERGE achieved 2.7x improvement in semantic diversity with only ~0.04 quality degradation compared to vanilla RAG.

## Step-by-Step Workflow

1. **Classify the query as open-ended.** Determine whether the user's question genuinely has multiple valid answers. If the question has a single factual answer (e.g., "What is the capital of France?"), use standard RAG instead. Open-ended indicators: asks for recommendations, perspectives, approaches, ideas, opinions, or alternatives.

2. **Generate the initial response with standard RAG.** Retrieve relevant documents for the original query, then generate a first-pass answer. This serves as the baseline and seeds the memory.

3. **Extract covered viewpoints from the initial response.** Decompose the initial answer into atomic claims or perspectives. For example, if the question is about remote work policies, the initial answer might cover "productivity benefits" and "communication challenges." List these explicitly.

4. **Reflect to identify an uncovered viewpoint.** Given the list of already-covered perspectives, reason about what meaningful angle has not yet been explored. The new viewpoint must be (a) relevant to the original query, (b) substantively different from existing viewpoints, and (c) neither too broad nor too narrow. Frame it as a concise statement: "Employee mental health and isolation effects."

5. **Generate a viewpoint-conditioned query.** Rewrite the original query to specifically target the new viewpoint. This query will drive retrieval for the next iteration. Example: original "What are the effects of remote work?" becomes "How does remote work affect employee mental health and social isolation?"

6. **Retrieve diverse documents with memory-aware scoring.** Use the conditioned query for retrieval, but penalize documents that are similar to those already stored in memory from prior iterations. Score each candidate document as: `score = alpha * relevance(doc, query) - beta * max_similarity(doc, memory_docs) - (1-alpha) * max_similarity(doc, current_batch)`.

7. **Generate a viewpoint-conditioned response.** Produce a new answer explicitly conditioned on both the original query and the target viewpoint. The prompt must instruct the LLM to address the specific perspective while staying grounded in the retrieved evidence and aligned with the original question's intent.

8. **Store the iteration in memory.** Append the tuple (conditioned_query, retrieved_docs, viewpoint, generated_answer) to the memory store. This ensures subsequent iterations have full context of what has been covered.

9. **Repeat steps 4-8 for K iterations.** Each cycle identifies a new viewpoint, retrieves fresh evidence, and generates a perspective-specific response. The number of iterations K depends on the breadth of the question — typically 3-5 for practical use, up to 10 for comprehensive exploration.

10. **Compile and present the diverse response set.** Organize the K responses into a structured output. Optionally add a synthesis section that acknowledges the range of perspectives without collapsing them into a single conclusion.

## Concrete Examples

**Example 1: Multi-perspective technology recommendation**

```
User: What database should I use for my new web application?

Approach:
1. Classify as open-ended — multiple valid answers depending on context
2. Initial RAG response covers PostgreSQL (the dominant recommendation)
3. Extract viewpoints: ["relational/SQL with strong ecosystem", "ACID compliance"]
4. Reflect → uncovered viewpoint: "document-oriented flexibility for evolving schemas"
5. Conditioned query: "When is MongoDB better than PostgreSQL for web apps?"
6. Retrieve and generate MongoDB-focused response
7. Reflect → uncovered viewpoint: "real-time data and caching layer"
8. Retrieve and generate Redis/in-memory store response
9. Reflect → uncovered viewpoint: "serverless and zero-ops for small teams"
10. Retrieve and generate managed/serverless DB response (PlanetScale, Supabase)

Output:
## Perspective 1: PostgreSQL — The Reliable Workhorse
PostgreSQL handles most web app needs with strong SQL support, JSON columns
for flexibility, and a mature ecosystem. Best when your schema is well-defined
and you need transactions...

## Perspective 2: MongoDB — Schema Flexibility First
If your data model is evolving rapidly or naturally document-shaped (user
profiles, product catalogs with variable attributes), MongoDB avoids costly
migrations...

## Perspective 3: Redis as Primary + Persistence Layer
For applications where read latency is critical (leaderboards, session stores,
real-time features), Redis with persistence can serve as a primary store...

## Perspective 4: Managed Serverless (PlanetScale, Supabase)
For small teams that cannot afford database operations overhead, serverless
databases eliminate connection management, scaling, and backup concerns...
```

**Example 2: Open-ended brainstorming**

```
User: Brainstorm ways to reduce employee turnover at a mid-size tech company.

Approach:
1. Open-ended — many valid strategies across different dimensions
2. Initial response: competitive compensation and equity packages
3. Viewpoints covered: ["compensation", "equity/stock options"]
4. Iteration 2 viewpoint: "career growth and internal mobility"
5. Iteration 3 viewpoint: "manager quality and team dynamics"
6. Iteration 4 viewpoint: "remote/hybrid flexibility and autonomy"
7. Iteration 5 viewpoint: "mission alignment and meaningful work"

Output:
## 1. Compensation Restructuring
Benchmark against current market rates quarterly, not annually. Implement
transparent pay bands. Retention bonuses at 2-year and 4-year marks...

## 2. Internal Mobility Program
Create a formal internal transfer process with a 2-week exploration period.
Employees who see lateral movement options are 30% less likely to leave...

## 3. Manager Training Pipeline
Exit interviews consistently cite management as the #1 departure reason.
Invest in manager training focused on 1:1 quality, feedback cadence...

## 4. Flexible Work Architecture
Offer genuine flexibility, not performative hybrid. Let teams decide their
own in-office cadence. Provide home office stipends...

## 5. Mission Connection
Regularly connect individual work to company impact. Engineers who understand
how their code affects end users report higher job satisfaction...
```

**Example 3: Implementing the DIVERGE pattern in code**

```
User: I'm building a RAG pipeline in Python. Help me add diversity-aware
generation using the DIVERGE approach.

Approach:
1. This is a code implementation request — apply the framework directly
2. Implement the memory store, reflection step, and iteration loop

Output (key code structure):

class DivergeRAG:
    def __init__(self, llm, retriever, k_perspectives=5):
        self.llm = llm
        self.retriever = retriever
        self.k = k_perspectives
        self.memory = []  # List of (query, docs, viewpoint, answer) tuples

    def generate_diverse(self, query: str) -> list[dict]:
        # Step 1: Initial RAG
        docs = self.retriever.retrieve(query)
        initial_answer = self.llm.generate(query=query, context=docs)
        viewpoints = self._extract_viewpoints(initial_answer)
        self.memory.append({
            "query": query, "docs": docs,
            "viewpoint": "initial", "answer": initial_answer
        })
        results = [{"viewpoint": "initial", "answer": initial_answer}]

        # Steps 2-K: Iterative viewpoint expansion
        for i in range(1, self.k):
            new_vp = self._reflect_new_viewpoint(query, viewpoints)
            viewpoints.append(new_vp)
            conditioned_query = self._condition_query(query, new_vp)
            docs = self._diverse_retrieve(conditioned_query)
            answer = self.llm.generate(
                query=query, viewpoint=new_vp, context=docs
            )
            self.memory.append({
                "query": conditioned_query, "docs": docs,
                "viewpoint": new_vp, "answer": answer
            })
            results.append({"viewpoint": new_vp, "answer": answer})
        return results

    def _reflect_new_viewpoint(self, query, existing_viewpoints):
        prompt = f"""Given the question: {query}
The following perspectives have already been covered:
{chr(10).join(f'- {v}' for v in existing_viewpoints)}

Identify ONE new, meaningfully different perspective that has not been
explored. It must be relevant to the original question, substantively
distinct, and neither too broad nor too narrow. Return only the
viewpoint as a concise phrase."""
        return self.llm.generate(prompt=prompt)

    def _diverse_retrieve(self, query, alpha=0.6, beta=0.3):
        candidates = self.retriever.retrieve(query, top_k=20)
        memory_docs = [d for m in self.memory for d in m["docs"]]
        # Iteration-aware MMR: penalize similarity to memory
        scored = []
        selected = []
        for doc in candidates:
            relevance = doc.score
            memory_penalty = max(
                cosine_sim(doc.embedding, md.embedding)
                for md in memory_docs
            ) if memory_docs else 0
            batch_penalty = max(
                (cosine_sim(doc.embedding, s.embedding)
                 for s in selected), default=0
            )
            score = alpha * relevance - beta * memory_penalty \
                    - (1 - alpha) * batch_penalty
            scored.append((doc, score))
        scored.sort(key=lambda x: x[1], reverse=True)
        return [doc for doc, _ in scored[:5]]
```

## Best Practices

- **Do:** Always extract explicit viewpoints after each generation and store them in memory. The reflection step only works when it has a concrete list of "already covered" perspectives to reason against.
- **Do:** Condition both retrieval and generation on the target viewpoint. Conditioning only one of the two still produces homogeneous outputs — the paper shows both steps must be diversity-aware.
- **Do:** Use the harmonic mean of diversity and quality as your optimization target. Maximizing diversity alone degrades quality (as shown by Verbalized Sampling baselines dropping quality significantly).
- **Do:** Set K (number of perspectives) proportional to the question's breadth. A narrow question like "good Python web frameworks" needs 3-4 perspectives; a broad question like "how to improve education" supports 8-10.
- **Avoid:** Generating all perspectives in a single prompt ("give me 10 different viewpoints"). This produces superficial variation. The iterative approach with retrieval grounds each perspective in distinct evidence.
- **Avoid:** Viewpoints that are too broad ("consider the economic angle") or too narrow ("consider the impact on left-handed users in rural Finland"). The reflection prompt must explicitly instruct for medium-granularity perspectives.

## Error Handling

- **Viewpoint drift from original query (40% of failures):** If a generated response diverges from the user's core intent, add a refinement step that checks alignment: "Does this response address the original question?" Re-generate with stronger query conditioning if not.
- **Overly generic responses (30% of failures):** When a viewpoint is too broad, the generation becomes vague. Detect this by checking claim density (few atomic claims per paragraph). Prompt for more specific sub-viewpoints.
- **Repetitive viewpoints across iterations:** If the reflection step produces viewpoints semantically similar to existing ones (cosine similarity > 0.85), reject and re-prompt with an explicit instruction to explore a different dimension of the question.
- **Quality degradation at high K:** Beyond 5-7 iterations, quality tends to drop as viewpoints become increasingly peripheral. Monitor quality scores and stop early if the last 2 iterations score below a threshold.
- **Empty or low-quality retrieval for niche viewpoints:** If the conditioned query returns few relevant documents, fall back to the original query with the viewpoint as a soft filter rather than a hard constraint.

## Limitations

- **Single-answer questions:** DIVERGE adds overhead with no benefit for factual queries with one correct answer. Always classify the query type first.
- **Latency:** Each perspective requires a full retrieval + generation cycle. For K=5, expect ~5x the latency of standard RAG. Not suitable for real-time autocomplete or sub-second responses.
- **Viewpoint quality depends on LLM capability:** The reflection step requires strong reasoning to identify genuinely novel perspectives. Weaker models may produce superficial or repetitive viewpoints.
- **No automatic K selection:** The framework generates exactly K perspectives regardless of whether the question warrants that many. Implementing quality-based early stopping is recommended but not built into the core algorithm.
- **Domain knowledge gaps:** For highly specialized domains, the LLM may lack the background to identify meaningful alternative perspectives, leading to generic or incorrect viewpoints.

## Reference

**Paper:** [DIVERGE: Diversity-Enhanced RAG for Open-Ended Information Seeking](https://arxiv.org/abs/2602.00238v1) — Tianyi Hu, Niket Tandon, Akhil Arora (2026). Look for Algorithm 1 (the full DIVERGE loop), the iteration-aware MMR scoring function, and the Viewpoint Diversity metric definition. The ablation study in Section 5 confirms that both search grounding and result refinement are necessary components.