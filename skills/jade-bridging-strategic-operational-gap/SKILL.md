---
name: "jade-bridging-strategic-operational-gap"
description: "Build jointly-optimized agentic RAG pipelines using the JADE pattern: a central planner co-adapted with specialized executors sharing a unified backbone. Eliminates the strategic-operational mismatch where planners issue strategies executors cannot fulfill. Use when: 'build an agentic RAG system', 'create a multi-step retrieval pipeline', 'design a planner-executor workflow for question answering', 'implement dynamic retrieval with joint optimization', 'build a multi-hop QA agent', 'connect my planner to retrieval executors'."
---

This skill teaches Claude to design and implement **JADE-style agentic RAG systems** (Joint Agentic Dynamic Engine) where a central planner and multiple specialized executors are jointly optimized under a shared backbone. Instead of treating retrieval, rewriting, and answer generation as frozen black-box tools called by a separate planner, the JADE pattern co-adapts all components so the planner learns what executors can actually do and executors evolve to fulfill the planner's strategic intent. This produces systems where a small jointly-trained model outperforms much larger models running decoupled pipelines.

## When to Use

- When the user wants to build an agentic RAG pipeline with a planner that routes queries to retrieval, rewriting, summarization, or answer-generation steps
- When implementing multi-hop question answering that requires decomposing complex queries into sub-questions with sequential dependencies
- When the user asks to create a dynamic retrieval workflow that adapts its depth and breadth based on query complexity
- When designing a system where a planning agent orchestrates multiple executor agents (retriever, selector, generator) and you need them to work cohesively
- When an existing agentic RAG system suffers from the planner requesting actions that executors fail to carry out correctly (strategic-operational mismatch)
- When the user wants to jointly train or fine-tune a planner and its executor modules end-to-end with reinforcement learning

## Key Technique

### The Strategic-Operational Mismatch

Existing agentic RAG systems fall into two camps. **Fixed-graph systems** (like MMOA-RAG) optimize all modules jointly but lock the workflow topology at design time -- every query follows the same pipeline regardless of complexity. **Dynamic agentic systems** (like MAO-ARAG) allow a planner to construct flexible workflows on the fly but treat each executor as a frozen tool, creating a gap: the planner devises strategies the executors were never trained to carry out. JADE calls this the *strategic-operational mismatch*. A planner might instruct a query rewriter to produce a specific style of sub-question, but if the rewriter was trained independently, it produces whatever it was originally optimized for, and the downstream pipeline degrades.

### JADE's Solution: Shared Backbone with Role-Specific Prompts

JADE models the entire system as a **cooperative multi-agent team on a single shared LLM backbone**. The planner and all seven executor roles (Query Decomposer-Serial, Query Decomposer-Parallel, Query Rewriter, Retrieval Agent, Document Selector, Answer Generator, Answer Summarizer) share identical model parameters and are distinguished only by role-specific system prompts. Training uses **Proximal Policy Optimization (PPO)** over a unified experience buffer: every agent interaction produces a transition tuple (observation, action, reward), and all transitions -- regardless of which role generated them -- are aggregated and used to update the shared backbone. The reward signal is global: `R = F1_score - (alpha * reasoning_rounds + beta * retrieval_calls) + local_format_penalties`. This forces the planner to learn what the executors can realistically accomplish, while executors adapt to fulfill the planner's strategies. Credit assignment across the multi-turn chain uses Generalized Advantage Estimation.

### Dynamic Workflow Construction

Each reasoning round follows three phases: (1) **Workflow Planning** -- the planner inspects the current state (query + accumulated sub-question/answer pairs) and emits either a decomposition action (split the query) or a solving action (retrieve, select, generate); (2) **Workflow Execution** -- the selected executor chain runs, expanding the reasoning trace; (3) **State Update** -- the global state is updated with new query-answer pairs, and the planner decides whether to continue or terminate. This produces a dynamic graph whose topology is shaped by query complexity: single-hop queries trigger one-round solve chains while multi-hop queries produce multi-round serial decomposition trees.

## Step-by-Step Workflow

1. **Define the executor roles.** Enumerate the specialized roles your pipeline needs. A standard JADE configuration uses seven: Query Decomposer (Serial), Query Decomposer (Parallel), Query Rewriter, Retrieval Agent, Document Selector, Answer Generator, and Answer Summarizer. Write a role-specific system prompt for each that constrains its output format (e.g., the Retrieval Agent must output a search query string; the Document Selector must output a list of document IDs).

