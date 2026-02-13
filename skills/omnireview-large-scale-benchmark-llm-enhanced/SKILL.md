---
name: "omnireview-large-scale-benchmark-llm-enhanced"
description: "Build reviewer/expert recommendation systems using LLM-generated semantic profiles and Multi-gate Mixture-of-Experts (MMoE) multi-task learning. Applies the OmniReview Pro-MMoE architecture for matching documents to qualified reviewers or experts. Use when: 'recommend reviewers for this paper', 'build an expert matching system', 'find qualified reviewers from a candidate pool', 'rank experts by relevance to a submission', 'build a reviewer recommendation pipeline', 'match papers to domain experts'."
---

# OmniReview: LLM-Enhanced Expert/Reviewer Recommendation

This skill enables Claude to build expert recommendation and reviewer matching systems using the Pro-MMoE (Profiling Scholars with Multi-gate Mixture-of-Experts) architecture from the OmniReview paper. The core technique replaces lossy single-vector embeddings with LLM-generated semantic profiles that preserve fine-grained expertise nuances, then routes these through a task-adaptive MMoE network that simultaneously optimizes for candidate recall, negative filtering, and precision ranking -- three goals that conflict when optimized independently.

## When to Use

- When building a system to recommend peer reviewers for academic paper submissions
- When matching experts to documents, RFPs, grant proposals, or technical review tasks
- When the user needs to rank a candidate pool by domain expertise relevance to a query document
- When implementing multi-task retrieval where recall and precision objectives conflict (e.g., broad retrieval vs. fine-grained ranking)
- When an embedding-only approach loses critical expertise distinctions and the user needs more interpretable matching
- When disambiguating and profiling entities (authors, reviewers, experts) across multiple data sources

## Key Technique

**The Information Bottleneck Problem.** Standard embedding approaches (SPECTER2, SciBERT, BGE-M3) compress an expert's entire publication history into a single dense vector, losing the fine-grained distinctions between subspecialties. A researcher who publishes on both "federated learning privacy" and "graph neural network scalability" gets a blurred average embedding that matches neither query well. Pro-MMoE solves this by having an LLM generate structured semantic profiles -- natural language summaries of each expert's key contributions and expertise areas -- before embedding. This preserves interpretability (you can read why a match was made) and retains nuance that raw vector averaging destroys.

**Multi-gate Mixture-of-Experts for Conflicting Objectives.** Reviewer recommendation requires three distinct capabilities: (1) recalling all plausible reviewers from a large pool, (2) filtering out superficially-similar but unqualified candidates, and (3) ranking the remaining experts with precision. These objectives conflict -- optimizing for broad recall hurts precision ranking, and vice versa. The MMoE architecture addresses this with shared expert networks (typically 3) and task-specific gating networks that dynamically weight which experts contribute to each task. A confidence tower handles recall/filtering via weighted binary cross-entropy with entropy regularization, while a ranking tower uses a hybrid AUC-margin loss for pairwise ordering.

**Two-Stage Training Strategy.** Training proceeds in two phases: epochs 1-50 freeze the ranking tower and train only the confidence (recall/filtering) components; epochs 51-100 freeze confidence and train only ranking. This curriculum prevents the easier recall task from dominating gradients and starving the harder ranking objective.

## Step-by-Step Workflow

1. **Collect and disambiguate candidate data.** Gather expert profiles from multiple sources (publication databases, ORCID, institutional pages). Normalize names, match publications across sources using normalized title comparison, and verify identities by requiring at least one cross-source publication match. Deduplicate aggressively -- false merges are worse than missed merges.

2. **Select representative publications per expert.** For each candidate, extract the 5 most highly-cited and 5 most recently-published papers from their portfolio. This balances established expertise (citations) with current research direction (recency). Store titles, abstracts, and venue metadata.

3. **Generate LLM semantic profiles.** For each selected publication, prompt an LLM to summarize key contributions, methods, and domain in 2-3 sentences. Then aggregate the per-paper summaries into a single expert profile prompt: "Given these publication summaries, describe this researcher's expertise areas, methodological strengths, and domain focus." Store the resulting profile text alongside raw metadata.

4. **Generate the query document profile.** For the submission/document that needs reviewers, generate an analogous LLM summary capturing: core research question, methodology, domain, and required reviewer expertise. Be explicit about what expertise a qualified reviewer needs.

5. **Embed profiles into dense vectors.** Encode both expert profiles and query profiles using a high-quality text embedding model (e.g., a Qwen3-Embedding or similar instruction-tuned embedding model). These vectors become the input features to the MMoE network. Optionally concatenate bibliometric features (h-index, publication count in target venue, co-author overlap).

6. **Build the MMoE architecture.** Implement 3 shared expert networks (feed-forward layers), two gating networks (one for confidence/filtering, one for ranking), and two task-specific tower heads. The confidence tower outputs a scalar confidence score; the ranking tower outputs pairwise preference scores.

