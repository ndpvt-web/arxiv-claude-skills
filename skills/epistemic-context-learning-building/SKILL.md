---
name: "epistemic-context-learning-building"
description: "Build trust-aware multi-agent systems using Epistemic Context Learning (ECL). Constructs peer reliability profiles from interaction history so agents weight information by source credibility instead of blindly conforming. Use when: 'build a multi-agent pipeline with trust', 'add peer reliability tracking to my agent system', 'prevent sycophancy in my LLM agents', 'implement history-aware trust for multi-agent', 'make agents evaluate peer credibility', 'add epistemic context to agent orchestration'."
---

# Epistemic Context Learning: Trust-Aware Multi-Agent Systems

This skill teaches you to implement Epistemic Context Learning (ECL), a two-stage reasoning framework where LLM-based agents build explicit trust profiles from peer interaction history before incorporating peer responses into their decisions. Instead of treating all peer input equally (which causes sycophantic conformity), ECL forces agents to first compress historical evidence into a reliability belief, then use that belief to weight current-round peer suggestions. This makes small models outperform much larger ones by accurately identifying which peers to trust.

## When to Use

- When building a multi-agent system where agents consult each other and you need to prevent blind agreement with incorrect peers
- When implementing an LLM orchestration pipeline that aggregates outputs from multiple models and needs source-quality weighting
- When the user asks to "add trust" or "track reliability" across agents in a collaborative system
- When designing adversarial-robust agent architectures where some peers may inject subtly wrong answers
- When creating a code review pipeline with multiple LLM reviewers of varying quality and you need to weight their feedback
- When building RAG systems that pull from multiple retrieval sources of uneven reliability
- When the user wants to reduce sycophancy in agent-to-agent communication

## Key Technique

**The core insight**: In multi-agent LLM systems, agents fail not because they lack reasoning ability, but because they cannot distinguish trustworthy peers from unreliable ones. Standard approaches either treat all peer opinions equally (majority vote) or evaluate individual reasoning quality per-response. ECL shifts the problem: instead of judging *what* a peer says right now, judge *how reliable the peer has been historically*, then condition your decision on that reliability estimate.

**The two-stage architecture**: ECL splits each decision into Stage 1 (Trust Estimation) and Stage 2 (Trust-Informed Aggregation). In Stage 1, the agent receives *only* the historical interaction record for each peer -- a sequence of past questions and that peer's past answers -- with no access to the current question or current peer responses. This information bottleneck forces the agent to compress history into a structured belief profile (e.g., "Peer A was correct on 4/5 past rounds; Peer B on 1/5"). In Stage 2, the agent receives the compressed belief profiles alongside the current question and current peer responses, then generates its final answer by weighting peer input according to estimated reliability.

**Why the separation matters**: If you give the model history and current responses simultaneously, it takes shortcuts -- matching surface patterns between historical and current answers instead of genuinely modeling trust. The two-stage split, acting as an information bottleneck, forces real reliability estimation. The paper further shows this can be optimized with reinforcement learning using a Peer Recognition Reward (did Stage 1 correctly identify the most reliable peer?) alongside the standard outcome reward (did Stage 2 get the right answer?).

## Step-by-Step Workflow

1. **Define your peer set and interaction schema.** Identify the N agents (models, tools, or retrieval sources) that will serve as peers. For each peer, define what a "historical interaction" looks like: a tuple of `(query, peer_response, ground_truth_or_outcome)`. Store these in a structured format per peer.

2. **Collect interaction history.** Run each peer through a calibration set of T questions (the paper uses T=5 rounds with 4 peers). Record each peer's response and whether it was correct. Structure this as: `history[peer_id] = [(q1, response1, correct1), (q2, response2, correct2), ...]`.

3. **Implement Stage 1: Trust Estimation (history-only).** Build a prompt that presents ONLY the historical interactions for each peer. Explicitly withhold the current question and current peer responses. Instruct the model to analyze past behavior and output a structured belief profile. The output should name each peer and summarize their reliability (accuracy rate, domain strengths, failure patterns), concluding with an explicit ranking: "The most reliable peer is: [PEER_NAME]".

4. **Implement the information bottleneck.** Pass only the compressed belief profile string from Stage 1 into Stage 2. Do NOT pass raw history into Stage 2. This forces the model to rely on its own trust summary rather than re-processing raw data or pattern-matching.

5. **Implement Stage 2: Trust-Informed Aggregation.** Build a prompt that presents: (a) the belief profiles from Stage 1, (b) the current question, and (c) the current peer responses labeled by peer name. Instruct the model to weight peer suggestions according to their reliability profiles and produce a final answer.

