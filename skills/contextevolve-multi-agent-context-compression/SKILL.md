---
name: "contextevolve-multi-agent-context-compression"
description: "Multi-agent iterative code optimization using context compression. Decomposes optimization into three agents (Summarizer, Navigator, Sampler) that mirror RL state/policy/replay to evolve better code across iterations. Trigger phrases: 'optimize this algorithm iteratively', 'evolve better code', 'multi-agent code optimization', 'compress optimization context', 'iterative code improvement with agents', 'ContextEvolve approach'"
---

# ContextEvolve: Multi-Agent Context Compression for Iterative Code Optimization

This skill enables Claude to iteratively optimize code using the ContextEvolve framework -- a three-agent system that achieves reinforcement-learning-level search efficiency without parameter updates. Instead of brute-force prompting or naive evolutionary search, it decomposes the optimization context into three orthogonal dimensions: semantic state compression (Summarizer), optimization direction distillation (Navigator), and prioritized exemplar retrieval (Sampler). These agents collaborate to pack maximum information into limited context windows, enabling principled code evolution that outperforms baselines by 33% while using 29% fewer tokens.

## When to Use

- When the user asks to iteratively optimize a performance-critical algorithm (sorting, scheduling, load balancing, caching, query optimization)
- When the user wants to evolve code through multiple rounds of generation and evaluation against a measurable metric (latency, throughput, memory usage, accuracy)
- When the user has an existing solution and wants to systematically explore better alternatives without retraining any model
- When context window space is tight and the user needs to compress optimization history efficiently across iterations
- When the user wants to set up a multi-agent pipeline where each agent handles a distinct aspect of code improvement
- When the user asks to apply evolutionary or RL-inspired search to code generation under API-only LLM access

## Key Technique

ContextEvolve establishes a functional isomorphism between reinforcement learning and a training-free multi-agent framework. Three agents decompose the optimization context:

**Summarizer Agent (State Representation):** Converts raw code into concise natural language abstracts that capture both inherited traits from the parent solution and novel modifications in the offspring. This code-to-language abstraction compresses high-dimensional code into dense semantic descriptions, freeing context space. Critically, the summarizer must preserve ancestral traits -- summarizing only novel changes causes an "amnesia effect" that loses valuable inherited properties (10%+ performance drop in ablations).

**Navigator Agent (Policy Gradient):** Analyzes trajectories of parent-child pairs with their score deltas to distill high-level optimization directions. It samples from three trajectory categories: consistent improvement, mixed fluctuation, and consistent decline. The navigator outputs ambiguous directional guidance rather than specific implementation steps -- over-specificity narrows the solution space prematurely (32.7% performance collapse when too specific). Think of it as estimating which direction to explore, not dictating the exact path.

**Sampler Agent (Experience Replay):** Curates a small set of high-value exemplars from the full population buffer based on relevance to the current parent state and navigator guidance. It prioritizes informative semantics over raw score -- failed candidates with novel logic often drive breakthroughs. Restricting to only top-scoring exemplars caused 27.8% performance loss by preventing heuristic discovery. This mirrors prioritized experience replay in RL, where surprising transitions teach more than predictable ones.

## Step-by-Step Workflow

1. **Define the evaluation function.** Write or identify a concrete scoring function that takes generated code and returns a numeric fitness score. This must be deterministic and capture the optimization objective (e.g., execution time, correctness rate, combined weighted metrics). Without measurable feedback, the framework cannot guide search.

2. **Initialize the Evolve Buffer.** Generate 2-3 initial candidate solutions and evaluate each. Store tuples of `(code, score, semantic_abstract)` in a buffer data structure. The semantic abstract is a natural language summary of what the code does and its key design choices.

3. **Select a parent from the buffer.** Choose a candidate from the evolve buffer as the starting point for this iteration. Prefer higher-scoring candidates but maintain diversity -- don't always pick the top scorer.

4. **Run the Summarizer Agent.** Given the parent's existing abstract and any offspring code from the previous iteration, produce an updated natural language summary. The prompt should ask: "Given the parent's description [z_parent] and this new code [c_child], summarize the complete solution -- both inherited design choices and new modifications." Preserve ancestral traits explicitly.

5. **Run the Navigator Agent.** Sample 3-5 trajectory pairs from the buffer -- each pair consisting of (parent_abstract, child_abstract, score_delta). Include a mix of improving, declining, and mixed trajectories. Prompt the navigator: "Analyze these optimization trajectories and describe high-level directions that tend to improve performance. Be directional, not prescriptive -- suggest what kinds of changes help, not specific code." Output is a short paragraph of optimization guidance.

6. **Run the Sampler Agent.** Present the full buffer contents (abstracts and scores) along with the current parent abstract and navigator guidance to the sampler. Prompt: "Select 2-3 exemplars from this population that would be most informative as few-shot references for the next code generation. Prioritize diversity of approach and informativeness over raw score." Return the selected code examples.

