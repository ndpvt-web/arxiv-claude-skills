---
name: "temp-r1-unified-autonomous-agent"
description: "Build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets using structured action decomposition and reverse curriculum reasoning. Use when: 'answer questions about events over time', 'temporal knowledge graph QA', 'build a temporal reasoning agent', 'multi-hop time-constrained question answering', 'reverse curriculum RL for reasoning', 'search-filter-rank pipeline for temporal data'."
---

# Temp-R1: Autonomous Temporal Reasoning via Structured Actions and Reverse Curriculum Learning

This skill enables Claude to build autonomous agents that answer complex temporal questions over knowledge graphs or time-stamped datasets. The core technique from Temp-R1 decomposes monolithic "thinking" into explicit action types -- Plan, Search, Filter, Rank, Answer -- preventing cognitive overload and temporal hallucinations. Combined with reverse curriculum learning (training on hard multi-hop queries before easy ones), this architecture handles temporal constraints like "before," "after," "first," "last," duration calculations, and interval overlaps far more reliably than single-pass reasoning.

## When to Use

- When building a QA system over temporal data (event logs, historical databases, versioned knowledge graphs, timelines)
- When the user needs multi-hop reasoning across time-stamped facts (e.g., "Who was president before the person who signed treaty X?")
- When implementing a search-and-reason agent loop over structured data with temporal constraints
- When the user wants to decompose complex temporal queries into plan-search-filter-rank pipelines
- When designing curriculum strategies for training reasoning agents, especially to avoid shortcut collapse on easy examples
- When integrating semantic retrieval (e.g., embedding-based search) with explicit temporal filtering and ranking

## Key Technique

**Structured Action Decomposition.** Instead of a single monolithic "think" step, Temp-R1 splits agent cognition into discrete action types with distinct roles. *Internal actions* (`plan`, `filter`, `rank`) operate on already-retrieved information without querying external sources. *External actions* (`search`) retrieve new facts from the knowledge graph. A terminal `answer` action ends the episode. This separation eliminates a class of temporal hallucinations where models confuse chronological ordering or fabricate intermediate facts, because each cognitive function is isolated and auditable.

**Reverse Curriculum Learning.** Standard curriculum learning starts easy and progresses to hard. Temp-R1 inverts this: it trains exclusively on complex multi-hop temporal queries first, then mixes in simple single-hop queries. This prevents *shortcut learning* -- without it, agents learn a minimal `search -> answer` pattern that handles easy questions but collapses on hard ones (performance drops from 0.550 to 0.143 on complex queries in ablations). By forcing the agent to develop sophisticated plan-search-filter-rank trajectories before encountering shortcuts, the resulting policy generalizes across all difficulty levels.

**Adaptive Trajectory Planning.** The trained agent dynamically adjusts its action sequence based on query complexity. Simple queries trigger ~1.4 internal actions and ~1.3 searches; complex queries trigger ~2.9 internal actions and ~1.9 searches. This emergent behavior means the agent doesn't over-reason on easy queries or under-reason on hard ones.

## Step-by-Step Workflow

1. **Parse the temporal question into components.** Identify the question type (simple lookup, before/after, first/last, duration, interval overlap), extract explicit time constraints (dates, ranges), and identify entities and relations mentioned.

2. **Execute the Plan action.** Generate a structured decomposition: what sub-questions need answering, what temporal constraints apply, what order of operations is needed, and what the expected answer format is. Write this as an explicit plan object, not free-form thought.

3. **Execute Search actions against the temporal data source.** Use semantic similarity (e.g., embedding-based retrieval) to find relevant time-stamped facts (quadruples: subject, predicate, object, timestamp). Each search query should target a specific sub-question from the plan. Limit to a maximum number of search turns (e.g., 5) to prevent unbounded retrieval.

4. **Execute Filter actions on retrieved results.** Apply the temporal constraints from step 1 to the retrieved facts. Filter by date ranges, before/after relations, entity matches, and semantic relevance of predicates. Discard facts that don't satisfy the constraints. This is an internal action -- no new data is fetched.

5. **Execute Rank actions to order filtered facts.** Sort remaining facts chronologically by timestamp. For "first/last" questions, identify extremal entries. For duration questions, compute time deltas between relevant events. For interval questions, detect overlaps. This explicit ordering prevents the model from hallucinating temporal sequences.

