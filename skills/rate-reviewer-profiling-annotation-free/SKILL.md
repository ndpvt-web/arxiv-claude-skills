---
name: "rate-reviewer-profiling-annotation-free"
description: |
  Build reviewer-paper matching systems using keyword-based reviewer profiling and annotation-free weak supervision training.
  Implements the RATE framework for expertise ranking in peer review: distill publication histories into compact keyword profiles,
  construct weak preference pairs from heuristic retrieval signals, and fine-tune embedding models for reviewer-manuscript matching.

  Trigger phrases:
  - "match reviewers to papers"
  - "build a reviewer assignment system"
  - "rank reviewers by expertise for this paper"
  - "create reviewer profiles from publications"
  - "peer review matching without labeled data"
  - "reviewer expertise ranking"
---

# RATE: Reviewer Profiling and Annotation-Free Training for Expertise Ranking

This skill enables Claude to build reviewer-paper matching systems using the RATE framework. RATE constructs compact keyword-based profiles from each reviewer's recent publications, then fine-tunes an embedding model using weak preference supervision derived from heuristic retrieval signals (no human annotations required). The result is a system that ranks candidate reviewers for any manuscript by embedding both the paper and each reviewer profile into a shared space and computing similarity. This approach outperforms strong embedding baselines on the LR-bench and CMU gold-standard benchmarks.

## When to Use

- When the user needs to assign reviewers to conference or journal submissions based on expertise
- When building an automated paper-reviewer matching pipeline for a workshop, venue, or internal review process
- When the user wants to create structured expertise profiles from a researcher's publication history
- When ranking a pool of candidate reviewers for a specific manuscript without manually labeled training data
- When the user asks to implement keyword-based researcher profiling from Semantic Scholar, DBLP, or OpenReview data
- When migrating from simple keyword/TF-IDF matching to a learned embedding approach for reviewer assignment
- When evaluating reviewer-paper matching quality using pairwise ranking metrics (paper-centric or reviewer-centric)

## Key Technique

**Reviewer Profile Construction.** RATE takes a fundamentally reviewer-centric approach. Instead of comparing a manuscript's full text against each reviewer's full publication corpus (expensive and noisy), RATE distills each reviewer's recent publications into a compact keyword-based profile. An LLM extracts topical keywords from each paper's title and abstract, then aggregates and deduplicates them across the reviewer's publication list. The result is a concise textual profile (typically 50-200 keywords organized by theme) that captures the reviewer's research expertise without retaining verbose abstracts. This compression step is critical: it reduces noise, normalizes terminology, and makes embedding comparison tractable.

**Annotation-Free Weak Supervision.** The key innovation is eliminating the need for expert-annotated paper-reviewer-score labels. RATE constructs weak preference training pairs using heuristic retrieval signals. First, a fast retrieval method (e.g., BM25 or TF-IDF cosine) scores all reviewer profiles against a corpus of papers. For each paper, reviewers whose profiles score highly are treated as "positive" matches, while low-scoring reviewers become "negatives." These heuristic pairs form triplets: (manuscript, better-matched reviewer profile, worse-matched reviewer profile). The embedding model is then fine-tuned with a contrastive or ranking loss on these triplets, learning to surpass the heuristic signal that generated the training data. This self-improving property is what allows RATE to beat both the heuristic baseline and strong off-the-shelf embeddings.

**Embedding Fine-Tuning and Inference.** RATE fine-tunes a pre-trained text embedding model (e.g., `intfloat/e5-base-v2` or similar sentence transformer) on the weak preference triplets using a margin-based ranking loss. At inference time, the manuscript's title and abstract are embedded, each reviewer's keyword profile is embedded, and cosine similarity produces a ranking. The framework evaluates from two complementary perspectives: paper-centric (for a given paper, rank reviewers) and reviewer-centric (for a given reviewer, rank papers by familiarity).

## Step-by-Step Workflow

1. **Collect reviewer publication data.** For each candidate reviewer, gather their recent publications (last 2-3 years) including titles and abstracts. Use Semantic Scholar API, DBLP, or OpenReview as data sources. Store as a list of `{author_id, papers: [{title, abstract}]}` records.

