---
name: "cowork-x-experience-optimized-co-evolution-multi-a"
description: "Build multi-agent collaboration systems with experience-driven co-evolution using HTN skill libraries and post-episode optimization. Use when: 'build a multi-agent system that improves over episodes', 'create agents that coordinate in real-time with low latency', 'design a skill library for collaborative agents', 'implement experience-based co-evolution for agent teams', 'optimize multi-agent token budget while improving performance', 'set up HTN-based task decomposition for cooperative agents'."
---

# CoWork-X: Experience-Optimized Co-Evolution for Multi-Agent Collaboration

This skill enables Claude to design and implement multi-agent collaboration systems that improve across episodes through structured skill libraries and post-episode optimization. The core technique from CoWork-X separates fast in-episode execution (via hierarchical task network skill retrieval with zero online LLM calls) from slow post-episode reasoning (via a Co-Optimizer that patches the skill library based on diagnosed failures). This produces agents that coordinate in real time, accumulate performance gains episode over episode, and steadily reduce token cost.

## When to Use

- When the user needs to build a multi-agent system where agents must coordinate on shared tasks in real time (e.g., order fulfillment, assembly lines, game AI)
- When agents must improve their joint behavior across repeated episodes without ballooning inference cost
- When the user wants to structure agent capabilities as a composable, editable skill library rather than free-form prompts
- When latency is critical and in-episode LLM reasoning calls are too slow for coordination
- When the user asks to implement hierarchical task network (HTN) planning for agent behavior
- When designing a system where agent skills are patched and consolidated from execution logs automatically

## Key Technique

CoWork-X treats multi-agent collaboration as a closed-loop optimization problem inspired by fast-slow memory separation from neuroscience. During execution, a **Skill-Agent** retrieves and invokes pre-compiled skills from a structured HTN skill library -- no LLM calls happen in-episode. After each episode, a **Co-Optimizer** analyzes execution logs, diagnoses failures (runtime errors, stagnation, action inefficiencies), and synthesizes targeted patches to the skill library. The loop is: `Execute(S_k) -> Diagnose(D_k) -> Update(S_k -> S_{k+1}) -> Execute(S_{k+1})`.

The skill library is a Python module with three layers: (1) **State representation** -- compact abstractions of task-relevant features, omitting spatial noise; (2) **Operators** -- atomic primitives with precondition checks and state-update effects; (3) **Methods** -- task decompositions that map high-level goals to ordered operator sequences. This HTN structure makes skills interpretable, composable, and cheap to execute. The library starts deliberately impaired (syntactically valid but semantically incomplete) to force the Co-Optimizer to learn corrections from real trajectories.

Drift regularization prevents catastrophic edits: the Co-Optimizer receives both the current library and the best-performing historical library, discouraging radical rewrites and enabling rollback when a patch degrades performance. Budget constraints are enforced by design -- the Skill-Agent uses zero online tokens, and the Co-Optimizer operates offline with validation retries (up to 3). In experiments, CoWork-X achieved 15.6x the score of baselines while using 27% fewer total tokens over 30 episodes.

## Step-by-Step Workflow

1. **Define the state representation.** Identify the minimal set of task-relevant features agents need to reason about. Strip spatial details and ephemeral data. Encode as a Python class or dictionary with typed fields (e.g., ingredient counts, queue status, agent assignments).

2. **Implement atomic operators.** Write Python functions for each primitive action. Each operator must: (a) check preconditions against the current state, (b) return the unchanged state on precondition failure, (c) apply state updates and return the modified state on success. Name them descriptively (e.g., `op_assemble_burger`, `op_serve_order`).

3. **Compose methods as task decompositions.** For each high-level goal (e.g., "fulfill order X"), write a method that decomposes it into an ordered sequence of operators. Methods should check applicability conditions and return the subtask list. Register operators and methods with an HTN planner (e.g., Pyhop).

4. **Initialize the skill library as deliberately impaired.** Start with syntactically valid but semantically incomplete code -- operators that skip precondition checks or return unchanged state. This forces the optimization loop to discover corrections from real execution data rather than relying on a priori assumptions.

