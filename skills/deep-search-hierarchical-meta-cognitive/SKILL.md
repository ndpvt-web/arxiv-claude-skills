---
name: "deep-search-hierarchical-meta-cognitive"
description: "Implement hierarchical meta-cognitive monitoring for deep search agents. Embeds a two-tier self-monitoring system (fast consistency checks + slow experience-driven reflection) into multi-step retrieval-reasoning loops to detect anomalies, prevent reasoning drift, and trigger corrective interventions. Use when: 'build a deep search agent with self-monitoring', 'add metacognitive monitoring to my search pipeline', 'detect and fix reasoning failures in multi-step retrieval', 'implement DS-MCM for search quality', 'add anomaly detection to my RAG agent', 'build a self-correcting research agent'."
---

# Deep Search with Hierarchical Meta-Cognitive Monitoring (DS-MCM)

This skill enables Claude to build and apply a two-tier metacognitive monitoring framework for deep search agents. Inspired by cognitive neuroscience — where the anterior cingulate cortex performs fast conflict detection and the prefrontal cortex handles deliberate control — DS-MCM embeds a Fast Consistency Monitor (lightweight anomaly detection at every reasoning step) and a Slow Experience-Driven Monitor (selectively triggered corrective reflection using past trajectory memory) directly into the retrieval-reasoning loop. This prevents common failure modes like confidence-evidence misalignment, query loops, contradiction blindness, and reasoning drift, while adding only 3-7% time overhead.

## When to Use

- When building a multi-step search agent (ReAct, tool-use, or agentic RAG) and you need it to detect when its own reasoning is going off-track
- When the user asks to "add self-monitoring" or "self-correction" to a retrieval pipeline
- When implementing a research agent that must handle conflicting or ambiguous sources without silently producing wrong answers
- When a search agent gets stuck in repetitive query loops or hallucinates confidence despite weak evidence
- When building a production search system that needs to know when to escalate, reformulate queries, or seek alternative sources
- When the user wants to apply the DS-MCM paper's technique to their existing LLM agent framework

## Key Technique

**The Core Insight**: A deep search agent operates correctly when its internal reasoning uncertainty is *calibrated* to the uncertainty of external evidence. When retrieved documents are highly ambiguous (high Searching Entropy), the model's reasoning should reflect proportional uncertainty (high Reasoning Entropy). A mismatch — e.g., the model is overconfident despite contradictory sources, or confused despite clear evidence — signals a metacognitive failure requiring intervention.

**Two-Tier Architecture**: The Fast Consistency Monitor runs at every step with minimal cost. It computes Searching Entropy (SE) by clustering retrieved documents in embedding space and measuring Shannon entropy over semantic clusters, and Reasoning Entropy (RE) from the token-level probability distributions during reasoning generation. A learned linear calibration function (RE_hat = a*SE + b) predicts expected reasoning entropy given evidence entropy. When the residual |RE - RE_hat| exceeds a threshold (k=2 standard deviations from successful trajectory baselines), an anomaly is flagged.

**Experience-Driven Correction**: Only when the Fast Monitor flags an anomaly does the Slow Experience-Driven Monitor activate. It maintains dual memory banks — M+ (successful trajectories) and M- (failed trajectories) — each storing session state, context summaries, and LLM-generated cognitive abstractions of what went right or wrong. The current anomalous state is encoded and matched against both banks via cosine similarity. A Critical Model receives the current state plus retrieved positive and negative examples, then outputs an error classification and a natural-language corrective suggestion that conditions the agent's next action. This keeps expensive reflection rare (triggered only on anomalies) while making corrections informed by concrete past experience.

## Step-by-Step Workflow

1. **Set up the ReAct reasoning-retrieval loop**: Implement or identify the existing agent loop: query formulation -> reasoning/thought -> action (search/retrieve/terminate) -> observation/tool feedback -> next step. DS-MCM hooks into the gap between reasoning and action.

