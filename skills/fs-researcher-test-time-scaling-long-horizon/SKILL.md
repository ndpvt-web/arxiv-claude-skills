---
name: "fs-researcher-test-time-scaling-long-horizon"
description: >
  File-system-based dual-agent deep research framework that scales beyond context windows.
  Separates evidence gathering (Context Builder) from report writing (Report Writer) using
  a persistent hierarchical knowledge base on disk. Use this skill when the user says:
  "research this topic in depth", "write a comprehensive report on X",
  "do deep research about Y", "investigate and write up Z thoroughly",
  "build a knowledge base and then write a report", "scale up research quality with more rounds".
---

# FS-Researcher: File-System-Based Deep Research with Test-Time Scaling

This skill enables Claude to conduct long-horizon deep research tasks that exceed a single context window by implementing a dual-agent, file-system-based architecture from the FS-Researcher paper. Instead of cramming search results and report drafting into one overloaded context, you separate the work into two distinct phases: a **Context Builder** that browses, distills, and archives information into a hierarchical knowledge base on disk, and a **Report Writer** that composes a final report section-by-section using only that knowledge base as its source of facts. The file system acts as durable external memory, enabling iterative refinement across multiple sessions without context overflow.

## When to Use

- When the user asks for a comprehensive research report on a complex topic (technical, business, scientific)
- When a research task requires synthesizing 10+ sources and the gathered material would exceed context limits
- When the user wants iterative, progressively deeper research rather than a single-shot answer
- When the user explicitly requests a structured knowledge base before writing begins
- When report quality matters more than speed, and the user is willing to allocate more compute rounds
- When the user needs traceable citations from report claims back to original sources
- When a consulting-style deliverable or PhD-level literature review is expected

## Key Technique

FS-Researcher solves the fundamental bottleneck of deep research with LLM agents: long trajectories of web browsing and evidence collection consume so many tokens that little budget remains for thoughtful report composition. Prior approaches stuff everything into one context, forcing a tradeoff between breadth of evidence and quality of writing. FS-Researcher eliminates this tradeoff by externalizing state to the file system.

The **Context Builder** agent acts as a digital librarian. It decomposes the research topic into subtopics, searches the web, reads pages, and distills findings into structured Markdown notes organized in a tree of folders reflecting semantic relationships. Each note includes inline citations as relative file paths pointing to archived raw source pages. The agent maintains control files (todos with `[PENDING]`/`[IN-PROGRESS]`/`[COMPLETE]` status, checklists for acceptance criteria, and logs of session decisions). At the end of each session, it performs a checklist-based review, identifying gaps for the next iteration. Running more Context Builder rounds directly improves final report quality -- this is the test-time scaling mechanism.

The **Report Writer** agent then works exclusively from the knowledge base -- no web access. It first creates an outline, then writes exactly one section per session, performing section-level reviews against quality checklists before marking each complete. After all sections are written, it conducts an overall report-level review and revises as needed. This section-by-section approach prevents shallow "fact-listing" and enables analytical depth through local planning and self-correction.

## Step-by-Step Workflow

1. **Initialize the workspace.** Create a workspace directory with this structure:
   ```
   workspace/
   ├── index.md              # Topic decomposition and KB table of contents
   ├── todos.md              # Task tracker with [PENDING]/[IN-PROGRESS]/[COMPLETE]
   ├── checklist.md          # Acceptance criteria for research quality
   ├── log.md                # Session-by-session decisions and review findings
   ├── knowledge_base/       # Hierarchical distilled notes (Markdown)
   └── sources/              # Archived raw webpage content
   ```

2. **Decompose the research topic into subtopics.** Write an `index.md` that breaks the user's question into 5-15 investigable subtopics arranged hierarchically. Create corresponding folders in `knowledge_base/` with descriptive names (e.g., `knowledge_base/scaling_laws/compute_optimal/`).

3. **Run Context Builder rounds.** For each round, follow the inspect-plan-execute cycle:
   - **Inspect**: Read `index.md`, `todos.md`, and existing notes to understand current coverage gaps.
   - **Plan**: Identify 3-5 subtopics or angles to investigate this round.
   - **Execute**: Search the web, read relevant pages, distill findings into structured notes under `knowledge_base/`, and archive raw pages in `sources/`. Each note must include specific facts (not vague summaries) and cite sources via relative file paths.
   - **Review**: Check notes against `checklist.md` criteria. Log gaps in `log.md` and update `todos.md`.

