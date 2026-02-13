---
name: "when-iterative-rag-beats"
description: "Build iterative retrieval-reasoning RAG pipelines that outperform single-shot retrieval, using staged evidence gathering with hypothesis refinement and evidence-aware stopping. Use when: 'build an iterative RAG pipeline', 'multi-hop question answering system', 'implement staged retrieval with reasoning', 'RAG system for scientific questions', 'retrieval loop with stopping criteria', 'diagnose RAG failure modes'."
---

# Iterative Retrieval-Reasoning RAG Pipelines

This skill enables Claude to design and implement iterative RAG systems where retrieval and reasoning alternate in controlled loops, rather than retrieving all context in a single shot. Based on the finding that staged retrieval with hypothesis refinement outperforms even *ideal* (gold/oracle) context by up to 25.6 percentage points on multi-hop scientific QA, this approach is especially powerful for domains requiring chained reasoning over sparse, specialized knowledge. The core insight: giving an LLM all the evidence at once often causes context overload and reasoning failures, while feeding evidence incrementally with intermediate hypothesis checkpoints yields dramatically better answers.

## When to Use

- When building a QA system that must chain facts across 2+ documents or knowledge sources (multi-hop reasoning)
- When the user asks to implement a RAG pipeline for scientific, medical, legal, or other specialized domains with sparse evidence
- When a single-shot RAG system is returning poor answers despite retrieving relevant documents
- When the user wants to add self-correction or iterative refinement to an existing RAG pipeline
- When diagnosing why a RAG system fails on complex questions that require synthesizing multiple pieces of evidence
- When the user needs a training-free RAG controller (no fine-tuning required) that improves answer quality through architecture alone

## Key Technique

**The Iterative RAG Controller** operates on a fixed step budget (typically 5 steps maximum). At each step, the system retrieves evidence, updates a running hypothesis (partial answer), and decides whether to retrieve more or finalize. This contrasts with standard RAG, which retrieves once and generates. The critical innovation is the *synchronized* loop: each retrieval is informed by what the model has reasoned so far, and each reasoning step is grounded only in the evidence gathered to that point. This prevents two major failure modes of single-shot RAG: (1) context overload, where the model drowns in too many passages and loses track of the reasoning chain, and (2) late-hop failure, where the model never connects the final link in a multi-step reasoning chain because it cannot distinguish which retrieved passages matter for which step.

**Context management** is essential to making this work. At each step, the system receives the full top-K passages from the current retrieval (typically K=10) plus a curated selection from prior steps (best 2 passages per prior step). This caps total context at roughly 18 passages at the final step, preventing dilution while preserving the reasoning chain. The **evidence-aware stopping criterion** uses two metrics: a *coverage score* (fraction of required reasoning hops whose key entities appear in retrieved snippets) and a *sufficiency score* (fraction of partial-answer sentences grounded in retrieved text). The system finalizes when both thresholds are met, avoiding both premature stopping and wasteful over-retrieval.

**Failure mode diagnostics** from the paper identify four systematic problems to guard against: (1) *incomplete hop coverage* -- never retrieving a required piece of evidence; (2) *distractor latch* -- locking onto a similar-but-wrong entity and reinforcing it across iterations; (3) *early stopping miscalibration* -- finalizing before all hops are covered; (4) *composition failure* -- having all the right evidence but failing to synthesize it correctly. Awareness of these modes is critical for building robust systems.

## Step-by-Step Workflow

1. **Decompose the question into expected reasoning hops.** Analyze the user's question to estimate how many retrieval steps are needed. A question like "What is the molecular weight of the compound that inhibits enzyme X, which is overexpressed in disease Y?" requires at least 3 hops: disease Y -> enzyme X -> inhibitor compound -> molecular weight.

2. **Set the step budget and context window parameters.** Configure maximum iterations (default: 5), passages per retrieval (default: top-10), and carryover passages per prior step (default: best-2). These prevent context explosion while maintaining chain continuity.