7. **Construct training data with three-tier labels.** For each query document, label candidates as: (a) ground-truth reviewers (positive for recall), (b) hard negatives -- researchers in the same broad field but wrong subspecialty (negative for filtering), and (c) ranked positives with preference ordering based on expertise match quality (for the ranking task).

8. **Train with two-stage curriculum.** Phase 1 (first half of epochs): train confidence tower only with weighted binary cross-entropy + entropy regularization, freezing the ranking tower. Phase 2 (second half): freeze confidence, train ranking tower with hybrid AUC-margin loss. Use early stopping on validation MAP.

9. **Inference and recommendation.** For a new query document: generate its LLM profile, embed it, score all candidates through both towers, filter by confidence threshold (reject candidates below 0.5), then rank remaining candidates by the ranking tower output. Return the top-K with their LLM-generated profile summaries as explanations.

10. **Evaluate with hierarchical metrics.** Measure recall via Real Reviewer Confidence (RRC -- mean confidence on ground-truth reviewers), filtering via Unqualified Candidate Confidence (UCC -- mean confidence on hard negatives, lower is better), and ranking via MAP, R-Precision, Reciprocal Rank, NDCG, and Success@5.

## Concrete Examples

**Example 1: Academic Reviewer Recommendation System**

User: "I have a dataset of 10,000 researchers with their publications. Build a system to recommend the top 5 reviewers for a submitted paper on 'federated learning with differential privacy for medical imaging'."

Approach:
1. For each of the 10,000 researchers, select their top 5 cited + 5 recent papers.
2. Generate LLM profiles:
   ```
   Prompt per paper: "Summarize the key contribution, methodology, and domain
   of this paper in 2-3 sentences: [title + abstract]"

   Aggregation prompt: "Based on these publication summaries, describe this
   researcher's primary expertise areas, methodological strengths, and domain
   focus in a structured paragraph: [summaries]"
   ```
3. For the submission, generate a query profile:
   ```
   "This paper proposes a federated learning framework with differential
   privacy guarantees for training medical imaging models across hospitals.
   Required reviewer expertise: federated learning optimization, differential
   privacy mechanisms, medical image analysis, distributed systems security."
   ```
4. Embed all profiles, score through confidence tower (filter to ~200 candidates above threshold), rank via ranking tower, return top 5.

Output:
```
Rank 1: Dr. A. Smith (confidence: 0.94, rank score: 0.91)
  Profile: "Specializes in privacy-preserving federated learning with 12 papers
  on differential privacy in distributed ML. Recent work on medical FL."
  Why: Direct overlap on FL + DP + medical imaging.

Rank 2: Dr. B. Chen (confidence: 0.91, rank score: 0.87)
  Profile: "Expert in federated optimization algorithms with focus on
  healthcare applications. Strong differential privacy background."
  Why: FL optimization + healthcare domain match.

[... top 5 with explanations]
```

**Example 2: Grant Proposal Expert Matching**

User: "I need to match 50 grant proposals to a pool of 500 potential reviewers. Each reviewer should only review proposals in their specific subfield, not just their broad discipline."

Approach:
1. Generate LLM profiles for all 500 reviewers from their CVs/publication lists.
2. Generate query profiles for each of the 50 proposals emphasizing the specific subfield expertise needed.
3. Build the MMoE with emphasis on the filtering task (Task 2) -- the user's core concern is avoiding broad-field-but-wrong-subfield matches.
4. Adjust confidence threshold higher (e.g., 0.7) to aggressively filter superficially-similar candidates.
5. For each proposal, output a ranked shortlist of 5-10 reviewers with profile-based explanations.

Output:
```
Proposal: "Quantum error correction for topological qubits"
  Matched (confidence > 0.7):
    - Dr. X: topological quantum computing, surface codes [0.93]
    - Dr. Y: quantum error correction theory, stabilizer codes [0.88]
  Filtered out (confidence < 0.7):
    - Dr. Z: quantum computing hardware (wrong subfield) [0.42]
    - Dr. W: classical error correction codes (wrong domain) [0.31]
```

**Example 3: Implementing the MMoE Component in PyTorch**

User: "Show me how to implement the core MMoE architecture for a two-task recommendation system."

```python
import torch
import torch.nn as nn

class MMoERecommender(nn.Module):
    def __init__(self, input_dim, expert_dim=256, num_experts=3, num_tasks=2):
        super().__init__()
        # Shared expert networks
        self.experts = nn.ModuleList([
            nn.Sequential(
                nn.Linear(input_dim, expert_dim),
                nn.ReLU(),
                nn.Linear(expert_dim, expert_dim),
                nn.ReLU()
            ) for _ in range(num_experts)
        ])
        # Task-specific gating networks
        self.gates = nn.ModuleList([
            nn.Sequential(
                nn.Linear(input_dim, num_experts),
                nn.Softmax(dim=-1)
            ) for _ in range(num_tasks)
        ])
        # Task-specific towers
        self.confidence_tower = nn.Sequential(
            nn.Linear(expert_dim, 128), nn.ReLU(),
            nn.Linear(128, 1), nn.Sigmoid()
        )
        self.ranking_tower = nn.Sequential(
            nn.Linear(expert_dim, 128), nn.ReLU(),
            nn.Linear(128, 1)
        )

    def forward(self, query_emb, candidate_emb):
        x = torch.cat([query_emb, candidate_emb,
                        query_emb * candidate_emb], dim=-1)
        # Expert outputs: [batch, num_experts, expert_dim]
        expert_outs = torch.stack([e(x) for e in self.experts], dim=1)
        # Task-specific gated mixtures
        gate_conf = self.gates[0](x).unsqueeze(-1)   # [batch, num_experts, 1]
        gate_rank = self.gates[1](x).unsqueeze(-1)
        mixed_conf = (expert_outs * gate_conf).sum(dim=1)  # [batch, expert_dim]
        mixed_rank = (expert_outs * gate_rank).sum(dim=1)
        confidence = self.confidence_tower(mixed_conf)
        rank_score = self.ranking_tower(mixed_rank)
        return confidence.squeeze(-1), rank_score.squeeze(-1)
```