4. **Scale by running additional Context Builder rounds.** Each round deepens coverage. Run at least 3 rounds for adequate breadth; 5+ rounds for comprehensive research. Each round should target gaps identified in the previous review.

5. **Create the report outline.** The Report Writer reads `index.md` and scans the knowledge base to draft a section-by-section outline. Write this as `outline.md` in the workspace. Each section heading should map to specific knowledge base folders.

6. **Write the report section by section.** For each section:
   - Read only the relevant knowledge base notes (not the entire KB).
   - Draft the section with inline citations referencing source files.
   - Review the section against quality checklist criteria (comprehensiveness, insight depth, factual accuracy, readability).
   - Revise if needed, then mark complete in `todos.md`.

7. **Conduct a full report review.** After all sections are written, read the complete report and check for: logical flow between sections, redundancy, missing cross-references, citation consistency, and overall coherence. Revise as needed.

8. **Produce the final deliverable.** Assemble all sections into a single report file with a proper introduction, table of contents, and bibliography derived from the sources directory.

## Concrete Examples

**Example 1: Technical Deep Research**
```
User: "Research the current state of protein structure prediction methods
and write a comprehensive report."

Approach:
1. Create workspace at ./protein_research/
2. Decompose into subtopics in index.md:
   - AlphaFold2 and AlphaFold3 architecture
   - Competing methods (ESMFold, RoseTTAFold, OpenFold)
   - Benchmarks and accuracy metrics (CASP, CAMEO)
   - Limitations and failure modes
   - Downstream applications (drug design, enzyme engineering)
   - Open challenges (dynamics, complexes, disordered regions)

3. Context Builder Round 1: Search for each subtopic, archive 15-20 source
   pages, write initial notes. Discover gap: limited coverage of industrial
   applications.

4. Context Builder Round 2: Focus on industrial applications, recent 2025-2026
   papers, and comparative benchmarks. Archive 10 more sources.

5. Context Builder Round 3: Checklist review reveals weak coverage of
   limitations. Search specifically for failure cases and critical analyses.

6. Report Writer: Create outline with 8 sections. Write each section drawing
   from the relevant KB subfolder. Section on limitations cites 6 sources
   from knowledge_base/limitations/.

Output: A 4000-word report with 30+ cited sources, organized as:
  workspace/
  ├── report.md           # Final assembled report
  ├── outline.md          # Section outline
  ├── index.md            # Topic map
  ├── knowledge_base/
  │   ├── alphafold/
  │   │   ├── architecture.md
  │   │   └── alphafold3_changes.md
  │   ├── competing_methods/
  │   │   ├── esmfold.md
  │   │   └── rosettafold.md
  │   ├── benchmarks/
  │   │   └── casp_results.md
  │   ├── limitations/
  │   │   ├── disordered_regions.md
  │   │   └── dynamics.md
  │   └── applications/
  │       ├── drug_design.md
  │       └── enzyme_engineering.md
  └── sources/
      ├── alphafold3_nature_2024.md
      ├── esmfold_science_2023.md
      └── ... (30+ archived pages)
```

**Example 2: Business Consulting Research**
```
User: "Investigate the market opportunity for AI-powered legal document
review tools. I need a thorough analysis."

Approach:
1. Create workspace at ./legal_ai_research/
2. Decompose into: market size, key players, technology landscape,
   regulatory environment, buyer personas, competitive dynamics,
   pricing models, adoption barriers.

3. Context Builder Round 1: Broad search across all subtopics.
   Archive market reports, vendor pages, regulatory documents.

4. Context Builder Round 2: Deep dive on competitive landscape.
   Search for each identified vendor, pricing, and customer reviews.

5. Context Builder Round 3: Fill gaps on regulatory requirements
   (GDPR, attorney-client privilege implications, bar association
   guidance on AI tools).

6. Report Writer: Outline follows consulting format:
   - Executive Summary
   - Market Overview & Sizing
   - Technology Landscape
   - Competitive Analysis
   - Regulatory Considerations
   - Go-to-Market Recommendations

   Write each section from KB. The competitive analysis section
   cross-references 4 vendor notes and 2 market reports.

Output: A structured consulting-style report with data-backed claims
and traceable sources.
```

