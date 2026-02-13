---
name: "wdscaling-parallel-tool-calling-deep"
description: "Scale deep research tasks by issuing parallel tool calls (width) alongside sequential reasoning (depth), following the Wide & Deep agent pattern. Use when: 'research this topic thoroughly', 'find information about X from multiple sources', 'deep dive into this question', 'investigate this from multiple angles', 'do a comprehensive search on X', 'compare sources on this claim'."
---

# Wide & Deep Parallel Tool Calling for Deep Research

This skill enables Claude to tackle complex research questions by scaling both **width** (multiple parallel tool calls per reasoning step) and **depth** (sequential multi-turn reasoning). Instead of issuing one search at a time and reasoning after each result, Claude decomposes queries into multiple parallel searches within a single turn, aggregates results, then decides the next step. This approach—drawn from the W&D framework (Lin et al., 2026)—reduces total turns by ~48%, cuts cost by ~36%, and improves accuracy by enabling source cross-verification and better query decomposition.

## When to Use

- When the user asks you to research a complex question that requires finding information from multiple sources or perspectives.
- When a research query has multiple independent constraints that can be searched separately (e.g., "Which Brazilian referee in the 1990-1994 World Cups issued the most yellow cards in the first 25 minutes?").
- When the user wants you to verify a claim by cross-checking across different sources.
- When a deep-dive investigation is hitting diminishing returns with single sequential searches—switch to parallel to broaden coverage.
- When performing competitive analysis, literature review, or any task where breadth of information gathering matters.
- When the user explicitly asks to search from "multiple angles" or "compare sources."

## Key Technique

**Width vs. Depth.** Traditional research agents scale *depth*: they issue one tool call, read the result, reason, issue the next call, and repeat for many turns. The W&D framework adds a *width* dimension: at each reasoning step, the agent issues **m parallel tool calls** with different query formulations, angles, or target URLs. All results return simultaneously, and the agent aggregates them in a single reasoning phase. This is not multi-agent orchestration—it is a single agent making coordinated parallel calls within one turn.

**Why parallel calls improve results.** Three mechanisms drive the improvement: (1) **Source credibility**—retrieving the same fact from multiple sources lets you pick the most authoritative one (e.g., preferring an official UN report over an unofficial API). (2) **Hallucination detection**—when parallel calls to the same page with slightly different parameters return inconsistent summaries, it flags unreliable extraction. (3) **Query decomposition**—complex multi-faceted questions yield poor results when crammed into a single search string, but decomposing into simpler parallel queries dramatically improves recall.

**Scheduling strategy.** The paper finds that a **descending** (explore-then-exploit) schedule works best: start with more parallel calls (3) in early turns when the search space is wide, then narrow to fewer (1) in later turns when refining a specific answer. This outperforms fixed-width and ascending schedules. When unsure, start wide and taper.

## Step-by-Step Workflow

1. **Analyze the research question.** Identify the core question, its independent sub-constraints, and what types of sources would be authoritative. Determine if the question is multi-faceted (benefits from parallel decomposition) or single-faceted (sequential is fine).

2. **Decompose into parallel search queries.** Break the question into 2-4 independent search angles for the first turn. Each query should target a different facet, time period, source type, or phrasing. Avoid redundant near-duplicate queries—each call should cover distinct ground.

3. **Issue all searches in a single turn.** Use parallel tool calls (multiple `WebFetch`, `Bash` with `curl`, or `web-search` invocations) in one response. Do NOT serialize them—send them all at once so they execute concurrently.

4. **Aggregate and cross-verify results.** In your next reasoning step, compare what each parallel call returned. Look for: agreement across sources (high confidence), contradiction (needs further investigation), and gaps (topics no query covered). Prefer authoritative primary sources over secondary summaries.

5. **Detect unreliable extractions.** If two parallel calls to the same URL return materially different facts, flag the extraction as unreliable. Re-fetch with a more targeted query or try a different source entirely.

6. **Decide: go deeper or go wider.** If results are consistent and sufficient, synthesize the answer. If there are gaps, issue another round of parallel calls targeting the missing information. If contradictions remain, issue targeted verification calls to authoritative sources.

7. **Taper width as you converge.** In early turns, use 3-4 parallel calls (exploration). In middle turns, use 2-3 (focused investigation). In final turns, use 1-2 (pinpoint verification). This descending schedule optimizes the width-depth tradeoff.

8. **Synthesize with provenance.** When presenting findings, cite which source provided which fact. If sources disagreed, state the disagreement and which source you trust more and why.

9. **Know when to stop.** Set a mental budget of turns. If after 6-8 turns with parallel calls you haven't converged, present what you have with confidence levels rather than spiraling indefinitely.

## Concrete Examples

**Example 1: Multi-constraint factual research**

```
User: "Which country had the highest GDP growth rate in Southeast Asia
       between 2015-2020, and what drove it?"

Approach (Turn 1 — width=3, exploration):
  Parallel call 1: web-search "Southeast Asia GDP growth rates 2015 2020 ranking"
  Parallel call 2: web-search "fastest growing economy ASEAN 2015-2020 World Bank data"
  Parallel call 3: web-search "IMF GDP growth Southeast Asia annual 2015 2016 2017 2018 2019 2020"

Aggregation: All three sources point to Cambodia and Vietnam as top
performers. World Bank data gives exact numbers. IMF data confirms.
One source mentions Myanmar but with pre-revision data.

Approach (Turn 2 — width=2, focused):
  Parallel call 1: web-search "Cambodia GDP growth drivers 2015-2020 garment tourism"
  Parallel call 2: web-search "Vietnam economic growth factors 2015-2020 FDI manufacturing"

Approach (Turn 3 — width=1, verification):
  Single call: Fetch World Bank data page for Cambodia GDP growth 2015-2020

Output: "Cambodia averaged 7.0% annual GDP growth from 2015-2019
(pre-pandemic), the highest in Southeast Asia according to World Bank
data. The primary drivers were garment/textile exports, tourism
(Angkor Wat corridor), and construction investment. Vietnam was a
close second at 6.8%. Note: 2020 figures are anomalous due to
COVID-19, with Cambodia dropping to -3.1%."
```

