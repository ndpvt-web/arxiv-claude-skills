---
name: "tkg-thinker-dynamic-reasoning-over"
description: >
  Dynamic agentic reasoning over temporal knowledge graphs using a ReAct-style
  think-plan-retrieve loop with specialized temporal search tools and multi-dimensional
  reward shaping. Use this skill when the user asks to:
  "answer time-sensitive questions from a knowledge graph",
  "build a temporal QA agent with tool-augmented reasoning",
  "reason over events with before/after/between constraints",
  "query a knowledge base with temporal filtering",
  "implement agentic retrieval for time-dependent facts",
  "design a multi-step temporal reasoning pipeline"
---

# TKG-Thinker: Dynamic Reasoning over Temporal Knowledge Graphs

This skill enables Claude to implement **agentic, multi-turn reasoning pipelines** for answering questions that involve temporal constraints over structured knowledge graphs. Rather than issuing a single static prompt, the technique decomposes temporal questions into a plan, then executes a think-action-observation loop using purpose-built temporal search tools (e.g., `search_before`, `search_between`). The approach is drawn from the TKG-Thinker framework, which combines supervised fine-tuning on chain-of-thought trajectories with reinforcement learning shaped by format, retrieval, and outcome rewards. The result is a system that dynamically decides *what* to retrieve, *when* to stop, and *how* to compose temporal evidence into a final answer -- eliminating the hallucinations that plague one-shot prompting on time-sensitive queries.

## When to Use

- When the user needs to answer questions with explicit temporal constraints ("before X happened", "between 2019 and 2021", "the last time Y occurred") against a knowledge base or structured dataset.
- When building a retrieval-augmented generation (RAG) system where facts have timestamps and ordering matters (event logs, news archives, versioned records).
- When implementing a multi-step agent that must plan which temporal queries to issue, interpret partial results, and decide whether to refine or finalize its answer.
- When the user wants to reduce hallucinations in LLM-based QA by grounding every claim in a verified temporal retrieval step.
- When designing a reward function or evaluation harness for an agent that interacts with time-indexed data sources.
- When converting a static prompt-based QA pipeline into a dynamic, tool-augmented reasoning loop over temporal data.

## Key Technique

**Problem.** Temporal knowledge graphs (TKGs) store facts as quadruples `(subject, relation, object, timestamp)`. Questions like *"Which country negotiated with Yemen before 2015?"* require the system to parse temporal operators ("before"), retrieve matching facts, apply temporal filtering, and compose an answer. Static prompting fails here because the model must guess which facts are relevant in one shot, leading to hallucinated timestamps and missed constraints.

**Solution: Agentic Tool-Augmented Reasoning.** TKG-Thinker replaces the single-prompt paradigm with a ReAct-style loop. The agent first generates a **plan** decomposing the question into sub-goals (e.g., "find negotiation events involving Yemen", "filter to those before 2015", "identify the most recent"). It then iterates through **think-action-observation** turns: each *thought* reasons about what is known and what is missing; each *action* invokes one of five temporal search primitives; each *observation* returns structured facts from the TKG. The agent continues until it has sufficient evidence to produce a final answer or exhausts a turn budget. This design transforms vague temporal reasoning into concrete, verifiable retrieval steps.

**Optimization: Multi-Dimensional Reward Shaping.** To train or evaluate such an agent, TKG-Thinker introduces three reward signals: (1) a **format reward** ensuring the agent follows the think/act/observe protocol, (2) a **retrieval reward** checking that at least one retrieved fact contains the correct answer, and (3) an **outcome reward** for exact-match correctness. These can be combined into a composite score: `R = R_outcome * format_penalty + (1 - R_outcome) * (R_format + R_retrieval)`. Even without full RL training, this reward decomposition is directly useful for scoring agent trajectories, building evaluation harnesses, and debugging failure modes.

## Step-by-Step Workflow

