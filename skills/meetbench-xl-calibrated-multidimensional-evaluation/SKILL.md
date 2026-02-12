---
name: "meetbench-xl-calibrated-multidimensional-evaluation"
description: |
  Build dual-policy meeting AI agents with fast/slow query routing and multi-dimensional response evaluation.
  Implements the MeetBench-XL framework: a lightweight classifier routes queries between a fast Talker path
  (sub-1s, direct extraction) and a slow Planner path (tool-augmented reasoning), while responses are scored
  across five calibrated dimensions. Use this skill when:
  - "Build a meeting assistant agent with fast and slow reasoning paths"
  - "Evaluate meeting Q&A responses across multiple quality dimensions"
  - "Route meeting queries by complexity to different processing pipelines"
  - "Design a dual-policy agent that balances latency and answer quality"
  - "Score AI responses for factual fidelity, conciseness, and completeness"
  - "Classify query complexity along cognitive load and context dependency axes"
---

# MeetBench-XL: Calibrated Multi-Dimensional Evaluation and Dual-Policy Meeting Agents

This skill enables Claude to design and implement dual-policy AI agents for real-time meeting assistance using the MeetBench-XL framework. The core technique is a learned routing policy that classifies incoming queries along four cognitive axes (cognitive load, context dependency, domain knowledge, task execution) and dispatches them to either a fast Talker path for simple extraction or a slow Planner path with tool invocation (retrieval, cross-meeting aggregation, web search). Responses are evaluated using a five-dimension scoring protocol calibrated to human judgment via isotonic regression (Pearson r=0.78).

## When to Use

- When building a meeting assistant that must handle both simple factual lookups and complex multi-hop reasoning under latency constraints
- When designing a query routing system that classifies questions by cognitive complexity and selects processing pipelines accordingly
- When implementing a multi-dimensional evaluation rubric for meeting Q&A that goes beyond single-score metrics
- When optimizing the quality-latency tradeoff in an agent that invokes retrieval, aggregation, or web search tools conditionally
- When scoring AI-generated meeting summaries or answers for factual accuracy, completeness, conciseness, structure, and intent alignment
- When building any dual-process (System 1 / System 2) agent architecture where a lightweight classifier gates expensive reasoning

## Key Technique

**Dual-Policy Routing.** MeetMaster-XL uses a 300K-parameter MLP classifier (2 layers, 512 hidden units, ~1ms inference) that consumes frozen encoder features from the query and routes to one of four actions: (1) Fast/Talker with no tools for low cognitive load queries, (2) Slow/Planner with in-meeting context only, (3) Slow+RAG for domain-expert queries needing knowledge base retrieval, (4) Slow+CrossSession for queries spanning multiple meetings. The classifier is trained with focal loss over 3 epochs on replay tuples, improving routing accuracy from 70.4% to 81.2%. A token-level sentinel gating mechanism lets the Planner emit a "T" token within 400ms to fall back to cached Talker responses when a query turns out simpler than initially classified (miss-trigger rate 1.8%).

**Four-Axis Complexity Taxonomy.** Every query is classified along four orthogonal axes: Cognitive Load (low: direct extraction under 5s, medium: 2-5 utterance synthesis, high: multi-hop reasoning over 45s), Context Dependency (none, recent 3-5 utterances, long-range 15+ minutes, cross-meeting), Domain Knowledge (general, basic terminology, expert technical/regulatory), and Task Execution (passive recall, structured organization, strategic planning). The 3x4x3x3=108 Cartesian cells are consolidated into 13 balanced classes matching real enterprise frequency distributions (38% low-CL/no-CD, 29% medium, 18% high, 15% cross-meeting).

**Five-Dimension Evaluation.** Responses are scored 1-10 on: Factual Accuracy (correctness of extractions), User Need Alignment (how well the response addresses the query intent), Conciseness (brevity without losing information), Structural Clarity (organization and readability), and Completeness (coverage of relevant content). The composite score uses equal weights combined via isotonic regression to calibrate against human ratings. The training objective jointly optimizes quality, latency, and token cost: J(pi) = E[alpha * TaskSuccess - beta * Latency - gamma * TokenCost] with alpha=1.0, beta=0.05, gamma=0.01, trained using Conservative Q-Learning over 10 epochs on 8,500 replay tuples.

## Step-by-Step Workflow

1. **Classify the query on four cognitive axes.** For each incoming query, assign a level on Cognitive Load (low/medium/high), Context Dependency (none/recent/long-range/cross-meeting), Domain Knowledge (general/basic/expert), and Task Execution (passive/structured/strategic). Use the query text and any available meeting context to determine these.

2. **Map the classification to a routing action.** Based on the four-axis classification, select one of: FAST (Talker, no tools), SLOW (Planner, in-meeting context), SLOW+RAG (Planner + knowledge base retrieval), or SLOW+CROSS (Planner + cross-meeting aggregation). Low-CL/no-CD queries go FAST; high-CL or cross-meeting queries go SLOW+CROSS; domain-expert queries go SLOW+RAG.

