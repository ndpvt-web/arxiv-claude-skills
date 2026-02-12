---
name: "internalizing-reasoning-discovery-replay"
description: "Apply the STIR (Self-Distilled Tools for Internal Reasoning) pattern to build systems that discover reusable reasoning primitives from successful solution traces and dynamically replay them to steer future problem-solving. Use when the user says 'build a reasoning replay system', 'extract reasoning patterns from traces', 'create a latent action library', 'steer reasoning with learned corrections', 'internalize chain-of-thought into activations', or 'dynamic reasoning trajectory control'."
---

# Internalizing Reasoning via Discovery and Replay of Latent Actions

This skill teaches Claude to apply the STIR framework (Self-Distilled Tools for Internal Reasoning) from arXiv:2602.04925 to real software systems. The core idea: instead of relying on verbose chain-of-thought at inference time, you can **mine contrastive correction vectors from successful vs. failed reasoning traces**, build a compact library of reusable steering primitives, and **dynamically inject the right correction at each reasoning step** based on the current state. This translates directly into building retrieval-augmented reasoning pipelines, self-improving agent loops, and adaptive few-shot prompt steering systems.

## When to Use

- When building an agent that should learn from its own successes and failures across episodes (e.g., a coding agent that gets better at debugging over time)
- When the user wants to extract reusable reasoning patterns from a corpus of solved problems and apply them to new problems
- When designing a system that dynamically selects which reasoning strategy to apply at each step of a multi-step task, rather than using one fixed prompt
- When implementing contrastive self-distillation: mining what distinguishes correct solutions from incorrect ones at the representation level
- When reducing token cost by replacing explicit chain-of-thought with compact steering signals
- When the user asks to build a "reasoning memory" or "strategy library" that an LLM can query at inference time

## Key Technique

**The problem with static reasoning aids:** A single fixed prompt, system message, or control vector tries to help with every reasoning step equally. But reasoning is non-stationary -- the guidance that helps during problem decomposition actively hurts during verification. A static approach averages over these conflicting needs and underperforms.

**STIR's solution -- Discover, Compress, Replay:** The framework operates in two phases. **Offline**, it generates multiple solution attempts (rollouts) per problem, scores them with a length-penalized reward, and computes *steering impulses* -- the vector difference between the average hidden state of successful traces and the average hidden state of failed traces at aligned checkpoints. These impulses capture "what the model needs to do differently at this reasoning stage to succeed." The impulses are then compressed into a diverse, non-redundant library of ~256 tools using determinantal point process (DPP) selection, which maximizes both quality and geometric diversity. **Online**, at each reasoning decision point, the system retrieves candidate steering impulses, runs lightweight counterfactual probes to validate their utility, and applies the best one with confidence-scaled strength -- or abstains entirely if the current trajectory is already optimal (anchor gating).

**Why this matters for software:** The three-phase pattern -- (1) contrastive mining of what works vs. what fails, (2) diverse library construction, (3) dynamic retrieval-and-apply with validation -- is a general architecture applicable to any system where you have traces of successful and failed attempts and want to build adaptive, self-improving behavior.

## Step-by-Step Workflow

