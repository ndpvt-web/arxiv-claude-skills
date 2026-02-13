---
name: "refer-agent-collaborative-multi-agent-system"
description: "Build collaborative multi-agent systems that use alternating reasoning-reflection cycles with specialized agent roles, coarse-to-fine selection, dynamic focus layouts, and chain-of-reflection verification. Use when asked to: 'build a multi-agent pipeline with self-verification', 'create agents that reflect on their own outputs', 'design a coarse-to-fine selection system', 'implement a questioner-responder verification loop', 'build a reasoning pipeline with iterative refinement', 'create a multi-step agent workflow with feedback loops'."
---

# Refer-Agent: Collaborative Multi-Agent System with Reasoning and Reflection

This skill teaches Claude to design and implement collaborative multi-agent systems inspired by Refer-Agent, a framework that decomposes complex tasks into specialized agent roles connected by alternating reasoning-reflection cycles. The core innovation is a Chain-of-Reflection mechanism where a Questioner-Responder pair verifies intermediate results and feeds corrections back into earlier pipeline stages, enabling iterative self-refinement without supervised fine-tuning. This pattern generalizes beyond video segmentation to any multi-step pipeline where intermediate outputs need verification before downstream consumption.

## When to Use

- When the user needs a multi-agent pipeline where each agent handles a distinct subtask (selection, analysis, grounding, execution) and results must be verified between stages
- When building a system that must select the best candidates from a large pool using a coarse-then-fine scoring strategy (e.g., document retrieval, frame sampling, candidate ranking)
- When implementing self-verification loops where an agent checks its own or another agent's outputs before proceeding
- When designing a dynamic focus mechanism that allocates more processing resources to the highest-priority input while still maintaining context from secondary inputs
- When the user asks for an iterative refinement pipeline that retries with explicit feedback when intermediate quality checks fail
- When building a zero-shot or plug-and-play agent architecture where the underlying model can be swapped without retraining

## Key Technique

**Decomposition into Specialized Agents with Feedback Loops.** Refer-Agent breaks a complex task into four sequential agents: (1) a Selection Agent that picks the best inputs via coarse-to-fine filtering, (2) an Analysis Agent that interprets intent and generates structured descriptions, (3) a Grounding Agent that localizes targets, and (4) an Execution Agent that produces final outputs. The critical insight is that these agents do not simply run in a linear pipeline. Instead, a Chain-of-Reflection mechanism sits between stages and can send execution back to earlier agents with specific corrective feedback.

**Chain-of-Reflection: Questioner-Responder Verification.** Between reasoning stages, a Questioner agent generates targeted verification questions in two categories: *existence checks* (is the target present, visible, and optimally represented?) and *consistency checks* (do detected attributes match the original query?). A separate Responder agent answers these questions by re-examining the intermediate outputs. If verification fails, the system does not simply retry blindly; it generates explicit feedback describing *what went wrong* and routes it to the specific upstream agent that needs correction. This alternation continues until verification passes or a maximum turn limit (default: 4) is reached.

**Coarse-to-Fine Selection with Dynamic Focus.** The selection stage uses a two-pass strategy: first, a fast similarity metric (e.g., embedding cosine distance) coarsely filters N candidates from the full pool, then a more expensive model scores the top-K using a weighted combination (e.g., `S = 0.3 * S_fast + 0.7 * S_detailed`). The Dynamic Focus Layout then allocates asymmetric resources to the highest-scored candidate (larger context, more detail) while keeping secondary candidates as compressed context, ensuring the downstream agent concentrates on the most promising candidate without losing situational awareness.

## Step-by-Step Workflow

1. **Decompose the task into 3-5 specialized agent roles.** Identify the sequential subtasks: selection/filtering, intent analysis, target localization/grounding, and final execution. Define clear input/output contracts for each agent.

2. **Implement the Coarse-to-Fine Selection Agent.** First pass: use a fast scoring function (embeddings, heuristics, keyword matching) to reduce the candidate pool from N to a manageable set. Second pass: apply a more expensive evaluator and combine scores with weighted fusion (`S = alpha * S_fast + beta * S_detailed`, where `beta > alpha`). Return the top-K ranked candidates.

3. **Build the Dynamic Focus Layout.** For downstream agents, present the highest-scored candidate with maximum detail (full context, high resolution, complete metadata) and remaining candidates in compressed form (summaries, thumbnails, key fields only). This focuses the agent's attention without discarding context.

4. **Implement the Analysis Agent.** This agent receives the focused layout and the original query. It decomposes the query into structured attributes (e.g., categories, properties, constraints) and generates detailed intermediate descriptions that the Grounding Agent can act on.

5. **Implement the Grounding Agent.** Using the structured descriptions from the Analysis Agent, localize or identify the target within the selected candidates. Output bounding regions, identifiers, or structured references.

6. **Build the Existence Reflection check.** Create a Questioner that generates three verification questions: (a) Is the target clearly present in the selected candidate? (b) Does the selection cover all aspects of the query? (c) Is there a better candidate among the secondary options? A Responder evaluates each question. If any check fails, generate specific feedback and route it back to the Selection Agent for re-execution.

