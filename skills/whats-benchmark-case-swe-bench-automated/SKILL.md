---
name: "whats-benchmark-case-swe-bench-automated"
description: >
  Systematic benchmark leaderboard auditing and APR tool evaluation using
  multi-dimensional classification (submitter origin, LLM backbone, openness,
  product type). Applies the Martinez & Franch (2026) framework from their
  ICSE-SEIP study of SWE-Bench to any software engineering benchmark.
  Trigger phrases:
  - "Analyze this benchmark leaderboard"
  - "Audit SWE-Bench submissions"
  - "Compare automated program repair tools"
  - "Evaluate APR approaches on a leaderboard"
  - "Classify benchmark entries by openness and origin"
  - "Which LLM dominates this SE benchmark?"
---

# Benchmark Leaderboard Auditing & APR Ecosystem Analysis

This skill enables Claude to perform rigorous, multi-dimensional analysis of
software engineering benchmark leaderboards — especially automated program
repair (APR) benchmarks like SWE-Bench. It applies the systematic classification
framework from Martinez & Franch (ICSE-SEIP 2026), who conducted the first
comprehensive study of SWE-Bench Lite (79 entries) and Verified (133 entries)
leaderboards. The framework classifies every submission along four axes:
submitter origin, product/system type, backbone LLM, and degree of openness —
revealing ecosystem-level patterns invisible when looking at raw resolve rates
alone.

## When to Use

- When the user asks to analyze, audit, or compare entries on a software
  engineering benchmark leaderboard (SWE-Bench, HumanEval, MBPP, LiveCodeBench,
  or similar).
- When evaluating which automated program repair tool or coding agent to adopt
  for a project, and the user wants a structured comparison beyond headline
  accuracy numbers.
- When a user needs to assess the openness and reproducibility of benchmark
  submissions (e.g., "Which top SWE-Bench tools can I actually run myself?").
- When writing a research survey or literature review on APR, coding agents, or
  LLM-based software engineering and the user needs a classification taxonomy.
