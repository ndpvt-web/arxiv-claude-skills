---
name: "sparc-rag-adaptive-sequential-parallel-scaling"
description: "Build multi-agent RAG pipelines that adaptively scale retrieval depth (sequential iterations) and breadth (parallel branches) with unified context management. Use when: 'build a multi-hop RAG system', 'implement adaptive retrieval pipeline', 'create a multi-agent question answering system', 'build RAG with iterative refinement', 'implement parallel retrieval with context management', 'scale RAG inference with sub-query decomposition'."
---

# SPARC-RAG: Adaptive Sequential-Parallel Scaling for Retrieval-Augmented Generation

This skill enables Claude to build multi-agent RAG systems that coordinate **sequential depth** (iterative retrieval rounds) and **parallel width** (concurrent sub-query branches) under a unified context management mechanism. Based on the SPARC-RAG framework, the technique uses five specialized agents -- Query Rewriter, Retriever, Answer Generator, Answer Evaluator, and Context Manager -- that share a global memory state to avoid context contamination and scaling inefficiency. The result is a RAG pipeline that retrieves more relevant evidence for multi-hop questions while using fewer tokens than naive scaling approaches.

## When to Use

- When the user needs to build a RAG system that handles **multi-hop questions** requiring evidence from multiple documents (e.g., "Who is the spouse of the director of Film X?")
- When a single retrieval pass returns incomplete or irrelevant evidence and the user wants **iterative refinement**
- When the user asks to implement **parallel retrieval branches** that explore different aspects of a complex query simultaneously
- When an existing RAG pipeline suffers from **context contamination** -- accumulated irrelevant passages degrading answer quality over iterations
- When the user wants an **adaptive exit strategy** that stops retrieving once the answer is sufficiently grounded, rather than running a fixed number of rounds
- When building agentic RAG pipelines where different components (query rewriting, retrieval, evaluation) need to **coordinate through shared state**

## Key Technique

SPARC-RAG models the RAG reasoning process as three composed operators: **I** (single-path inference), **P** (parallel expansion), and **S** (sequential refinement), combined as `Pipeline = S(P(I))`. At each sequential step `t`, a Query Rewriter generates `W` parallel sub-queries from the original question plus current memory state. Each branch independently retrieves documents, updates a branch-local memory, generates a candidate answer, and evaluates that answer. A Context Manager then selects the best branch and merges complementary evidence from other branches while filtering noise. The loop continues until the Answer Evaluator signals STOP or a maximum depth `D_max` is reached.

The critical innovation is **unified context management**. Rather than concatenating all retrieved passages (which causes contamination), the system maintains a memory object `m(t)` that is query-aware -- it retains only evidence relevant to the current question. Branch-local memories `m'_k` are created independently, preventing cross-branch contamination, and the post-branch merge operation consolidates only useful information. This prevents the diminishing returns that plague naive multi-round RAG.

The exit decision uses **asymmetric error weighting**: a wrong stop (accepting an incorrect answer) is far more costly than a wrong continue (doing one extra retrieval round). The Answer Evaluator is trained with weighted DPO where early-stop failures receive penalty weight `lambda=2`, biasing the system toward continuing when uncertain. The Query Rewriter is supervised using paragraph-level retrieval recall -- rewrites that retrieve more gold evidence are preferred -- which implicitly encourages diverse yet focused sub-queries.

## Step-by-Step Workflow

1. **Define the five agent interfaces.** Create distinct modules for: `QueryRewriter` (decomposes complex queries into targeted sub-queries), `Retriever` (fetches documents via hybrid sparse+dense search), `AnswerGenerator` (produces answers conditioned on query + memory), `AnswerEvaluator` (emits CONTINUE/STOP signals based on correctness and grounding), and `ContextManager` (selects best branch, merges memories, filters noise).

2. **Initialize global memory as empty.** Set `memory = {}` and `t = 1`. Configure `D_max` (maximum sequential depth, typically 6-8) and `W` (parallel branch width, typically 2). These are the scaling knobs.

3. **Generate parallel sub-queries.** At each step `t`, call `QueryRewriter(original_query, current_memory, W)` to produce `W` rewritten queries. Each sub-query should target a different information need -- for "Who directed the film starring Actor X born in City Y?", one branch might target "films starring Actor X" while another targets "Actor X birthplace City Y".

4. **Execute parallel retrieval branches independently.** For each sub-query `k` in `1..W`, run in parallel:
   - `docs_k = Retriever(sub_query_k, corpus)`
   - `branch_memory_k = ContextManager.update(current_memory, docs_k)` -- query-aware merge that retains only relevant evidence
   - `answer_k = AnswerGenerator(original_query, branch_memory_k)`
   - `decision_k = AnswerEvaluator(answer_k, branch_memory_k)` -- returns STOP or CONTINUE

