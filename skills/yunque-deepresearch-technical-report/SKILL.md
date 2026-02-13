---
name: "yunque-deepresearch-technical-report"
description: "Hierarchical multi-agent deep research framework with dynamic context management and supervisor-based error recovery. Implements Yunque DeepResearch's orchestration pattern: decompose complex research questions into subtasks, route them to specialized tools/sub-agents, fold completed results into semantic summaries, and detect/recover from failures automatically. Use when: 'deep research on this topic', 'investigate this question thoroughly', 'multi-step research task', 'find and synthesize information from multiple sources', 'research agent for complex questions', 'autonomous research workflow'."
---

# Yunque DeepResearch: Hierarchical Multi-Agent Research Orchestration

This skill enables Claude to tackle complex, open-ended research tasks by implementing the Yunque DeepResearch architecture -- a hierarchical framework where a central orchestrator decomposes questions into subtasks, routes them to an atomic capability pool of tools and specialized sub-agents, manages context through semantic folding of completed sub-goals, and recovers from failures via a proactive supervisor module. The key insight is that deep research fails not from lack of capability but from contextual noise accumulation, cascading errors, and rigid pipelines. This framework solves all three.

## When to Use

- When the user asks to research a complex question requiring multiple search queries, source cross-referencing, and synthesis across domains
- When a task requires browsing multiple web pages, extracting data, and reasoning over accumulated findings
- When the user needs an autonomous agent workflow that can recover from dead ends, bad search results, or parsing failures
- When investigating a question that requires 5+ sequential steps with intermediate reasoning (long-horizon tasks)
- When the user asks to "deep research" or "thoroughly investigate" a topic with evidence gathering
- When building a research pipeline that needs to stay coherent across many subtasks without losing track of prior findings

## Key Technique

**Hierarchical Orchestration with Atomic Capability Routing.** The central orchestrator (Main Agent) performs high-level intent recognition and dynamic planning, then routes each subtask to the appropriate tool from an Atomic Capability Pool. Simple operations (search, read, parse) are invoked directly as tools. Complex, multi-step objectives (browser navigation, data analysis) are delegated to specialized sub-agents. This two-tier routing -- lightweight tools for atomic operations, full sub-agents for compound tasks -- prevents over-engineering simple steps while giving complex steps the autonomy they need.

**Dynamic Context Management via Semantic Folding.** This is the critical innovation. Instead of letting the context window fill with raw tool outputs and ReAct traces, the system partitions the research trajectory into memory units -- 4-tuples of (round index, sub-goal description, tool-use logs, incremental summary). When a sub-goal completes, its detailed traces are "folded" into a concise semantic summary. This compresses context from O(t) (linear in total steps) to O(n) (linear in sub-goals), keeping the active context focused on the current sub-goal's fine-grained traces plus folded summaries of everything prior. A folding function decides whether new observations merge into the current memory unit or start a new one.

**Proactive Supervisor for Error Recovery.** Unlike post-hoc reflection, the Supervisor continuously monitors the agent's trajectory for two failure modes: syntactic errors (malformed tool calls, parse failures) and semantic stagnation (repetitive outputs, circular reasoning). On detection, it executes a three-stage recovery: (1) diagnose the root cause from execution history, (2) prune invalid interaction traces from the context window, and (3) re-generate from the pruned state to break the failure loop. This preemptive interrupt mechanism prevents cascading errors before they corrupt downstream reasoning.

## Step-by-Step Workflow

1. **Decompose the research question into sub-goals.** Analyze the user's query for implicit complexity. Break it into 3-7 concrete, sequentially-ordered sub-goals. Each sub-goal should be achievable with 1-3 tool invocations. Write these out explicitly before starting execution.

2. **Initialize the memory structure.** Create a running context document with sections: `Current Sub-goal`, `Active Traces` (detailed step-by-step for current work), and `Completed Findings` (folded summaries of finished sub-goals). This is your working memory.

3. **Route each subtask to the right capability.** For each sub-goal, decide the routing tier:
   - **Direct tool call**: web search, file read, code execution, URL fetch -- use for single-operation tasks
   - **Sub-agent delegation**: spawn a Task agent for compound operations like multi-page browsing, data analysis requiring iteration, or document parsing pipelines

4. **Execute the current sub-goal with fine-grained tracing.** Perform the search/browse/analysis steps. Record each tool invocation and its result in the Active Traces section. Maintain the full ReAct chain (thought-action-observation) for the current sub-goal only.

