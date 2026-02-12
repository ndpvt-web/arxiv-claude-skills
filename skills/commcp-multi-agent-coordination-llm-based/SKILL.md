---
name: "commcp-multi-agent-coordination-llm-based"
description: "Build decentralized multi-agent coordination systems using LLM-based communication calibrated with conformal prediction. Agents share only statistically reliable messages, reducing noise and redundancy. Use when: 'coordinate multiple agents with LLM messaging', 'build a multi-agent system with calibrated communication', 'reduce noisy messages between cooperating agents', 'implement conformal prediction for agent communication filtering', 'design decentralized agent coordination without a central controller', 'filter unreliable LLM-generated messages in multi-agent pipelines'."
---

# CommCP: Decentralized Multi-Agent Coordination with Conformal-Prediction-Calibrated LLM Communication

This skill enables Claude to build multi-agent coordination systems where agents communicate through LLM-generated messages that are **statistically calibrated using conformal prediction**. Instead of agents flooding each other with every observation, each message passes through a confidence filter derived from split conformal prediction, guaranteeing that only messages exceeding a calibrated relevance threshold are transmitted. This eliminates the core problem in multi-agent LLM systems: receiver distraction from irrelevant or misleading inter-agent messages.

## When to Use

- When the user asks to build a **multi-agent system** where agents must share observations or findings with each other and noise/redundancy is a concern
- When designing a **decentralized coordination protocol** where no central controller exists and agents must decide independently what to communicate
- When the user wants to **filter or calibrate LLM outputs** before passing them to downstream agents or consumers, using statistical guarantees rather than ad-hoc thresholds
- When implementing **cooperative task allocation** among heterogeneous agents that each have different capabilities or explore different regions
- When building a **multi-agent question-answering or information-gathering pipeline** where agents explore, observe, and share relevant findings
- When the user needs to add **uncertainty quantification** to LLM-generated classifications that drive inter-agent decisions

## Key Technique

**The Core Problem:** In multi-agent systems using LLMs for communication, agents generate messages about their observations (e.g., "I found object X, it may be relevant to your task"). Raw LLM outputs are uncalibrated — a model saying "80% confident" does not mean it is correct 80% of the time. Sending all messages creates noise; naive thresholding misses important information unpredictably.

**Conformal Prediction as the Solution:** CommCP applies **split conformal prediction** to calibrate LLM-generated relevance assessments. During a calibration phase, the system collects LLM confidence scores on labeled examples (where ground truth relevance is known). It computes the quantile threshold `q̂` at level `(1 - ε)` from these calibration scores, where `ε` controls the acceptable miscoverage rate (e.g., `ε = 0.05` for 95% coverage). At runtime, an agent only transmits a message if the LLM's confidence score for that message exceeds `q̂`. This provides a **distribution-free statistical guarantee**: the probability that a truly relevant observation is filtered out is at most `ε`.

**Decentralized Architecture:** Each agent independently runs four modules: (1) a **perception module** that detects objects/entities from observations, (2) a **communication module** that uses an LLM to assess relevance of observations to other agents' tasks and applies conformal filtering before sending, (3) a **planning module** that computes action priorities from its own observations plus received messages, and (4) a **confidence check module** that determines when accumulated evidence is sufficient to commit to an answer or action. No central coordinator is needed — agents exchange peer-to-peer messages, and each receiver trusts incoming messages because they have passed the sender's calibrated filter.

## Step-by-Step Workflow

1. **Define the agent roles and their tasks.** Enumerate each agent's capabilities and assigned objectives. Each agent should have a clear task description string (e.g., "Find and identify the red mug on the kitchen counter") and a set of possible observations it can make.

2. **Design the message schema.** Define a structured message format that agents exchange. At minimum, include: `sender_id`, `receiver_id`, `observation` (what was seen), `relevance_category` (classification of how it relates to the receiver's task), `confidence_score` (LLM's probability for the relevance classification), and `metadata` (position, timestamp, etc.).

   ```python
   @dataclass
   class AgentMessage:
       sender_id: str
       receiver_id: str
       observation: str
       relevance_category: str  # e.g., "same_target", "related", "irrelevant"
       confidence_score: float  # LLM probability for the chosen category
       metadata: dict           # position, timestamp, context
   ```

