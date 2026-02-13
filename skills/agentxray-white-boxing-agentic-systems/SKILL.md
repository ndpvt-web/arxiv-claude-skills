---
name: "agentxray-white-boxing-agentic-systems"
description: "Reverse-engineer black-box agentic systems into editable, interpretable workflows using search-based reconstruction. Use when the user says 'reconstruct this agent workflow', 'reverse-engineer this pipeline', 'white-box this agentic system', 'explain what this agent chain is doing', 'approximate this black-box agent', or 'build an interpretable surrogate for this system'."
---

# AgentXRay: White-Boxing Agentic Systems via Workflow Reconstruction

This skill enables Claude to reconstruct interpretable, editable surrogate workflows from black-box agentic systems using only input-output access. Based on the AgentXRay framework, it formulates Agentic Workflow Reconstruction (AWR) as a combinatorial search over discrete agent roles and tool invocations, using Monte Carlo Tree Search with Red-Black Pruning to efficiently navigate the space of possible chain-structured workflows. The result is a transparent, modifiable pipeline that approximates the original system's behavior without needing access to its internals.

## When to Use

- When the user has a multi-step agent pipeline (e.g., AutoGPT, CrewAI, LangGraph chain) producing outputs but wants to understand what roles and tools are actually contributing
- When the user asks to "reverse-engineer" or "white-box" an opaque agent system they can only query via API
- When the user needs to build a cheaper, more interpretable surrogate of an expensive agentic pipeline
- When debugging why an agent chain produces certain outputs and the user wants to isolate which steps matter
- When the user wants to prune redundant agent steps from an existing pipeline to reduce token cost
- When reconstructing a competitor or third-party agent system's workflow from observed input-output pairs
- When the user wants to convert an implicit multi-agent collaboration into an explicit, editable sequential workflow

## Key Technique

**Agentic Workflow Reconstruction (AWR)** treats workflow recovery as an optimization problem: given a black-box system that maps task inputs to outputs, find a sequential workflow `s = [s1, s2, ..., sL]` of primitives that maximizes output similarity. Each primitive `sj` is a 4-tuple `(role, model, thought_pattern, toolset)` representing one agent step. The search space is all sequences up to length `Lmax`, drawn from a catalog of available agent roles and tools. The key insight is the **Linearity Hypothesis** -- most deployed agentic systems, regardless of their internal architecture, execute as sequential chains at runtime, making linear reconstruction a practical approximation.

**Monte Carlo Tree Search (MCTS) with Red-Black Pruning** navigates this combinatorial space efficiently. Each tree node represents a partial workflow prefix; edges append one primitive. The search cycles through: (1) selection via UCB balancing exploration/exploitation, (2) expansion adding new agent-tool combinations, (3) simulation by completing the workflow randomly and scoring it, and (4) backpropagation of rewards. The **Red-Black Pruning** mechanism scores each node as `Score(v) = (Q(v)/N(v)) * ((d(v)+1)/(Lmax+1)) * (|C(v)|/M)`, combining quality (average reward), depth (how far into the workflow), and width (branching diversity). Nodes scoring above a quantile threshold are marked **Red** (promising -- exploit via depth refinement); those below are **Black** (unpromising -- explore via width expansion or prune). This reduces token consumption by 8-22% while maintaining or improving reconstruction fidelity.

**Output comparison uses Static Functional Equivalence (SFE)**, a proxy metric measuring structural and semantic similarity between the reconstructed workflow's output and the black-box target output. SFE correlates with human judgment (Spearman rho=0.61, p<0.001) and avoids requiring execution-based evaluation, making it practical for diverse output types including code, text, and structured data.

## Step-by-Step Workflow

1. **Define the black-box interface.** Identify what inputs the target agentic system accepts and what outputs it produces. Collect 5-20 representative input-output pairs as your evaluation dataset `D`. Ensure diverse task coverage.

2. **Build a primitive catalog.** Enumerate candidate agent roles (e.g., "Planner", "Coder", "Reviewer", "Researcher") and available tools (e.g., code_executor, web_search, file_writer, calculator). Each primitive is a tuple `(role, model, thought_pattern, toolset)`. Start with 8-15 primitives covering likely capabilities.

3. **Set search parameters.** Choose `Lmax` (maximum workflow length, typically 3-7 steps), branching cap `M` (children per node, typically 5-10), iteration budget (50-200 MCTS iterations), and pruning quantile `beta` (0.3-0.5 for moderate pruning).

