---
name: "termigen-high-fidelity-environment-robust"
description: "Synthesize verifiable Docker-based task environments and error-resilient terminal agent trajectories using TermiGen's multi-agent refinement loop and Generator-Critic protocol. Use when: 'create a training environment for terminal tasks', 'generate error-recovery training data', 'build a Docker-based coding challenge', 'synthesize resilient agent trajectories', 'create self-correcting terminal workflows', 'generate verifiable CLI task environments'."
---

# TermiGen: Synthesizing Verifiable Environments and Error-Resilient Terminal Trajectories

This skill enables Claude to apply the TermiGen pipeline for two core capabilities: (1) generating functionally valid, Docker-containerized task environments through iterative multi-agent refinement, and (2) synthesizing terminal agent trajectories enriched with deliberate error-injection and recovery cycles using a Generator-Critic protocol. The technique addresses the fundamental problem that training data built from expert-only trajectories leaves models unable to recover from their own runtime mistakes. By deliberately injecting domain-specific errors at a controlled rate (epsilon=0.2) and forcing recovery, the resulting data teaches models not just how to solve tasks, but how to diagnose and recover from failures.

## When to Use

- When the user needs to create reproducible, Docker-based environments for terminal/CLI task evaluation or training
- When building training datasets for terminal agents that must handle errors gracefully
- When designing coding challenges or benchmarks that require verifiable success criteria
- When the user wants to generate synthetic trajectories that include realistic mistake-and-recovery patterns
- When creating CI/CD test harnesses that validate command-line tool usage across diverse domains
- When the user asks to build a self-contained task environment with automated validation (unit tests, exit codes)
- When fine-tuning or evaluating models on terminal interaction tasks and needing diverse, high-quality data

## Key Technique

**Environment Synthesis via Multi-Agent Refinement Loop.** TermiGen generates task environments through a three-stage pipeline. First, a Task Proposal Agent expands seed categories from a hierarchical taxonomy (spanning infrastructure, data/algorithms, and specialized development) into structured specifications scored on Environment Complexity, Data Generatability, and Verification Determinism (all must exceed 4/5). Second, three sequential agents materialize the environment: a File Planner (filesystem blueprint), a Construct Agent (actual file content), and an Env Agent (self-contained Dockerfile). Third, an iterative Docker build loop captures stderr and feeds it back for correction (up to 5 rounds), achieving 100% functional validity. A separate Unit Test Generator and Judge Agent pair ensures tasks are both solvable and discriminative.

**Trajectory Synthesis via Generator-Critic Protocol with Error Injection.** At each step of trajectory collection, a Bernoulli coin flip (epsilon=0.2) determines whether the agent should advance the task correctly or commit a deliberate, domain-specific error. Errors fall into five categories: Analysis Errors (misreading environment state), Command Errors (syntax/formatting), Hallucinations (assuming nonexistent tools), Requirement Violations (ignoring constraints), and Verification Failures (skipping self-checks). A Critic Agent validates that error steps are syntactically plausible and produce informative stderr feedback, while correct steps genuinely advance the task. When an error executes, the environment transitions to a failure state, and subsequent correct-intent steps must diagnose the error from stderr and synthesize corrective actions. Because sampling is independent per step, consecutive errors naturally occur, exposing the model to cascading failure modes.

**Why This Works.** Standard expert trajectories create a distributional mismatch: the model only sees perfect execution but must handle its own imperfect outputs at inference. TermiGen's error-correction cycles close this gap. Ablation studies show that even including failed trajectories (those that never reach the correct answer) improves performance, because they contain valid local error-correction segments that provide diagnosis and recovery supervision.

## Step-by-Step Workflow

### Phase 1: Environment Synthesis

1. **Define the task taxonomy.** Create a hierarchical category structure with 2-3 tiers (e.g., "Infrastructure > Container Orchestration > Multi-stage Builds"). Generate seed descriptions that are concise abstractions of what the task tests.

2. **Expand seeds into structured task specifications.** For each seed, produce: a clear objective statement, required input artifacts, expected output/success criteria, and a difficulty estimate. Score each proposal on three axes (Environment Complexity 1-5, Data Generatability 1-5, Verification Determinism 1-5). Reject anything scoring below 4 on any axis; provide feedback and retry up to 3 rounds.

3. **Materialize the filesystem blueprint.** Decompose the task into a directory structure listing every file needed (source code, configs, data files, Makefiles). Specify the purpose and content requirements of each file.

4. **Generate file contents and Dockerfile.** Instantiate actual file content for each blueprint entry. Write a self-contained Dockerfile that installs all system-level dependencies, copies task files, and sets up the working environment. Use explicit version pins for reproducibility.