5. **Select the best branch and merge context.** Call `ContextManager.select_best(branches)` to pick the branch `k*` with the highest-quality answer. Then call `ContextManager.merge(branch_memory_k*, other_branch_memories)` to integrate complementary evidence from non-winning branches while discarding contradictory or irrelevant passages.

6. **Check exit conditions.** If `decision_k* == STOP`, return `answer_k*` as the final answer. If `t >= D_max`, return the best available answer. Otherwise, set `t = t + 1` and loop back to step 3.

7. **Implement query-aware memory updates.** The memory update function must attend to the original question and filter retrieved passages. Use a relevance scoring mechanism (e.g., cross-encoder reranking or LLM-based relevance judgment) to keep only passages that contribute to answering the question. Discard passages below a relevance threshold.

8. **Implement the asymmetric evaluator logic.** The Answer Evaluator should check two conditions: (a) does the answer appear correct given the evidence? (b) is the evidence sufficient to ground the answer? Bias toward CONTINUE when uncertain -- it is cheaper to do one extra retrieval round than to return a wrong answer.

9. **Add cost tracking and adaptive scaling.** Track token usage per round. If early rounds yield diminishing retrieval quality (measured by relevance scores), reduce `W` for subsequent rounds. If the question is simple (single-hop), the evaluator should signal STOP after round 1.

10. **Test with multi-hop queries.** Validate the pipeline on questions requiring 2-4 reasoning hops. Verify that: (a) sub-queries target different evidence needs, (b) memory does not accumulate irrelevant passages over rounds, (c) the system exits early on easy questions and persists on hard ones.

## Concrete Examples

**Example 1: Multi-hop question answering pipeline**

User: "Build a RAG pipeline that can answer multi-hop questions like 'What university did the director of Inception attend?'"

Approach:
1. Parse the question to identify it requires two hops: (1) find the director of Inception, (2) find their university.
2. Initialize `memory = {}`, `W = 2`, `D_max = 6`.
3. Round 1 -- QueryRewriter generates:
   - Branch 1: "Who directed the movie Inception?"
   - Branch 2: "Inception film director education background"
4. Each branch retrieves independently, generates answers, evaluates.
5. Branch 1 finds "Christopher Nolan directed Inception" with CONTINUE signal (partial answer).
6. ContextManager merges: memory now contains the director identity.
7. Round 2 -- QueryRewriter uses updated memory:
   - Branch 1: "Christopher Nolan university education"
   - Branch 2: "Where did Christopher Nolan study film?"
8. Branch 1 retrieves "Nolan studied English Literature at University College London."
9. Evaluator signals STOP -- answer is grounded in evidence.

Output:
```
Answer: Christopher Nolan attended University College London.
Evidence chain:
  [Round 1] "Inception is a 2010 film directed by Christopher Nolan..."
  [Round 2] "Nolan studied English Literature at University College London..."
Rounds used: 2 of 6 max | Branches per round: 2
```

**Example 2: Implementing the context manager in Python**

User: "Implement the context management component that prevents contamination across retrieval rounds."

Approach:
1. Define a `ContextManager` class with `memory` state and relevance scoring.
2. Implement `update()` to merge new documents with query-aware filtering.
3. Implement `select_best()` to pick the winning branch.
4. Implement `merge()` to consolidate cross-branch evidence.

Output:
```python
class ContextManager:
    def __init__(self, relevance_threshold=0.5):
        self.memory = []  # List of (passage, relevance_score) tuples
        self.relevance_threshold = relevance_threshold

    def update(self, current_memory, new_docs, query):
        """Query-aware memory update -- keeps only relevant evidence."""
        branch_memory = list(current_memory)
        for doc in new_docs:
            score = self._score_relevance(doc, query)
            if score >= self.relevance_threshold:
                branch_memory.append((doc, score))
        # Deduplicate by content similarity
        branch_memory = self._deduplicate(branch_memory)
        return branch_memory

    def select_best(self, branches):
        """Pick the branch with highest answer confidence."""
        return max(branches, key=lambda b: b["eval_score"])

    def merge(self, best_memory, other_memories):
        """Merge complementary evidence from non-winning branches."""
        merged = list(best_memory)
        seen = {self._fingerprint(p) for p, _ in best_memory}
        for mem in other_memories:
            for passage, score in mem:
                fp = self._fingerprint(passage)
                if fp not in seen and score >= self.relevance_threshold:
                    merged.append((passage, score))
                    seen.add(fp)
        # Trim to max context window, keeping highest-scored passages
        merged.sort(key=lambda x: x[1], reverse=True)
        return merged[:self.max_passages]
```

**Example 3: Adaptive exit with asymmetric error handling**

User: "Implement the answer evaluator that decides when to stop retrieving."

Approach:
1. Build an evaluator that checks answer correctness and evidence grounding.
2. Apply asymmetric logic: bias toward CONTINUE when confidence is borderline.
3. Return structured decision with reasoning.