3. **Execute the first mandatory retrieval.** Generate an initial search query directly from the original question. Retrieve top-K passages from your knowledge base. This establishes the evidence foundation.

4. **Generate a partial hypothesis from current evidence.** After each retrieval, produce an intermediate answer that captures confirmed knowledge so far. This hypothesis serves as the reasoning bridge to the next step -- it tells the system what it knows and what it still needs.

5. **Classify the query quality of the next retrieval.** Before issuing a follow-up query, check for four anti-patterns: *vague* (no concrete search targets), *over-broad* (too wide in scope), *fusion* (attempting multiple hops in one query), *off-topic* (unrelated to remaining unknowns). Rewrite queries that match any anti-pattern.

6. **Decide: retrieve again or finalize.** Compute coverage (are key entities from each required hop present in retrieved text?) and sufficiency (are partial-answer claims grounded in evidence?). If coverage >= 0.8 and sufficiency >= 0.6, finalize. Otherwise, formulate a targeted sub-query for the next missing hop and retrieve again.

7. **Manage context across steps.** Pass the full current-step passages plus the best 2 passages from each prior step into the next reasoning call. Do not pass all prior passages -- this causes context overload and is the primary reason single-shot RAG fails on complex questions.

8. **Monitor for anchor carry-drop.** After step 2+, check that key entities from the previous hypothesis appear in the new query. A "carry-drop" (losing track of a key entity) signals hypothesis drift. If detected, regenerate the query while explicitly including the dropped entity.

9. **Guard against distractor latch.** If the system's hypothesis shifts to a new entity that is lexically similar to the previous one (e.g., "benzylic" vs. "phenoxyl"), flag this for verification. Require the model to explicitly justify entity substitutions against retrieved evidence.

10. **Synthesize the final answer with composition checks.** When finalizing, verify the answer entity actually appears in the retrieved evidence (not hallucinated), is specific (not vague paraphrasing), and does not incorrectly merge multiple entities. Composition failure accounts for 87% of errors even with perfect retrieval.

## Concrete Examples

**Example 1: Multi-hop scientific question answering**

```
User: Build me a RAG pipeline that can answer multi-hop chemistry questions
like "What is the boiling point of the solvent used in the Fischer
esterification of benzoic acid?"

Approach:
1. Decompose: This requires 3 hops:
   - Hop 1: Fischer esterification of benzoic acid -> identify the solvent
   - Hop 2: Solvent identification -> confirm specific solvent compound
   - Hop 3: Compound -> look up boiling point

2. Implement the iterative controller:

   class IterativeRAGController:
       def __init__(self, retriever, llm, max_steps=5, top_k=10, carry_k=2):
           self.retriever = retriever
           self.llm = llm
           self.max_steps = max_steps
           self.top_k = top_k
           self.carry_k = carry_k

       def answer(self, question: str) -> dict:
           hypothesis = ""
           all_step_passages = []

           for step in range(self.max_steps):
               # Generate query from question + current hypothesis
               query = self._generate_query(question, hypothesis, step)

               # Retrieve passages
               current_passages = self.retriever.search(query, top_k=self.top_k)
               all_step_passages.append(current_passages)

               # Build context: full current + best-2 from each prior step
               context = self._build_context(all_step_passages)

               # Update hypothesis
               hypothesis = self._refine_hypothesis(question, context, hypothesis)

               # Check stopping criteria
               coverage, sufficiency = self._evaluate_evidence(
                   question, hypothesis, context
               )
               if coverage >= 0.8 and sufficiency >= 0.6:
                   break

           return self._compose_final_answer(question, hypothesis, context)

       def _build_context(self, all_step_passages):
           context = all_step_passages[-1]  # Full current step
           for prior in all_step_passages[:-1]:
               context += self._select_best(prior, k=self.carry_k)
           return context

3. The pipeline retrieves "Fischer esterification" docs first, extracts
   that the solvent is typically sulfuric acid/ethanol mixture or
   refluxing ethanol, then retrieves boiling point data for ethanol
   specifically -- rather than dumping all chemistry docs at once.

Output:
Answer: "The solvent commonly used in Fischer esterification of benzoic
acid is ethanol (as both solvent and reactant). Its boiling point is
78.37 degrees C."
Evidence chain: [Hop1: Fischer esterification procedure] ->
[Hop2: ethanol as solvent] -> [Hop3: ethanol boiling point = 78.37C]
Steps used: 3 of 5 | Coverage: 1.0 | Sufficiency: 0.83
```