2. **Implement the shared backbone.** Choose a single LLM (e.g., Qwen-2.5-7B, Llama-3-8B, or an API-based model). All roles share the same model weights. At inference, prepend the role-specific system prompt to steer behavior. In code, this is a single `generate(system_prompt, context)` function parameterized by role.

3. **Build the global state tracker.** Create a data structure that accumulates the evolving reasoning trace: a list of `(sub_query, retrieved_docs, selected_docs, answer)` tuples plus the original query. Each round appends to this trace. The planner sees the full trace when deciding the next action.

4. **Implement the planner's action space.** The planner outputs one of: `DECOMPOSE_SERIAL(query)`, `DECOMPOSE_PARALLEL(query)`, `SOLVE(query)`, or `TERMINATE`. Parse the planner's output into a structured action. If the format is invalid, apply a local penalty and re-prompt.

5. **Wire up the three-phase loop.** For each round: (a) the planner inspects the global state and emits an action; (b) the action triggers the corresponding executor chain -- decomposition actions invoke the decomposer then recurse, solve actions invoke Retrieval Agent -> Document Selector -> Answer Generator in sequence; (c) the state tracker records results. Loop until the planner emits `TERMINATE` or a max-round limit is reached.

6. **Implement the reward function.** Compute a global reward at termination: `R = F1(predicted_answer, gold_answer) - alpha * num_rounds - beta * num_retrieval_calls`. Add local format penalties (e.g., -0.5 for malformed executor output). The cost terms `alpha` and `beta` control the efficiency-effectiveness tradeoff; start with `alpha=0.1, beta=0.05` and tune.

7. **Collect transition tuples.** During each rollout, record `(observation, action, reward, next_observation)` for every agent interaction across all roles. Flatten these heterogeneous transitions into a single experience buffer -- do not separate by role.

8. **Train with PPO on the unified buffer.** Run standard PPO updates on the shared backbone using the aggregated buffer. Use Generalized Advantage Estimation (GAE, lambda=0.95) for temporal credit assignment so early planning decisions receive appropriate credit for final answer quality. Train for 3-5 epochs per batch of rollouts.

9. **Evaluate with adaptive complexity.** Test on queries of varying hop-count (1-hop, 2-hop, multi-hop). Verify that the system learns to use fewer rounds for simple queries and more rounds for complex ones. Check that the Document Selector invocation rate scales with query difficulty (target: ~20% for simple, ~60% for complex).

10. **Deploy with fallback guards.** In production, set a hard max of 5 reasoning rounds, validate every executor output against its expected schema before feeding it downstream, and implement a timeout per executor call. If any executor produces invalid output twice consecutively, force the planner to `TERMINATE` and return the best available answer.

## Concrete Examples

**Example 1: Building a JADE-style multi-hop QA pipeline in Python**

User: "I want to build a multi-hop question answering system that can handle queries like 'What award did the director of Inception win in 2015?' using my document corpus."

Approach:
1. Define executor roles with system prompts:
   - `DECOMPOSER_SERIAL`: "Break this question into sequential sub-questions where each depends on the previous answer."
   - `RETRIEVER`: "Given a query, produce a search string optimized for BM25/dense retrieval."
   - `SELECTOR`: "Given retrieved documents and a query, output the IDs of the most relevant documents."
   - `GENERATOR`: "Given selected documents and a query, produce a concise factual answer."
   - `SUMMARIZER`: "Given multiple sub-answers, synthesize a final answer to the original question."
2. Implement shared backbone as a single LLM call parameterized by role prompt.
3. Wire the planner loop that decomposes "What award did the director of Inception win in 2015?" into:
   - Sub-Q1: "Who directed Inception?" -> Retrieve -> Select -> Generate -> "Christopher Nolan"
   - Sub-Q2: "What award did Christopher Nolan win in 2015?" -> Retrieve -> Select -> Generate -> "..."
4. Summarizer combines sub-answers into the final response.