5. **Fold completed sub-goals into semantic summaries.** When a sub-goal is done, compress its Active Traces into a 2-4 sentence summary capturing: what was found, from which sources, and how it relates to the overall question. Move this summary to Completed Findings. Clear the Active Traces for the next sub-goal.

6. **Monitor for failure signals after each step.** Check for: (a) search queries returning no useful results, (b) repeated similar actions without progress, (c) tool call errors or malformed outputs, (d) contradictory findings that suggest a wrong path. If any signal is detected, proceed to step 7.

7. **Execute supervisor recovery on failure.** Diagnose what went wrong by reviewing the last 2-3 actions. Prune the failed traces from your active context. Reformulate the approach -- try alternative search terms, different sources, or reframe the sub-goal. Do not retry the exact same action.

8. **Synthesize across all folded memories.** After all sub-goals complete, read through the Completed Findings summaries. Identify convergent evidence, contradictions, and gaps. Produce a coherent synthesis that directly answers the original question with citations to sources discovered during research.

9. **Validate the answer against the evidence chain.** Before presenting results, verify that each claim traces back to a specific finding in the folded summaries. Flag any claims that rest on a single source or that required inference beyond what was found.

10. **Present results with provenance.** Structure the final output with: a direct answer, supporting evidence organized by sub-goal, source attributions, and confidence assessment noting any gaps or limitations in the research.

## Concrete Examples

**Example 1: Cross-domain factual investigation**

```
User: "Who was the first person to climb K2 in winter without supplemental oxygen,
and what was their prior mountaineering record?"

Approach:
1. Decompose into sub-goals:
   SG1: Identify the first winter K2 ascent without supplemental oxygen
   SG2: Find the climber's biographical details and prior record
   SG3: Cross-reference with mountaineering databases for verification

2. Execute SG1: Search "K2 winter ascent without supplemental oxygen first"
   -> Active Trace: Found reports about 2021 Nepali team, but need to check
      supplemental oxygen status specifically
   -> Follow-up search: "K2 winter 2021 oxygen use"
   -> Fold SG1: "K2 was first summited in winter on Jan 16, 2021 by a
      10-member Nepali team. Supplemental oxygen was used. No confirmed
      winter ascent without supplemental O2 as of knowledge cutoff."

3. Supervisor triggers: Initial assumption was wrong (someone did it solo
   without O2). Prune that assumption. Reformulate: the answer may be
   "no one has done this yet."

4. Execute SG2 (adjusted): Verify current status of winter K2 without O2
   -> Fold SG2: "As of latest records, winter K2 without supplemental
      oxygen remains unachieved."

Output:
"No climber has yet summited K2 in winter without supplemental oxygen.
The first winter ascent was completed on January 16, 2021, by a team of
10 Nepali climbers, all using supplemental oxygen. This remains one of
the last great challenges in Himalayan mountaineering. [Sources: ...]"
```

**Example 2: Multi-source technical synthesis**

```
User: "Research the current state of RLHF alternatives for LLM alignment
and summarize the top 3 approaches with their tradeoffs."

Approach:
1. Decompose into sub-goals:
   SG1: Survey RLHF alternatives (DPO, KTO, SPIN, IPO, etc.)
   SG2: Deep-dive the top 3 by adoption and benchmark results
   SG3: Analyze tradeoffs (compute cost, data requirements, alignment quality)
   SG4: Synthesize comparison

2. Execute SG1: Search for recent papers and surveys on RLHF alternatives
   -> Spawn Explore agent to search arxiv and ML blogs
   -> Fold SG1: "Major alternatives: DPO (direct preference optimization),
      KTO (Kahneman-Tversky optimization), SPIN (self-play fine-tuning),
      IPO (identity preference optimization), ORPO (odds-ratio preference).
      DPO, KTO, and ORPO have highest adoption based on citation counts
      and industry usage."

3. Execute SG2 for each method (parallel sub-agent delegation):
   -> Task agent 1: Research DPO details and results
   -> Task agent 2: Research KTO details and results
   -> Task agent 3: Research ORPO details and results
   -> Fold each into separate memory units

4. Execute SG3: Compare folded summaries for compute, data, quality axes
   -> Fold SG3: Structured comparison matrix

5. Synthesize: Merge all folded findings into coherent comparison

Output:
Structured comparison table with:
| Method | Compute | Data Needs | Quality | Best For |
Plus 2-3 paragraph narrative on tradeoffs and recommendations.
```

**Example 3: Recovering from a dead-end search**