4. **Initialize the MCTS tree.** Create a root node representing the empty workflow prefix. Each expansion generates child nodes by appending one primitive from the catalog to the current prefix.

5. **Run the search loop.** For each iteration: (a) Select a leaf node via UCB traversal, (b) Expand by trying a new primitive, (c) Simulate by completing the workflow with random primitives up to `Lmax`, (d) Execute the complete workflow on a sampled input and compute SFE similarity against the target output, (e) Backpropagate the reward through ancestor nodes.

6. **Apply Red-Black Pruning periodically.** Every `k` iterations (e.g., k=10), compute `Score(v)` for all active nodes. Color nodes Red (score >= threshold) or Black (below). Prune subtrees rooted at consistently Black leaf nodes. Focus subsequent search budget on Red branches.

7. **Extract the best workflow.** After the budget is exhausted, select the complete workflow path with the highest average SFE score. This is your reconstructed white-box surrogate.

8. **Validate on held-out inputs.** Run the reconstructed workflow on inputs not used during search. Compare outputs against the black-box system. Acceptable SFE scores are typically 0.35+ (partial reconstruction) or 0.45+ (strong reconstruction).

9. **Refine and edit.** The reconstructed workflow is now an explicit, modifiable sequence. Remove steps that ablation shows are redundant. Swap agent roles or tools to improve quality. This editability is the core advantage over distillation.

10. **Deploy the surrogate.** Use the reconstructed workflow as a transparent, cheaper replacement or as documentation of the original system's effective behavior.

## Concrete Examples

**Example 1: Reconstructing a Code-Generation Agent Pipeline**

User: "I have a black-box coding agent that takes feature requests and produces Python modules. I can query it but don't know its internal workflow. Help me reconstruct what it's doing."

Approach:
1. Collect 10 feature-request/output pairs from the black-box agent
2. Define primitive catalog:
   - (Planner, gpt-4o, chain-of-thought, {}) -- decomposes the request
   - (Coder, gpt-4o, direct, {code_executor}) -- writes implementation
   - (Reviewer, gpt-4o, critique, {code_executor}) -- reviews and tests code
   - (Refiner, gpt-4o, iterative, {code_executor, file_writer}) -- polishes output
3. Set Lmax=4, M=8, 100 iterations, beta=0.4
4. Run MCTS: discover that the best-scoring workflow is [Planner, Coder, Reviewer, Coder]
5. Validate: the 4-step surrogate achieves SFE=0.43 on held-out inputs

Output:
```
Reconstructed Workflow (SFE: 0.43):
  Step 1: Planner (chain-of-thought, no tools)
          -> Decomposes feature request into subtasks
  Step 2: Coder (direct, tools: [code_executor])
          -> Generates initial implementation
  Step 3: Reviewer (critique, tools: [code_executor])
          -> Runs tests, identifies issues
  Step 4: Coder (direct, tools: [code_executor])
          -> Fixes issues from review

Token savings vs. black-box: ~35% fewer tokens
Editable: You can remove Step 3 for simple features or add a second Reviewer pass for complex ones.
```

**Example 2: Pruning a Bloated Multi-Agent Data Analysis Pipeline**

User: "Our data analysis agent chain has 8 steps and costs too much. Can you figure out which steps actually matter?"

Approach:
1. Collect 15 input-output pairs from the 8-step pipeline
2. Map each existing step to a primitive in the catalog
3. Run MCTS with Lmax=8 but enable aggressive pruning (beta=0.3)
4. Track Red-Black coloring: steps 2, 5, and 7 consistently score Black
5. Ablation confirms removing those steps drops SFE by only 0.02

Output:
```
Original pipeline: 8 steps, ~45K tokens/query
Red-Black Analysis:
  Step 1: DataLoader        [RED]  Score: 0.72 -- essential
  Step 2: SchemaValidator   [BLACK] Score: 0.11 -- redundant
  Step 3: Analyst           [RED]  Score: 0.68 -- essential
  Step 4: Statistician      [RED]  Score: 0.55 -- essential
  Step 5: Formatter         [BLACK] Score: 0.09 -- redundant
  Step 6: Visualizer        [RED]  Score: 0.61 -- essential
  Step 7: SummaryWriter     [BLACK] Score: 0.13 -- redundant (Visualizer covers this)
  Step 8: ReportAssembler   [RED]  Score: 0.58 -- essential

Recommended pruned pipeline: [DataLoader, Analyst, Statistician, Visualizer, ReportAssembler]
Estimated token reduction: ~22%
SFE retention: 0.98 of original
```

