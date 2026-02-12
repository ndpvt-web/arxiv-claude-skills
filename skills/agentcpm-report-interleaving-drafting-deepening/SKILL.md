---
name: "agentcpm-report-interleaving-drafting-deepening"
description: "Generate deep research reports by interleaving evidence-based drafting with reasoning-driven deepening (WARP). Use when: 'write a deep research report on X', 'generate an in-depth analysis of Y', 'create a comprehensive technical report', 'research and write about Z with citations', 'produce an insight-driven report on this topic', 'deep dive report with iterative refinement'."
---

# Interleaving Drafting and Deepening for Deep Research Reports

This skill enables Claude to produce deep, insight-driven research reports by applying the Writing As Reasoning Policy (WARP) from AgentCPM-Report. Instead of planning a full outline upfront and then filling it in (plan-then-write), WARP treats writing itself as a reasoning mechanism: the agent starts with a sparse outline, drafts evidence-grounded sections, then pauses to diagnose logical gaps and selectively expand the outline before resuming drafting. This interleaving of drafting and deepening mirrors how expert researchers actually write — discovering what they don't know through the act of writing, then going back to investigate and restructure.

## When to Use

- When the user asks for a comprehensive research report, technical survey, or deep analysis on a topic
- When the user requests a document that requires synthesizing information from multiple sources with citations
- When a writing task demands progressive discovery — where the full scope cannot be known before writing begins
- When the user wants an iterative, self-improving report rather than a single-pass generation
- When producing consulting-style deliverables, literature reviews, competitive analyses, or due-diligence reports
- When the user says something like "research X thoroughly and write a report" or "deep dive into Y"

## Key Technique: Writing As Reasoning Policy (WARP)

**The core insight:** Traditional report generation follows plan-then-write — generate a detailed outline first, then fill each section. This fails because constructing a good outline requires the very knowledge you haven't gathered yet. WARP solves this by making outline construction and content generation co-evolve. The outline starts sparse and grows organically as writing reveals gaps.

**Two alternating macro-phases:** The agent cycles between (1) **Evidence-Based Drafting**, where it processes each section sequentially — formulating search queries conditioned on what's already been written, retrieving evidence, and synthesizing grounded content that extends the logical flow; and (2) **Reasoning-Driven Deepening**, where it steps back to assess the entire draft globally, identifies the single section most in need of expansion, decomposes it into sub-sections, and updates the outline. Critically, retrieval during drafting is conditioned on the accumulated narrative, not just the section title, ensuring coherence across sections.

**Termination as a learned decision:** Rather than expanding forever or stopping at an arbitrary count, the agent evaluates whether the report has reached information saturation — whether the logical chain is complete and depth is sufficient. This is a deliberate decision point, not a loop counter.

## Step-by-Step Workflow

1. **Receive query and initialize a sparse Level-1 outline.** Analyze the user's question and produce only top-level section titles (3-6 sections) with one-sentence writing intents for each. Do NOT attempt a comprehensive hierarchical outline yet — keep it intentionally skeletal.

2. **Enter Evidence-Based Drafting for the first section.** Formulate 1-5 search keywords conditioned on the query and the section's writing intent. Use web search or provided sources to retrieve relevant evidence. Synthesize the section grounding every claim in retrieved information with inline citations.

3. **Draft subsequent sections sequentially, not in parallel.** For each next section, formulate search queries conditioned on both the section intent AND all previously drafted content. This ensures new information extends the logical flow rather than repeating or contradicting earlier sections. Cite sources for all factual claims.

4. **Shift to Reasoning-Driven Deepening after completing a drafting pass.** Read the entire draft as a fresh observation. Diagnose: Which section is most superficial? Where are logical gaps? Where would a domain expert push back? Identify exactly one section that most needs expansion.

5. **Expand the weakest section by decomposing it into sub-sections.** Add 2-4 sub-section headings under the identified section, each with a specific writing intent. Update the outline to reflect this new structure. Constrain expansion to one additional hierarchy level per deepening step.