6. **Evaluate whether the accumulated evidence is sufficient.** If the plan's sub-questions are all resolved, proceed to answer. If gaps remain, loop back to step 3 with a refined search query targeting the missing information.

7. **Execute the Answer action.** Produce the final answer based on the filtered and ranked evidence chain. Include the reasoning trace (which facts were retrieved, how they were filtered, how they were ordered) for auditability.

8. **When designing training or evaluation, apply reverse curriculum ordering.** Start evaluation or fine-tuning with the hardest multi-hop temporal questions. Only after the system handles those reliably should you mix in simpler single-hop queries. Monitor for shortcut collapse: if simple-query accuracy rises while complex-query accuracy drops, the curriculum is too easy too early.

## Concrete Examples

**Example 1: Multi-hop temporal query over event data**

```
User: "Given this dataset of international diplomatic events, who was the head
of state of France before the person who signed the 2015 Paris Agreement?"

Approach:
1. PLAN: Decompose into sub-questions:
   a) Who signed the 2015 Paris Agreement on behalf of France?
   b) When did that person take office?
   c) Who was head of state of France immediately before that date?

2. SEARCH(sub-question a): Query for (France, signed, Paris Agreement, 2015)
   -> Retrieved: (Francois Hollande, signed, Paris Agreement, 2015-12-12)

3. SEARCH(sub-question b): Query for (Francois Hollande, took_office, ?, ?)
   -> Retrieved: (Francois Hollande, took_office, President, 2012-05-15)

4. FILTER: Apply "before 2012-05-15" constraint to head-of-state records
   -> Retrieved: (Nicolas Sarkozy, held_office, President, 2007-05-16 to 2012-05-15)

5. RANK: Sort by end_date descending, take first entry
   -> Nicolas Sarkozy (office ended 2012-05-15, immediately before Hollande)

6. ANSWER: Nicolas Sarkozy

Output:
{
  "answer": "Nicolas Sarkozy",
  "reasoning_trace": [
    {"action": "plan", "output": "3 sub-questions identified"},
    {"action": "search", "query": "France signed Paris Agreement 2015", "results": 1},
    {"action": "search", "query": "Hollande took office", "results": 1},
    {"action": "filter", "constraint": "before 2012-05-15", "kept": 1},
    {"action": "rank", "order": "end_date desc", "top": "Nicolas Sarkozy"},
    {"action": "answer", "value": "Nicolas Sarkozy"}
  ]
}
```

**Example 2: Duration reasoning over a timeline dataset**

```
User: "How long did the trade embargo between countries A and B last,
given this CSV of international relations events?"

Approach:
1. PLAN: Need start and end dates of the embargo to compute duration.
   Sub-questions: (a) When did the embargo start? (b) When did it end?

2. SEARCH("A embargo B start"):
   -> (A, imposed_embargo, B, 2018-03-15)

3. SEARCH("A embargo B end"):
   -> (A, lifted_embargo, B, 2021-09-01)

4. FILTER: Confirm both events refer to the same embargo instance
   (check no intermediate lift-and-reimpose cycles exist)
   -> SEARCH("A lifted_embargo B between 2018-03-15 and 2021-09-01")
   -> No intermediate events found; single continuous embargo confirmed.

5. RANK: Order [start, end] chronologically -> [2018-03-15, 2021-09-01]

6. ANSWER: Compute delta = 2021-09-01 minus 2018-03-15 = 3 years, 5 months, 17 days

Output:
{
  "answer": "3 years, 5 months, and 17 days",
  "start": "2018-03-15",
  "end": "2021-09-01",
  "reasoning_trace": [...]
}
```

**Example 3: Building the search-filter-rank pipeline in code**