2. **Implement Searching Entropy (SE) computation**: After each retrieval step, take the top-K retrieved documents (K=5), embed them using a sentence embedding model, cluster them using k-means or DBSCAN in embedding space, compute cluster probability distribution p(c_i), and calculate Shannon entropy: `SE = -sum(p_i * log(p_i))`. High SE means retrieved evidence is semantically fragmented/contradictory.

3. **Implement Reasoning Entropy (RE) computation**: During the reasoning generation step, capture the token-level probability distributions over the top-K vocabulary tokens. Compute per-token entropy and average across the generated reasoning trace: `RE = mean(-sum(p_k * log(p_k)))` for each token position. High RE means the model is internally uncertain.

4. **Fit the calibration function**: Collect a set of successful agent trajectories (even 50-100 is useful). For each step, compute (SE, RE) pairs. Fit a simple linear regression `RE_hat = a*SE + b`. Compute the standard deviation sigma of residuals `epsilon = RE - RE_hat` across successful trajectories. Set the anomaly threshold `tau = 2 * sigma`.

5. **Deploy the Fast Consistency Monitor**: At each reasoning step in the live agent, compute SE and RE, calculate the residual `epsilon = RE - RE_hat`, and flag an anomaly if `|epsilon| > tau`. Log the anomaly type: overconfident (RE too low for given SE) or under-confident (RE too high for given SE).

6. **Build the experience memory banks**: From historical trajectories, label each step as belonging to a successful or failed trajectory. For each step, generate a cognitive abstraction using an LLM prompt like: "Summarize what cognitive behavior (search strategy, reasoning pattern, error type) this step exhibits." Store entries as `(state, context_summary, abstraction, label)` in a vector index (FAISS or similar) keyed by the state embedding.

7. **Implement the Slow Experience-Driven Monitor**: When an anomaly is detected, encode the current agent state, retrieve the top-2 most similar entries from M+ and top-2 from M-. Feed the current state plus these four examples to a Critical Model with a prompt: "Given this search state and these examples of past successes and failures, identify whether a cognitive error is occurring and suggest a specific corrective action." The output is `(error_flag, correction_suggestion)`.

8. **Apply corrective interventions**: If the Critical Model identifies an error, append the correction suggestion to the agent's context before generating the next action. Typical corrections include: reformulate the query with different terms, broaden/narrow the search scope, resolve a specific contradiction between sources, or abandon a dead-end line of investigation.

9. **Implement online memory consolidation (optional)**: After each completed trajectory, generate a cognitive abstraction for the new experience and add it to M+ or M- based on outcome. Deduplicate against existing entries using an embedding similarity threshold (tau_dup = 0.95).

10. **Monitor and tune**: Track anomaly trigger rate (should be 10-25% of steps), false positive rate, and correction acceptance rate. Adjust k (anomaly sensitivity) and calibration parameters based on observed performance.

## Concrete Examples

**Example 1: Building a self-correcting research agent**

User: "Build me a research agent that can search the web for complex questions and detect when it's going down the wrong path."

Approach:
1. Implement a ReAct loop with web search tool access (e.g., using Tavily, Serper, or Brave Search API)
2. Add SE computation: after each search, embed the top-5 results using a sentence transformer, cluster them, compute entropy
3. Add RE computation: log token probabilities during the reasoning step, compute average entropy
4. Pre-fit calibration from 100 training questions with known answers
5. On anomaly detection, trigger a secondary LLM call that reviews the current trajectory and suggests query reformulation