5. **Build the execution harness.** Run episodes where multiple agents share the same skill library. Each agent independently invokes the HTN planner to decompose goals into primitives, then a low-level controller executes the primitives. Log every timestep: actions taken, state transitions, errors, and stagnation intervals.

6. **Implement the diagnostic pipeline.** After each episode, analyze logs to extract three categories of signal: (a) **Runtime failures** -- exact error locations with surrounding context (2 timesteps before and after), (b) **Stagnation** -- intervals of 100+ consecutive inactive timesteps with the state at stagnation onset, (c) **Action distribution** -- frequency counts by action category to identify inefficient patterns.

7. **Build the Co-Optimizer prompt.** Construct an LLM prompt containing: the current skill library source, the original (initial) library for reference, the best-performing historical library for rollback context, the diagnostic report, and performance metrics from the last 5 episodes. Instruct the LLM to output a patched Python skill library file.

8. **Validate and rollback.** Parse the Co-Optimizer's output as Python, run syntax validation and up to 3 retry attempts if validation fails. Execute one episode with the patched library. If performance drops below the historical best, rollback to that best library. Track a rolling window of the last 5 episode scores.

9. **Iterate the co-evolution loop.** Repeat steps 5-8 across episodes. Monitor three convergence signals: rising episode scores, decreasing failure counts in diagnostics, and stabilizing library diffs (fewer patches needed per iteration).

10. **Extract and freeze the mature skill library.** Once scores plateau and patches become minimal, freeze the library as the production artifact. The frozen library executes with zero LLM tokens and deterministic behavior.

## Concrete Examples

**Example 1: Multi-agent order fulfillment system**

User: "Build a multi-agent kitchen system where two agents collaboratively prepare burger orders, improving their coordination over time."

Approach:
1. Define state: `{beef_patties: int, lettuce: int, assembled_burgers: dict, pending_orders: list, agent_assignments: dict}`
2. Write operators:
```python
def op_grill_patty(state, agent):
    if state.beef_patties <= 0:
        return state  # precondition failure
    state.beef_patties -= 1
    state.grilled_patties += 1
    return state

def op_assemble_beef_burger(state, agent):
    if state.grilled_patties < 1 or state.buns < 1:
        return state
    state.grilled_patties -= 1
    state.buns -= 1
    state.assembled_burgers["beef"] += 1
    return state
```
3. Write methods:
```python
def method_fulfill_beef_order(state, agent):
    if state.pending_orders and "beef" in state.pending_orders[0]:
        return [("op_grill_patty", agent),
                ("op_assemble_beef_burger", agent),
                ("op_serve", agent, "beef")]
    return False
```
4. Start with impaired library (operators return unchanged state), run 10 episodes, feed logs to Co-Optimizer, patch skill library each iteration. After 30 episodes, agents coordinate without collisions and token cost drops to zero in-episode.

**Example 2: Document processing pipeline with co-evolving agents**

User: "I have two agents -- one extracts data from PDFs, one validates and stores it. They keep stepping on each other. Can you make them improve their coordination automatically?"

Approach:
1. Define state: `{pending_docs: list, extracted: dict, validated: dict, errors: list, agent_locks: dict}`
2. Operators: `op_claim_document`, `op_extract_fields`, `op_validate_record`, `op_store_record` -- each with lock-checking preconditions
3. Methods: `method_process_document` decomposes into claim -> extract -> handoff -> validate -> store
4. Initial library omits lock checks (agents collide). Diagnostic pipeline flags stagnation (both agents waiting) and runtime errors (duplicate processing).
5. Co-Optimizer patches in lock-checking preconditions and agent-role specialization after 3-5 episodes.

Output after co-evolution:
```python
# Patched operator with learned precondition
def op_claim_document(state, agent):
    if not state.pending_docs:
        return state
    if state.agent_locks.get(state.pending_docs[0]) is not None:
        return state  # another agent already claimed this doc
    doc = state.pending_docs.pop(0)
    state.agent_locks[doc] = agent
    state.in_progress[agent] = doc
    return state
```

