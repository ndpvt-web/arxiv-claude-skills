---
name: "inficoevalchain-blockchain-based-decentralized-fra"
description: "Design and implement decentralized LLM evaluation systems using blockchain-based consensus, multi-node validation, and statistical aggregation to produce reliable model rankings. Use when: 'build a decentralized eval framework', 'reduce variance in LLM benchmarks', 'design a blockchain validator for model scoring', 'create a robust evaluation pipeline with consensus', 'implement Schelling point consensus for scoring', 'aggregate benchmark results across heterogeneous hardware'."
---

# InfiCoEvalChain: Decentralized LLM Evaluation with Blockchain Consensus

This skill enables Claude to design and implement decentralized evaluation frameworks for large language models, based on the InfiCoEvalChain protocol. The core insight: centralized LLM evaluation is statistically unreliable -- the standard deviation across repeated runs on HumanEval (1.67) exceeds the performance gap among the top-10 models on the leaderboard (0.91), making rankings meaningless. This skill teaches the architecture for multi-node, blockchain-verified evaluation with Schelling point consensus, stratified task distribution, and Gaussian-decay reward systems that reduce evaluation variance by over 80%.

## When to Use

- When the user wants to build a system that evaluates LLMs across multiple machines or cloud instances and aggregates results into a trustworthy ranking
- When the user asks how to reduce variance or improve statistical confidence in benchmark results (e.g., HumanEval, GPQA, GSM8K)
- When designing a reward/penalty mechanism for crowdsourced or decentralized computation tasks
- When implementing commit-reveal consensus protocols to prevent collusion in distributed scoring
- When the user needs to partition benchmark datasets across workers with controlled redundancy and difficulty stratification
- When building leaderboard infrastructure that must resist gaming, overfitting, or hardware-specific artifacts

## Key Technique

**The Variance Problem.** Current LLM evaluation runs benchmarks on a single hardware configuration with fixed inference parameters. The resulting scores fluctuate significantly across runs due to GPU non-determinism, floating-point variance, and sampling stochasticity. InfiCoEvalChain addresses this by distributing evaluation across heterogeneous compute nodes (different GPUs, different inference parameter profiles) and statistically aggregating results. Each model is evaluated under ten or more distinct configuration profiles varying hardware (H800, A800, RTX 5090), temperature, top-p, top-k, and repetition penalty. The Central Limit Theorem guarantees that with sufficient independent trials (roughly n >= 28 for 80% confidence), the aggregated mean converges to the true performance.

**Schelling Point Consensus.** To prevent dishonest reporting, the framework uses a game-theoretic consensus mechanism. Evaluators submit scores via a two-phase commit-reveal scheme: first broadcasting `Hash(score || nonce)` without revealing the actual score, then revealing the score for validation. Honest evaluators naturally converge on the true result (the Schelling focal point) because the reward function pays out proportionally to proximity to the median. The reward weight for evaluator i is `w_i = exp(-(s_i - M)^2 / (2 * sigma^2))` where M is the median score and `sigma = k * MAD` (Median Absolute Deviation, k in [1, 1.5]). This Gaussian decay naturally suppresses outliers -- roughly 3% of nodes receive near-zero rewards -- without requiring explicit blacklisting.

**Validator Selection with Diversity.** Participants are selected using a quality score `q_i = r_i / (1 + gamma * t_i)` where r_i is reputation and t_i is task frequency. The gamma penalty prevents monopolization. Final selection applies Maximal Marginal Relevance (MMR): `Score = lambda * q_i - (1 - lambda) * max Sim(u_i, u_j)`, ensuring hardware and configuration diversity across the validator set rather than clustering on identical environments.

## Step-by-Step Workflow

1. **Define the evaluation task and benchmark dataset.** Specify the benchmark (e.g., HumanEval, GPQA-Diamond, GSM8K), the models to evaluate, and the target statistical confidence (e.g., 95% CI width < 1.0 point). Compute the minimum number of independent evaluations needed using `n >= (1 - 0)^2 * ln(2 / delta) / (2 * epsilon^2)` where delta is the failure probability and epsilon is the desired precision.

2. **Partition the dataset using stratified redundant partitioning.** Divide the benchmark into K difficulty strata (e.g., by historical solve rate). Within each stratum, assign samples to evaluator subsets such that each sample appears in exactly rho (redundancy factor, typically 3-5) distinct subsets. Each evaluator receives a representative mix reflecting the global difficulty distribution.

3. **Design the configuration profile matrix.** Create 10+ distinct evaluation profiles varying: GPU type, inference temperature (e.g., 0.0, 0.3, 0.6), top-p (0.9, 0.95, 1.0), top-k (40, 50, disabled), and repetition penalty. Each evaluator is assigned a specific profile, ensuring hardware and parameter diversity across the validator set.

