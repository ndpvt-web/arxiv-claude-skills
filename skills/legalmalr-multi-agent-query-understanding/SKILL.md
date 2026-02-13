---
name: "legalmalr-multi-agent-query-understanding"
description: "Multi-agent query reformulation and LLM reranking for retrieval over legal, regulatory, or domain-specific corpora. Use when building legal search systems, statute retrieval pipelines, or any RAG system where user queries are ambiguous, colloquial, or multi-issue. Triggers: 'build a legal search system', 'improve statute retrieval', 'multi-agent query reformulation', 'legal RAG pipeline', 'rerank legal search results', 'query understanding for retrieval'"
---

# LegalMALR: Multi-Agent Query Understanding and LLM-Based Reranking

This skill teaches Claude to build retrieval systems that use **multiple specialized agents** to reformulate ambiguous user queries into precise, legally grounded search terms, then apply **LLM-based zero-shot reranking** to select the most relevant documents. The approach comes from the LegalMALR paper, which demonstrated that decomposing query understanding into six agent roles with iterative dense retrieval and a final LLM reranker substantially outperforms single-pass RAG on both in-distribution and out-of-distribution legal benchmarks (+6-9% recall, +6-8% MRR).

## When to Use

- When building a statute, regulation, or case-law retrieval system where user queries are informal, implicit, or multi-issue
- When a single-pass dense retrieval pipeline returns low recall because queries use colloquial language instead of legal terminology
- When implementing a RAG pipeline for any specialized domain (medical, financial, compliance) where query-document vocabulary mismatch is a core problem
- When the user asks to "improve search quality" or "handle ambiguous queries" in a document retrieval system
- When designing a multi-agent system where each agent contributes a different perspective on query interpretation
- When reranking retrieved candidates using an LLM's reasoning ability rather than a lightweight cross-encoder

## Key Technique

**Multi-Agent Query Understanding System (MAS):** Instead of rewriting a query once, LegalMALR instantiates six agents from the same base LLM, each differentiated by a system prompt that gives it a distinct reformulation strategy. A Planner agent orchestrates an iterative loop: it inspects the current candidate pool, selects the most promising reformulation agent for the next round, and decides when to stop. Each reformulated query triggers independent dense retrieval (top-30) and lightweight reranking (top-10), with results merged via deduplication. Simple queries resolve in one iteration; complex ones average ~2 rounds, never exceeding four. This broadens candidate coverage without proportional compute cost.

**GRPO Policy Optimization:** LLM-generated rewrites are stochastic -- the paper measured a gap of ~6 recall points between best and average rollouts. To stabilize this, the entire MAS (planner decisions + agent rewrites) is treated as a single policy optimized with Generalized Reinforcement Policy Optimization. The reward combines terminal recall against gold labels, a step penalty (-0.05 per iteration to discourage unnecessary loops), intermediate hit bonuses for newly discovered gold documents, and a harsh penalty (-5) for invalid terminations. Eight rollout trajectories per query with group-wise normalization train lightweight LoRA adapters while the backbone stays frozen.

**LLM Reranker:** After MAS accumulates ~14 candidate statutes, a large commercial LLM (e.g., GPT-4, Qwen-Max) evaluates each candidate against the original query in zero-shot mode. It assesses doctrinal applicability, factual alignment, and conditional structure, outputting a compact JSON ranked list without chain-of-thought to maintain determinism. This lifts recall@10 from ~0.72 to ~0.81 on held-out benchmarks.

## Step-by-Step Workflow

1. **Define the agent roles.** Create six agent configurations, each with a distinct system prompt:
   - *Planner*: Analyzes the query, selects which reformulation strategy to apply next, monitors candidate pool growth, and decides when to terminate.
   - *Single-Element Rewriter*: Converts colloquial or vague expressions into precise domain terminology (e.g., "kicked out of my apartment" -> "unlawful eviction of residential tenant").
   - *Supplementary-Element Rewriter*: Makes implicit conditions explicit (e.g., adds "without written notice" or "during lease term" when context implies them).
   - *Multi-Element Decomposer*: Splits compound queries into focused sub-queries, each targeting a single legal issue.
   - *Supportive-Law Rewriter*: Generates queries targeting procedural or interpretive provisions related to the core issue.
   - *Semantic-Abnormality Repairer*: Fixes contradictions, procedural dependency errors, or missing causal links in the query.

2. **Implement the Planner's iterative loop.** The Planner receives the original query and the current candidate pool summary (count, diversity score, growth rate). It outputs: (a) the next agent to invoke, or (b) a termination signal. Cap iterations at 4 to bound compute.

3. **Execute reformulation.** Pass the original query (and current candidates if relevant) to the selected agent. The agent returns one or more reformulated queries. For the Multi-Element Decomposer, expect multiple sub-queries.

4. **Run dense retrieval per reformulated query.** Use an embedding model (e.g., `text-embedding-3-large`, domain-tuned BERT, or Qwen3-Embedding) to retrieve top-30 candidates from your corpus index.

