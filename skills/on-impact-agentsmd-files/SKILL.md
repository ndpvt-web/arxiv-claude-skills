---
name: "on-impact-agentsmd-files"
description: "Generate and optimize AGENTS.md / CLAUDE.md repository instruction files to reduce AI coding agent runtime and token consumption. Use when: 'create an AGENTS.md', 'write a CLAUDE.md', 'optimize agent instructions', 'reduce agent token usage', 'set up repository for AI agents', 'add coding agent configuration'."
---

# Generating Effective Repository Instruction Files for AI Coding Agents

This skill enables Claude to create, audit, and optimize repository-level instruction files (AGENTS.md, CLAUDE.md, .github/copilot-instructions.md) that measurably improve AI coding agent efficiency. Research shows that well-structured instruction files reduce agent median runtime by ~29% and output token consumption by ~17% while maintaining equivalent task completion quality. The technique works by front-loading critical repository context so agents navigate codebases directly instead of exploring blindly.

## When to Use

- When the user asks to create an AGENTS.md, CLAUDE.md, or similar agent instruction file for a repository
- When setting up a repository for use with AI coding agents (Codex, Claude Code, Copilot, Gemini)
- When an existing instruction file needs auditing or optimization for better agent performance
- When agent runs are slow or consuming excessive tokens and the user wants to reduce cost
- When onboarding a new project to AI-assisted development workflows
- When the user asks "how do I make AI agents work better on my repo"

## Key Technique

The core insight from the research (Lulla et al., 2026) is that AI coding agents spend significant time and tokens on **exploratory overhead**: reading files to discover project structure, inferring conventions from code patterns, and guessing at build/test commands. An instruction file eliminates this exploration by providing authoritative answers upfront.

The study tested OpenAI Codex across 10 repositories and 124 pull requests in a paired design (with vs. without AGENTS.md). Files containing three categories of information produced the strongest efficiency gains: **(1) coding conventions and best practices**, **(2) architecture and project structure**, and **(3) project description**. These categories are actionable — they directly reduce the number of file reads, directory traversals, and speculative tool calls an agent makes before writing code.

Critically, the improvement manifests in **output tokens** (agent reasoning and tool calls) rather than input tokens. This means the agent isn't just receiving less context — it's *doing less unnecessary work*. The instruction file acts as a routing table: instead of the agent discovering that tests live in `__tests__/` by listing directories, it knows immediately and proceeds to the task.

## Step-by-Step Workflow

1. **Analyze the repository structure.** Run `find` or use Glob to map the top-level directory layout, identify the primary language(s), frameworks, and key directories (source, tests, config, build output).

2. **Extract build and test commands.** Read `package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, or equivalent. Identify the exact commands for: installing dependencies, building, running tests, linting, and formatting. Include any required environment setup.

3. **Identify coding conventions.** Check for linter configs (`.eslintrc`, `ruff.toml`, `.editorconfig`), formatter configs (`prettier`, `black`), and TypeScript/type-checking configs. Summarize the enforced conventions in plain language rather than pointing at config files — agents process natural language instructions more efficiently than parsing config.

4. **Document the architecture.** Describe the major components, their responsibilities, and how they connect. Include the directory-to-responsibility mapping (e.g., `src/routes/` = API endpoints, `src/models/` = database models). Call out non-obvious patterns like dependency injection, plugin systems, or code generation.

5. **Note critical constraints.** Document things an agent must never do (e.g., "never modify migration files directly", "all API changes require OpenAPI spec update") and things it must always do (e.g., "all new functions need JSDoc", "run `make lint` before committing").

6. **Write the instruction file.** Use concise markdown with clear headers. Keep total length under 2 pages (~100 lines). Front-load the most impactful information: build commands, directory map, and conventions. Avoid prose — use bullet points and code blocks.

7. **Validate against common agent tasks.** Mentally simulate an agent receiving a task like "fix the failing test in auth module." Would the instruction file tell the agent where tests live, how to run them, and what conventions to follow? If not, add the missing context.

8. **Place the file correctly.** For Claude Code: `CLAUDE.md` at project root or `.claude/CLAUDE.md`. For Codex/generic: `AGENTS.md` at project root. For Copilot: `.github/copilot-instructions.md`. Multiple files can coexist for different agents.

9. **Iterate based on agent behavior.** After deploying the file, monitor agent runs. If the agent still explores directories that are documented, sharpen the language. If it makes convention violations, make the rules more explicit.

## Concrete Examples

**Example 1: Creating CLAUDE.md for a Node.js API**

User: "Create a CLAUDE.md for this Express API project"

Approach:
1. Read `package.json` for scripts, dependencies, and project metadata
2. Scan `src/` directory structure to map components
3. Check for `.eslintrc.json`, `tsconfig.json`, `.prettierrc`
4. Read a few source files to identify patterns (error handling, middleware usage)
5. Write the instruction file

Output:
```markdown
# Project: user-service

REST API for user management. Node.js 20, Express, TypeScript, PostgreSQL via TypeORM.

## Commands
- Install: `npm ci`
- Dev server: `npm run dev` (port 3000)
- Build: `npm run build`
- Test: `npm test` (Jest, requires `DATABASE_URL` in `.env.test`)
- Lint: `npm run lint` (ESLint + Prettier)
- Type check: `npx tsc --noEmit`

## Directory Structure
- `src/routes/` — Express route handlers, one file per resource
- `src/models/` — TypeORM entity definitions
- `src/middleware/` — Auth, validation, error handling middleware
- `src/services/` — Business logic, one service per model
- `src/migrations/` — TypeORM migrations (auto-generated, do not edit by hand)
- `tests/` — Jest tests, mirrors `src/` structure

