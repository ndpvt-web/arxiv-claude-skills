---
name: "self-evolving-recommendation-system-end-to-end"
description: "Build autonomous ML optimization pipelines that use LLM agents to generate, evaluate, and deploy model improvements in a dual-loop (offline/online) workflow. Use when: 'build a self-evolving recommendation system', 'automate hyperparameter and architecture search with LLMs', 'set up an autonomous ML experiment loop', 'create an LLM-driven model optimization pipeline', 'design a reward function search agent', 'automate recommendation model iteration'."
---

# Self-Evolving Recommendation System: End-to-End Autonomous Model Optimization

This skill enables Claude to design and implement autonomous ML optimization systems that use LLM agents to generate code-level model changes (optimizer tweaks, architecture modifications, reward function redesigns), evaluate them against proxy metrics offline, promote winners to live A/B tests, and feed production outcomes back into the next generation cycle. The technique comes from YouTube's production system where Gemini agents act as specialized ML engineers, discovering novel improvements that surpass manual iteration in both velocity and quality.

## When to Use

- When the user wants to build an automated pipeline that iterates on a recommendation model without manual intervention
- When the user asks to use LLMs to generate and test ML architecture or hyperparameter candidates programmatically
- When the user needs a dual-loop system: fast offline screening followed by online A/B validation
- When the user wants to automate reward function design for engagement optimization
- When the user is building a system that generates, trains, and evaluates ML model variants in a loop
- When the user asks how to structure an agent that proposes code-level changes to a training pipeline and self-evaluates

## Key Technique

The core insight is a **dual-loop agent architecture** that separates high-throughput exploration from rigorous production validation. The **Inner Loop (Offline Agent)** rapidly generates candidate model modifications as actual code diffs — not just hyperparameter values, but structural changes to optimizers, activation functions, loss formulations, and network topology. Each candidate is trained on a proxy task with lightweight metrics (validation loss, offline AUC, computational cost) that correlate with production outcomes but evaluate in hours instead of days. The LLM agent uses chain-of-thought reasoning to explain its modification rationale before generating code, which improves solution quality and provides an audit trail.

The **Outer Loop (Online Agent)** takes the top-performing candidates from the inner loop and deploys them to live traffic via A/B experiments. It monitors delayed north-star business metrics (long-term engagement, retention, satisfaction) that cannot be captured offline. Statistical significance testing determines whether a candidate graduates to full production. Crucially, the outcomes — both successes and failures — feed back as context to the inner loop agent, creating an evolutionary pressure that focuses subsequent generations on promising directions.

What makes this different from standard AutoML is that the LLM agent generates **semantically meaningful code changes** rather than searching a predefined parameter grid. It can invent new activation function compositions, restructure attention mechanisms, or formulate multi-objective reward functions that a grid search would never explore. The agent receives structured context about the current model architecture, recent experiment history, performance bottlenecks, and implementation constraints, enabling it to reason about what to try next rather than searching blindly.

## Step-by-Step Workflow

1. **Define the model specification context**: Create a structured document describing the current model architecture (layers, activations, optimizer, loss function, feature schema) in a machine-readable format (JSON or YAML). This becomes the base context the LLM agent receives with every generation prompt.

2. **Build the hypothesis generation prompt template**: Construct a prompt that includes: (a) current model spec, (b) recent experiment results with metrics, (c) known failure modes to avoid, (d) implementation constraints (latency budget, memory limits, backward compatibility requirements). End with an instruction to produce a concrete code diff and a written rationale.

3. **Implement the inner-loop code generation pipeline**: Write a harness that calls the LLM to produce candidate modifications as code patches. Parse the output into an executable change (e.g., a Python function replacement, a config override, or a new module definition). Validate syntax and run basic sanity checks before training.

4. **Set up proxy metric evaluation**: Define 2-4 fast-to-compute proxy metrics that correlate with your production objectives (e.g., offline NDCG, validation cross-entropy, inference latency). Train each candidate on a sampled dataset and score against these proxies. Implement automated stopping rules — kill runs that diverge or exceed resource budgets.

5. **Rank and filter candidates**: After the inner loop produces N candidates (typically 10-50 per cycle), rank by a weighted combination of proxy metrics. Apply Pareto filtering if objectives conflict (e.g., quality vs. latency). Select the top K (typically 1-3) for online testing.

