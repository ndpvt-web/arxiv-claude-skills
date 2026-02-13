---
name: "deep-researcher-sequential-plan"
description: |
  Conduct deep, multi-step research on complex topics using Sequential Plan Refinement with Reflection and Candidates Crossover.
  Maintains a Global Research Context across iterations so each search step builds on prior findings, avoids redundancy, and adapts the plan at runtime.
  Use this skill when: "research this topic in depth", "write a comprehensive report on", "deep dive into",
  "investigate and synthesize findings on", "generate a research report about", "analyze this complex topic thoroughly".
---

# Deep Researcher with Sequential Plan Reflection and Candidates Crossover

This skill enables Claude to conduct deep, iterative research on complex topics by maintaining a centralized Global Research Context and refining the research plan after each search step. Unlike parallel research strategies that split a topic into independent subtopics and research them in isolation (creating knowledge silos), this sequential approach lets each step see everything discovered so far, reflect on whether the plan still makes sense, and adapt -- adding new subtopics, re-prioritizing, or dropping redundant paths. A Candidates Crossover mechanism runs multiple answer-generation passes with varied parameters per query, then merges them into a single high-fidelity answer, broadening the search space without multiplying tool calls.

## When to Use

- When a user asks for a comprehensive research report on a PhD-level or multi-faceted topic (e.g., "Write a deep research report on federated learning in healthcare")
- When the research question is open-ended and the full scope cannot be known upfront -- subtopics will emerge as you search
- When the user asks to "investigate", "deep dive", or "thoroughly analyze" a subject that spans multiple domains or perspectives
- When prior parallel research produced shallow or redundant coverage and the user wants a more integrated result
- When synthesizing findings from many web sources into a single cohesive narrative with high fact density
- When the user asks to iteratively refine a research plan based on what has been discovered so far

## Key Technique

**Sequential Plan Refinement via Reflection.** The system starts by generating an initial research plan -- a numbered list of concrete subtopics and search angles. After executing each search step and recording the results into a Global Research Context (a structured log of every query, answer, and raw artifact), a Reflection phase fires: the planning agent reviews the entire context, checks whether the current plan still covers the topic adequately, identifies knowledge gaps that only became visible after earlier searches, and proposes plan mutations (add steps, reorder, drop redundant ones). This is the core advantage over parallel approaches: the plan evolves with the evidence.

**Candidates Crossover.** For each search query, instead of generating a single answer, the system spawns multiple answer candidates using different generation parameters (e.g., varied temperature and top-k). Each candidate produces a concise, fact-dense answer from the same search results. A crossover step then merges the candidates, consolidating the best information from each -- capturing facts one candidate emphasized that another missed. This is adapted from Google's TTD-DR Self-Evolution algorithm but omits the revision loop for lower latency.

**One-Shot Report Generation.** After the research loop reaches a completion threshold (roughly 90% of the plan executed and reflected upon), a report-writing pass synthesizes the entire Global Research Context into a cohesive long-form document in a single inference, relying on the high-fidelity context built during sequential reflection rather than iterative report refinement.

## Step-by-Step Workflow

1. **Parse the research request.** Extract the core topic, any constraints (scope, depth, domains to cover, output format), and the user's intent (survey, comparison, gap analysis, etc.). Clarify ambiguities before proceeding.

2. **Generate the initial research plan.** Produce a numbered list of 5-12 concrete research steps, each targeting a specific subtopic or angle. Each step should name the information it seeks and why it matters to the overall topic. Store this as the active plan.

3. **Initialize the Global Research Context (GRC).** Create a structured document with three sections: (a) **Plan History** -- the current plan plus all past versions, (b) **Search Trajectories** -- a log of each query and its synthesized answer, (c) **Artifact Store** -- raw facts, numbers, quotes, and source URLs collected during search.