## Conventions
- All source files are TypeScript with strict mode
- Use async/await, never raw Promises
- API responses follow `{ data, error, meta }` envelope
- New endpoints require: route handler, service method, Jest test, OpenAPI spec update in `docs/openapi.yaml`
- Error handling uses `AppError` class from `src/utils/errors.ts`
- Import order: node builtins, external packages, internal modules (enforced by ESLint)

## Constraints
- Never modify files in `src/migrations/` directly — use `npm run migration:generate`
- Never commit `.env` files
- All database queries go through services, never in route handlers
```

**Example 2: Auditing an existing AGENTS.md**

User: "My AI agent runs are slow, can you check my AGENTS.md?"

Approach:
1. Read the existing AGENTS.md
2. Check what information is missing against the three key categories
3. Compare against actual repo structure to find gaps
4. Rewrite or supplement

Common problems found during audits:
```
PROBLEM: File says "see the README for architecture" instead of inlining it.
FIX: Agents re-read referenced files, costing tokens. Inline the critical info.

PROBLEM: Build commands are missing or incomplete.
FIX: Add exact commands. "npm test" not "run the tests."

PROBLEM: Conventions described vaguely ("follow best practices").
FIX: Be specific. "Use 2-space indentation, single quotes, trailing commas."

PROBLEM: File is 500+ lines with history and rationale.
FIX: Cut to <100 lines. Agents need instructions, not documentation.
```

**Example 3: Setting up a Python monorepo**

User: "Set up agent instructions for our Django monorepo"

Output:
```markdown
# Project: acme-platform

Django 5.1 monorepo with three apps. Python 3.12, Poetry, PostgreSQL 16.

## Commands
- Install: `poetry install`
- Run server: `poetry run python manage.py runserver`
- Test all: `poetry run pytest`
- Test single app: `poetry run pytest apps/{app_name}/`
- Lint: `poetry run ruff check .`
- Format: `poetry run ruff format .`
- Migrations: `poetry run python manage.py makemigrations`
- Type check: `poetry run mypy .`

## Directory Structure
- `apps/accounts/` — User auth, profiles, permissions
- `apps/billing/` — Stripe integration, invoices, subscriptions
- `apps/dashboard/` — Analytics views, report generation
- `config/` — Django settings (base, dev, prod), URL routing, WSGI
- `libs/` — Shared utilities used across apps
- `tests/` — Integration tests; unit tests live inside each app's `tests/` dir

## Conventions
- Models: one file per model in `apps/{name}/models/`, imported via `__init__.py`
- Views: class-based views, inherit from `libs.views.BaseAPIView`
- Serializers: DRF serializers adjacent to their models
- All new model fields require a migration and a test
- Use `from __future__ import annotations` in every file
- f-strings for formatting, never `.format()` or `%`

## Constraints
- Never modify `config/settings/prod.py` without review
- Billing app changes require corresponding Stripe webhook test
- All queries must use `.select_related()` or `.prefetch_related()` for FK access
```

## Best Practices

**Do:**
- Front-load build/test commands — these are the highest-value lines in the file
- Use exact commands with flags, not descriptions ("run `pytest -x -q`" not "run the tests")
- Inline critical architecture info rather than referencing other files
- Keep the file under 100 lines — every line costs input tokens on every agent invocation
- Update the file when project structure changes (treat it like CI config)

**Avoid:**
- Vague instructions like "follow standard practices" or "see documentation" — agents need specifics
- Including project history, rationale, or onboarding prose — save that for the README
- Listing every file in the repo — focus on directory-level responsibilities
- Adding instructions that duplicate linter/formatter config — only document what tools don't enforce
- Making the file longer than ~2 pages — diminishing returns set in quickly as agents must process the entire file on every invocation

## Error Handling

- **File not being picked up by the agent:** Check the filename and location match the agent's expected path. Claude Code reads `CLAUDE.md`, Codex reads `AGENTS.md`, Copilot reads `.github/copilot-instructions.md`. Some agents also check `.claude/CLAUDE.md` or nested `AGENTS.md` files.
- **Agent still explores despite documented structure:** The instruction wording may be ambiguous. Use imperative directives ("Tests are in `tests/`") not suggestions ("You might find tests in `tests/`").
- **Token usage increased after adding the file:** The file is too long. Every agent invocation ingests the full instruction file. Cut aggressively — the research showed gains with files focused on conventions, architecture, and project description, not exhaustive documentation.
- **Agent violates stated conventions:** Strengthen the language. Use "ALWAYS" and "NEVER" for hard constraints. Place critical constraints under a dedicated `## Constraints` heading so they stand out.

## Limitations

- The research measured efficiency (runtime and tokens), not correctness. A faster agent is not necessarily a more accurate one — instruction files should be verified against actual agent output quality.
- Results were demonstrated with Codex (gpt-5.2-codex) on PRs with <= 100 LOC changes and <= 5 files. Gains may differ for larger tasks, different agents, or non-PR workflows.
- The study tested on 10 repositories — the optimal content and structure likely varies by project type, language, and framework.
- Instruction files add a maintenance burden. Stale instructions (wrong build commands, outdated directory structure) can actively mislead agents and produce worse outcomes than having no file at all.
- The technique works for repository-level context. It does not replace task-specific prompting for complex, novel, or ambiguous requests.

## Reference

Lulla, J., Mohsenimofidi, S., Galster, M., Zhang, J. M., & Baltes, S. (2026). *On the Impact of AGENTS.md Files on the Efficiency of AI Coding Agents.* arXiv:2601.20404v1. Key finding: AGENTS.md files with conventions, architecture, and project descriptions reduce median agent runtime by 28.64% and output tokens by 16.58% across 124 pull requests.