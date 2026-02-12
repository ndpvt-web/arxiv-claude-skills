---
name: "mas-prove-understanding-process-verification"
description: "Design and implement process verification for multi-agent LLM systems. Add intermediate-step evaluation to multi-agent workflows using LLM-as-a-Judge, reward models, or process reward models at agent-level or iteration-level granularity. Trigger phrases: 'add verification to my multi-agent system', 'evaluate intermediate steps in my agent pipeline', 'process verification for MAS', 'verify agent reasoning steps', 'judge agent outputs in a multi-agent workflow', 'add quality checks between agent steps'."
---

# MAS-ProVe: Process Verification for Multi-Agent Systems

This skill enables Claude to design, implement, and integrate process verification into multi-agent LLM systems. Based on the MAS-ProVe empirical study, it teaches how to evaluate intermediate reasoning steps within multi-agent workflows using three verification paradigms — LLM-as-a-Judge, reward models, and process reward models — applied at either agent-level (individual agent outputs) or iteration-level (full round outputs) granularity. The skill encodes the paper's hard-won findings: that process verification is unreliable without careful design, that trained LLM judges outperform general-purpose models and reward-based approaches, and that context management strategy critically affects verification quality.

## When to Use

- When the user is building a multi-agent system and wants to add quality checks on intermediate agent outputs before passing them downstream
- When the user asks how to verify or score reasoning steps in an agent pipeline (debate, chain-of-agents, hierarchical delegation)
- When the user wants to implement best-of-N selection across parallel agent branches using process-level scoring
- When the user is experiencing high variance in multi-agent output quality and wants to add verification gates
- When the user asks about LLM-as-a-Judge patterns for evaluating agent work within a pipeline
- When the user wants to compare verification strategies (reward models vs. LLM judges vs. process reward models) for their multi-agent workflow

## Key Technique

**The Problem.** Multi-agent systems exhibit high variance in reasoning trajectories — the same system can produce wildly different intermediate steps across runs. Process verification (scoring intermediate steps, not just final outputs) has been proposed to tame this variance, but MAS-ProVe demonstrates it does not consistently help and frequently introduces its own variance.

**What Works.** Across six MAS frameworks (Debate, DyLAN, MaAS, AFlow, ADAS, MAS-ZERO), three verification paradigms, and five verifiers, the study finds: (1) LLM-as-a-Judge consistently outperforms reward model and process reward model approaches for multi-agent verification; (2) task-specific trained judges outperform general-purpose LLMs used as judges; (3) there is a context-length-performance trade-off — longer context improves solution diversity but degrades verification accuracy, with accuracy declining beyond ~4K-8K tokens of context. The practical implication: keep verification context lean by using checkpoint-based or sliding-window strategies rather than feeding the full trajectory.