2. **Extract keywords from each publication.** For every paper in each reviewer's list, use an LLM to extract 5-15 topical keywords from the title and abstract. Prompt pattern:
   ```
   Extract 5-15 specific research keywords from this paper.
   Focus on methods, tasks, domains, and techniques.
   Title: {title}
   Abstract: {abstract}
   Return keywords as a comma-separated list.
   ```

3. **Aggregate keywords into reviewer profiles.** For each reviewer, merge keywords across all their papers. Deduplicate near-synonyms (e.g., "LLM" and "large language model"), count frequency, and retain the top-k most distinctive keywords (typically k=50-150). Format as a single text string: `"expertise: keyword1, keyword2, ..., keywordN"`.

4. **Prepare the manuscript corpus.** For each manuscript to be assigned, extract its title and abstract. Format as: `"title: {title}. abstract: {abstract}"`.

5. **Generate weak preference pairs via heuristic retrieval.** Run BM25 or TF-IDF cosine similarity between each manuscript and all reviewer profiles. For each manuscript, select the top-ranked reviewer profiles as positives and bottom-ranked as negatives. Construct triplets: `(manuscript_text, positive_profile, negative_profile)`. Aim for 5-10 triplets per manuscript with score margin filtering (positive score must exceed negative by a threshold).

6. **Fine-tune the embedding model.** Load a pre-trained sentence embedding model (e.g., `sentence-transformers/all-MiniLM-L6-v2` or `intfloat/e5-base-v2`). Fine-tune on the triplets using `MultipleNegativesRankingLoss` or `TripletLoss` from the `sentence-transformers` library. Train for 1-3 epochs with a learning rate of 2e-5 and batch size 16-32.

7. **Embed and rank at inference.** Encode each manuscript and each reviewer profile using the fine-tuned model. Compute cosine similarity between the manuscript embedding and every reviewer profile embedding. Sort reviewers by descending similarity to produce the expertise ranking.

8. **Apply conflict-of-interest and load-balancing constraints.** Filter out reviewers who are co-authors or from the same institution. Optionally apply a maximum review load per reviewer and solve assignment as a constrained optimization (e.g., linear sum assignment).

9. **Evaluate using pairwise ranking metrics.** For each paper (paper-centric view), form all reviewer pairs and check if the model ranks the more-expert reviewer higher. For each reviewer (reviewer-centric view), form all paper pairs and check ordering. Report pairwise accuracy, NDCG, and MAP.

10. **Iterate on profile quality.** If performance is low, refine keyword extraction prompts, adjust the number of keywords per profile, or increase the margin threshold for weak supervision pair construction.

## Concrete Examples

**Example 1: Building a Workshop Reviewer Assignment System**

```
User: I'm organizing an NLP workshop and have 30 reviewers and 85 submissions.
Each reviewer has a Semantic Scholar profile. Help me build a matching system.

Approach:
1. Fetch each reviewer's last 3 years of publications via the Semantic Scholar API:
   GET https://api.semanticscholar.org/graph/v1/author/{id}/papers?fields=title,abstract&limit=50

2. Extract keywords from each paper using an LLM. Example output for one reviewer:
   "retrieval-augmented generation, dense passage retrieval, knowledge grounding,
    open-domain QA, contrastive learning, bi-encoder, FAISS indexing, entity linking"

3. Build reviewer profiles by aggregating keywords across all papers:
   reviewer_profiles = {
     "reviewer_042": "expertise: dense retrieval, RAG, passage ranking, ...",
     "reviewer_078": "expertise: syntax parsing, dependency trees, treebanks, ..."
   }

4. Run BM25 over profiles against each submission's title+abstract:
   from rank_bm25 import BM25Okapi
   tokenized_profiles = [p.split() for p in profiles.values()]
   bm25 = BM25Okapi(tokenized_profiles)
   scores = bm25.get_scores(submission_tokens)

5. Construct ~500 triplets from top-5 / bottom-5 reviewer matches per paper.

6. Fine-tune sentence-transformers model on triplets:
   from sentence_transformers import SentenceTransformer, losses, InputExample
   model = SentenceTransformer("intfloat/e5-base-v2")
   train_loss = losses.TripletLoss(model=model)
   # Train for 2 epochs

7. Rank all 30 reviewers for each of the 85 papers using cosine similarity.

Output:
   Paper "Efficient KV-Cache Compression for Long-Context LLMs":
     1. reviewer_042 (score: 0.87) - expertise in LLM efficiency, KV-cache
     2. reviewer_015 (score: 0.81) - expertise in transformer optimization
     3. reviewer_063 (score: 0.74) - expertise in memory-efficient inference
```