1. **Parse temporal constraints from the question.** Identify temporal operators (before, after, between, during, first, last, most recent) and their associated timestamps or relative references. Extract the core relation query stripped of temporal modifiers.

2. **Define the temporal search tool interface.** Implement or expose five primitives that mirror TKG structure:
   - `search_time(query)` -- returns all timestamps where the relation holds.
   - `search_specific(query, t)` -- returns facts at exact timestamp `t`.
   - `search_before(query, t)` -- returns facts with timestamp strictly < `t`.
   - `search_after(query, t)` -- returns facts with timestamp strictly > `t`.
   - `search_between(query, t1, t2)` -- returns facts with `t1 <= timestamp <= t2`.

3. **Generate an explicit plan before any retrieval.** Decompose the question into ordered sub-goals. For example, for *"Who was the last president to visit France before 2020?"*: (a) retrieve presidential visits to France, (b) filter to those before 2020, (c) select the most recent among filtered results.

4. **Execute the think-action-observation loop.** For each sub-goal:
   - **Think:** State what you know so far and what you need next.
   - **Act:** Select the appropriate temporal search tool with precise parameters.
   - **Observe:** Record the returned facts, noting entity names and timestamps.

5. **Apply temporal ordering and aggregation.** After collecting raw facts, sort by timestamp and apply the question's aggregation operator (first, last, count, all). Verify that every returned fact satisfies the stated temporal constraint.

6. **Cross-validate retrieved evidence.** If the question involves multi-hop reasoning (e.g., "before event X, which itself happened after event Y"), verify the temporal chain: confirm Y's timestamp, then X's timestamp, then filter the target facts accordingly.

7. **Compose the final answer with provenance.** State the answer along with the supporting facts and their timestamps. This makes the reasoning auditable and prevents hallucinated temporal claims.

8. **Score the trajectory (for evaluation or training).** Compute the three reward dimensions:
   - Format: Did every turn follow think/act/observe structure? (binary)
   - Retrieval: Does any retrieved fact contain the gold answer? (binary)
   - Outcome: Does the final answer exactly match ground truth? (0 or 1)

9. **Handle failure modes.** If no facts are returned for a sub-goal, broaden the temporal window or relax the query. If contradictory facts appear, flag the conflict and prefer the fact with the most specific timestamp.

10. **Iterate or terminate.** Cap the loop at a fixed turn budget (8 turns is a good default). If the agent has not converged, return the best-supported answer with a confidence caveat.

## Concrete Examples

**Example 1: Simple temporal filtering over event data**

```
User: I have a dataset of diplomatic events as (country_a, action, country_b, date)
tuples. "Before the trade agreement between India and Japan in March 2022,
which country last signed a defense pact with Japan?"

Approach:
1. Parse constraints: "before March 2022", aggregation "last", relation "defense pact with Japan".
2. Plan:
   - Sub-goal A: Confirm the India-Japan trade agreement date (validate March 2022).
   - Sub-goal B: Retrieve all defense pacts involving Japan before March 2022.
   - Sub-goal C: Select the most recent (last) from results.
3. Execute:
   Think: "I need to verify the anchor event first."
   Act:  search_specific("India, trade_agreement, Japan", "2022-03")
   Obs:  [(India, trade_agreement, Japan, 2022-03-15)] -- confirmed.

   Think: "Now retrieve defense pacts with Japan before this date."
   Act:  search_before("*, defense_pact, Japan", "2022-03-15")
   Obs:  [(Australia, defense_pact, Japan, 2021-11-02),
          (UK, defense_pact, Japan, 2020-06-18),
          (Philippines, defense_pact, Japan, 2022-01-09)]

   Think: "The most recent before March 2022 is Philippines on 2022-01-09."
4. Answer: Philippines (defense pact signed 2022-01-09).

Output:
  Answer: Philippines
  Evidence: (Philippines, defense_pact, Japan, 2022-01-09)
  Reasoning: This is the latest defense pact with Japan strictly before the
  India-Japan trade agreement on 2022-03-15.
```