Output structure:
```python
class DSMCMAgent:
    def __init__(self, llm, search_tool, embedder, calibration_params):
        self.llm = llm
        self.search_tool = search_tool
        self.embedder = embedder
        self.a, self.b, self.sigma = calibration_params  # from fitted calibration
        self.memory_positive = FAISSIndex()
        self.memory_negative = FAISSIndex()

    def fast_monitor(self, retrieved_docs, reasoning_logprobs):
        se = self.compute_searching_entropy(retrieved_docs)
        re = self.compute_reasoning_entropy(reasoning_logprobs)
        re_hat = self.a * se + self.b
        epsilon = abs(re - re_hat)
        return epsilon > 2 * self.sigma, {"se": se, "re": re, "epsilon": epsilon}

    def slow_monitor(self, current_state):
        embedding = self.embedder.encode(current_state)
        pos_examples = self.memory_positive.search(embedding, k=2)
        neg_examples = self.memory_negative.search(embedding, k=2)
        correction = self.llm.generate(
            f"Current state: {current_state}\n"
            f"Similar successes: {pos_examples}\n"
            f"Similar failures: {neg_examples}\n"
            f"Identify any cognitive error and suggest a correction."
        )
        return correction

    def step(self, query, history):
        docs = self.search_tool.search(query)
        reasoning, logprobs = self.llm.reason(query, docs, history, return_logprobs=True)
        is_anomaly, metrics = self.fast_monitor(docs, logprobs)
        correction = None
        if is_anomaly:
            correction = self.slow_monitor(self.format_state(query, docs, reasoning))
        action = self.llm.decide_action(query, docs, reasoning, correction=correction)
        return action, metrics
```

**Example 2: Adding anomaly detection to an existing RAG pipeline**

User: "My RAG agent sometimes gives confident but wrong answers when sources conflict. Add monitoring."

Approach:
1. Identify the retrieval and generation stages in the existing pipeline
2. Insert SE computation between retrieval and generation — cluster the retrieved chunks and compute entropy
3. Approximate RE using a sampling-based approach: generate the answer 3 times with temperature=0.7 and measure agreement (semantic entropy as a proxy)
4. Flag cases where SE is high (contradictory sources) but RE-proxy is low (model generates the same answer confidently every time)
5. When flagged, prepend a metacognitive prompt: "The retrieved sources contain contradictions. Before answering, explicitly identify the conflicting claims and assess which sources are more credible."

Output:
```python
def monitor_rag_response(retrieved_chunks, query, llm, embedder):
    # Searching Entropy
    embeddings = embedder.encode([c.text for c in retrieved_chunks])
    labels = kmeans(embeddings, n_clusters=min(3, len(chunks))).labels_
    cluster_probs = np.bincount(labels) / len(labels)
    se = -np.sum(cluster_probs * np.log(cluster_probs + 1e-10))

    # Reasoning Entropy (sampling proxy)
    samples = [llm.generate(query, retrieved_chunks, temperature=0.7) for _ in range(3)]
    sample_embeddings = embedder.encode(samples)
    pairwise_sim = cosine_similarity(sample_embeddings)
    re_proxy = 1.0 - pairwise_sim.mean()  # high agreement = low entropy

    # Anomaly: high evidence uncertainty but low reasoning uncertainty
    if se > SE_THRESHOLD and re_proxy < RE_THRESHOLD:
        return {"anomaly": True, "type": "overconfident_despite_contradiction",
                "correction": "Sources conflict. Identify contradictions explicitly."}
    return {"anomaly": False}
```

**Example 3: Preventing query loops in a multi-step search agent**

User: "My agent keeps searching for the same thing with slightly different queries and never converges."

Approach:
1. Track query history and compute pairwise similarity between consecutive queries
2. Monitor SE across steps — if SE stays flat (same type of results each time) while the agent keeps searching, the Fast Monitor flags low progress
3. When flagged, the Slow Monitor retrieves past examples of loop-breaking strategies from M+ (e.g., switching to a completely different search angle, using a different source type, or synthesizing partial answers)
4. Inject the corrective suggestion: "You have searched 4 times with similar queries yielding similar results. Based on past successful strategies, try: [specific alternative approach from memory]."

Output:
```
Step 5 anomaly detected:
  SE trend: [2.1, 2.0, 2.1, 1.9, 2.0] (flat - no information gain)
  Query similarity to previous: 0.92 (near-duplicate)
  Correction from experience memory:
    "Past success pattern: When search stalls, decompose the question into
     sub-questions and search for each independently. Similar failure avoided
     by switching from 'who invented X' to 'history of X' + 'patents filed for X'."
  -> Agent reformulates into two targeted sub-queries
```

