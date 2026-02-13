---
name: "darwin-dynamic-agentically-rewriting"
description: "Evolutionary multi-agent code optimization using genetic algorithms. Agents mutate each other's training/configuration code, benchmark results, and select survivors across generations. Use when: 'evolve my training config', 'optimize this code with genetic search', 'set up evolutionary hyperparameter tuning', 'multi-agent code mutation pipeline', 'self-improving training loop', 'darwin-style evolutionary optimization'."
---

This skill enables Claude to implement DARWIN-style evolutionary optimization pipelines where multiple independent agents iteratively mutate, benchmark, and select code variants using a genetic algorithm structure. Rather than tuning parameters by hand or running grid search, you set up a population of code variants that evolve: agents rewrite each other's source code (training scripts, configs, pipelines), run benchmarks in isolated environments, and a selection step retains only the top performers for the next generation. Persistent JSON memory tracks what changes correlated with improvements, so mutations become increasingly informed over iterations.

## When to Use

- When the user wants to automatically optimize training scripts, hyperparameters, or pipeline configurations through iterative search
- When the user asks to "evolve" or "genetically optimize" code across multiple variants in parallel
- When the user needs a self-improving loop where code changes are benchmarked and the best variants survive
- When the user wants multi-agent code mutation where agents propose changes to each other's implementations
- When the user has a working baseline (e.g., a nanoGPT training script, a model config, a data pipeline) and wants to explore the optimization landscape automatically
- When the user asks for an alternative to manual hyperparameter tuning that leverages LLM reasoning about code structure

## Key Technique

DARWIN treats code optimization as an evolutionary process. A **population** of N code variants (typically 10) is maintained, each in an isolated working directory. Each generation, variants are paired and an LLM agent rewrites targeted sections of one variant's code — adjusting hyperparameters, refactoring data loading, changing optimization strategies, or restructuring model architecture code. This is the **mutation** step. The mutation is not random: the LLM receives the full code context, chunked by regex into imports, class definitions, and function bodies, along with a memory log of what previous changes helped or hurt. Each chunk has a mutation probability (default 0.3), and module-level imports are preserved to maintain dependency consistency.

After mutation, all variants **train in parallel** in isolated containers or directories. A standardized benchmark evaluates each variant on metrics like perplexity and model FLOPS utilization (MFU). The **selection** step retains the top K performers (typically 4 out of 10) as parents for the next generation. Failed variants get one error-correction attempt before being dropped. The population is replenished to N by sampling from survivors before the next mutation cycle begins.

The critical differentiator is the **persistent JSON memory system**. Every mutation is logged with timestamps, file paths, LLM-generated summaries of changes, and the resulting performance delta. This memory is fed back into mutation prompts, creating an increasingly informed search. Ablation experiments showed removing memory degraded performance by ~3%. A bidirectional human-in-the-loop interface lets agents request structural changes (new datasets, libraries, file reorganization) that require human approval, keeping the system safe while allowing it to break out of local optima.

## Step-by-Step Workflow

1. **Define the baseline**: Identify the code to optimize (training script, config file, pipeline). Establish a working version that runs successfully and produces measurable benchmark metrics (loss, perplexity, throughput, accuracy, latency, etc.).

2. **Set up the population structure**: Create N isolated working directories (e.g., `agents/agent_00/` through `agents/agent_09/`), each containing a copy of the baseline code. Use Docker containers or separate virtualenvs if mutations might introduce dependency changes.

3. **Implement the code chunker**: Write a regex-based parser that segments each target source file into chunks — module-level imports (frozen by default), class definitions, and function bodies. This allows selective mutation within token limits and preserves structural integrity.

4. **Build the mutation prompt**: For each chunk eligible for mutation (probability 0.3), construct a prompt containing: (a) the chunk source code, (b) global variables and file structure metadata, (c) summaries of other already-mutated chunks in this pass, and (d) relevant entries from the memory log showing what prior changes improved or degraded performance.

5. **Execute mutations via LLM**: Send mutation prompts to the LLM (Claude, GPT-4o-mini, etc.). The LLM returns modified code for each chunk. Reassemble the full source file, preserving import blocks. Write the mutated code to the agent's working directory.

