---
name: "deepera-deep-evidence-reranking"
description: "Rerank retrieved passages for RAG pipelines using step-by-step logical reasoning to filter out semantically similar but logically irrelevant (SSLI) documents. Use when: 'rerank these search results', 'filter irrelevant passages from retrieval', 'build a scientific QA pipeline', 'improve RAG answer quality', 'passages look relevant but answers are wrong', 'reduce hallucinations in retrieval-augmented generation'."
---

# DeepEra: Deep Evidence Reranking for RAG Pipelines

This skill enables Claude to implement a deep evidence reranking strategy based on the DeepEra framework (Chen et al., 2026). Instead of relying on embedding similarity scores alone to rank retrieved passages, Claude applies structured chain-of-thought reasoning to each candidate passage, evaluating whether it **logically supports** answering the query — not merely whether it **shares vocabulary** with it. This directly combats the Semantically Similar but Logically Irrelevant (SSLI) problem, where passages score highly on vector similarity but contribute nothing (or worse, misleading content) to the final answer.

## When to Use

- When building or improving a two-stage RAG pipeline (retrieval + reranking) and retrieved passages frequently mislead the generator.
- When the user reports that their RAG system hallucinates despite retrieving "relevant-looking" documents.
- When implementing scientific question answering over a large corpus where precision matters more than recall.
- When the user asks to rerank search results, filter noisy retrieval output, or score passages for relevance.
- When debugging a RAG pipeline where the retriever returns topically related but logically unhelpful passages (e.g., a passage about "transformer architecture" for a query about "electrical transformers").
- When the user wants to add an LLM-as-judge reranking step to an existing retrieval system.

## Key Technique: Reasoning-Based Reranking to Defeat SSLI

Traditional rerankers (cross-encoders like BGE-Reranker, Jina-Reranker) compute a single relevance score from the concatenated query-passage pair. They are fast but blind to **logical relevance** — a passage about "attention mechanisms in neural networks" will score highly against a query about "attention deficit disorder" because they share the token "attention." These are SSLI passages: they pass the semantic filter but fail the logic test.

DeepEra replaces the single-score paradigm with a **reasoning agent** that evaluates each passage through a structured chain of thought. The agent decomposes the query into its core information need, then examines each candidate passage across four dimensions: (1) whether the passage directly addresses the query's actual question, (2) whether the factual claims in the passage are internally consistent, (3) whether the passage provides evidence that could ground a correct answer, and (4) whether the passage would be useful — not just topically adjacent — for generating a faithful response. Only passages that survive all four checks are promoted.

The critical insight is that **reasoning is cheap relative to retrieval failure**. Running an LLM reasoning pass over 10-20 candidate passages costs far less than generating a hallucinated answer from bad evidence and having to detect/correct it downstream. The two-stage design keeps the approach practical: a fast retriever (dense or sparse) produces a broad candidate set, and the reasoning reranker applies expensive-but-precise evaluation only to the top-k candidates.

## Step-by-Step Workflow

1. **Accept the query and candidate passages.** Receive the user's question and a list of retrieved passages (typically 10-50 candidates from a first-stage retriever like BM25, FAISS, or a dense encoder).

2. **Decompose the query into its core information need.** Rewrite the query as a precise statement of what fact, explanation, or evidence is being sought. Strip away ambiguous phrasing. For example, "What role does attention play in transformers?" becomes "Seeking: explanation of the attention mechanism within transformer neural network architecture."

3. **For each candidate passage, run a four-dimension reasoning evaluation:**
   - **Direct Relevance:** Does this passage contain information that directly answers or contributes to answering the decomposed query? Look past keyword overlap to the actual content.
   - **Logical Consistency:** Are the claims in this passage internally consistent? Does it contradict the query's premises or established facts?
   - **Evidence Grounding:** Does the passage provide concrete evidence (data, citations, experimental results, definitions) rather than vague or tangential discussion?
   - **Generation Utility:** If this passage were the only context provided to an LLM, could it generate a correct and complete answer? Would it mislead the LLM?

4. **Assign a structured verdict to each passage.** Produce a per-passage judgment: STRONG SUPPORT, WEAK SUPPORT, IRRELEVANT, or MISLEADING. Include a one-sentence justification citing the specific reasoning dimension that determined the verdict.

5. **Flag SSLI passages explicitly.** Any passage rated IRRELEVANT or MISLEADING that would have scored in the top-k by embedding similarity alone should be flagged as an SSLI case. Log why: "High semantic overlap on tokens X, Y but passage discusses Z context, not the queried context."

6. **Rerank by verdict tier, then by evidence density within tiers.** STRONG SUPPORT passages come first, ordered by how much concrete evidence they contain. WEAK SUPPORT follows. IRRELEVANT and MISLEADING passages are demoted to the bottom or excluded entirely.

7. **Select the final passage set.** Return the top-n passages (typically 3-5) from the reranked list. Include the reasoning verdict alongside each passage so the downstream generator or the user can audit the selection.

8. **Feed the reranked passages to the generator.** If generating an answer, use only the selected passages as context. Instruct the generator to cite which passage supports each claim in its answer.

