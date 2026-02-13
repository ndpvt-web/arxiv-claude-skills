---
name: "apex-agents"
description: |
  Design and execute long-horizon, cross-application agent workflows for professional knowledge work
  (finance, consulting, legal). Applies the APEX-Agents benchmark methodology to structure multi-step
  tasks that span files, spreadsheets, documents, email, calendars, and code execution within realistic
  work environments.

  Trigger phrases:
  - "Build an agent workflow for this banking/consulting/legal task"
  - "Create a cross-application task pipeline"
  - "Design a multi-step professional workflow with rubric evaluation"
  - "Set up an Archipelago-style sandboxed agent environment"
  - "Evaluate agent performance on a long-horizon task"
  - "Break this professional task into rubric-graded criteria"
---

# APEX-Agents: Long-Horizon Cross-Application Agent Workflows

This skill enables Claude to design, decompose, and execute complex professional knowledge-work tasks
that span multiple applications and file types — the kind of work investment banking analysts,
management consultants, and corporate lawyers do daily. Drawing from the APEX-Agents benchmark
(Vidgen et al., 2026), it teaches Claude to structure tasks as multi-criteria workflows operating
across spreadsheets, documents, PDFs, email, calendars, code execution, and file systems, then
evaluate outputs against binary rubric criteria. The core insight: professional agent tasks are not
single-tool problems — they require navigating a "world" of ~166 interrelated files using 8-10
different application types, with success measured by whether specific factual criteria are met.

## When to Use

- When the user needs to automate a multi-step professional workflow that touches spreadsheets, documents, email, and other tools simultaneously
- When building an agent system that must navigate a realistic file environment (100+ files) to complete a knowledge-work task
- When designing evaluation rubrics for agent task completion with binary pass/fail criteria
- When decomposing a vague professional request ("prepare the deal memo") into concrete, verifiable sub-criteria
- When setting up sandboxed execution environments for agent benchmarking across multiple application types
- When the user wants to evaluate whether an AI agent can handle realistic analyst/consultant/lawyer work
- When orchestrating a pipeline where an agent must read data from one source, process it, and produce output in a different format or application

## Key Technique

**Cross-application task decomposition with rubric-based evaluation.** APEX-Agents demonstrates that
professional knowledge work is fundamentally multi-application: a single task might require reading
a PDF contract, cross-referencing data in a spreadsheet, drafting an email summary, and updating a
calendar — all within a shared "world" of ~166 files. The benchmark's 480 tasks average 1.82
estimated human-hours each and require satisfying an average of 4.06 binary criteria per task. The
critical design choice is that each criterion is independently gradable as Met/Not Met, making
evaluation unambiguous. Tasks span three professional domains: investment banking (deal analysis,
financial modeling), management consulting (market research, slide decks), and corporate law
(contract review, due diligence).

**The "world" abstraction.** Rather than giving agents isolated inputs, APEX-Agents places them in a
realistic work environment — a directory tree of 150-180 files including spreadsheets, documents,
PDFs, presentations, emails, calendar entries, and chat logs. The agent must discover which files are
relevant, extract the right information, and produce outputs using the correct application. This
mirrors real professional work where knowing *where* to find information is half the challenge. The
benchmark includes 33 such worlds (10 banking, 11 consulting, 12 law), each shared across multiple
tasks.

**Archipelago execution model.** The open-source Archipelago infrastructure provides sandboxed
environments where agents can execute code, read/write files, send simulated emails, and interact
with tool APIs — all without side effects. Web search is intentionally disabled to ensure
reproducibility. This sandboxed approach is directly applicable when building agent evaluation
pipelines: isolate the agent, give it a file system and tool access, and grade its outputs against
predetermined criteria.

## Step-by-Step Workflow

1. **Define the world**: Inventory all files, data sources, and applications the agent will need. Organize them into a directory structure mirroring a realistic work environment (documents/, spreadsheets/, emails/, presentations/, etc.). Aim for completeness — include both relevant and irrelevant files so the agent must exercise judgment about what matters.

2. **Write the task prompt as a single-turn instruction**: Phrase the task as a professional would delegate it: "Review the Q3 earnings data in the financials folder, cross-reference against the analyst projections in the strategy deck, and draft a client email summarizing the three largest variances." The prompt should be self-contained but require multi-file navigation.

3. **Decompose into binary rubric criteria**: Break the expected output into 2-10 independently verifiable criteria. Each criterion must be answerable as Met or Not Met with no ambiguity. Examples: "Email correctly states the largest variance is in COGS at $2.3M" or "Output spreadsheet contains a pivot table grouped by region."