Output:
```python
class AnswerEvaluator:
    def __init__(self, llm, confidence_threshold=0.7):
        self.llm = llm
        self.confidence_threshold = confidence_threshold

    def evaluate(self, answer, memory, original_query):
        """Evaluate answer quality and decide STOP or CONTINUE."""
        prompt = f"""Given the question and retrieved evidence, evaluate:
1. Is the answer factually supported by the evidence? (0-1)
2. Is the evidence sufficient to fully answer the question? (0-1)

Question: {original_query}
Evidence: {self._format_memory(memory)}
Answer: {answer}

Return JSON: {{"correctness": float, "grounding": float, "reasoning": str}}"""

        result = self.llm(prompt)
        correctness = result["correctness"]
        grounding = result["grounding"]

        # Asymmetric decision: wrong-stop is 2x worse than wrong-continue
        # Only stop when BOTH signals exceed threshold
        if correctness >= self.confidence_threshold and grounding >= self.confidence_threshold:
            return {"decision": "STOP", "confidence": min(correctness, grounding)}
        return {"decision": "CONTINUE", "confidence": min(correctness, grounding)}
```

## Best Practices

- **Do:** Keep branch-local memories isolated during parallel execution. Only merge after all branches complete and the best branch is selected. This prevents cross-contamination where one branch's irrelevant passages pollute another's context.

- **Do:** Generate sub-queries that are complementary, not redundant. If branch 1 searches for "director of Film X", branch 2 should search for a different facet like "Film X cast and production team", not a paraphrase of the same query.

- **Do:** Implement early exit aggressively for single-hop questions. If the evaluator is confident after round 1, stop immediately. The cost savings compound: a system that exits at round 1 for simple queries can afford more rounds on hard queries.

- **Do:** Track per-round retrieval quality metrics (relevance scores, new-information ratio). If a round adds no new relevant passages, that signals diminishing returns -- consider reducing branch width or exiting.

- **Avoid:** Concatenating all retrieved passages across rounds without filtering. This is the "context contamination" problem -- accumulated noise drowns out signal. Always apply query-aware filtering during memory updates.

- **Avoid:** Setting `D_max` too high without cost controls. Each round multiplies compute by `W` branches. Use `D_max=6` for 7B-class models and `D_max=8` for 30B+ models as starting points, and always track total token usage.

## Error Handling

- **Retrieval returns no relevant documents:** If a branch retrieves nothing above the relevance threshold, mark that branch as failed and do not include it in the merge step. Generate a more specific sub-query for the next round.

- **All branches return CONTINUE at D_max:** Return the answer from the highest-scoring branch with a low-confidence flag. Log which evidence gaps remain so the caller can decide whether to expand the corpus or accept the partial answer.

- **Sub-queries are redundant across branches:** If the Query Rewriter produces near-duplicate sub-queries (detected via embedding similarity > 0.9), regenerate with explicit diversity instructions or fall back to a single branch for that round to save compute.

- **Memory grows beyond context window:** Enforce a hard passage limit (e.g., top-20 by relevance score). When the limit is hit, evict the lowest-scored passages before adding new ones. Never silently truncate -- always evict by relevance rank.

- **Evaluator oscillates between STOP and CONTINUE:** If the evaluator flips decisions across consecutive rounds on the same evidence, force a STOP and return the answer with a medium-confidence flag. Oscillation indicates the question may be ambiguous or the evidence genuinely insufficient.

## Limitations

- **Requires a retrieval corpus.** SPARC-RAG assumes access to a searchable document collection. It does not help with questions that require real-time computation, database queries, or information not present in the corpus.

- **Compute scales multiplicatively.** With `W=2` branches and `D=6` rounds, worst case is 12 retrieval+generation cycles. For latency-sensitive applications, consider `W=1` (pure sequential) or strict `D_max=3`.

- **Query Rewriter quality is the bottleneck.** If sub-queries are poorly decomposed, parallel branches retrieve redundant or irrelevant evidence. The framework's gains depend heavily on generating diverse, targeted sub-queries.

- **Single-hop questions gain little.** The overhead of multi-agent coordination is not justified for simple factoid questions. Use a standard single-pass RAG for queries that don't require reasoning chains.

- **Fine-tuning requires gold evidence labels.** The process-level preference training for the Query Rewriter needs paragraph-level annotations of which passages support the correct answer. Without these labels, fall back to heuristic relevance scoring.

## Reference

- **Paper:** [SPARC-RAG: Adaptive Sequential-Parallel Scaling with Context Management for Retrieval-Augmented Generation](https://arxiv.org/abs/2602.00083v1) (Yang et al., 2026). Look for Algorithm 1 (the main loop), Section 3.2 (context management mechanism), and Section 4 (process-level verifiable preferences for fine-tuning the Query Rewriter and Answer Evaluator).