**Granularity Matters.** Agent-level verification (scoring each individual agent's output) gives finer control but higher computational cost and more noise. Iteration-level verification (scoring the output of a complete round/cycle) is coarser but more stable. The choice depends on your architecture: use agent-level for pipelines where individual agent failures cascade (e.g., tool-use chains), and iteration-level for collaborative architectures where agents refine each other's work (e.g., debate, refinement loops).

## Step-by-Step Workflow

1. **Map your MAS architecture to a verification topology.** Identify each agent's role, the data flow between agents, and where intermediate outputs are produced. Draw the directed graph of agent interactions — verification points are the edges.

2. **Choose verification granularity.** For sequential pipelines (Agent A -> Agent B -> Agent C), prefer agent-level verification at each handoff. For iterative systems (debate rounds, refinement loops), prefer iteration-level verification at the end of each round.

3. **Select a verification paradigm.** Default to LLM-as-a-Judge — it is the most reliable approach per MAS-ProVe findings. Use a task-specific fine-tuned judge if available (e.g., Qwen2.5-Math-PRM for math reasoning). Fall back to reward models only if latency constraints prohibit LLM judge calls.

4. **Design the verification prompt or scoring function.** For LLM-as-a-Judge: write a structured evaluation prompt that includes the task description, the agent's input, the agent's output, and a rubric with explicit scoring criteria (correctness, coherence, relevance, completeness). For reward models: select a model trained on your task domain and define the score threshold for acceptance.

5. **Implement context management.** Choose one of four strategies:
   - **Full-context**: Pass the entire trajectory history to the verifier. Only viable for short pipelines (<4K tokens total).
   - **Summary-based**: Condense prior steps into a summary before passing to the verifier. Good for long chains.
   - **Checkpoint-based**: Store strategic state snapshots (e.g., after each major decision point). Best balance of fidelity and compactness.
   - **Sliding-window**: Keep only the most recent N steps. Simplest to implement, works well for iterative systems.

6. **Implement the verification gate.** At each verification point, call the verifier, receive a score, and apply a decision rule: pass/fail threshold for single-path systems, or best-of-N selection for parallel branching systems.

7. **Add fallback behavior for verification failures.** When an intermediate step fails verification: (a) retry the agent with modified instructions, (b) route to an alternative agent, or (c) flag for human review. Never silently drop failed steps.

8. **Instrument with logging.** Record every verification score, the context length passed to the verifier, the latency, and the pass/fail decision. This data is essential for tuning thresholds and diagnosing variance.

9. **Tune verification thresholds empirically.** Run your pipeline on a held-out evaluation set. Plot the distribution of verification scores for correct vs. incorrect trajectories. Set the threshold at the point that maximizes separation. Expect to revisit this as your agents or tasks change.

10. **Monitor for verification overhead vs. benefit.** Track end-to-end accuracy with and without verification, plus the added latency and cost. If verification does not measurably improve accuracy on your task, remove it — MAS-ProVe shows this is a realistic outcome.

## Concrete Examples

**Example 1: Adding LLM-as-a-Judge to a Sequential Agent Pipeline**

User: "I have a 3-agent pipeline: Researcher -> Analyst -> Writer. The Analyst sometimes produces bad analysis that the Writer blindly uses. How do I add verification?"

Approach:
1. Insert an agent-level verification gate between Analyst and Writer.
2. Use LLM-as-a-Judge with a structured evaluation prompt.
3. Apply checkpoint-based context: pass the original query + Researcher output + Analyst output (not the full research trajectory).

```python
import openai

VERIFICATION_PROMPT = """You are a verification judge for a multi-agent analysis pipeline.

Task: {task_description}

Research findings (from Researcher agent):
{researcher_output}

Analysis (from Analyst agent — this is what you are evaluating):
{analyst_output}

Evaluate the analysis on these criteria. Score each 1-5:
1. FACTUAL CONSISTENCY: Does the analysis accurately reflect the research findings?
2. LOGICAL COHERENCE: Are the reasoning steps valid and well-connected?
3. COMPLETENESS: Does the analysis address all aspects of the task?
4. ACTIONABILITY: Can the Writer agent produce useful output from this analysis?

Respond in JSON: {{"scores": {{"factual": N, "coherence": N, "completeness": N, "actionability": N}}, "overall": N, "pass": true/false, "feedback": "..."}}
Set pass=true if overall >= 3.5."""

def verify_analyst_output(task, researcher_output, analyst_output, threshold=3.5):
    prompt = VERIFICATION_PROMPT.format(
        task_description=task,
        researcher_output=researcher_output,
        analyst_output=analyst_output,
    )
    response = openai.chat.completions.create(
        model="gpt-4o",  # or a fine-tuned judge model
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"},
    )
    result = json.loads(response.choices[0].message.content)
    return result

def run_pipeline_with_verification(task, max_retries=2):
    researcher_output = researcher_agent(task)
    for attempt in range(max_retries + 1):
        analyst_output = analyst_agent(task, researcher_output)
        verdict = verify_analyst_output(task, researcher_output, analyst_output)
        if verdict["pass"]:
            break
        # On failure: retry analyst with feedback
        task = f"{task}\n\nPrevious attempt feedback: {verdict['feedback']}"
    return writer_agent(task, researcher_output, analyst_output)
```

**Example 2: Best-of-N Selection in a Debate Architecture**

User: "I have a debate system where 3 agents argue for different solutions. How do I pick the best trajectory after each round?"

Approach:
1. Use iteration-level verification — score the state after each debate round.
2. Run N parallel debate trajectories and select the best-scoring one.
3. Use sliding-window context: only the current round's arguments.

```python
def run_debate_with_verification(task, n_parallel=3, n_rounds=3):
    trajectories = [{"history": [], "score": 0.0} for _ in range(n_parallel)]

    for round_num in range(n_rounds):
        for t in trajectories:
            # Each agent argues in this round
            round_output = run_debate_round(task, t["history"], round_num)
            t["history"].append(round_output)

            # Iteration-level verification: score the round output
            # Sliding-window context: only last 2 rounds
            context_window = t["history"][-2:]
            t["score"] = judge_round(
                task=task,
                round_arguments=context_window,
                round_number=round_num,
            )

        # Prune: keep top trajectory, branch into N copies for next round
        best = max(trajectories, key=lambda t: t["score"])
        trajectories = [
            {"history": list(best["history"]), "score": best["score"]}
            for _ in range(n_parallel)
        ]

    return extract_final_answer(trajectories[0]["history"])

def judge_round(task, round_arguments, round_number):
    prompt = f"""Evaluate this debate round for task: {task}

Round {round_number} arguments:
{format_arguments(round_arguments)}

Score 0.0-1.0 based on: convergence toward a correct answer,
quality of counter-arguments, identification of flaws in prior reasoning.
Respond with just the numeric score."""
    # Call your judge model here
    return float(call_judge(prompt))
```

**Example 3: Integrating with the MAS-ProVe Framework Directly**

User: "I want to use the MAS-ProVe codebase to add process verification to my existing multi-agent system."

Approach:
1. Install `mas_proceval` and start the appropriate judge server.
2. Use `BaseClient` and `llm_parallel_search_decorator` to instrument your agent functions.
3. Choose agent-level vs. iteration-level by decorating individual agent calls or full-round functions.

```bash
# Install the framework
pip install -e .  # from the MAS-ProVe repo root

# Start a judge server (pick one):
python -m mas_proceval.servers.server_judge    # LLM-as-a-Judge (GPT-based)
python -m mas_proceval.servers.server_prm      # Process Reward Model (Qwen2.5-Math-PRM-7B)
python -m mas_proceval.servers.server_rm       # Reward Model (Skywork-Reward-V2)
```

```python
from mas_proceval import BaseClient, llm_parallel_search_decorator

client = BaseClient()

# Agent-level: decorate individual agent functions
@llm_parallel_search_decorator
def my_analyst_agent(task, context):
    # Your agent logic here
    return analysis_result

# Iteration-level: decorate the full round function
@llm_parallel_search_decorator
def run_full_iteration(task, agent_outputs):
    # Orchestrate all agents for one round
    return combined_output

# The decorator handles:
# - Sending intermediate outputs to the judge server
# - Receiving verification scores
# - Enabling parallel search across solution branches
```

## Best Practices

**Do:**
- Start with LLM-as-a-Judge as your default verification paradigm — it consistently outperforms reward models and PRMs across MAS architectures in the MAS-ProVe study.
- Use checkpoint-based or sliding-window context management to keep verifier input under 4K-8K tokens. Verification accuracy degrades with longer contexts.
- Log all verification scores and decisions. Use this data to set thresholds empirically rather than guessing.
- Provide the verifier with explicit, structured rubrics — vague prompts like "is this good?" produce noisy scores.

**Avoid:**
- Do not assume process verification will automatically improve your system. MAS-ProVe shows it frequently does not, and can increase variance. Always A/B test with and without verification.
- Do not pass the full trajectory history to the verifier for long pipelines. The context-length-performance trade-off is real — more context means worse verification accuracy past a threshold.
- Do not use a general-purpose LLM as a judge when a task-specific fine-tuned judge is available. Trained judges consistently outperform general-purpose models.
- Do not apply agent-level verification to every agent in a large system without measuring the cost-benefit ratio. The computational overhead can exceed the accuracy gains.

## Error Handling

- **Verifier timeout or failure**: Implement a bypass with logging — if the verifier is unavailable, pass the output through unverified and flag it. Never block the entire pipeline on a verification failure.
- **All N branches fail verification**: Set a maximum retry count, then accept the highest-scoring branch with a degraded-confidence flag rather than entering an infinite retry loop.
- **Score distribution collapse**: If the verifier assigns nearly identical scores to all outputs (good and bad), the rubric is likely too vague or the verifier model is too weak for the task. Switch to a stronger judge model or make the rubric more specific.
- **Context truncation artifacts**: When using sliding-window context, earlier steps may lose critical context. If verification quality drops in later rounds, switch to checkpoint-based context that preserves key decision points.

## Limitations

- **No guarantee of improvement.** The core finding of MAS-ProVe is that process verification does not consistently help. On simple tasks or well-tuned pipelines, it may add cost without benefit.
- **Verification itself has variance.** LLM judges are stochastic — the same intermediate output can receive different scores across runs. Use temperature=0 and multiple verification calls with majority voting to reduce this.
- **Task-domain dependency.** Results from MAS-ProVe are primarily on mathematical reasoning benchmarks (AIME) and multi-step reasoning (GAIA). Effectiveness on code generation, creative writing, or other domains is not established.
- **Latency cost.** Each verification call adds an LLM inference round-trip. For latency-sensitive applications, consider asynchronous verification or post-hoc verification rather than inline gates.
- **The gap between LLM-as-judge and single-agent is small.** MAS-ProVe observes that LLMs acting as judges perform only slightly better than the same LLM acting as a single agent — suggesting the multi-agent overhead may not justify itself for simpler tasks.

## Reference

**Paper:** [MAS-ProVe: Understanding the Process Verification of Multi-Agent Systems](https://arxiv.org/abs/2602.03053v1) — Venkataramani et al., 2026. Look for Table 2 (verifier comparison across frameworks), Figure 3 (context-length-performance trade-off), and Section 5 (practical recommendations for verification strategy selection).

**Code:** [github.com/Wang-ML-Lab/MAS-ProVe](https://github.com/Wang-ML-Lab/MAS-ProVe) — Client-server verification framework with decorator-based integration for six MAS architectures.