4. **Create gold reference outputs**: Produce the correct output(s) manually. These serve as the ground truth for evaluation. Include the exact format expected (e.g., .xlsx with specific sheet names, .docx with required sections, plain text email body).

5. **Configure the execution sandbox**: Set up an isolated environment with file system access, spreadsheet read/write capability, document creation tools, code execution (Python/JS), and simulated email/calendar APIs. Disable external network access for reproducibility.

6. **Execute the agent against the task**: Provide the agent with the task prompt and access to the world's file system. Let it navigate, read files, execute code, and produce outputs without human intervention. Log all tool calls and file accesses for debugging.

7. **Grade each rubric criterion independently**: For each criterion, compare the agent's output against the gold reference. Use a judge (human or LLM) that receives the prompt, the agent's output, and relevant artifacts. Score each criterion as 1 (Met) or 0 (Not Met).

8. **Compute Pass@1**: A task passes only if ALL rubric criteria are met on a single attempt. This strict metric reflects professional standards — a deal memo with the wrong numbers is a failure regardless of how well-formatted it is.

9. **Analyze failure modes**: Categorize failures by type: wrong file accessed, correct file but wrong data extracted, correct data but wrong computation, correct computation but wrong output format. This diagnosis drives targeted improvements.

10. **Iterate on agent capabilities**: Based on failure analysis, improve the agent's file navigation heuristics, tool selection logic, or output formatting. Re-run against the same tasks to measure improvement.

## Concrete Examples

**Example 1: Investment Banking — Variance Analysis Email**

```
User: "I have a folder with Q3 financial reports, analyst projections, and client
correspondence. I need to identify the top 3 revenue variances between actuals and
projections, then draft a client-ready email summarizing the findings."

Approach:
1. Scan the world directory to find relevant files:
   - financials/q3_actuals.xlsx (actual revenue by segment)
   - projections/analyst_forecast_q3.xlsx (projected revenue)
   - correspondence/client_templates/ (email format reference)
2. Load both spreadsheets, align on segment names, compute variance = actual - projected
   for each segment.
3. Rank by absolute variance, select top 3.
4. Draft email using the client template format, citing specific figures.

Rubric Criteria:
- [Met/Not Met] Email identifies correct top 3 segments by variance magnitude
- [Met/Not Met] Each variance figure matches computed difference within rounding
- [Met/Not Met] Email follows the client template format from correspondence/
- [Met/Not Met] Email includes both actual and projected figures for context

Output:
Subject: Q3 Revenue Variance Summary — Top 3 Segments

Dear [Client],

Following our review of Q3 actuals against projections:

1. Enterprise Cloud: $4.2M actual vs. $3.1M projected (+$1.1M / +35.5%)
2. Consumer Hardware: $2.8M actual vs. $3.5M projected (-$0.7M / -20.0%)
3. Professional Services: $1.9M actual vs. $1.5M projected (+$0.4M / +26.7%)

[Remainder of email following template structure...]
```

**Example 2: Management Consulting — Market Sizing Deck**

```
User: "Using the industry reports and internal data in the project folder, build a
market sizing slide for the European EV charging market. Include TAM, SAM, SOM with
supporting calculations."

Approach:
1. Locate relevant files in the world:
   - data/eurostat_ev_registrations.xlsx
   - reports/mckinsey_ev_charging_2025.pdf
   - internal/client_geographic_footprint.xlsx
   - templates/slide_template.pptx
2. Extract European EV registration data and growth rates from the spreadsheet.
3. Pull charging infrastructure market size from the McKinsey PDF.
4. Cross-reference client's geographic presence to compute SAM (countries where
   client operates) and SOM (realistic capture rate based on internal data).
5. Build calculations in a supporting spreadsheet, create the summary slide.

Rubric Criteria:
- [Met/Not Met] TAM figure sourced from the McKinsey report with correct citation
- [Met/Not Met] SAM correctly filtered to client's 6 operating countries
- [Met/Not Met] SOM applies the 3-5% capture rate from internal strategy doc
- [Met/Not Met] Supporting calculations provided in a separate .xlsx file
- [Met/Not Met] Slide uses the provided template format

Output:
- market_sizing_slide.pptx (single slide with TAM/SAM/SOM funnel)
- market_sizing_calcs.xlsx (detailed build-up with sourced assumptions)
```

**Example 3: Corporate Law — Contract Red Flag Review**