6. **Add structured output parsing.** Parse Stage 1 output to extract per-peer reliability scores and the top-peer designation. Parse Stage 2 output to extract the final answer. Use these for logging, evaluation, and the reward signals described next.

7. **Implement dual reward signals for optimization (optional but powerful).** If fine-tuning or using RL: assign a Peer Recognition Reward (PRR) of +1.0 to Stage 1 when it correctly identifies the most reliable peer, and an Outcome Reward (OR) of +1.0 to Stage 2 when the final answer is correct. Training with both rewards provides denser feedback than outcome-only.

8. **Handle dynamic peer sets.** When peers change between rounds (new agents join, old ones leave), maintain per-peer history independently. For new peers with no history, instruct Stage 1 to assign a neutral prior ("insufficient history to assess reliability") and have Stage 2 treat their input with moderate skepticism.

9. **Validate with adversarial testing.** Test with a configuration where one peer is reliably correct and others inject plausible-but-wrong answers. Verify that Stage 1 correctly identifies the reliable peer and Stage 2 follows that peer's guidance over the majority.

10. **Log and iterate on trust calibration.** Track Stage 1's peer identification accuracy and Stage 2's final answer accuracy separately. If Stage 1 accuracy is low, increase the history window (more rounds) or improve the Stage 1 prompt. If Stage 1 is accurate but Stage 2 still fails, the aggregation prompt needs refinement.

## Concrete Examples

**Example 1: Multi-Model Code Review Pipeline**

```
User: I have three LLM reviewers (GPT-4o, Claude, Gemini) checking PRs for bugs.
Sometimes the weaker model's wrong suggestion overrides the correct one.
Add trust-based weighting so the system learns which reviewer to trust.

Approach:
1. Collect calibration data: run all three reviewers on 20 known-buggy code
   snippets where ground truth is established. Record each reviewer's
   verdicts (bug found / missed / false positive).

2. Build per-reviewer history records:
   history = {
     "gpt4o": [
       {"snippet": "off-by-one in loop", "response": "flagged correctly", "correct": true},
       {"snippet": "null deref", "response": "missed", "correct": false},
       ...
     ],
     "claude": [...],
     "gemini": [...]
   }

3. Stage 1 prompt (history only, no current PR):
   """
   Analyze the historical review accuracy of each peer reviewer below.
   For each reviewer, summarize their accuracy rate and typical failure modes.
   Conclude with: "The most reliable reviewer is: <NAME>"

   ## GPT-4o History
   - Round 1: [snippet summary] -> [response] -> [correct/incorrect]
   - Round 2: ...

   ## Claude History
   ...

   ## Gemini History
   ...
   """

   Stage 1 output:
   "GPT-4o: 16/20 correct (80%), tends to miss null-safety issues.
    Claude: 18/20 correct (90%), strong on logic bugs, occasionally verbose.
    Gemini: 12/20 correct (60%), high false-positive rate on style issues.
    The most reliable reviewer is: Claude"

4. Stage 2 prompt:
   """
   You are reviewing a pull request. Use these peer reliability profiles
   to weight reviewer feedback appropriately.

   ## Reliability Profiles
   [Stage 1 output inserted here]

   ## Current PR Diff
   [diff content]

   ## Reviewer Feedback
   GPT-4o says: "No bugs found."
   Claude says: "Line 42 has a race condition in the mutex acquisition."
   Gemini says: "Variable naming could be improved on line 15."

   Provide your final review, weighting feedback by reviewer reliability.
   """

Output: Final review flags the race condition (trusting Claude's high-reliability
assessment) while deprioritizing Gemini's style comment and noting GPT-4o's
miss pattern on concurrency issues.
```

**Example 2: Multi-Source RAG with Source Reliability**

```
User: My RAG pipeline pulls from internal docs, Stack Overflow, and a legacy
wiki. The legacy wiki often has outdated info that poisons answers.
Help me add source trust tracking.

Approach:
1. Treat each retrieval source as a "peer agent." Collect history by
   sampling 10 past queries where you know the correct answer, recording
   which source provided correct vs. outdated/wrong passages.

2. Build source history:
   history = {
     "internal_docs": [{"query": "auth flow", "relevant": true, "accurate": true}, ...],
     "stackoverflow": [{"query": "auth flow", "relevant": true, "accurate": true}, ...],
     "legacy_wiki": [{"query": "auth flow", "relevant": true, "accurate": false}, ...]
   }

3. Stage 1 (run once, cache the profile, refresh periodically):
   Prompt with history only. Output:
   "internal_docs: 9/10 accurate, authoritative for current architecture.
    stackoverflow: 7/10 accurate, good for general patterns, sometimes outdated versions.
    legacy_wiki: 3/10 accurate, frequently references deprecated APIs.
    Most reliable source: internal_docs"

4. Stage 2 (per query):
   Present source profiles + current retrieved passages + user question.
   The model prioritizes internal_docs passages, cross-checks stackoverflow,
   and treats legacy_wiki content with high skepticism -- citing it only
   when corroborated by a reliable source.

Output: Answers grounded in internal docs, with legacy wiki info either
excluded or explicitly flagged as "from a lower-reliability source, verify
against current docs."
```

