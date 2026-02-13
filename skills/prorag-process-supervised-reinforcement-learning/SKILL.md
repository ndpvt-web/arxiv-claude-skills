---
name: prorag-process-supervised-reinforcement-learning
description: >
  Build process-supervised RAG pipelines that use step-level reward signals to
  eliminate flawed reasoning chains. Implements the ProRAG four-stage framework:
  policy warmup, MCTS-based process reward modeling, rejection-sampling refinement,
  and dual-granularity RL. Use this skill when asked to:
  "build a multi-hop RAG pipeline with step-level verification",
  "add process supervision to my retrieval-augmented agent",
  "implement MCTS-based reward modeling for reasoning chains",
  "reduce hallucination in my RAG search agent",
  "create a retrieval agent with step-by-step quality scoring",
  "train a process reward model to evaluate reasoning steps".
---

# ProRAG: Process-Supervised Reinforcement Learning for RAG

This skill enables Claude to design, implement, and debug **process-supervised RAG systems** that score every intermediate reasoning step -- not just the final answer. The core insight from ProRAG is that outcome-only rewards cause "process hallucinations" where a model reaches the right answer via wrong logic or redundant retrievals. By combining an MCTS-trained Process Reward Model (PRM) with dual-granularity advantages (step-level + outcome-level), you get dense credit assignment that pinpoints exactly which search/think/reflect step went wrong.

## When to Use

- When building a multi-hop question-answering agent that chains multiple retrieval calls and you need to verify each hop independently
- When an existing RAG pipeline produces correct final answers but via flawed or redundant reasoning steps (process hallucination)
- When training or fine-tuning an LLM-based search agent with reinforcement learning and outcome-only rewards produce unstable training
- When implementing an agentic search loop with `think -> search -> reflect -> answer` structure and you need per-step quality signals
- When you want to build a Process Reward Model that scores intermediate reasoning quality rather than just final correctness
- When debugging why a retrieval agent issues redundant or off-topic searches in long reasoning chains

## Key Technique

**The Problem**: Standard RL for RAG uses a single scalar reward (did the final answer match?). This is sparse and ambiguous -- if a 6-step reasoning chain fails, the model cannot tell which step caused the failure. Worse, when the chain succeeds by luck, every step gets reinforced equally, including flawed logic. This creates "process hallucinations."

**ProRAG's Solution**: A four-stage pipeline that builds dense step-level supervision into the RL loop. First, supervised fine-tuning teaches the model a structured action format with explicit control tokens (`<step>`, `<subquery>`, `<retrieval>`, `<subanswer>`, `<answer>`). Second, Monte Carlo Tree Search explores the space of possible reasoning steps and uses rollout-based value estimation to construct contrastive pairs of good vs. bad steps at each decision point. A PRM is trained on these pairs using pairwise ranking loss. Third, rejection sampling filters training trajectories through both outcome correctness and PRM validity, creating a curated dataset for intermediate fine-tuning. Fourth, RL training uses a dual-granularity advantage: `A = A_outcome + beta * A_process`, where the process advantage normalizes PRM scores group-wise and the outcome advantage normalizes final correctness, fused with a mixing coefficient beta (typically 0.3).

**Why It Works**: The MCTS exploration decouples step quality from trajectory outcome -- a good step in a failed trajectory still gets credit, and a bad step in a successful trajectory gets penalized. The dual-granularity advantage gives every token dense supervision while still respecting the global correctness signal.

## Step-by-Step Workflow

1. **Define the structured action format.** Create explicit control tokens for your reasoning loop. At minimum: `<step>` (internal planning/thinking), `<subquery>` (formulate a retrieval query), `<retrieval>` (slot for retrieved documents), `<subanswer>` (intermediate conclusion from evidence), and `<answer>` (final output). Weight the loss on control tokens higher (lambda > 1) during SFT to enforce format adherence.

2. **Collect SFT warmup data.** For each training query, generate gold reasoning chains that follow the structured format. Use an instruction-tuned model to produce step-by-step decompositions of multi-hop questions, inserting actual retrieval calls. Filter to keep only chains where the final answer is correct. Fine-tune your base model on this data (ProRAG uses ~109k examples on Qwen3-8B).

3. **Build the MCTS tree for process reward data.** For a seed set of queries (~700-1000), run MCTS with the SFT policy as the prior. At each node, sample K candidate next-steps at high temperature. For each candidate, complete the trajectory greedily and evaluate against ground truth (binary: correct/incorrect). Update node values via: `Q(s,a) <- [Q(s,a)*N(s,a) + gamma^(T-t)*v] / [N(s,a)+1]`, where gamma (0.99) discounts longer paths. Run ~200 simulations per query with exploration constant c_puct=2.5.

4. **Construct contrastive pairs from sibling nodes.** For each parent node in the MCTS tree, extract sibling actions (same parent, different next step). Pair the higher-valued sibling as "chosen" and the lower-valued as "rejected." Use an LLM judge (GPT-4o or similar) to validate that the chosen step is genuinely more logical. This produces ~8k high-quality contrastive pairs.