Output:
```python
import json
from dataclasses import dataclass, field

@dataclass
class ReasoningState:
    original_query: str
    trace: list = field(default_factory=list)  # [(sub_q, docs, answer)]
    round_num: int = 0
    retrieval_calls: int = 0

ROLE_PROMPTS = {
    "planner": (
        "You are a query planner. Given the original query and reasoning trace, "
        "output exactly one action: DECOMPOSE_SERIAL(query), SOLVE(query), or TERMINATE.\n"
        "Choose DECOMPOSE_SERIAL when the query requires information from a prior sub-answer. "
        "Choose SOLVE when the query can be answered directly. Choose TERMINATE when done."
    ),
    "decomposer_serial": (
        "Break the given query into 2-3 sequential sub-questions. "
        "Output a JSON list of strings. Each sub-question may reference '[PREV_ANSWER]' "
        "as a placeholder for the answer to the preceding sub-question."
    ),
    "retriever": "Rewrite the query into an optimized search string. Output only the search string.",
    "selector": (
        "Given retrieved documents and a query, output a JSON list of the "
        "most relevant document indices (0-indexed). Select at most 3."
    ),
    "generator": "Answer the query using only the provided documents. Be concise and factual.",
    "summarizer": "Synthesize the sub-answers into a final answer to the original question.",
}

def call_llm(role: str, context: str, backbone) -> str:
    """All roles share the same backbone, differentiated by system prompt."""
    return backbone.generate(system=ROLE_PROMPTS[role], user=context)

def jade_pipeline(query: str, retriever_fn, backbone, max_rounds: int = 5) -> str:
    state = ReasoningState(original_query=query)

    while state.round_num < max_rounds:
        # Phase 1: Planning
        plan_ctx = f"Query: {query}\nTrace: {json.dumps(state.trace)}"
        action = call_llm("planner", plan_ctx, backbone).strip()

        if action == "TERMINATE":
            break

        if action.startswith("DECOMPOSE_SERIAL"):
            sub_qs = json.loads(call_llm("decomposer_serial", query, backbone))
            for sq in sub_qs:
                sq_resolved = sq.replace(
                    "[PREV_ANSWER]",
                    state.trace[-1]["answer"] if state.trace else ""
                )
                answer = _solve_chain(sq_resolved, state, retriever_fn, backbone)
                state.trace.append({"sub_query": sq_resolved, "answer": answer})

        elif action.startswith("SOLVE"):
            answer = _solve_chain(query, state, retriever_fn, backbone)
            state.trace.append({"sub_query": query, "answer": answer})

        state.round_num += 1

    # Final summarization
    summary_ctx = f"Original: {query}\nSub-answers: {json.dumps(state.trace)}"
    return call_llm("summarizer", summary_ctx, backbone)

def _solve_chain(query, state, retriever_fn, backbone):
    search_str = call_llm("retriever", query, backbone)
    docs = retriever_fn(search_str)
    state.retrieval_calls += 1
    selected = json.loads(call_llm("selector", f"Query: {query}\nDocs: {docs}", backbone))
    chosen_docs = [docs[i] for i in selected if i < len(docs)]
    return call_llm("generator", f"Query: {query}\nDocs: {chosen_docs}", backbone)
```

**Example 2: Adding joint optimization via PPO to an existing pipeline**

User: "I have a planner-executor RAG pipeline but the planner keeps issuing decomposition strategies the executors can't handle well. How do I fix this mismatch?"

Approach:
1. Diagnose the mismatch: log planner actions alongside executor success rates. Identify which planner strategies produce low-quality executor outputs.
2. Merge planner and executor onto a shared backbone with role-specific prompts.
3. Implement PPO training with a unified reward signal.

Output:
```python
def compute_jade_reward(predicted: str, gold: str, state: ReasoningState,
                        alpha: float = 0.1, beta: float = 0.05) -> float:
    """Global reward combining quality and efficiency."""
    from rouge_score import rouge_scorer
    scorer = rouge_scorer.RougeScorer(["rougeL"], use_stemmer=True)
    f1 = scorer.score(gold, predicted)["rougeL"].fmeasure

    efficiency_penalty = alpha * state.round_num + beta * state.retrieval_calls
    return f1 - efficiency_penalty

def collect_transitions(rollout_episodes):
    """Flatten heterogeneous transitions from all roles into one buffer."""
    buffer = []
    for episode in rollout_episodes:
        for step in episode.steps:
            # step contains: role, observation, action, reward, next_obs
            # Do NOT filter by role -- all roles share the backbone
            buffer.append({
                "obs": step.observation,
                "action": step.action,
                "reward": step.reward,
                "next_obs": step.next_observation,
                "done": step.is_terminal,
            })
    return buffer

# Training loop sketch
for epoch in range(num_epochs):
    episodes = run_rollouts(jade_pipeline, training_queries, num_episodes=64)
    buffer = collect_transitions(episodes)
    advantages = compute_gae(buffer, gamma=0.99, lam=0.95)
    ppo_update(shared_backbone, buffer, advantages, clip_epsilon=0.2)
```

**Example 3: Adaptive complexity for mixed-difficulty queries**