```
User: "Review the draft SPA in the deals folder against our standard terms checklist.
Flag any clauses that deviate from our templates and summarize the risk exposure."

Approach:
1. Locate files:
   - deals/draft_spa_acme_v3.docx (the contract to review)
   - templates/standard_spa_terms.docx (firm's standard terms)
   - checklists/spa_review_checklist.xlsx (checklist of 42 standard clauses)
2. Parse both documents, align clause-by-clause against the checklist.
3. For each deviation, classify severity (High/Medium/Low) based on
   the checklist's risk weighting column.
4. Produce a summary memo listing deviations with page references.

Rubric Criteria:
- [Met/Not Met] All High-severity deviations identified (indemnity cap, governing law, IP assignment)
- [Met/Not Met] Each flagged deviation references the correct clause number in the draft
- [Met/Not Met] Risk exposure summary includes estimated financial impact where applicable
- [Met/Not Met] Output formatted as a memo following firm template in templates/memo_format.docx
```

## Best Practices

**Do:**
- Structure every task with explicit binary rubric criteria before execution — vague "quality" assessments are not actionable
- Include distractor files in the work environment; real professionals must filter signal from noise among hundreds of files
- Test tasks at the 1-2 human-hour complexity level — this is the sweet spot where agents provide real value but tasks are still verifiable
- Use domain-specific output formats (deal memos, slide decks, redline markups) rather than generic text responses
- Log every file access and tool call during execution to enable failure diagnosis
- Design criteria that test factual correctness (specific numbers, specific clause references) not subjective quality

**Avoid:**
- Don't create tasks that require only a single file or single application — the cross-application navigation is where agents fail and where the benchmark provides signal
- Don't use criteria that require subjective judgment ("well-written", "professional tone") — binary grading requires objective verifiability
- Don't allow web search during evaluation runs — it destroys reproducibility and lets agents compensate for poor file navigation
- Don't score partial credit on criteria — a criterion is Met or Not Met, and a task passes only if all criteria pass
- Don't skip the gold output step — without a reference answer, rubric grading becomes unreliable

## Error Handling

| Failure Mode | Detection | Resolution |
|---|---|---|
| Agent opens wrong file | File access log shows irrelevant files read | Improve file naming conventions or add a manifest/index file to the world |
| Correct file, wrong data extracted | Output contains stale or adjacent-cell data | Ensure spreadsheets have clear headers; add data validation criteria to rubric |
| Computation error | Rubric criterion fails on specific number | Include the expected computation method in the task prompt or provide a formula reference |
| Output format mismatch | Agent produces .txt instead of .xlsx | Specify exact output format in the task prompt; add format-check criterion to rubric |
| Agent halts mid-task | Execution log shows tool error or timeout | Sandbox must handle tool failures gracefully; set per-task time limits with clear timeout behavior |
| Rubric criterion ambiguous | Two human graders disagree on Met/Not Met | Rewrite criterion to reference a specific, verifiable fact or artifact property |

## Limitations

- **Top agent scores are ~24%**: Even the best models solve fewer than 1 in 4 tasks. This benchmark identifies the frontier of agent capability, not a solved problem. Do not expect reliable automation of complex professional workflows today.
- **Single-turn prompting only**: APEX-Agents uses single-turn task prompts. Real professional work involves iterative clarification. Multi-turn agent interaction may yield higher success rates but is outside this benchmark's scope.
- **Domain specificity**: The three professional domains (banking, consulting, law) have particular conventions. Applying this methodology to other domains (healthcare, engineering, research) requires creating new worlds with domain-appropriate file types and evaluation criteria.
- **Evaluation cost**: Binary rubric grading by an LLM judge still requires careful prompt engineering to avoid false positives. Human verification of judge accuracy is recommended for high-stakes evaluations.
- **File-heavy setup**: Creating realistic 166-file worlds is labor-intensive. For quick agent prototyping, start with 10-20 file worlds and scale up once the basic pipeline works.

## Reference

**Paper:** Vidgen, B., Mann, A., Fennelly, A., Stanly, J.W., & Rothman, L. (2026). *APEX-Agents: AI Productivity Index for Agents.* arXiv:2601.14242v2. [https://arxiv.org/abs/2601.14242v2](https://arxiv.org/abs/2601.14242v2)

**Key takeaway:** Look at Section 3 for the task creation methodology (how professionals designed 480 tasks across 33 worlds), Section 4 for the Archipelago execution infrastructure, and the results tables for failure mode analysis showing that cross-application file navigation — not computation — is the primary bottleneck for current agents.

**Dataset:** [mercor/apex-agents on HuggingFace](https://huggingface.co/datasets/mercor/apex-agents) — 480 tasks with prompts, rubrics, gold outputs, and metadata.