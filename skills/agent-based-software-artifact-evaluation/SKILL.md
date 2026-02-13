---
name: "agent-based-software-artifact-evaluation"
description: "Automatically evaluate software research artifacts (code repositories with READMEs) by constructing dependency-aware command graphs, building containerized environments, and executing instructions with structured error recovery. Use when asked to: 'evaluate this artifact', 'reproduce this paper's results', 'run this repo's README instructions', 'check if this artifact builds and runs', 'automate artifact evaluation', 'verify research reproducibility'."
---

# Agent-Based Software Artifact Evaluation

This skill enables Claude to systematically evaluate software artifacts -- code repositories accompanying research papers -- by applying the ArtifactCopilot methodology. Instead of naively executing README instructions top-to-bottom, Claude constructs an **Artifact Evaluation Graph** (a dependency-aware command graph) from the README, builds a containerized environment, and executes commands in topological order with structured state tracking and error recovery. This approach matches human artifact evaluation outcomes 85% of the time, compared to ~33% for unstructured execution.

## When to Use

- When the user asks to **reproduce results from a research paper** given its code repository
- When evaluating whether a GitHub repository's README instructions actually work end-to-end
- When the user provides an artifact (repo + README) and asks "does this build and run?"
- When automating evaluation of multiple software artifacts for a conference review process
- When debugging why a repository's instructions fail and systematically recovering from errors
- When the user needs to verify that a repository's claimed outputs (figures, tables, benchmarks) are reproducible

## Key Technique: Artifact Evaluation Graphs and Execution Normalization

The core insight from ArtifactCopilot is that README documents are **narrative prose with embedded commands**, not structured execution plans. Humans maintain implicit mental models of execution state, but automated tools lose track of context, especially across Docker container boundaries. The solution is to transform the README into an **Artifact Evaluation Graph** G=(V,E) with three node types:

- **Start nodes**: Entry points carrying metadata (e.g., whether Docker is needed)
- **Command nodes**: Executable instructions with environment context and status attributes
- **Artifact nodes**: Files consumed or produced, with path and type information

Edges encode three relationship types: **sequential** (execution order), **artifact-input** (data dependency from artifact to command), and **artifact-output** (production from command to artifact). This graph enables topological execution, selective continuation after failures (skip only affected downstream nodes), and structured state tracking at the node level.

The second key technique is **execution normalization**: all commands are issued from the host via container execution APIs rather than entering interactive Docker sessions. This eliminates the invisible context switches (host vs. container filesystem, environment variables) that cause most automated evaluation failures. For containers with custom entrypoints, a detached shell session replays the original entrypoint, then commands are injected sequentially.

## Step-by-Step Workflow

1. **Acquire and inspect the repository.** Clone the target repository, identify the primary README file (check `README.md`, `INSTALL.md`, `ARTIFACT.md`, and subdirectories). Read the README fully before extracting any commands.

2. **Parse the README into an Artifact Evaluation Graph.** Using chain-of-thought reasoning, extract every command from the README. For each command, identify: (a) the execution environment (host, container, specific shell), (b) input artifacts it depends on (datasets, config files, model weights), (c) output artifacts it produces (figures, logs, tables). Build the graph with sequential edges for ordering and artifact edges for data dependencies.

3. **Validate artifact paths against the repository.** Check that every artifact node in the graph corresponds to an actual file or directory in the repo. For mismatches, perform name-based search to find the correct path and update the graph. Flag missing datasets or external dependencies that must be downloaded.

4. **Construct the execution environment.** Apply three strategies in order: (a) If a `Dockerfile` exists, reuse it -- extract the base image and entrypoint, build the image. (b) If no Dockerfile but dependency manifests exist (`requirements.txt`, `environment.yml`, `package.json`), synthesize a Dockerfile from them. (c) If both fail after 3 attempts, fall back to an Ubuntu 22.04 base image and install dependencies incrementally.

5. **Normalize the execution context.** Issue all commands from the host using `docker exec` rather than entering interactive containers. Map file paths between host and container. For containers with custom entrypoints, start a detached shell session that replays the entrypoint, then inject commands through that session.

6. **Execute commands in topological order.** Traverse the AE Graph, executing each command node in dependency order. Track execution status (pending, running, succeeded, failed) at the node level. After each command, verify expected output artifacts exist.

