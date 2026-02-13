---
name: "fat-cat-document-driven-metacognitive-multi-agent"
description: >
  Implement the Fat-Cat document-driven metacognitive agent architecture for complex
  multi-step reasoning tasks. Uses Markdown documents as global state instead of JSON,
  a four-stage reasoning pipeline (metacognitive analysis, strategy selection, step
  decomposition, execution), textual strategy evolution for accumulating task-solving
  knowledge, and a closed-loop watcher to prevent hallucinations and infinite loops.
  Trigger phrases: "use fat-cat for this task", "document-driven agent",
  "metacognitive reasoning pipeline", "markdown state management",
  "multi-agent with strategy evolution", "fat-cat agent workflow"
---

# Fat-Cat: Document-Driven Metacognitive Multi-Agent System

This skill enables Claude to orchestrate complex reasoning, retrieval, and coding tasks using the Fat-Cat architecture: a four-stage metacognitive pipeline where agent state is represented as Markdown documents rather than nested JSON. The core insight is that LLMs waste attention budget parsing rigid syntax; by representing state as natural-language Markdown aligned with pre-training corpora, the model dedicates more capacity to semantic reasoning. The system adds a strategy evolution module that learns and stores reusable approaches without parameter updates, and a closed-loop watcher that detects loops, goal deviation, and hallucinations at runtime.

## When to Use

- When a user asks you to solve a multi-hop reasoning problem that requires decomposing intent, planning strategy, and executing steps across multiple tools or data sources
- When building or orchestrating a multi-agent pipeline where agents need shared, human-readable state
- When a task is failing due to context overload from deeply nested JSON state representations
- When the user wants an agent workflow that accumulates learned strategies across sessions (e.g., "remember how we solved this type of problem")
- When a coding or research task requires metacognitive analysis before execution -- understanding *why* the user wants something, not just *what* they asked for
- When orchestrating retrieval-augmented generation where intermediate findings must be traceable and debuggable
- When the user explicitly mentions "fat-cat", "document-driven agent", or "metacognitive pipeline"

## Key Technique

**The LLM-as-CPU Metaphor.** Fat-Cat treats the LLM as a CPU, the context window as RAM, and external tools as I/O peripherals. The framework acts as a kernel managing process scheduling (four pipeline stages), memory management (Markdown document I/O), and exception handling (the Watcher). The critical design choice is that all inter-stage communication flows through Markdown files -- `reasoner.md`, `strategy.md`, `step.md` -- rather than JSON objects. This keeps state in a format the model has seen billions of tokens of during pre-training, improving comprehension and reducing parsing errors. Empirically, replacing Markdown state with equivalent JSON drops performance on HotPotQA by over 12%.

**Textual Strategy Evolution.** Rather than fine-tuning model weights, Fat-Cat maintains a `strategy_library/` of Markdown documents encoding learned problem-solving approaches. When Stage 2 encounters a novel problem type and existing strategies score below a confidence threshold, it triggers a "capability upgrade" subprocess that researches the topic (via search, documentation reading, or exploration), synthesizes a new strategy document, and stores it permanently. This gives the system organizational learning: solve a class of problem once, and every future invocation benefits.