```
User: "Help me implement a temporal QA agent over my event database."

Approach:
1. Define the action schema as an enum or tagged union:

   class Action(Enum):
       PLAN = "plan"
       SEARCH = "search"
       FILTER = "filter"
       RANK = "rank"
       ANSWER = "answer"

2. Implement the retrieval backend (SEARCH action):

   def search(query: str, kg: TemporalKG, top_k: int = 10) -> list[Quad]:
       embeddings = encode(query)
       return kg.similarity_search(embeddings, top_k=top_k)

3. Implement the temporal filter (FILTER action):

   def filter_temporal(facts: list[Quad], constraints: TemporalConstraint) -> list[Quad]:
       return [f for f in facts
               if constraints.satisfied_by(f.timestamp)]

4. Implement chronological ranking (RANK action):

   def rank_chronological(facts: list[Quad], order: str = "asc") -> list[Quad]:
       return sorted(facts, key=lambda f: f.timestamp,
                      reverse=(order == "desc"))

5. Wire the agent loop with state tracking:

   def run_agent(question: str, kg: TemporalKG, max_turns: int = 5):
       plan = generate_plan(question)
       history = []
       for turn in range(max_turns):
           action = select_action(question, history, plan)
           if action.type == Action.SEARCH:
               obs = search(action.query, kg)
           elif action.type == Action.FILTER:
               obs = filter_temporal(history[-1].results, action.constraints)
           elif action.type == Action.RANK:
               obs = rank_chronological(history[-1].results, action.order)
           elif action.type == Action.ANSWER:
               return action.value, history
           history.append(StepRecord(action, obs))
       return None, history  # max turns exceeded
```

## Best Practices

- **Do:** Separate internal reasoning (plan/filter/rank) from external retrieval (search) as distinct, typed actions. This makes each step auditable and prevents the agent from hallucinating facts it never retrieved.
- **Do:** Cap the maximum number of search turns (5-7 is typical). Unbounded retrieval loops waste resources and rarely improve answer quality.
- **Do:** When training or evaluating, start with the hardest temporal questions first. Monitor complex-query accuracy as a leading indicator -- if it drops while easy accuracy rises, shortcut collapse is occurring.
- **Do:** Store the full (subject, predicate, object, timestamp) quadruple, not just entity names. Temporal reasoning requires the timestamp to be a first-class citizen in every fact.
- **Avoid:** Monolithic "think" steps that combine planning, filtering, and ranking into one blob of text. This causes cognitive overload and temporal ordering errors, especially on multi-hop queries.
- **Avoid:** Training on easy questions first or mixing difficulties uniformly from the start. Both lead to shortcut learning where the agent converges on `search -> answer` patterns insufficient for complex reasoning.

## Error Handling

- **No results from search:** If a search returns zero relevant facts, refine the query by relaxing temporal constraints or using alternate entity names. Log the empty search for debugging. After 2 consecutive empty searches on the same sub-question, surface a "cannot determine" answer rather than hallucinating.
- **Ambiguous temporal constraints:** If the question contains implicit time references ("recently," "around that time"), resolve them relative to the most recent explicitly dated fact in the conversation history. Flag the ambiguity in the reasoning trace.
- **Contradictory facts retrieved:** When filter produces facts with conflicting timestamps for the same event, rank by source reliability or recency. Surface the contradiction to the user rather than silently choosing one.
- **Max turns exceeded:** If the agent hits the search turn limit without resolving all sub-questions, return a partial answer with the unresolved sub-questions explicitly listed. Do not fabricate missing information.
- **Shortcut collapse during training/evaluation:** If complex-query accuracy degrades while simple-query accuracy improves, revert to hard-only training for additional iterations before reintroducing easy examples.

## Limitations

- This approach requires structured temporal data (knowledge graphs or time-stamped records). It does not work well on unstructured text without a prior extraction step to produce (entity, relation, entity, timestamp) quadruples.
- The reverse curriculum strategy assumes questions can be classified by difficulty (single-hop vs. multi-hop). For domains where difficulty is continuous or hard to assess, the curriculum boundary is unclear.
- Semantic search quality is a bottleneck. If the retriever misses relevant facts, no amount of filtering or ranking downstream can recover. Invest in retrieval quality first.
- The plan-search-filter-rank loop adds latency compared to single-pass QA. For latency-sensitive applications with only simple temporal queries, a direct retrieval approach may be more appropriate.
- Duration and interval arithmetic require clean timestamp formats. Messy or inconsistent date representations (mixed granularities, missing day/month) need normalization before this pipeline can operate reliably.

## Reference

**Paper:** [Temp-R1: A Unified Autonomous Agent for Complex Temporal KGQA via Reverse Curriculum Reinforcement Learning](https://arxiv.org/abs/2601.18296v1) -- Focus on Section 3 (action space decomposition into plan/search/filter/rank/answer), Section 4 (reverse curriculum training protocol), and Table 2 (ablation showing shortcut collapse without reverse curriculum).