7. **Build the Consistency Reflection check.** The Questioner decomposes the original query into attribute-level checks and generates targeted questions about both high-level concepts and fine-grained details. The Responder validates each attribute against the Grounding Agent's output. If inconsistency exceeds a threshold (e.g., >30% of attributes fail), generate feedback and route it back to the Analysis Agent.

8. **Wire up the alternating reasoning-reflection loop.** After each reasoning stage, run the corresponding reflection check. On failure, feed the corrective feedback into the appropriate upstream agent and re-execute from that point. Continue until both reflection checks pass or the maximum turn count is reached.

9. **Implement the Execution Agent.** Once verification passes, the final agent produces the output using the validated intermediate results. This agent trusts its inputs because they have been verified.

10. **Add termination safeguards.** Set a `MAX_TURNS` limit (default: 4) to prevent infinite loops. If the limit is reached without passing verification, return the best result so far with a confidence indicator.

## Concrete Examples

**Example 1: Multi-Agent Document QA Pipeline**

User: "Build a system that answers questions about a large document corpus by selecting relevant passages, analyzing them, and verifying the answer."

Approach:
1. **Selection Agent** - Coarse pass: embed the query and all document chunks, retrieve top-50 by cosine similarity. Fine pass: re-rank with an LLM scorer, combine scores (`S = 0.3 * cosine + 0.7 * LLM_score`), select top-5.
2. **Dynamic Focus Layout** - Present the top-ranked passage in full with surrounding context. Present passages 2-5 as one-sentence summaries with source references.
3. **Analysis Agent** - Decompose the query into sub-questions and identify which attributes the answer must contain (dates, names, quantities, causal relationships).
4. **Grounding Agent** - Extract specific answer spans from the focused passage, linking each to the required attributes.
5. **Existence Reflection** - Questioner: "Does the top passage contain information directly relevant to the query?" / "Are all sub-questions addressable from this passage?" / "Would any of the secondary passages be a better primary source?" Responder evaluates. If the top passage is inadequate, feed back "Passage lacks [specific attribute], re-rank with emphasis on [attribute]" to the Selection Agent.
6. **Consistency Reflection** - Questioner generates per-attribute checks: "Does the extracted date match the query's time constraint?" Responder validates. On failure, re-run Analysis Agent with feedback.
7. **Execution Agent** - Compose the final answer from verified extractions.

Output:
```python
class DocumentQAPipeline:
    def __init__(self, max_turns=4):
        self.selector = CoarseToFineSelector(alpha=0.3, beta=0.7, top_n=50, top_k=5)
        self.analyzer = IntentAnalyzer()
        self.grounder = SpanExtractor()
        self.reflector = ChainOfReflection(max_turns=max_turns)

    def answer(self, query, corpus):
        for turn in range(self.reflector.max_turns):
            passages = self.selector.select(query, corpus)
            focused = DynamicFocusLayout(passages, primary_detail="full", secondary_detail="summary")
            attributes = self.analyzer.decompose(query, focused)
            spans = self.grounder.extract(attributes, focused)

            existence_ok, existence_fb = self.reflector.check_existence(query, passages, spans)
            if not existence_ok:
                self.selector.refine(existence_fb)
                continue

            consistency_ok, consistency_fb = self.reflector.check_consistency(query, attributes, spans)
            if not consistency_ok:
                self.analyzer.refine(consistency_fb)
                continue

            return self.compose_answer(spans, confidence="verified")
        return self.compose_answer(spans, confidence="best_effort")
```

**Example 2: Code Review Agent with Self-Verification**

User: "Create a multi-agent code review system that selects relevant files, analyzes changes, identifies issues, and verifies its findings before reporting."

Approach:
1. **Selection Agent** - Coarse: identify all changed files from the diff. Fine: score each file by change magnitude and architectural importance, select top-K for deep review.
2. **Dynamic Focus** - Present the highest-impact file with full diff and surrounding context (20 lines). Show remaining files as change summaries (lines added/removed, function signatures affected).
3. **Analysis Agent** - For the focused file, decompose the review into categories: correctness, security, performance, style. Generate structured descriptions of what each change does.
4. **Grounding Agent** - Map each concern to specific line ranges and code patterns.
5. **Existence Reflection** - "Does the primary file actually contain the most critical changes?" / "Are there security-relevant changes in secondary files being overlooked?" On failure, re-rank files.
6. **Consistency Reflection** - "Does the identified bug actually manifest given the surrounding code context?" / "Is the flagged performance issue real or a false positive?" On failure, re-analyze with additional context.
7. **Execution** - Produce the final review with verified findings only.

Output:
```python
review_pipeline = AgentPipeline(
    agents=[
        FileSelector(coarse="git_diff_stats", fine="llm_importance_score"),
        ChangeAnalyzer(categories=["correctness", "security", "performance"]),
        IssueGrounder(output="line_ranges_with_evidence"),
    ],
    reflections=[
        ExistenceReflection(questions=[
            "Does the primary file contain the most critical changes?",
            "Are security-relevant changes in secondary files covered?",
        ], on_fail="re_select"),
        ConsistencyReflection(questions=[
            "Does the identified issue manifest in actual execution paths?",
            "Is the severity assessment consistent with the code context?",
        ], on_fail="re_analyze", inconsistency_threshold=0.3),
    ],
    max_turns=4,
)
findings = review_pipeline.run(diff=pr_diff)
```