5. **Validate via iterative Docker build loop.** Build the Docker image. If the build fails, capture the full stderr log, diagnose the root cause, patch the Dockerfile or task files, and rebuild. Repeat up to 5 iterations until the build succeeds with exit code 0.

6. **Generate and validate unit tests.** Write unit tests that verify the task's success criteria (checking output files, exit codes, specific stdout patterns). Run tests against both a known-correct solution and a plausible-but-wrong solution to confirm discriminative power. If the tests fail validation, regenerate with feedback.

### Phase 2: Trajectory Synthesis

7. **Collect trajectories with error injection.** Execute the task inside the Docker container using ReAct-style interaction (thought + bash command per step). At each step, sample intent: with probability 0.8 advance the task correctly, with probability 0.2 inject a domain-specific error from the five failure categories. Generate the action conditioned on current state and sampled intent.

8. **Apply critic validation to each step.** For error-intent steps, verify the action is syntactically plausible and that execution produces informative error feedback (not a silent failure). For correct-intent steps, confirm the action meaningfully advances the task state. Regenerate any step that fails critic validation.

9. **Capture error-recovery transitions.** When an injected error executes, record the full stderr/stdout. On the next correct-intent step, the agent must: (a) analyze the error output, (b) diagnose the root cause in its reasoning trace, and (c) produce a corrective command. This error-diagnosis-recovery triplet is the core training signal.

10. **Assemble the final dataset.** Include both successful trajectories (task completed) and failed trajectories (task not completed but containing valid recovery segments). Do not filter out failures -- they provide coverage of harder tasks and local recovery supervision that improves downstream performance.

## Concrete Examples

**Example 1: Creating a Verifiable Docker Environment for a Log Parsing Task**

User: "Create a Docker-based environment that tests whether an agent can parse nginx access logs, extract the top 10 IPs by request count, and output them sorted."

Approach:
1. Define task specification:
   - Objective: Parse `/var/log/nginx/access.log`, extract client IPs, count occurrences, output top 10 sorted descending
   - Input: Synthetic nginx access log (~10,000 lines with realistic patterns)
   - Success criteria: Output file matches expected sorted IP list exactly
   - Scores: Complexity 4/5 (standard tools), Generatability 5/5 (logs are easy to synthesize), Determinism 5/5 (exact match)

2. Generate filesystem blueprint:
```
/task/
  access.log          # Synthetic nginx log (10K lines, 50 unique IPs, varied distribution)
  expected_output.txt # Ground truth: top 10 IPs sorted by count
  verify.sh           # Diff agent output against expected_output.txt
Dockerfile            # Ubuntu 22.04, coreutils, awk, sort pre-installed
```

3. Write Dockerfile:
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y coreutils gawk
COPY task/ /task/
WORKDIR /task
```

4. Build, verify, generate unit test that runs `verify.sh` and checks exit code 0.

**Example 2: Generating an Error-Resilient Trajectory for Package Configuration**

User: "Generate a training trajectory for configuring a Python project with Poetry, including realistic mistakes and recovery."

Approach (showing the Generator-Critic protocol in action):

```
Step 1 [Intent: Correct]
Thought: I need to initialize a Poetry project. Let me check if Poetry is installed.
Command: poetry --version
Output: Poetry (version 1.8.2)
Critic: Valid - confirms tool availability before use.

Step 2 [Intent: Correct]
Thought: Poetry is available. Let me initialize the project.
Command: poetry init --name myproject --no-interaction
Output: Created pyproject.toml
Critic: Valid - advances task state.

Step 3 [Intent: Error (Command Error, epsilon=0.2 triggered)]
Thought: Now I'll add the requests dependency.
Command: poetry add request
Output: Could not find a matching version of package request
Critic: Valid error - plausible typo (missing 's'), produces informative error message.

Step 4 [Intent: Correct]
Thought: The package name was wrong - it should be 'requests' not 'request'. The error message confirms no package named 'request' exists. Let me correct this.
Command: poetry add requests
Output: Using version ^2.31.0 for requests
Critic: Valid - correctly diagnoses typo from error output and recovers.

Step 5 [Intent: Error (Hallucination, epsilon=0.2 triggered)]
Thought: Let me verify the installation by running the test suite.
Command: poetry run pytest tests/
Output: ERROR: file or directory not found: tests/
Critic: Valid error - assumes test directory exists without checking.