3. **Implement the LLM relevance assessor.** For each observation an agent makes, prompt the LLM to classify its relevance to every other agent's task. Use chain-of-thought reasoning and request logprob outputs for the classification options. The key is extracting calibrated probability scores, not just the top-1 label.

   ```python
   def assess_relevance(observation: str, other_agent_task: str, llm) -> dict:
       prompt = f"""Given this observation: "{observation}"
       And this agent's task: "{other_agent_task}"

       Classify the relevance:
       A) Same target object — directly fulfills the task
       B) Highly related — provides useful information for the task
       C) Unrelated — no connection to the task
       D) Common/generic — too generic to be useful

       Think step by step, then choose A, B, C, or D."""

       response = llm.generate(prompt, return_logprobs=True)
       probs = extract_option_probabilities(response.logprobs, options=["A","B","C","D"])
       return {"category": max(probs, key=probs.get), "scores": probs}
   ```

4. **Build the conformal calibration set.** Collect 50-200 labeled examples where you know the ground-truth relevance of observations to tasks. Run each through the LLM relevance assessor. Store the confidence scores for positive-class predictions (categories A and B separately).

   ```python
   def calibrate(calibration_data: list, llm, epsilon: float = 0.05) -> dict:
       scores_a, scores_b = [], []
       for obs, task, true_label in calibration_data:
           result = assess_relevance(obs, task, llm)
           if true_label == "A":
               scores_a.append(result["scores"]["A"])
           elif true_label == "B":
               scores_b.append(result["scores"]["B"])

       # Compute quantile thresholds
       q_a = np.quantile(scores_a, epsilon)  # Lower bound for "same target"
       q_b = np.quantile(scores_b, epsilon)  # Lower bound for "related"
       return {"threshold_A": q_a, "threshold_B": q_b}
   ```

5. **Implement the conformal message filter.** Before any message is sent, check whether its confidence score exceeds the calibrated threshold for its category. Only transmit if it passes. This is the core mechanism that provides the statistical guarantee.

   ```python
   def should_send_message(relevance_result: dict, thresholds: dict) -> bool:
       cat = relevance_result["category"]
       score = relevance_result["scores"][cat]
       if cat == "A" and score >= thresholds["threshold_A"]:
           return True
       if cat == "B" and score >= thresholds["threshold_B"]:
           return True
       return False  # Categories C, D are never sent
   ```

6. **Build the agent planning module.** Each agent maintains a priority map over possible actions. Own observations contribute directly; received (calibrated) messages from other agents contribute with a weighting factor. Use semantic value scoring to rank next actions.

   ```python
   def compute_action_priority(own_observations, received_messages, task):
       value_map = {}
       for obs in own_observations:
           value_map[obs.target] = score_relevance(obs, task) * TAU_1
       for msg in received_messages:
           # Received messages already passed conformal filter, so trust them
           value_map[msg.metadata["position"]] = (
               score_relevance(msg, task) * TAU_2  # TAU_2 > TAU_1 rewards communication
           )
       return max(value_map, key=value_map.get)
   ```

7. **Implement the confidence check for task completion.** An agent commits to an answer or action only when `P(answer) * P(relevant_view) > 1 - ε₂`, where `ε₂` is a task-specific confidence threshold. This prevents premature commitment on insufficient evidence.

8. **Wire up the decentralized communication loop.** Each agent runs an independent event loop: observe → assess relevance to peers → filter with conformal threshold → send surviving messages → receive messages from peers → update action priorities → act → check completion.

9. **Run online calibration updates (optional).** As agents accumulate new labeled interactions during deployment, periodically recompute the conformal thresholds on the expanded calibration set to adapt to distribution shifts.

