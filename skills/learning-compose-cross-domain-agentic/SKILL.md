---
name: "learning-compose-cross-domain-agentic"
description: "Generate cross-domain agentic workflows using decompose-recompose-decide composition over reusable capability bases. Use when asked to: 'design a multi-step agent workflow', 'create a pipeline that works across different task types', 'build a reusable agentic workflow', 'compose agent operators for a new domain', 'generate a single-pass workflow for complex tasks', 'orchestrate reasoning-verification-repair agents'."
---

# Cross-Domain Agentic Workflow Generation via Capability Composition

This skill enables Claude to design and generate agentic workflows -- executable operator graphs that orchestrate reasoning, verification, repair, and aggregation steps -- by composing a small set of reusable capability patterns rather than building workflows from scratch for each domain. Based on the CapFlow decompose-recompose-decide framework (Wang et al., 2026), the core insight is that successful workflows across wildly different domains (math, code, QA, science) share latent structural factors like multifaceted analysis, verification/repair loops, and ensemble aggregation. Instead of iteratively refining a workflow through expensive trial-and-error, you identify which capability bases a new task needs, sparsely compose them, and emit a complete workflow in a single pass.

## When to Use

- When the user asks to design a multi-step agentic pipeline for a task that spans reasoning, coding, QA, or other domains
- When building a workflow that must generalize across task types without per-domain customization
- When the user wants to orchestrate multiple LLM calls with verification, repair, and aggregation steps
- When porting an existing workflow from one domain (e.g., math problem solving) to another (e.g., scientific reasoning)
- When the user needs to reduce the cost of workflow search by generating a good workflow on the first attempt rather than iterating
- When composing reusable operator templates (generate, verify, repair, ensemble) into a task-specific graph
- When analyzing why a multi-agent workflow succeeded or failed and attributing outcomes to specific capability components

## Key Technique: Decompose-Recompose-Decide

**Decompose into capability bases.** The central finding of CapFlow is that despite surface differences across domains, successful agentic workflows repeatedly instantiate a small number of underlying capability factors. The paper identifies these recurring patterns empirically: (1) **multifaceted analysis** -- attacking a problem from multiple perspectives in parallel, (2) **verification/repair loops** -- testing outputs and feeding errors back for correction, (3) **ensemble aggregation** -- generating multiple candidate solutions and selecting or merging the best. Rather than treating each domain's workflow as unique, you decompose workflows into these reusable bases. In the paper, K=8 bases with top-3 sparse activation suffice to cover math, code, QA, and science domains.

**Recompose via sparse task-conditioned mixing.** Given a new task, you select and weight a small subset (typically 3) of capability bases to compose the workflow. This is not a rigid template -- the composition weights are task-conditioned, meaning the same bases combine differently for "solve this differential equation" versus "debug this Python function." The sparsity constraint (top-m activation) prevents degenerate collapse to a single base and forces each base to specialize. The result is a complete workflow topology (control flow graph) plus customized operator prompts, generated in a single pass.

**Decide via counterfactual attribution.** After generating and executing a workflow, you evaluate each capability base's marginal contribution: "If I removed this base from the composition, how much would performance drop?" A large positive drop means the base was essential; a negligible drop means it was deadweight. This counterfactual signal is used to refine future compositions -- upweighting bases that causally drive success and pruning those that don't. This avoids the trap of attributing success to superficially correlated but causally irrelevant components.

## Step-by-Step Workflow

1. **Classify the task domain and requirements.** Determine what the task demands: reasoning (multi-hop QA, math), generation (code, text), verification (testing, fact-checking), or a combination. Identify whether the task requires single-answer precision or exploratory multi-solution generation.

2. **Select capability bases from the reusable set.** Choose 2-4 bases from these canonical workflow capabilities:
   - **Multi-perspective analysis**: Generate multiple independent reasoning chains or solution approaches for the same problem
   - **Verification/repair loop**: Execute outputs, check correctness (via tests, validators, or self-consistency), and feed failures back for targeted repair
   - **Ensemble aggregation**: Collect multiple candidate outputs and select the best via voting, scoring, or merging
   - **Decomposition cascade**: Break a complex task into sub-tasks, solve each independently, then compose results
   - **Critique-and-refine**: Generate a draft, produce a structured critique, then revise based on the critique
   - **Evidence retrieval and grounding**: Retrieve supporting evidence before reasoning, anchor conclusions to sources

3. **Assign sparse composition weights.** For the selected bases, assign relative importance weights based on the task. A math proof task might weight verification/repair heavily (0.5) with some multi-perspective analysis (0.3) and light ensemble (0.2). A code generation task might weight verification/repair (0.4), decomposition cascade (0.4), and critique-and-refine (0.2). Keep activation sparse -- use at most 3-4 bases.

