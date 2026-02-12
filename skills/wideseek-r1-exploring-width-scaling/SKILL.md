---
name: "wideseek-r1-exploring-width-scaling"
description: "Decompose broad information-seeking tasks into parallel subtasks using a lead-agent-subagent pattern with isolated contexts and result aggregation. Use when: 'research multiple competitors and build a comparison table', 'gather information about all X and summarize', 'find and compare pricing across providers', 'collect attributes for a list of items', 'build a structured overview from many sources', 'search for multiple things in parallel'."
---

# WideSeek-R1: Width-Scaled Multi-Agent Orchestration for Broad Information Seeking

This skill enables Claude to tackle broad information-gathering tasks -- those requiring data about many entities, attributes, or sources -- by decomposing the work into parallel subtasks assigned to independent subagents. Instead of sequentially searching for each piece of information (depth scaling), Claude acts as a lead agent that identifies the full scope of needed information, crafts targeted subtask prompts for parallel subagents, then aggregates their findings into a structured result. This is the core insight of the WideSeek-R1 paper: width scaling through parallel subagents consistently outperforms deeper single-agent reasoning for tasks that are broad rather than deep.

## When to Use

- When the user asks to **research and compare multiple entities** (e.g., "Compare the pricing, features, and limits of the top 5 cloud storage providers")
- When the user needs a **structured table or matrix** synthesized from information scattered across many sources
- When a task requires **gathering the same set of attributes for a list of items** (e.g., "For each of these 12 libraries, find the license, last release date, and GitHub stars")
- When the user asks to **search for or collect broad information** that naturally decomposes into independent parallel queries
- When a single-pass search would miss items because the task scope exceeds what one query can cover
- When building **competitive analyses, market surveys, feature matrices, or compliance checklists** that span many targets

**Do not use** for single-entity deep research, step-by-step debugging, or tasks where each step depends on the previous step's output.

## Key Technique

### Width Scaling vs. Depth Scaling

Traditional agentic approaches use depth scaling: a single agent reasons across many turns, accumulating context as it searches, reads, and synthesizes. This works well for deep, focused problems but hits a bottleneck on broad tasks -- those requiring information about many independent entities or attributes. Context windows fill with irrelevant prior searches, accuracy degrades, and parallelism is impossible.

WideSeek-R1 flips this by introducing **width scaling**: a lead agent decomposes the broad task into independent subtasks, each handled by a subagent with an **isolated context**. Subagents never see each other's search results, preventing context pollution. The lead agent receives only the final condensed outputs from each subagent (thinking/scratchpad content is stripped), keeping its aggregation context clean. The key finding is that performance scales consistently with the number of parallel subagents -- a 4B parameter model with 10 subagents matches a 671B single-agent model.

### The Orchestration Protocol

The lead agent has exactly one tool: `call_subagent`. It cannot search directly -- its sole job is decomposition and aggregation. Each subagent gets a focused, self-contained prompt and operates with search/retrieval tools independently. This strict separation prevents the lead agent from getting distracted by raw search results and forces it to produce clear, well-scoped subtask definitions. After all subagents complete, the lead agent receives their condensed outputs and synthesizes the final structured answer.

## Step-by-Step Workflow

1. **Analyze the breadth of the request.** Identify the independent dimensions: How many entities? How many attributes per entity? Can information for entity A be gathered independently of entity B? If yes, this is a width-scaling candidate.

2. **Define the output schema.** Before decomposing, establish what the final output looks like -- typically a table with rows (entities) and columns (attributes). This schema guides subtask design and ensures subagents collect compatible information.

3. **Decompose into parallel subtasks.** Write one subtask prompt per independent unit of work. Each prompt must be self-contained: it should include the specific entity/entities to research, the exact attributes to find, and the expected output format. Aim for subtasks that are roughly equal in scope.

4. **Launch subagents with isolated contexts.** Use the Task tool to spawn one agent per subtask. Each agent gets only its subtask prompt -- no shared context from other subtasks. Run all subagents in parallel using concurrent Task tool calls in a single message.

