---
name: "rethinking-reranker-boundary-aware-evidence"
description: >
  Implement boundary-aware evidence selection for RAG systems using the BAR-RAG technique.
  Replaces relevance-only reranking with difficulty-calibrated evidence selection that targets
  the generator's competence boundary -- passages that are challenging yet solvable.
  Use when: "build a robust RAG pipeline", "improve reranking for noisy retrieval",
  "select better evidence for question answering", "make my RAG system handle noisy documents",
  "implement boundary-aware reranking", "calibrate evidence difficulty for my LLM generator".
---

# Boundary-Aware Evidence Selection for Robust RAG

This skill teaches Claude to build and reason about RAG evidence selection pipelines that go beyond simple relevance scoring. Based on the BAR-RAG technique, the core idea is to reframe the reranker as a **boundary-aware selector** that chooses evidence near the generator's competence boundary -- passages that are neither trivially answer-revealing nor fundamentally unanswerable, but challenging yet sufficient for inference. This produces generators that are robust to realistic retrieval noise and achieve significantly better accuracy without additional inference-time cost.

## When to Use

- When the user is building a RAG pipeline and wants to improve answer quality under noisy retrieval conditions
- When a reranker keeps surfacing passages that leak the answer verbatim or are too vague to be useful
- When the user asks to implement a two-stage selector-generator co-training loop for RAG
- When designing an evidence scoring function that accounts for generator capability, not just query-document relevance
- When the user wants to create a curriculum-style training regime for their RAG generator using retrieval difficulty
- When debugging a RAG system where the generator performs well on easy queries but collapses on realistic, noisy top-K results

## Key Technique

**The Problem with Relevance-Only Reranking.** Standard rerankers score each document independently against the query and return the top-ranked passages. This ignores a critical variable: the generator's ability to reason over the selected evidence set. In practice, this leads to two failure modes. First, the reranker selects **trivial** passages that contain the answer verbatim, so the generator never learns to reason compositionally. Second, it selects **unsolvable** combinations where critical information is absent, causing hallucinations.

**Boundary-Aware Selection.** BAR-RAG introduces a triangular boundary reward function `Rbdy(S) = min(p_hat(S)/c, (1 - p_hat(S))/(1 - c))` where `p_hat(S)` is the estimated probability the generator answers correctly given evidence set `S`, and `c` is a target correctness rate (typically 0.5). This reward peaks when evidence difficulty is perfectly calibrated to the generator's current ability -- the "Goldilocks Zone." Evidence that is too easy (`p_hat ~ 1`) or too hard (`p_hat ~ 0`) gets suppressed.

**Two-Stage Co-Adaptation.** Stage 1 freezes the generator and trains the selector via GRPO (Group Relative Policy Optimization) to find boundary-calibrated evidence sets. Stage 2 freezes the selector and fine-tunes the generator on the induced evidence distribution. Alternating these stages progressively hardens the training signal: as the generator improves, the selector must find harder-yet-solvable combinations, creating a natural curriculum. At inference time, the selector is discarded entirely -- the generator, having been trained on adversarially difficult evidence, handles noisy retrieval robustly with zero additional overhead.

## Step-by-Step Workflow

1. **Set up the retrieval corpus and query set.** Index your document corpus with a dense retriever (e.g., E5, Contriever, or BGE). For each query, retrieve the top-K candidate documents (K=20-50 is typical). Store these as `(query, [doc_1, ..., doc_K])` pairs.

2. **Estimate generator correctness per evidence subset.** For each query, sample M=8 candidate evidence subsets from the top-K pool (e.g., subsets of size 3-5 documents). For each subset S, run K=10 generation rollouts and compute `p_hat(S) = (1/K) * sum(I[reward(a_k) >= delta])` where delta=0.8 is the correctness threshold (using token-F1 or exact match).

3. **Filter degenerate queries.** Remove queries where all evidence subsets yield `p_hat ~ 1` (trivial) or `p_hat ~ 0` (unsolvable). These provide no useful training signal for the selector.