5. **Train the Process Reward Model.** Initialize from the SFT checkpoint. Train with pairwise ranking loss to maximize the score margin between chosen and rejected steps. The PRM takes a partial trajectory and outputs a scalar score for the most recent step. Validate that the PRM's rankings correlate with actual downstream correctness.

6. **Run PRM-guided rejection sampling refinement.** Generate N candidate trajectories per query from the SFT policy. Apply dual-criterion filtering: (a) the final answer must be correct, and (b) all intermediate steps must exceed the PRM validity threshold. Fine-tune on the filtered high-quality segments. This stage bridges the gap between SFT and RL, preventing cold-start instability.

7. **Implement dual-granularity RL training.** For each training query, sample G trajectories (group size, typically 8). Compute per-step process advantage: `A_proc = (r_step - mean_step) / std_step` where r_step is the PRM score. Compute per-trajectory outcome advantage: `A_out = (r_out - mean_out) / std_out` where r_out is binary final correctness. Combine: `A = A_out + beta * A_proc` with beta=0.3. Optimize using GRPO-style clipped policy gradient: `L = -E[min(rho*A, clip(rho, 1-eps, 1+eps)*A)]`.

8. **Evaluate on multi-hop benchmarks.** Test on datasets with varying hop counts (HotpotQA for 2-hop, MuSiQue for 3-4 hop, 2WikiMultiHopQA for compositional). Measure both EM/F1 (outcome quality) and step-level metrics: retrieval precision (fraction of searches that return relevant docs), reasoning validity (PRM score distribution), and path efficiency (average steps to correct answer).

9. **Diagnose process hallucinations.** Compare the PRM score distribution of correct-answer trajectories before and after process supervision. A healthy distribution should show high PRM scores clustering near 1.0. If correct answers still have low PRM steps, the process reward is underweighted (increase beta). If training is unstable, reduce the RL learning rate or increase the rejection sampling filtering threshold.

## Concrete Examples

**Example 1: Building a multi-hop QA agent with step-level verification**

User: "I have a RAG pipeline that answers multi-hop questions but it often retrieves irrelevant documents in intermediate steps. How do I add process supervision?"

Approach:
1. Restructure the agent loop to emit explicit control tokens at each step:
```python
REASONING_FORMAT = """
<step>Decompose the question into sub-questions</step>
<subquery>first sub-question keywords</subquery>
<retrieval>{retrieved_docs}</retrieval>
<subanswer>intermediate answer from evidence</subanswer>
<step>Identify what remains unanswered</step>
<subquery>second sub-question keywords</subquery>
<retrieval>{retrieved_docs}</retrieval>
<subanswer>second intermediate answer</subanswer>
<answer>final synthesized answer</answer>
"""
```
2. Collect 500-1000 seed queries. For each, run MCTS with the current policy: at each `<subquery>` decision point, sample 5 alternative queries, complete each trajectory, and score by final correctness. Build contrastive pairs from sibling subqueries.
3. Train a PRM on the contrastive pairs. At inference time, after each step, score it with the PRM. If the score falls below threshold (e.g., 0.3), backtrack and resample that step.
4. For RL training, compute dual advantages. The process advantage penalizes low-PRM steps even in correct trajectories, and the outcome advantage ensures final answer quality is maintained.

Output: An agent that self-corrects mid-chain. When a subquery scores low on the PRM, it reformulates before proceeding, reducing redundant retrievals by ~30%.

**Example 2: Training a Process Reward Model from scratch**

User: "I want to build a reward model that scores individual reasoning steps, not just final answers."

Approach:
1. Start with your SFT-trained reasoning model as the MCTS prior policy.
2. Define the MCTS parameters:
```python
mcts_config = {
    "num_simulations": 200,
    "candidates_per_node": 5,    # K: expansions per leaf
    "max_depth": 10,              # maximum reasoning steps
    "c_puct": 2.5,                # exploration constant
    "gamma": 0.99,                # discount for path length
    "temperature": 1.0,           # sampling temperature for expansion
}
```
3. For each seed query, build the search tree. At each leaf node, sample K candidate next-steps at high temperature. For each candidate, greedily complete the remaining trajectory and evaluate (binary correctness against gold answer).
4. Extract sibling pairs: for every parent node with 2+ children, pair the child with higher Q-value (chosen) against the lower (rejected). Validate with an LLM judge to filter noise.
5. Train PRM with pairwise ranking loss:
```python
loss = -log(sigmoid(prm_score(chosen) - prm_score(rejected)))
```
6. Validate: on held-out trajectories, check that PRM scores correlate with actual step quality (steps in correct chains score higher than steps in incorrect chains).

Output: A PRM that takes a partial reasoning chain and returns a scalar quality score for the latest step. Use it for: rejection sampling, beam search reranking, or as a dense reward signal in RL.

**Example 3: Debugging process hallucination in an existing agent**

User: "My RAG agent gets 72% accuracy on HotpotQA but I suspect it's reasoning incorrectly even on correct answers. How do I detect and fix this?"

