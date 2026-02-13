---
name: "internalizing-multi-agent-reasoning-accurate"
description: "Distill multi-agent reasoning into a single efficient model for recommendation or retrieval. Use when: 'build a recommendation system with LLM reasoning', 'distill multi-agent logic into one model', 'convert collaborative filtering signals to natural language', 'create a trajectory-driven training pipeline', 'compress agentic tool-use into a single pass', 'internalize plan-execute-reflect into a compact model'."
---

# Internalizing Multi-Agent Reasoning for Efficient LLM-based Systems

This skill teaches Claude to apply the STAR (Single-agent Trajectory-Aligned Recommender) framework from Wu et al. (2026). The core technique is a three-stage pipeline: (1) build a multi-agent teacher system with planning, specialized execution, and reflection agents that use collaborative signal tools, (2) serialize the teacher's multi-turn reasoning trajectories into a linear chain-of-thought format, and (3) distill these trajectories into a compact single model via supervised fine-tuning followed by group relative policy optimization. This eliminates iterative multi-agent latency while preserving (and often exceeding) multi-agent accuracy.

## When to Use

- When the user wants to build a recommendation or retrieval system that combines LLM reasoning with collaborative filtering signals (user-item interaction graphs)
- When the user needs to compress a multi-agent workflow (planner, executors, reflector) into a single-model inference pass for production deployment
- When the user asks to translate graph-based collaborative signals (user-CF, item-CF) into natural language evidence that an LLM can reason over
- When the user wants to design a trajectory-driven distillation pipeline that preserves tool-use and self-reflection capabilities
- When the user is building a cold-start or evolving-interest recommendation system that needs semantic reasoning beyond pattern matching
- When the user asks to reduce inference latency of an agentic system by 5x+ while maintaining or improving accuracy

## Key Technique

**Multi-Agent Teacher with Collaborative Signal Translation.** The teacher system (MARS) uses a Plan-Execute-Reflect architecture. A Planner decomposes the recommendation task into subtasks dispatched to specialized agents: User Profile, Historical Interest, Recent Interest, and Interest Divergence agents. Each agent invokes collaborative signal tools -- Item-CF (item -> user -> item traversal) and User-CF (user -> item -> user traversal) -- that translate graph-based co-occurrence patterns into natural language summaries. For example, Item-CF takes an anchor item, finds users who interacted with it, retrieves their other interactions, and produces text like "Users who read The Three-Body Problem also enjoyed Dune and Foundation." These translations are pre-computed offline and stored as static metadata, decoupling expensive verbalization from online inference. A Reflector agent then verifies outputs for consistency before a Ranking agent produces the final list.

**Trajectory-Driven Distillation.** The multi-agent dialogue logs are serialized into linear chain-of-thought sequences using structured tokens: `<plan>`, `<tool_call>`, `<tool_response>`, `<reflection>`, `<recommend>`. Only trajectories where the ground-truth item appears in the top-1 position are retained for training. The student model (STAR) is trained in two stages: (1) Supervised Fine-Tuning (SFT) on filtered trajectories teaches task decomposition, tool invocation syntax, and structured output format; (2) Group Relative Policy Optimization (GRPO) samples multiple outputs per input and optimizes a composite reward combining format adherence (valid trajectory structure and tool syntax) with a tiered outcome score (graduated by rank position of the ground-truth item). This produces a model that performs planning, tool-calling, and self-reflection in a single forward pass.

**Why the student outperforms the teacher.** The distilled model avoids error propagation across agent boundaries and benefits from exposure to many diverse reasoning trajectories during training, effectively learning a more robust policy than any single multi-agent run produces. Experiments show 8.7% to 39.5% improvement over the teacher across classic, cold-start, and evolving-interest scenarios.

## Step-by-Step Workflow

1. **Model the interaction graph.** Construct a user-item bipartite graph from interaction logs. Each node is a user or item; edges represent interactions (purchases, ratings, clicks). Store this as an adjacency list or sparse matrix for efficient traversal.

2. **Pre-compute collaborative signal translations.** For each item, run Item-CF traversal (item -> neighboring users -> their other items) and generate a natural language summary of co-occurrence patterns. For each user, run User-CF traversal (user -> interacted items -> other users of those items) and summarize similar users' preferences. Store these as a key-value lookup (item_id -> NL evidence, user_id -> NL evidence).

3. **Build the multi-agent teacher system.** Implement four components:
   - **Planner**: Takes user history and candidate items; decomposes into subtasks (profile extraction, historical interest, recent interest, divergence exploration).
   - **Executor agents**: Each handles one subtask and can invoke the pre-computed CF tools via function calls that return the NL evidence.
   - **Reflector**: Reviews all agent outputs for consistency and completeness; flags contradictions or gaps.
   - **Ranker**: Aggregates evidence and produces a ranked recommendation list with justification.

4. **Handle long user histories.** Implement a sliding-window iterative summarizer: compress distant interactions into summary paragraphs while preserving recent interactions in raw form. This ensures the context window is not exceeded while retaining both long-term patterns and short-term shifts.