5. **Enforce tool discipline in subtask prompts.** Instruct each subagent to use search/retrieval tools (web search, grep, file reads) and return structured findings. Subagents should make up to 3-5 targeted searches per subtask rather than one broad search.

6. **Collect and condense subagent outputs.** When subagents return, extract only the factual findings -- discard reasoning traces, search queries, and intermediate steps. This prevents context bloat during aggregation.

7. **Aggregate into the target schema.** Map each subagent's findings into the predefined output schema. Flag any cells where subagents returned conflicting or missing information.

8. **Validate completeness and consistency.** Check: Are all rows populated? Are there obvious contradictions between subagents? If gaps exist, spawn targeted follow-up subagents for just the missing cells rather than re-running the entire task.

9. **Format and present the final structured output.** Render the aggregated result as a markdown table, JSON, or whatever format the user requested. Include source attribution where available.

## Concrete Examples

**Example 1: Competitive Feature Matrix**

```
User: "Compare React, Vue, Svelte, and Angular on bundle size, learning curve,
TypeScript support, SSR framework, and GitHub stars."

Approach:
1. Identify 4 entities (frameworks) x 5 attributes = 20 data points.
   Each framework's attributes are independent -- perfect for width scaling.

2. Define schema:
   | Framework | Bundle Size | Learning Curve | TS Support | SSR Framework | GitHub Stars |

3. Spawn 4 parallel subagents, one per framework. Each subtask prompt:
   "Research [Framework]. Find: (a) minimum production bundle size in KB,
    (b) learning curve rating from community consensus, (c) TypeScript support
    level (native/plugin/none), (d) primary SSR framework name,
    (e) current GitHub star count. Return as a single-row markdown table."

4. Collect 4 single-row tables from subagents.

5. Merge into final table, resolving any format inconsistencies.

Output:
| Framework | Bundle Size | Learning Curve | TS Support | SSR Framework | Stars |
|-----------|------------|----------------|------------|---------------|-------|
| React     | ~6 KB      | Moderate       | Via @types | Next.js       | 234k  |
| Vue       | ~16 KB     | Easy           | Native 3.x | Nuxt          | 210k  |
| Svelte    | ~2 KB      | Easy           | Native     | SvelteKit     | 82k   |
| Angular   | ~45 KB     | Steep          | Native     | Angular SSR   | 98k   |
```

**Example 2: Codebase Audit Across Multiple Modules**

```
User: "Audit our monorepo for deprecated API usage. Check packages/auth,
packages/billing, packages/notifications, and packages/analytics."

Approach:
1. Four independent packages to audit -- no cross-dependencies for this task.

2. Define schema: package name, deprecated APIs found, file locations, severity.

3. Spawn 4 parallel subagents. Each prompt:
   "Search packages/[name] for usage of deprecated APIs. Look for:
    (a) deprecated Node.js APIs (Buffer(), url.parse, etc.),
    (b) deprecated library methods from package.json dependencies,
    (c) any @deprecated JSDoc annotations in internal code.
    Return a list of findings with file path, line number, and what to replace."

4. Collect findings from each subagent.

5. Aggregate into a unified audit report sorted by severity.

Output:
## Deprecated API Audit Report

### packages/auth (3 issues)
- `src/token.ts:45` - `new Buffer()` -> `Buffer.from()` [HIGH]
- `src/oauth.ts:112` - `url.parse()` -> `new URL()` [MEDIUM]
- `src/session.ts:78` - `@deprecated validateLegacy()` internal [LOW]

### packages/billing (1 issue)
- `src/stripe.ts:23` - `stripe.charges.create()` -> Payment Intents API [HIGH]

### packages/notifications (0 issues)
No deprecated API usage found.

### packages/analytics (2 issues)
- `src/track.ts:91` - `querystring.stringify()` -> `URLSearchParams` [LOW]
- `src/report.ts:34` - deprecated moment.js -> date-fns suggested [MEDIUM]
```

**Example 3: Multi-Source Data Collection**

