---
name: "agent-primitives-reusable-latent"
description: "Design and orchestrate multi-agent systems using reusable Agent Primitives (Review, Voting/Selection, Planning/Execution) that compose into task-specific pipelines. Use when asked to: 'build a multi-agent workflow', 'create an agent pipeline for this task', 'set up agents to review and refine output', 'orchestrate parallel agent voting', 'decompose this into a planning and execution pipeline', 'design a reusable agent architecture'."
---

# Agent Primitives: Reusable Building Blocks for Multi-Agent Systems

This skill enables Claude to design, compose, and implement multi-agent systems (MAS) from three reusable primitives -- **Review**, **Voting and Selection**, and **Planning and Execution** -- instead of hand-crafting bespoke agent roles for every task. Inspired by how neural networks are built from standard layers (convolution, attention, etc.), Agent Primitives let you decompose any complex MAS architecture into a small set of recurring computation patterns, then recombine them for new problems. This yields systems that are 12-16% more accurate than single-agent baselines while using 3-4x fewer tokens than text-based multi-agent approaches.

## When to Use

- When the user wants to build a multi-agent system for a complex task (math reasoning, code generation, Q&A) and needs a structured architecture rather than ad-hoc agent chaining.
- When an existing single-agent approach produces unreliable or inconsistent results and the user asks for a review/critique loop or ensemble voting.
- When the user asks to decompose a large task into planning and execution phases with separate agent responsibilities.
- When the user wants to refactor an existing multi-agent system into reusable, composable components.
- When the user needs to reduce token cost and latency of a multi-agent pipeline without sacrificing accuracy.
- When building agentic workflows that must generalize across different task types (math, code, QA) without per-task prompt engineering.

## Key Technique

**The core insight** is that most multi-agent architectures -- Self-Refine, Multi-Agent Debate, Self-Consistency, task decomposition -- reduce to just three internal computation patterns. Rather than designing unique agent roles and interaction prompts per task, you instantiate one or more of these primitives and wire them together:

1. **Review Primitive** -- A Solver generates an initial response; a Critic evaluates it and provides structured feedback; the Solver refines iteratively. This replaces Self-Refine, debate, and any iterative-improvement loop. The Critic must identify errors but never revise the solution itself -- separation of concerns prevents feedback collapse.

2. **Voting and Selection Primitive** -- Multiple independent Solvers each produce a candidate solution in parallel. A Selector agent evaluates all candidates and picks or aggregates the best one based on correctness and consistency. This replaces majority voting, self-consistency, and any ensemble approach.

3. **Planning and Execution Primitive** -- A Planner decomposes the task into structured intermediate steps. An Executor follows the plan to produce the final output, without modifying the plan. This replaces task decomposition, chain-of-thought scaffolding, and orchestrator-worker patterns.

**Composition via an Organizer.** An Organizer agent acts as a lightweight router: given a new query, it consults a knowledge pool of (query-pattern, effective-primitive-composition) pairs, selects which primitives to instantiate, determines execution order (sequential, parallel, nested), and emits a composition plan. The knowledge pool is populated from successful runs and can be bootstrapped from known MAS architectures mapped to their primitive decompositions.

**Efficiency through structured state passing.** In the original paper, primitives communicate via KV cache concatenation rather than serialized text, reducing information degradation across stages. When implementing with Claude Code agents (which communicate via text), we approximate this by passing structured JSON state objects between primitives -- not free-form natural language summaries. This preserves signal fidelity and keeps token counts low.

## Step-by-Step Workflow

1. **Analyze the task and identify the primitive composition.** Classify the user's task: does it need iterative refinement (Review), parallel diversity (Voting/Selection), decomposition (Planning/Execution), or a combination? Map the task to a primitive graph. For example, a complex math problem might use Planning/Execution -> Review, while a code generation task might use Voting/Selection -> Review.