**Example 3: Applying CoWork-X to an existing agent codebase**

User: "I have a multi-agent system but agents waste tokens re-reasoning every step. How do I apply the CoWork-X pattern to reduce cost?"

Approach:
1. Audit the current system: identify which LLM calls happen in-episode vs. between episodes
2. Extract recurring decision patterns from agent logs into operator/method format
3. Build an HTN skill library encoding those patterns as deterministic code
4. Replace in-episode LLM calls with HTN planner invocations
5. Add a post-episode Co-Optimizer that receives execution logs and patches the library
6. Measure: expect online tokens to drop to near-zero, latency to drop by 10-30x, with scores improving over episodes as the library matures

## Best Practices

- **Do:** Start with an impaired but executable skill library. Operators should be syntactically valid Python that runs without crashing, even if they do nothing useful. This gives the Co-Optimizer concrete failure signals to learn from.
- **Do:** Include the best historical library in every Co-Optimizer prompt. This anchors edits and enables automatic rollback, preventing catastrophic forgetting.
- **Do:** Log generously during execution -- runtime errors with 2-timestep context windows, stagnation intervals with onset states, and full action distributions. The Co-Optimizer is only as good as its diagnostic input.
- **Do:** Validate patched libraries syntactically before execution. Use up to 3 retry attempts with error feedback passed back to the LLM.
- **Avoid:** Making in-episode LLM calls. The entire point of the fast-slow separation is that execution uses pre-compiled skills. If you need LLM reasoning during an episode, your skill library is incomplete -- fix it in the optimization phase.
- **Avoid:** Radical library rewrites in a single optimization step. Constrain the Co-Optimizer to patch-style edits (add preconditions, fix state updates, complete method logic) rather than full rewrites. This is the drift regularization principle.

## Error Handling

- **Co-Optimizer produces invalid Python:** Run `ast.parse()` on the output. On failure, feed the syntax error back to the LLM with the original library and retry (up to 3 times). If all retries fail, keep the current library for the next episode.
- **Performance regression after a patch:** Compare the new episode score against the historical best. If it drops, immediately rollback to the best-performing library. Log the failed patch for analysis.
- **Stagnation not resolving across episodes:** If the same stagnation pattern persists for 3+ consecutive episodes, escalate by including a broader context window (5 timesteps) in the diagnostic report and explicitly flagging it as a recurring issue in the Co-Optimizer prompt.
- **Agents deadlock on shared resources:** Ensure operators encode mutual-exclusion preconditions. If deadlocks appear in logs, the diagnostic pipeline should flag them as a distinct failure category so the Co-Optimizer can patch in lock-ordering or turn-taking logic.
- **Token budget exceeded in Co-Optimizer:** Cap the prompt size by summarizing older episode diagnostics and only including full details for the most recent 2-3 episodes. Use the 5-episode rolling window for metrics.

## Limitations

- Requires a repeatable episode structure. One-shot or highly variable tasks don't generate the consistent trajectory data needed for cross-episode optimization.
- The HTN skill library assumes tasks can be decomposed hierarchically. Domains with emergent, non-decomposable coordination patterns (e.g., continuous physical manipulation) are a poor fit.
- Initial episodes will perform poorly by design (impaired library). Not suitable when first-episode performance matters.
- The Co-Optimizer depends on high-quality diagnostics. If the execution environment doesn't support detailed logging (timestep-level actions, error traces), the optimization loop has insufficient signal.
- Agents share a single skill library, which means agent specialization must be encoded within the library's method dispatch, not via separate agent architectures.

## Reference

**Paper:** [CoWork-X: Experience-Optimized Co-Evolution for Multi-Agent Collaboration System](https://arxiv.org/abs/2602.05004v1) (Lin et al., 2026)
**Key insight:** Separating fast skill-based execution from slow post-episode optimization lets multi-agent systems achieve real-time coordination with zero in-episode token cost while accumulating performance gains across episodes through structured, patch-style library evolution.