4. **Compute the boundary-aware composite reward.** For each candidate evidence set, calculate:
   ```
   R(S) = R_fmt(S) * (lambda_bdy * R_bdy(S) + lambda_rel * R_rel(S) - P_cnt(S))
   ```
   where `R_bdy` is the triangular boundary function (peaks at `p_hat = c`), `R_rel` is the mean retrieval score of selected documents, `R_fmt` is a binary format check, and `P_cnt` penalizes deviation from the target document count. Use `lambda_bdy=1.0`, `lambda_rel=0.5` as starting weights.

5. **Train the selector (Stage 1).** Fine-tune a reranker model (e.g., a cross-encoder or small LM) using GRPO. Group-normalize rewards across the M candidate subsets per query to compute advantages. Update the selector to prefer evidence sets near the generator's competence boundary. Keep the generator frozen during this stage.

6. **Fine-tune the generator (Stage 2).** Freeze the selector. For each query, use the selector to produce a single evidence set. Generate K rollout answers and reward them with a composite score: `R_g = R_fmt * (lambda_acc * R_acc + lambda_cite * R_cite)` where `R_acc` combines token-F1 (weight 0.7) and exact match (weight 0.3), and `R_cite` rewards citing ~2 source documents. Update the generator via GRPO.

7. **Iterate stages.** Alternate Stages 1 and 2 for T=2-3 iterations. After each cycle, the generator's competence boundary shifts, forcing the selector to find progressively harder evidence combinations -- creating an automatic curriculum.

8. **Deploy with zero overhead.** At inference time, discard the selector entirely. Use the standard retriever to fetch top-K documents and feed them directly to the fine-tuned generator. The generator's robustness comes from its training distribution, not from runtime reranking.

9. **Evaluate with retrieval noise simulation.** Test by injecting distractors (topically similar but non-answering passages) into the evidence set. Compare against the baseline generator on noisy vs. clean evidence to measure robustness gains.

10. **Monitor and recalibrate.** If the domain or retriever changes, re-estimate `p_hat` on a held-out set. If most queries now fall outside the Goldilocks Zone, retrain the selector for one iteration to recalibrate.

## Concrete Examples

**Example 1: Building a Boundary-Aware RAG Training Pipeline**

User: "I have a QA dataset and a retriever that returns top-20 docs per query. My generator hallucinates when retrieval is noisy. Help me implement BAR-RAG-style training."

Approach:
1. Load the query-document pairs and split into train/dev/test
2. Implement the correctness estimator:
   ```python
   def estimate_correctness(generator, query, evidence_subset, n_rollouts=10, delta=0.8):
       correct = 0
       for _ in range(n_rollouts):
           answer = generator.generate(query, evidence_subset, temperature=0.7)
           score = token_f1(answer, gold_answer)
           if score >= delta:
               correct += 1
       return correct / n_rollouts
   ```
3. Implement the boundary reward:
   ```python
   def boundary_reward(p_hat, c=0.5):
       return min(p_hat / c, (1.0 - p_hat) / (1.0 - c))
   ```
4. Sample 8 evidence subsets per query, compute `p_hat` for each, filter queries where all subsets are trivial or unsolvable
5. Train selector with GRPO using the composite reward, then fine-tune generator on selector-curated evidence
6. Iterate 2-3 times and evaluate on held-out noisy retrieval

Output: A fine-tuned generator that handles noisy top-K retrieval without needing the selector at inference time.

**Example 2: Adding Difficulty-Calibrated Evidence Scoring to an Existing Reranker**

User: "I already have a cross-encoder reranker. How do I make it boundary-aware without a full RL loop?"

Approach:
1. Use the existing generator to estimate `p_hat` for the top-K documents in various subsets
2. Label each document combination with its boundary reward score
3. Create training pairs: (query, doc_subset) -> boundary_reward_score
4. Fine-tune the cross-encoder to predict the boundary reward instead of raw relevance:
   ```python
   # Pseudo-training loop
   for query, doc_subsets, boundary_scores in dataloader:
       predicted = cross_encoder(query, doc_subsets)
       loss = mse_loss(predicted, boundary_scores)
       loss.backward()
   ```
5. At inference, use the fine-tuned cross-encoder to rerank, then pass top results to the generator