2. **Design the Organizer's routing logic.** Define a decision function that maps task characteristics to primitive compositions. Use a simple heuristic or a knowledge pool lookup:
   - Tasks requiring correctness verification -> Review
   - Tasks with multiple valid approaches -> Voting/Selection
   - Tasks requiring multi-step decomposition -> Planning/Execution
   - Complex or high-stakes tasks -> compose multiple primitives

3. **Instantiate the Review primitive (if selected).** Create two agents with distinct system prompts:
   - **Solver**: "Generate a solution to the given problem. If feedback is provided from a prior review round, incorporate it to fix identified errors."
   - **Critic**: "Evaluate the proposed solution. Identify specific errors, inconsistencies, or gaps. Do NOT revise or complete the solution yourself -- only provide diagnostic feedback."
   - Set a maximum iteration count (typically 2-3 rounds) and a stopping condition (Critic finds no errors, or max iterations reached).

4. **Instantiate the Voting/Selection primitive (if selected).** Create N independent Solver agents (typically 3-5) and one Selector:
   - **Each Solver**: "Independently generate a candidate solution. Rely only on the input query -- do not reference other candidates."
   - **Selector**: "You are given N candidate solutions. Evaluate each for correctness, completeness, and consistency. Select the best candidate or synthesize a final answer from the strongest elements."
   - Run all Solvers in parallel for maximum throughput.

5. **Instantiate the Planning/Execution primitive (if selected).** Create a Planner and an Executor:
   - **Planner**: "Decompose this task into a structured plan with numbered intermediate steps. Each step should be specific and actionable. Output the plan as a JSON array of step objects."
   - **Executor**: "Execute the following plan step by step. Follow the plan exactly -- do not modify, skip, or reorder steps. Produce the final output after completing all steps."

6. **Wire primitives together via structured state.** Define the data contract between primitives as a JSON schema. Each primitive receives a structured input and emits a structured output. Example state object:
   ```json
   {
     "query": "original user query",
     "plan": ["step1", "step2"],
     "candidates": [{"solution": "...", "confidence": 0.9}],
     "review_feedback": ["issue1", "issue2"],
     "final_answer": "..."
   }
   ```

7. **Implement the execution engine.** Write the orchestration code that:
   - Calls the Organizer to select the primitive composition
   - Instantiates each primitive with its system prompt
   - Passes structured state between primitives in sequence or parallel as specified
   - Collects the final output from the last primitive in the chain

8. **Populate the knowledge pool.** After each successful run, store the (task_type, primitive_composition, accuracy) triple. On future runs, the Organizer retrieves the top-k most similar past configurations to guide its routing decision. Start with these seed mappings:
   - Iterative refinement tasks (essay writing, code review) -> Review
   - High-variance generation (creative writing, brainstorming) -> Voting/Selection
   - Multi-step reasoning (math proofs, research synthesis) -> Planning/Execution -> Review

9. **Tune iteration and parallelism parameters.** For Review: 2-3 iterations is typically optimal (diminishing returns beyond 3). For Voting: 3-5 candidates balances diversity against cost. For Planning: limit plan steps to 3-7 to avoid over-decomposition.

10. **Validate and measure.** Compare the primitive-based MAS against a single-agent baseline on the target task. Track three metrics: accuracy, total tokens used, and end-to-end latency. Expect 12-16% accuracy gains at 1.3-1.6x the single-agent token cost.

## Concrete Examples

**Example 1: Code Generation with Review Primitive**

User: "Build an agent pipeline that generates a Python function and then reviews it for bugs."

Approach:
1. Identify this as an iterative refinement task -> select the **Review** primitive.
2. Instantiate a Solver agent with prompt: "Write a Python function for the given specification."
3. Instantiate a Critic agent with prompt: "Review this Python function. Identify bugs, edge cases, type errors, and logic issues. Do NOT fix the code -- only list problems."
4. Run for 2 iterations, passing structured feedback between rounds.