- When the user asks about LLM dominance patterns in SE benchmarks ("Which model
  family wins on SWE-Bench?" or "Is Claude or GPT better for APR?").
- When building or proposing a new benchmark and the user wants to design
  submission metadata requirements that promote transparency.

## Key Technique

The core insight of the Martinez & Franch framework is that a benchmark
leaderboard's **resolve rate tells you what works, but not who built it, how
open it is, or what powers it**. Their study revealed that the SWE-Bench
ecosystem is dominated by industry submissions (especially from small startups
and large publicly traded companies), that proprietary LLMs — particularly the
Claude model family — dominate the backbone choices, and that academic
contributions, while fewer, remain competitive and are typically open source.
These patterns are invisible if you only sort by score.

The framework classifies each leaderboard entry along four orthogonal
dimensions: **(1) Submitter origin** — academic lab, startup, large public
company, or independent; **(2) Product type** — standalone agent, IDE
integration, API-backed service, research prototype; **(3) Backbone LLM** —
specific model family and version, proprietary vs. open-weight; **(4) Openness**
— whether source code, prompts, system architecture, and reproduction
instructions are publicly available. By cross-tabulating these dimensions against
the benchmark's primary metric (e.g., resolve rate), you surface structural
biases, transparency gaps, and concentration risks that inform both tool
selection and benchmark design.

This approach generalizes beyond SWE-Bench. Any benchmark with a public
leaderboard can be audited the same way: collect the entries, classify on
these four axes, and analyze the distributions and correlations.

## Step-by-Step Workflow

1. **Identify the target benchmark and collect raw leaderboard data.** Scrape or
   retrieve the current leaderboard table, capturing every column: entry name,
   submitter, date, primary metric (resolve rate / pass@1 / etc.), and any
   metadata the leaderboard already provides.

2. **Classify submitter origin for each entry.** Research each entry's
   affiliated organization and assign one of: `academic` (university lab or
   research institute), `startup` (private company, <1000 employees),
   `large-corp` (publicly traded or >1000 employees), `independent`
   (individual contributor with no organizational affiliation), or `hybrid`
   (academic-industry collaboration). Use LinkedIn company pages, CrunchBase,
   or the submission's own "About" page to determine size and type.

3. **Identify the backbone LLM for each entry.** From the submission's paper,
   blog post, or documentation, determine which language model(s) power the
   system. Record model family (Claude, GPT, Gemini, Llama, DeepSeek, Qwen,
   etc.), specific version, and whether the model is proprietary or
   open-weight. If an entry uses multiple models, record the primary one and
   note the ensemble.

4. **Classify product type.** Assign each entry one of: `research-prototype`
   (code released for reproducibility, not productized), `agent-framework`
   (general-purpose coding agent like SWE-agent, Agentless, OpenHands),
   `commercial-product` (IDE plugin, SaaS platform, or API service sold to
   customers), or `model-only` (direct LLM evaluation without agent
   scaffolding).

5. **Score openness on a 4-point scale.** For each entry, check four criteria
   and assign 1 point each: (a) source code publicly available, (b) prompts /
   system instructions published, (c) architecture / pipeline documented in a
   paper or detailed blog, (d) results reproducible without proprietary
   infrastructure. Sum to get an openness score from 0 (fully closed) to 4
   (fully open).

6. **Build a structured classification table.** Create a table with columns:
   `Entry Name | Submitter | Origin Type | Product Type | Backbone LLM |
   Proprietary LLM? | Openness Score | Primary Metric | Date`. This is the
   master dataset for all downstream analysis.

7. **Compute distributional statistics.** For each classification dimension,
   compute: (a) count and percentage of entries per category, (b) mean and
   median primary metric per category, (c) top-5 entries per category. Report
   these as summary tables and highlight any category that is over- or
   under-represented relative to its performance.

8. **Cross-tabulate and find correlations.** Look for patterns: Do proprietary
   LLM entries score higher than open-weight ones? Do startups outperform
   large corps? Is there a correlation between openness score and resolve rate?
   Report findings with specific numbers.

9. **Synthesize findings into actionable recommendations.** For tool selection:
   recommend top entries that are both high-performing AND open enough to
   evaluate/adopt. For benchmark design: flag transparency gaps and suggest
   metadata requirements. For research positioning: identify under-explored
   niches (e.g., open-weight LLM agents with high openness).

10. **Format the final report.** Present: (a) an executive summary of 3-5 key
    findings, (b) the full classification table, (c) distributional charts or
    tables, (d) cross-tabulation insights, (e) recommendations.

## Concrete Examples

**Example 1: Auditing SWE-Bench Verified for tool adoption**

```
User: I need to pick an automated program repair tool for our Python
monorepo. Can you analyze the SWE-Bench Verified leaderboard and
recommend the best open-source option?

Approach:
1. Retrieve current SWE-Bench Verified leaderboard entries.
2. Classify each entry on the four axes (origin, product type,
   backbone LLM, openness).
3. Filter to entries with openness score >= 3 (source available,
   prompts published, architecture documented).
4. Among those, rank by resolve rate.
5. Cross-reference backbone LLM — flag if the top open-source
   agents still depend on proprietary LLMs (Claude, GPT) so
   the user knows the full cost picture.

Output:
## SWE-Bench Verified: Top Open-Source APR Tools

| Rank | Tool         | Resolve % | Origin   | Backbone LLM     | Openness |
|------|------------- |-----------|----------|-------------------|----------|
| 1    | OpenHands    | 53.0%     | Academic | Claude 4 Sonnet   | 4/4      |
| 2    | SWE-agent    | 40.2%     | Academic | GPT-4o            | 4/4      |
| 3    | Agentless    | 32.0%     | Academic | Claude 3.5 Sonnet | 4/4      |

Note: All three top open-source tools use proprietary LLMs as their
backbone. For a fully open stack, consider Agentless + DeepSeek-V3
(resolve rate: ~27%) which trades ~5 points of accuracy for zero
API costs.

Recommendation: OpenHands offers the best resolve rate among fully
open-source agent frameworks. Budget for Claude API costs (~$0.50-2
per issue resolution attempt).
```

**Example 2: Benchmarking a new APR tool against the ecosystem**

```
User: We built a new coding agent using Llama 3.3 70B. We got 28%
on SWE-Bench Verified. How does that compare, and where does it
sit in the ecosystem?

Approach:
1. Place the 28% result in context of the full leaderboard
   distribution.
2. Classify the user's tool: origin=startup/independent,
   product=agent-framework, backbone=open-weight, openness=TBD.
3. Compare against entries in the same category (open-weight
   backbone LLM entries).
4. Identify the performance gap to proprietary-LLM entries.

Output:
## Ecosystem Positioning: Your Agent at 28% on SWE-Bench Verified

**Overall ranking:** ~45th-55th percentile among all 133 entries.
The median resolve rate across all entries is approximately 26%.

**Within open-weight LLM entries:** Top 25%. Most open-weight
entries cluster between 15-25%. Your 28% with Llama 3.3 70B
is above the open-weight median of ~20%.

**Gap to proprietary frontier:** The top proprietary-LLM entry
(Claude 4 Sonnet backbone) achieves ~62%. The gap is 34 points,
but this is typical — the Martinez & Franch study found proprietary
LLMs consistently outperform open-weight alternatives on SWE-Bench.

**Key insight:** Your agent outperforms several commercial products
that use proprietary LLMs, which suggests your agent scaffolding
is strong. The largest marginal gain would come from swapping in a
stronger backbone LLM rather than changing the agent architecture.
```

**Example 3: Designing submission requirements for a new benchmark**

```
User: We're creating a new benchmark for automated code review.
What metadata should we require from submitters to avoid the
transparency problems seen in SWE-Bench?

Approach:
1. Apply the Martinez & Franch openness framework.
2. Map their four openness criteria to code review context.
3. Add benchmark-specific metadata fields.
4. Recommend enforcement mechanisms.

Output:
## Required Submission Metadata for CodeReview-Bench

Based on the Martinez & Franch (2026) analysis of SWE-Bench
transparency gaps, require these fields for every submission:

### Mandatory fields
- **Organization type:** Academic / Startup / Large Corp / Independent
- **Backbone model(s):** Exact model name, version, and API endpoint
  (e.g., "claude-4-sonnet-20260514" not just "Claude")
- **Model access type:** Proprietary API / Open-weight / Fine-tuned
  open-weight
- **Source code URL:** Public repo link or explicit "closed source"
- **Prompt/system instruction availability:** Published / Available
  on request / Proprietary
- **Compute cost per run:** Approximate API cost or GPU hours
- **Reproducibility attestation:** Self-reported Yes/No with link
  to reproduction instructions

### Enforcement
- Reject submissions missing any mandatory field.
- Publish an openness score (0-4) alongside the primary metric.
- Require at least one independent reproduction before a submission
  appears on the "Verified" tier of the leaderboard.
```

## Best Practices

- **Do:** Always check the date of leaderboard data. SWE-Bench rankings shift
  frequently; an analysis from 3 months ago may be stale. Note the snapshot
  date in your report.
- **Do:** Distinguish between the agent framework and the backbone LLM when
  attributing performance. A high resolve rate may reflect the LLM's strength
  more than the agent's design. The Martinez & Franch study found LLM choice
  is the single largest predictor of resolve rate.
- **Do:** Report openness score alongside performance. A tool that scores 55%
  but is fully closed is less useful to most users than one scoring 45% with
  full source and prompt availability.
- **Do:** Note when entries use multi-model ensembles or retry strategies, as
  these inflate effective cost per resolution even if the headline metric is
  high.
- **Avoid:** Treating resolve rate as the only comparison axis. The whole point
  of this framework is that origin, openness, and LLM backbone matter for
  adoption decisions, reproducibility, and research contribution.
- **Avoid:** Assuming "open source agent" means "fully open." Many open-source
  APR agents require proprietary LLM API keys to function, making them only
  partially open in practice.

## Error Handling

- **Leaderboard data unavailable or stale:** If the live leaderboard cannot be
  accessed, fall back to the most recent known snapshot and clearly note the
  date. Recommend the user verify against the live version.
- **Missing metadata for entries:** Many leaderboard entries lack documentation
  about their backbone LLM or architecture. Mark these as `unknown` in the
  classification table rather than guessing. Report the percentage of entries
  with incomplete metadata as a finding in itself (this is a transparency gap).
- **Ambiguous submitter origin:** Some entries come from researchers who are
  simultaneously academic and startup-affiliated. Use the `hybrid` category
  and note the dual affiliation.
- **Conflicting performance claims:** If an entry reports different numbers on
  the leaderboard vs. their paper, use the leaderboard number (as it comes
  from standardized evaluation) and flag the discrepancy.

## Limitations

- **SWE-Bench scope:** The Martinez & Franch framework was developed on
  SWE-Bench, which only covers Python repositories. The classification axes
  generalize to other benchmarks, but the specific ecosystem findings (Claude
  dominance, industry concentration) are specific to SWE-Bench as of early
  2026.
- **Snapshot-in-time analysis:** Benchmark leaderboards are dynamic. Any audit
  is a snapshot. Trends (e.g., rising open-weight LLM competitiveness) require
  longitudinal tracking across multiple snapshots.
- **Openness is not quality:** A high openness score means the approach is
  transparent and reproducible, not that it is well-engineered. Conversely,
  a closed submission may still be excellent — you just cannot verify it.
- **Resolve rate limitations:** SWE-Bench itself has known issues — some tasks
  have ambiguous specifications, test suites with gaps, or solutions that pass
  tests without truly fixing the bug. This framework audits the leaderboard
  ecosystem, not the benchmark's task quality.
- **No causal claims:** Correlations between origin type and performance do not
  imply causation. Industry entries may score higher because they have more
  compute budget, not because industry engineers are better at APR.

## Reference

Martinez, M. & Franch, X. (2026). *What's in a Benchmark? The Case of SWE-Bench
in Automated Program Repair.* IEEE/ACM 48th International Conference on Software
Engineering (ICSE-SEIP'26). https://arxiv.org/abs/2602.04449

Look for: The four-axis classification framework (submitter origin, product type,
backbone LLM, openness), the distributional analysis of 79 Lite and 133 Verified
entries, and the finding that proprietary LLMs — especially the Claude family —
dominate SWE-Bench performance while academic open-source entries remain
competitive.