---
name: "swe-world-building-software-engineering"
description: >
  Build Docker-free software engineering agent workflows that replace containerized execution
  with learned surrogate models. Predicts execution feedback (stdout/stderr/exit codes) and
  test outcomes without running code in containers. Use when asked to: "set up a Docker-free
  agent training pipeline", "predict test results without running them", "build an execution
  surrogate for SWE tasks", "simulate code execution feedback", "implement test-time scaling
  for code generation", "train a software engineering agent without Docker".
---

# SWE-World: Docker-Free Software Engineering Agent Framework

This skill enables Claude to build software engineering agent systems that operate **without Docker containers** by replacing physical code execution with a three-tier surrogate architecture: a deterministic Sandbox for file operations, a learned Transition Model (SWT) that predicts execution feedback, and a learned Reward Model (SWR) that predicts test outcomes. Based on the SWE-World framework, this approach reduces infrastructure costs while preserving the standard agent-environment interaction loop, enabling agent training via SFT, RL, and test-time scaling (TTS).

## When to Use

- When the user wants to build or train a software engineering agent but lacks Docker infrastructure or wants to reduce container overhead
- When evaluating multiple candidate code patches and needing to rank them without executing tests in containers
- When setting up a reinforcement learning pipeline for code modification agents that requires execution feedback
- When implementing test-time scaling to select the best solution from multiple generated attempts
- When building a surrogate environment that predicts stdout/stderr/exit codes for shell commands in a repository context
- When creating a training data pipeline from real Docker rollouts to train execution prediction models
- When the user asks to simulate pytest or test runner output for a given code patch

## Key Technique

SWE-World decomposes the execution environment into three components with different fidelity requirements. **File operations** (ls, cat, grep, str_replace) are deterministic and execute directly in a lightweight Sandbox without any model involvement -- LLM simulation of file contents would hallucinate and derail the agent. **Code execution commands** (running Python scripts, pytest invocations, shell commands) route to the **SWT (Surrogate World Transition)** model, which predicts structured output as `{stdout, stderr, exit_code}` conditioned on the repository state, the agent's current patch, and the command being run. **Final test evaluation** routes to the **SWR (Surrogate World Reward)** model, which generates a structured test report (pytest-style pass/fail per test case) and a binary reward signal.

The critical insight is that SWT and SWR are trained with **chain-of-thought augmentation**: a teacher model generates reasoning traces wrapped in `<think>...</think>` tags before the ground-truth output. This is especially important for SWR, where CoT boosts accuracy by over 13% and prevents reward hacking during RL training. Without CoT, the reward model's low precision causes the RL policy to collapse trajectory length and exploit the signal. The combination of deterministic file ops + learned execution prediction + learned test evaluation creates a complete Docker-free loop that supports SFT, RL (via GRPO++), and test-time scaling (sampling N trajectories and selecting the best via majority-vote SWR scoring).

## Step-by-Step Workflow

### 1. Classify each agent action into the correct execution tier

Route every agent tool call to the appropriate handler:
- **Sandbox** (deterministic): `ls`, `cat`, `grep`, `find`, `view`, `create`, `str_replace_editor` -- execute directly against the repository filesystem
- **SWT** (learned model): `execute_bash`, `python`, `pytest`, any shell command that produces runtime output
- **SWR** (learned model): `submit` -- triggered once at the end of a trajectory to evaluate the final patch

### 2. Build the Sandbox with strict filesystem fidelity

Implement a workspace manager that tracks the repository state as a mutable file tree. Apply `str_replace` edits atomically. Never use an LLM to simulate file reads -- serve actual file contents. Maintain a running diff (the "agent patch") representing all modifications since the initial repo state.

### 3. Collect transition training data from real Docker rollouts

For each code execution step in a real containerized trajectory, record:
```json
{
  "context": {
    "instance_id": "repo__issue_number",
    "problem_statement": "...",
    "initial_analysis": "5-10 bullet points on bug symptoms and fix approach",
    "current_patch": "unified diff of agent edits so far",
    "command": "pytest tests/test_foo.py -x"
  },
  "output": {
    "stdout": "...",
    "stderr": "...",
    "exit_code": 0
  }
}
```
A single rollout yields multiple SWT samples (one per execution step) and one SWR sample (from the final test evaluation).

### 4. Augment training data with chain-of-thought reasoning

