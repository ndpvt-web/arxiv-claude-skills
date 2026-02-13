---
name: "reprompt-prompt-generation-intelligent"
description: "Generate optimized system and user prompts for coding agents using requirements engineering principles from the REprompt framework. Use when: 'generate a system prompt for my coding agent', 'optimize this prompt using RE principles', 'turn my vague feature request into a structured prompt', 'create an agent prompt from requirements', 'write a structured task list for this project', 'decompose this user story into agent-ready prompts'."
---

# REprompt: Requirements-Engineering-Guided Prompt Generation

This skill applies the REprompt multi-agent prompt optimization framework to generate high-quality system prompts and user prompts for coding agents. REprompt grounds prompt construction in four phases of requirements engineering — elicitation, analysis, specification, and validation — producing prompts that are complete, consistent, and dependency-aware. Instead of ad-hoc prompt writing, it systematically interviews for requirements, drafts an SRS-style specification, converts that specification into structured prompts or task lists, and validates the result across seven quality dimensions.

## When to Use

- When the user has a vague or informal feature request (e.g., "I want a Tic-Tac-Toe game") and needs it converted into a structured, dependency-ordered task list for a coding agent
- When designing or refining a system prompt for a coding agent that needs role definition, tool specs, knowledge domains, and behavioral modes
- When an existing prompt produces incomplete or inconsistent outputs and needs systematic optimization
- When the user wants to decompose a project idea into an ordered set of implementation tasks with explicit dependencies
- When building a multi-agent system and each agent needs a well-specified system prompt covering its role, tools, context, and work modes
- When converting user stories or PRDs into executable prompt sequences for AI-assisted development

## Key Technique

REprompt mirrors the four canonical phases of requirements engineering. **Elicitation** uses a structured interview process that examines system components and interactions, core functionalities and workflows, additional features and scope boundaries, and front-end requirements (layout, typography, visual elements). This interview separates requirements into three tiers: overall system requirements, constant component requirements (shared across all parts), and conditional component requirements (specific to individual modules). The goal is to extract complete, unambiguous information before any prompt is written.

**Analysis and Specification** transform the elicited requirements into a preliminary Software Requirements Specification aligned with IEEE 29148-2018 structure. The specification is then converted into one of two output formats: a system prompt decomposed into five components (Role Definition, Knowledge Domain, Available Tools, Context, Work Modes with behavior codes and examples), or a user prompt rendered as a strictly ordered, dependency-aware JSON task list. The chain-of-thought decomposition ensures each task references its prerequisites.

**Validation** scores the generated prompt across seven dimensions — completeness, correctness, organization, traceability, quality attributes, clarity, and consistency — using a structural scoring rubric. Weaknesses identified during validation feed back into revision, producing prompts that measurably improve document quality (PRD completeness from 3.85 to 4.25) and end-user satisfaction (overall satisfaction from 4.5 to 5.75 on a 7-point scale).

## Step-by-Step Workflow

1. **Elicit requirements via structured interview.** Ask the user four categories of questions: (a) what are the system's main components and how do they interact? (b) what are the core functionalities and their workflows? (c) what additional features exist and what is explicitly out of scope? (d) what are the front-end or interface requirements — layout, typography, visual elements, data displays?

2. **Classify requirements into three tiers.** Separate the answers into overall system requirements (architecture, tech stack, deployment), constant component requirements (shared patterns like error handling, authentication, logging), and conditional component requirements (module-specific logic and UI).

3. **Draft a preliminary SRS.** Organize the classified requirements into an IEEE 29148-2018-aligned specification with sections for purpose, scope, functional requirements, non-functional requirements, interface requirements, and constraints. Use precise language — avoid ambiguous terms like "fast" or "user-friendly" without measurable criteria.

4. **Choose the output format.** If the goal is a system prompt, prepare five components: Role Definition, Knowledge Domain, Available Tools, Context, and Work Modes. If the goal is a user prompt / task list, prepare a dependency-ordered JSON task sequence.

5. **Generate the system prompt** (if applicable). Write each of the five components: define the agent's role and persona, enumerate the knowledge domains it should draw on, list available tools with their parameters and constraints, establish the conversational context and project state, and specify work modes with concrete behavior codes and input/output examples.