5. **Generate and serialize trajectories.** Run the teacher system on the training set. Log all inter-agent communication. Serialize each multi-agent session into a linear sequence using structured delimiters:
   ```
   <plan>Decompose into profile, historical, recent, divergence subtasks</plan>
   <tool_call>item_cf("The Three-Body Problem")</tool_call>
   <tool_response>Users who read this also enjoyed: Dune, Foundation, ...</tool_response>
   <reflection>Historical and recent interests align on hard sci-fi; divergence suggests exploring adjacent genres</reflection>
   <recommend>1. Dune 2. Foundation 3. Hyperion ...</recommend>
   ```

6. **Filter trajectories for quality.** Retain only trajectories where the ground-truth next item appears at rank 1. This ensures the student learns from successful reasoning chains only. For GRPO training, stratify remaining samples by difficulty (easy/medium/hard based on popularity or history length) in roughly 3:4:3 ratio.

7. **Fine-tune the student model (SFT stage).** Train a base LLM (e.g., Qwen3-8B or similar) on the filtered trajectories using standard causal language modeling. Key hyperparameters: learning rate ~2e-5, batch size 16, max sequence length 16384 tokens, 3 epochs. The model learns task decomposition logic, tool invocation syntax, and structured output format.

8. **Optimize with GRPO.** Sample G=8 candidate outputs per input. Score each with a composite reward:
   - **Format reward**: +1 if output contains all required phases (`<plan>`, `<tool_call>`, `<reflection>`, `<recommend>`) with valid syntax; -1 otherwise.
   - **Outcome reward**: 1.0 if ground-truth at rank 1; 2/3 for top-3; 1/3 for top-5; 0 otherwise.
   Use actor learning rate ~1e-6, KL coefficient 0.001, rollout temperature 0.7.

9. **Deploy as single-pass inference.** At serving time, the STAR model receives user history + candidates and generates the full reasoning trajectory (plan, tool calls, reflection, recommendation) in one autoregressive pass. Tool calls are resolved against the pre-computed CF evidence store. No multi-agent orchestration is needed.

10. **Evaluate across scenarios.** Test on three settings: classic recommendation (full history), cold-start (sparse user or item history), and evolving-interest (temporal preference shifts). Use Hit Rate@{1,3,5} averaged as the primary metric.

## Concrete Examples

**Example 1: Building a book recommendation system**
```
User: "I have a Goodreads-like dataset with user ratings. Build me a recommendation
system that uses LLM reasoning but is fast enough for production."

Approach:
1. Parse the ratings CSV into a user-item interaction graph (adjacency list).
2. Pre-compute Item-CF and User-CF translations:
   - For each book, find top-20 co-read books and generate:
     "Readers of '{title}' frequently also enjoyed: {list}. Common themes: {themes}."
   - For each user, find top-10 similar users and generate:
     "Users with similar taste prefer: {genres}, particularly {specific_titles}."
3. Store translations in a JSON/SQLite lookup keyed by book_id and user_id.
4. Build the teacher pipeline as a Python script using an LLM API:
   - Planner prompt: "Given this user's history, decompose the recommendation task."
   - Executor prompts: Each subtask agent gets history + CF tool access.
   - Reflector prompt: "Check these analyses for contradictions."
   - Ranker prompt: "Aggregate and rank candidates with justification."
5. Run teacher on 10K training examples, serialize trajectories, filter top-1 matches.
6. Fine-tune Qwen3-8B on ~8K filtered trajectories (SFT), then GRPO on 500 samples.
7. Deploy: single model call with user history -> full reasoning + ranked list.

Output:
- Inference: ~89s per batch vs. ~461s for the multi-agent teacher
- Accuracy: HR@1 ~52%, HR@5 ~88% (surpassing multi-agent teacher by ~15%)
```

**Example 2: Cold-start product recommendations**
```
User: "New users on our e-commerce platform get bad recommendations because we have
almost no interaction data for them. Can we use LLM reasoning to help?"

Approach:
1. For new users (< 5 interactions), the collaborative signals are sparse.
   Pre-compute Item-CF translations for all catalog items -- these remain useful
   even when user-level signals are weak.
2. Design the teacher's Interest Divergence agent to focus on semantic expansion:
   given a user's few interactions, infer broader category interests and explore
   adjacent product categories using item descriptions and CF evidence.
3. In the Reflector stage, add a specific check: "Is there sufficient evidence
   to support each recommendation, or is the agent hallucinating preferences?"
4. During trajectory filtering, include cold-start examples specifically --
   stratify the GRPO training set so 30% of samples are cold-start users.
5. The distilled model learns to lean on item-level CF evidence and semantic
   reasoning when user-level signals are absent.

Output:
- Cold-start user HR@5 improves from ~49% (baseline) to ~76% (STAR)
- The model learns to compensate for sparse user history by leveraging
  item-to-item relationships and semantic category reasoning.
```