4. **Execute the next plan step: generate search queries.** Read the GRC to understand what has already been covered. Formulate 1-3 non-redundant search queries targeting the current step. Use web search tools (or the user's provided data sources) to retrieve results.

5. **Apply Candidates Crossover to synthesize an answer.** For the retrieved results, generate 2-3 answer candidates with varied reasoning approaches (e.g., one emphasizing quantitative data, one focusing on qualitative insights, one prioritizing recency). Merge the candidates into a single consolidated answer that retains all unique facts, statistics, and citations. Append the consolidated answer and raw artifacts to the GRC.

6. **Reflect on the research plan.** With the updated GRC, critically assess: (a) Does the current plan still cover the topic adequately? (b) Did this step reveal new subtopics or contradictions that need investigation? (c) Are any remaining steps now redundant given what was found? (d) Should the priority order change? Document the reflection reasoning.

7. **Update the plan if reflection warrants it.** Add new steps for discovered gaps, remove or merge redundant steps, reorder based on new priorities. Record the plan mutation in the GRC's Plan History so no information is lost.

8. **Check completion.** If 90%+ of the current plan's steps are executed and the last reflection found no critical gaps, proceed to report generation. Otherwise, loop back to step 4 with the next plan step.

9. **Generate the research report.** In a single synthesis pass, produce a structured long-form report from the entire GRC. The report should have: a clear thesis or framing, logically ordered sections corresponding to research findings, inline citations to sources, and a conclusion that addresses the original research question. Prioritize fact density and coherence.

10. **Present and iterate.** Deliver the report to the user. Offer to drill deeper into any section, update the plan with new angles, or reformat the output.

## Concrete Examples

**Example 1: Multi-domain technical survey**

```
User: "Research the current state of on-device LLM inference --
covering hardware acceleration, quantization techniques, and
real-world deployment challenges. Write a comprehensive report."

Approach:
1. Parse request: three explicit domains (hardware, quantization, deployment),
   survey-style output expected.
2. Initial plan:
   - Step 1: Survey current on-device LLM hardware (NPUs, GPUs, Apple Neural Engine)
   - Step 2: Quantization methods (GPTQ, AWQ, GGUF/llama.cpp approaches)
   - Step 3: Memory and latency benchmarks for popular models on-device
   - Step 4: Real deployment challenges (thermal throttling, battery, privacy)
   - Step 5: Framework ecosystem (MLC-LLM, ExecuTorch, MediaPipe)
   - Step 6: Future directions and open problems
3. Execute Step 1, record findings in GRC.
4. Candidates Crossover for Step 1: Candidate A focuses on chip specs,
   Candidate B on comparative benchmarks, Candidate C on vendor roadmaps.
   Merge into consolidated answer with all unique data points.
5. Reflect: Step 1 revealed that Apple Intelligence uses a novel
   adapter-switching approach not in the original plan. Add Step 2b:
   "Adapter and LoRA switching for on-device personalization."
6. Continue through updated plan, reflecting after each step.
7. Final report: 2500-word structured document with sections, inline
   citations, comparison tables, and a conclusion on the maturation
   trajectory of on-device inference.

Output structure:
# On-Device LLM Inference: Current State and Challenges
## 1. Hardware Landscape
   [findings with specific chip names, TOPS figures, citations]
## 2. Quantization Techniques
   [GPTQ vs AWQ vs GGUF comparison table, accuracy-latency tradeoffs]
## 2b. Adapter Switching for Personalization  ← added via reflection
   [Apple Intelligence approach, LoRA on-device fine-tuning]
## 3. Benchmarks
   [latency/memory tables for Llama, Phi, Gemma on various hardware]
## 4. Deployment Challenges
   [thermal, battery, privacy, UX considerations]
## 5. Framework Ecosystem
   [MLC-LLM, ExecuTorch, MediaPipe comparison]
## 6. Future Directions
   [open problems, expected trajectory]
## Sources
   [numbered list of URLs and papers]
```

**Example 2: Focused investigation with emergent subtopics**

```
User: "Deep dive into why retrieval-augmented generation (RAG) systems
fail in production. I need actionable findings, not a literature review."

Approach:
1. Parse: focus on failure modes, production context, actionable output.
2. Initial plan:
   - Step 1: Common RAG failure taxonomies from practitioner reports
   - Step 2: Retrieval failures (chunking, embedding drift, relevance)
   - Step 3: Generation failures (hallucination despite retrieval, citation errors)
   - Step 4: Infrastructure failures (latency spikes, index staleness)
   - Step 5: Mitigation strategies with evidence of effectiveness
3. Execute Step 1. GRC now contains practitioner blog posts and post-mortems.
4. Reflect: Step 1 revealed that "lost-in-the-middle" attention patterns
   are a dominant failure mode not explicitly in the plan. Also, evaluation
   gaps (teams cannot measure RAG quality) emerged as a root cause.
   Update plan:
   - Insert Step 2b: "Lost-in-the-middle and context window utilization failures"
   - Insert Step 5b: "Evaluation frameworks and observability for RAG"
5. Continue executing. After Step 3, reflection finds that multi-hop
   reasoning failures are a distinct category worth separating out.
   Add Step 3b: "Multi-hop retrieval-generation failures."
6. Generate report structured around failure modes, each with:
   root cause, symptoms, real-world example, and mitigation.

Output: Actionable report with ~8 failure categories (3 emerged from
reflection), each containing diagnosis criteria and fixes.
```

**Example 3: Comparative analysis with crossover benefit**

```
User: "Compare WebSocket, Server-Sent Events, and WebTransport for
real-time data in modern web apps. I need to make an architecture decision."

Approach:
1. Parse: decision-support comparison, three specific technologies.
2. Initial plan:
   - Step 1: WebSocket capabilities, limitations, browser support
   - Step 2: SSE capabilities, limitations, HTTP/2 multiplexing behavior
   - Step 3: WebTransport capabilities, QUIC underpinnings, current support
   - Step 4: Head-to-head performance benchmarks
   - Step 5: Decision framework based on use-case characteristics
3. For each step, Candidates Crossover generates three perspectives:
   - Candidate A: emphasizes raw performance and protocol-level details
   - Candidate B: emphasizes developer experience and ecosystem maturity
   - Candidate C: emphasizes edge cases and production war stories
   Crossover merges all three, ensuring no perspective is lost.
4. Reflect after Step 3: WebTransport search reveals that HTTP/3 proxy
   support is a critical blocker not yet covered. Add Step 3b:
   "Infrastructure compatibility: proxy, CDN, and firewall considerations."
5. Final report includes a decision matrix table and clear recommendations
   keyed to specific use cases (chat, live dashboards, gaming, etc.).
```

## Best Practices

**Do:**
- Keep the Global Research Context as a structured, append-only document. Never delete prior entries -- reflection needs the full history to detect redundancy and gaps.
- Make each search query explicitly non-redundant by checking the GRC's Search Trajectories before generating new queries. Prefix queries with context like "excluding X which was already covered."
- During Candidates Crossover, ensure candidates genuinely differ in focus (quantitative vs. qualitative, recent vs. foundational, mainstream vs. contrarian) rather than just varying temperature randomly.
- Record your reflection reasoning explicitly. Write out what gaps you found and why you are (or are not) modifying the plan. This makes the process auditable and helps the final report writer.
- Set a concrete completion threshold (e.g., 90% of plan steps executed + last reflection found no critical gaps) to avoid infinite research loops.

**Avoid:**
- Do not research all subtopics in parallel and merge at the end. The entire point of this technique is that step N informs step N+1. Parallelism destroys the sequential advantage.
- Do not skip the reflection step even when results seem straightforward. Subtle gaps and emergent connections are only visible when you force a critical review against the full context.
- Do not let the plan grow unboundedly. If reflection keeps adding steps, impose a maximum (e.g., 15 steps) and force prioritization -- drop the least critical additions.
- Do not generate the final report incrementally section-by-section. The one-shot synthesis over the full GRC produces better coherence and avoids repetition across sections.

## Error Handling

- **Search yields no useful results for a step:** Record the null result in the GRC. During reflection, decide whether to reformulate the query with different terms, merge the step into an adjacent one, or mark it as an acknowledged gap in the final report.
- **Candidates Crossover produces contradictory facts:** Flag the contradiction explicitly in the GRC. During the next reflection, add a verification step targeting the specific disagreement. In the final report, present both positions with their sources if unresolved.
- **Plan reflection loops indefinitely (keeps finding gaps):** Enforce a hard cap on reflection-triggered plan mutations (e.g., maximum 3 rounds of additions after the initial plan). After the cap, proceed to report generation and note remaining open questions.
- **Global Research Context exceeds context window:** Summarize older search trajectories while preserving key facts, source URLs, and the artifact store. Keep the most recent 3-4 full trajectories and the complete plan history intact.
- **User's topic is too broad for a single research cycle:** Propose decomposing into 2-3 focused research sessions, each producing its own report, with a final synthesis pass.

## Limitations

- This technique is optimized for depth on a single complex topic. For breadth-first surveys across many unrelated topics, a parallel approach may be more efficient.
- The sequential nature means total latency scales linearly with the number of plan steps. For time-sensitive requests, limit the plan to 5-6 steps and accept reduced coverage.
- Candidates Crossover adds overhead per query. For simple factual lookups where a single search suffices, skip the crossover and use direct search-and-answer.
- The quality of reflection depends on the accumulated GRC. Very early reflections (after only 1-2 steps) have limited context and may not produce meaningful plan mutations. The technique works best after 3+ steps have been executed.
- One-shot report generation can struggle with extremely long GRCs (20+ search trajectories). In such cases, consider a two-pass approach: outline first from the GRC, then fill in sections.

## Reference

**Paper:** [Deep Researcher with Sequential Plan Reflection and Candidates Crossover](https://arxiv.org/abs/2601.20843v1) by Saurav Prateek (2026). Look for: the Global Research Context structure, the 7-step architecture loop, the reflection mechanism that distinguishes this from parallel approaches like STORM and GPT-Researcher, and the Candidates Crossover adaptation from Google's TTD-DR. The system scored 46.21 on DeepResearch Bench, competitive with OpenAI Deep Research (46.45) and ahead of Claude Researcher (45.0) and Perplexity Research (40.46).