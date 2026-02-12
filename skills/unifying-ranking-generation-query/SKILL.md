---
name: "unifying-ranking-generation-query"
description: "Build production RAG-based query auto-completion systems that unify ranking and generation with multi-objective alignment. Use when: 'build a query autocomplete system', 'implement search suggestions with RAG', 'create type-ahead with LLM generation', 'add multi-objective DPO to search', 'deploy low-latency autocomplete serving', 'generate ranked suggestion lists from prefixes'."
---

# RAG-Based Query Auto-Completion with Multi-Objective Alignment

This skill enables Claude to build query auto-completion (QAC) systems that replace traditional retrieve-and-rank pipelines with end-to-end list generation powered by Retrieval-Augmented Generation and multi-objective Direct Preference Optimization (DPO). Instead of separately retrieving candidates and then ranking them, the LLM synthesizes retrieved context into a complete, optimized suggestion list in a single pass -- enabling holistic list-level optimization that addresses redundancy, safety, relevance, and diversity simultaneously.

## When to Use

- When the user asks to build or improve a search autocomplete / type-ahead suggestion system
- When implementing a RAG pipeline specifically for query suggestion or completion tasks
- When the user wants to replace a traditional retrieve-and-rank QAC pipeline with an LLM-based generator
- When training or fine-tuning a model with multi-objective DPO for search suggestion quality
- When designing a hybrid serving architecture that pre-caches high-frequency completions and generates long-tail ones in real time
- When building a verifier suite (rule-based, model-based, LLM-as-judge) to evaluate search suggestion quality
- When generating synthetic preference data for autocomplete training via critique-revision loops

## Key Technique

**End-to-End List Generation with RAG.** Traditional QAC uses a two-stage pipeline: (1) retrieve candidate completions from logs or an index, (2) rank them with learned-to-rank models. This suffers from limited long-tail coverage and expensive feature engineering. The unified approach reformulates QAC as a single generation task: given a prefix `p` and retrieved context `C`, a language model `M` directly outputs an ordered suggestion list `S = (q_1, ..., q_k)` that maximizes a composite utility function. Retrieved context comes from two sources -- a query index (prefix-to-frequent-completion lookup from historical logs) and a content retriever (embedding-based retrieval with catalog metadata like titles, descriptions, and ratings). This context is compiled into a structured prompt that grounds generation in real searchable content, dramatically reducing hallucination.

**Multi-Objective DPO and Verifier Suite.** The model is aligned across six objectives via Direct Preference Optimization: relevance (plausible intent coverage), safety (policy compliance), engagement (likelihood of downstream user action), catalog groundedness (suggestions map to actual searchable items), context groundedness (derived from retrieved context, not hallucinated), and diversity (multiple intents without near-duplicates). A composite reward `R(p, S) = I_fmt(S) * r_base(p, S)` gates on format correctness and combines weighted per-objective scores. Preference pairs are constructed by sampling multiple candidate lists per prefix, scoring each with the composite reward, and selecting pairs where `r_chosen - r_rejected >= delta` (typically 0.08-0.10). Seven verifiers enforce quality: a format verifier (rule-based XML structure check), relevance verifier (fine-tuned model with position-weighted discounting), engagement verifier (conditional conversion probability), safety verifier (binary policy judgment), catalog groundedness verifier (search backend returns non-empty results), context groundedness verifier (LLM-as-judge with majority voting), and diversity verifier (adjusted entropy penalizing redundant result pages).

**Hybrid Serving for Production Latency.** A two-tier architecture splits traffic: a large generator pre-computes suggestions for high-frequency prefixes offline (sub-millisecond cache lookup), while a compact generator handles cache misses (long-tail, novel prefixes) with real-time inference under ~100ms latency. This balances quality and speed for production deployment.

## Step-by-Step Workflow

1. **Set up the retrieval layer.** Build two retrieval sources: (a) a prefix-to-completion index from historical search logs mapping common prefixes to their most frequent completions, and (b) an embedding-based content retriever that fetches catalog items (titles, descriptions, ratings) relevant to the prefix. Store these in a fast lookup store (e.g., Redis for the prefix index, a vector database for embeddings).

2. **Design the structured prompt template.** Create a prompt that includes: the user's partial prefix, the top-k retrieved query candidates from the prefix index, the top-n catalog items from the content retriever (with metadata), and instructions to output an ordered list of `k` suggestion completions in a structured format (e.g., XML tags like `<suggestions><q>...</q>...</suggestions>`).

3. **Generate seed training data via critique-revision.** Use a teacher LLM to generate initial suggestion lists for a diverse set of prefixes. Feed each output to a critic LLM that provides structured feedback on semantic redundancy, relevance-diversity trade-offs, and coverage gaps. Have the teacher revise based on critique. Repeat until quality plateaus. Augment with ~10% human-labeled examples to produce ~50K prompt-suggestion pairs.