Approach:
1. Sample 200 correct-answer trajectories. For each, manually or with an LLM judge score each intermediate step as valid/invalid.
2. Compute the "process hallucination rate": fraction of correct trajectories containing at least one invalid step. If this exceeds 20-30%, process supervision is warranted.
3. Use the PRM scoring approach: train a lightweight PRM (even LoRA on your existing model) on ~5k contrastive pairs. Score all steps in the 200 trajectories.
4. Identify the most common failure mode:
   - **Redundant retrieval**: multiple searches for the same information (PRM scores drop on repeated subqueries)
   - **Logic shortcut**: skipping a reasoning step and jumping to the answer (PRM scores a missing `<step>` gap)
   - **Irrelevant retrieval**: subquery doesn't relate to the question decomposition (low PRM at `<subquery>` tokens)
5. Apply targeted fixes: for redundant retrieval, increase the gamma discount in your reward; for logic shortcuts, increase beta to weight process advantage higher; for irrelevant retrieval, add retrieval-precision as an auxiliary reward term.

Output: A diagnostic report showing per-step PRM scores across trajectories, with identified failure modes and specific hyperparameter adjustments to fix them.

## Best Practices

- **Do** weight control token loss higher (1.5-2x) during SFT. Format adherence is critical -- if the model drops `<step>` or `<subanswer>` tokens, the PRM cannot score steps properly.
- **Do** use a discount factor (gamma=0.99) in MCTS value backup. Without it, the model has no incentive to find efficient (shorter) reasoning paths, leading to verbose chains.
- **Do** start with beta=0.3 for the process-outcome mixing coefficient, then tune. Higher beta (0.5+) prioritizes reasoning quality over answer correctness; lower beta (0.1) keeps the focus on outcomes.
- **Do** run the rejection sampling refinement stage before RL. Skipping it causes cold-start instability because the initial policy is too far from the PRM's preference distribution.
- **Avoid** using only PRM scores as reward (dropping outcome reward). Process-only supervision causes the model to produce "beautifully reasoned" but wrong answers. The dual signal is essential.
- **Avoid** running MCTS with fewer than 100 simulations per query. Insufficient exploration produces noisy value estimates that corrupt the contrastive pairs and degrade PRM quality.
- **Avoid** training the PRM on auto-labeled pairs without LLM judge verification. Sibling nodes with similar Q-values produce ambiguous pairs that inject noise. Filter pairs where the Q-value gap is below a threshold.

## Error Handling

- **PRM scores plateau near 0.5 for all steps**: The contrastive pairs lack diversity. Increase MCTS candidates per node (K) or seed query diversity. Check that the chosen/rejected pairs have sufficient Q-value gaps.
- **RL training collapses (all trajectories identical)**: The KL penalty is too low or missing. Add a KL divergence term against the reference policy, or reduce the learning rate. Verify the group sampling produces diverse trajectories (G >= 8).
- **Model drops structured format during RL**: The format-adherence loss weight was too low during SFT, or RL has overwritten it. Add a format reward component (binary: did the trajectory follow the control token structure?) or freeze the token embeddings for control tokens.
- **Correct answers decrease after process supervision**: Beta is too high -- the model over-optimizes for process quality at the expense of outcome. Reduce beta incrementally (0.3 -> 0.2 -> 0.1) until outcome metrics recover.
- **MCTS is prohibitively slow**: Reduce simulations to 100 (minimum viable), reduce max depth to 6, or use a smaller draft model for rollout completion instead of the full policy.

## Limitations

- **Requires substantial compute for MCTS data generation**: Building the process reward training data needs hundreds of MCTS rollouts per query, each requiring multiple LLM forward passes. Budget 4x A100-80GB or equivalent for the Qwen-8B scale.
- **PRM quality bottleneck**: The entire framework depends on the PRM accurately scoring step quality. If your domain lacks clear step-level correctness criteria (e.g., creative writing vs. factual QA), the MCTS-based approach degrades.
- **Limited to structured reasoning formats**: The step-level scoring requires explicit boundaries between reasoning steps. Free-form chain-of-thought without control tokens cannot be scored at step granularity.
- **Single-hop questions see minimal benefit**: The overhead of process supervision is only justified for multi-hop (2+ retrieval steps) tasks. For single-hop factual lookup, standard RAG with outcome reward is sufficient.
- **LLM judge dependency**: Contrastive pair validation relies on GPT-4o or equivalent. For fully self-contained training, you need an alternative quality signal (e.g., retrieval relevance metrics + logical consistency checks).

## Reference

**Paper**: [ProRAG: Process-Supervised Reinforcement Learning for Retrieval-Augmented Generation](https://arxiv.org/abs/2601.21912v1) (Wang, Zhao, Dou, 2026). Look for: Section 3 for the four-stage framework details, Section 3.2 for the MCTS value backup formula and PUCT selection, Section 3.4 for the dual-granularity advantage equations, and Table 1 for benchmark results showing 2.2% EM improvement over the strongest baseline.

**Code**: [github.com/lilinwz/ProRAG](https://github.com/lilinwz/ProRAG)