**Example 2: Quick Reviewer Ranking Without Fine-Tuning (Baseline)**

```
User: I just need to rank 5 reviewers for a single paper. Is fine-tuning necessary?

Approach:
1. For a single paper, skip fine-tuning. Use keyword profiling + off-the-shelf
   embeddings as a strong baseline.

2. Extract keywords for each reviewer's recent publications:
   reviewer_A: "code generation, program synthesis, unit testing, LLM agents"
   reviewer_B: "sentiment analysis, opinion mining, aspect-based SA, social media"
   reviewer_C: "code review automation, static analysis, software engineering, LLM"

3. Embed the paper abstract and each reviewer profile with a pre-trained model:
   from sentence_transformers import SentenceTransformer, util
   model = SentenceTransformer("intfloat/e5-base-v2")
   paper_emb = model.encode("query: " + paper_abstract)
   profile_embs = model.encode(["passage: " + p for p in profiles])
   scores = util.cos_sim(paper_emb, profile_embs)

4. Rank by similarity:

Output:
   Paper "LLM-Based Automated Code Review with Static Analysis Feedback":
     1. reviewer_C (score: 0.82) - code review + static analysis + LLM
     2. reviewer_A (score: 0.71) - code generation + LLM agents
     3. reviewer_B (score: 0.34) - sentiment analysis (poor match)

Note: Fine-tuning becomes valuable when you have 50+ papers and want to beat
the off-the-shelf embedding baseline by learning domain-specific matching.
```

**Example 3: Evaluating with Pairwise Metrics (LR-bench Style)**

```
User: I have ground-truth familiarity scores (1-5) for 100 paper-reviewer pairs.
How do I evaluate my matching system?

Approach:
1. Load ground-truth annotations:
   annotations = [
     {"paper_id": "p1", "reviewer_id": "r1", "score": 4},
     {"paper_id": "p1", "reviewer_id": "r2", "score": 2},
     ...
   ]

2. Paper-centric pairwise evaluation:
   For each paper, enumerate all reviewer pairs where scores differ.
   Check if the model ranks the higher-scored reviewer above the lower-scored one.

   correct, total = 0, 0
   for paper_id in papers:
       reviewers = get_reviewers_for(paper_id)
       for (r_i, r_j) in combinations(reviewers, 2):
           if gt_score[r_i] != gt_score[r_j]:
               total += 1
               model_higher = model_score[r_i] > model_score[r_j]
               gt_higher = gt_score[r_i] > gt_score[r_j]
               if model_higher == gt_higher:
                   correct += 1
   paper_centric_accuracy = correct / total

3. Reviewer-centric pairwise evaluation:
   Same logic, but iterate over reviewers and compare their papers.

4. Also compute NDCG@k for ranked lists.

Output:
   Paper-centric pairwise accuracy: 0.73
   Reviewer-centric pairwise accuracy: 0.69
   NDCG@5 (paper-centric): 0.78

   Interpretation: The model correctly orders 73% of reviewer pairs
   by expertise, consistent with RATE's reported improvements over
   BM25 baselines (~5-8% absolute gain after fine-tuning).
```

## Best Practices