6. **Return to Evidence-Based Drafting for the newly expanded sub-sections.** Formulate fresh, more targeted search queries for each sub-section. Retrieve new evidence and draft the sub-sections, again conditioning on the full accumulated draft for coherence.

7. **Repeat the Drafting-Deepening cycle.** After each drafting pass, re-enter the deepening phase. Continue cycling until one of these termination conditions is met: (a) all sections have sufficient depth and logical completeness, (b) further expansion would add redundancy rather than insight, or (c) the maximum hierarchy depth (3 levels) or deepening budget (8-12 cycles) is reached.

8. **Make an explicit termination decision.** Before stopping, verify: Does the report answer the original query comprehensively? Is the logical chain from introduction to conclusion complete? Are insights grounded in evidence? If yes, terminate. If not, identify the gap and do one more cycle.

9. **Assemble the final report.** Consolidate all drafted sections under the final evolved outline. Ensure section transitions are coherent. Compile a references section from all cited sources. Add an executive summary synthesizing key insights.

10. **Output the report with the evolved outline visible.** Present the final document with clear hierarchical structure reflecting the outline as it evolved through deepening, not as it was initially planned.

## Concrete Examples

**Example 1: Technical Research Report**

```
User: Write a deep research report on retrieval-augmented generation (RAG)
      architectures and their trade-offs for production systems.

Approach:

1. Initialize sparse outline:
   - Introduction and Motivation
   - Core RAG Architectures
   - Retrieval Strategies
   - Production Deployment Considerations
   - Conclusion

2. Draft "Introduction and Motivation" — search for RAG origin papers,
   retrieve key definitions, write grounded intro with citations.

3. Draft "Core RAG Architectures" — search conditioned on intro context,
   retrieve papers on naive RAG, advanced RAG, modular RAG. Write section
   covering each variant.

4. Draft remaining sections sequentially, each search informed by prior content.

5. DEEPENING PASS 1: Read full draft. Diagnose that "Core RAG Architectures"
   is too surface-level — covers what they are but not how they differ.
   Expand into:
   - 2.1 Naive RAG: Retrieve-then-Read
   - 2.2 Advanced RAG: Pre/Post-Retrieval Optimization
   - 2.3 Modular RAG: Composable Pipelines
   - 2.4 Comparative Analysis

6. Draft expanded sub-sections with targeted searches for each variant.

7. DEEPENING PASS 2: "Production Deployment" section lacks specifics.
   Expand into:
   - 4.1 Latency-Accuracy Trade-offs
   - 4.2 Index Management at Scale
   - 4.3 Evaluation and Monitoring

8. Draft new sub-sections. Terminate — all sections now have sufficient depth.

Output: A 3-level hierarchical report with ~15 sections, inline citations,
comparative analysis tables, and an executive summary.
```

**Example 2: Competitive Analysis Report**

```
User: Research the current state of real-time collaboration tools and
      write a report comparing approaches.

Approach:

1. Initialize sparse outline:
   - Market Overview
   - Technical Approaches to Real-Time Sync
   - Major Players and Their Architectures
   - Trade-offs and Selection Criteria
   - Future Directions

2. Draft each section sequentially. "Technical Approaches" search is
   conditioned on market context from section 1.

3. DEEPENING PASS 1: "Technical Approaches" names OT and CRDTs but
   doesn't explain the algorithmic differences or failure modes.
   Expand into:
   - 2.1 Operational Transformation (OT)
   - 2.2 Conflict-Free Replicated Data Types (CRDTs)
   - 2.3 Hybrid Approaches
   Draft with targeted technical searches.

4. DEEPENING PASS 2: "Major Players" is a list without insight.
   Expand into architecture breakdowns per vendor with specific
   technical choices each made and why.

5. DEEPENING PASS 3: Evaluate termination — report now has depth on
   both technical and competitive dimensions. Terminate.

Output: Structured report with technical depth, vendor-specific analysis,
comparison matrices, and cited sources throughout.
```

**Example 3: Investigative Code Architecture Report**

