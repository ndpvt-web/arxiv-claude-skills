---
name: "why-deep-research-agent"
description: >
  Audit and diagnose hallucinations in multi-step AI research agent workflows using the PIES taxonomy
  (Planning/Summarization x Explicit/Implicit). Decomposes agent trajectories into atomic sub-queries,
  actions, and claims, then systematically detects fabrication, misattribution, noise domination,
  action deviation, and restriction neglect. Use this skill when:
  - "audit my research agent for hallucinations"
  - "why is my deep research pipeline producing wrong answers"
  - "evaluate the reliability of my agent's research trajectory"
  - "diagnose hallucination propagation in my multi-step agent"
  - "check if my agent is ignoring retrieved information"
  - "find where my research agent goes off track"
---

# Why Deep Research Agent: Trajectory-Level Hallucination Diagnosis

This skill enables Claude to audit multi-step research agent workflows for hallucinations using the PIES taxonomy from Zhan et al. (2026). Instead of only checking final outputs, this approach decomposes the full research trajectory — every plan, search, and summary step — into atomic units and classifies failures across four dimensions: Explicit Planning errors (flawed actions), Implicit Planning errors (ignored constraints), Explicit Summarization errors (fabricated or misattributed claims), and Implicit Summarization errors (noise domination over relevant results). This process-aware evaluation catches cascading failures that end-to-end checks miss entirely.

## When to Use

- When a user reports their research agent produces plausible-sounding but incorrect final answers and wants to know *where* in the pipeline it breaks down
- When building or debugging a plan-search-summarize loop (e.g., a LangChain/LangGraph/AutoGPT deep research pipeline) and intermediate steps seem unreliable
- When reviewing agent logs or traces to determine if the agent fabricated claims, cited wrong sources, repeated redundant searches, or silently dropped user constraints
- When designing guardrails or verification layers for a multi-step retrieval-augmented agent
- When the user wants to evaluate whether their agent suffers from anchoring bias (over-relying on early results) or homogeneity bias (preferring redundant over novel information)
- When comparing multiple research agent architectures for reliability across the full trajectory, not just final output quality

## Key Technique: The PIES Taxonomy

The core insight is that deep research agents fail in **four distinct ways**, mapped across two axes. The **functional axis** separates Planning (deciding what to search) from Summarization (synthesizing what was found). The **error property axis** separates Explicit errors (verifiable against sources) from Implicit errors (subtle omissions detectable only by structural analysis).

The four failure modes are: **(1) Claim Hallucination** (Explicit Summarization) — the agent fabricates facts or cites documents that don't support the claim. **(2) Noise Domination** (Implicit Summarization) — the agent retrieves relevant information but ignores it, letting irrelevant content dominate the output. **(3) Action Hallucination** (Explicit Planning) — the agent generates search queries that deviate from the user's intent, redundantly repeat prior searches, or propagate earlier fabrications into new plans. **(4) Restriction Neglect** (Implicit Planning) — the agent formulates executable plans that silently ignore user-specified constraints like time ranges, geographic scope, or domain restrictions.

The critical diagnostic finding is **hallucination propagation**: a single fabricated claim in an early summarization step poisons downstream planning, which then retrieves irrelevant information, which then produces more hallucinations. Proprietary agents exhibit early-stage cascading (>57% of source errors in the first iterations), while open-source agents suffer late-stage collapse from long-context degradation. This means fixing the *first* error in a chain has outsized impact on overall reliability.

## Step-by-Step Workflow

1. **Decompose the trajectory into atomic units.** Parse the agent's logs, traces, or execution history into three unit types: (a) atomic sub-queries extracted from the user's original request, (b) atomic actions (each individual search/tool call the agent made), and (c) atomic claims from each intermediate and final summary, preserving any citation links.

2. **Map each unit to a PIES quadrant.** Label every atomic action as a Planning unit and every atomic claim as a Summarization unit. This creates the functional axis. The error property axis (Explicit vs. Implicit) is determined during verification in the following steps.