- **Do:** Use recent publications only (last 2-3 years) for profile construction. Research interests shift, and stale keywords degrade matching quality significantly.
- **Do:** Apply margin filtering when constructing weak supervision pairs. Only pair reviewers whose heuristic scores differ by a meaningful gap (e.g., top-20% vs. bottom-20%), not adjacent ranks. This produces cleaner training signal.
- **Do:** Deduplicate and normalize keywords across a reviewer's publications. Merge "LLM," "large language model," and "large language models" into a single canonical form.
- **Do:** Evaluate from both paper-centric and reviewer-centric perspectives. A system can rank reviewers well for popular topics but fail for niche papers, and vice versa.
- **Avoid:** Using full abstracts as reviewer profiles. The whole point of keyword distillation is to remove noise and normalize vocabulary. Full-text profiles dilute the expertise signal.
- **Avoid:** Training on too few triplets. Aim for at least 5 triplets per paper and at least 200 papers in the training corpus for meaningful fine-tuning gains over the baseline.
- **Avoid:** Ignoring the cold-start problem. Reviewers with fewer than 3 recent publications will have sparse profiles. Flag them for manual review or supplement with older publications.

## Error Handling

- **Semantic Scholar API rate limits:** Batch API calls with delays (100ms between requests). Cache responses locally. Fall back to DBLP if Semantic Scholar is unavailable.
- **LLM keyword extraction returns garbage:** Validate extracted keywords against a domain vocabulary or check that at least 50% of keywords appear in the paper's abstract. Re-prompt with stricter instructions if validation fails.
- **Too few weak supervision triplets:** If the reviewer pool is small (<20), the heuristic retrieval step may not produce enough discriminative pairs. In this case, skip fine-tuning and use the off-the-shelf embedding baseline with keyword profiles (still a strong approach).
- **Embedding model produces uniform scores:** This usually means profiles are too similar (over-generic keywords like "deep learning," "NLP"). Increase keyword specificity in the extraction prompt by requesting method names, dataset names, and task-specific terminology.
- **Cosine similarity scores are all low (<0.3):** The paper and profiles may be in mismatched domains. Check that the paper's field aligns with the reviewer pool. Also verify that the embedding model's input format is correct (some models like E5 require "query:" and "passage:" prefixes).

## Limitations

- **Requires recent publications:** Reviewers who are knowledgeable but haven't published recently (e.g., senior researchers, industry practitioners) will be underrepresented. The profile quality is directly tied to publication volume.
- **Keyword profiles lose nuance:** A reviewer who published one paper on "reinforcement learning" and another who has 20 RL papers may have similar keyword overlap. Frequency weighting helps but doesn't fully capture depth of expertise.
- **Weak supervision ceiling:** The fine-tuned model can outperform the heuristic retrieval signal, but its upper bound is still constrained by the quality of that initial signal. If BM25 produces poor initial rankings (e.g., in highly interdisciplinary fields), the training pairs will be noisy.
- **English-centric:** The keyword extraction and embedding pipeline assumes English-language publications. Multilingual venues may need adapted prompts and models.
- **No novelty awareness:** The system matches on topical overlap, not on whether a reviewer can evaluate the novelty of an approach. Two papers on "graph neural networks" may be routine for one reviewer and groundbreaking for another.
- **Five-level familiarity is coarse:** The LR-bench evaluation uses a 1-5 scale, which collapses meaningful distinctions. Pairwise evaluation mitigates this but doesn't eliminate it.

## Reference

**Paper:** Liu, W., Yang, Z., Zhao, Y., & Li, X. (2026). RATE: Reviewer Profiling and Annotation-free Training for Expertise Ranking in Peer Review Systems. arXiv:2601.19637v1. [https://arxiv.org/abs/2601.19637v1](https://arxiv.org/abs/2601.19637v1)

Look for: Section 3 (RATE Framework) for the keyword profile construction algorithm, weak supervision pair generation strategy, and embedding fine-tuning details. Section 4 for LR-bench construction and the five-level familiarity scale. Table 2 for comparison against BM25, SPECTER, and other embedding baselines.

**Dataset:** LR-bench on Hugging Face: [https://huggingface.co/datasets/Gnociew/LR-bench](https://huggingface.co/datasets/Gnociew/LR-bench) -- 1055 paper-reviewer-score annotations with paper-centric and reviewer-centric evaluation splits.