10. **Monitor and log communication efficiency.** Track the message send rate, filter rate, and downstream task success. The ratio of messages sent to messages generated quantifies how much noise the conformal filter eliminates.

## Concrete Examples

**Example 1: Multi-Agent Document Research System**

User: "Build a system where 3 agents each search different document databases, and share relevant findings with each other. Agent 1 searches legal docs, Agent 2 searches financial filings, Agent 3 searches news articles. They're all working on the same investigation query."

Approach:
1. Define each agent with its database scope and the shared investigation query.
2. Each agent retrieves candidate documents and uses an LLM to classify relevance to each other agent's domain focus (e.g., "this financial filing mentions a legal entity from Agent 1's search").
3. Calibrate conformal thresholds using 100 pre-labeled cross-domain relevance examples.
4. Agent 2 finds a filing referencing a legal entity. LLM scores relevance to Agent 1's task at 0.91 (threshold_A = 0.82). Score exceeds threshold → message sent.
5. Agent 3 finds a generic market report. LLM scores relevance to Agent 1 at 0.45 (below threshold). Message suppressed.
6. Each agent incorporates received messages to re-rank its own search priorities.

Output:
```
Agent 2 → Agent 1: "Filing SEC-2024-4821 references entity 'Meridian Holdings'
  matching your legal search. Confidence: 0.91 (threshold: 0.82).
  Relevance: same_target. Source: financial_db/sec_filings/"

Communication stats:
  Messages generated: 47 | Messages sent: 12 | Filter rate: 74.5%
  Task completion: 3/3 agents converged | False leads avoided: ~35
```

**Example 2: Parallel Code Review Agents**

User: "I want multiple agents reviewing different parts of a large PR — one checks security, one checks performance, one checks correctness. They should share findings only when relevant to another reviewer's domain."

Approach:
1. Assign each agent a review focus: security, performance, correctness.
2. As each agent reviews files, it uses an LLM to assess whether findings are relevant to the other agents' focuses (e.g., "this SQL query is both a correctness issue AND a security issue").
3. Calibrate thresholds on 80 examples of cross-domain code review relevance from historical PRs.
4. Security agent finds an unparameterized SQL query. LLM classifies it as "highly related" to correctness agent with score 0.88 (threshold_B = 0.75). Message sent.
5. Performance agent finds a slow loop. LLM classifies relevance to security agent at 0.31. Message suppressed — performance issues rarely imply security issues.

Output:
```python
# Calibrated thresholds (from 80 historical examples, epsilon=0.05):
thresholds = {
    "security→performance": {"A": 0.89, "B": 0.78},
    "security→correctness": {"A": 0.85, "B": 0.75},
    "performance→security": {"A": 0.92, "B": 0.84},
    # ... other pairs
}

# Runtime message log:
# [SENT] security → correctness: "Unparameterized SQL in db/queries.py:142,
#         potential injection AND incorrect escaping" (score=0.88, thresh=0.75)
# [FILTERED] performance → security: "Slow loop in api/handler.py:89"
#            (score=0.31, thresh=0.84)
# Review completed: 6 cross-domain findings shared out of 23 generated (73.9% filtered)
```

**Example 3: Multi-Agent Data Pipeline Monitoring**

User: "Set up agents that monitor different stages of our data pipeline (ingestion, transformation, serving). When one detects an anomaly, it should alert the others only if it's genuinely relevant to their stage."

Approach:
1. Deploy three monitoring agents, one per pipeline stage.
2. When an agent detects an anomaly, it uses an LLM to assess downstream/upstream impact.
3. Calibrate using 60 historical incidents with known cross-stage impact labels.
4. Ingestion agent detects schema drift. LLM assesses relevance to transformation agent at 0.94 (direct impact) and to serving agent at 0.61 (indirect). With thresholds A=0.80 and B=0.72, schema drift alert goes to transformation (passes) but not serving (fails threshold_B).