```
User: "Find the specific algorithm used in TikTok's recommendation engine
for cold-start users."

Approach:
1. SG1: Search for TikTok recommendation algorithm technical details
   -> Search returns mostly blog speculation, no technical papers
   -> Supervisor detects: low-quality results, no authoritative sources

2. Recovery protocol:
   - Diagnose: Direct search terms too broad, proprietary info unlikely
     to be in public papers
   - Prune: Discard speculative blog results
   - Reformulate SG1: Search for ByteDance/TikTok published research
     papers on recommendation systems, specifically cold-start

3. Adjusted SG1: Search "ByteDance cold start recommendation paper"
   -> Find ByteDance's Monolith paper, explore-exploit framework papers
   -> Fold SG1: "ByteDance published Monolith (KDD 2022) for real-time
      recommendation. Cold-start handled via feature hashing and
      collisionless embedding tables."

4. SG2: Verify and enrich with follow-up technical details
   -> Fold SG2: Additional implementation details from paper

Output:
"TikTok's parent company ByteDance published the Monolith system
(KDD 2022) which handles cold-start through [specific details]...
Note: TikTok's exact production system is proprietary; this represents
the closest published research from ByteDance. [Sources: ...]"
```

## Best Practices

- **Do:** Explicitly write out sub-goals before starting execution. The decomposition step is where most research quality is determined. Spend time getting the sub-goals right.
- **Do:** Fold aggressively. Once a sub-goal is complete, summarize it and release the detailed traces. Carrying raw tool outputs forward is the primary cause of context degradation in long research tasks.
- **Do:** Use the two-tier routing deliberately. Simple lookups should be direct tool calls. Only spawn sub-agents for genuinely multi-step compound tasks (multi-page browsing, iterative data analysis).
- **Do:** Record source provenance in every folded summary. A finding without a source is useless for synthesis.
- **Avoid:** Retrying failed actions without reformulation. If a search query failed, changing one word is not recovery. Diagnose why it failed and try a fundamentally different approach.
- **Avoid:** Keeping all sub-goals' raw traces in context simultaneously. This is the "escalating contextual noise" problem the paper specifically solves. Fold completed work.
- **Avoid:** Treating the sub-goal plan as fixed. If early findings reveal the question is different than assumed, re-decompose. Rigid plans cause cascading errors.

## Error Handling

| Failure Mode | Detection Signal | Recovery Action |
|---|---|---|
| Search returns irrelevant results | No results match the sub-goal's intent after 2 queries | Reformulate search terms using different vocabulary; try specialized sources |
| Tool call malformation | Parse errors or unexpected output format | Prune the failed trace; re-execute with corrected parameters |
| Semantic stagnation | Same information retrieved across 3+ consecutive steps | Interrupt; diagnose if the sub-goal is too broad; decompose further or pivot approach |
| Contradictory findings | Two folded summaries disagree on a factual claim | Add a verification sub-goal specifically to resolve the contradiction before synthesis |
| Context window pressure | Folded summaries exceeding useful length | Re-fold: compress existing summaries further, keeping only claims with source attribution |
| Cascading error from bad assumption | Downstream sub-goals produce nonsensical results | Trace back to the faulty assumption in folded memory; prune dependent sub-goals; re-plan from the corrected state |

## Limitations

- **Proprietary/paywalled information**: The framework cannot access gated content. If the answer lives behind a paywall, the supervisor will detect stagnation but cannot solve the access problem. Be transparent about this limitation in results.
- **Real-time data**: Research findings are only as current as the search tools provide. For rapidly evolving topics, flag temporal limitations explicitly.
- **Single-session depth**: Very deep research (50+ sub-goals) will still hit context limits even with folding. For such tasks, recommend breaking into multiple sessions with saved intermediate results.
- **Verification ceiling**: The framework can cross-reference sources but cannot independently verify primary claims. If all available sources share a common upstream error, the framework will propagate it.
- **Subjective questions**: The hierarchical decomposition works best for factual, verifiable research. Opinion-based or deeply subjective questions benefit less from this structured approach.

## Reference

**Paper:** [Yunque DeepResearch Technical Report](https://arxiv.org/abs/2601.19578v1) -- Cai et al., 2026. Look for: the 4-tuple memory unit structure (Section on Dynamic Context Management), the folding function formulation, and the three-stage supervisor recovery protocol (diagnose, prune, re-generate). The ablation study quantifies each component's contribution: removing memory costs -10.4 on BrowseComp; removing the supervisor costs -8.7 on GAIA.