**Example 3: Distilling any multi-agent workflow into a single model**
```
User: "I have a multi-agent code review system (planner, security checker,
style checker, summarizer). It's accurate but too slow. Can I compress it?"

Approach:
1. Adapt the STAR framework to code review:
   - Map agents: Planner -> task decomposition, Security/Style -> specialized executors,
     Summarizer -> ranker equivalent.
   - Define tools: lint results, CVE database lookups, style guide references.
2. Pre-compute tool responses where possible (e.g., lint and static analysis
   results for the codebase can be cached).
3. Run the multi-agent system on 5K code review examples. Serialize trajectories:
   <plan>Check security, then style, then summarize findings</plan>
   <tool_call>run_linter("src/auth.py")</tool_call>
   <tool_response>3 warnings: unused import, broad except, hardcoded secret</tool_response>
   <reflection>Security issue (hardcoded secret) is critical; style issues are minor</reflection>
   <recommend>Priority: 1. Remove hardcoded secret 2. Narrow except clause ...</recommend>
4. Filter to trajectories matching expert-labeled review outcomes.
5. SFT + GRPO to train a single model that produces structured reviews in one pass.

Output:
- Single model generates plan + tool calls + reflection + review in one pass
- Latency drops from ~45s (multi-agent) to ~8s (distilled)
- Review quality matches or exceeds the multi-agent system
```

## Best Practices

- **Do** pre-compute collaborative signal translations offline and store them as static metadata. This decouples the expensive graph traversal and LLM verbalization from inference time.
- **Do** retain `<tool_call>` tokens in serialized trajectories even though the student resolves them via lookup. This teaches the model *when* external evidence is needed, which is critical for reasoning quality.
- **Do** use tiered outcome rewards in GRPO rather than binary correct/incorrect. Graduated scoring (1.0/0.67/0.33/0) provides a smoother optimization signal.
- **Do** stratify GRPO training samples by difficulty (easy:medium:hard ~ 3:4:3). This prevents the policy from collapsing to only handling easy cases.
- **Avoid** skipping the Reflector agent traces during distillation. Ablation shows removing reflection causes the largest accuracy drop (~6.5 points). Self-correction reasoning paths are the most valuable component to internalize.
- **Avoid** training only on SFT without GRPO. SFT alone teaches format imitation but not robust reasoning; GRPO is essential for the student to exceed the teacher.

## Error Handling

- **Sparse collaborative signals**: When a user or item has very few interactions, CF tool responses will be thin or empty. Handle this by having the teacher's Interest Divergence agent fall back to semantic similarity (item descriptions, categories) and ensure these fallback trajectories are included in training data.
- **Trajectory filtering yields too few samples**: If the top-1 filtering is too aggressive, relax to top-3 matching for the SFT stage but keep top-1 filtering for GRPO. Quality matters more for reinforcement learning.
- **Format reward dominance**: If GRPO training collapses to always producing well-formatted but inaccurate outputs, increase the weight of the outcome reward relative to the format reward, or increase rollout temperature to encourage exploration.
- **Context length overflow**: For users with very long histories, the sliding-window summarizer must aggressively compress distant interactions. If the serialized trajectory exceeds the model's context window, truncate the oldest summarized segments first while preserving all recent raw interactions and the full plan/reflection phases.
- **Tool call hallucination**: The student model may generate tool calls for items or users not in the evidence store. At inference time, return a "no evidence available" response for unknown keys and let the model proceed with semantic reasoning only.

## Limitations

- Requires a high-quality multi-agent teacher system to generate training trajectories. If the teacher is poor, the student will internalize flawed reasoning.
- Pre-computed collaborative signal translations become stale as the interaction graph evolves. They need periodic recomputation (daily or weekly depending on interaction velocity).
- The approach assumes access to a user-item interaction graph. Pure content-based scenarios without behavioral data cannot leverage the Collaborative Signal Translation mechanism.
- Trajectory serialization linearizes inherently parallel multi-agent reasoning, which may lose some information from cross-agent deliberation.
- GRPO training requires substantial compute (16+ GPUs in the paper's setup). For smaller teams, the SFT stage alone still provides significant gains over non-distilled baselines.
- The technique is validated on next-item prediction in recommendation. Applying it to other domains (code review, document retrieval) requires careful adaptation of the agent roles and tool definitions.

## Reference

Wu, Y., Wang, H., Li, Q., Zhang, J., & Yu, H. (2026). *Internalizing Multi-Agent Reasoning for Accurate and Efficient LLM-based Recommendation.* arXiv:2602.09829v1. [https://arxiv.org/abs/2602.09829v1](https://arxiv.org/abs/2602.09829v1)

Key sections to study: Section 3.2 (Collaborative Signal Translation mechanism), Section 3.3 (trajectory serialization with structured tokens), Section 3.4 (GRPO reward design with tiered outcome scoring), and Table 1 (comprehensive results showing student outperforming teacher across all scenarios).