Use a strong reasoning model to generate CoT traces for each sample. Format as:
```
<think>
The command runs pytest on test_foo.py. The agent's patch modifies the parse()
function to handle empty input. The F2P test checks empty string parsing.
Given the fix correctly adds an early return for empty strings, this test
should now pass. However, the patch also changes line 42 which affects
the normalize() helper used by test_bar, which may cause a regression...
</think>
{"stdout": "...", "stderr": "...", "exit_code": 0}
```
This CoT is present during training but can be generated at inference time by the surrogate models themselves.

### 5. Train SWT on execution prediction

Fine-tune a code-capable LLM (e.g., Qwen2.5-Coder-32B) on the transition dataset. The model input is the execution context (problem description, initial analysis, current patch, command + relevant code snippets). The model output is structured JSON with stdout/stderr/exit_code. Train on both CoT-augmented and non-CoT samples.

### 6. Train SWR on test outcome prediction

Fine-tune a separate model on reward data. Input includes the full final patch plus the actual unit test code (categorized as Fail-to-Pass and Pass-to-Pass tests). The model generates a structured pytest-style report, then outputs a binary reward `{0, 1}`. CoT augmentation is critical here -- it prevents the model from collapsing to shallow heuristics.

### 7. Wire the surrogate loop for agent training

Replace the Docker environment in your agent training pipeline:
```
Agent Action -> Router
  |-- File operation? -> Sandbox (deterministic)
  |-- Code execution? -> SWT(context) -> predicted {stdout, stderr, exit_code}
  |-- Submit?         -> SWR(final_patch, tests) -> {test_report, reward}
```
The agent sees the same interface as a real Docker environment. No changes to the agent architecture are needed.

### 8. Run Docker-free RL with GRPO++

Use Group Relative Policy Optimization with these stabilizations:
- Leave-one-out advantage estimation across a group of rollouts per problem
- Length normalization to prevent long-horizon bias
- Partial reward: if the agent doesn't submit, assign `reward = 0.5 * SWR_score` to provide learning signal from incomplete trajectories
- Sample 4 parallel rollouts per problem, batch size of 32 problems

### 9. Implement test-time scaling via SWR majority voting

At inference, generate N candidate trajectories (N=8 works well). For each candidate:
1. Run the full trajectory through the surrogate environment
2. Score the final patch with SWR M=3 times
3. Compute `score = mean(reward_1, reward_2, reward_3)`
4. Select the trajectory with the highest average score

This provides monotonic improvement with more candidates and is more reliable than token-probability-based verifiers.

### 10. Validate surrogate fidelity periodically

Compare surrogate predictions against real Docker execution on a held-out set. Track:
- SWT: exact match rate on exit codes, BLEU/ROUGE on stdout
- SWR: precision/recall on binary reward, accuracy on per-test pass/fail
- Agent: resolve rate correlation between surrogate and real evaluation

## Concrete Examples

**Example 1: Building an execution prediction model for a Python repository**

User: "I want to predict what pytest output would look like for a given code patch without actually running the tests."

Approach:
1. Collect the repository source, the code patch (unified diff), and the test file contents
2. Construct the SWT input context:
   ```
   Problem: Function parse_date() raises ValueError on ISO 8601 dates with timezone offsets
   Patch: [unified diff adding timezone handling to parse_date()]
   Command: pytest tests/test_date_parser.py::test_iso8601_tz -xvs
   Relevant code: [test function source + modified function source]
   ```
3. Feed this context to the SWT model, which generates:
   ```json
   {
     "stdout": "tests/test_date_parser.py::test_iso8601_tz PASSED\n\n1 passed in 0.03s",
     "stderr": "",
     "exit_code": 0
   }
   ```
4. The agent uses this predicted feedback to decide whether to submit or continue debugging

Output: Structured execution prediction without Docker, enabling the agent to iterate on its patch.

**Example 2: Ranking multiple patches with test-time scaling**

User: "I generated 8 different patches for this bug fix. How do I pick the best one without running the test suite 8 times?"

Approach:
1. For each of the 8 candidate patches, construct the SWR evaluation context:
   - Include the patch diff, the Fail-to-Pass test code, and the Pass-to-Pass regression tests
2. Score each patch 3 times with SWR (to reduce variance):
   ```
   Patch A: scores [1, 1, 1] -> avg = 1.0
   Patch B: scores [1, 0, 1] -> avg = 0.67
   Patch C: scores [0, 0, 0] -> avg = 0.0
   ...
   ```
3. Select Patch A (highest average score) as the final submission
4. Optionally verify only the top-ranked patch in a real Docker environment

Output: Best patch selected with 3 Docker runs instead of 8, or zero if trusting the surrogate.

**Example 3: Setting up a Docker-free RL training loop**