3. **Verify claims against sources (Explicit Summarization — H_ES).** For each claim, retrieve the cited document chunk and check support using NLI or direct comparison. If unsupported, classify as fabrication (no source supports it) or misattribution (the cited source doesn't, but another might). Compute H_ES as the ratio of hallucinated claims to total claims.

4. **Measure noise domination (Implicit Summarization — H_IS).** Cluster retrieved chunks by semantic similarity. Rank clusters by relevance to the query. Identify which clusters the agent actually used versus ignored. Penalize ignoring high-relevance clusters in favor of lower-ranked content. Score 0 (all relevant info used) to 1 (relevant info completely ignored).

5. **Audit actions for deviation, redundancy, and propagation (Explicit Planning — H_EP).** For each action, check: Does this search query relate to the user's intent? (deviation) Has this exact or near-identical query been issued before? (redundancy) Is this action logically sound but grounded in a previously fabricated claim? (propagation). Compute H_EP as the ratio of flawed actions to total actions.

6. **Check restriction coverage (Implicit Planning — H_IP).** Extract all constraints from the user's original query (temporal bounds, domain filters, geographic scope, entity restrictions). For each action, score which constraints it effectively addresses. Use the elbow method to separate "addressed" from "neglected" constraints. Compute H_IP as the proportion of unaddressed restrictions.

7. **Compute the composite hallucination score.** Average the four normalized scores: `H_composite = (H_ES + H_IS + H_EP + H_IP) / 4`. Break down by quadrant to identify the dominant failure mode.

8. **Trace propagation chains.** For each hallucinated claim, check if it was referenced by a subsequent planning action. For each flawed action grounded in a hallucinated claim, mark it as a propagation error. Map the causal chain to identify the earliest root error — this is the highest-leverage fix point.

9. **Diagnose cognitive biases.** Check for anchoring bias by plotting utilized-chunk counts across trajectory stages (do they drop sharply after early stages?). Check for homogeneity bias by comparing the size of utilized vs. ignored clusters (does the agent prefer redundant large clusters over novel singleton insights?).

10. **Generate actionable recommendations.** Based on the dominant failure quadrant and propagation analysis, recommend targeted architectural fixes: claim verification gates after each summarization step, constraint checklist enforcement in planning, retrieval diversity mechanisms, or context window management strategies.

## Concrete Examples

**Example 1: Debugging a LangGraph Research Agent**

```
User: My deep research agent keeps getting facts wrong in its final report.
      Here are the agent traces from the last 5 runs. Can you diagnose where
      it's hallucinating?

Approach:
1. Parse each trace into atomic sub-queries, actions, and claims
2. For each claim in intermediate summaries, verify against the retrieved
   documents the agent actually fetched
3. Found: 40% of claims in Step 2 summaries are fabrications (H_ES = 0.40)
4. Traced propagation: 3 of those fabricated claims became search queries
   in Step 3, leading to irrelevant retrieval
5. Planning scores are clean (H_EP = 0.05) when not counting propagation,
   but propagation errors push it to H_EP = 0.25
6. Root cause: the summarization LLM call after the first retrieval round
   has no verification gate

Output:
## Hallucination Diagnosis Report

| Quadrant                  | Score | Severity |
|---------------------------|-------|----------|
| Claim Hallucination (ES)  | 0.40  | HIGH     |
| Noise Domination (IS)     | 0.12  | LOW      |
| Action Hallucination (EP) | 0.25  | MEDIUM   |
| Restriction Neglect (IP)  | 0.08  | LOW      |
| **Composite**             | 0.21  | MEDIUM   |

**Root Cause:** Fabrication in early summarization (Step 2) propagates
into planning (Step 3), causing a cascade. 60% of action hallucinations
are propagation errors, not independent failures.

**Fix:** Add a claim verification gate between summarization and the
next planning step. Use NLI to check each atomic claim against the
retrieved chunks before passing the summary to the planner.
```

**Example 2: Evaluating Constraint Adherence**

```
User: I asked my research agent to find "peer-reviewed studies from 2020-2024
      on transformer efficiency in edge devices, excluding survey papers."
      It returned results but I suspect it ignored some of my constraints.

Approach:
1. Extract atomic restrictions: [peer-reviewed], [2020-2024], [transformer
   efficiency], [edge devices], [exclude surveys]
2. Parse agent's search actions from trace logs
3. Score each action against each restriction using relevance ranking
4. Apply elbow method to separate addressed vs. neglected constraints

Output:
## Restriction Neglect Analysis (H_IP)

| Restriction            | Addressed? | Coverage |
|------------------------|------------|----------|
| peer-reviewed          | NO         | 0.10     |
| 2020-2024              | PARTIAL    | 0.55     |
| transformer efficiency | YES        | 0.92     |
| edge devices           | YES        | 0.88     |
| exclude surveys        | NO         | 0.05     |

**H_IP Score: 0.42** (HIGH — 2 of 5 restrictions effectively neglected)

The agent never filtered for peer-reviewed status and did not exclude
survey papers. The date constraint was only partially enforced — 3 of 8
search actions included date filters.

**Fix:** Implement a restriction checklist that the planner must verify
before each action. Enforce hard filters (date range, document type)
at the retrieval layer rather than relying on the LLM to remember them.
```

**Example 3: Diagnosing Anchoring Bias in Retrieval**

```
User: My agent retrieves good sources but the final summary seems to only
      reflect the first few things it found. Later, better sources are ignored.

Approach:
1. Parse trajectory into retrieval stages (iterations of the search loop)
2. Cluster all retrieved chunks by semantic similarity, rank by relevance
3. Track which clusters were utilized in the final summary vs. ignored
4. Plot utilization rate by retrieval stage

Output:
## Cognitive Bias Analysis

**Anchoring Bias Detected:**
- Stage 1 retrieval: 85% of chunks utilized in final summary
- Stage 2 retrieval: 42% utilized
- Stage 3 retrieval: 11% utilized (despite higher avg relevance)

**Homogeneity Bias Detected:**
- Average utilized cluster size: 4.2 chunks (redundant)
- Average ignored cluster size: 1.3 chunks (novel/unique)
- The agent systematically prefers information it has seen multiple
  times over unique high-quality singleton findings.

**Noise Score (H_IS): 0.38** — substantial relevant information ignored.

**Fix:** Implement retrieval-stage weighting that compensates for recency
neglect. Add a diversity penalty that forces the summarizer to integrate
at least one claim from each retrieval stage. Consider a sliding context
window that deprioritizes early-stage chunks in later iterations.
```

## Best Practices

- **Do:** Decompose trajectories into the smallest atomic units (one claim, one action, one constraint) before analysis. Coarse-grained analysis misses propagation chains.
- **Do:** Always trace propagation — a high planning error score that's actually caused by summarization fabrication needs a different fix than an independent planning failure.
- **Do:** Check for cognitive biases (anchoring, homogeneity) even when hallucination scores are moderate. These biases degrade output quality without producing outright falsehoods.
- **Do:** Verify claims using a two-round approach: first against cited sources, then against the full retrieval history if unsupported. This distinguishes fabrication from misattribution.
- **Avoid:** Relying solely on final-output evaluation. A correct final answer can mask severe intermediate hallucinations that will fail on harder queries.
- **Avoid:** Treating all four PIES quadrants equally when prioritizing fixes. Explicit Summarization errors (fabrication) are the most common root cause of propagation cascades — fix those first.

## Error Handling

- **Incomplete traces:** If the agent doesn't log intermediate summaries, you can only evaluate Planning (EP, IP) and final Summarization. Flag this as a gap and recommend adding intermediate logging.
- **No citations in summaries:** Without citation links, you cannot distinguish fabrication from misattribution. Treat all unsupported claims as fabrication and recommend the agent attach source citations to every claim.
- **Ambiguous constraints:** If the user's original query has implicit constraints (e.g., "recent" without specifying dates), document the assumed restriction boundaries before scoring. Ask the user to clarify.
- **Short trajectories:** For single-step research (one search, one summary), propagation analysis is not applicable. Focus on H_ES and H_IP only.
- **Contradictory sources:** When retrieved documents genuinely conflict, a claim supported by one source but contradicted by another should not be scored as hallucination. Flag as a source conflict requiring the agent to acknowledge uncertainty.

## Limitations

- This framework requires access to the agent's full execution trace (plans, search queries, retrieved documents, intermediate summaries). Black-box agents that only expose final output cannot be audited this way.
- The noise domination metric (H_IS) depends on reliable semantic clustering and relevance ranking, which themselves can introduce errors on highly technical or ambiguous topics.
- Propagation analysis assumes a linear plan-search-summarize loop. Agents with parallel branches, backtracking, or non-sequential architectures need adapted decomposition.
- The framework evaluates hallucination *detection*, not automatic correction. Fixes still require architectural changes to the agent pipeline.
- Cognitive bias diagnosis (anchoring, homogeneity) provides correlational evidence, not definitive causal proof. Use it as a signal for investigation, not a guaranteed root cause.

## Reference

Zhan, Y., Fan, T., Huang, L., Guo, Z., & Huang, C. (2026). *Why Your Deep Research Agent Fails? On Hallucination Evaluation in Full Research Trajectory.* arXiv:2601.22984v1. [https://arxiv.org/abs/2601.22984v1](https://arxiv.org/abs/2601.22984v1)

Look for: The PIES taxonomy definition (Section 3), trajectory decomposition and verification pipeline (Section 4), the four hallucination metrics (H_ES, H_IS, H_EP, H_IP), propagation chain analysis (Section 6), and cognitive bias findings on anchoring and homogeneity effects (Section 6.2). Code and data: [https://github.com/yuhao-zhan/DeepHalluBench](https://github.com/yuhao-zhan/DeepHalluBench)