5. **Apply lightweight reranking.** Use a cross-encoder or small reranker model to prune each retrieval result from 30 to 10 candidates.

6. **Merge and deduplicate.** Accumulate all candidates across iterations into a unified pool. Remove duplicates by document ID. Track pool growth -- if fewer than 2 new unique candidates were added, signal diminishing returns to the Planner.

7. **Feed the accumulated pool to the LLM Reranker.** Construct a prompt containing the original user query and all candidate documents (typically 10-20). Instruct the LLM to evaluate each candidate for domain applicability, factual alignment, and conditional relevance, then output a JSON array of document IDs ranked by relevance.

8. **Return the top-K results.** Extract the top 10 (or user-specified K) from the LLM Reranker's output as the final answer set.

9. **(Optional) Apply GRPO for production systems.** If you have gold-labeled query-document pairs, treat the full MAS trajectory (planner decisions + agent outputs) as a policy. Sample 8 rollouts per query, compute rewards (terminal recall + step penalty + hit bonus), normalize within the group, and update LoRA adapters via policy gradient. This typically yields +4-5 recall points.

10. **Evaluate with recall@K, MRR@K, and nDCG@K.** Always measure against a held-out test set. Compare single-pass retrieval vs. MAS-only vs. MAS+Reranker to quantify each component's contribution.

## Concrete Examples

**Example 1: Building a Legal Statute Retrieval API**

User: "Build a retrieval system for Chinese legal statutes that handles informal queries like '房东不退押金怎么办' (What do I do if my landlord won't return my deposit?)."

Approach:
1. Define the six agent system prompts in a config file. Each prompt instructs the LLM to reformulate from a specific angle:
   ```python
   AGENT_PROMPTS = {
       "planner": "You are a legal query analysis planner. Given a user query and the current retrieval pool status (candidate count, new additions), decide which reformulation strategy to apply next or whether to terminate. Output JSON: {\"action\": \"rewrite_single\" | \"rewrite_supplement\" | \"decompose\" | \"supportive_law\" | \"repair\" | \"terminate\"}",
       "single_element": "You are a legal terminology specialist. Rewrite the user's colloquial query into precise legal language. Preserve all factual elements but replace informal terms with statutory terminology. Output only the rewritten query.",
       "supplementary_element": "You are a legal context expander. Identify implicit conditions in the query (time limits, procedural requirements, parties involved) and make them explicit in a rewritten query.",
       "multi_element_decompose": "You are a legal issue decomposer. Split this multi-issue query into separate single-issue queries. Output a JSON array of sub-queries.",
       "supportive_law": "You are a procedural law specialist. Generate a query targeting procedural provisions, judicial interpretations, or administrative regulations related to the user's core issue.",
       "semantic_repair": "You are a legal logic validator. Check the query for contradictions, missing causal links, or procedural dependency errors. Output a corrected query."
   }
   ```
2. Implement the iterative loop:
   ```python
   def mas_retrieve(query: str, corpus_index, max_iterations=4):
       candidate_pool = set()
       for i in range(max_iterations):
           pool_summary = f"Candidates: {len(candidate_pool)}, Iteration: {i}"
           action = call_planner(query, pool_summary)
           if action == "terminate":
               break
           reformulated = call_agent(action, query)
           for rq in (reformulated if isinstance(reformulated, list) else [reformulated]):
               hits = dense_retrieve(rq, corpus_index, top_k=30)
               reranked = lightweight_rerank(query, hits, top_k=10)
               new_adds = set(reranked) - candidate_pool
               candidate_pool.update(reranked)
           if len(new_adds) < 2:  # diminishing returns
               break
       return list(candidate_pool)
   ```
3. Apply LLM reranking on the pool:
   ```python
   def llm_rerank(query: str, candidates: list, top_k=10) -> list:
       prompt = f"""Given the legal query: "{query}"
   Evaluate each candidate statute for: (1) doctrinal applicability, (2) factual alignment, (3) conditional relevance.
   Return a JSON array of statute IDs ranked by relevance, most relevant first.
   Candidates:\n{format_candidates(candidates)}
   Output format: {{"ranked": ["id1", "id2", ...]}}"""
       result = call_llm(prompt)
       return parse_ranked_ids(result)[:top_k]
   ```

Output: A retrieval API where the informal query "房东不退押金怎么办" triggers three iterations -- single-element rewrite produces "租赁合同押金返还纠纷" (lease deposit return dispute), supplementary rewrite adds "合同期满后" (after lease expiration), and the decomposer splits into deposit return obligations and tenant remedies -- yielding ~14 candidate statutes that the LLM reranker narrows to the 10 most relevant.

**Example 2: Adapting for Compliance Document Retrieval**

User: "Our compliance team searches a corpus of 5,000 regulatory documents but queries are often vague like 'do we need to report this transaction'. Improve retrieval quality."