1. **Collect paired reasoning traces.** For each problem in your training set, generate K rollouts (K=8 is the paper's default). Score each rollout with a reward that combines correctness and brevity: `R(Y) = is_correct(Y) - eta * len(Y) / max_len`. Partition into positive (high-reward) and negative (low-reward) sets.

2. **Align traces at structural checkpoints.** Identify natural breakpoints in reasoning traces (paragraph boundaries, tool calls, intermediate answers). Align positive and negative traces at these checkpoints so you compare equivalent reasoning stages, not arbitrary token positions.

3. **Compute contrastive steering impulses.** At each aligned checkpoint m, compute the centroid embedding of positive traces (mu_plus) and negative traces (mu_minus). The steering impulse is `v_m = mu_plus - mu_minus`. This vector encodes the implicit correction needed to move from a failing reasoning state to a succeeding one.

4. **Store dual-entry memory units.** For each impulse, create two entries: a *correction entry* keyed by the negative centroid (mu_minus) with the impulse as its payload, and an *anchor entry* keyed by the positive centroid (mu_plus) with a null impulse. Anchors let the system detect when no correction is needed.

5. **Build a sparse, diverse tool library.** From all computed impulses, select a compact subset (B=256) using greedy DPP optimization that maximizes `J = sum(log(1 + quality(v))) + lambda * log(det(K + eps*I))`. This ensures the library covers diverse reasoning failure modes without redundancy.

6. **Implement the online Retrieve-Preview-Commit cycle.** At each decision point during inference: (a) **Retrieve** top-k=8 candidate impulses by cosine similarity to the current state embedding. (b) **Preview** each candidate by running a short counterfactual probe (4 tokens) to measure likelihood gain. (c) **Commit** the best candidate if its unified score exceeds the abstention threshold, applying it with clipped, confidence-scaled strength.

7. **Implement anchor gating.** If the retrieved candidates are predominantly anchor entries (null impulse), abstain from intervention -- the reasoning trajectory is already on track. This prevents over-correction and preserves good reasoning chains.

8. **Normalize and index for fast retrieval.** L2-normalize all state keys and store them in a vector index (FAISS, Annoy, or a simple cosine-similarity lookup for small libraries). This makes the online retrieval step sub-millisecond.

9. **Evaluate on held-out problems.** Measure both accuracy improvement and token reduction compared to vanilla chain-of-thought. The paper reports +1.9% to +7.5% accuracy with up to 35% fewer tokens.

10. **Iterate: re-mine impulses on failures.** After deployment, collect new failure traces, compute fresh impulses, and merge them into the library with another round of DPP selection to keep the library compact and current.

## Concrete Examples

**Example 1: Building a self-improving code debugging agent**

```
User: I want my debugging agent to learn from past debugging sessions.
      When it encounters similar bugs, it should apply strategies that
      worked before instead of reasoning from scratch every time.

Approach:
1. Collect paired traces: For 500 past debugging sessions, store both
   the successful fix path and the failed attempts. Each trace is a
   sequence of (hypothesis, investigation, result) tuples.

2. Align at structural checkpoints: Align traces at each
   hypothesis-investigation boundary. Compute embeddings of the agent's
   state (current hypothesis + code context) at each checkpoint.

3. Compute steering impulses: For bug category "off-by-one errors,"
   the impulse at the hypothesis stage might encode "check loop
   boundary conditions" -- the difference between states where the
   agent checked boundaries (success) vs. where it didn't (failure).

4. Build library: Select 50 diverse impulses covering different bug
   categories using DPP selection.

5. Online replay: When the agent encounters a new bug, at each
   reasoning step, retrieve the most relevant impulse and inject it
   as an additional context block:

   retrieved_strategy = library.retrieve(current_state_embedding, k=3)
   probe_scores = [probe(s, current_context) for s in retrieved_strategy]
   if max(probe_scores) > threshold:
       inject(best_strategy, strength=scaled_confidence)
   else:
       proceed_without_intervention()  # anchor gating

Output (library entry example):
{
  "id": "debug-042",
  "trigger_state": "hypothesis_stage:null_pointer_suspicion",
  "impulse_type": "correction",
  "strategy": "Check caller chain for uninitialized optional fields
               before investigating null pointer itself",
  "confidence": 0.82,
  "source_problems": ["BUG-1234", "BUG-1891", "BUG-2003"]
}
```

**Example 2: Adaptive few-shot prompt selection for math reasoning**

```
User: I have a bank of 2000 solved math problems with full
      chain-of-thought solutions. I want to dynamically pick the
      best few-shot examples for each new problem instead of using
      fixed examples.

Approach:
1. Generate rollouts: For each of 200 test problems, try 8 different
   few-shot prompt combinations. Record which led to correct answers
   and which failed.

2. Embed and align: Embed the problem statement + each few-shot
   context using a sentence transformer. Align at the point where
   the model begins its own reasoning.

3. Compute impulses: For each test problem, compute the centroid of
   embeddings from successful few-shot contexts minus the centroid
   from failed contexts. This impulse encodes "what kind of example
   this problem needs."

4. Build retrieval index:
   impulse_library = []
   for problem_type in aligned_checkpoints:
       pos_centroid = mean(embeddings[successful_contexts])
       neg_centroid = mean(embeddings[failed_contexts])
       impulse = pos_centroid - neg_centroid
       impulse_library.append({
           "key": neg_centroid,  # triggers on similar "stuck" states
           "correction": impulse,
           "recommended_examples": successful_contexts[:3]
       })
   library = dppp_select(impulse_library, budget=256)

5. Online serving: For a new problem, embed it, retrieve the closest
   impulse, and use its recommended examples as the few-shot context.
   Run a 4-token probe to confirm the examples help before committing.

Output:
  New problem: "Find all primes p such that p^2 + 2 is also prime"
  Retrieved impulse: "number_theory:parity_argument"
  Recommended examples: [problem_1447, problem_892, problem_2101]
  Probe result: +0.34 log-likelihood gain -> COMMIT
  (vs. static examples which scored +0.08 -> would have been worse)
```

**Example 3: Implementing the contrastive mining pipeline in Python**

```
User: Show me how to implement the core contrastive impulse
      extraction from reasoning traces.

Output:
import numpy as np
from sklearn.cluster import KMeans
from collections import defaultdict

def extract_steering_impulses(traces, reward_fn, eta=0.1):
    """
    traces: list of (problem_id, rollout_embeddings[], reward)
    Returns: list of {key, impulse, anchor_key} dicts
    """
    # Group rollouts by problem
    by_problem = defaultdict(list)
    for pid, embeddings, reward in traces:
        by_problem[pid].append((embeddings, reward))

    impulses = []
    for pid, rollouts in by_problem.items():
        # Partition into positive and negative sets
        median_r = np.median([r for _, r in rollouts])
        pos = [e for e, r in rollouts if r >= median_r]
        neg = [e for e, r in rollouts if r < median_r]

        if not pos or not neg:
            continue

        # Align at structural checkpoints (assume pre-aligned)
        n_checkpoints = min(len(e) for e in pos + neg)
        for m in range(n_checkpoints):
            mu_plus = np.mean([e[m] for e in pos], axis=0)
            mu_minus = np.mean([e[m] for e in neg], axis=0)
            v_m = mu_plus - mu_minus

            # Normalize key vectors
            key = mu_minus / (np.linalg.norm(mu_minus) + 1e-8)
            anchor = mu_plus / (np.linalg.norm(mu_plus) + 1e-8)

            impulses.append({
                "key": key,           # correction entry
                "impulse": v_m,
                "anchor_key": anchor, # anchor entry (null impulse)
                "problem_id": pid,
                "checkpoint": m
            })

    return impulses


def dppp_select(impulses, budget=256, lambda_div=0.5):
    """Select diverse subset via greedy DPP approximation."""
    keys = np.array([imp["impulse"] for imp in impulses])
    norms = np.linalg.norm(keys, axis=1)
    quality = np.log1p(norms)  # quality score

    selected = []
    remaining = list(range(len(impulses)))

    for _ in range(min(budget, len(impulses))):
        best_idx, best_score = -1, -float("inf")
        for i in remaining:
            # Greedy: quality + diversity from selected set
            div = 0
            if selected:
                sel_keys = keys[selected]
                sims = sel_keys @ keys[i] / (norms[selected] * norms[i] + 1e-8)
                div = -lambda_div * np.max(sims)
            score = quality[i] + div
            if score > best_score:
                best_score, best_idx = score, i
        selected.append(best_idx)
        remaining.remove(best_idx)

    return [impulses[i] for i in selected]
```

## Best Practices

- **Do:** Use length-penalized rewards (`R = correctness - eta * length/max_length`) when scoring rollouts. This prevents the library from learning "be verbose" as a strategy, which is the opposite of the internalization goal.
- **Do:** Always store anchor entries alongside correction entries. Without anchors, the system has no way to detect that the current trajectory is already good, leading to destructive over-correction on problems the model can already solve.
- **Do:** Run counterfactual probes before committing a steering impulse. A 4-token lookahead that measures log-likelihood gain costs almost nothing but catches cases where retrieval similarity is misleading.
- **Do:** Re-run DPP selection periodically as you add new impulses. The library should stay compact (256 entries is sufficient for most domains) to keep retrieval fast and avoid redundancy.
- **Avoid:** Using a single global steering vector for all reasoning steps. The paper shows this underperforms by +1.8% vs. STIR's +6.8% because different reasoning stages need different corrections.
- **Avoid:** Skipping the alignment step. Comparing embeddings from different reasoning stages (e.g., problem decomposition vs. verification) produces meaningless impulses. Always align traces at structural checkpoints first.

## Error Handling

- **Too few rollouts per problem:** If K < 4, the positive/negative partition becomes noisy. Fall back to a simpler contrastive method (e.g., correct vs. incorrect without length penalty) or increase K.
- **Degenerate impulses (near-zero norm):** If `||v_m|| < epsilon`, the positive and negative centroids are nearly identical at that checkpoint. Discard these -- they carry no corrective signal.
- **Retrieval returns only anchors:** This means the current state resembles a successful trajectory. Abstain from intervention. This is correct behavior, not an error.
- **Probe scores are all negative:** The retrieved impulses would hurt performance. Abstain and let the model reason on its own. Log these cases for later analysis -- they may indicate a gap in the library.
- **Library drift:** As the problem distribution shifts, old impulses become less relevant. Track hit rates per library entry and prune entries with consistently low retrieval frequency or negative probe scores.

## Limitations

- **Requires paired success/failure traces.** If you only have successful completions (no failures to contrast against), you cannot compute steering impulses. You need at least some variance in outcomes per problem.
- **Embedding quality is critical.** The entire pipeline depends on meaningful embeddings at structural checkpoints. If your embedding model conflates distinct reasoning states, the impulses will be noisy.
- **Does not replace chain-of-thought for novel problem types.** The library can only steer toward patterns it has seen. For genuinely novel reasoning challenges outside the training distribution, explicit chain-of-thought remains necessary.
- **Checkpoint alignment is domain-specific.** The paper uses double-newline boundaries for math reasoning. For code, you might align at function boundaries or tool-call boundaries. Choosing the wrong alignment granularity degrades impulse quality.
- **Scaling beyond 256 library entries shows diminishing returns.** The DPP selection ensures diversity, but a larger library increases retrieval latency without proportional accuracy gains.

## Reference

- **Paper:** [Internalizing LLM Reasoning via Discovery and Replay of Latent Actions](https://arxiv.org/abs/2602.04925v1) (Shi et al., 2026)
- **Key takeaway:** Look at Section 3 for the full offline/online algorithm pseudocode, and Table 1 for the accuracy vs. token-efficiency Pareto improvements across six benchmarks. The retrieve-preview-commit cycle (Section 3.3) is the most directly implementable component.