Implementation:
```python
import subprocess, json

def run_agent(system_prompt, user_message):
    """Call Claude via CLI or API with a system prompt and user message."""
    # Replace with your preferred invocation method
    result = subprocess.run(
        ["claude", "-p", user_message, "--system", system_prompt, "--output-format", "json"],
        capture_output=True, text=True
    )
    return json.loads(result.stdout)["result"]

def review_primitive(query, max_iterations=2):
    solver_prompt = (
        "You are a Solver. Write a Python function for the given specification. "
        "If review feedback is provided, fix the identified issues."
    )
    critic_prompt = (
        "You are a Critic. Review the code below for bugs, edge cases, and logic errors. "
        "Output a JSON array of issues found. Output an empty array [] if no issues remain. "
        "Do NOT fix or rewrite the code."
    )

    solution = run_agent(solver_prompt, f"Specification: {query}")

    for i in range(max_iterations):
        feedback = run_agent(critic_prompt, f"Code to review:\n```python\n{solution}\n```")
        if feedback.strip() == "[]":
            break
        solution = run_agent(
            solver_prompt,
            f"Specification: {query}\n\nPrevious solution:\n```python\n{solution}\n```\n\nFeedback: {feedback}"
        )
    return solution

# Usage
result = review_primitive("Write a function that merges two sorted lists into one sorted list.")
print(result)
```

Output: A refined Python function that has been reviewed for bugs across 2 iterations, with each iteration targeting specific issues identified by the Critic.

---

**Example 2: Math Problem Solving with Voting/Selection**

User: "I need a reliable agent setup for solving competition math problems. Single attempts are too inconsistent."

Approach:
1. High-variance reasoning task -> select the **Voting and Selection** primitive.
2. Spawn 5 independent Solver agents in parallel.
3. A Selector agent evaluates all candidates and picks the most consistent answer.

Implementation:
```python
import concurrent.futures

def voting_primitive(query, num_solvers=5):
    solver_prompt = (
        "Solve the following math problem step by step. "
        "Show your work and state the final answer clearly."
    )

    # Run solvers in parallel
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_solvers) as pool:
        futures = [pool.submit(run_agent, solver_prompt, query) for _ in range(num_solvers)]
        candidates = [f.result() for f in concurrent.futures.as_completed(futures)]

    # Selector evaluates candidates
    selector_prompt = (
        "You are a Selector. You are given multiple candidate solutions to a math problem. "
        "Evaluate each for correctness. Identify the most common final answer. "
        "If answers disagree, verify the reasoning of each and select the correct one. "
        "Output ONLY the final verified answer."
    )
    candidate_text = "\n\n---\n\n".join(
        [f"Candidate {i+1}:\n{c}" for i, c in enumerate(candidates)]
    )
    return run_agent(selector_prompt, f"Problem: {query}\n\n{candidate_text}")

result = voting_primitive("Find all real solutions to x^4 - 5x^2 + 6 = 0.")
```

Output: The Selector identifies that 4 of 5 candidates agree on x = +/-sqrt(2), +/-sqrt(3), confirms the reasoning, and returns the verified answer.

---

**Example 3: Research Synthesis with Composed Primitives (Planning/Execution -> Review)**

User: "Design an agent system that can take a research question, break it down, gather findings, and produce a vetted synthesis."

Approach:
1. Multi-step decomposition + quality assurance -> compose **Planning/Execution** then **Review**.
2. Planner decomposes the research question into sub-questions.
3. Executor addresses each sub-question and synthesizes.
4. Review primitive critiques the synthesis for gaps and inaccuracies.

Implementation:
```python
def composed_pipeline(query):
    # Stage 1: Planning/Execution
    planner_prompt = (
        "Decompose this research question into 3-5 specific sub-questions. "
        "Output as a JSON array of strings."
    )
    plan = json.loads(run_agent(planner_prompt, query))

    executor_prompt = (
        "You are given a research question and a structured plan of sub-questions. "
        "Address each sub-question, then synthesize a coherent answer to the main question. "
        "Follow the plan order exactly."
    )
    plan_text = "\n".join([f"{i+1}. {step}" for i, step in enumerate(plan)])
    synthesis = run_agent(executor_prompt, f"Question: {query}\n\nPlan:\n{plan_text}")

    # Stage 2: Review
    refined = review_primitive(
        f"Review and improve this research synthesis for accuracy and completeness:\n\n"
        f"Original question: {query}\n\nSynthesis:\n{synthesis}",
        max_iterations=1
    )
    return refined
```