**Example 2: Multi-hop temporal reasoning with a code implementation**

```
User: Build me a Python agent that answers temporal questions over my event log
stored as a list of (subject, relation, object, timestamp) tuples.

Approach:
1. Define the five temporal search tools as Python functions over the tuple list.
2. Implement the think-act-observe loop as a function that takes a question,
   parses temporal operators, generates a plan, and iteratively calls tools.
3. Return the answer with supporting evidence.

Output (key code structure):

  from datetime import datetime
  from typing import List, Tuple, Optional

  Fact = Tuple[str, str, str, str]  # (subj, rel, obj, timestamp)

  class TemporalKGAgent:
      def __init__(self, facts: List[Fact], max_turns: int = 8):
          self.facts = [(s, r, o, datetime.fromisoformat(t))
                        for s, r, o, t in facts]
          self.max_turns = max_turns
          self.trajectory: list = []

      # -- Temporal search tools --
      def search_time(self, subj=None, rel=None, obj=None):
          """Return timestamps where the pattern matches."""
          return sorted(set(
              t for s, r, o, t in self.facts
              if (subj is None or s == subj)
              and (rel is None or r == rel)
              and (obj is None or o == obj)
          ))

      def search_before(self, ts, subj=None, rel=None, obj=None):
          return [(s,r,o,t) for s,r,o,t in self.facts
                  if t < ts
                  and (subj is None or s == subj)
                  and (rel is None or r == rel)
                  and (obj is None or o == obj)]

      def search_after(self, ts, subj=None, rel=None, obj=None):
          return [(s,r,o,t) for s,r,o,t in self.facts
                  if t > ts and (subj is None or s == subj)
                  and (rel is None or r == rel)
                  and (obj is None or o == obj)]

      def search_between(self, t1, t2, subj=None, rel=None, obj=None):
          return [(s,r,o,t) for s,r,o,t in self.facts
                  if t1 <= t <= t2
                  and (subj is None or s == subj)
                  and (rel is None or r == rel)
                  and (obj is None or o == obj)]

      def search_specific(self, ts, subj=None, rel=None, obj=None):
          return [(s,r,o,t) for s,r,o,t in self.facts
                  if t == ts and (subj is None or s == subj)
                  and (rel is None or r == rel)
                  and (obj is None or o == obj)]

  # The LLM orchestrator calls these tools in a loop, recording
  # (thought, action, observation) at each step, then composes
  # the final answer from accumulated evidence.
```

**Example 3: Designing a reward function for temporal QA evaluation**

```
User: I'm evaluating an LLM agent that answers temporal questions. How should
I score its reasoning traces?

Approach:
1. Parse each trace into (thought, action, observation) turns.
2. Compute three reward components per trace.
3. Combine into a single scalar for ranking or RL training.

Output (scoring logic):

  def score_trajectory(trace, gold_answer, gold_facts):
      # Format reward: every turn must have think + act + observe
      fmt_ok = all(
          "think" in turn and "act" in turn and "observe" in turn
          for turn in trace["turns"]
      )
      r_format = 1.0 if fmt_ok else 0.0

      # Retrieval reward: at least one observation contains the answer
      retrieved = [obs for turn in trace["turns"] for obs in turn["observe"]]
      r_retrieval = 1.0 if any(gold_answer in str(f) for f in retrieved) else 0.0

      # Outcome reward: final answer matches ground truth
      r_outcome = 1.0 if trace["final_answer"].strip() == gold_answer.strip() else 0.0

      # Composite (from TKG-Thinker paper):
      # Penalize format violations even when the answer is correct
      lam = 0.5  # format penalty weight
      delta = 0.1  # fallback incentive
      if r_outcome > 0:
          score = r_outcome * (1 - (1 - r_format) * lam)
      else:
          score = r_format + r_retrieval + delta * (1 - r_format)
      return {"format": r_format, "retrieval": r_retrieval,
              "outcome": r_outcome, "composite": score}
```

