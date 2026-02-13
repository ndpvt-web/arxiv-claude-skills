---
name: "live-evo-online-evolution-agentic"
description: "Implement online self-evolving memory for LLM agents using dual-bank architecture (Experience Bank + Meta-Guideline Bank) with reinforcement-weighted retrieval. Use when asked to: 'build an agent that learns from past mistakes', 'add evolving memory to my agent', 'implement experience-weighted retrieval', 'make my agent improve over time from feedback', 'create a self-improving agent pipeline', 'add reinforcement-based memory to my LLM system'."
---

# Live-Evo: Online Evolution of Agentic Memory from Continuous Feedback

This skill teaches Claude to implement **Live-Evo**, an online self-evolving memory system for LLM agents that separates *what happened* (Experience Bank) from *how to use it* (Meta-Guideline Bank). Unlike static memory systems that store and replay past interactions, Live-Evo maintains weighted experiences that are reinforced when helpful and decayed when misleading -- analogous to human memory consolidation. Claude applies this to build agent systems that genuinely improve over successive tasks through contrastive evaluation and selective memory updates.

## When to Use

- When the user wants to build an LLM agent that improves its performance over repeated tasks (e.g., forecasting, research, Q&A pipelines)
- When implementing a memory/retrieval system for agents that must adapt to distribution shift over time
- When the user asks for experience-weighted retrieval where past successes are prioritized over failures
- When building a self-evolving agent that generates and refines its own operational guidelines from feedback
- When the user needs to add contrastive evaluation (memory-on vs. memory-off) to measure whether stored knowledge actually helps
- When designing selective write-back logic so an agent's memory grows only with proven-useful experiences

## Key Technique

Live-Evo's core innovation is **decoupling experience storage from experience usage** via two memory banks. The **Experience Bank** stores structured records of past task interactions -- including the task description, failure analysis, improvement insights, and domain category. The **Meta-Guideline Bank** stores higher-level procedural instructions that tell the agent *how to compile* retrieved experiences into task-specific guidance. This separation means the system can independently improve *what it remembers* and *how it applies memories*.

The weight update mechanism is what makes Live-Evo truly online. For each task, the system runs **contrastive evaluation**: it executes the task both with compiled guidelines (memory-on) and without (memory-off). The performance gap `delta = score_without_memory - score_with_memory` directly adjusts the weights of all retrieved experiences. Positive delta reinforces those experiences (they helped); negative delta decays them (they misled). Retrieval scores are computed as `Score = Weight * Similarity(experience, query)`, so well-reinforced experiences surface more often while stale or harmful ones fade. A minimum threshold (tau = 0.3) filters out low-relevance retrievals entirely.

The **selective write-back protocol** prevents unbounded memory growth. Rather than storing every interaction, Live-Evo identifies the worst-performing fraction of tasks, summarizes their trajectories into candidate experiences, re-evaluates whether adding them actually improves performance, and commits only those that pass a minimum improvement threshold. Failed compilations also trigger generation of new meta-guidelines, so the system learns better ways to *use* its memory, not just better memories.

## Step-by-Step Workflow

1. **Define the experience schema.** Create a structured format for experiences with fields: `task_description`, `outcome`, `failure_reason`, `improvement_insight`, `domain_category`, `weight` (initialized to 1.0), and `embedding` (dense vector from a sentence encoder like all-MiniLM-L6-v2).

2. **Initialize the dual-bank storage.** Set up an Experience Bank (vector store with weighted retrieval) and a Meta-Guideline Bank (collection of procedural templates that instruct the LLM how to compile experiences into task-specific guidelines). Seed the Meta-Guideline Bank with 1-3 default compilation templates (e.g., "Extract common failure patterns from retrieved experiences and formulate avoidance rules for the current task").

3. **Implement multi-dimensional retrieval.** For each incoming task, generate multiple search queries covering different relevance dimensions -- semantic similarity to the task, structural/reasoning pattern matches, and domain overlap. Retrieve top-k experiences ranked by `weight * cosine_similarity(experience_embedding, query_embedding)`, filtering out any below threshold tau.

4. **Build the guideline compilation step.** Select a meta-guideline from the Meta-Guideline Bank, then prompt the LLM to apply it to the retrieved experience set: (a) extract cross-experience regularities, (b) ground findings in the current task's context, (c) produce a concrete, task-specific guideline string that steers downstream decision-making.

5. **Execute with contrastive evaluation.** Run the agent on the task twice: once with the compiled guideline injected into the system prompt (memory-on), once without (memory-off). Record both outcomes and compute the performance gap `delta`.

6. **Update experience weights.** For each experience retrieved in step 3, adjust: `weight_new = weight_old + delta`. Experiences that contributed to better performance get reinforced; those that led to worse performance decay. Clamp weights to a reasonable range (e.g., [0.0, 5.0]) to prevent runaway values.