User: "I want to fine-tune my code agent with RL but I don't have GPU resources to run hundreds of Docker containers in parallel."

Approach:
1. Pre-train SWT and SWR models on collected Docker interaction data (one-time cost)
2. Set up the three-tier routing: Sandbox for file ops, SWT for execution, SWR for rewards
3. Configure GRPO++ with 4 rollouts per problem, 32-problem batches:
   ```python
   config = {
       "lr": 1e-6,
       "rollouts_per_problem": 4,
       "batch_size": 32,
       "max_turns": 150,
       "max_context_tokens": 108000,
       "temperature": 1.0,
       "partial_reward_weight": 0.5,  # for non-submit endings
   }
   ```
4. Run RL training where all execution feedback comes from SWT/SWR models running on the same GPU cluster as the agent -- no Docker containers needed
5. Validate periodically against real Docker evaluation to monitor surrogate drift

Output: RL-trained agent achieving ~55% on SWE-bench Verified using zero Docker resources during training.

## Best Practices

- **Do:** Keep file operations deterministic in the Sandbox. Never use an LLM to predict the output of `cat`, `grep`, or `ls` -- hallucinated file contents will catastrophically mislead the agent.
- **Do:** Include chain-of-thought augmentation when training SWR. Without it, precision drops enough to cause reward hacking during RL, where the agent learns to exploit the model's blind spots instead of actually solving problems.
- **Do:** Use majority voting (M>=3) when scoring candidates with SWR at test time. Single-shot SWR predictions have meaningful variance that averaging smooths out.
- **Do:** Include both Fail-to-Pass and Pass-to-Pass tests in SWR context. Regression test outcomes are essential for distinguishing patches that fix the bug but break other functionality.
- **Avoid:** Training SWT and SWR on the same model checkpoint as the agent itself. The surrogate models should be independently trained to prevent circular dependencies in the learning signal.
- **Avoid:** Skipping the initial analysis step. Providing a structured 5-10 bullet point analysis of the bug symptoms, affected code regions, and fix strategy significantly improves both SWT prediction accuracy and agent performance.

## Error Handling

- **SWT predicts wrong exit code:** The agent may proceed down an incorrect debugging path. Mitigate by training SWT on diverse error types and including relevant source code (not just the command) in the context. Monitor exit code accuracy on held-out data -- below 80% accuracy suggests insufficient training data.
- **SWR gives false positives (predicts pass when tests would fail):** This is the most dangerous failure mode in RL, leading to reward hacking. Use CoT-augmented SWR and validate SWR precision on held-out data. If precision drops below 85%, retrain before continuing RL.
- **SWR gives false negatives:** Less dangerous but wastes good patches. In TTS, majority voting (M=3) reduces false negatives significantly compared to single-shot evaluation.
- **Sandbox state diverges from expected:** If the agent issues commands that modify the filesystem in ways the Sandbox doesn't handle (e.g., `mv`, `chmod`), extend the Sandbox with deterministic handlers for those operations rather than routing them to SWT.
- **Context length overflow:** Long trajectories with many execution steps can exceed model context limits. Truncate older interaction history while preserving the current patch diff and problem statement, which are the most critical context elements.

## Limitations

- **Surrogate fidelity ceiling:** The learned models cannot perfectly replicate real execution. Complex runtime behaviors (concurrency bugs, memory issues, environment-specific failures) are poorly predicted. Best suited for logic bugs, type errors, and API misuse patterns.
- **Training data dependency:** SWT and SWR require real Docker rollout data for initial training. The framework eliminates Docker at training/inference time for the *agent*, but not for building the surrogate models themselves.
- **Repository coverage:** Surrogate accuracy degrades on repositories or languages not well-represented in the training data. Performance is strongest on Python repositories with pytest-style test suites.
- **No real environment side effects:** The surrogate cannot capture actual side effects like network calls, database writes, or file system race conditions. It is limited to predicting textual execution output.
- **Reward model drift:** As the agent policy improves through RL, it may produce patches that fall outside the SWR training distribution, reducing reward accuracy. Periodic retraining with on-policy data is recommended.

## Reference

**Paper:** [SWE-World: Building Software Engineering Agents in Docker-Free Environments](https://arxiv.org/abs/2602.03419v1) (Sun et al., 2026). Focus on Section 3 (three-tier surrogate architecture), Section 4.2 (CoT augmentation for SWR), and Section 5.3 (TTS via majority voting) for the most actionable implementation details. Code: [github.com/RUCAIBox/SWE-World](https://github.com/RUCAIBox/SWE-World).