6. **Deploy to online A/B experimentation**: Push winning candidates to a controlled traffic split. Define the north-star metrics to monitor (engagement duration, return visits, conversion rate). Set minimum experiment duration and statistical significance thresholds before any candidate can graduate.

7. **Implement the feedback loop**: After each online experiment concludes, serialize the results (candidate description, code diff, proxy metrics, online metrics, pass/fail verdict) into the experiment history log. This log becomes part of the context for the next inner-loop generation cycle.

8. **Add the evolution memory**: Maintain a structured archive of all past hypotheses, their rationales, proxy scores, and online outcomes. Summarize this into a compact context window (recent successes, recent failures, known dead ends) so the LLM avoids repeating failed directions and builds on successful patterns.

9. **Orchestrate the full loop**: Wire the inner loop and outer loop into an automated pipeline with scheduling. The inner loop runs continuously (e.g., daily batches of candidates). The outer loop picks up promoted candidates and manages experiment lifecycle. Add alerting for anomalies (training failures, metric regressions, resource overruns).

10. **Implement safety guardrails**: Add hard constraints before any candidate reaches production: latency regression limits, memory budget checks, output distribution sanity checks, and a rollback mechanism that reverts to the previous model if online metrics degrade beyond a threshold.

## Concrete Examples

**Example 1: Automated optimizer discovery for a video recommendation model**

User: "I have a video recommendation model trained with Adam. Build me an agent loop that generates optimizer modifications, tests them offline, and promotes the best to an A/B test."

Approach:
1. Define the model context as a JSON spec including current optimizer config (Adam, lr=1e-3, weight_decay=1e-5), model architecture summary, and recent training metrics.
2. Create the hypothesis generator:

```python
GENERATION_PROMPT = """
You are an ML engineer optimizing a video recommendation model.

Current optimizer: {optimizer_config}
Model architecture: {architecture_summary}
Recent experiments:
{experiment_history}

Known constraints:
- Training must complete in under 4 hours on 8 GPUs
- Inference latency must stay under 10ms p99

Generate ONE optimizer modification as a Python code block that replaces
the current optimizer factory function. Explain your rationale first,
then provide the code.
"""

def generate_candidate(llm_client, context):
    response = llm_client.generate(
        GENERATION_PROMPT.format(**context),
        temperature=0.8  # moderate creativity
    )
    rationale, code = parse_rationale_and_code(response)
    return Candidate(rationale=rationale, code=code)
```

3. Run 20 candidates through offline training on a 10% data sample, scoring on validation NDCG@10 and training wall-clock time.
4. Promote the top 2 to an A/B experiment with 1% traffic each for 7 days.
5. Log all results back to the experiment history for the next cycle.

Output: An autonomous loop that might discover, for example, a per-layer learning rate schedule with cosine warmup that improves NDCG@10 by 0.3% — a change an engineer might not have prioritized manually.

---

**Example 2: Reward function search for long-term engagement**

User: "Our recommendation model optimizes click-through rate but we want to optimize for long-term engagement. Build an agent that designs and tests reward functions."

Approach:
1. Define available signals: click, watch_time, likes, shares, return_visit_24h, return_visit_7d, session_length.
2. Prompt the LLM to compose reward functions as weighted combinations or nonlinear transformations of these signals:

```python
REWARD_PROMPT = """
Available engagement signals and their statistics:
{signal_descriptions}

Current reward function:
  reward = click * 1.0

The goal is long-term user retention, not just immediate clicks.

Design a new reward function as a Python function that takes a dict of
signals and returns a scalar reward. You may use nonlinear combinations,
thresholds, or time-decayed weighting. Explain why your formulation
targets long-term engagement before providing code.
"""
```

3. Generate 30 candidate reward functions. Train the model with each for 2 epochs on proxy data. Measure offline metrics: predicted return_visit_7d correlation, reward distribution entropy (to avoid degenerate solutions), and Gini coefficient of item exposure (to penalize filter bubbles).
4. Promote top 3 candidates to a 14-day A/B test measuring actual 7-day return rate.
5. Feed back which formulations improved retention and which degraded it.

