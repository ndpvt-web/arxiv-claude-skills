---
name: "evoconfig-self-evolving-multi-agent-systems"
description: "Autonomous environment configuration using multi-agent diagnosis and self-evolving error repair. Use when: 'set up the dev environment for this repo', 'configure dependencies and get tests passing', 'debug why my environment build fails', 'create a Dockerfile for this project', 'fix my broken Python environment', 'auto-configure this repository so tests run'."
---

# EvoConfig: Self-Evolving Multi-Agent Environment Configuration

This skill enables Claude to autonomously configure runtime environments for software repositories using the EvoConfig framework's multi-agent architecture. Instead of treating environment setup as a single monolithic task, EvoConfig separates execution control from diagnostic analysis: a **Main Configuration Agent** generates and executes setup commands, while an **Expert Diagnostic Agent** performs fine-grained post-execution analysis, classifying each outcome as success, failure, or potential risk, and producing structured repair recommendations. A self-evolving feedback loop lets the diagnostic agent refine its error-fixing priorities dynamically based on accumulated experience -- without external memory modules or additional token overhead.

## When to Use

- When the user asks to set up a development environment for an unfamiliar repository
- When environment configuration fails with dependency conflicts, missing packages, or toolchain mismatches
- When generating a reproducible Dockerfile from a repository's source code
- When debugging why `pytest`, `tox`, or other test runners fail due to environment issues rather than code bugs
- When configuring complex Python projects that use multiple dependency management tools (pip, poetry, conda, setuptools)
- When a user says "get the tests passing" on a freshly cloned repository
- When iteratively fixing cascading dependency errors that simple `pip install` cannot resolve

## Key Technique

**Separation of Execution and Diagnosis.** Traditional single-agent approaches accumulate raw stdout, stderr, and exit codes into the agent's context, polluting its reasoning with noise. EvoConfig decouples these concerns: the Main Configuration Agent performs ReAct-style reasoning with a compact context window, generating atomic shell commands. After execution, the Expert Diagnostic Agent receives the commands, exit codes, and output, then produces a structured diagnostic report -- classifying each result as success, failure, or potential risk. Only this high-level summary feeds back to the main agent, preserving context quality.

**Self-Evolving Error Repair.** After each diagnostic cycle, the expert agent incrementally adjusts its internal rules for repair suggestion generation, tool creation, and risk assessment. This means the system gets better at fixing errors *within a single configuration session* -- learning, for instance, that a repository's build system requires `poetry install` before `pip install -e .`, or that a specific C library must be installed via `apt` before a Python wheel can compile. Priorities shift dynamically: if the same dependency error recurs after an attempted fix, it escalates in priority and triggers alternative repair strategies.

**Prior Environment Extraction.** Before issuing any commands, the system extracts a structured summary from the repository: dependency management strategy M (from pyproject.toml, requirements.txt, poetry.lock, setup.cfg), project importability I (src layout, package structure), and test structure hypothesis T (test location, framework, module imports). This prior guides all downstream decisions.

## Step-by-Step Workflow

1. **Extract environment priors from the repository.** Scan for `pyproject.toml`, `requirements.txt`, `setup.py`, `setup.cfg`, `poetry.lock`, `conda.yml`, `Makefile`, `tox.ini`, and `Dockerfile`. Identify the dependency management strategy (pip, poetry, conda, or hybrid), the project layout (flat vs src/), and the test framework (pytest, unittest, nose).

2. **Formulate the initial configuration plan.** Based on the priors, draft an ordered sequence of setup commands: base image or Python version selection, system dependency installation, dependency manager invocation, project installation, and test execution. This is the main agent's first action set.

3. **Execute commands atomically and sequentially.** Run each command one at a time in the target environment. Capture the full stdout, stderr, and exit code for each command. Do not batch commands -- atomic execution ensures precise diagnosis.

4. **Perform expert diagnosis on each execution result.** For every command, classify the outcome into one of three states:
   - **Success**: Command achieved its goal (e.g., package installed, tests discovered).
   - **Failure**: Command errored (e.g., dependency not found, compilation failed). Generate a specific repair command.
   - **Potential risk**: Command succeeded but output suggests a latent problem (e.g., deprecation warning, version pinning conflict). Flag for monitoring.

5. **Generate structured diagnostic report.** Summarize all classifications, repair commands, and risk flags into a compact report. Include only actionable information -- strip raw logs, keep error names, package versions, and suggested fixes.