7. **Detect stalled execution.** Monitor resource utilization (CPU) across intervals. If utilization drops to near-zero for a sustained period during a long-running command, analyze logs to determine if execution is stalled or waiting for interactive input. Inject responses to interactive prompts (e.g., `y` for confirmation, default values for configuration wizards).

8. **Recover from errors with targeted repair.** On command failure, retry up to 5 times. Analyze the error trace to generate a targeted fix (install missing dependency, adjust path, fix permissions) rather than blind retries. If a command ultimately fails, mark it and identify all downstream nodes that depend on it -- skip those but continue executing independent branches of the graph.

9. **Collect and compare outputs.** After execution completes, collect all produced artifacts. Compare against expected outputs described in the README (tables, figures, benchmark numbers). Allow reasonable numerical tolerance for non-deterministic results.

10. **Generate an evaluation report.** Produce a structured report: which commands succeeded/failed, which artifacts were produced, whether outputs match expectations, and an overall reproducibility assessment (Reproducible / Partially Reproducible / Not Reproducible).

## Concrete Examples

**Example 1: Evaluating a machine learning paper's artifact**

```
User: Can you evaluate the artifact at github.com/example/ml-paper?
The README says it trains a model and produces accuracy tables.

Approach:
1. Clone the repo and read the README fully.
2. Build the AE Graph:
   - Start -> install dependencies -> download dataset ->
     train model -> evaluate -> generate Table 1
   - Artifact edges: dataset -> train, model.pt -> evaluate,
     results.json -> generate Table 1
3. Find Dockerfile in repo root. Build image.
4. Normalize: execute all commands via docker exec.
5. Execute in order. Dataset download (wget) succeeds.
   Training command runs for 10 minutes -- monitor CPU to
   confirm active computation.
6. Evaluation produces results.json. Table generation
   produces table1.csv matching expected format.

Output:
## Artifact Evaluation Report

| Step | Command | Status | Notes |
|------|---------|--------|-------|
| 1 | pip install -r requirements.txt | Passed | All deps installed |
| 2 | bash download_data.sh | Passed | Dataset: 2.3GB |
| 3 | python train.py --config default | Passed | 10m23s runtime |
| 4 | python evaluate.py | Passed | Accuracy: 94.2% (paper: 94.5%) |
| 5 | python generate_tables.py | Passed | table1.csv produced |

**Verdict: Reproducible**
All commands executed successfully. Results within expected
tolerance of reported values (0.3% accuracy difference).
```

**Example 2: Artifact with Docker context-switching issues**

```
User: This repo's instructions say to run "docker run -it myimage"
then execute commands inside the container. Can you evaluate it?

Approach:
1. Read README. Instructions say:
   docker run -it myimage bash
   cd /app && python setup.py install
   python run_experiments.py
2. Build AE Graph. Identify the implicit context switch
   at "docker run -it" -- all subsequent commands target
   the container.
3. Normalize execution. Instead of interactive docker run:
   - docker build -t myimage .
   - docker run -d --name eval_container myimage tail -f /dev/null
   - docker exec eval_container bash -c "cd /app && python setup.py install"
   - docker exec eval_container bash -c "cd /app && python run_experiments.py"
4. Execute normalized commands sequentially.
5. setup.py install fails: missing libffi-dev.
   Error recovery: docker exec eval_container apt-get install -y libffi-dev
   Retry setup.py install -- succeeds.
6. Experiments complete, results produced.

Output:
## Artifact Evaluation Report

| Step | Command | Status | Notes |
|------|---------|--------|-------|
| 1 | docker build | Passed | Image built from Dockerfile |
| 2 | setup.py install | Failed->Recovered | Missing libffi-dev, auto-installed |
| 3 | run_experiments.py | Passed | All 5 experiments completed |

**Verdict: Reproducible** (1 auto-recovered error)
```

**Example 3: Artifact with missing dependencies and partial failure**