Step 6 [Intent: Correct]
Thought: There's no tests/ directory yet. I assumed it existed. Let me first check the project structure, then create the test directory.
Command: ls -la && mkdir -p tests && touch tests/__init__.py
Output: [directory listing, then successful creation]
Critic: Valid - recovers by verifying state before acting.
```

**Example 3: Multi-Round Docker Build Refinement**

User: "Set up a Rust-based environment for testing binary search tree implementations."

Iterative build loop in action:
```
Round 1 - Build fails:
  stderr: "error: linker `cc` not found"
  Diagnosis: Base image lacks build-essential/gcc
  Fix: Add `RUN apt-get install -y build-essential` before cargo build

Round 2 - Build fails:
  stderr: "error[E0433]: failed to resolve: use of undeclared crate `serde`"
  Diagnosis: Cargo.toml references serde but it's not in dependencies
  Fix: Add `serde = { version = "1.0", features = ["derive"] }` to Cargo.toml

Round 3 - Build succeeds (exit code 0)
  Validation: Run unit tests against correct BST impl (pass) and broken impl missing rebalance (fail)
  Result: Environment verified as both functional and discriminative
```

## Best Practices

- **Do:** Score every task proposal on all three quality axes (Complexity, Generatability, Determinism) before investing effort in environment creation. Reject early and iterate on the spec rather than debugging a poorly-conceived environment.
- **Do:** Pin exact versions in Dockerfiles (`python:3.11.7-slim`, not `python:3-slim`) to ensure reproducibility across time and machines.
- **Do:** Keep the error injection rate around 0.2 (1 in 5 steps). This was empirically validated -- too many errors make trajectories unrecoverable; too few fail to teach recovery.
- **Do:** Include failed trajectories in training data. Counter-intuitively, trajectories that never solve the task still contain valuable local error-correction segments.
- **Avoid:** Injecting random noise as errors. Errors must be domain-specific and plausible (typos in package names, missing `sudo`, wrong file paths) -- not gibberish commands that no real model would produce.
- **Avoid:** Simulating environment outputs instead of executing them. Simulated observations introduce three error types: Spurious Verbosity (53% of simulation errors -- hallucinating confirmations for silent commands), Semantic Deviation (35% -- misrepresenting exit codes), and State Inconsistency (12% -- breaking logical chains). Always execute in real containers.
- **Avoid:** Filtering to only perfect trajectories. Strict filtering biases toward easy tasks and discards recovery supervision from harder ones.

## Error Handling

| Failure Mode | Diagnosis | Recovery |
|---|---|---|
| Docker build fails after 5 iterations | Dependency conflict or unavailable package | Simplify the environment: reduce dependencies, switch base image, or use pre-built images with required tools |
| Unit test passes for both correct and incorrect solutions | Test is not discriminative | Add edge-case assertions, test for specific output format, or add negative test cases that check for known wrong behaviors |
| Error-injected step produces silent failure (no stderr) | The error is not informative enough for recovery learning | Regenerate with a different error category; prefer Command Errors or Hallucinations that produce explicit error messages |
| Critic rejects valid steps repeatedly | Critic is overly conservative or misaligned | Review critic prompt; ensure it validates alignment with intent rather than judging solution quality |
| Trajectory exceeds token budget | Task is too complex for single-trajectory capture | Break the task into subtasks, each with its own trajectory; or increase the sequence length budget |

## Limitations

- Environment synthesis requires actual Docker execution, so this workflow cannot run in sandboxed or serverless contexts without Docker access.
- The five error categories (Analysis, Command, Hallucination, Requirement Violation, Verification Failure) may not cover all failure modes for highly specialized domains (e.g., hardware description languages, exotic build systems).
- The technique is designed for terminal/CLI interactions using ReAct-style (thought + command) trajectories. It does not directly apply to GUI-based tasks, API-only workflows, or multi-turn conversational agents.
- Generating high-quality environments at scale requires a capable backbone model for all agent roles; using weaker models for synthesis degrades environment validity and error plausibility.
- The 0.2 error rate was tuned on TerminalBench tasks spanning 420 CLI tools. Optimal epsilon may differ for narrower or broader task distributions.

## Reference

**Paper:** [TermiGen: High-Fidelity Environment and Robust Trajectory Synthesis for Terminal Agents](https://arxiv.org/abs/2602.07274v1) (Zhu et al., 2026). Key sections: Section 3.1 for the three-stage environment synthesis pipeline, Section 3.2 for the Generator-Critic error-injection protocol, and Section 4.2 for ablation results showing why real execution and imperfect trajectories matter. Dataset: [github.com/ucsb-mlsec/terminal-bench-env](https://github.com/ucsb-mlsec/terminal-bench-env).