Output:
```
Alert routing decision:
  Anomaly: schema_drift in ingestion.users_table
  → transformation agent: SEND (score=0.94 ≥ threshold_A=0.80) ✓
  → serving agent: SUPPRESS (score=0.61 < threshold_B=0.72) ✗

  Reason: Schema drift directly breaks transformation mappings but
  serving uses cached transformed data unaffected until next refresh.
```

## Best Practices

**Do:**
- Calibrate thresholds **per communication direction** (Agent A→B may need different thresholds than B→A) because relevance is asymmetric between different task types.
- Use **logprobs from the LLM** rather than asking the model to self-report confidence. Self-reported confidence is poorly calibrated; token logprobs are the raw signal conformal prediction needs.
- Keep calibration sets **representative of deployment conditions**. If your agents will encounter new task types, include diverse examples in calibration.
- Set `ε` conservatively (0.05-0.10) to start. A lower `ε` means fewer relevant messages are accidentally filtered, at the cost of more irrelevant messages getting through.

**Avoid:**
- Do not use a **single global threshold** for all message types. The paper shows that "same target" (category A) and "related" (category B) need separate thresholds (empirically 0.60 vs 0.82 quantiles).
- Do not skip the calibration phase and use **hardcoded thresholds**. The entire value of conformal prediction is that thresholds are derived from data with statistical guarantees, not guessed.
- Do not send **raw LLM reasoning chains** as messages. Send structured, filtered conclusions. The reasoning is for the sender's internal assessment; the receiver needs only the actionable result.
- Do not assume messages are **always helpful**. The paper demonstrates that uncalibrated communication can degrade performance below the no-communication baseline because bad messages actively mislead receivers.

## Error Handling

- **Calibration set too small:** With fewer than 30 calibration examples, conformal thresholds become unreliable. Fall back to a conservative high threshold (e.g., 0.90) and flag that calibration is provisional. Log filtered messages for later review.
- **LLM does not expose logprobs:** Some API providers do not return token-level probabilities. In this case, use a **self-consistency approach**: sample the LLM N times (N=10-20) and use the empirical frequency of each category as the confidence score. Calibrate conformal thresholds on these frequencies instead.
- **Threshold filters everything:** If the conformal threshold is so high that no messages pass, the calibration data may not match deployment distribution. Increase `ε` incrementally (e.g., from 0.05 to 0.15) or expand the calibration set with more representative examples.
- **Threshold filters nothing:** If nearly all messages pass, the LLM may be overconfident on irrelevant items. Add harder negative examples to the calibration set, or switch to a better-calibrated base model.
- **Message delivery failures:** In distributed systems, messages may be lost or delayed. Design receivers to operate correctly with zero messages (graceful degradation). Received messages improve performance but are never required for basic operation.

## Limitations

- **Requires a calibration phase** with labeled examples of message relevance. Fully zero-shot deployment without any calibration data sacrifices the statistical guarantee that makes this approach valuable.
- **Assumes exchangeability** of calibration and test data. If the deployment distribution shifts significantly (new task types, new environments), thresholds degrade and must be recalibrated.
- **LLM latency** adds overhead to every communication decision. In latency-critical systems (sub-100ms requirements), the LLM relevance assessment may be the bottleneck. Consider caching assessments for repeated observation-task pairs.
- **Does not handle adversarial agents.** Conformal prediction calibrates honest uncertainty, not deceptive messages. In settings where agents may be compromised, additional verification is needed.
- **Scales quadratically** in the number of agents for pairwise communication assessment. For large swarms (>10 agents), consider hierarchical grouping or broadcast channels with shared thresholds.

## Reference

**Paper:** [CommCP: Efficient Multi-Agent Coordination via LLM-Based Communication with Conformal Prediction](https://arxiv.org/abs/2602.06038v1) — Zhang et al., ICRA 2026. Focus on Section III (framework architecture), Section III-C (conformal prediction calibration), and Section IV (MM-EQA benchmark results). Key insight: conformal prediction provides distribution-free coverage guarantees for LLM-generated inter-agent messages, reducing communication volume by ~74% while improving task success rates.