**Example 3: Adversarial-Robust Agent Debate**

```
User: I'm running a 4-agent debate system for complex reasoning tasks.
Three agents sometimes converge on a wrong answer and outvote the correct one.
Implement ECL so the system trusts track records over majority.

Approach:
1. Run all 4 agents through 5 calibration rounds on known questions.
   Record per-agent correctness.

2. Stage 1 identifies that Agent-2 has 5/5 accuracy while Agents 1, 3, 4
   average 2/5. Profile: "Agent-2 is the most reliable peer."

3. Stage 2 receives the current debate. Even though Agents 1, 3, 4 agree
   on answer "B", the system weights Agent-2's answer "A" more heavily
   based on the trust profile, and selects "A".

Result: The system breaks free of majority-rules failure mode by
conditioning on historical reliability rather than current vote counts.
```

## Best Practices

- **Do** separate trust estimation from answer generation into two distinct LLM calls with an information bottleneck between them. Combining them into one prompt defeats the purpose -- the model will shortcut by pattern-matching instead of genuinely modeling trust.
- **Do** use at least 5 historical interaction rounds per peer. Fewer rounds produce unreliable trust estimates; more rounds improve accuracy but increase prompt length.
- **Do** refresh trust profiles periodically. Agent quality drifts over time (model updates, data staleness). Re-run Stage 1 on recent interactions on a schedule.
- **Do** include explicit failure-mode summaries in Stage 1 output, not just accuracy numbers. "Peer X struggles with concurrency bugs" is more actionable than "Peer X: 70% accurate."
- **Avoid** passing raw interaction history into Stage 2. The bottleneck is the mechanism that forces genuine trust modeling. Leaking history into Stage 2 enables shortcut learning.
- **Avoid** using ECL when all peers have equivalent, high reliability. The overhead of trust profiling adds latency without benefit when sources are uniformly trustworthy.
- **Avoid** treating the trust profile as static forever. A peer that was reliable last month may have degraded. Build in a decay or windowing mechanism.

## Error Handling

- **Insufficient history**: If a peer has fewer than 3 interaction records, Stage 1 should output "insufficient data" for that peer. Stage 2 should treat such peers as moderate-confidence sources rather than ignoring them entirely.
- **Stage 1 produces vague profiles**: If the trust estimation is generic ("all peers seem okay"), the history set may lack discriminative power. Add harder calibration questions where peers diverge in correctness.
- **Conflicting trust signals**: If Stage 1 identifies two peers as equally reliable but they disagree in Stage 2, instruct the model to flag the disagreement explicitly and present both perspectives rather than arbitrarily choosing one.
- **Prompt length overflow**: With many peers and long histories, the Stage 1 prompt can exceed context limits. Summarize older interactions or use a sliding window of the most recent T rounds.
- **Trust profile gaming**: In adversarial settings, a malicious peer could perform well during calibration then degrade. Mitigate by continuously updating profiles with recent production interactions, not just initial calibration data.

## Limitations

- ECL requires a calibration phase with known ground-truth answers to build initial trust profiles. It cannot cold-start on tasks where no ground truth is available.
- The two-stage architecture adds latency (two sequential LLM calls per decision). For latency-sensitive applications, consider caching Stage 1 profiles and only re-computing them periodically.
- Trust profiles are task-domain-specific. A peer reliable for code review may be unreliable for natural language translation. Maintain separate profiles per domain if your system spans multiple task types.
- The approach works best with 3-6 peers. With very large peer sets (>10), the Stage 1 prompt becomes unwieldy and trust estimation quality degrades.
- ECL does not address the case where *all* peers are unreliable. It identifies the *most* reliable peer relative to others, but if the best peer is still poor, the system will still underperform.

## Reference

- **Paper**: [Epistemic Context Learning: Building Trust the Right Way in LLM-Based Multi-Agent Systems](https://arxiv.org/abs/2601.21742v1) (Zhou et al., 2026)
- **Key takeaway**: Look for the two-stage information bottleneck design (Section 3) and the dual-reward RL optimization (Section 4). The adversarial evaluation in Section 5 demonstrates how ECL breaks majority-vote failure modes.