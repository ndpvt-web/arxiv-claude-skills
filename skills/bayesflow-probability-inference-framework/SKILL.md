---
name: "bayesflow-probability-inference-framework"
description: "Generate high-quality multi-step LLM workflows using Bayesian inference with parallel look-ahead rollouts and importance-weighted resampling. Use when: 'build a workflow for this task', 'generate an agent pipeline', 'create a multi-step LLM chain', 'optimize my prompt chain', 'design an agentic workflow', 'Bayesian workflow generation'."
---

# BayesFlow: Bayesian Workflow Generation for Multi-Step LLM Pipelines

This skill enables Claude to design and construct multi-step LLM workflows (chains of LLM calls, tool invocations, and post-processing logic) using the Bayesian Workflow Generation (BWG) framework from the BayesFlow paper. Instead of hand-crafting a single pipeline or brute-force searching over workflow configurations, BWG treats workflow generation as Bayesian inference: the LLM's internal knowledge serves as a prior over plausible workflows, task-specific reward signals form the likelihood, and the posterior concentrates probability on workflows that actually solve the task. The framework builds workflows step-by-step using parallel look-ahead rollouts for importance weighting and a sequential refiner for pool-wide quality improvements.

## When to Use

- When a user asks you to **design a multi-step LLM pipeline** (e.g., "build me an agent workflow that researches a topic, drafts an outline, writes sections, then edits")
- When a user wants to **optimize an existing prompt chain** by exploring alternative step orderings, prompts, or control flow
- When a user needs a **robust workflow for a complex task** that benefits from self-correction loops, conditional branching, or ensemble strategies
- When a user asks to **generate code that orchestrates multiple LLM calls** with tool use (search, code execution, retrieval) for an end-to-end pipeline
- When a user wants to **compare multiple candidate workflows** and select the best one based on validation performance
- When building **agentic systems** where the sequence of actions (LLM calls, tool use, verification) needs principled construction rather than ad-hoc chaining

## Key Technique

**Core insight:** Most workflow generation methods treat pipeline design as an optimization problem -- they search or evolve workflows heuristically. BWG recasts this as Bayesian inference over a posterior distribution on workflows: `q(s_{1:T} | s_0) proportional to p(s_{1:T} | s_0) * exp[R(s_{1:T})]`, where `p` is the prior (the LLM's natural distribution over workflow steps given a task description `s_0`) and `R` is a reward function measuring task performance. The posterior concentrates mass on workflows that are both plausible (under the LLM's prior knowledge) and effective (high reward).

**How it works in practice:** BWG maintains a pool of N partial workflows. At each step t, it (1) extends each partial workflow by sampling a next step from the LLM, (2) scores each extended prefix by running K parallel look-ahead rollouts to completion and averaging the rewards to compute importance weights `w_i = (1/K) * sum(exp[R(completion_k)])`, and (3) resamples the pool according to normalized weights so promising prefixes are duplicated and poor ones are dropped. This step-level resampling avoids the "weight degeneracy" problem of scoring only complete trajectories. An optional in-loop refiner selects top candidates, applies targeted edits (a single consequential change), and inserts refined workflows back into the pool for diversity.

**Why it matters:** The framework has convergence guarantees -- without the refiner, the weighted empirical distribution provably converges to the target posterior as N grows. In practice, BayesFlow improved accuracy by up to 9 percentage points over SOTA baselines (AFlow, ADAS) across math, QA, knowledge, and coding benchmarks, while using 5-80% fewer tokens. The key practical takeaway: build workflows incrementally, score partial progress via look-ahead, and prune aggressively.

## Step-by-Step Workflow

When a user asks you to design a multi-step LLM workflow using BWG principles:

1. **Define the task and reward signal.** Clarify the end-to-end task (e.g., "answer multi-hop questions", "generate and test code", "research and summarize"). Identify a concrete reward function: exact-match accuracy, F1 score, test pass rate, or user-defined quality criteria. The reward must be computable on validation examples.

2. **Establish the helper interface.** Define the two primitives available to the workflow: (a) `chat_completion(system, instructions, history) -> response` for LLM calls, and (b) `exec_code(code, test_input) -> result` or equivalent tool invocation. Keep the interface minimal -- BayesFlow deliberately avoids predefined agent modules (no hardcoded "Ensemble" or "Revise" nodes) and instead lets the workflow discover its own structure.

3. **Generate an initial pool of N candidate workflow prefixes.** Prompt the LLM to propose 5-15 distinct first steps for the task. Each step should be a self-contained code chunk with explicit `# Step 1:` annotation. Encourage diversity by varying the system prompt slightly (e.g., "propose a direct approach", "propose an approach with self-verification", "propose a chain-of-thought approach").

4. **Extend each prefix by one step.** For each partial workflow, prompt the LLM: "Given the workflow so far [steps 1..t-1], propose Step t." The LLM generates the next meaningful code chunk (an LLM call, a tool invocation, a parsing/validation step, or a conditional branch).