```
User: Analyze our microservices codebase and write a report on
      architectural debt and recommended refactoring priorities.

Approach:

1. Initialize sparse outline from codebase exploration:
   - System Overview and Service Map
   - Dependency Analysis
   - Identified Architectural Debt
   - Refactoring Recommendations
   - Risk Assessment

2. Draft "System Overview" by reading config files, docker-compose,
   service manifests. Draft "Dependency Analysis" by tracing imports
   and API calls across services.

3. DEEPENING PASS 1: "Architectural Debt" section is vague.
   Expand into specific categories discovered during drafting:
   - 3.1 Circular Dependencies Between Services
   - 3.2 Shared Database Anti-Patterns
   - 3.3 Inconsistent Error Handling Contracts
   Draft each with code-level evidence and file references.

4. DEEPENING PASS 2: "Refactoring Recommendations" needs to be
   prioritized by impact. Expand with effort/impact analysis per item.

5. Terminate when each recommendation is grounded in specific code
   evidence found during the drafting process.

Output: Architecture report with file:line references, dependency diagrams
described in text, and prioritized action items with rationale.
```

## Best Practices

**Do:**
- Start with the sparsest possible outline (3-6 top-level sections). Trust the deepening process to add structure where it's actually needed.
- Condition every search query on the accumulated draft, not just the section title. This prevents redundant retrieval and maintains narrative coherence.
- During deepening, identify exactly ONE section to expand per cycle. Trying to expand everything at once defeats the purpose of focused deepening.
- Cite sources inline during drafting. Evidence-grounding is not a post-processing step — it's integral to drafting quality.
- Make termination an explicit reasoned decision, not a default. Ask: "Would a domain expert find this section satisfyingly deep?"

**Avoid:**
- Do NOT generate a detailed multi-level outline before writing anything. This is the plan-then-write anti-pattern that WARP specifically replaces.
- Do NOT draft sections in parallel or out of order. Each section's search and synthesis must be informed by all prior sections.
- Do NOT expand sections that are already sufficient just because deepening cycles remain. Stop when information saturation is reached.
- Do NOT treat deepening as copy-editing. Deepening adds structural depth (new sub-sections with new evidence), not stylistic polish.

## Error Handling

- **Insufficient sources found during search:** If a search returns poor results for a section, reformulate the query using terms and context from already-drafted sections. If still insufficient, note the gap explicitly in the draft and flag it during deepening for a different angle of investigation.
- **Outline grows too large:** Cap hierarchy at 3 levels and deepening at 8-12 cycles. If the report is still incomplete at these limits, prioritize the highest-impact sections and note remaining gaps in a "Further Research" section.
- **Circular deepening (same section keeps being flagged):** If a section is repeatedly identified as weakest, it may indicate the topic genuinely lacks available evidence. Acknowledge this limitation and move to the next-weakest section.
- **Coherence drift between cycles:** After each deepening pass, re-read transitions between the expanded section and its neighbors. Adjust bridging sentences to maintain narrative flow.
- **User provides a pre-made outline:** Treat it as the Level-1 initialization but still apply deepening. Inform the user the outline may evolve during writing — this is a feature, not a deviation.

## Limitations

- This approach is designed for reports requiring depth and synthesis (1,000+ words). For short summaries, quick answers, or single-topic explanations, the overhead of interleaving cycles is unnecessary — use direct generation instead.
- The quality of deepening depends on the ability to diagnose gaps in the draft. For highly specialized domains where Claude has limited training data, gap detection may miss important deficiencies.
- Sequential section drafting means earlier sections influence later ones. If the initial framing is significantly wrong, the error can propagate. Mitigate by treating the first deepening pass as an opportunity to reassess the overall framing.
- Without access to specialized databases or APIs, evidence retrieval is limited to web search and provided context. For domains requiring proprietary data, the user must supply source material.

## Reference

**Paper:** [AgentCPM-Report: Interleaving Drafting and Deepening for Open-Ended Deep Research](https://arxiv.org/abs/2602.06540v1) — Focus on Section 3 (WARP framework), Figure 2 (interleaving architecture), and Section 4 (multi-stage training) for the core methodology showing how writing-as-reasoning outperforms plan-then-write for deep research.