7. **Compose the generation context.** Assemble the prompt for code generation by combining: (a) the parent's semantic abstract (not raw code -- this is the compression), (b) the navigator's directional guidance, (c) the sampler's curated exemplars with their scores. This composed context replaces naive concatenation of all previous code.

8. **Generate offspring code.** Prompt the code generator with the composed context and the task specification. The generator produces a new candidate solution informed by compressed state, directional guidance, and curated examples.

9. **Evaluate and update the buffer.** Run the evaluation function on the offspring. Generate a semantic abstract for the new code via the Summarizer. Store `(offspring_code, score, abstract)` in the evolve buffer. Track the best score seen so far.

10. **Iterate or terminate.** Repeat steps 3-9 for a fixed budget of iterations (typically 30-100). Terminate early if the score plateaus for 10+ consecutive iterations. Return the highest-scoring solution from the buffer.

## Concrete Examples

**Example 1: Optimizing a load balancer assignment algorithm**

User: "I have a GPU load balancer that assigns tasks to GPUs using round-robin. I need to optimize it for both speed and balance. Here's my evaluation function that scores solutions 0-100."

Approach:
1. Initialize buffer with the round-robin baseline (score: 42) and two simple variants (greedy-by-load: 38, random: 25). Summarize each.
2. Iteration 1 -- Select round-robin as parent.
   - Summarizer: "Assigns tasks sequentially across GPUs in fixed order. O(1) per assignment. Ignores current load, causing imbalance under heterogeneous workloads."
   - Navigator (from 3 trajectory pairs): "Vectorized operations improve speed. Proportional allocation improves balance. Avoid per-task Python loops."
   - Sampler selects: greedy-by-load (novel logic despite lower score) + round-robin (baseline reference).
3. Generator produces: Snake round-robin with vectorized numpy assignment. Score: 61.
4. Iteration 10 -- Navigator identifies: "Proportional apportionment methods yield balance gains without sacrificing O(1) speed."
5. Generator produces: Largest-remainder proportional allocation with tensor ops. Score: 78.
6. After 50 iterations, best solution scores 91 -- combining proportional allocation with vectorized assignment discovered across separate lineages.

Output (best solution summary):
```
Score: 91/100 (speed: 95, balance: 87)
Algorithm: Largest-remainder proportional apportionment with snake-order
GPU assignment using vectorized numpy operations. O(1) amortized per
task batch. Discovered at iteration 66 by reintegrating speed techniques
from iteration 10 lineage via exemplar retrieval.
```

**Example 2: Iteratively improving a SQL query optimizer heuristic**

User: "Optimize my query reordering heuristic to maximize KV cache hit rate while minimizing reordering latency."

Approach:
1. Start with the user's baseline heuristic. Evaluate: cache_hit_rate=0.34, latency=12ms, combined_score=45.
2. Summarizer: "Greedy frequency-based reordering. Sorts clauses by historical access frequency. Single-pass O(n log n). No awareness of clause dependencies or cache line geometry."
3. Navigator (after 5 iterations): "Dependency-aware ordering improves cache locality. Batch processing amortizes reordering cost. Clause clustering by access pattern shows consistent improvement."
4. Sampler picks: a dependency-graph approach (score 52, novel structure) and a batch-frequency hybrid (score 48, complementary to parent).
5. Generator synthesizes: dependency-aware clause clustering with batch amortization. Score: 67.
6. Continue iterating. The compressed abstracts keep context under 4K tokens per iteration despite accumulating 30+ candidates in the buffer.

Output (iteration log excerpt):
```
Iter 1:  score=45  (baseline)
Iter 5:  score=52  Navigator: "dependency-aware ordering improves locality"
Iter 12: score=67  Merged dependency-graph + batch-frequency approaches
Iter 25: score=74  Navigator: "pre-computed access matrices reduce runtime"
Iter 40: score=81  Best: clause clustering with precomputed affinity matrix
Buffer: 40 candidates, context per iteration: ~3.2K tokens (vs ~18K raw)
```

**Example 3: Setting up the three-agent pipeline for a custom optimization task**

User: "I want to use the ContextEvolve approach to optimize my packet scheduling algorithm."

Approach:
1. Define three prompt templates:
   ```
   SUMMARIZER_PROMPT: "Given the parent solution description: {parent_abstract}
   And this new implementation: {offspring_code}
   Write a 3-5 sentence summary covering: (a) inherited design choices,
   (b) new modifications, (c) key algorithmic properties (complexity, data structures).
   Preserve description of ancestral traits even if unchanged."

   NAVIGATOR_PROMPT: "Analyze these optimization trajectories:
   {for each trajectory: parent_desc -> child_desc, score_change: +/-N}
   What high-level directions tend to improve performance?
   Be directional (e.g., 'reduce memory allocations') not prescriptive
   (e.g., 'use array pooling on line 42'). Output 2-3 sentences."

   SAMPLER_PROMPT: "From this population of solutions:
   {for each: abstract, score}
   Current parent: {parent_abstract}
   Current guidance: {navigator_output}
   Select 2-3 exemplars as few-shot references. Prioritize:
   - Diverse approaches (not just highest scores)
   - Novel logic that could inspire new directions
   - Relevance to current guidance
   Return the selected solution codes."
   ```