```
User: Evaluate this artifact. It has 3 experiments but
experiment 2 requires a GPU and a proprietary dataset.

Approach:
1. Parse README into AE Graph with 3 independent experiment
   branches sharing a common setup phase.
2. Build environment from requirements.txt (no Dockerfile).
   Synthesize Dockerfile: FROM python:3.9, COPY requirements.txt,
   RUN pip install -r requirements.txt.
3. Execute setup phase -- succeeds.
4. Experiment 1: succeeds, produces expected figure.
5. Experiment 2: fails on torch.cuda.is_available() check.
   Error is fundamental (no GPU) -- mark as failed after
   5 retries. Downstream artifact (table2.csv) marked skipped.
6. Experiment 3: independent of Exp 2, continues execution.
   Succeeds, produces expected output.

Output:
## Artifact Evaluation Report

| Step | Command | Status | Notes |
|------|---------|--------|-------|
| 1 | pip install -r requirements.txt | Passed | |
| 2 | python experiment1.py | Passed | figure1.png produced |
| 3 | python experiment2.py | Failed | Requires GPU (CUDA not available) |
| 4 | python experiment3.py | Passed | table3.csv produced |

**Verdict: Partially Reproducible**
2/3 experiments reproduced. Experiment 2 requires GPU hardware
not available in current environment. This is an infrastructure
limitation, not a code defect.
```

## Best Practices

- **Do:** Always read the entire README before extracting commands. Instructions often have forward references, conditional branches, and implicit ordering that only make sense in full context.
- **Do:** Normalize all Docker interactions to host-side `docker exec` commands. Never enter interactive container sessions -- this is the single largest source of automated evaluation failures.
- **Do:** Build the full dependency graph before executing anything. Commands that appear independent in the README may share artifact dependencies.
- **Do:** Use targeted error repair based on error trace analysis. A missing Python package needs `pip install`, a missing system library needs `apt-get install` -- don't retry the same failing command without changing something.
- **Avoid:** Executing commands in README order without checking dependencies. A README may describe setup for Experiment 2 before completing Experiment 1's execution steps.
- **Avoid:** Treating all failures as fatal. When one branch of the AE Graph fails, continue executing independent branches. Report partial reproducibility rather than giving up entirely.
- **Avoid:** Silently substituting placeholder values (like `<YOUR_PATH>`) without flagging them to the user. These require explicit user input.

## Error Handling

| Error Type | Detection | Recovery |
|------------|-----------|----------|
| Missing system package | Error trace mentions missing `.so` or header file | `apt-get install` the package, retry |
| Missing Python/Node dependency | `ModuleNotFoundError` or `Cannot find module` | Install from manifest or error message, retry |
| Interactive prompt blocking | Low CPU utilization sustained over monitoring interval | Inject default response (`y`, Enter, or `1`), retry |
| Docker context confusion | Command-not-found errors after `docker run` | Re-normalize to `docker exec` pattern |
| Path mismatch | `FileNotFoundError` on an expected artifact | Search repo for filename, update path in graph |
| Network timeout | Connection refused or timeout during download | Retry with exponential backoff (3 attempts) |
| Out of memory | OOM killer or memory allocation failure | Report as infrastructure limitation, skip downstream |
| Permission denied | EACCES or sudo requirement | Add appropriate permissions or run with elevated context |

Limit retries to 5 per command. After 5 failures, mark the command as permanently failed and continue with independent graph branches.

## Limitations

- **GPU-dependent artifacts** cannot be evaluated without GPU hardware. The framework correctly identifies this as an infrastructure limitation but cannot work around it.
- **Proprietary datasets** or artifacts requiring external credentials (API keys, licensed data) cannot be evaluated without user-provided access.
- **Non-deterministic outputs** (e.g., ML training with random seeds not fixed) may produce numerically different results that are still correct. Use reasonable tolerances.
- **Very large artifacts** (multi-hour training runs, TB-scale datasets) may exceed practical execution time and storage limits.
- **Poorly documented repositories** with minimal or inaccurate READMEs will produce low-quality AE Graphs. The framework depends on README quality.
- **Interactive GUIs or notebook-based workflows** (Jupyter notebooks with manual cell execution) are not well-suited to this automated pipeline.

## Reference

**Paper:** "Agent-Based Software Artifact Evaluation" by Wu et al. (2026). [arXiv:2602.02235v2](https://arxiv.org/abs/2602.02235v2)

**Key takeaway:** The paper's core contribution is showing that transforming unstructured README instructions into a dependency-aware graph (the Artifact Evaluation Graph), combined with execution normalization to eliminate Docker context-switching problems, enables automated artifact evaluation that matches human outcomes 85% of the time. The graph structure is what enables selective continuation after failures and structured state tracking -- without it, automated tools lose context and fail at 3x the rate.