## Best Practices

**Do:**
- Always pass structured data (JSON objects) between primitives, not free-form narrative. This mimics the paper's KV cache communication and prevents information degradation across stages.
- Keep the Critic role strictly diagnostic -- it identifies problems but never writes solutions. Mixing roles causes feedback collapse where the Critic just rewrites the answer.
- Run Voting/Selection Solvers with temperature > 0 (or varied system prompts) to ensure genuine diversity among candidates. Identical prompts at temperature 0 produce identical outputs.
- Start with the simplest single-primitive composition and add complexity only when metrics justify it. A single Review primitive often outperforms elaborate multi-primitive chains.

**Avoid:**
- Do not nest more than 2 primitives deep. Beyond that, error propagation offsets the accuracy gains and token costs escalate.
- Do not use the Voting primitive with fewer than 3 candidates -- below that threshold, the Selector lacks enough signal to identify the correct answer reliably.
- Do not let the Executor modify the Planner's plan. The separation is load-bearing: if the Executor can rewrite the plan, you lose the decomposition benefit and revert to single-agent behavior.
- Do not skip the Organizer/routing step for production systems. Hard-coding a single primitive composition works for prototyping but fails to generalize across task types.

## Error Handling

- **Critic produces no actionable feedback on first round:** The solution may already be correct. Accept it and exit the Review loop. Do not force additional iterations -- this wastes tokens and can introduce regressions.
- **All Voting candidates produce different answers:** Increase the number of candidates to 7-9, or fall back to a Review primitive on the Selector's best pick. Persistent disagreement signals the problem may exceed the model's capability.
- **Planner produces an overly granular plan (>7 steps):** Re-prompt the Planner with an explicit constraint: "Decompose into at most 5 high-level steps." Over-decomposition causes the Executor to lose coherence across steps.
- **Organizer selects an inappropriate primitive:** This usually means the knowledge pool lacks coverage for the task type. Add a manual override and record the correct mapping to improve future routing.
- **Token budget exceeded mid-pipeline:** Implement a token budget tracker that aborts gracefully after any primitive completes, returning the best partial result rather than failing silently.

## Limitations

- **Same-model requirement for KV cache sharing:** The paper's latent communication via KV cache concatenation requires all agents to share the same model weights and tokenizer. When using Claude Code agents (which communicate via text), you lose the raw efficiency gains but retain the architectural benefits of primitive composition.
- **Diminishing returns on simple tasks:** For straightforward tasks (single-step lookups, simple formatting), the overhead of even one primitive exceeds the benefit. Use primitives only when single-agent accuracy is insufficient.
- **Knowledge pool cold start:** The Organizer's routing quality depends on accumulated (task, composition) pairs. New deployments must rely on the seed mappings until enough data accumulates.
- **Not suitable for real-time interaction:** Even the lightest single-primitive composition adds 1.3-1.6x latency over a single agent call. For sub-second response requirements, use single-agent inference.
- **Assumes decomposable tasks:** Tasks requiring holistic judgment that cannot be separated into review, voting, or plan/execute phases (e.g., nuanced creative writing with a singular voice) may not benefit from primitive decomposition.

## Reference

**Paper:** [Agent Primitives: Reusable Latent Building Blocks for Multi-Agent Systems](https://arxiv.org/abs/2602.03695v1) (Jin et al., 2026). Look for Section 3 (primitive definitions and KV cache formulation), Section 4 (Organizer and knowledge pool design), and Table 2 (accuracy comparisons across 8 benchmarks showing 12-16% gains over single-agent baselines at 1.3-1.6x cost).