4. **Implement the commit-reveal consensus protocol.** Build two transaction phases: (a) Commit phase -- each evaluator computes `Hash(score || random_nonce)` and publishes the hash to the shared ledger. (b) Reveal phase -- after all commits are collected, evaluators broadcast their raw score and nonce. Verify each reveal against its commitment. Discard any evaluator whose reveal doesn't match.

5. **Implement the validator selection algorithm.** Maintain a reputation registry for each evaluator. Compute quality scores `q_i = r_i / (1 + gamma * t_i)` and apply MMR selection to maximize both quality and diversity. Use explicit features (GPU model, region, framework version) and implicit features (historical score distributions) for the similarity function.

6. **Compute consensus scores using median-anchored aggregation.** For each benchmark problem, collect all validated scores. Compute the median M and MAD. Assign weights `w_i = exp(-(s_i - M)^2 / (2 * (k * MAD)^2))`. The final score for a problem is the weighted mean: `S = sum(w_i * s_i) / sum(w_i)`. Aggregate problem-level scores into the model's overall benchmark score.

7. **Distribute rewards using the Gaussian proximity function.** From a fixed token pool per task (e.g., 100 tokens), allocate rewards as `R_i = R_total * (w_i / sum(w_j))`. This incentivizes honest reporting (scores near the consensus) and naturally penalizes outliers without explicit slashing, maintaining a stable token economy.

8. **Compute confidence intervals and report.** Calculate standard error `SE = sqrt(variance / n)` and 95% CI as `score +/- 1.96 * SE`. Compare CI widths against the performance gap between ranked models. Flag any rankings where CIs overlap, indicating statistically indistinguishable performance.

9. **Update evaluator reputations.** After each round, update reputation scores: increase for evaluators whose scores fell within 1 MAD of the median, decrease for outliers. Apply exponential decay to historical reputation so that recent behavior matters more.

10. **Persist results to the verification ledger.** Record final aggregated scores, individual evaluator contributions (hashed), confidence intervals, and the configuration profile used for each evaluation. This creates an auditable trail that any third party can verify.

## Concrete Examples

**Example 1: Building a Decentralized HumanEval Evaluation Service**

User: "I want to build a system where multiple contributors can run HumanEval on different GPUs and we aggregate results into a reliable leaderboard."

Approach:
1. Design the evaluator registration API -- each node reports its GPU model, VRAM, CUDA version, and framework (vLLM, TGI, etc.)
2. Implement stratified partitioning: split HumanEval's 164 problems into 5 difficulty strata based on historical solve rates, assign each problem to 4 evaluators (rho=4), giving each evaluator ~131 problems
3. Create 10 configuration profiles varying temperature (0.0, 0.2, 0.6), top-p (0.9, 0.95), top-k (40, disabled)
4. Build the commit-reveal flow: evaluators POST hash commitments to `/api/commit`, then POST reveals to `/api/reveal` after deadline
5. Aggregate using median-anchored weighted mean with k=1.2 for the MAD multiplier
6. Publish leaderboard with 95% CIs alongside point estimates

Output:
```
Model: Qwen-3-14B
HumanEval Pass@1: 82.3 +/- 0.35 (95% CI: [81.95, 82.65])
Evaluators: 28 nodes, 10 config profiles
Std Dev: 0.28 (vs 1.67 centralized baseline)
Consensus: 27/28 evaluators within 1 MAD of median
```

**Example 2: Implementing the Reward Mechanism in Python**

User: "Show me how to implement the Gaussian-decay reward distribution from InfiCoEvalChain."

Approach:
1. Collect validated scores from all evaluators
2. Compute median and MAD
3. Apply the Gaussian weight function
4. Distribute a fixed reward pool proportionally

Output:
```python
import numpy as np

def compute_rewards(scores: list[float], reward_pool: float = 100.0, k: float = 1.2) -> list[float]:
    """Distribute rewards using InfiCoEvalChain's Gaussian proximity mechanism.

    Scores closer to the median receive higher rewards. Outliers are naturally
    suppressed without explicit slashing.
    """
    scores_arr = np.array(scores)
    median = np.median(scores_arr)
    mad = np.median(np.abs(scores_arr - median))

    # Prevent division by zero when all scores are identical
    sigma = k * mad if mad > 0 else 1.0

    # Gaussian decay weights: peak at median, decay with distance
    weights = np.exp(-((scores_arr - median) ** 2) / (2 * sigma ** 2))

    # Proportional reward distribution from fixed pool
    rewards = reward_pool * (weights / weights.sum())
    return rewards.tolist()

# Example: 6 evaluators, one outlier
scores = [82.1, 82.5, 82.3, 82.0, 82.4, 75.0]
rewards = compute_rewards(scores)
# Honest nodes (~82): ~17-18 tokens each
# Outlier (75.0): ~0.3 tokens
```

**Example 3: Commit-Reveal Protocol for Score Submission**

User: "How do I prevent evaluators from copying each other's scores in a decentralized eval system?"

Approach:
1. Implement two-phase commit-reveal using cryptographic hashing
2. Enforce a commitment deadline before any reveals are accepted
3. Validate reveals against stored commitments

