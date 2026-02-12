---
name: "agentcpm-report-interleaving-drafting-deepening"
description: >
  Generate deep research reports by interleaving evidence-based drafting with reasoning-driven
  deepening. Uses the WARP (Writing As Reasoning Policy) framework from AgentCPM-Report to
  dynamically evolve outlines during writing instead of rigidly following a static plan.
  Trigger phrases: "deep research report", "write a comprehensive analysis",
  "investigate and write up", "research report on", "deep dive report",
  "analyze this topic thoroughly and produce a report"
---

# Interleaving Drafting and Deepening for Deep Research Reports

This skill enables Claude to generate deep, insight-rich research reports by alternating between **evidence-based drafting** (writing grounded in retrieved information) and **reasoning-driven deepening** (identifying gaps, expanding underdeveloped sections, and evolving the outline). Rather than committing to a fixed outline upfront and filling it in linearly, this approach mirrors how expert human writers work: they start writing, discover what they don't know, research further, restructure, and deepen iteratively. The result is reports with substantially richer insight and more coherent argumentation than plan-then-write methods produce.

## When to Use

- When the user asks to produce a comprehensive research report on an open-ended topic (e.g., "Write a deep analysis of WebAssembly adoption in production systems")
- When generating technical documentation that requires synthesizing information from multiple sources, codebases, or domains
- When the user needs an investigation report — e.g., "Research why our API latency spiked and write up findings"
- When producing architecture decision records (ADRs) or design documents that require exploring trade-offs in depth
- When writing competitive analyses, technology evaluations, or literature reviews where surface-level coverage is insufficient
- When the user says "deep dive", "thorough analysis", "comprehensive report", or "investigate and document"

## Key Technique: Writing As Reasoning Policy (WARP)

Traditional report generation follows a **plan-then-write** paradigm: first produce a complete outline, then fill each section. This fails for deep research because constructing a good outline *itself* requires understanding the material — creating a chicken-and-egg problem. WARP breaks this by treating the outline as a living document that co-evolves with the content.

The core insight is that **writing reveals what you don't know**. When you draft a section on, say, "Performance Characteristics of Column-Store Databases," you discover specific sub-questions (write amplification under concurrent writes, compression ratio vs. query speed trade-offs) that weren't apparent during initial planning. WARP captures this by alternating between two phases:

1. **Evidence-Based Drafting**: For each section in the current outline, formulate context-aware queries conditioned on what has already been written, retrieve relevant information, and synthesize it into grounded prose. The key is that queries are informed by the *accumulating narrative*, not just the section title.

2. **Reasoning-Driven Deepening**: After drafting, evaluate the content for logical gaps, superficial arguments, and underdeveloped areas. Decompose weak sections into granular sub-sections, expanding the outline hierarchy. Then trigger a new drafting cycle targeting only the newly created sections. The agent autonomously decides when to stop deepening by evaluating whether additional expansion would yield diminishing returns.

This interleaving produces reports that score significantly higher on *insight* — the ability to surface non-obvious connections and deep analysis — compared to static-outline approaches.

## Step-by-Step Workflow

1. **Capture the research question and constraints.** Parse the user's request to identify: the core topic, desired depth, target audience, any specific angles or sub-questions, output format preferences, and available sources (codebase, URLs, documents, or web search).

2. **Generate a sparse Level-1 outline.** Create an intentionally minimal outline with only 3-5 top-level section titles and one-sentence writing intents for each. Do NOT try to produce a comprehensive outline — leave room for discovery. Example:
   ```
   ## 1. Background and Motivation
   Intent: Establish why this topic matters and what problem it addresses.
   ## 2. Core Technical Approach
   Intent: Explain the primary mechanism or architecture.
   ## 3. Empirical Evidence and Trade-offs
   Intent: Present data, benchmarks, and practical considerations.
   ## 4. Synthesis and Recommendations
   Intent: Draw cross-cutting insights and actionable conclusions.
   ```

3. **Begin the first Drafting pass.** For each Level-1 section, formulate 2-3 specific search queries conditioned on the section intent AND any context already gathered. Retrieve information (via web search, codebase search, file reads, or provided documents). Write a substantive first draft of each section grounded in retrieved evidence, citing sources inline.

4. **Execute Reasoning-Driven Deepening.** Read through the full draft and for each section ask:
   - Are there logical gaps where claims lack supporting evidence?
   - Are there surface-level statements that could be decomposed into deeper sub-questions?
   - Did drafting reveal new angles not in the original outline?
   - Are there contradictions or tensions between sections that need resolution?

   For each identified gap, create a new sub-section in the outline (Level-2 or Level-3).