7. **Reflect on failures and evolve meta-guidelines.** If `delta <= 0` (memory hurt performance), prompt the LLM to analyze why the compilation failed and generate a new meta-guideline that addresses the failure mode. Add it to the Meta-Guideline Bank so future compilations can use improved strategies.

8. **Selectively write back new experiences.** Identify the worst-performing fraction (bottom 30%) of recent tasks. Summarize their trajectories into candidate experience entries. Re-evaluate each candidate by checking if including it improves performance on a held-out or replayed task by at least the minimum threshold (e.g., 0.05 Brier score improvement or equivalent metric). Commit only validated candidates.

9. **Prune stale entries.** Periodically scan the Experience Bank for entries whose weight has decayed below a floor threshold (e.g., 0.1). Archive or remove them to keep the memory bank focused on useful knowledge.

10. **Iterate continuously.** Repeat steps 3-9 for each new incoming task or batch. The system improves online -- no retraining, no static train/test splits required.

## Concrete Examples

**Example 1: Building a self-improving research agent**

User: "I have a deep-research agent that answers complex questions by searching the web. I want it to learn from its mistakes over time so it gets better at finding and synthesizing information."

Approach:
1. Define experience schema capturing each research task: question asked, sources found, answer produced, ground-truth feedback, failure analysis (e.g., "relied on outdated source", "missed key contradiction").
2. Store experiences in a vector DB (e.g., ChromaDB) with weight metadata.
3. For each new question, retrieve top-5 weighted experiences and compile a guideline like: "When researching economic forecasts, cross-reference at least 3 sources from the last 6 months; prior tasks failed when using single-source answers."
4. Run contrastive eval: answer with and without the guideline, compare accuracy.
5. Update weights of retrieved experiences based on whether the guideline helped.

Output structure:
```python
# experience_bank.py
from dataclasses import dataclass, field
from typing import Optional
import numpy as np

@dataclass
class Experience:
    task_description: str
    outcome: str  # "success" or "failure"
    failure_reason: Optional[str]
    improvement_insight: str
    domain: str
    weight: float = 1.0
    embedding: Optional[np.ndarray] = None

class ExperienceBank:
    def __init__(self, encoder, threshold=0.3):
        self.experiences: list[Experience] = []
        self.encoder = encoder  # e.g., SentenceTransformer("all-MiniLM-L6-v2")
        self.threshold = threshold

    def add(self, exp: Experience):
        exp.embedding = self.encoder.encode(exp.task_description)
        self.experiences.append(exp)

    def retrieve(self, query: str, top_k: int = 5) -> list[Experience]:
        q_emb = self.encoder.encode(query)
        scored = []
        for exp in self.experiences:
            sim = np.dot(q_emb, exp.embedding) / (
                np.linalg.norm(q_emb) * np.linalg.norm(exp.embedding) + 1e-8
            )
            weighted_score = exp.weight * sim
            if weighted_score >= self.threshold:
                scored.append((exp, weighted_score))
        scored.sort(key=lambda x: x[1], reverse=True)
        return [exp for exp, _ in scored[:top_k]]

    def update_weights(self, experiences: list[Experience], delta: float,
                       min_w: float = 0.0, max_w: float = 5.0):
        for exp in experiences:
            exp.weight = max(min_w, min(max_w, exp.weight + delta))

    def prune(self, floor: float = 0.1):
        self.experiences = [e for e in self.experiences if e.weight >= floor]
```

**Example 2: Adding evolving memory to a forecasting agent**

User: "My prediction agent makes probability forecasts on real-world events. I want it to learn calibration lessons from past predictions."

Approach:
1. After each resolved prediction, store an experience: the question, predicted probability, actual outcome, Brier score, and a reflection on what went wrong or right.
2. Maintain meta-guidelines like: "When compiling forecasting experiences, focus on calibration errors -- identify if the agent is systematically overconfident or underconfident in the retrieved domain."
3. For new predictions, retrieve relevant past forecasts weighted by their track record.
4. Compile a guideline: "In geopolitical questions, you historically overpredict likelihood by ~15%. Adjust base rates downward."
5. Contrastive eval: forecast with and without guideline, compare Brier scores.
6. Update: if the guideline improved calibration, reinforce those experiences.

Output structure:
```python
# contrastive_eval.py
def contrastive_evaluate(agent, task, compiled_guideline, metric_fn):
    """Run memory-on vs memory-off and return delta."""
    result_on = agent.run(task, system_context=compiled_guideline)
    result_off = agent.run(task, system_context=None)

    score_on = metric_fn(result_on, task.ground_truth)
    score_off = metric_fn(result_off, task.ground_truth)

    # For Brier score (lower is better), delta > 0 means memory helped
    delta = score_off - score_on
    return delta, result_on, result_off
```