**Example 3: Building an Interpretable Surrogate for a Competitor's Agent**

User: "A competitor has an AI tutoring agent. I can send it questions and get responses. Help me understand its workflow structure."

Approach:
1. Query the competitor's system with 20 diverse tutoring questions across subjects
2. Define broad primitive catalog covering tutoring strategies:
   - (SubjectExpert, model, direct, {knowledge_base})
   - (Socratic, model, questioning, {})
   - (Simplifier, model, analogies, {})
   - (Assessor, model, evaluation, {quiz_generator})
   - (Encourager, model, motivational, {})
3. Run MCTS with Lmax=5, 150 iterations
4. Best workflow: [SubjectExpert, Socratic, Simplifier, Assessor]

Output:
```
Reconstructed Tutoring Workflow (SFE: 0.38):
  Step 1: SubjectExpert -- retrieves domain knowledge, sets context
  Step 2: Socratic -- generates guiding questions rather than direct answers
  Step 3: Simplifier -- rephrases complex concepts using analogies
  Step 4: Assessor -- creates a check-for-understanding question

Insight: The competitor's agent likely uses a 4-stage pedagogical
pipeline. No "Encourager" step was detected -- their system is
knowledge-focused rather than motivational.
```

## Best Practices

- **Do:** Start with a diverse primitive catalog that over-covers likely agent roles. It is cheaper to prune irrelevant primitives than to miss the right one.
- **Do:** Collect input-output pairs that span the full capability range of the target system. Reconstruction quality depends on evaluation diversity.
- **Do:** Use Red-Black Pruning aggressively (beta=0.3-0.4) when token budget is limited. The paper shows 8-22% token savings with negligible quality loss.
- **Do:** Validate on held-out data. A workflow that scores well on search data but poorly on new inputs is overfitting to the evaluation set.
- **Avoid:** Setting Lmax too high. Most practical agentic workflows are 3-6 steps. Lmax > 8 explodes the search space without proportional benefit.
- **Avoid:** Using this for systems with highly stochastic outputs (e.g., creative writing agents with high temperature). SFE-based scoring assumes output consistency across runs.
- **Avoid:** Treating the reconstructed workflow as ground truth. It is a behavioral approximation -- the real system may use a different internal architecture that happens to produce similar outputs.

## Error Handling

- **Low SFE scores across all workflows (<0.25):** The primitive catalog likely lacks a critical capability. Expand the catalog with more agent roles or tools, then re-run search.
- **Search stagnation (no improvement after 50+ iterations):** Increase branching cap `M` or reduce `Lmax` to focus on shorter, more achievable workflows.
- **Token budget exhaustion before convergence:** Enable more aggressive pruning (lower beta), reduce Lmax by 1, or reduce the evaluation dataset size.
- **Inconsistent black-box outputs:** If the target system produces different outputs for the same input across runs, average SFE over 3 queries per input to stabilize scoring.
- **All nodes colored Black:** The quantile threshold is too high. Lower beta to 0.2 to retain more candidates.

## Limitations

- **Sequential-only reconstruction.** The Linearity Hypothesis restricts surrogate workflows to chains. Systems with genuine parallel execution, conditional branching, or recursive loops cannot be faithfully captured.
- **Output-level only.** Reconstruction matches observable outputs, not internal reasoning. Two very different architectures can produce identical outputs -- the surrogate may not reflect the true mechanism.
- **Requires multiple queries.** You need 50-200+ queries to the black-box system during search. Rate-limited or expensive APIs make this costly.
- **SFE metric limitations.** Structural/semantic similarity is an imperfect proxy for functional equivalence, especially for tasks where small output differences have large downstream impact (e.g., security-critical code).
- **Primitive catalog dependence.** The reconstruction can only use primitives you define. If the target system uses an agent role or tool absent from your catalog, that capability will be approximated poorly or missed entirely.

## Reference

[AgentXRay: White-Boxing Agentic Systems via Workflow Reconstruction](https://arxiv.org/abs/2602.05353v2) -- Shi et al., 2026. Focus on Section 3 (AWR formalization), Section 4 (MCTS + Red-Black Pruning algorithm), and Table 1 (benchmark results across five domains showing 0.426 average SFE vs. 0.339 for the best baseline).