5. **Score via parallel look-ahead rollouts.** For each extended prefix, generate K=3-5 stochastic completions (the remaining steps through the end of the workflow) by prompting the LLM to finish the workflow. Run each completed workflow against validation examples and compute the reward. The importance weight for prefix i is: `w_i = (1/K) * sum(exp[R(completion_k)])`.

6. **Resample the pool.** Normalize the importance weights across all N prefixes. Resample N prefixes with replacement according to these weights. High-scoring prefixes get duplicated; low-scoring ones are dropped. This is the core Bayesian update -- it concentrates the pool on promising partial workflows.

7. **Apply the in-loop refiner (optional but recommended).** Select the top 3 candidates by validation reward. Sample one proportionally (softmax with temperature 0.1). Prompt the LLM: "Make exactly one consequential edit to this workflow to improve it. Preserve the overall structure and code format." Insert the edited workflow back into the pool. This adds diversity and enables global revisions that step-level extension cannot achieve.

8. **Repeat steps 4-7 for T rounds** (typically T=4-8, matching the natural number of steps in the workflow). At each round, collect any workflows that are already complete (all steps generated and functional).

9. **Select the best workflow.** From all completed workflows across all rounds, select the one with the highest validation reward. Optionally, keep the top-3 as an ensemble for even higher accuracy.

10. **Output the final workflow as executable code.** Present the workflow as a clean Python function (or structured pseudocode) with annotated steps, clear LLM call parameters, and error handling. Include the validation score and a brief rationale for why this structure was selected.

## Concrete Examples

**Example 1: Building a multi-hop QA workflow**

User: "Design an LLM workflow that answers multi-hop questions like HotpotQA -- questions requiring information from multiple sources."

Approach:
1. Define reward: F1 score against gold answers on 20 validation questions.
2. Generate initial pool of 5 candidate first-steps with diverse strategies.
3. Iteratively extend, score via look-ahead, and resample over 5 rounds.

Output (best workflow discovered):
```python
def multi_hop_qa(question: str) -> str:
    # Step 1: Decompose the question into sub-questions
    decomposition = chat_completion(
        system="You are a question decomposition expert.",
        instructions=f"Break this multi-hop question into 2-3 independent sub-questions:\n{question}",
    )

    # Step 2: Answer each sub-question independently
    sub_answers = []
    for sub_q in parse_sub_questions(decomposition):
        ans = chat_completion(
            system="Answer the following factual question concisely.",
            instructions=sub_q,
        )
        sub_answers.append((sub_q, ans))

    # Step 3: Synthesize sub-answers into a final answer
    context = "\n".join(f"Q: {q}\nA: {a}" for q, a in sub_answers)
    final = chat_completion(
        system="You synthesize information from multiple sources.",
        instructions=f"Original question: {question}\n\nSub-answers:\n{context}\n\nProvide the final answer.",
    )

    # Step 4: Self-verify and correct
    verification = chat_completion(
        system="You are a critical reviewer.",
        instructions=f"Question: {question}\nProposed answer: {final}\n\nIs this answer correct and complete? If not, provide a corrected answer.",
    )
    return extract_answer(verification)
```
Validation F1: 0.78 (vs 0.64 for single-shot CoT baseline).

**Example 2: Code generation with test-driven self-correction**

User: "Create a workflow for generating Python functions that pass unit tests, like the MBPP benchmark."

Approach:
1. Define reward: pass@1 rate on validation test cases.
2. BWG discovers a 6-step test-centered workflow through 4 rounds of extend-score-resample.

Output (best workflow discovered):
```python
def code_gen_workflow(task_description: str, test_cases: list[str]) -> str:
    # Step 1: Generate initial solution
    solution = chat_completion(
        system="You are an expert Python programmer.",
        instructions=f"Write a Python function for:\n{task_description}",
    )

    # Step 2: Early exit if tests pass
    result = exec_code(solution, test_cases)
    if result.all_passed:
        return solution

    # Step 3: Analyze errors and generate targeted fix
    fix = chat_completion(
        system="You are a debugging expert. Do not use try-except blocks. Fix the root cause.",
        instructions=f"Code:\n{solution}\n\nErrors:\n{result.error_log}\n\nFix the code.",
    )

    # Step 4: Re-test the fix
    result2 = exec_code(fix, test_cases)
    if result2.all_passed:
        return fix

    # Step 5: Rewrite from scratch with error context
    rewrite = chat_completion(
        system="You are an expert Python programmer. Previous attempts failed.",
        instructions=f"Task: {task_description}\n\nPrevious errors: {result2.error_log}\n\nWrite a completely new solution.",
    )

    # Step 6: Final fallback test
    result3 = exec_code(rewrite, test_cases)
    return rewrite if result3.all_passed else fix
```
Validation pass@1: 85.0% (vs 75.5% for AFlow baseline).