4. **Construct the workflow topology as an operator graph.** Map the weighted capability bases into a concrete directed graph of operators. Each node is an LLM call with a specific prompt template. Edges represent data flow (output of one operator feeds into the next). Use control flow constructs: sequential chains, parallel branches, conditional loops (repeat verification/repair until pass or max iterations).

5. **Write customized operator prompts for each node.** Each operator node gets a domain-adapted prompt. A "Generate" node for math gets a chain-of-thought math prompt; the same structural role for code gets a function-implementation prompt. The topology stays the same -- only the prompts change across domains.

6. **Execute the workflow graph.** Run operators in topological order, respecting parallel branches. Collect outputs at each node and pass them forward. For verification nodes, use concrete validators where possible (code execution, unit tests, calculators) rather than LLM-only self-assessment.

7. **Evaluate and attribute outcomes counterfactually.** After execution, assess: did the workflow produce a correct result? For each active capability base, estimate its marginal contribution by asking "would the workflow have succeeded without this component?" If verification/repair was active but never triggered a repair (all outputs passed first try), its marginal contribution was low for this instance -- note this for future composition decisions.

8. **Record and reuse composition patterns.** Maintain a lightweight log of task-type to successful composition mappings. When a similar task arrives, start from the previously successful composition rather than reselecting from scratch. Over time, this builds a task-conditioned routing table that converges on effective compositions per domain.

## Concrete Examples

**Example 1: Building a cross-domain code+math pipeline**

User: "I need a workflow that can handle both solving math word problems and generating Python functions with tests. It should work without me specifying which type each task is."

Approach:
1. Classify: The task spans math reasoning and code generation -- two domains requiring a unified workflow
2. Select bases: Multi-perspective analysis (generate multiple solution attempts), Verification/repair loop (test code, check math answers), Ensemble aggregation (pick best solution)
3. Compose the workflow graph:

```
[Input Task]
     |
[Task Router] -- classifies as math or code
     |                    |
[Math Branch]        [Code Branch]
     |                    |
[Generate x3]       [Generate x3]     <- multi-perspective
     |                    |
[Verify: calc]      [Verify: exec]    <- verification with concrete tools
     |                    |
[Repair if fail]    [Repair if fail]   <- repair loop (max 2 iterations)
     |                    |
[Vote best]         [Vote best]        <- ensemble aggregation
     |                    |
     +----[Merge]--------+
           |
      [Final Output]
```

4. Operator prompts:
   - Math Generate: "Solve this step by step, showing all work: {problem}"
   - Code Generate: "Write a Python function that: {spec}. Include docstring."
   - Math Verify: Execute numerical check or symbolic verification
   - Code Verify: Run generated code against provided or inferred test cases
   - Repair: "The previous solution failed because: {error}. Fix it."
   - Vote: Compare solutions, select the one that passes verification

Output: A reusable workflow definition that handles both domains through shared topology with domain-specific prompts at the leaves.

**Example 2: Porting a QA workflow to scientific reasoning**

User: "I have a multi-hop QA workflow that works well on HotpotQA. I need to adapt it for answering graduate-level science questions (like GPQA) without redesigning from scratch."

Approach:
1. Analyze the existing QA workflow's capability bases: it likely uses decomposition (break multi-hop into sub-questions), evidence retrieval, and aggregation
2. Identify what science reasoning additionally needs: domain-specific verification (dimensional analysis, unit checking), deeper chain-of-thought for derivations
3. Recompose by adding verification/repair base and adjusting weights:

```
Existing QA composition:   [Decompose: 0.4, Retrieve: 0.4, Aggregate: 0.2]
Science composition:       [Decompose: 0.3, Retrieve: 0.2, Verify/Repair: 0.3, Critique: 0.2]
```

4. Modify the workflow graph:
   - Keep the decomposition cascade (break complex science problem into sub-problems)
   - Replace pure retrieval with retrieval + formula/principle lookup
   - Add a verification node that checks dimensional consistency and boundary conditions
   - Add a critique node that reviews the reasoning chain for logical gaps
5. Update operator prompts to reference scientific reasoning patterns

Output: The adapted workflow reuses 60% of the QA topology, adds verification and critique nodes, and handles science questions without full redesign.

**Example 3: Diagnosing a failing workflow via counterfactual attribution**

User: "My agent pipeline generates code, runs tests, repairs failures, and ensembles 3 candidates -- but it's only solving 40% of problems. What's going wrong?"