**Example 3: Meta-guideline evolution after compilation failure**

User: "My agent's compiled guidelines sometimes make things worse. How do I handle that?"

Approach:
1. When contrastive eval shows `delta <= 0`, trigger meta-guideline reflection.
2. Prompt the LLM: "The following guideline was compiled from these experiences but hurt performance. Analyze why and write a new meta-guideline that avoids this failure mode."
3. Add the new meta-guideline to the bank for future use.

Output structure:
```python
# meta_guideline_bank.py
class MetaGuidelineBank:
    def __init__(self):
        self.guidelines: list[str] = [
            "Extract common failure patterns from retrieved experiences. "
            "Formulate specific avoidance rules grounded in the current task context."
        ]

    def add_from_failure(self, llm, compiled_guideline: str,
                         experiences: list, task: str, outcome: str):
        prompt = (
            f"Task: {task}\n"
            f"Compiled guideline: {compiled_guideline}\n"
            f"Outcome: This guideline HURT performance.\n"
            f"Experiences used: {[e.improvement_insight for e in experiences]}\n\n"
            "Analyze why compilation failed. Write a new meta-guideline "
            "that would avoid this failure mode in future compilations."
        )
        new_guideline = llm.generate(prompt)
        self.guidelines.append(new_guideline)

    def select(self, task_context: str = "") -> str:
        # Simple: rotate or use LLM to pick most relevant
        # Advanced: weight meta-guidelines by their success rate
        return self.guidelines[-1]  # prefer newest for simplicity
```

## Best Practices

- **Do:** Always run contrastive evaluation (memory-on vs. memory-off) before updating weights. Without this causal signal, you cannot distinguish helpful memories from harmful ones.
- **Do:** Use multi-dimensional queries for retrieval -- generate queries targeting semantic similarity, reasoning patterns, and domain overlap rather than a single embedding match.
- **Do:** Initialize experience weights to 1.0 and clamp them within a bounded range (e.g., [0.0, 5.0]) to prevent any single experience from dominating retrieval.
- **Do:** Set a minimum improvement threshold for write-back (e.g., 5% metric improvement) to prevent memory bloat with marginal entries.
- **Avoid:** Storing every task interaction as an experience. Use the selective write-back protocol -- only the bottom 30% of task outcomes warrant experience extraction, and only after validation.
- **Avoid:** Treating the Meta-Guideline Bank as static. The whole point is that *how* you use memory evolves alongside *what* you remember. Generate new meta-guidelines after every compilation failure.

## Error Handling

- **Empty retrieval set:** If no experiences exceed the similarity threshold for a new task, skip guideline compilation and run the agent without memory augmentation. Log the task for potential future experience seeding.
- **Contrastive eval is expensive:** If running the agent twice per task is too costly, batch contrastive evaluation -- run memory-on for all tasks, then periodically sample a subset for memory-off comparison to estimate delta.
- **Weight collapse:** If most experiences decay to near-zero, the Experience Bank has become stale relative to current task distribution. Trigger a refresh: lower the write-back threshold temporarily to admit new experiences, or reset weights to 1.0 for a fresh start.
- **Meta-guideline divergence:** If newly generated meta-guidelines produce increasingly poor compilations, cap the bank size (e.g., 10 guidelines) and remove the worst-performing ones based on their associated delta history.
- **Embedding drift:** If you change or update your embedding model, re-encode all stored experiences to maintain retrieval consistency.

## Limitations

- **Requires measurable feedback:** Live-Evo needs a quantifiable outcome signal (accuracy, Brier score, user rating) for contrastive evaluation. Tasks with purely subjective or delayed feedback are harder to support.
- **Contrastive evaluation doubles compute cost:** Running each task with and without memory is expensive. For high-throughput systems, sampling-based approximations are necessary.
- **Cold start:** With an empty Experience Bank, the system provides no benefit. It needs an initial burn-in period of ~10-20 tasks before retrieval becomes useful.
- **Single-agent assumption:** The paper's weight update formula assumes one agent acting on one compiled guideline. Multi-agent systems with shared memory banks need additional coordination to avoid conflicting weight updates.
- **Not a replacement for fine-tuning:** Live-Evo improves *prompting strategy* through dynamic context injection, not the model's underlying weights. For tasks requiring deep capability gains, model training is still necessary.

## Reference

**Paper:** [Live-Evo: Online Evolution of Agentic Memory from Continuous Feedback](https://arxiv.org/abs/2602.02369v1) -- Zhang et al., 2026. Focus on Algorithm 1 (the four-stage online loop: Retrieve, Compile, Act, Update) and the selective write-back protocol in Section 3.4 for implementation details.