Output:
```python
import hashlib
import secrets
from dataclasses import dataclass

@dataclass
class Commitment:
    evaluator_id: str
    hash_value: str
    revealed: bool = False
    score: float | None = None

class CommitRevealProtocol:
    def __init__(self):
        self.commitments: dict[str, Commitment] = {}
        self.phase: str = "commit"  # "commit" | "reveal" | "finalized"

    def commit(self, evaluator_id: str, score: float) -> tuple[str, str]:
        """Evaluator commits hash(score || nonce) without revealing score."""
        if self.phase != "commit":
            raise ValueError("Commit phase has ended")
        nonce = secrets.token_hex(16)
        payload = f"{score:.6f}||{nonce}"
        hash_value = hashlib.sha256(payload.encode()).hexdigest()
        self.commitments[evaluator_id] = Commitment(evaluator_id, hash_value)
        return hash_value, nonce  # nonce kept secret by evaluator

    def close_commits(self):
        """Transition to reveal phase -- no more commits accepted."""
        self.phase = "reveal"

    def reveal(self, evaluator_id: str, score: float, nonce: str) -> bool:
        """Evaluator reveals score and nonce; verified against commitment."""
        if self.phase != "reveal":
            raise ValueError("Not in reveal phase")
        commitment = self.commitments.get(evaluator_id)
        if not commitment:
            raise ValueError("No commitment found")
        payload = f"{score:.6f}||{nonce}"
        expected = hashlib.sha256(payload.encode()).hexdigest()
        if expected != commitment.hash_value:
            return False  # Mismatch -- evaluator is dishonest or made an error
        commitment.revealed = True
        commitment.score = score
        return True

    def get_valid_scores(self) -> list[float]:
        """Return only scores from evaluators who revealed correctly."""
        return [c.score for c in self.commitments.values()
                if c.revealed and c.score is not None]
```

## Best Practices

- **Do** require a minimum of 28 independent evaluations for 80% statistical confidence, and scale up for tighter bounds. The framework's power comes from volume.
- **Do** use the median (not the mean) as the consensus anchor, because median is robust to the ~3% of outlier/dishonest nodes that every open system attracts.
- **Do** enforce hardware and parameter diversity via MMR-based validator selection. Evaluating on 28 identical A100s is worse than evaluating on 10 H800s + 10 A800s + 8 RTX 5090s.
- **Do** use a fixed reward pool per task rather than minting new tokens per evaluation, preventing inflation and anchoring economic incentives.
- **Avoid** using mean-based aggregation without weighting -- a single dishonest node reporting 0% can dramatically skew results.
- **Avoid** allowing evaluators to see others' scores before submitting their own. The commit-reveal protocol exists precisely to prevent herding behavior.

## Error Handling

- **Hash mismatch on reveal**: Discard the evaluator's score for this round and decrement their reputation. Do not slash rewards from previous rounds -- the mismatch may be a software bug rather than malice.
- **Insufficient evaluators**: If fewer than the minimum n evaluators participate, widen the CI accordingly and flag the result as low-confidence rather than publishing a misleading point estimate.
- **MAD equals zero**: When all evaluators report identical scores (MAD = 0), set sigma to a small default (e.g., 1.0) and distribute rewards equally. This edge case is actually the ideal outcome.
- **Commitment deadline missed**: Evaluators who don't commit in time are excluded from the round. Do not extend deadlines, as this creates opportunities for strategic timing attacks.
- **Sybil attacks**: A single actor running many evaluator nodes on identical hardware. The MMR diversity constraint mitigates this by penalizing similar configurations, but monitor for suspiciously correlated score distributions.

## Limitations

- **Not useful for single-run evaluations.** The framework's value comes from statistical aggregation over many nodes. If you only have one machine, use standard evaluation with explicit confidence intervals instead.
- **Requires economic incentive design.** The reward mechanism assumes participants are economically rational. In volunteer or academic settings without real token incentives, the game-theoretic guarantees weaken.
- **Adds latency.** The commit-reveal protocol requires two network round-trips plus a commitment deadline. Real-time evaluation or rapid iteration loops are better served by centralized pipelines.
- **Benchmark-specific.** The statistical framework assumes evaluation produces a numeric score. Qualitative evaluations (e.g., human preference rankings, open-ended generation quality) require adapted consensus mechanisms.
- **Does not fix benchmark quality.** Decentralized evaluation reduces measurement noise but cannot compensate for poorly designed benchmarks. If HumanEval problems are ambiguous, consensus on wrong answers is still wrong.

## Reference

**Paper:** [InfiCoEvalChain: A Blockchain-Based Decentralized Framework for Collaborative LLM Evaluation](https://arxiv.org/abs/2602.08229v1) (Yang et al., 2026). Key sections: Section 3 for the Schelling point consensus and commit-reveal protocol, Section 4 for the stratified redundant partitioning algorithm, and Section 5 for experimental validation showing std dev reduction from 1.67 to 0.28.