6. **Run parallel training/benchmarking**: Execute each agent's code in its isolated directory with standardized parameters. Capture benchmark metrics (perplexity, MFU, loss, accuracy, wall-clock time). If a variant crashes, send the error traceback back to the LLM for one correction attempt; drop the variant if it fails again.

7. **Log results to memory**: For each agent, write a JSON entry recording: timestamp, file path modified, LLM summary of changes made, benchmark metrics achieved, and the delta from baseline/parent. Append to the persistent memory file.

8. **Select survivors**: Rank all agents by the target metric. Retain the top K (e.g., 4 out of 10) as parents for the next generation. Replenish the population to N by copying code from randomly sampled survivors into empty agent slots.

9. **Iterate**: Repeat steps 4-8 for the target number of generations. Monitor for convergence (diminishing improvement deltas) or divergence (increasing error rates).

10. **Extract and report the best variant**: After all generations complete, identify the top-performing agent. Diff its code against the original baseline to produce a human-readable summary of all beneficial changes discovered.

## Concrete Examples

**Example 1: Evolving a nanoGPT training script**

User: "I have a nanoGPT training script that gets 38.5 perplexity. Can you set up an evolutionary loop to optimize it?"

Approach:
1. Copy the baseline `train.py` into 10 agent directories
2. Parse `train.py` into chunks: imports, `get_batch()`, model config block, training loop, optimizer setup
3. For each generation, mutate chunks like batch construction, learning rate schedule, dropout values, and optimizer parameters
4. Train each variant for 100 iterations, measure perplexity and MFU
5. Select top 4, replenish to 10, repeat for 5 generations
6. Log all mutations and results to `memory.json`

Output structure:
```
darwin_workspace/
  memory.json                    # Persistent mutation/performance log
  baseline/train.py              # Original unmodified script
  agents/
    agent_00/train.py            # Mutated variant
    agent_01/train.py
    ...
    agent_09/train.py
  results/
    generation_0.json            # Per-generation benchmark results
    generation_1.json
    ...
    best_diff.patch              # Diff of best variant vs baseline
    summary.md                   # Human-readable optimization report
```

Sample `memory.json` entry:
```json
{
  "generation": 2,
  "agent_id": "agent_03",
  "timestamp": "2026-02-13T14:22:01Z",
  "file": "train.py",
  "chunk": "get_batch",
  "change_summary": "Replaced dual np.memmap calls with single shared memmap, added prefetch buffer of 4 batches",
  "parent_perplexity": 38.12,
  "result_perplexity": 37.85,
  "delta": -0.27,
  "mfu": 0.401
}
```

**Example 2: Optimizing a data pipeline config**

User: "I have a Spark ETL pipeline config that takes 45 minutes. Set up a DARWIN-style evolutionary search over the config parameters."

Approach:
1. Identify the mutable config surface: partition count, shuffle settings, executor memory, batch sizes, compression codec, join strategies
2. Create 8 agent directories each with a copy of `pipeline_config.yaml`
3. Mutation prompt includes the config, the pipeline DAG structure, and memory of prior changes
4. Each variant runs a scaled-down benchmark (10% data sample) measuring wall-clock time and resource utilization
5. Select top 3, replenish to 8, iterate for 4 generations
6. Report the best config with a diff and projected full-scale runtime

Output:
```
Generation 0: Best runtime 4m32s (baseline 4m30s on 10% sample)
Generation 1: Best runtime 4m18s — increased parallelism in shuffle stage
Generation 2: Best runtime 3m55s — switched to broadcast join for small tables
Generation 3: Best runtime 3m48s — reduced partition count post-filter, added compression
Final improvement: ~15% reduction in runtime on benchmark sample
```

**Example 3: Multi-agent prompt optimization**

User: "I have a classification prompt that gets 72% accuracy. Can we evolve it?"