6. **Feed the diagnostic summary back to the main agent.** The main agent receives the structured report (not raw output) and decides the next action: execute a repair command, adjust the installation strategy, or proceed to the next phase.

7. **Apply self-evolving rule adjustment.** After each feedback cycle, update internal priorities: if a repair failed, escalate that error type and try an alternative strategy (e.g., switch from pip to conda, pin a different version, install a system-level library). Track which strategies succeeded for this repository and prefer them in subsequent rounds.

8. **Create diagnostic tools on demand.** When the expert agent needs more information to diagnose a failure, generate single-line diagnostic commands (e.g., `python -c "import pkg; print(pkg.__version__)"`, `dpkg -l | grep libssl`, `pip show numpy`). These collect evidence only -- never perform repairs.

9. **Iterate until tests pass or the budget is exhausted.** Repeat steps 3-8 for up to t_max rounds (typically 10-15). Stop early if all tests pass. If the budget is exhausted, output the best configuration achieved along with a list of unresolved issues.

10. **Consolidate into a reproducible artifact.** Collect all successful commands into a Dockerfile (or shell script), ordered by execution sequence, with comments explaining each step. Validate the artifact by building it from scratch.

## Concrete Examples

**Example 1: Configuring a Python repository with Poetry and native extensions**

```
User: "Set up the dev environment for this repo so the tests pass.
       It uses Poetry and has some C extensions."

Approach:
1. Extract priors: find pyproject.toml with [tool.poetry], poetry.lock present,
   tests/ directory with pytest markers, src/ layout detected.
2. Formulate plan:
   - Install system deps (build-essential, libffi-dev)
   - Install poetry
   - Run `poetry install --with dev`
   - Run `pytest tests/`
3. Execute `apt-get install -y build-essential libffi-dev` → Success
4. Execute `pip install poetry` → Success
5. Execute `poetry install --with dev` → Failure: "SolverProblem: package
   cryptography requires OpenSSL >= 1.1.1"
6. Expert diagnosis: Failure classified as "missing system library".
   Repair suggestion: `apt-get install -y libssl-dev`
7. Execute repair → Success
8. Re-execute `poetry install --with dev` → Success
9. Execute `pytest tests/` → 142 passed, 3 failed (test failures are code
   bugs, not environment issues)
10. Output: Dockerfile consolidating the successful command sequence.

Output (Dockerfile):
FROM python:3.11-slim
RUN apt-get update && apt-get install -y build-essential libffi-dev libssl-dev
RUN pip install poetry
COPY . /app
WORKDIR /app
RUN poetry install --with dev
CMD ["pytest", "tests/"]
```

**Example 2: Debugging cascading dependency failures**

```
User: "I keep getting import errors when I try to run tests on this repo.
       pip install -r requirements.txt fails halfway through."

Approach:
1. Extract priors: requirements.txt with 47 pinned packages, no lock file,
   setup.py present, tests use unittest.
2. Execute `pip install -r requirements.txt` → Failure at line 23:
   "Could not build wheel for pandas==1.3.5"
3. Expert diagnosis: Failure type = "compilation error". Root cause =
   missing numpy build dependency. Risk flag: pandas 1.3.5 incompatible
   with Python 3.12.
4. Self-evolving adjustment: Escalate Python version mismatch to high
   priority (affects many downstream packages).
5. Repair strategy: Switch to Python 3.10 base, then retry.
6. Execute `pip install -r requirements.txt` under Python 3.10 → Failure
   at line 31: "grpcio==1.48.0 requires protobuf<4.0"
7. Expert diagnosis: Version conflict. Repair: install grpcio before
   protobuf, letting pip resolve the constraint.
8. Execute `pip install grpcio==1.48.0` → Success
9. Execute `pip install -r requirements.txt` → Success (grpcio already
   satisfied, protobuf version auto-resolved)
10. Execute `python -m pytest` → All 89 tests pass.

Diagnostic summary provided to user:
- Root cause: Python 3.12 incompatible with pandas 1.3.5 and several
  other pinned packages
- Secondary issue: grpcio/protobuf version conflict resolved by
  installation ordering
- Recommendation: Pin Python 3.10 in the project config and update
  pandas to >=2.0 when ready
```

**Example 3: Generating a Dockerfile from a repo with no existing containerization**