4. **Implement the seven verifiers.** Build each verifier as an independent scoring module:
   - **Format**: regex/rule check that output matches the expected XML structure
   - **Relevance**: fine-tuned classifier with position-weighted DCG-style discounting
   - **Engagement**: score using historical conversion rates and conditional conversion probability
   - **Safety**: binary classifier trained on domain-specific policy violations
   - **Catalog groundedness**: issue each suggested query against the search backend; flag if zero results
   - **Context groundedness**: LLM-as-judge with 3-way majority vote on whether each suggestion derives from retrieved context
   - **Diversity**: compute adjusted entropy `H_adj` with position-weighted probabilities and overlap penalties

5. **Construct DPO preference pairs.** For each prefix in the training set, sample multiple candidate suggestion lists from the model. Score each list using the composite reward (weighted combination of all seven verifier scores, gated by format correctness). Select pairs where `r_chosen - r_rejected >= 0.08`. Keep the top-4 pairs per prefix to balance training signal and cost.

6. **Train with multi-objective DPO.** Fine-tune the base model on the preference pairs using the standard DPO loss. Monitor per-objective metrics during training to detect trade-offs (e.g., engagement optimization slightly reducing diversity). Adjust objective weights if specific dimensions degrade.

7. **Build the hybrid serving architecture.** Deploy two inference paths:
   - **Offline path**: Run the large (highest-quality) model over all high-frequency prefixes, store results in a cache keyed by prefix. Refresh periodically (e.g., daily).
   - **Online path**: Deploy a compact (distilled or quantized) model behind an API with retrieval access. Route cache misses here with a latency budget of ~100ms including retrieval.
   Add a router that checks the cache first and falls back to online generation.

8. **Implement output post-processing.** Parse the structured output, validate format, deduplicate near-identical suggestions, apply safety filtering as a final gate, and verify catalog groundedness by issuing backend searches. Return only validated suggestions.

9. **Evaluate with offline metrics and human judgments.** Measure coverage (% of prefixes with at least one valid suggestion), relevance (position-weighted scoring), unsafe rate, catalog ungrounded rate, and diversity (adjusted entropy). Run pairwise human evaluation comparing against the existing system.

10. **Run a controlled online A/B test.** Deploy to a percentage of production traffic. Track keystrokes-to-completion, suggestion adoption rate (CTR), and downstream conversion. Target benchmarks: ~5% keystroke reduction and ~3.5% adoption increase indicate a successful deployment.

## Concrete Examples

**Example 1: Building a QAC system for an e-commerce search bar**

User: "I need to build an autocomplete system for our product search. We have search logs and a product catalog in Elasticsearch. Currently we just do prefix matching on popular queries."

Approach:
1. Export the top 1M prefix-to-completion mappings from search logs into a Redis hash for fast lookup.
2. Configure Elasticsearch as the content retriever -- for a given prefix, retrieve the top 10 matching products with title, category, and rating.
3. Build the prompt template:
```
Prefix: "{prefix}"
Popular completions: {retrieved_queries}
Relevant products: {retrieved_products}
Generate 8 search suggestions as an ordered list. Each must be a plausible
search query that maps to real products. Diversify across categories.
<suggestions>
<q>suggestion 1</q>
...
</suggestions>
```
4. Implement the verifier suite: format check via regex, catalog groundedness by issuing each suggestion against Elasticsearch (must return >0 hits), diversity via category entropy.
5. Generate 50K training examples using GPT-4 as teacher with Claude as critic, then train a compact 7B model via DPO.
6. Deploy with the hybrid architecture: pre-compute for the top 100K prefixes, serve the rest via the 7B model under 100ms.

Output: A system achieving ~93% prefix coverage (up from ~85% with prefix matching), with diverse, catalog-grounded suggestions and sub-100ms latency.

**Example 2: Adding safety and multi-objective alignment to an existing QAC model**

User: "Our autocomplete sometimes suggests inappropriate or nonsensical queries. We want to add safety filtering and improve overall quality without rebuilding everything."

Approach:
1. Build a safety verifier: fine-tune a small classifier on labeled examples of policy-violating suggestions (offensive content, PII patterns, disallowed categories).
2. Build a context groundedness verifier: use an LLM-as-judge prompt that takes the retrieved context and each suggestion, asking "Is this suggestion derivable from the provided context?" Run 3 judges and take majority vote.
3. Score existing model outputs across all objectives. Identify the weakest dimensions.
4. Construct DPO preference pairs from existing model samples: for each prefix, generate 8 candidate lists, score with composite reward, select pairs with margin >= 0.08.
5. Fine-tune with DPO, weighting safety and groundedness higher (e.g., 2x) to address the identified weaknesses.
6. Add the safety verifier as a post-processing gate in the serving pipeline -- reject any suggestion that fails the safety check and backfill from lower-ranked candidates.