Training loop sketch with two-stage curriculum:
```python
for epoch in range(100):
    for query, candidate, conf_label, rank_label in dataloader:
        conf_pred, rank_pred = model(query, candidate)
        if epoch < 50:  # Phase 1: confidence only
            loss = weighted_bce(conf_pred, conf_label) + entropy_reg(conf_pred)
        else:           # Phase 2: ranking only
            loss = auc_margin_loss(rank_pred, rank_label)
        loss.backward()
        optimizer.step()
```

## Best Practices

- **Do:** Generate LLM profiles from a balanced selection of cited + recent papers. Using only highly-cited papers misses current research pivots; using only recent papers misses established expertise depth.
- **Do:** Include hard negatives (same broad field, wrong subfield) in training data. Without them, the model learns to distinguish computer science from biology but fails to distinguish "NLP for healthcare" from "NLP for machine translation."
- **Do:** Use the two-stage training curriculum. Joint training from epoch 1 consistently underperforms because the recall objective is easier and dominates early gradients, starving the ranking task.
- **Do:** Return the LLM-generated profile text alongside scores. This is a key advantage over pure embedding methods -- stakeholders can verify why a reviewer was recommended.
- **Avoid:** Averaging all of an expert's paper embeddings into a single vector. This is the information bottleneck the paper identifies as the core weakness of prior methods. Use LLM summarization instead.
- **Avoid:** Using more than 3-5 expert networks in the MMoE. The paper's ablation shows 3 experts is optimal; more experts increase parameters without improving performance and risk overfitting on smaller datasets.

## Error Handling

- **Sparse publication records:** When an expert has fewer than 10 publications, use all available papers for profiling rather than the 5+5 selection. Flag these candidates as having lower profile confidence.
- **LLM hallucination in profiles:** Cross-validate generated profiles against actual paper titles and abstracts. If the LLM invents expertise areas not grounded in the source publications, regenerate with stricter grounding prompts (e.g., "Only describe expertise directly evidenced by these papers").
- **Cold-start candidates:** New experts with zero or very few publications cannot produce meaningful LLM profiles. Fall back to metadata matching (affiliation, self-reported keywords) and flag as low-confidence recommendations.
- **Imbalanced training data:** Review datasets are heavily skewed (few positives, many negatives). Use the weighted binary cross-entropy from the paper and oversample hard negatives rather than random negatives.
- **Gate collapse:** If all gating networks converge to the same weights (selecting the same expert mix for both tasks), add a diversity regularization term penalizing low KL-divergence between gate distributions.

## Limitations

- **LLM cost at scale:** Generating semantic profiles for hundreds of thousands of candidates requires significant LLM inference. For pools exceeding ~50K candidates, consider a two-stage pipeline: fast embedding pre-filter to ~1K candidates, then LLM profiling on the shortlist only.
- **Profile staleness:** LLM profiles are static snapshots. Experts who shift research focus need periodic re-profiling. Build in a refresh mechanism triggered by new publications.
- **Domain specificity:** The OmniReview benchmark covers primarily CS/ML venues. The three-tier evaluation framework generalizes, but the specific hard-negative construction strategy (same broad field, wrong subfield) requires domain-specific taxonomy knowledge for other fields.
- **Not suitable for:** Reviewer matching where expertise is less important than other factors (conflict of interest checking, load balancing, diversity requirements). Pro-MMoE optimizes for expertise match, not assignment logistics.
- **Training data requirements:** The MMoE architecture needs labeled review records (who actually reviewed what). In domains without this ground truth, the full pipeline cannot be trained -- though the LLM profiling + embedding components still work as a strong baseline without the MMoE.

## Reference

[OmniReview: A Large-scale Benchmark and LLM-enhanced Framework for Realistic Reviewer Recommendation](https://arxiv.org/abs/2602.08896v1) -- Huang et al., 2026. Focus on Section 4 (Pro-MMoE architecture), Section 3.2 (three-tier evaluation framework), and Figure 3 (MMoE gating mechanism). The ablation studies in Section 5.3 provide guidance on expert count, profile construction, and training schedule choices.