## Best Practices

- **Do**: Keep the Fast Monitor truly lightweight — SE and RE computation should add less than 5% overhead per step. Use pre-computed embeddings and simple clustering.
- **Do**: Build memory banks from real trajectories with ground-truth outcome labels. Even 100-200 trajectories provide useful experience patterns for the Slow Monitor.
- **Do**: Log all anomaly detections and corrections for offline analysis. This data improves calibration parameters and memory quality over time.
- **Do**: Use the dual memory design (M+ and M-) — showing the model both what success looks like and what failure looks like produces much better corrections than either alone.
- **Avoid**: Running the Slow Monitor at every step. The whole point of the hierarchy is that expensive reflection is triggered only on detected anomalies (~10-25% of steps). Running it everywhere defeats the efficiency gain.
- **Avoid**: Setting the anomaly threshold too aggressively (k < 1.5). This causes excessive false positives and intervention fatigue. Start with k=2 and tune downward only if real failures are being missed.
- **Avoid**: Using the correction suggestion as a hard override. Append it as advisory context that the agent can weigh against its own reasoning — this preserves agent autonomy while providing guidance.

## Error Handling

- **Calibration data insufficient**: If you have fewer than 50 successful trajectories, skip the fitted calibration and use a heuristic: flag any step where SE > 2.0 and RE < 0.5 (or vice versa). Replace proper calibration once enough data accumulates.
- **Embedding model unavailable**: Approximate SE using lexical diversity metrics (unique n-grams / total n-grams across retrieved documents). Less precise but captures the core signal of evidence fragmentation.
- **Token logprobs not available**: Many API-based LLMs don't expose logprobs. Use the sampling-proxy approach from Example 2 (generate N responses, measure agreement via embedding similarity). Three samples with temperature 0.7 is a reasonable minimum.
- **Memory banks empty at deployment start**: Run the agent without the Slow Monitor initially, collecting trajectories. After accumulating 50+ labeled trajectories, build the memory banks and enable the full two-tier system. The Fast Monitor alone still catches obvious anomalies.
- **False positive storms**: If the anomaly rate exceeds 40%, the calibration is likely misconfigured. Re-fit using only clearly successful trajectories and increase k to 2.5 temporarily.

## Limitations

- **Requires trajectory-level ground truth**: Building the experience memory banks needs labeled trajectories (success/failure). For novel domains with no historical data, only the Fast Monitor is available initially.
- **Linear calibration assumption**: The SE-RE relationship is modeled as linear, which may not hold for all domains. Highly technical or adversarial queries may exhibit non-linear uncertainty patterns. Monitor calibration residuals and consider polynomial fits if needed.
- **LLM-dependent entropy quality**: Reasoning Entropy is only meaningful if the LLM's token probabilities are well-calibrated. Heavily RLHF-tuned models may have artificially peaked distributions, reducing RE signal quality.
- **Not a replacement for better retrieval**: DS-MCM monitors and corrects, but cannot compensate for fundamentally poor retrieval (e.g., wrong corpus, broken search API). Fix retrieval quality first, then add monitoring.
- **Cold-start problem**: The Slow Experience-Driven Monitor provides its highest value after accumulating diverse trajectory examples. Early deployment relies primarily on the Fast Monitor.

## Reference

Sun, Z., Wang, Q., Yu, W., Yang, J., & Lu, H. (2026). *Deep Search with Hierarchical Meta-Cognitive Monitoring Inspired by Cognitive Neuroscience*. arXiv:2601.23188v1. https://arxiv.org/abs/2601.23188v1

Key sections to study: the Searching Entropy / Reasoning Entropy calibration framework (Section 3.2), the dual experience memory architecture (Section 3.3), and the ablation study showing that the two-tier hierarchy outperforms always-on reflection while using 3-7% overhead vs 12-22% (Section 4.3).