**Example 2: Claim verification across sources**

```
User: "I read that Virginia Tech's out-of-state tuition is $35,000.
       Can you verify this?"

Approach (Turn 1 — width=3, cross-verification):
  Parallel call 1: web-search "Virginia Tech out-of-state tuition 2025-2026 official"
  Parallel call 2: Fetch Virginia Tech admissions cost page directly
  Parallel call 3: web-search "Virginia Tech tuition fees College Board"

Aggregation: Call 1 returns a news article citing $34,972. Call 2
returns the official page showing $35,290 for 2025-2026. Call 3
returns College Board estimate of ~$35,000. The official source
(Call 2) is most authoritative and shows the figure is close but
not exact.

Output: "The $35,000 figure is approximately correct but slightly
low. Virginia Tech's official 2025-2026 out-of-state tuition is
$35,290 per year (per vt.edu). The $35,000 number likely comes
from rounding or refers to a prior year."
```

**Example 3: Decomposing a complex multi-faceted question**

```
User: "Compare the environmental policies of the three largest
       automakers by revenue."

Approach (Turn 1 — width=3, identify targets):
  Parallel call 1: web-search "largest automakers by revenue 2025 ranking"
  Parallel call 2: web-search "top 3 car companies revenue global 2025"
  Parallel call 3: web-search "biggest automotive companies worldwide revenue"

Aggregation: Consensus is Toyota, Volkswagen, Stellantis (or Mercedes
depending on metric). Use Toyota, VW, Stellantis.

Approach (Turn 2 — width=3, parallel per-company research):
  Parallel call 1: web-search "Toyota environmental policy EV targets 2030 carbon neutral"
  Parallel call 2: web-search "Volkswagen environmental sustainability EV strategy 2030"
  Parallel call 3: web-search "Stellantis climate commitment EV roadmap 2030"

Approach (Turn 3 — width=2, fill gaps):
  Parallel call 1: web-search "Toyota hydrogen fuel cell strategy vs EV criticism"
  Parallel call 2: web-search "Volkswagen emissions scandal cleanup progress 2025"

Output: Structured comparison table with sourced claims for each
company's EV targets, carbon neutrality timeline, and controversies.
```

## Best Practices

**Do:**
- Issue parallel calls with genuinely different query angles—vary phrasing, target different source types (official sites, news, databases), or split by sub-question.
- Start wide (3-4 parallel calls) and taper to narrow (1-2) as you converge on an answer. The descending schedule consistently outperforms other strategies.
- Cross-verify critical facts across at least two independent sources before presenting them as established.
- When parallel results contradict each other, explicitly state the disagreement and which source you trust, rather than silently picking one.

**Avoid:**
- Issuing near-duplicate queries that will return the same results (e.g., "GDP growth Asia" and "Asia GDP growth")—each parallel call should cover distinct ground.
- Using parallel calls for simple factual lookups that a single search will resolve. Width scaling helps most for complex, multi-faceted questions.
- Cramming all constraints of a complex question into one giant search string. Decompose into focused parallel queries instead.
- Continuing to scale width in late turns when you already have a strong candidate answer. Switch to single targeted verification calls.

## Error Handling

- **Parallel calls return contradictory facts:** Do not average or guess. Issue a follow-up verification call targeting the most authoritative source (official databases, primary documents). Present the discrepancy if it cannot be resolved.
- **One or more parallel calls fail or return empty results:** Proceed with the successful results. Re-issue failed queries with modified phrasing in the next turn. Do not block on a single failed call.
- **All parallel calls return irrelevant results:** The question may need reframing. Step back, reason about what specific terms or sources would have the answer, and issue a new batch of differently-worded queries.
- **Context window filling up from too many parallel results:** Summarize and discard raw search results after extracting key facts. Carry forward only the synthesized findings, not the full text of every page fetched.
- **Inconsistent extractions from the same page:** The page content may be dynamic or the extraction unreliable. Fetch the page directly and read the raw content rather than relying on AI-summarized snippets.

## Limitations

- **Simple questions don't benefit.** If a single search reliably answers the question, parallel calls add cost without value. Use this technique for multi-faceted or verification-heavy queries.
- **Token cost scales with width.** Each parallel call returns content that consumes context. For very long research sessions, aggressive summarization between turns is essential to avoid context overflow.
- **Parallel calls cannot express dependencies.** If query B depends on the result of query A, they cannot be parallelized. Identify the dependency graph of your sub-questions first—only parallelize truly independent ones.
- **Tool execution environments may serialize.** Some tool environments execute "parallel" calls sequentially under the hood. The reasoning benefits (decomposition, cross-verification) still apply, but wall-clock speedup depends on the execution layer.
- **Not a substitute for deep sequential reasoning.** Some research questions require chained multi-hop reasoning where each step depends on the last. Width scaling complements depth—it does not replace it.

## Reference

Lin, X., Liew, J. H., Savarese, S., & Li, J. (2026). *W&D: Scaling Parallel Tool Calling for Efficient Deep Research Agents.* arXiv:2602.07359. Key insight: a descending width schedule (explore wide, then narrow) with 3 parallel tool calls per early turn achieves 62.2% on BrowseComp with fewer turns, lower cost, and better accuracy than sequential baselines—without multi-agent orchestration.