**Example 3: Scaling Up Quality on a Specific Question**
```
User: "I need the highest quality analysis possible on quantum error
correction approaches. Spend extra time on research."

Approach:
1. Initialize workspace. Decompose into 12 subtopics covering
   surface codes, color codes, concatenated codes, LDPC codes,
   hardware implementations, threshold theorems, etc.

2. Run 7 Context Builder rounds (more than default) to maximize depth:
   - Rounds 1-2: Broad coverage of all subtopics
   - Rounds 3-4: Fill checklist gaps, seek primary sources
   - Rounds 5-6: Seek conflicting viewpoints, recent preprints
   - Round 7: Final gap analysis and supplementary searches

3. Knowledge base grows to 50+ notes across 12 subfolders.

4. Report Writer produces a 12-section report with extensive
   cross-referencing between sections.

Key insight: Each additional Context Builder round measurably
improves comprehensiveness and analytical insight, with diminishing
returns after round 5. For maximum quality, allocate 5-7 rounds.
```

## Best Practices

**Do:**
- Keep knowledge base notes specific and factual. Write "GPT-4 scores 86.4% on MMLU" not "GPT-4 performs well on benchmarks." Vague notes produce vague reports.
- Use descriptive folder names in the knowledge base that reflect semantic hierarchy. `knowledge_base/transformer_architectures/attention_mechanisms/` is navigable; `knowledge_base/topic_3/subtopic_2/` is not.
- Include relative file path citations in every note (e.g., `[source](../sources/arxiv_2024_attention.md)`). The Report Writer depends on traceability.
- Run the checklist review at the end of each Context Builder round. Without it, subsequent rounds lack direction and repeat prior coverage.
- Write exactly one report section per pass. This forces focused, analytical writing rather than shallow enumeration.

**Avoid:**
- Do not let the Report Writer browse the web. Its sole source of facts must be the knowledge base. This separation is the core architectural insight that prevents context overflow.
- Do not skip the topic decomposition step. A flat list of search queries produces disorganized notes that the Report Writer cannot navigate efficiently.
- Do not dump entire raw web pages into the knowledge base. Distill them into structured notes with specific data points. Raw pages go in `sources/` for reference only.
- Do not write the entire report in one pass. Section-by-section composition with per-section review produces measurably better quality (ablation shows -5.13 quality score for one-shot writing).

## Error Handling

- **Search returns irrelevant results**: Reformulate queries with more specific terminology. Use the knowledge base's existing notes to identify precise technical terms for better queries.
- **Source contradicts existing notes**: Archive both sources and add a note in the knowledge base explicitly flagging the contradiction with citations to both sides. The Report Writer should address conflicting evidence.
- **Knowledge base becomes too large to scan**: Use `index.md` as the navigation map. The Report Writer should read the index first, then selectively read only the notes relevant to the current section.
- **Report section fails quality review**: Do not proceed to the next section. Identify specific deficiencies (missing evidence, unclear argument, unsupported claim), return to the relevant KB notes, and revise. If the KB itself lacks coverage, flag it for an additional Context Builder round.
- **Context window pressure during a round**: Summarize the current session's findings in `log.md` before ending. The next session picks up by reading the log, not by re-reading all prior context.

## Limitations

- This approach adds overhead that is only justified for substantial research tasks. For quick factual lookups or simple summaries, a single-pass search-and-answer is faster and sufficient.
- The quality of the final report is bounded by the quality of web sources available. If the topic is poorly covered online, more Context Builder rounds yield diminishing returns early.
- Without actual web search tool access, you can only simulate this workflow using the user's provided documents or by asking the user to supply source material.
- The file-system coordination assumes a workspace the agent can read and write to. In constrained environments without persistent file access, the dual-agent separation still helps conceptually but loses the scaling benefit.
- Test-time scaling shows diminishing returns after approximately 5 Context Builder rounds. Allocating 10+ rounds rarely improves quality proportionally.

## Reference

**Paper**: [FS-Researcher: Test-Time Scaling for Long-Horizon Research Tasks with File-System-Based Agents](https://arxiv.org/abs/2602.01566v1) (Zhu et al., 2026). Look for: Section 2.2-2.3 on the dual-agent architecture and workspace design, Table 1 for benchmark results showing +3.02 RACE improvement, and Table 3 for ablation results quantifying the contribution of each component (dual-agent split: -10.35 RACE when removed; persistent workspace: -4.07; section-by-section writing: -5.13).