Output: Unsafe rate drops from ~0.8% to ~0.65%, catalog ungrounded rate drops from ~0.67% to ~0.5%, with relevance maintained or improved.

**Example 3: Generating synthetic training data with critique-revision**

User: "We don't have labeled QAC training data. How do I bootstrap high-quality training examples?"

Approach:
1. Sample 50K diverse prefixes from search logs (stratified by frequency: 30% head, 40% torso, 30% tail).
2. For each prefix, retrieve context from the query index and content retriever.
3. Prompt a teacher LLM to generate suggestion lists:
```
Given prefix "{prefix}" and context: {context}
Generate 8 ranked query suggestions optimizing for relevance, diversity,
safety, and catalog coverage. Output in <suggestions> XML format.
```
4. Feed each output to a critic LLM:
```
Evaluate these suggestions for prefix "{prefix}":
{suggestions}
Score each dimension (relevance, diversity, safety, groundedness) 1-5.
Identify: semantic duplicates, missing intent categories, hallucinated items.
Provide specific revision instructions.
```
5. Have the teacher revise based on critique. Repeat up to 3 rounds or until critic scores plateau.
6. Run the automated verifier suite on final outputs. Discard examples where catalog groundedness < 90% or safety fails.
7. Augment with ~5K human-labeled examples for calibration.

Output: ~45K high-quality synthetic training pairs ready for supervised fine-tuning or DPO preference pair construction.

## Best Practices

- **Do** retrieve from both a query frequency index and a content/catalog retriever -- the dual-source approach provides complementary coverage (popular queries + long-tail catalog items).
- **Do** use structured output formats (XML tags) for the suggestion list and enforce them with a format verifier -- this makes parsing reliable and enables the format gate in the composite reward.
- **Do** construct DPO pairs with a minimum margin threshold (delta >= 0.08) -- pairs without meaningful reward difference add noise to training.
- **Do** pre-compute suggestions for high-frequency prefixes offline and serve from cache -- this handles the majority of traffic at sub-millisecond latency.
- **Avoid** optimizing a single objective (e.g., relevance alone) -- this leads to degenerate lists with redundant suggestions that lack diversity and may include unsafe completions.
- **Avoid** generating suggestions without retrieval context -- pure generation without RAG produces hallucinated queries that don't map to real catalog items.
- **Avoid** using a single LLM judge for groundedness verification -- use 3-way majority voting to reduce variance and improve reliability.

## Error Handling

- **Empty retrieval context**: When the prefix is too short or novel to retrieve meaningful candidates, fall back to category-level suggestions or popular queries. Never generate without any grounding signal.
- **Format parsing failures**: If the model output doesn't match the expected XML structure, retry once with an explicit format reminder in the prompt. If it fails again, fall back to cached or retrieval-only results.
- **Catalog groundedness failures**: When a suggestion returns zero search results, drop it from the list and backfill from the retrieved candidate pool rather than serving ungrounded suggestions.
- **Latency budget exceeded**: If online generation exceeds the ~100ms budget, return partial results (top suggestions generated so far) or fall back to the prefix index lookup.
- **Safety verifier flags**: Remove flagged suggestions immediately. If more than half the list is flagged, discard the entire generation and serve from the safe cached/retrieval fallback.
- **DPO training instability**: If per-objective metrics diverge during training (e.g., safety improves but relevance drops sharply), reduce the weight of the dominant objective and increase the margin threshold for pair selection.

## Limitations

- **Latency constraints limit model size**: The online path requires a compact model (~7B parameters or smaller with quantization) to stay within ~100ms, which may underperform the large offline model on complex long-tail prefixes.
- **Catalog dependency**: The catalog groundedness verifier requires a functioning search backend -- this creates a runtime dependency and means suggestion quality is bounded by catalog coverage.
- **Cold-start for new domains**: The approach relies heavily on search logs for the prefix index and engagement verifier. New domains without historical data require bootstrapping with synthetic data, which may underperform.
- **Multi-objective trade-offs are inevitable**: Increasing engagement optimization can mildly reduce diversity, and aggressive safety filtering can reduce coverage. These trade-offs require domain-specific weight tuning.
- **Synthetic data quality ceiling**: The critique-revision loop improves data quality but is bounded by the teacher LLM's capabilities and the critic's judgment accuracy. Human labeling remains important for calibration (~10% of training data).
- **Not suitable for real-time conversational contexts**: This framework is designed for search prefix completion, not open-ended dialogue or conversational AI autocomplete.

## Reference

[Unifying Ranking and Generation in Query Auto-Completion via RAG and Multi-Objective Alignment](https://arxiv.org/abs/2602.01023v3) -- Yuan et al., 2026. Key sections: Section 3 for the end-to-end list generation formulation, Section 4 for the verifier suite and DPO training methodology, Section 5 for the hybrid serving architecture, and Section 6 for the critique-revision synthetic data pipeline.