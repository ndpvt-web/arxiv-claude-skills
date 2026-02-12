---
name: "sparc-rag-adaptive-sequential-parallel-scaling"
description: "Implement multi-agent RAG systems with coordinated sequential-parallel scaling and shared context management for complex multi-hop question answering. Use when: 'build a multi-hop RAG pipeline', 'implement adaptive retrieval with parallel branching', 'create a RAG system that scales retrieval depth and width', 'build a multi-agent retrieval framework', 'implement SPARC-RAG pattern for complex QA', 'design a RAG system with context management across retrieval rounds'."
---

# SPARC-RAG: Adaptive Sequential-Parallel Scaling for Retrieval-Augmented Generation

This skill enables Claude to design and implement multi-agent RAG systems that scale retrieval along two coordinated dimensions: **sequential depth** (iterative refinement across rounds) and **parallel width** (multiple complementary retrieval branches per round). The core innovation from the SPARC-RAG paper is a unified context management mechanism that prevents context contamination -- where accumulated irrelevant evidence degrades answers -- while maximizing information coverage. The system uses five specialized agents (Query Rewriter, Retriever, Answer Generator, Answer Evaluator, Context Manager) that maintain a shared global context, enabling adaptive scaling where simple queries exit early and hard multi-hop queries receive deeper exploration.

## When to Use

- When the user needs a RAG system that handles **multi-hop questions** requiring evidence synthesis across multiple documents (e.g., "Who founded the company that acquired the maker of X?")
- When building a retrieval pipeline where **naive single-pass retrieval fails** because the answer depends on chaining facts from separate sources
- When the user wants to **parallelize retrieval** across complementary sub-queries without accumulating noise from redundant or irrelevant passages
- When designing a system that needs **adaptive compute** -- spending more retrieval rounds on hard questions and exiting early on simple ones
- When implementing a **multi-agent orchestration** pattern for search/retrieval with explicit roles for query rewriting, retrieval, generation, evaluation, and context management
- When the user asks to reduce hallucination in RAG by grounding exit decisions in evidence quality rather than fixed iteration counts

## Key Technique

SPARC-RAG coordinates two scaling dimensions through a loop of parallel expansion followed by sequential refinement. At each round `t`, a **Query Rewriter** agent generates `W` complementary sub-queries from the original question plus accumulated context. Each sub-query spawns an independent branch that retrieves passages, updates a branch-local context, generates a candidate answer, and evaluates that answer. An **Answer Evaluator** agent then selects the best branch and decides CONTINUE or STOP based on answer correctness and evidence grounding. If continuing, a **Context Manager** merges the best branch's context with complementary evidence from other branches, discarding noise, and feeds this consolidated memory into the next round.

The critical design choice is **branch-local context compression before cross-branch consolidation**. Each parallel branch maintains its own focused memory state derived from its specific sub-query. Only after evaluation does the Context Manager selectively merge evidence, preserving complementary information while filtering redundancy. This prevents the "context contamination" problem where indiscriminate accumulation of all retrieved passages degrades answer quality at higher scaling budgets.

The paper also introduces a preference-based fine-tuning strategy: Query Rewriter preferences are constructed from paragraph-level retrieval recall (rewrites that retrieve more gold evidence are preferred), while Answer Evaluator preferences asymmetrically penalize premature stopping (weight `lambda=2` on "wrong stop" errors). This makes the system spend compute wisely -- diverse queries per round, and confident early exits when evidence suffices.

## Step-by-Step Workflow

1. **Define the agent interfaces.** Create five agent roles with clear input/output contracts: `QueryRewriter(question, memory, width) -> [sub_queries]`, `Retriever(query, corpus) -> [passages]`, `AnswerGenerator(question, memory) -> answer`, `AnswerEvaluator(answer, memory, question) -> {decision, score}`, `ContextManager(best_branch, all_branches) -> merged_memory`.

2. **Initialize the global context.** Set `memory = {}` (empty), `round = 1`, and configure scaling bounds: `D_max` (maximum sequential depth, typically 3-5) and `W` (parallel width, typically 2-3; the paper finds W=2 optimal for cost-quality tradeoff).

3. **Generate complementary sub-queries.** At each round, prompt the Query Rewriter with the original question plus current memory. Instruct it to produce `W` queries that explore **orthogonal retrieval directions** -- not paraphrases, but queries targeting different missing evidence. For example, for "Who directed the film starring the actor born in city X?", generate one query about actors born in city X and another about films and their directors.

4. **Execute parallel retrieval branches.** For each sub-query `k` in `{1..W}`, run independently and in parallel: (a) retrieve top-K passages from the corpus, (b) compress retrieved passages into branch-local memory `m'_k` by integrating with prior global memory, (c) generate a candidate answer conditioned on the question and `m'_k`, (d) evaluate the candidate answer for correctness and evidence grounding.

5. **Select the best branch.** Compare evaluation scores across all `W` branches. Select `k*` as the branch with the highest evidence-grounded confidence score.