**Example 3: Applying BWG principles to optimize an existing workflow**

User: "I have a summarization pipeline that calls the LLM once. Help me improve it using BayesFlow principles."

Approach:
1. Treat the existing single-call pipeline as one candidate in the initial pool.
2. Generate 4 alternative first-steps (e.g., extract key points first, chunk-then-summarize, generate-then-critique).
3. Run 3 rounds of look-ahead scoring on 10 validation documents using ROUGE as reward.
4. Resample and refine.

Output:
```python
# Original (baseline): Single LLM call
# BWG-discovered workflow (3 steps, ROUGE-L improved from 0.34 to 0.41):

def summarize(document: str) -> str:
    # Step 1: Extract key claims and supporting evidence
    extraction = chat_completion(
        system="Extract the 5 most important claims from this document with supporting quotes.",
        instructions=document,
    )

    # Step 2: Draft summary from extracted claims
    draft = chat_completion(
        system="Write a concise summary covering all the key claims below.",
        instructions=extraction,
    )

    # Step 3: Verify coverage and compress
    final = chat_completion(
        system="Review this summary against the original. Fix any missing key points. Remove redundancy. Keep it under 150 words.",
        instructions=f"Original:\n{document[:2000]}\n\nDraft summary:\n{draft}",
    )
    return final
```

## Best Practices

- **Do:** Use a concrete, computable reward function. BWG's power comes from scoring candidates against real validation data. Even 10-20 validation examples dramatically improve workflow selection over pure intuition.
- **Do:** Keep the helper interface minimal (just `chat_completion` and tool execution). Let the workflow discover its own structure -- avoid hardcoding agent roles or module types.
- **Do:** Annotate each step explicitly (`# Step N: description`) so the LLM can reason about workflow structure when extending or refining.
- **Do:** Vary the pool size based on task complexity. Simple tasks (single-domain QA) work with N=5; complex tasks (multi-hop reasoning, code generation) benefit from N=10-15.
- **Avoid:** Scoring only complete workflows end-to-end. The key insight of BWG is step-level resampling -- prune bad prefixes early rather than wasting rollouts on doomed trajectories.
- **Avoid:** Using the refiner without the base sampling loop. The refiner adds diversity but lacks convergence guarantees on its own. Always pair it with importance-weighted resampling.

## Error Handling

- **Look-ahead rollouts fail or timeout:** Set a per-rollout timeout and assign reward 0 to failed completions. The importance weighting naturally downweights prefixes whose completions fail.
- **All candidates score equally (weight degeneracy):** Increase K (number of rollouts per prefix) from 3 to 5-8. If scores are still flat, the reward function may be too coarse -- switch to a more granular metric.
- **Pool collapses to duplicates after resampling:** Increase the refiner budget M (number of refined workflows per round). The refiner's targeted edits restore diversity. A ratio of N=10, M=5 works well.
- **Workflow steps are too large or monolithic:** Prompt the LLM explicitly: "Each step should be a single coherent action (one LLM call, one tool invocation, or one post-processing block). Do not combine multiple actions into one step."
- **Self-correction loops don't converge:** Cap retry loops at 2-3 iterations. BayesFlow's MBPP workflow uses exactly one retry + one fallback rewrite, which balances thoroughness against token cost.

## Limitations

- **Requires validation data.** BWG needs a reward signal computed on held-out examples. For purely creative or open-ended tasks with no ground truth, you must define a proxy reward (e.g., LLM-as-judge scoring), which introduces its own biases.
- **Token cost scales with pool size and rollouts.** N prefixes times K rollouts times T steps means many LLM calls. For a typical configuration (N=10, K=3, T=5), expect ~150 completions during the search phase. This is cost-effective for reusable workflows but expensive for one-off tasks.
- **The discovered workflow is task-specific.** A workflow optimized on math problems will not transfer to code generation. You must re-run BWG for each substantially different task domain.
- **Step-level decomposition assumes sequential structure.** BWG builds workflows left-to-right. It does not natively discover parallel branches or DAG-structured pipelines, though individual steps can contain internal parallelism.
- **Convergence guarantees are asymptotic.** With small N (5-15), the empirical distribution is an approximation. The refiner helps but introduces bounded perturbation (TV distance up to (T-1)*epsilon from the true posterior).

## Reference

**Paper:** "BayesFlow: A Probability Inference Framework for Meta-Agent Assisted Workflow Generation" (Yuan et al., EACL 2026 Findings). arXiv: [2601.22305](https://arxiv.org/abs/2601.22305v1). Look for Algorithm 1 (StepUpdate) and Algorithm 2 (BayesFlow main loop) for the core pseudocode, and Tables 1-2 for benchmark comparisons against AFlow, ADAS, and other baselines.