Approach:
1. Adapt the six agent roles to the compliance domain:
   - Single-Element Rewriter: maps "report this transaction" -> "suspicious transaction reporting obligation under AML regulations"
   - Supplementary-Element Rewriter: adds implicit context like transaction threshold amounts, reporting timelines, entity types
   - Multi-Element Decomposer: splits into reporting triggers, reporting procedures, and penalty provisions
2. Use the same iterative MAS loop with domain-specific embedding model
3. Apply LLM reranker with a compliance-specific evaluation rubric: regulatory applicability, jurisdictional match, temporal validity

Output: A pipeline that transforms "do we need to report this transaction" into 3 focused sub-queries, retrieves ~15 candidate regulations across AML, KYC, and sanctions frameworks, and reranks them to surface the specific reporting thresholds and procedures applicable to the user's jurisdiction.

**Example 3: Query Reformulation Without Full Pipeline**

User: "I just need the multi-agent query reformulation part -- my retrieval is fine but queries are bad."

Approach:
1. Implement only the MAS component as a query expansion preprocessor:
   ```python
   def expand_query(query: str) -> list[str]:
       expanded = [query]  # always include original
       expanded.append(call_agent("single_element", query))
       expanded.append(call_agent("supplementary_element", query))
       sub_queries = call_agent("multi_element_decompose", query)
       expanded.extend(sub_queries)
       return deduplicate(expanded)
   ```
2. Feed all expanded queries to the existing retrieval system
3. Merge results with reciprocal rank fusion or simple union + deduplication

Output: A lightweight query expansion module that turns one ambiguous query into 3-6 precise reformulations, improving recall without changing the retrieval backend.

## Best Practices

- **Do:** Keep the Planner agent stateless between iterations -- pass the full context (original query + pool stats) each time. This makes the system easier to debug and parallelize.
- **Do:** Cap iterations at 4 and track diminishing returns (fewer than 2 new candidates per round). The paper found no query needed more than 4 rounds.
- **Do:** Use the original user query (not reformulations) as the reference for the LLM Reranker. Reformulations broaden recall; the reranker judges relevance to the actual user intent.
- **Do:** Output the reranker result as structured JSON without chain-of-thought. CoT increases output variability and parsing complexity without improving ranking quality in this setting.
- **Avoid:** Running all six agents on every query. The Planner should select the most relevant 1-3 agents based on query characteristics. Simple queries need only one rewrite round.
- **Avoid:** Using the LLM Reranker on more than ~20 candidates. Cost and latency scale linearly; the paper's sweet spot was 10-15 candidates.
- **Avoid:** Skipping the lightweight reranker stage. Dense retrieval top-30 contains noise; pruning to top-10 per round before merging keeps the candidate pool manageable.

## Error Handling

- **LLM returns malformed JSON from reranker:** Wrap the reranker call with retry logic (up to 2 retries) and a JSON schema validator. Fall back to the lightweight reranker's ordering if all retries fail.
- **Planner enters infinite loop:** The hard cap of 4 iterations prevents this, but also check for the Planner selecting the same agent consecutively with no new candidates -- force termination after 2 consecutive no-growth rounds.
- **Reformulation agent produces hallucinated legal terms:** Validate reformulated queries by checking that key terms appear in the corpus vocabulary. If >50% of terms are unseen, discard that reformulation and log it.
- **Candidate pool is empty after first iteration:** The original query likely has zero relevant documents in the corpus. Return an empty result with a confidence flag rather than forcing more reformulation rounds.
- **High latency from multiple LLM calls:** Parallelize the dense retrieval + lightweight reranking for each reformulated query. The LLM Reranker is the bottleneck -- batch candidates into a single prompt rather than scoring individually.

## Limitations

- The multi-agent approach adds latency (2-4 LLM calls for reformulation + 1 for reranking). Not suitable for sub-100ms retrieval requirements without aggressive caching.
- GRPO optimization requires gold-labeled query-document pairs, which are expensive to annotate for new domains. Without GRPO, expect ~4-5 recall points lower but still better than single-pass.
- The LLM Reranker uses a commercial model in zero-shot mode. Quality depends heavily on the base model's domain knowledge -- works well for law, finance, and medicine where LLMs have strong training data, less well for niche technical domains.
- The six-agent taxonomy was designed for legal queries. Other domains need agent role redesign -- the decomposition strategies that work for legal issues (element clarification, procedural provisions, multi-issue splitting) may not map directly.
- Evaluated primarily on Chinese legal corpora. Cross-lingual transfer to other legal systems requires re-tuning the reformulation agents for jurisdiction-specific terminology.

## Reference

[LegalMALR: Multi-Agent Query Understanding and LLM-Based Reranking for Chinese Statute Retrieval](https://arxiv.org/abs/2601.17692v1) -- Li et al., 2026. Focus on Section 3 (MAS architecture and agent roles), Section 4 (GRPO reward design), Section 5 (LLM Reranker prompt structure), and Appendix B/C for agent and reranker prompt templates.