9. **Validate the generated answer against the selected evidence.** Cross-check that every factual claim in the generated answer traces back to a STRONG SUPPORT or WEAK SUPPORT passage. Flag unsupported claims.

## Concrete Examples

**Example 1: Scientific QA with SSLI Filtering**

User: "I have 15 passages retrieved for the query 'How does CRISPR-Cas9 achieve target specificity?' — several look relevant but my RAG system keeps hallucinating. Help me rerank them."

Approach:
1. Decompose query: "Seeking: mechanism by which CRISPR-Cas9 identifies and binds to specific DNA target sequences, including guide RNA complementarity and PAM recognition."
2. Evaluate each passage:
   - Passage 3: Describes Cas9 protein structure and guide RNA binding mechanism. **Verdict: STRONG SUPPORT** — directly explains target specificity.
   - Passage 7: Discusses CRISPR applications in agriculture. **Verdict: IRRELEVANT (SSLI)** — high keyword overlap ("CRISPR", "Cas9", "target") but discusses applications, not specificity mechanism.
   - Passage 11: Covers off-target effects and mismatch tolerance. **Verdict: WEAK SUPPORT** — related to specificity but from the failure-mode angle.
   - Passage 2: Reviews gene editing history including zinc finger nucleases. **Verdict: IRRELEVANT (SSLI)** — mentions CRISPR in passing, primarily about older technologies.
3. Reranked order: [3, 11, 5, ...] with passages 7 and 2 demoted.

Output:
```
Reranked passages (top 5):
1. [Passage 3] STRONG SUPPORT — Directly describes guide RNA complementarity
   and PAM sequence recognition as the dual mechanism for target specificity.
2. [Passage 11] WEAK SUPPORT — Discusses mismatch tolerance thresholds,
   indirectly illuminating what governs specificity.
3. [Passage 5] STRONG SUPPORT — Explains the 20-nt guide RNA seed region
   and its role in target discrimination.
4. [Passage 9] WEAK SUPPORT — Covers engineered high-fidelity Cas9 variants,
   relevant but focused on improvements rather than native mechanism.
5. [Passage 14] WEAK SUPPORT — Structural biology of Cas9 conformational
   change upon target binding.

SSLI passages filtered:
- [Passage 7] CRISPR agriculture applications — keyword overlap only.
- [Passage 2] Gene editing history survey — tangential mention of CRISPR.
- [Passage 12] Cas13 RNA targeting — different Cas protein entirely.
```

**Example 2: Building a Reranking Function in Python**

User: "Write me a reranking function that uses LLM reasoning to filter SSLI passages from my retrieval results."

Approach:
1. Implement a function that takes a query and list of passages.
2. For each passage, construct a structured reasoning prompt covering the four evaluation dimensions.
3. Parse the LLM's verdict and justification.
4. Sort by verdict tier and return the reranked list.

Output:
```python
import json
from openai import OpenAI

RERANK_PROMPT = """You are a passage relevance evaluator. Given a query and a passage,
evaluate the passage on four dimensions, then assign a verdict.

Query: {query}
Passage: {passage}

Evaluate:
1. DIRECT RELEVANCE: Does this passage contain information that directly answers
   the query? Look past shared keywords to actual content.
2. LOGICAL CONSISTENCY: Are the passage's claims consistent and non-contradictory?
3. EVIDENCE GROUNDING: Does the passage provide concrete evidence (data, results,
   definitions) or just vague discussion?
4. GENERATION UTILITY: Could an LLM produce a correct answer using only this passage?

Respond in JSON:
{{
  "direct_relevance": "high|medium|low|none",
  "logical_consistency": "consistent|contradictory|unclear",
  "evidence_grounding": "strong|weak|none",
  "generation_utility": "high|medium|low|misleading",
  "verdict": "STRONG_SUPPORT|WEAK_SUPPORT|IRRELEVANT|MISLEADING",
  "justification": "one sentence explaining the verdict",
  "ssli_flag": true/false
}}"""

VERDICT_RANK = {
    "STRONG_SUPPORT": 0,
    "WEAK_SUPPORT": 1,
    "IRRELEVANT": 2,
    "MISLEADING": 3,
}

def rerank_passages(query: str, passages: list[str], client: OpenAI,
                    model: str = "gpt-4o", top_k: int = 5) -> list[dict]:
    """Rerank passages using DeepEra-style reasoning evaluation."""
    results = []
    for i, passage in enumerate(passages):
        prompt = RERANK_PROMPT.format(query=query, passage=passage[:2000])
        response = client.chat.completions.create(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
            temperature=0.0,
        )
        evaluation = json.loads(response.choices[0].message.content)
        results.append({
            "index": i,
            "passage": passage,
            "verdict": evaluation["verdict"],
            "justification": evaluation["justification"],
            "ssli_flag": evaluation.get("ssli_flag", False),
            "sort_key": VERDICT_RANK.get(evaluation["verdict"], 2),
        })

    results.sort(key=lambda r: r["sort_key"])
    return results[:top_k]
```