Approach:
1. Map the existing workflow to capability bases: Generate (multi-perspective x3), Verify (test execution), Repair (error-conditioned retry), Ensemble (vote on 3 candidates)
2. Run counterfactual analysis on recent failures:
   - Remove Ensemble: performance drops to 35% -> Ensemble contributes +5%, modest value
   - Remove Repair: performance drops to 25% -> Repair contributes +15%, high value
   - Remove multi-perspective (use 1 candidate): performance drops to 30% -> Multi-perspective contributes +10%
3. Diagnosis: Repair is the strongest contributor but still leaves 60% unsolved. The bottleneck is likely the Generate step -- if initial generations are fundamentally wrong, repair cannot salvage them
4. Recommendation: Add a **Decomposition cascade** base -- break problems into sub-functions before generating. This addresses root cause (poor initial generation on complex problems) rather than symptom (failed repairs)

Output: Concrete diagnosis showing which capability bases carry weight and which are underperforming, with a targeted composition change to add decomposition.

## Best Practices

- **Do:** Keep the number of active capability bases sparse (3-4 maximum). The paper shows top-3 activation outperforms using all bases -- forced specialization prevents muddy, unfocused workflows.
- **Do:** Use concrete validators in verification nodes wherever possible (code execution, calculators, type checkers) rather than relying on LLM self-assessment, which is unreliable for factual correctness.
- **Do:** Reuse the same workflow topology across domains and vary only the operator prompts. This is the core transferability mechanism -- structure generalizes, content specializes.
- **Do:** Run counterfactual attribution periodically to prune deadweight bases and identify bottleneck components.
- **Avoid:** Building a unique bespoke workflow for every new task. If you're designing from scratch each time, you're not composing -- you're iterating, which is the expensive pattern this approach replaces.
- **Avoid:** Collapsing to a single dominant capability base (e.g., always using verification/repair for everything). If one base dominates all compositions, the others atrophy and cross-domain transfer degrades.
- **Avoid:** Deep nesting of repair loops (more than 2-3 iterations). Diminishing returns set in quickly -- if the approach is fundamentally wrong, more repair iterations won't fix it. Redirect to a different capability base instead.

## Error Handling

- **Workflow generates but produces wrong answers**: Run counterfactual attribution. If removing a base doesn't change the outcome, that base is not contributing -- replace it or adjust composition weights toward bases with higher marginal effect.
- **Verification node passes incorrect outputs**: The validator is too weak. Strengthen it by using execution-based verification (run code, compute math) rather than LLM-based checking. For domains without automatic validators, use self-consistency across multiple independent generations.
- **Workflow topology too complex / high latency**: Reduce parallel branches. Move from 3-way multi-perspective to 2-way. Check if ensemble aggregation is earning its cost via counterfactual analysis -- if marginal contribution is <3%, drop it.
- **New domain has no prior composition data**: Start with the most domain-agnostic composition: [Multi-perspective: 0.3, Verify/Repair: 0.4, Critique: 0.3]. This is the safest default because verification and critique adapt to any domain with minimal prompt changes.
- **Capability bases seem redundant**: Check co-activation patterns. If two bases always activate together with near-identical weights, they may be encoding the same factor. Merge them conceptually and free a slot for a genuinely different capability.

## Limitations

- This approach assumes the task can be decomposed into a graph of discrete LLM calls with clear data flow. Highly interactive, stateful tasks (e.g., long-running dialogues, real-time agent environments) don't fit the operator-graph model well.
- Counterfactual attribution requires running the workflow multiple times with different base configurations, which adds evaluation cost. For one-off tasks, the attribution overhead may not be justified.
- The reusable capability bases are derived from patterns observed across reasoning, coding, math, and QA. Domains with fundamentally different structure (e.g., creative writing, open-ended exploration) may not decompose into these bases cleanly.
- Single-pass generation trades away the ability to discover genuinely novel workflow structures that iterative refinement might find through exploration. If the optimal workflow is structurally unlike anything in the capability basis set, this approach will miss it.
- The effectiveness of sparse composition depends on having a well-chosen basis set. A poorly designed or too-small set of bases will force bad compositions; too large a set causes selection difficulty.

## Reference

**Paper:** [Learning to Compose for Cross-domain Agentic Workflow Generation](https://arxiv.org/abs/2602.11114v1) (Wang et al., 2026). Look for: Section 3 (capability basis decomposition with LoRA factors), Section 3.2 (task-conditioned sparse composer with adaptive temperature), Section 3.3 (counterfactual capability attribution via marginal effects), and Table 1 (single-pass vs. 20-iteration baselines across 8 benchmarks).