## Best Practices

- **Do:** Always generate an explicit plan *before* the first retrieval call. Ablations in the paper show removing the planning step drops accuracy by ~6%.
- **Do:** Use the most specific temporal search tool available. Prefer `search_before(query, t)` over `search_time(query)` followed by manual filtering -- it reduces noise in observations and keeps the context window lean.
- **Do:** Record the full think-act-observe trajectory. This makes debugging straightforward and enables reward-based evaluation even without RL training.
- **Do:** Validate anchor timestamps before using them in downstream queries. Multi-hop errors compound: a wrong anchor date poisons every subsequent temporal filter.
- **Avoid:** Stuffing all retrieved facts into a single prompt and asking the LLM to sort them. The whole point of the temporal tools is to offload filtering to deterministic code.
- **Avoid:** Open-ended retrieval without a turn budget. Cap at 8 turns to prevent loops. If the agent hasn't converged by then, return the best-supported partial answer.
- **Avoid:** Treating timestamps as strings for comparison. Always parse into proper datetime objects to handle edge cases (timezone offsets, varying date formats, granularity mismatches).

## Error Handling

| Failure Mode | Symptom | Recovery |
|---|---|---|
| Empty retrieval | Tool returns no facts | Broaden the temporal window (e.g., expand "before 2022" to "before 2023") or relax one query parameter (drop subject constraint). |
| Ambiguous timestamps | Question says "last year" without an absolute reference | Ask the user for clarification, or default to the most recent year in the dataset. |
| Conflicting facts | Two facts with same relation but different objects at overlapping timestamps | Prefer the fact with finer temporal granularity (day > month > year). Flag the conflict in the output. |
| Format violations in agent traces | Agent skips the think step or issues malformed tool calls | Reject the turn and re-prompt with an explicit format reminder. In evaluation, assign `r_format = 0`. |
| Turn budget exhaustion | Agent reaches max turns without a final answer | Return the entity most frequently appearing in retrieved facts, with a low-confidence flag. |
| Multi-hop chain break | Intermediate anchor event not found in KG | Report the missing link explicitly. Do not fabricate an intermediate timestamp. |

## Limitations

- **Requires structured temporal data.** The technique assumes facts are stored as `(s, r, o, t)` quadruples with parseable timestamps. Unstructured text corpora need a temporal extraction preprocessing step first.
- **Temporal granularity mismatches.** If the KG stores dates at month granularity but the question asks about a specific day, the search tools may over- or under-retrieve. Align granularity before querying.
- **No native support for fuzzy temporal expressions.** Phrases like "around the time of" or "shortly after" require heuristic conversion to concrete time windows, which introduces noise.
- **Scalability on very large KGs.** The linear-scan implementations shown above work for moderate datasets (up to ~1M facts). For larger graphs, back the search tools with indexed databases (e.g., timestamp-indexed SQL tables or time-series stores).
- **LLM-dependent planning quality.** The plan step relies on the LLM's ability to decompose temporal questions. Highly nested or ambiguous temporal logic (e.g., "before the event that happened after the third occurrence of X") may still produce flawed plans.

## Reference

- **Paper:** [TKG-Thinker: Towards Dynamic Reasoning over Temporal Knowledge Graphs via Agentic Reinforcement Learning](https://arxiv.org/abs/2602.05818v2) (Jiang et al., 2026)
- **Key takeaway:** Section 3 details the five temporal search primitives and the think-plan-act-observe loop; Section 4.2 specifies the multi-dimensional reward formula `R = R_out(1-(1-I_fmt)lambda) + (1-R_out)(R_fmt+R_ret) + (1-R_out)delta(1-I_fmt)` that balances format compliance, retrieval quality, and answer correctness.