Output: A reranker that balances relevance with generator-calibrated difficulty, reducing hallucinations on noisy retrieval.

**Example 3: Evaluating Evidence Selection Quality**

User: "How do I measure whether my evidence selection is actually in the Goldilocks Zone?"

Approach:
1. For each test query, collect the selected evidence set S
2. Run 20+ generator rollouts to get a stable `p_hat(S)`
3. Plot the distribution of `p_hat` values across queries:
   ```python
   import matplotlib.pyplot as plt

   p_hats = [estimate_correctness(gen, q, selected[q]) for q in test_queries]
   plt.hist(p_hats, bins=20, range=(0, 1))
   plt.axvline(x=0.5, color='r', linestyle='--', label='Target boundary')
   plt.xlabel('Generator Correctness Probability')
   plt.title('Evidence Difficulty Distribution')
   plt.legend()
   plt.savefig('boundary_calibration.png')
   ```
4. A well-calibrated selector should show a peak near `c=0.5`. A relevance-only reranker typically shows a bimodal distribution clustered at 0 and 1

Output: A histogram showing evidence difficulty distribution, with a clear peak near the target boundary indicating proper calibration.

## Best Practices

- **Do:** Estimate `p_hat` with enough rollouts (>=10) to get stable correctness estimates. Noisy estimates poison the boundary reward signal.
- **Do:** Filter trivial and unsolvable queries before selector training. These queries waste compute and add no gradient signal.
- **Do:** Start with `c=0.5` as the boundary target. Adjust upward (e.g., 0.6) if your generator is strong, or downward (e.g., 0.4) for weaker generators.
- **Do:** Use the composite reward that includes relevance alongside boundary targeting. Pure boundary optimization may select irrelevant but confusing passages.
- **Avoid:** Training the selector and generator simultaneously. The two-stage alternation is essential for stability -- joint training causes reward hacking.
- **Avoid:** Using the selector at inference time. The entire point of the two-stage approach is that the generator internalizes robustness during training, making the selector unnecessary at deployment.

## Error Handling

- **All evidence subsets are trivial (p_hat ~ 1).** The query is too easy for the current generator. Exclude from selector training but keep for generator evaluation.
- **All evidence subsets are unsolvable (p_hat ~ 0).** Either the gold answer is missing from the corpus, or the generator lacks the capability. Exclude from training; diagnose whether the retriever or generator is the bottleneck.
- **Selector collapses to always picking the same documents.** Increase the entropy bonus in GRPO or add diversity regularization to the reward (e.g., penalize high overlap between selected subsets within a batch).
- **Generator performance degrades after Stage 2.** The evidence distribution shifted too aggressively. Reduce `lambda_bdy` or increase `lambda_rel` to keep evidence closer to natural retrieval distributions.
- **Boundary reward is always near zero.** The target `c` is miscalibrated. Re-estimate `p_hat` on a sample of queries and set `c` to the median observed correctness.

## Limitations

- **Requires generator rollouts for training.** Estimating `p_hat` is expensive (10+ generations per evidence subset per query). This makes training significantly more costly than standard reranker fine-tuning.
- **Assumes the gold answer is present in the corpus.** BAR-RAG targets difficulty of *solvable* queries. If critical evidence is genuinely absent, boundary calibration cannot help.
- **Single-generator assumption.** The boundary is calibrated to a specific generator. Swapping generators (e.g., from Llama to Qwen) invalidates the selector and requires retraining.
- **Not suited for open-ended generation.** The correctness estimation relies on verifiable answers (F1, exact match). Creative or long-form generation tasks lack the clean reward signal this method needs.
- **Curriculum effect plateaus.** After 2-3 iterations, the generator's improvement saturates and the selector struggles to find harder-yet-solvable combinations. Diminishing returns beyond T=3.

## Reference

**Paper:** [Rethinking the Reranker: Boundary-Aware Evidence Selection for Robust Retrieval-Augmented Generation](https://arxiv.org/abs/2602.03689v1) (Sun et al., 2026). Look for Section 3 (the boundary reward formulation and two-stage training algorithm) and Table 6 (the iterative evidence hardening case study showing how selected evidence evolves across training iterations).