**Example 2: Diagnosing an existing RAG system's failures**

```
User: My RAG system answers simple questions fine but fails on anything
requiring 2+ pieces of evidence. How do I fix it?

Approach:
1. Identify the failure mode by running diagnostics on failed questions:

   def diagnose_rag_failure(question, retrieved_docs, model_answer, gold_answer):
       diagnostics = {}

       # Check retrieval coverage: are all required hops present?
       required_hops = decompose_into_hops(question)
       covered = [any(hop_entity in doc for doc in retrieved_docs)
                  for hop_entity in required_hops]
       diagnostics["coverage"] = sum(covered) / len(covered)

       # Check composition: is the answer entity in the context?
       answer_in_context = gold_answer.lower() in " ".join(retrieved_docs).lower()
       diagnostics["composition_failure"] = answer_in_context and (
           model_answer != gold_answer
       )

       # Check distractor latch: similar-but-wrong entities
       similar_entities = find_similar_entities(retrieved_docs, gold_answer)
       diagnostics["distractor_risk"] = len(similar_entities) > 0

       # Check context overload: too many passages
       diagnostics["context_tokens"] = count_tokens(retrieved_docs)
       diagnostics["overload_risk"] = diagnostics["context_tokens"] > 4000

       return diagnostics

2. Common diagnosis results and fixes:
   - coverage < 0.8 -> Switch to iterative retrieval (staged queries)
   - composition_failure = True -> Reduce context size, add explicit
     synthesis prompting, use chain-of-thought for final answer
   - distractor_risk = True -> Add entity verification step
   - overload_risk = True -> Cap passages per step, use carry-k strategy

Output:
Diagnosis: 73% of failures are composition failures (evidence present but
answer wrong). 21% are coverage gaps. 6% are distractor latch.
Recommendation: Implement iterative RAG with max 18 passages in context
and explicit entity verification at each hop.
```

**Example 3: Adding evidence-aware stopping to an existing pipeline**

```
User: I have a RAG loop but it either stops too early or wastes tokens
over-retrieving. How do I add smart stopping?

Approach:
1. Implement the dual-threshold stopping criterion:

   def should_stop(question, hypothesis, retrieved_passages):
       """Evidence-aware stopping with coverage + sufficiency."""
       # Coverage: fraction of required hops with key entity in passages
       hops = extract_required_entities(question)
       passage_text = " ".join(retrieved_passages)
       covered = sum(1 for h in hops if h.lower() in passage_text.lower())
       coverage = covered / len(hops) if hops else 0.0

       # Sufficiency: fraction of hypothesis claims grounded in evidence
       claims = split_into_claims(hypothesis)
       grounded = sum(1 for c in claims if is_grounded(c, passage_text))
       sufficiency = grounded / len(claims) if claims else 0.0

       # Dual threshold -- both must be met
       should_finalize = coverage >= 0.8 and sufficiency >= 0.6

       # Flag overconfidence if finalizing with poor coverage
       overconfident = should_finalize and (coverage < 0.8 or sufficiency < 0.6)

       return {
           "finalize": should_finalize,
           "coverage": coverage,
           "sufficiency": sufficiency,
           "overconfident": overconfident
       }

2. Integrate into the loop: check after each retrieval+reasoning step.
   Log coverage and sufficiency at each step for diagnostics.

Output:
Step 1: coverage=0.33, sufficiency=0.25 -> RETRIEVE MORE
Step 2: coverage=0.67, sufficiency=0.50 -> RETRIEVE MORE
Step 3: coverage=1.00, sufficiency=0.75 -> FINALIZE
Answer generated at step 3 with full coverage and sufficient grounding.
```