3. **Implement the fast path (Talker).** For FAST-routed queries, extract the answer directly from cached meeting transcript context. Target sub-1s response time. Use sliding-window attention over recent utterances (3-5) for context. Return a concise, single-point answer.

4. **Implement the slow path (Planner) with a 3-hop reasoning loop.** Step A: Analyze the query against the four axes to confirm complexity. Step B: Plan and invoke tools -- fetch up to 6 relevant snippets via retrieval, cross-meeting search, or web search as determined by the routing action. Step C: Compose the final answer synthesizing all evidence.

5. **Build the lightweight routing classifier.** Train a 2-layer MLP (512 hidden units) on frozen encoder features from query embeddings. Use focal loss to handle class imbalance across the 13 consolidated complexity classes. Fine-tune for 3 epochs on labeled query-action pairs.

6. **Implement sentinel token gating.** During Planner execution, if the model determines within 400ms that the query is simpler than initially classified, emit a sentinel token to fall back to the Talker's cached response. This prevents unnecessary latency on misrouted queries.

7. **Configure tool invocation policies.** Define when each tool fires: RAG retrieval when the query references domain terminology or requires factual grounding; cross-meeting aggregation when temporal markers reference prior sessions; web search when knowledge base confidence is low or the query references current events.

8. **Score responses on five dimensions.** Evaluate every response on Factual Accuracy, User Need Alignment, Conciseness, Structural Clarity, and Completeness (each 1-10). Use LLM-as-judge (e.g., CompassJudger or Prometheus) with calibrated rubrics, or use proxy metrics: ROUGE-L > 0.6 for factual queries, structured element detection via regex for task-execution queries, cache hit rate > 0.5 for cross-meeting queries.

9. **Compute the composite quality-latency objective.** Aggregate the five dimension scores equally, then apply the tradeoff formula: J = 1.0 * TaskSuccess - 0.05 * Latency_seconds - 0.01 * TokenCost. Use this to compare routing strategies and tune the classifier threshold.

10. **Validate with ablation.** Test each component independently: routing-only (no tools), tools-only (no routing), serial execution, and parallel execution. The parallel dual-policy configuration should yield the best quality-latency tradeoff (target: ~6.5+ composite score with sub-1s P50 latency for simple queries).

## Concrete Examples

**Example 1: Building a dual-policy meeting agent**

```
User: I want to build a meeting assistant that answers simple questions instantly
but takes more time for complex analytical queries. How should I architect this?

Approach:
1. Classify queries on the four cognitive axes (CL, CD, DK, TE)
2. Route using a lightweight MLP classifier:
   - Low CL + No CD -> FAST path (Talker): direct extraction, sub-1s
   - Medium CL + Recent CD -> SLOW path: in-meeting context reasoning
   - High CL or Expert DK -> SLOW+RAG: retrieval-augmented reasoning
   - Cross-meeting CD -> SLOW+CROSS: multi-session aggregation
3. Implement sentinel gating so the Planner can bail out to Talker within 400ms

Architecture:
  Query -> [Frozen Encoder] -> [2-layer MLP, 512 hidden, 300K params]
                                       |
                          +-------------+-------------+
                          |             |             |
                       FAST          SLOW        SLOW+RAG
                     (Talker)     (Planner)    (Planner+KB)
                    <1s latency   ~20s P50      ~25s P50
                                       |
                              Sentinel gate: emit "T" within 400ms
                              to fall back to Talker if query is simple

Output:
A routing classifier trained with focal loss achieving ~81% accuracy,
with overall composite score ~6.59 across five quality dimensions
and P50 latency of 1.1s for simple queries, 19.3s for complex ones.
```

**Example 2: Evaluating meeting Q&A response quality**

```
User: I need to evaluate how good my meeting bot's answers are across
multiple quality dimensions, not just a single accuracy score.

Approach:
1. Define five scoring dimensions (each 1-10):
   - Factual Accuracy: Are extracted facts correct against transcript?
   - User Need Alignment: Does the answer address what was actually asked?
   - Conciseness: Is it brief without losing critical information?
   - Structural Clarity: Is it well-organized with clear formatting?
   - Completeness: Does it cover all relevant meeting content?

2. Use LLM-as-judge with calibrated prompts (CompassJudger or Prometheus)
3. Calibrate via isotonic regression against human ratings (target r>=0.78)
4. Compute composite: equal-weight average of all five dimensions

Sample evaluation output for a response:
  Factual Accuracy:     8/10  (one minor date error)
  User Need Alignment:  9/10  (directly answers the budget question)
  Conciseness:          7/10  (includes some redundant context)
  Structural Clarity:   8/10  (uses bullet points, clear headers)
  Completeness:         9/10  (covers all three discussed budget items)
  ----------------------------------------
  Composite Score:      8.2/10
  Latency:              0.8s  (FAST path)
  Quality-Latency J:    8.2 - 0.05*0.8 - 0.01*150 = 6.66
```

**Example 3: Classifying query complexity for routing**