6. **Generate the user prompt / task list** (if applicable). Convert the SRS into a strictly ordered list of implementation tasks in JSON format. Each task must include: task ID, description, acceptance criteria, dependencies (referencing other task IDs), and priority derived from software dependency analysis (foundational tasks first).

7. **Validate across seven dimensions.** Score the generated prompt on completeness (are all requirements covered?), correctness (do statements match intent?), organization (is structure logical?), traceability (can each prompt element be traced to a requirement?), quality attributes (are non-functional concerns addressed?), clarity (is language unambiguous?), and consistency (do parts not contradict each other?).

8. **Identify and fix weaknesses.** For any dimension scoring below threshold, revise the specific prompt sections. Common fixes: add missing edge cases for completeness, resolve contradictory instructions for consistency, replace vague language for clarity.

9. **Present the final prompt with rationale.** Deliver the optimized prompt along with a brief traceability note explaining which requirements map to which prompt sections, so the user can verify coverage.

10. **Iterate on feedback.** If the user identifies gaps or changes requirements, cycle back to step 1 (elicitation) for the changed scope only, preserving validated sections.

## Concrete Examples

**Example 1: Vague feature request to dependency-ordered task list**

User: "I want a Tic-Tac-Toe game"

Approach:
1. Elicit: Ask about platform (web/CLI), players (PvP/AI), UI preferences, win display, additional features (score tracking, restart)
2. Classify: Overall = web app with vanilla JS; Constant = responsive layout, clean error states; Conditional = game logic module, UI rendering module, AI opponent module
3. Specify and generate task list

Output:
```json
{
  "tasks": [
    {
      "id": "T1",
      "description": "Initialize project structure with index.html, style.css, and main.js",
      "acceptance_criteria": "Project runs in browser with empty page",
      "dependencies": [],
      "priority": 1
    },
    {
      "id": "T2",
      "description": "Implement game state module: 3x3 board array, current player tracker, move validation, win/draw detection",
      "acceptance_criteria": "Unit tests pass for all win conditions, invalid moves rejected, draw detected when board full",
      "dependencies": ["T1"],
      "priority": 2
    },
    {
      "id": "T3",
      "description": "Build UI rendering: 3x3 grid layout, click handlers for cell selection, visual X/O placement, turn indicator",
      "acceptance_criteria": "Clicking empty cell places current player mark, occupied cells reject clicks, turn label updates",
      "dependencies": ["T1", "T2"],
      "priority": 3
    },
    {
      "id": "T4",
      "description": "Implement minimax AI opponent with difficulty toggle (random vs. optimal)",
      "acceptance_criteria": "AI makes valid moves, optimal mode never loses, random mode plays legal random moves",
      "dependencies": ["T2"],
      "priority": 4
    },
    {
      "id": "T5",
      "description": "Add end-game flow: win/draw announcement overlay, score tracker, restart button",
      "acceptance_criteria": "Winner highlighted, scores persist across rounds, restart resets board but not scores",
      "dependencies": ["T2", "T3"],
      "priority": 5
    },
    {
      "id": "T6",
      "description": "Responsive styling and polish: mobile layout, hover states, animations for mark placement",
      "acceptance_criteria": "Playable on 320px+ screens, hover previews mark, placement animates",
      "dependencies": ["T3", "T5"],
      "priority": 6
    }
  ]
}
```

**Example 2: System prompt generation for a code review agent**

User: "Create a system prompt for a code review agent that checks Python PRs"

Approach:
1. Elicit: What aspects to review (style, bugs, security, performance)? What tools available (linter output, test results, git diff)? What tone — strict or mentoring? What standards — PEP 8, type hints required?
2. Classify: Overall = Python code review agent; Constant = respectful tone, cite line numbers; Conditional = security checks for web code, performance checks for data pipelines
3. Generate five-component system prompt