**Closed-Loop Watcher.** An independent monitoring process runs alongside execution. It detects three failure modes: (1) infinite loops (3+ consecutive identical errors), (2) goal deviation (execution results diverging from Stage 1's metacognitive analysis), and (3) hallucination cascades. The Watcher has authority to interrupt execution, force rollback to a previous stage, or escalate to the user. This prevents the common failure where an agent retries the same broken approach endlessly.

## Step-by-Step Workflow

1. **Create the Markdown state directory.** Before any reasoning, create a working directory (or use the project's existing structure) with empty state files: `reasoner.md`, `strategy.md`, `step.md`, and `results.md`. These are the shared memory for all pipeline stages.

2. **Stage 1 -- Metacognitive Analysis.** Before touching any tool or writing any code, deeply analyze the user's request. Write `reasoner.md` containing:
   - **Actual Intent**: What the user truly needs vs. what they literally said
   - **Implicit Constraints**: Language, performance requirements, dependencies, coding style, environment
   - **Information Completeness Check**: List what information is available and what is missing
   - **Complexity Assessment**: Single-step vs. multi-hop; does this need retrieval, computation, or both?
   If information is insufficient, stop and ask the user for clarification rather than guessing.

3. **Stage 2 -- Strategy Selection and Evolution.** Search for prior approaches that match the current problem type. Check if the project has a `strategy_library/` or if you have solved similar problems in this session. Write `strategy.md` containing the selected approach. If no adequate strategy exists, perform a "capability upgrade": research the topic using available tools (web search, documentation reading, codebase exploration), synthesize a reusable strategy document, and store it for future use.

4. **Stage 3 -- Step Decomposition into SOP.** Convert the strategy into a precise, executable Standard Operating Procedure. Write `step.md` with numbered steps at pseudocode-level precision. Each step must specify: the action, the tool or function to use, the expected input/output, and the success criterion. This document is the contract that Stage 4 executes against.

5. **Stage 4 -- Execute with Tool Bridge.** Walk through `step.md` sequentially. For each step, invoke the appropriate tool, capture the output, and append results to `results.md`. If a step fails, record the failure in `results.md` and consult the Watcher logic (Step 6) before retrying.

6. **Watcher -- Monitor for Failure Modes.** After each execution step, check:
   - **Loop Detection**: Has the same error occurred 3+ times in sequence? If yes, stop retrying and re-enter Stage 2 to select a different strategy.
   - **Goal Deviation**: Compare current results against `reasoner.md`. Are we still solving the original problem? If not, roll back to Stage 3 and re-decompose.
   - **Hallucination Check**: Are the outputs grounded in actual tool results, or is the model fabricating data? Flag any ungrounded claims.

7. **Backfill and Synthesize.** Once all steps complete, update `results.md` with the final output. Cross-reference against `reasoner.md` to verify all aspects of the user's intent are addressed. Produce the final answer or artifact.

8. **Strategy Persistence.** If this task introduced a novel approach or the user confirmed a successful outcome, extract the generalizable strategy into a reusable Markdown document. Store it with a descriptive filename (e.g., `strategy_multi_hop_qa_with_date_filtering.md`) for future retrieval.

## Concrete Examples

**Example 1: Multi-Hop Research Question**

```
User: "What was the GDP growth rate of the country where the 2024 Olympics
were held, compared to its neighboring countries, in the year the Olympics
were announced?"

Approach:

1. [Stage 1 - reasoner.md]
   - Actual intent: Compare GDP growth rates across multiple countries for a specific year
   - Decomposed sub-questions: (a) Where were 2024 Olympics held? (b) When announced?
     (c) What are neighboring countries? (d) GDP data for that year
   - Missing info: None -- all answerable via search

2. [Stage 2 - strategy.md]
   - Strategy: Sequential entity resolution → parallel data retrieval → comparative synthesis
   - Resolve anchor entities first (host city/country, announcement year)
   - Then fan out to retrieve GDP data for all relevant countries

3. [Stage 3 - step.md]
   - Step 1: Search "2024 Olympics host country" → France
   - Step 2: Search "2024 Olympics announcement year" → 2017
   - Step 3: Identify France's neighbors: Belgium, Luxembourg, Germany, Switzerland,
     Italy, Spain, Andorra, Monaco
   - Step 4: Retrieve 2017 GDP growth for France and each neighbor
   - Step 5: Compile comparison table
   - Step 6: Synthesize narrative answer

4. [Stage 4 - execution with watcher]
   - Execute each step, recording results in results.md
   - Watcher checks: Step 4 returns GDP for all countries? No missing data?
   - If GDP unavailable for Monaco/Andorra, note limitation rather than hallucinating

Output:
| Country     | 2017 GDP Growth |
|-------------|----------------|
| France      | 2.4%           |
| Germany     | 2.7%           |
| Spain       | 3.0%           |
| Italy       | 1.7%           |
| ...         | ...            |

France's 2017 GDP growth of 2.4% was moderate among its neighbors, with
Spain leading at 3.0% and Italy trailing at 1.7%.
```

**Example 2: Complex Codebase Refactoring**

```
User: "Refactor our authentication module to support OAuth2 in addition
to the existing JWT flow, without breaking any existing tests."

Approach:

1. [Stage 1 - reasoner.md]
   - Actual intent: Add OAuth2 as a second auth method alongside JWT, not replacing it
   - Implicit constraints: All existing tests must pass; maintain backward compatibility
   - Information needed: Current auth module structure, test coverage, OAuth2 provider

2. [Stage 2 - strategy.md]
   - Strategy: Strategy Pattern -- abstract auth behind an interface, implement
     JWT and OAuth2 as concrete strategies
   - Capability upgrade triggered: Research the specific OAuth2 library for this
     framework (e.g., passport-oauth2 for Express)
   - New strategy file created: strategy_dual_auth_provider_pattern.md

3. [Stage 3 - step.md]
   - Step 1: Read current auth module files, map dependencies
   - Step 2: Run existing test suite, record baseline (all green)
   - Step 3: Extract AuthProvider interface from current JWT implementation
   - Step 4: Wrap existing JWT logic in JwtAuthProvider class
   - Step 5: Implement OAuth2AuthProvider class
   - Step 6: Create AuthProviderFactory for runtime selection
   - Step 7: Update route middleware to use factory
   - Step 8: Add OAuth2-specific tests
   - Step 9: Run full test suite -- verify zero regressions

4. [Watcher active throughout]
   - After Step 4: Run tests -- still green? Proceed.
   - After Step 7: Run tests -- regression detected in /api/logout?
     → Roll back Step 7, examine, fix middleware ordering, retry.
   - After Step 9: All tests pass. Goal deviation check: does the refactoring
     address the original intent? Yes -- both auth methods work.

Output: Pull request with 6 changed files, 2 new files, all 47 tests passing,
plus 12 new OAuth2 tests.
```

**Example 3: Data Pipeline with Strategy Learning**

```
User: "Parse these 50 CSV files, each with slightly different column naming
conventions, into a unified schema and load into our PostgreSQL database."

Approach:

1. [Stage 1 - reasoner.md]
   - Intent: ETL pipeline with schema normalization across heterogeneous CSVs
   - Constraints: Must handle column name variations gracefully
   - Complexity: High -- 50 files means manual mapping is infeasible

2. [Stage 2 - strategy.md]
   - No existing strategy for fuzzy column matching
   - Capability upgrade: Research column name similarity matching techniques
   - New strategy created: strategy_fuzzy_schema_normalization.md
     (Use Levenshtein distance + semantic embedding similarity to map variant
     column names to canonical schema)

3. [Stage 3 - step.md]
   - Step 1: Define target canonical schema from database DDL
   - Step 2: Sample 5 CSV files, extract all unique column headers
   - Step 3: Build fuzzy mapping from variant names → canonical names
   - Step 4: Validate mapping with user (present ambiguous matches for review)
   - Step 5: Apply mapping to all 50 files, transform to canonical schema
   - Step 6: Bulk load into PostgreSQL with transaction rollback on failure
   - Step 7: Verify row counts match source files

4. [Watcher]
   - Loop detection: If 3+ files fail with the same mapping error, pause and
     ask user to clarify the ambiguous column
   - Goal deviation: Are we loading data or still stuck on mapping? Escalate
     if Stage 5 takes too many iterations

Output: 50 files loaded, 2 files flagged for manual column review,
unified_schema_mapping.md stored in strategy_library for future ETL tasks.
```

## Best Practices

- **Do:** Write `reasoner.md` before any execution. The metacognitive analysis step catches 80% of wasted effort by identifying missing information, ambiguous intent, and hidden constraints upfront.
- **Do:** Keep state documents in Markdown with clear headers (`## Intent`, `## Constraints`, `## Steps`). This format is what the model processes most efficiently.
- **Do:** Store successful strategies as reusable Markdown files. Name them descriptively (e.g., `strategy_pagination_api_scraping.md`) so future retrieval by similarity search works well.
- **Do:** Let the Watcher interrupt execution early. Three identical failures means the strategy is wrong, not that you should try harder.
- **Avoid:** Representing agent state as nested JSON with deep hierarchies. This forces the model to track bracket matching and key paths instead of reasoning about content.
- **Avoid:** Skipping Stage 1 for "simple" tasks. Even straightforward requests benefit from a 2-sentence intent check -- it costs almost nothing and catches misunderstandings.
- **Avoid:** Storing strategies that are too specific to one task. A good strategy generalizes (e.g., "multi-hop entity resolution" not "finding GDP of Olympic host countries").

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Infinite retry loop | Watcher sees 3+ identical errors in `results.md` | Halt execution, return to Stage 2, select alternative strategy |
| Goal deviation | Current step output contradicts `reasoner.md` intent | Roll back to Stage 3, re-decompose with updated constraints |
| Missing information discovered mid-execution | Stage 4 tool returns empty or ambiguous result | Pause pipeline, update `reasoner.md` with new gap, ask user |
| Strategy library miss | Stage 2 finds no relevant prior strategies | Trigger capability upgrade subprocess: research, learn, store |
| Hallucination in synthesis | Watcher finds claims in `results.md` not grounded in tool outputs | Flag ungrounded content, re-execute relevant steps with stricter prompts |
| State document corruption | Document_Checking finds `step.md` missing required sections | Regenerate from previous stage output, log the corruption |

## Limitations

- **Context window pressure**: The four Markdown state files plus strategy documents consume context. For models with <32k context, truncate or summarize intermediate state documents aggressively.
- **Not suited for single-turn factual questions**: The four-stage pipeline adds overhead. For questions answerable in one step ("What is the capital of France?"), skip this framework entirely.
- **Strategy library cold start**: On the first run, there are no stored strategies. The system works but cannot benefit from learned approaches until it has solved a few problems in the domain.
- **Watcher is heuristic**: The 3-error loop detection and goal deviation checks are simple rules, not learned classifiers. Subtle drift or novel failure modes may escape detection.
- **Withdrawn paper**: The original paper (arXiv:2602.02206v2) was withdrawn for manuscript errors. The technique is validated by the open-source implementation, but specific benchmark numbers should be treated with caution.

## Reference

**Paper**: [Fat-Cat: Document-Driven Metacognitive Multi-Agent System for Complex Reasoning](https://arxiv.org/abs/2602.02206v2) -- Yang et al., 2026. Key insight: representing agent state as Markdown documents instead of JSON improves LLM reasoning by aligning runtime context with pre-training distribution, yielding 5-12% gains across reasoning, retrieval, and coding benchmarks.

**Implementation**: [github.com/answeryt/Fat-Cat](https://github.com/answeryt/Fat-Cat) -- Reference implementation with the four-stage pipeline, strategy library, and Watcher agent.