Approach:
1. Treat the system prompt as the "code" to mutate — chunk it into instruction section, few-shot examples, and output format specification
2. Create 6 prompt variants in isolated test harnesses
3. Each mutation pass rewrites one chunk, informed by memory of which phrasing changes improved accuracy on a held-out eval set
4. Benchmark each variant against 100 eval examples, measuring accuracy and consistency
5. Select top 2, replenish to 6, iterate for 6 generations
6. Return the best prompt with a changelog of effective mutations

Output:
```
Baseline accuracy: 72%
Gen 1 best: 74% — added explicit negative examples to few-shot section
Gen 3 best: 77% — restructured instructions as numbered constraints
Gen 5 best: 79% — replaced vague "classify appropriately" with decision tree
Final prompt saved to best_prompt.txt with full mutation history
```

## Best Practices

- **Do:** Keep module-level imports frozen during mutation to prevent dependency breakage. Only modify them through explicit HITL requests.
- **Do:** Use isolated directories or containers for each agent variant so a bad mutation cannot corrupt other agents or the baseline.
- **Do:** Feed the memory log into mutation prompts — the paper's ablation showed ~3% degradation without it. More context leads to smarter mutations.
- **Do:** Set a reasonable mutation probability (0.2-0.4). Too high causes chaos; too low causes stagnation. Adjust based on the complexity of the code being mutated.
- **Do:** Always preserve the original baseline in a read-only directory for comparison and rollback.
- **Avoid:** Mutating everything at once. Chunk the code and mutate selectively so you can attribute performance changes to specific modifications.
- **Avoid:** Running without error recovery. A 37% error rate is typical — always give failed variants one correction attempt before dropping them.
- **Avoid:** Skipping the selection step or using too large a survivor ratio. Keeping 4 out of 10 (40%) maintains diversity while applying meaningful selection pressure.

## Error Handling

- **Mutation produces invalid syntax**: Send the error traceback and the mutated code back to the LLM with a correction prompt. Allow exactly one retry. If it fails again, drop the variant and replenish from a survivor. Log the failure pattern in memory so future mutations avoid it.
- **Training crashes mid-run**: Capture the exception, check for common issues (OOM, NaN loss, missing files). If OOM, the correction prompt should suggest reducing batch size or model dimensions. Log the crash cause in memory.
- **All variants regress**: If no variant in a generation beats the parent, carry the parents forward unchanged. Consider reducing mutation probability or narrowing the mutation scope to fewer chunks.
- **Memory file grows too large**: Summarize older entries (keep only the change summary and delta, drop raw metadata) or implement a sliding window that retains only the last N generations of detailed entries.
- **Population collapse** (all agents converge to identical code): Inject diversity by resetting 30-50% of the population to the original baseline or to random earlier-generation variants from the memory log.

## Limitations

- **Costly at scale**: Each generation requires N full training/benchmark runs plus N LLM API calls for mutations. Budget and time constraints are real — the original paper used only 5 generations of 10 agents with 100-iteration training runs.
- **Modest improvements on simple tasks**: The paper achieved 2.07% perplexity improvement on a small nanoGPT setup. The approach is more valuable for complex configuration spaces where the search landscape is large and non-obvious.
- **High error rate**: 37.5% of mutations produced code that failed to train, and only 16.67% of those errors were auto-resolved. Expect significant waste in compute budget.
- **Attribution is noisy**: When multiple chunks mutate simultaneously, it is hard to determine which change caused improvement. Single-chunk mutation per generation gives cleaner signal but slower exploration.
- **Not a substitute for domain expertise**: The LLM mutations are guided by code structure and memory, but lack deep understanding of optimization theory. Human review of proposed changes remains valuable.
- **Security concerns**: Agents modifying code can introduce harmful changes. The paper recommends Docker isolation and manual review. Never run evolutionary code mutation on production systems without sandboxing.

## Reference

[DARWIN: Dynamic Agentically Rewriting Self-Improving Network](https://arxiv.org/abs/2602.05848v1) — Henry Jiang, 2026. Focus on Section 3 (System Design) for the mutation/selection architecture, Section 4 (Experiments) for benchmarking methodology, and the memory system design for implementing informed mutations.