## Best Practices

- **Do:** Cap total context at ~18 passages using the carry-k strategy (full current-step + best-2 from each prior step). Context overload is the primary reason single-shot RAG fails on multi-hop questions.
- **Do:** Generate an explicit partial hypothesis after every retrieval step. This intermediate reasoning state is the mechanism that makes iterative RAG outperform single-shot -- it focuses subsequent queries on what's actually missing.
- **Do:** Classify generated queries for quality anti-patterns (vague, over-broad, fusion, off-topic) before executing retrieval. Bad queries compound across iterations.
- **Do:** Monitor anchor carry-drop, especially at step 2 where a universal "transition shock" causes models to discard initial hypotheses at rates up to 47%.
- **Avoid:** Dumping all retrieved documents into a single prompt. The paper shows this causes accuracy to drop even when all correct evidence is present (the Gold Context paradox).
- **Avoid:** Relying solely on the LLM's confidence to decide when to stop. Use explicit coverage and sufficiency metrics grounded in the actual retrieved text. Overconfident early stopping (finalizing with coverage < 0.8) drops accuracy to 61.5%.
- **Avoid:** Allowing more than 5 retrieval steps without strong justification. Diminishing returns set in rapidly, and over-iteration wastes tokens without improving accuracy for most question types.

## Error Handling

- **Incomplete hop coverage** (no relevant document retrieved for a required hop): Rewrite the query with more specific terminology. If coverage remains below 0.5 after 3 steps, fall back to parametric knowledge for that hop and flag the answer as low-confidence.
- **Distractor latch** (system locks onto a similar-but-wrong entity): Detect by checking if the hypothesis entity changed to a lexically similar alternative between steps. Force an explicit comparison step: "Is entity A or entity B supported by passages X, Y, Z?"
- **Composition failure** (right evidence, wrong answer): Add a post-synthesis verification step that checks whether the final answer entity literally appears in the retrieved evidence. If not, regenerate with stricter grounding constraints.
- **Early stopping miscalibration**: Log coverage and sufficiency at each step. If the system frequently finalizes at step 1-2 with poor accuracy, raise the coverage threshold or set a minimum step count of 2.
- **Parametric suppression** (retrieval makes a previously correct answer wrong): When retrieved evidence contradicts parametric knowledge, require the model to cite which specific passage supports the retrieved answer. Models like Claude show the lowest suppression rates (~2.7%) but the risk is non-zero.

## Limitations

- **Latency cost**: Each iteration adds a retrieval call + LLM inference. A 5-step pipeline takes roughly 5x the latency of single-shot RAG. Use for accuracy-critical applications where latency is acceptable.
- **Retriever quality dependency**: The iterative controller cannot fix a fundamentally poor retriever. If relevant documents are not in the index, more iterations will not help -- they may make things worse via distractor latch.
- **Single-hop questions**: For questions requiring only one retrieval step, iterative RAG adds overhead with no accuracy gain. Route simple questions directly to single-shot RAG.
- **Composition failure persists**: Even with perfect retrieval (all hops covered), 58.6% of remaining failures are composition errors where the model fails to correctly synthesize the evidence. This is an LLM reasoning limitation, not a retrieval problem.
- **Domain-specific tuning**: The coverage and sufficiency thresholds (0.8 and 0.6) were validated on chemistry QA. Other domains may need recalibration.

## Reference

**Paper:** "When Iterative RAG Beats Ideal Evidence: A Diagnostic Study in Scientific Multi-hop Question Answering" -- Astaraki et al., 2026. [arXiv:2601.19827v2](https://arxiv.org/abs/2601.19827v2)

Look for: Section 3 (Iterative RAG controller architecture), Section 4 (failure mode taxonomy and diagnostic metrics), Table 2 (Gold Context vs. Iterative RAG accuracy gaps), and Figure 5 (anchor carry-drop dynamics across steps).