```
User: How do I classify meeting questions so my system knows which ones
need deep reasoning vs. quick answers?

Approach:
1. Score each query on four axes:

   Cognitive Load (CL):
     Low    -> "What time does the meeting end?" (direct extraction)
     Medium -> "Summarize the discussion on Q3 targets" (multi-utterance)
     High   -> "Compare the two proposals and recommend one" (multi-hop)

   Context Dependency (CD):
     None         -> "What is our company policy on PTO?"
     Recent       -> "What did Sarah just say about the deadline?"
     Long-range   -> "Earlier we discussed a risk -- what was it?"
     Cross-meeting-> "How does this compare to last week's decision?"

   Domain Knowledge (DK):
     General -> "Who is speaking?"
     Basic   -> "What does ARR mean in this context?"
     Expert  -> "Is this HIPAA-compliant given the data sharing plan?"

   Task Execution (TE):
     Low    -> "Repeat the last action item"
     Medium -> "Organize the discussed tasks by priority"
     High   -> "Draft a project plan based on this meeting"

2. Map to 13 consolidated classes (from 108 Cartesian cells):
   - 38% of real queries fall in low-CL/no-CD (-> FAST path)
   - 29% medium complexity (-> SLOW path)
   - 18% high complexity (-> SLOW+RAG)
   - 15% cross-meeting (-> SLOW+CROSS)

3. Train classifier on labeled examples with focal loss to handle imbalance.
```

## Best Practices

- **Do:** Start with the FAST path as default and only escalate to SLOW paths when the classifier confidence exceeds a threshold. In production, 38% of queries are simple -- serving these in sub-1s dramatically improves perceived responsiveness.
- **Do:** Use sentinel token gating as a safety valve. If the Planner detects within 400ms that a query was misrouted to the slow path, fall back to the Talker immediately rather than completing expensive reasoning.
- **Do:** Calibrate your evaluation rubric against human ratings before using it for optimization. Uncalibrated LLM judges can drift systematically on dimensions like Conciseness vs. Completeness (which naturally trade off).
- **Do:** Weight the quality-latency objective carefully. The paper's beta=0.05 latency penalty is tuned for enterprise meetings where 20s is acceptable for complex queries. For real-time chat, increase beta significantly.
- **Avoid:** Training the routing classifier on LLM-judge scores directly -- this creates circular dependency. Use proxy metrics (ROUGE-L for factual, regex for structured output, cache hits for cross-meeting) as training signal instead.
- **Avoid:** Over-consolidating complexity classes. The jump from 108 cells to fewer than 10 classes loses routing precision. The paper's 13-class consolidation preserves enterprise frequency distribution while remaining trainable.

## Error Handling

- **Misrouted queries (slow path for simple questions):** The sentinel token gate catches most cases within 400ms. Monitor the miss-trigger rate (target < 2%) and retrain the classifier when it drifts above 5%.
- **Tool invocation failures:** If RAG retrieval returns low-confidence results (below 0.5 cache hit rate), fall back to in-meeting context only rather than returning uncertain information. If web search times out, proceed with available context and flag the response as potentially incomplete.
- **Evaluation dimension conflicts:** Conciseness and Completeness scores naturally anti-correlate. When they diverge by more than 3 points, inspect whether the response is either padding with irrelevant content (low conciseness, high completeness) or truncating critical details (high conciseness, low completeness). Adjust response generation prompts accordingly.
- **Cross-meeting context staleness:** When aggregating across sessions, verify temporal relevance. Decisions from meetings older than the configured window may be superseded. Surface timestamps alongside cross-meeting citations.
- **Classifier drift in production:** Enterprise meeting topics shift over time. Retrain the routing MLP monthly on recent query-action pairs, using the replay buffer approach (8,500+ tuples minimum).

## Limitations

- The 300K-parameter routing classifier requires labeled query-action pairs for training. Cold-start deployments must rely on rule-based routing (e.g., keyword heuristics for complexity axes) until sufficient data is collected.
- The five-dimension evaluation is calibrated against enterprise meeting scenarios (finance, healthcare, technology). Domains with very different communication patterns (e.g., creative brainstorming, legal depositions) may need recalibrated rubrics.
- Cross-meeting aggregation assumes structured meeting records with aligned transcripts. Unstructured audio-only archives without reliable ASR will degrade the SLOW+CROSS path.
- The quality-latency tradeoff coefficients (alpha=1.0, beta=0.05, gamma=0.01) are tuned for GPU-local deployment on RTX 4090-class hardware. Cloud API deployments have different cost structures requiring retuning gamma.
- The framework evaluates textual response quality only. It does not cover non-verbal meeting dimensions like tone analysis, visual presentation quality, or participant engagement metrics.

## Reference

- **Paper:** [MeetBench-XL: Calibrated Multi-Dimensional Evaluation and Learned Dual-Policy Agents for Real-Time Meetings](https://arxiv.org/abs/2602.03285v1) -- Look for Section 3 (four-axis taxonomy and 13-class consolidation), Section 4 (dual-policy architecture and sentinel gating), and Section 5 (five-dimension evaluation with isotonic calibration).
- **Code:** [github.com/huyuelin/MeetBench](https://github.com/huyuelin/MeetBench) -- Reference implementation with CompassJudger/Prometheus evaluators and Talker/Planner agent code.