**Example 3: Auditing an Existing RAG Pipeline for SSLI Vulnerability**

User: "My RAG pipeline retrieves 20 passages per query but answer quality is inconsistent. Can you help me diagnose if SSLI is the problem?"

Approach:
1. Sample 5-10 queries where the pipeline produced poor answers.
2. For each query, retrieve the passages that were used as context.
3. Run the four-dimension evaluation on each passage.
4. Count how many top-5 passages by similarity score are actually IRRELEVANT or MISLEADING.
5. Report the SSLI contamination rate.

Output:
```
SSLI Diagnostic Report
======================
Queries sampled: 8
Total passages evaluated: 160 (20 per query)

SSLI contamination in top-5 by similarity:
- Query 1: 2/5 passages flagged SSLI (40%)
- Query 2: 3/5 passages flagged SSLI (60%)
- Query 3: 1/5 passages flagged SSLI (20%)
- Query 4: 4/5 passages flagged SSLI (80%)  <-- worst case
- Query 5: 1/5 passages flagged SSLI (20%)
- Query 6: 2/5 passages flagged SSLI (40%)
- Query 7: 0/5 passages flagged SSLI (0%)
- Query 8: 3/5 passages flagged SSLI (60%)

Average SSLI rate in top-5: 40%
Recommendation: SSLI is a significant issue. Implement reasoning-based
reranking on top-20 candidates before passing top-5 to the generator.
Expect answer quality improvement proportional to SSLI reduction.
```

## Best Practices

- **Do:** Decompose the query before evaluating passages. A precise information-need statement prevents your reasoning from drifting toward keyword matching.
- **Do:** Evaluate all four dimensions independently before assigning a verdict. A passage can be directly relevant but logically inconsistent (e.g., it contains the right topic but wrong claims).
- **Do:** Set temperature to 0 for the reranking LLM calls. Consistency in verdicts matters more than creativity.
- **Do:** Batch reranking calls where possible. The four-dimension evaluation for each passage is independent, so parallelize across passages.
- **Avoid:** Reranking more than 20-30 passages per query. The reasoning step is expensive — use the first-stage retriever to narrow the candidate set first.
- **Avoid:** Treating the reranker verdict as infallible. Use it to reorder, not as a hard filter. Keep a WEAK SUPPORT passage over discarding it entirely if the STRONG SUPPORT set is thin.
- **Avoid:** Skipping the SSLI flag. Logging which passages are semantically similar but logically irrelevant builds a diagnostic dataset you can use to improve your retriever over time.

## Error Handling

- **LLM returns unparseable JSON:** Wrap the verdict parsing in a try/except. On failure, assign a default verdict of WEAK SUPPORT (conservative — don't discard a passage you couldn't evaluate).
- **All passages rated IRRELEVANT:** If no passage receives STRONG or WEAK SUPPORT, fall back to the original similarity-ranked order and flag the query as having no good evidence. Do not generate an answer from IRRELEVANT passages — return "insufficient evidence" instead.
- **Timeout on reranking calls:** Set per-passage timeouts. If a passage evaluation times out, keep the passage at its original rank rather than discarding it.
- **Contradictory verdicts on re-evaluation:** If you rerank the same passages twice and get different verdicts, the query or passage is ambiguous. Flag for human review.
- **Cost overruns:** Track token usage per reranking call. If the pipeline is too expensive, reduce the candidate set size or switch to a smaller model for initial screening, reserving the full reasoning pass for borderline cases.

## Limitations

- **Latency:** Reasoning-based reranking adds significant latency compared to cross-encoder rerankers. Not suitable for real-time search with sub-100ms requirements. Best applied in batch processing or when answer quality justifies the cost.
- **LLM dependency:** The reranker's quality is bounded by the reasoning ability of the underlying LLM. Smaller models may produce unreliable verdicts, especially on highly technical scientific passages.
- **Domain knowledge gaps:** The four-dimension evaluation assumes the LLM has sufficient domain knowledge to judge logical relevance. For highly specialized fields (e.g., advanced mathematics, niche subfields), the LLM may misclassify passages.
- **Not a retriever replacement:** DeepEra-style reranking cannot recover relevant passages that the first-stage retriever missed entirely. It only reorders what was already retrieved.
- **Scale ceiling:** Evaluating 50+ passages per query with full reasoning is impractical. The technique works best as a precision-focused second stage on a pre-filtered candidate set of 10-20 passages.

## Reference

Chen, H., Long, Q., Pu, S., Luo, X., & Ju, W. (2026). *DeepEra: A Deep Evidence Reranking Agent for Scientific Retrieval-Augmented Generated Question Answering.* arXiv:2601.16478v1. https://arxiv.org/abs/2601.16478v1

Key takeaway: The paper demonstrates that 40%+ of top-ranked passages in standard RAG pipelines can be SSLI (semantically similar but logically irrelevant), and that structured reasoning-based reranking significantly outperforms embedding-based rerankers (BGE-Reranker, Jina-Reranker-v2, mxbai-rerank) on scientific QA benchmarks across 10 subject domains.