Output:
```markdown
## Role Definition
You are a senior Python code reviewer. You analyze pull request diffs
and provide actionable, line-referenced feedback prioritized by severity
(critical > major > minor > suggestion).

## Knowledge Domain
- Python 3.10+ idioms, PEP 8, PEP 484 type annotation conventions
- OWASP Top 10 for web-facing code (SQL injection, XSS, SSRF)
- Common performance anti-patterns: N+1 queries, unnecessary copies,
  blocking I/O in async contexts
- Testing best practices: pytest fixtures, mocking boundaries, coverage gaps

## Available Tools
- `read_diff`: Retrieve the PR diff. Parameters: pr_number (int)
- `read_file`: Read full file content. Parameters: path (str), ref (str)
- `run_linter`: Execute ruff/mypy on changed files. Returns structured findings
- `post_comment`: Post inline or general PR comment. Parameters: path, line, body

## Context
You are reviewing PRs in a production Python web service using FastAPI,
SQLAlchemy, and pytest. The team enforces type hints on all public
functions and requires test coverage for new logic paths.

## Work Modes
- **quick-scan**: Skim diff for critical issues only (security, crashes).
  Output: max 5 comments, critical/major severity only.
- **full-review**: Line-by-line analysis covering correctness, style,
  security, performance, and test coverage.
  Output: categorized comment list with severity labels.
- **mentoring**: Same as full-review but explain the *why* behind each
  suggestion with links to documentation or examples.
```

**Example 3: Optimizing an existing vague prompt**

User: "My agent prompt just says 'You are a helpful coding assistant. Help the user build their app.' Can you improve it?"

Approach:
1. Elicit what the agent actually does — what kinds of apps, what languages, what tools does it have access to
2. Identify the seven validation gaps: Role is vague (clarity), no tools listed (completeness), no behavioral modes (organization), no constraints (consistency)
3. Rewrite with all five components filled, validate, deliver

Output: A fully specified system prompt with concrete role boundaries, enumerated tools, knowledge scope, context assumptions, and at least two work modes — following the same five-component structure shown in Example 2.

## Best Practices

- **Do:** Always elicit before specifying. Even when the user thinks their request is clear, the four-category interview (components, workflows, additional features, interface) reliably surfaces missing requirements.
- **Do:** Make dependencies explicit in task lists. Every task must reference its prerequisite task IDs so a coding agent can determine execution order without guessing.
- **Do:** Include acceptance criteria for each task. A task without testable criteria cannot be validated — this is the single largest source of agent drift.
- **Do:** Validate using all seven dimensions, not just "does it look right." Scoring systematically catches contradictions and gaps that reading casually misses.
- **Avoid:** Skipping the analysis phase and jumping from raw user input to prompt generation. The SRS draft is what catches scope ambiguity — without it, prompts inherit the vagueness of the original request.
- **Avoid:** Generating monolithic prompts. System prompts must be decomposed into the five components (Role, Knowledge, Tools, Context, Modes) to remain maintainable and debuggable.

## Error Handling

- **User cannot answer elicitation questions:** Provide sensible defaults based on the project type and explicitly mark them as assumptions in the SRS. Flag these assumptions in the final output so the user can override them.
- **Requirements contradict each other:** Surface the contradiction during the analysis phase with a specific example (e.g., "You asked for real-time updates but also specified no WebSocket connections"). Ask the user to resolve before proceeding to specification.
- **Generated task list has circular dependencies:** Re-analyze the dependency graph. If true circularity exists, the requirements need decomposition — split the circular tasks into smaller units that break the cycle.
- **Validation scores low on completeness:** Return to elicitation for the specific missing area rather than guessing. Fabricated requirements cause more damage than incomplete prompts.
- **Prompt exceeds context window limits:** Prioritize by the ablation findings — requirements analysis content has the highest impact, followed by elicitation content, then specification structure. Trim validation commentary and examples first.

## Limitations

- REprompt is optimized for software development prompts. It will over-engineer prompts for simple Q&A or creative writing tasks where formal requirements specification adds friction without proportional value.
- The framework assumes requirements can be elicited upfront. For exploratory or research-oriented coding (e.g., "experiment with different ML architectures"), the elicitation phase may produce artificially rigid specifications.
- Validation scoring relies on judgment against the seven dimensions but does not guarantee runtime correctness — a prompt can score perfectly and still produce buggy code if the underlying model lacks capability.
- The IEEE 29148-2018 alignment is most useful for medium-to-large projects. For a single-function utility or a quick script, the full framework is overhead — use only the elicitation and task-list generation steps.

## Reference

**Paper:** [REprompt: Prompt Generation for Intelligent Software Development Guided by Requirements Engineering](https://arxiv.org/abs/2601.16507v1) — Shi et al., 2026. Focus on Section 3 (framework architecture with four RE-aligned agents) and Section 4 (evaluation showing PRD completeness and user satisfaction improvements).