**Example 3: Data Pipeline Validation Agent**

User: "Build a multi-agent system that selects the best data transformation approach, applies it, and validates the output before delivering results."

Approach:
1. **Selection Agent** - Coarse: enumerate candidate transformations (joins, aggregations, filters). Fine: score each by schema compatibility and query alignment.
2. **Analysis Agent** - Parse the user's intent into a structured transformation plan with expected output schema.
3. **Grounding Agent** - Execute the transformation and produce intermediate results.
4. **Existence Reflection** - "Does the output contain all expected columns?" / "Are there null values in required fields?" Re-select transformation on failure.
5. **Consistency Reflection** - "Do row counts match expectations?" / "Are aggregated values within plausible ranges?" Re-analyze on failure.
6. **Execution** - Deliver the validated dataset.

Output:
```
Turn 1: Selected LEFT JOIN, but Existence Reflection found missing rows
  -> Feedback: "LEFT JOIN drops unmatched rows from right table; switch to FULL OUTER JOIN"
Turn 2: Re-selected FULL OUTER JOIN, Existence passed
  -> Consistency check: row count 1,247 vs expected ~1,200 (within tolerance)
  -> Both checks passed. Delivering validated result.
```

## Best Practices

- **Do:** Define explicit input/output contracts between agents. Each agent should produce a structured output that the next agent and the reflection mechanism can inspect programmatically.
- **Do:** Make reflection feedback specific and actionable. Instead of "try again," generate feedback like "The selected candidate lacks attribute X; prioritize candidates containing X in the next selection round."
- **Do:** Use asymmetric resource allocation in the Dynamic Focus Layout. The primary candidate should get 60-70% of the context budget; secondary candidates share the remainder as compressed summaries.
- **Do:** Separate Existence checks (is the right thing selected?) from Consistency checks (is the analysis correct?) and route failures to different upstream agents accordingly.
- **Avoid:** Running reflection checks without a maximum turn limit. Always set `MAX_TURNS` (typically 3-5) to prevent infinite refinement loops.
- **Avoid:** Treating all reflection failures the same. Existence failures should re-trigger selection; consistency failures should re-trigger analysis. Routing feedback to the wrong agent wastes turns.
- **Avoid:** Making the Questioner and Responder the same agent instance. Separation prevents confirmation bias; the Questioner generates adversarial checks while the Responder evaluates honestly.

## Error Handling

- **Reflection loop exhaustion:** When `MAX_TURNS` is reached without passing verification, return the best intermediate result with a confidence flag (`"best_effort"` vs `"verified"`). Log which specific checks failed for debugging.
- **Empty selection:** If the Coarse-to-Fine selector returns zero candidates above threshold, fall back to returning the top-K by the fast metric alone, and flag the result as low-confidence.
- **Conflicting reflection feedback:** If Existence Reflection and Consistency Reflection produce contradictory feedback (e.g., "select a different candidate" vs "the current candidate is correct but misanalyzed"), prioritize Consistency feedback since the candidate has already passed existence checks once.
- **Agent timeout or failure:** If any agent in the pipeline fails, skip its reflection check, pass through the best available intermediate result, and annotate the final output with which verification stages were skipped.
- **Oscillating refinements:** If the same feedback is generated on consecutive turns (the system keeps selecting the same candidate and failing the same check), inject a diversity constraint: exclude previously-selected candidates from the next selection round.

## Limitations

- **Latency cost of reflection loops:** Each reflection cycle adds a full round of Questioner-Responder evaluation. For latency-sensitive applications, reduce `MAX_TURNS` to 2 or skip Consistency Reflection for low-stakes queries.
- **Dependent on question quality:** The Chain-of-Reflection is only as good as the Questioner's ability to generate targeted, falsifiable verification questions. Poorly constructed questions lead to rubber-stamp approvals.
- **Not suitable for single-step tasks:** If the task has no meaningful intermediate results to verify (e.g., a single function call), the overhead of the multi-agent pipeline and reflection mechanism is not justified.
- **Scoring function sensitivity:** The Coarse-to-Fine selection depends heavily on the weighting between fast and detailed scorers. The recommended `alpha=0.3, beta=0.7` works well when the detailed scorer is reliable; if it is noisy, increase `alpha`.
- **Does not replace domain-specific fine-tuning for maximum accuracy.** This pattern optimizes for flexibility and plug-and-play model swapping. For domains where labeled data is abundant, supervised approaches may still achieve higher absolute performance.

## Reference

[Refer-Agent: A Collaborative Multi-Agent System with Reasoning and Reflection for Referring Video Object Segmentation](https://arxiv.org/abs/2602.03595v2) - Jiang et al., 2026. Focus on Section 3 (Method) for the Coarse-to-Fine selection algorithm, Dynamic Focus Layout specifications, and the Chain-of-Reflection protocol with Existence and Consistency verification stages.