2. Initialize the evolve buffer with 2-3 baseline implementations.
3. Wire the pipeline: Parent Selection -> Summarizer -> Navigator -> Sampler -> Context Composition -> Generator -> Evaluation -> Buffer Update.
4. Run for 50 iterations, logging score progression.

## Understanding the RL Isomorphism

The power of ContextEvolve comes from mapping RL concepts to text-space operations. Understanding this mapping helps you tune the framework:

| RL Concept | ContextEvolve Agent | What It Means in Practice |
|---|---|---|
| State encoder | Summarizer | Compresses code into a latent representation (natural language abstract) that the "policy" can act on |
| Policy gradient | Navigator | Estimates which direction to move in solution space by analyzing which changes correlated with score improvement |
| Experience replay buffer | Sampler + Evolve Buffer | Stores and retrieves past experiences, prioritizing informative ones over merely successful ones |
| Policy network | Code Generator | Produces actions (new code) conditioned on the composed state |
| Reward signal | Evaluation function | Provides the scalar feedback that drives the entire loop |

The key insight: you get RL-like sample efficiency (learning from past experience, directed search) without any gradient computation or parameter updates. The "gradients" are natural language directions. The "state encoding" is summarization. The "replay" is exemplar selection. All of it happens in text.

## Best Practices

- **Do:** Keep navigator guidance ambiguous and directional. Saying "vectorized operations tend to improve throughput" is better than "replace the for-loop on line 12 with numpy.vectorize." Over-specificity kills exploration.
- **Do:** Include low-scoring candidates with novel logic in the sampler's consideration set. Breakthroughs often come from failed experiments that contain one good idea.
- **Do:** Have the summarizer preserve ancestral traits in every abstract. Always describe what was inherited, not just what changed. This prevents the amnesia effect where good inherited properties get lost.
- **Do:** Use semantic abstracts (natural language) instead of raw code in the context wherever possible. This is the core compression -- a 200-line function becomes a 3-sentence description that preserves the essential design choices.
- **Do:** Sample trajectories from all three categories (improving, declining, mixed) for the navigator. Decline trajectories teach what to avoid; mixed trajectories reveal tradeoffs.
- **Avoid:** Feeding all raw code from all previous iterations into the context. This is what naive evolutionary methods do and it wastes tokens. The whole point is compressed representation.
- **Avoid:** Restricting the sampler to only top-K scoring exemplars. This creates a greedy trap. Include diverse-scoring examples to maintain exploration breadth.
- **Avoid:** Making the navigator too prescriptive. If it outputs specific code suggestions instead of high-level directions, it narrows the generator's creative space and performance collapses.

## Error Handling

- **Score stagnation:** If the best score hasn't improved for 10+ iterations, reset the navigator by sampling trajectories from a wider range of the buffer history, including early iterations. The search may be stuck in a local optimum.
- **Abstract drift:** If generated code diverges wildly from the task, the summarizer abstracts may have drifted. Re-anchor by including the original task specification in the summarizer prompt and regenerating abstracts for the top-3 buffer entries.
- **Context overflow:** If the composed context exceeds the context window, reduce exemplar count from 3 to 1, shorten abstracts to 2 sentences, and truncate navigator guidance. Compression is the framework's strength -- lean into it.
- **Evaluation failures:** If generated code fails to execute, assign a score of 0 but still generate an abstract describing the attempted approach. Failed attempts with novel ideas are valuable for the navigator and sampler.
- **Degenerate buffer:** If the buffer converges to near-identical solutions, inject a random perturbation or re-generate from scratch with explicit diversity instructions in the generation prompt.

## Limitations

- Requires a **quantitative evaluation function**. If you can't score solutions numerically, the navigator has no trajectory data to analyze. Purely qualitative code improvements (readability, maintainability) don't fit this framework well.
- The three-agent overhead means **3x more LLM calls per iteration** compared to single-prompt approaches. The token savings come from compression, but latency increases. Not suitable for tasks where a single generation attempt suffices.
- Works best on **algorithmic/systems code** with clear performance metrics (latency, throughput, accuracy). Less effective for UI code, configuration, or business logic where "better" is subjective.
- The framework assumes **iterative refinement is possible** -- that small changes to code can yield measurable score differences. Problems with binary pass/fail evaluation (it works or it doesn't) provide no gradient signal for the navigator.
- Navigator guidance quality depends on having **enough trajectory diversity** in the buffer. The first 5-10 iterations produce weaker guidance because the trajectory sample is small.

## Reference

**Paper:** [ContextEvolve: Multi-Agent Context Compression for Systems Code Optimization](https://arxiv.org/abs/2602.02597v1) (Su, Zheng, Li, 2026). Look for Algorithm 1 (the main loop), Table 2 (ablation showing each agent's contribution), and Figure 3 (the load balancing case study showing how separate optimization lineages merge via exemplar retrieval).