```
User: "Create a Dockerfile for this Flask app so I can deploy it.
       There's no Docker setup yet."

Approach:
1. Extract priors: setup.cfg with install_requires, no pyproject.toml,
   requirements-dev.txt for test deps, app/ directory with __init__.py,
   tests/ with pytest, .python-version says 3.9.
2. Formulate plan: Python 3.9 base, install prod deps, install dev deps
   separately, verify with pytest.
3. Execute `pip install -e .` → Potential risk: "setup.cfg uses
   legacy metadata format, consider migrating to pyproject.toml"
4. Expert diagnosis: Success with risk flag. No action needed now.
5. Execute `pip install -r requirements-dev.txt` → Failure:
   "psycopg2 requires libpq-dev"
6. Repair: `apt-get install -y libpq-dev` → Success
7. Re-execute dev deps install → Success
8. Execute `pytest tests/` → Success
9. Consolidate into multi-stage Dockerfile:

Output (Dockerfile):
FROM python:3.9-slim AS base
RUN apt-get update && apt-get install -y libpq-dev && rm -rf /var/lib/apt/lists/*
COPY . /app
WORKDIR /app
RUN pip install --no-cache-dir -e .

FROM base AS test
RUN pip install --no-cache-dir -r requirements-dev.txt
RUN pytest tests/

FROM base AS production
EXPOSE 5000
CMD ["gunicorn", "app:create_app()", "-b", "0.0.0.0:5000"]
```

## Best Practices

- **Do:** Always extract environment priors before issuing any setup commands. Scanning pyproject.toml, setup.cfg, and lock files first avoids blind trial-and-error.
- **Do:** Execute commands atomically (one at a time) so each failure can be precisely diagnosed. Chaining commands with `&&` obscures which step failed.
- **Do:** Classify execution results into the three-state model (success/failure/risk). The "potential risk" category catches problems like deprecation warnings that become real failures later.
- **Do:** Track which repair strategies worked and reuse them. If `apt-get install libssl-dev` fixed one crypto-related error, apply it proactively when similar packages appear.
- **Avoid:** Dumping raw terminal output back into your reasoning context. Summarize diagnostics into structured reports with error type, affected package, and suggested fix.
- **Avoid:** Retrying the same failed command without changing something. After a failure, the self-evolving mechanism must try an alternative: different package version, different installer, or additional system dependency.

## Error Handling

| Error Pattern | Diagnosis | Repair Strategy |
|---|---|---|
| Compilation failure ("Failed building wheel") | Missing system library or incompatible Python version | Install system dev packages (`-dev` libs), or switch Python version |
| Version conflict ("requires X>=2.0 but Y needs X<2.0") | Dependency solver deadlock | Relax one pin, try installing conflicting packages in a specific order, or use a resolver like `pip-compile` |
| Import error after successful install | Broken package, namespace collision, or missing `__init__.py` | Verify with `python -c "import X"`, check sys.path, reinstall the package |
| Test discovery failure | Wrong test directory, missing conftest.py, or framework mismatch | Check pytest.ini / setup.cfg for testpaths, verify test framework matches |
| Timeout during large dependency install | Network issues or extremely large builds (e.g., torch) | Use pre-built wheels, add `--timeout` flags, or use `--prefer-binary` |
| Missing configuration files | Repository expects files not in version control (.env, config.yaml) | Generate minimal stubs based on `.env.example` or config templates in the repo |

## Limitations

- **Hardware-dependent builds**: Repositories requiring GPU drivers, specific CPU architectures, or large memory allocations (32.4% of EvoConfig's failures) cannot be resolved through command-line configuration alone.
- **Missing test suites**: If a repository has no tests, the framework cannot verify that the environment is correctly configured. It can only confirm that the package installs and imports.
- **Non-Python ecosystems**: The priors extraction and diagnostic rules are optimized for Python projects. Applying this to Node.js, Rust, or Java requires adapting the file scanning and error classification heuristics.
- **Private/authenticated dependencies**: Repositories depending on private PyPI indexes or authenticated Git URLs require credentials that the agent cannot autonomously obtain.
- **Extremely large dependency trees**: Projects with 200+ transitive dependencies may exhaust the iteration budget before resolving all conflicts.

## Reference

**Paper:** [EvoConfig: Self-Evolving Multi-Agent Systems for Efficient Autonomous Environment Configuration](https://arxiv.org/abs/2601.16489v1) (Guo et al., 2026). Focus on Section 3 for the multi-agent architecture, Section 3.3 for the self-evolving mechanism, and Table 6 for the failure mode taxonomy.