6. **Decide whether to stop or continue.** If the evaluator for branch `k*` signals STOP (answer is well-supported by retrieved evidence) or `round >= D_max`, return the answer from branch `k*`. Otherwise, proceed to consolidation.

7. **Consolidate cross-branch context.** The Context Manager takes the best branch's memory `m'_k*` as primary and selectively merges complementary (non-redundant) evidence from other branches. This is NOT concatenation -- it is a filtering operation that discards passages not relevant to the original question or already covered by the primary branch.

8. **Advance the sequential loop.** Set `memory = merged_context`, increment `round`, and return to step 3. The enriched memory now informs the next round's sub-query generation, enabling the system to ask increasingly targeted follow-up questions.

9. **Return the final answer with provenance.** When the loop terminates, return the answer along with the evidence chain: which passages from which rounds contributed to the answer, enabling traceability.

10. **Tune scaling parameters based on query difficulty.** For production systems, implement a lightweight classifier or heuristic: single-hop factoid questions get `D_max=1, W=1` (single pass), multi-hop questions get `D_max=3, W=2`, and open-ended synthesis questions get `D_max=5, W=3`.

## Concrete Examples

**Example 1: Multi-hop question answering pipeline**

User: "Build a RAG pipeline that can answer multi-hop questions like 'What university did the director of Inception attend?'"

Approach:
1. Decompose the orchestration loop:

```python
from dataclasses import dataclass, field

@dataclass
class RetrievalBranch:
    sub_query: str
    passages: list[dict]
    local_memory: str
    answer: str
    score: float
    decision: str  # "STOP" or "CONTINUE"

@dataclass
class GlobalContext:
    memory: str = ""
    evidence_chain: list[dict] = field(default_factory=list)

async def sparc_rag(question: str, corpus, llm, D_max=3, W=2):
    ctx = GlobalContext()

    for round_t in range(1, D_max + 1):
        # Phase 1: Generate W complementary sub-queries
        sub_queries = await rewrite_query(llm, question, ctx.memory, W)

        # Phase 2: Execute parallel branches
        branches = await asyncio.gather(*[
            execute_branch(llm, corpus, question, sq, ctx.memory)
            for sq in sub_queries
        ])

        # Phase 3: Select best, decide exit, consolidate
        best = max(branches, key=lambda b: b.score)
        if best.decision == "STOP" or round_t == D_max:
            return best.answer, ctx.evidence_chain

        ctx.memory = await consolidate_context(
            llm, best, branches, question
        )
        ctx.evidence_chain.extend(
            {"round": round_t, "query": best.sub_query,
             "passages": best.passages}
        )

    return best.answer, ctx.evidence_chain
```

2. Implement the Query Rewriter with diversity enforcement:

```python
async def rewrite_query(llm, question, memory, width):
    prompt = f"""Given the question and evidence gathered so far,
generate {width} complementary search queries that target DIFFERENT
missing pieces of evidence. Do NOT paraphrase -- each query should
retrieve distinct information needed to answer the question.

Question: {question}
Evidence so far: {memory or 'None yet'}

Return exactly {width} queries, one per line."""

    response = await llm.generate(prompt)
    return response.strip().split("\n")[:width]
```

3. Implement the Answer Evaluator with grounding check:

```python
async def evaluate_answer(llm, question, answer, memory):
    prompt = f"""Evaluate this answer for the given question.

Question: {question}
Answer: {answer}
Supporting evidence: {memory}

Assess:
1. Is the answer directly supported by the evidence? (not hallucinated)
2. Is the evidence chain complete? (no missing reasoning steps)

Return JSON: {{"score": 0.0-1.0, "decision": "STOP"|"CONTINUE",
"reasoning": "..."}}"""

    return json.loads(await llm.generate(prompt))
```

Output: A pipeline that answers "What university did the director of Inception attend?" by first retrieving "Christopher Nolan directed Inception", then in round 2 retrieving "Christopher Nolan attended University College London", returning the answer with full evidence chain.

---

**Example 2: Context-managed document research agent**

User: "I need an agent that researches complex topics across a document corpus without losing track of what it already knows"

Approach:
1. Implement the Context Manager with selective merging:

```python
async def consolidate_context(llm, best_branch, all_branches, question):
    other_evidence = "\n".join(
        b.local_memory for b in all_branches
        if b.sub_query != best_branch.sub_query
    )

    prompt = f"""You are a context manager. Merge evidence from multiple
retrieval branches into a single coherent memory state.

Primary evidence (best branch):
{best_branch.local_memory}

Other branch evidence:
{other_evidence}

Original question: {question}

Rules:
- Keep all facts from the primary branch
- Add ONLY facts from other branches that are complementary (not redundant)
- Remove passages unrelated to the original question
- Compress into a concise factual summary

Return the merged memory state."""

    return await llm.generate(prompt)