User: "My RAG system wastes too many retrieval calls on simple factoid questions. How do I make it adaptive?"

Approach:
1. The planner learns to emit `SOLVE` directly for simple queries and `DECOMPOSE_SERIAL` for complex ones.
2. The cost term in the reward function penalizes unnecessary rounds and retrieval calls.
3. After joint training, verify that simple queries converge to 1-round pipelines.

Output:
```
Simple query: "What is the capital of France?"
  -> Planner: SOLVE("What is the capital of France?")
  -> Retriever -> Selector -> Generator -> "Paris"
  -> Planner: TERMINATE
  Rounds: 1, Retrieval calls: 1

Complex query: "What university did the inventor of the telephone attend?"
  -> Planner: DECOMPOSE_SERIAL(...)
  -> Sub-Q1: "Who invented the telephone?" -> Solve chain -> "Alexander Graham Bell"
  -> Sub-Q2: "What university did Alexander Graham Bell attend?" -> Solve chain -> "University of Edinburgh"
  -> Planner: TERMINATE
  -> Summarizer: "Alexander Graham Bell attended the University of Edinburgh."
  Rounds: 3, Retrieval calls: 2
```

## Best Practices

- **Do:** Use a single shared backbone for all roles. The entire point of JADE is that joint optimization through shared parameters forces co-adaptation. Separate models for planner and executors reintroduce the mismatch.
- **Do:** Include efficiency costs (round count, retrieval calls) in the reward function. Without them, the system learns to over-retrieve and over-decompose since more retrieval generally helps accuracy but wastes resources.
- **Do:** Flatten all role transitions into a single experience buffer for PPO. Separating buffers by role prevents the cross-role gradient flow that drives co-adaptation.
- **Do:** Start with serial decomposition as the default for multi-hop queries. JADE's experiments show serial decomposition outperforms parallel decomposition on queries with sequential dependencies, which are the majority of real multi-hop questions.
- **Avoid:** Treating any executor as a frozen external API during training. If one module (e.g., a retriever using a fixed embedding model) cannot be updated, wrap it with a trainable query rewriter that can adapt to compensate.
- **Avoid:** Setting max rounds too high (>5). JADE's experiments show diminishing returns beyond 3-4 rounds, and the planner can learn to stall. Enforce a hard ceiling.

## Error Handling

- **Malformed executor output:** If an executor returns output that doesn't match its expected schema (e.g., the selector returns prose instead of document IDs), apply a local format penalty of -0.5 to that transition, log the failure, and re-prompt once. If it fails again, skip that executor and proceed with all retrieved documents.
- **Planner loops:** If the planner emits the same action three consecutive rounds without the state changing, force `TERMINATE` and return the best accumulated answer.
- **Empty retrieval results:** If the retrieval agent returns zero documents, have the query rewriter reformulate the search string before retrying. Limit to one retry.
- **Reward hacking:** If the system learns to `TERMINATE` immediately to avoid efficiency penalties, increase the F1 weight relative to cost terms. A ratio of 1.0 (F1) to 0.1 (round cost) to 0.05 (retrieval cost) is a stable starting point.

## Limitations

- **Requires RL infrastructure.** Joint PPO training on a shared backbone needs rollout collection, advantage estimation, and policy gradient updates. This is substantially more complex than supervised fine-tuning of individual modules.
- **Single-backbone constraint.** All roles must share one model. If your architecture requires heterogeneous models (e.g., a specialized dense retriever + a generative LLM), you cannot directly apply the shared-backbone pattern. You can approximate it by freezing the retriever and jointly training a rewriter + planner + generator.
- **Training data requirements.** You need queries with gold-standard answers to compute the F1 reward. The approach does not work for open-ended generation tasks without clear evaluation metrics.
- **Scale ceiling.** JADE's results are demonstrated on 7B-scale models. The co-adaptation benefit may diminish at very large scales (70B+) where each role already has sufficient capacity to generalize independently.
- **Not suitable for single-turn QA.** If your use case is always single-hop retrieval with no need for decomposition or multi-round reasoning, a standard RAG pipeline is simpler and sufficient.

## Reference

**Paper:** [JADE: Bridging the Strategic-Operational Gap in Dynamic Agentic RAG](https://arxiv.org/abs/2601.21916v1) (Chen et al., 2026)

Look for: Section 3 (the three-phase dynamic workflow), Section 4 (PPO training with unified experience buffer and GAE), and Table 2 (benchmark comparisons showing joint 7B model beating GPT-4o-based decoupled systems).