5. **Expand the outline with new sub-sections.** Update the working outline to reflect the deeper structure. Mark which new sub-sections need drafting. Typical expansion: a Level-1 section with a gap becomes 2-4 Level-2 sub-sections, each with a specific writing intent.

6. **Draft the newly added sub-sections.** Formulate new, targeted queries conditioned on the *existing draft content* (not just section titles). Retrieve additional evidence and write each new sub-section. This is where the deepest insights emerge — queries are now highly specific because they're informed by what was already written.

7. **Repeat deepening (2-4 cycles maximum).** After each drafting pass, re-evaluate for gaps. Continue the interleaving loop but apply diminishing-returns logic: stop expanding when new sub-sections would add marginal value, when the outline has reached Level-3 depth across most sections, or when the content adequately addresses the original research question.

8. **Synthesize cross-cutting insights.** Review the full expanded draft for themes, patterns, and connections that span multiple sections. Write or rewrite the synthesis/conclusion section to surface these non-obvious insights. This is where the iterative process pays off — connections invisible during initial planning become clear after deep drafting.

9. **Polish and verify coherence.** Ensure the final report reads as a unified document, not a collection of independently drafted sections. Check that: the introduction previews what the deep investigation actually found (not what was initially planned), transitions between sections are logical, evidence is consistently cited, and the overall argument flows.

10. **Deliver with a structural summary.** Present the final report along with a brief note on how the outline evolved — which sections were added during deepening, what unexpected findings emerged. This transparency helps the user understand the report's depth.

## Concrete Examples

**Example 1: Technical Architecture Investigation**

```
User: Write a deep research report on event sourcing vs. traditional CRUD
for our order management system. We process ~50K orders/day.

Approach:
1. Sparse Level-1 outline:
   - Background (why this matters for order systems)
   - Event Sourcing Mechanics
   - CRUD Approach Baseline
   - Comparative Analysis
   - Recommendation

2. First Drafting pass: Research event sourcing fundamentals,
   CRUD patterns in order systems, retrieve benchmark data.
   Write initial sections.

3. Deepening round 1 — gaps discovered:
   - "Event Sourcing Mechanics" is too shallow on replay/projection cost
   - Missing: how event schema evolution works at 50K orders/day
   - Missing: operational complexity (debugging, monitoring)
   → Add sub-sections: "Projection Rebuild Costs at Scale",
     "Schema Evolution Strategies", "Operational Observability"

4. Second Drafting pass: Targeted queries on projection rebuild
   benchmarks, schema versioning patterns (upcasting, lazy migration),
   and event store monitoring tools. Write new sub-sections.

5. Deepening round 2 — new insight emerges:
   - The operational complexity section reveals that hybrid approaches
     (CRUD + event log for audit) may dominate pure event sourcing
     for this scale
   → Add sub-section: "Hybrid Architecture: CRUD with Event Audit Trail"

6. Final synthesis surfaces the non-obvious conclusion:
   At 50K orders/day, pure event sourcing's replay costs likely
   outweigh its benefits unless the team needs full temporal queries.
   The hybrid approach captures 80% of the audit/replay value at
   20% of the operational cost.

Output: 2500-word report with 4 top-level sections expanded to
12 sub-sections, concrete latency estimates, and a decision matrix.
```

**Example 2: Codebase Investigation Report**

```
User: Investigate why our test suite takes 45 minutes and write
up a report with recommendations.

Approach:
1. Sparse Level-1 outline:
   - Current State (test counts, timing breakdown)
   - Bottleneck Analysis
   - Optimization Opportunities
   - Recommended Action Plan

2. First Drafting pass: Search codebase for test configuration,
   timing data, CI logs. Identify top-level numbers (test count,
   parallelism settings, fixture setup patterns).

3. Deepening round 1 — gaps discovered:
   - "Bottleneck Analysis" reveals database fixture setup is 60%
     of total time, but WHY is unclear
   - Missing: analysis of test isolation strategy (per-test DB
     reset vs. transaction rollback)
   → Add: "Database Fixture Teardown Profiling",
     "Test Isolation Strategy Analysis"

4. Second Drafting pass: Grep for setUp/tearDown patterns, analyze
   fixture factories, check for missing test database pooling.
   Write sub-sections with specific file:line references.

5. Deepening round 2:
   - Discover that 12 integration tests each spin up a Redis
     instance — invisible from top-level timing
   → Add: "Hidden Infrastructure Setup Costs"

6. Synthesis: The 45-minute runtime is 60% DB fixtures (fixable
   with transaction rollback), 20% redundant Redis instances
   (fixable with shared test container), 20% actual test execution.

Output: Report with specific file references, a prioritized fix
list with estimated impact, and a phased implementation plan.
```