```

2. The key insight: each branch compresses independently before merging, preventing noise accumulation. After 3 rounds with W=2, the system has explored 6 retrieval trajectories but the memory stays focused.

---

**Example 3: Adaptive scaling based on query complexity**

User: "Make my RAG system spend less compute on easy questions and more on hard ones"

Approach:
1. Implement the evaluator-driven early exit:

```python
async def execute_branch(llm, corpus, question, sub_query, memory):
    passages = await retrieve(corpus, sub_query, top_k=5)
    local_mem = await compress_and_merge(llm, memory, passages)
    answer = await generate_answer(llm, question, local_mem)
    eval_result = await evaluate_answer(llm, question, answer, local_mem)

    return RetrievalBranch(
        sub_query=sub_query,
        passages=passages,
        local_memory=local_mem,
        answer=answer,
        score=eval_result["score"],
        decision=eval_result["decision"],
    )
```

2. For the evaluator, bias toward continuing when uncertain (asymmetric cost):

```
- If evidence fully supports the answer -> STOP (score > 0.85)
- If evidence partially supports but gaps remain -> CONTINUE
- If answer contradicts evidence -> CONTINUE (and flag for re-generation)
- Err on the side of CONTINUE: premature stopping is costlier than an extra round
```

Output: Simple factoid questions ("What is the capital of France?") resolve in round 1 with W=1. Multi-hop questions ("Which country's capital hosted the 2024 Olympics, and who was its mayor at the time?") run 2-3 rounds with W=2, retrieving progressively targeted evidence.

## Best Practices

- **Do:** Generate sub-queries that target **different evidence types** (entities, relationships, temporal facts) rather than paraphrasing the same intent. Measure diversity by checking Jaccard overlap of retrieved passages across branches -- aim for <50% overlap.
- **Do:** Compress branch-local context **before** cross-branch merging. Concatenating raw passages from all branches is the primary cause of context contamination.
- **Do:** Asymmetrically weight stopping errors -- penalize premature STOP decisions more heavily than unnecessary CONTINUE decisions (the paper uses lambda=2). An extra retrieval round is cheap; a wrong answer is expensive.
- **Do:** Keep parallel width small (W=2 is optimal in most cases). Beyond W=3, diminishing returns set in rapidly as sub-queries become harder to differentiate.
- **Avoid:** Feeding all retrieved passages from all branches directly into the next round's prompt. This defeats the purpose of parallel exploration and bloats context.
- **Avoid:** Using fixed iteration counts. The entire point of adaptive scaling is that the evaluator controls depth. Hard-coding 3 rounds wastes compute on easy queries and under-serves hard ones.

## Error Handling

- **Retrieval returns no relevant passages:** If a branch retrieves zero useful passages, discard that branch entirely rather than generating an answer from empty evidence. Reduce effective width for that round.
- **All branches produce contradictory answers:** When branches disagree and all have low confidence, generate a synthesis query that explicitly targets the point of disagreement, then run an additional focused retrieval round.
- **Context memory grows too large:** If the consolidated memory exceeds the LLM's effective context window, apply aggressive compression: keep only facts directly referenced in the current best answer and discard supporting-but-tangential evidence.
- **Evaluator always says CONTINUE (infinite loop):** Enforce D_max as a hard ceiling. If the evaluator never triggers STOP within D_max rounds, return the highest-scoring answer across all rounds with a low-confidence flag.
- **Sub-queries are too similar:** If Jaccard overlap of retrieved passages across branches exceeds 70%, the Query Rewriter is failing to diversify. Inject explicit diversity instructions or use the previous round's retrieved passages as negative examples ("Do NOT search for information about X, which we already know").

## Limitations

- **Latency:** Each sequential round adds a full retrieval-generation-evaluation cycle. For real-time applications, cap D_max aggressively (1-2) and prefer wider parallelism.
- **Corpus dependency:** The framework assumes a retrievable corpus exists. It does not help with questions requiring reasoning over structured data (tables, KGs) without a text-based retrieval layer.
- **LLM quality floor:** The Query Rewriter and Answer Evaluator require an LLM capable of following nuanced instructions about diversity and evidence grounding. Smaller models (<7B) may not reliably generate complementary sub-queries or make sound stop/continue decisions.
- **Cost at scale:** Even with adaptive scaling, W=2 and D_max=3 means up to 6 LLM calls per question for generation alone, plus rewriting and evaluation. Budget-constrained deployments should start with W=1 and only enable parallel branching for detected multi-hop queries.
- **Single-hop overhead:** For simple factoid questions, the multi-agent overhead provides no benefit over standard single-pass RAG. Use a query complexity classifier to route simple questions to a fast path.

## Reference

[SPARC-RAG: Adaptive Sequential-Parallel Scaling with Context Management for Retrieval-Augmented Generation](https://arxiv.org/abs/2602.00083v1) -- Focus on Section 3 (framework architecture with the five agent roles), Algorithm 1 (the sequential-parallel loop pseudocode), and Section 4.2 (the DPO-based fine-tuning with asymmetric stopping penalties).