Output: The agent might discover a reward like `0.3 * watch_time_normalized + 0.5 * log1p(return_visit_24h) + 0.2 * share_indicator` — a formulation that explicitly rewards retention signals over clicks.

---

**Example 3: Architecture modification search**

User: "Search for architecture improvements to our transformer-based ranking model. Automate the process."

Approach:
1. Describe the current architecture: 6-layer transformer, 128-dim embeddings, ReLU activations, dot-product attention scoring.
2. Prompt the LLM to propose one architectural change per candidate (e.g., swap activation functions, add cross-attention layers, modify embedding factorization, insert mixture-of-experts layers).
3. Constrain candidates: inference latency must not increase by more than 15%, parameter count must stay under 50M.
4. Evaluate 15 candidates on offline ranking metrics (NDCG, MRR) and latency benchmarks.
5. Promote the best to an online test.

Output: The agent might discover that replacing ReLU with SwiGLU in the feed-forward layers and adding a lightweight cross-feature attention block yields a 0.5% MRR improvement within the latency budget.

## Best Practices

- **Do:** Provide rich structured context to the LLM agent — current model spec, recent experiment results with exact metric values, and explicit constraints. The quality of generated candidates directly correlates with context quality.
- **Do:** Use chain-of-thought prompting — require the agent to explain its reasoning before generating code. This produces better candidates and makes failures diagnosable.
- **Do:** Maintain an experiment archive that includes failures. Telling the agent "these 12 approaches were tried and degraded metrics" prevents it from repeating dead ends.
- **Do:** Set hard safety guardrails (latency limits, rollback triggers, resource budgets) that cannot be overridden by the agent. The agent proposes; the infrastructure disposes.
- **Avoid:** Letting the agent modify multiple components simultaneously in a single candidate. Atomic changes are easier to evaluate, debug, and attribute.
- **Avoid:** Using only offline proxy metrics to make launch decisions. The dual-loop design exists because proxy metrics are imperfect — always validate online before full deployment.
- **Avoid:** Running the system without human review of the experiment archive. Periodically audit what the agent is trying, what patterns it has converged on, and whether the proxy metrics still correlate with production outcomes.

## Error Handling

- **Generated code fails syntax validation**: Retry with the error message appended to the prompt. If 3 retries fail, log the failure and move to the next candidate. This is common and expected — treat it as part of the throughput cost.
- **Candidate causes training divergence**: Implement early stopping based on loss trajectory. If training loss exceeds 2x the baseline after 10% of training, kill the run and record the failure in the experiment history.
- **Proxy metrics disagree with online results**: This indicates proxy-metric drift. Recalibrate by running a set of known-good and known-bad configurations through both pipelines and adjusting proxy metric weights or thresholds.
- **Agent converges on a narrow family of similar proposals**: Increase generation temperature, inject diversity-encouraging instructions ("propose something fundamentally different from the last 5 candidates"), or reset the recent experiment context to force exploration.
- **Online experiment shows metric regression**: Automated rollback to the previous model version. Record the regression in the experiment history with the specific metrics that degraded so the agent avoids similar directions.

## Limitations

- **Requires existing ML infrastructure**: This approach assumes you already have a training pipeline, evaluation framework, and A/B testing platform. It automates the hypothesis-generation layer on top of that infrastructure — it does not replace the infrastructure itself.
- **Proxy metric quality is critical**: If your offline metrics do not correlate with production outcomes, the inner loop will optimize for the wrong thing. You need validated proxy metrics before this system adds value.
- **LLM context window limits**: The experiment history grows over time and must be summarized to fit within context limits. Poor summarization loses important patterns; this requires careful prompt engineering.
- **Not suitable for cold-start systems**: The dual-loop approach assumes a baseline model with established metrics. For new recommendation systems without historical performance data, manual iteration is still needed to establish the initial baseline.
- **Cost scales with candidate volume**: Each LLM call and training run has compute cost. Budget 10-50 candidates per inner-loop cycle and tune based on your hit rate.

## Reference

- **Paper**: [Self-Evolving Recommendation System: End-To-End Autonomous Model Optimization With LLM Agents](https://arxiv.org/abs/2602.10226v1) — Focus on the dual-loop agent architecture (Inner Loop for offline hypothesis generation, Outer Loop for online validation) and the feedback mechanism that enables evolutionary convergence across cycles.