**Example 3: Technology Evaluation**

```
User: Deep dive on whether we should migrate from REST to gRPC
for our internal microservices communication.

Approach:
1. Sparse outline: Background, gRPC Mechanics, Migration Costs,
   Performance Comparison, Recommendation.

2. First Drafting pass: Research gRPC streaming, protobuf schema
   management, HTTP/2 multiplexing benefits. Write initial sections.

3. Deepening round 1:
   - "Migration Costs" is vague — need concrete sub-sections
   → Add: "Client Library Generation Pipeline", "Backward
     Compatibility During Rollout", "Observability Tooling Gap"
   - Performance section lacks our-context specificity
   → Add: "Latency Profile for <1KB Payloads" (matches our p95)

4. Second Drafting pass: Research protobuf backward compatibility
   rules, gRPC-gateway for gradual migration, compare Jaeger/Zipkin
   gRPC support vs REST.

5. Deepening round 2:
   - Discover that gRPC reflection + server streaming enables a
     real-time dashboard pattern impossible with REST polling
   → Add: "Emergent Architectural Possibilities"

6. Synthesis: gRPC wins on latency and type safety, but the
   migration's hidden cost is observability tooling. Recommend
   gRPC for new services + gRPC-gateway adapter for existing ones.

Output: Structured report with latency benchmarks, migration
checklist, and a "what we'd gain beyond performance" section
that only emerged through iterative deepening.
```

## Best Practices

**Do:**
- Start with an intentionally sparse outline (3-5 top-level sections). Resist the urge to plan comprehensively upfront — the deepening process will discover what's actually needed.
- Condition each search query on what has already been written. A query for "event sourcing projection costs" is weaker than "event sourcing projection rebuild latency at 50K events/day with PostgreSQL" informed by earlier draft context.
- Track which sections were added during deepening and surface this in the final report. The evolution of the outline IS part of the insight.
- Cap deepening at 3-4 cycles. Research shows performance plateaus around 8-9 expansion steps — more is rarely better.

**Avoid:**
- Do not generate a detailed 20-section outline before writing anything. This is the plan-then-write anti-pattern that WARP specifically addresses.
- Do not draft all sections to completion before deepening. Interleave after each major section or pair of sections so that discoveries in one section can inform queries for the next.
- Do not expand every section uniformly. Deepening should be targeted — only sections where gaps were detected get expanded. Some sections are fine at Level-1 depth.
- Do not skip the synthesis step. The cross-cutting insight that emerges from reading the fully deepened draft is often the most valuable part of the report and cannot be produced by any individual section alone.

## Error Handling

- **Insufficient source material**: If search/retrieval yields thin results for a sub-section created during deepening, flag it explicitly in the report rather than padding with speculation. Write: "Limited evidence was found for [X]; the following is based on [Y] sources and should be validated."
- **Outline explosion**: If deepening produces more than 15-20 sub-sections, consolidate. Merge closely related sub-sections and move peripheral findings to an appendix. The goal is depth on important areas, not breadth on everything.
- **Circular deepening**: If a deepening cycle identifies the same gaps as the previous cycle, the issue is likely source quality, not outline structure. Stop expanding and note the knowledge gap in the report.
- **Contradictory evidence**: When drafting surfaces conflicting information across sections, do not silently resolve it. Create a dedicated sub-section analyzing the contradiction — this often produces the report's strongest insights.

## Limitations

- This approach is designed for **open-ended research topics** where the full scope isn't knowable upfront. For well-scoped, structured tasks (e.g., "document this API's endpoints"), a simple outline-then-write approach is more efficient.
- The iterative deepening process produces longer reports. If the user needs a brief summary (under 500 words), use a standard approach and reserve this technique for when depth is explicitly valued.
- Quality depends heavily on the retrieval step. If source material is poor (e.g., no web access, sparse codebase docs), deepening cycles will yield diminishing returns quickly. Acknowledge source limitations early.
- The technique works best when Claude can perform actual information retrieval (web search, codebase search, file reads). Without tool access to gather evidence, the "evidence-based" drafting phase degrades to reasoning from training knowledge alone.

## Reference

[AgentCPM-Report: Interleaving Drafting and Deepening for Open-Ended Deep Research](https://arxiv.org/abs/2602.06540v1) — Focus on Section 3 (WARP framework), particularly the five core actions (Initialize, Search, Write, Expand, Terminate) and how Evidence-Based Drafting and Reasoning-Driven Deepening interleave to produce reports that outperform closed-source systems on the Insight metric.