```
User: "I need a table of all AWS regions with their region code, location,
number of availability zones, and year launched. Cover all current regions."

Approach:
1. There are 30+ AWS regions. A single-agent sequential approach would
   require many searches and risk losing earlier results from context.

2. Group regions into batches of 5-6 per subagent to balance parallelism
   and per-agent workload. Spawn 6 subagents.

3. Each subtask prompt:
   "Find the following for AWS regions [list of 5-6 region codes]:
    region code, geographic location, number of AZs, and launch year.
    Return as markdown table rows."

4. Collect 6 partial tables and merge into one complete table.

5. Validate: check total count against known AWS region count.
   If any are missing, spawn one targeted follow-up subagent.

Output:
| Region Code      | Location              | AZs | Year |
|------------------|-----------------------|-----|------|
| us-east-1        | N. Virginia           | 6   | 2006 |
| us-east-2        | Ohio                  | 3   | 2016 |
| us-west-1        | N. California         | 2   | 2009 |
| ... (full table with all 30+ regions) ...
```

## Best Practices

**Do:**
- Write self-contained subtask prompts that a subagent can execute without needing context from other subtasks. Include the entity name, the attributes to find, and the output format in every prompt.
- Pre-define the output schema before decomposition. This ensures all subagents return compatible data structures that can be cleanly merged.
- Strip reasoning traces from subagent outputs before aggregation. Only pass factual findings to the aggregation step to keep context clean.
- Use targeted follow-up subagents for gaps rather than re-running everything. If 2 out of 20 cells are missing, spawn 2 small subagents, not 20.

**Avoid:**
- Spawning subagents for tasks that are inherently sequential or have step-to-step dependencies. Width scaling only helps when subtasks are independent.
- Giving the lead agent direct search capabilities. The lead agent's job is decomposition and aggregation -- mixing in raw search results degrades both.
- Creating too many tiny subtasks. Each subagent has overhead; batch related items (e.g., 5 entities per subagent) when individual items require little research.
- Sharing context between subagents. The entire point of isolated contexts is preventing information pollution. If subagent B needs subagent A's output, that's a sequential dependency -- handle it in a second wave, not in parallel.

## Error Handling

| Problem | Solution |
|---------|----------|
| A subagent returns empty or irrelevant results | Retry with a more specific prompt. Add example output format and explicit search terms. |
| Subagent outputs have incompatible formats | Normalize during aggregation. The lead agent should map free-text responses into the predefined schema columns. |
| Too many subtasks exceed available parallelism | Batch subtasks into waves. Run the first wave in parallel, collect results, then run the next wave. |
| Conflicting data across subagents | Flag the conflict in the output. If both subagents researched overlapping entities, prefer the one with a cited source. |
| Subtask decomposition misses entities | After aggregation, validate completeness against the original request. Spawn a "gap-filling" subagent for any missing items. |
| One subagent is dramatically slower than others | Set timeouts. Design subtasks to be roughly equal in scope so no single agent becomes a bottleneck. |

## Limitations

- **Sequential dependencies defeat width scaling.** If step N requires the output of step N-1, this approach provides no benefit. Use standard depth-scaling (single-agent multi-turn) instead.
- **Diminishing returns past ~10 subagents.** The paper shows consistent gains up to 10 parallel subagents, but coordination overhead grows. Batch into groups if you have more than 10 independent subtasks.
- **Aggregation quality depends on subtask prompt clarity.** Vague subtask prompts produce inconsistent outputs that are hard to merge. The lead agent's decomposition quality is the primary bottleneck.
- **Not suitable for tasks requiring holistic understanding.** If the answer depends on seeing all the data together (e.g., "what trend do these 20 data points show?"), a single agent with full context is better. Width scaling excels at collection, not synthesis.
- **Context isolation means no cross-referencing.** Subagent A cannot fact-check against subagent B's findings. Cross-validation requires a second pass after aggregation.

## Reference

**Paper:** [WideSeek-R1: Exploring Width Scaling for Broad Information Seeking via Multi-Agent Reinforcement Learning](https://arxiv.org/abs/2602.04634v1) (Xu et al., 2026)

**Key takeaway:** A small model (4B) with 10 parallel width-scaled subagents matches a single 671B model on broad information tasks. Look for: the lead-agent-subagent architecture (Section 3), the MARL training with dual-level reweighting (Section 4), and the width scaling curves in Figure 4 showing consistent gains with more subagents.