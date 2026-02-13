---
name: "drpg-decompose-retrieve-plan"
description: >
  Structured rebuttal and critique-response generation using the DRPG framework
  (Decompose, Retrieve, Plan, Generate). Breaks down multi-point feedback into
  atomic concerns, retrieves supporting evidence from source documents, selects
  a rebuttal strategy (clarification vs. justification), and generates targeted
  point-by-point responses.
  Trigger phrases:
  - "Write a rebuttal to this review"
  - "Respond to reviewer comments"
  - "Address this feedback point by point"
  - "Help me respond to this code review / PR review"
  - "Draft a response to these critique points"
  - "Decompose this review and plan responses"
---

# DRPG: Decompose, Retrieve, Plan, Generate

This skill enables Claude to produce structured, evidence-backed responses to multi-point critiques using the DRPG agentic framework from Han et al. (2026). Instead of generating a single monolithic reply to a block of feedback, DRPG decomposes the critique into individual atomic concerns, retrieves the most relevant evidence from the source material (paper, codebase, design doc, PR diff), classifies each concern into a rebuttal strategy, and then generates a focused response per point. This produces responses that are more targeted, better-evidenced, and more persuasive than naive end-to-end generation.

## When to Use

- When a user asks to respond to an academic peer review with multiple weakness points or questions
- When a user needs to address a detailed code review or PR review that raises several distinct issues
- When a user receives multi-point feedback on a design document, RFC, or technical proposal and needs structured responses
- When a user wants to draft a rebuttal letter for a conference submission
- When a user asks to analyze reviewer comments and plan a response strategy before writing
- When feedback is long and contains mixed concerns (misunderstandings, legitimate weaknesses, requests for additional work) that require different response approaches

## Key Technique

**The core insight of DRPG is that responding to multi-point feedback is not a single generation task but a four-stage agentic pipeline.** Directly prompting an LLM with an entire review and paper produces vague, generic responses because the model struggles with long-context understanding and cannot differentiate between concerns that require different strategies. DRPG solves this by breaking the problem into manageable subtasks that each have well-defined inputs and outputs.

**The Decompose-Retrieve-Plan-Generate pipeline works as follows.** The Decomposer extracts each distinct weakness or question from the review as a separate atomic concern, preserving the reviewer's original language. The Retriever uses embedding-based semantic search (cosine similarity over dense embeddings) to find the K most relevant paragraphs from the source document for each concern, reducing context by ~75%. The Planner then generates candidate perspectives for addressing each concern and classifies them into two strategy types: **Clarification** (the reviewer misunderstood or missed something present in the work) or **Justification** (the concern is acknowledged but argued to not undermine the work). Finally, the Generator produces a concise, focused response for each point using only the retrieved evidence and selected strategy.

**What makes the Planner critical is strategy selection.** Many rebuttals fail because they adopt the wrong posture -- conceding a point that was actually a misunderstanding, or arguing against a valid concern. The Planner's two-strategy framework (Clarification vs. Justification) forces an explicit decision about the response direction before any prose is generated. In the original paper, this achieves 98%+ accuracy in choosing the correct direction, which is the single largest driver of rebuttal quality.

## Step-by-Step Workflow

1. **Collect inputs.** Gather the full critique/review text and the source document being reviewed (paper, codebase, design doc, or PR diff). If the source is large, split it into logical paragraphs or sections.

2. **Decompose the review into atomic concerns.** Parse the review to extract each distinct weakness, question, or confusion as a separate item. Preserve the reviewer's original phrasing. Omit minor issues (typos, formatting) and focus on substantive points. Output a numbered JSON list of concern strings.

3. **Retrieve relevant evidence for each concern.** For each atomic concern, identify the most relevant sections of the source document. Use semantic similarity: embed the concern and each paragraph/section, then select the top-K (typically 10-15) most similar passages. Merge retrieved passages that share section headings to maintain context.

4. **Generate candidate perspectives for each concern.** For each concern, brainstorm up to 5 candidate response perspectives. Each perspective MUST be classified as one of two types:
   - **Clarification:** The reviewer has a factual error or misunderstanding. The information is actually present in the work, or the methodology is correct but was misread.
   - **Justification:** The concern is acknowledged as valid in isolation, but the work remains sound because the concern is infeasible, unnecessary, or does not undermine the core contribution.

5. **Select the best perspective (Plan).** For each concern, evaluate which candidate perspective is best supported by the retrieved evidence. Choose the perspective where the source document provides the strongest backing. If no perspective has strong support, flag the concern as one where a concession or future-work note may be appropriate.

6. **Generate a point-by-point response.** For each concern, generate a concise response (150-250 words) that uses the selected perspective and cites specific evidence from the retrieved passages. Do not introduce claims unsupported by the source material.

7. **Assemble the full rebuttal.** Combine all point-by-point responses into a structured rebuttal letter with a brief opening acknowledgment, each concern quoted and followed by its response, and a brief closing.

8. **Review for consistency.** Check that no response contradicts another, that the tone is professional and confident throughout, and that no response makes promises about future work or revisions (unless the user explicitly wants this).

## Concrete Examples

**Example 1: Academic Paper Review Rebuttal**

```
User: "I got this review for my NeurIPS paper on graph neural networks.
Help me write a rebuttal:

Reviewer comments:
'The paper proposes a new GNN aggregation but lacks comparison with
GraphSAGE and GAT on heterogeneous graphs. The theoretical analysis
in Section 4 assumes bounded degree which is unrealistic for
real-world networks. Also, the ablation study only tests on 2/5
datasets. Why were Cora and PubMed excluded?'"

Approach:
1. DECOMPOSE into atomic concerns:
   - Concern 1: Missing comparison with GraphSAGE and GAT on heterogeneous graphs
   - Concern 2: Theoretical analysis assumes bounded degree, unrealistic for real-world networks
   - Concern 3: Ablation study only covers 2/5 datasets, Cora and PubMed excluded

2. RETRIEVE relevant evidence from the paper for each concern
   (user provides paper text or Claude works with available context)

3. PLAN strategy per concern:
   - Concern 1 -> Clarification: Check if comparisons exist in appendix or
     supplementary. If not -> Justification: Explain why heterogeneous graph
     comparison is outside scope or that the method is designed for homogeneous settings.
   - Concern 2 -> Justification: Bounded-degree assumption is standard in
     spectral GNN theory (cite prior work); empirical results show it holds
     in practice on tested graphs.
   - Concern 3 -> Clarification: If ablation was run on all datasets but only
     2 shown for space, clarify and point to appendix. If not run, acknowledge.

4. GENERATE point-by-point:

Output:
Dear Reviewer,

Thank you for your constructive feedback. We address each point below.

**Q1: Missing comparison with GraphSAGE and GAT on heterogeneous graphs.**
Our method targets homogeneous graph settings, which is the focus stated
in Section 1. GraphSAGE and GAT comparisons on homogeneous benchmarks
are in Table 2, where our method outperforms both. Heterogeneous graph
evaluation would require architectural modifications (e.g., typed
message passing) that are orthogonal to our aggregation contribution.

**Q2: Bounded-degree assumption is unrealistic.**
The bounded-degree assumption (Assumption 4.1) follows the standard
framework established by [Xu et al., 2019] and [Morris et al., 2019].
As shown in Table 1, the maximum degree across our five benchmarks
ranges from 168 to 2,112, and Theorem 4.3 holds empirically with
approximation error < 0.01 in all cases (Appendix C, Figure 7).

**Q3: Ablation study missing Cora and PubMed.**
The full ablation across all five datasets is provided in Appendix D,
Table 8. We selected ogbn-arxiv and Reddit for the main text as they
are the largest benchmarks and most clearly demonstrate the scaling
behavior of each component. Results on Cora and PubMed are consistent.

We hope these clarifications address your concerns.
```

**Example 2: Code Review / PR Feedback Response**

```
User: "My PR got these review comments. Help me respond:

1. 'This retry logic with exponential backoff should use jitter
   to avoid thundering herd.'
2. 'The connection pool size of 100 seems arbitrary. What's the
   justification?'
3. 'You're catching all exceptions on line 45. This will swallow
   errors silently.'"

Approach:
1. DECOMPOSE: Three atomic concerns already separated by reviewer.

2. RETRIEVE: Examine the PR diff and related code for evidence.

3. PLAN:
   - Concern 1 -> Clarification: Check if jitter is already present.
     If not -> acknowledge and offer to add it (legitimate improvement).
   - Concern 2 -> Justification: Pool size derived from load testing
     or config, cite evidence from code/docs.
   - Concern 3 -> Clarification: Check if the except block actually
     logs and re-raises, or if it truly swallows errors.

4. GENERATE:

Output:
Thanks for the thorough review. Addressing each point:

**Re: Retry jitter.** Good catch -- the current backoff in `retry_with_backoff()`
(line 23) does include jitter via `random.uniform(0, delay * 0.1)` added to
each delay, but this is subtle. I'll add a comment to make it explicit.

**Re: Connection pool size.** The pool size of 100 comes from our load test
results in the `perf/` directory (see `load_test_results_2024.md`): at p99
we measured 87 concurrent connections under peak traffic. The value of 100
provides ~15% headroom. I'll add a code comment referencing this.

**Re: Broad exception handling.** The `except Exception` on line 45 does
log the full traceback (`logger.exception(...)` on line 46) and re-raises
after incrementing the retry counter (line 48). It's not swallowed -- but
I agree the bare `Exception` is too broad. I'll narrow it to
`ConnectionError | TimeoutError` to be explicit about what we retry.
```

**Example 3: Design Document Feedback**

```
User: "Our architect reviewed my design doc for the caching layer
and had these concerns. Help me structure a response:

- 'Redis adds operational complexity. Have you considered an
  in-process cache like Caffeine?'
- 'The TTL of 24h for user profiles seems too long given GDPR
  right-to-erasure requirements.'
- 'No mention of cache warming strategy for cold starts.'"

Approach:
1. DECOMPOSE: Three concerns extracted.

2. RETRIEVE: Reference the design doc sections on technology choice,
   data lifecycle, and deployment.

3. PLAN:
   - Redis vs. in-process -> Justification: Multi-instance deployment
     requires shared cache; in-process won't work.
   - TTL and GDPR -> Clarification: Check if cache invalidation on
     deletion is already described. If not, acknowledge gap.
   - Cache warming -> Acknowledge gap, propose solution.

4. GENERATE:

Output:
**Re: Redis vs. in-process cache.**
Our deployment runs 8+ instances behind a load balancer (Section 2.1).
An in-process cache like Caffeine would give each instance its own copy,
leading to inconsistencies and 8x memory overhead for the same data.
Redis provides a single shared cache with built-in eviction and pub/sub
for invalidation across instances.

**Re: 24h TTL and GDPR.**
Good point. The 24h TTL is for the happy path, but Section 3.4 describes
our invalidation hook: when a user deletion request comes in, we
explicitly call `cache.delete(user:*)` synchronously before returning
the 200 response. I'll add a cross-reference to make this clearer.

**Re: Cache warming.**
This is a genuine gap in the doc. I'll add a section describing a
warming job that pre-populates the top-1000 most active user profiles
on deploy, based on the access frequency data we already track.
```

## Best Practices

**Do:**
- Always decompose first, even if the review seems short. Separate concerns enable separate strategies.
- Preserve the reviewer's exact wording when quoting concerns. Paraphrasing can misrepresent the issue.
- Explicitly choose Clarification or Justification for each point before writing. This prevents muddled responses that half-concede and half-argue.
- Ground every claim in specific evidence from the source material (section numbers, code line references, data from tables).
- Keep each point-by-point response concise (150-250 words). Longer responses dilute the argument.
- When a concern is genuinely valid and unsupported by evidence, acknowledge it honestly rather than generating a weak justification.

**Avoid:**
- Do not generate a single long paragraph addressing all concerns at once. The decomposed structure is the core value.
- Do not promise future work or revisions unless the user explicitly requests it. Rebuttals should defend what exists.
- Do not use emotional appeals or praise the reviewer excessively. Be professional and factual.
- Do not fabricate evidence. If the source material doesn't support a clarification, switch to justification or acknowledge the gap.

## Error Handling

- **Review is too vague to decompose:** If a critique point is vague (e.g., "the methodology is weak"), ask the user to identify which specific aspect the reviewer is concerned about, or decompose it into plausible sub-concerns and confirm with the user.
- **No relevant evidence found in source material:** Flag the concern as unsupported and suggest the user either provide additional context or acknowledge the gap. Do not fabricate references.
- **Both strategies seem wrong:** If a concern is neither a misunderstanding (Clarification) nor defensible (Justification), it may be a legitimate weakness. Recommend the user acknowledge it with a brief explanation of why it doesn't invalidate the overall contribution, or propose concrete remediation.
- **Contradictory concerns from multiple reviewers:** Decompose each reviewer's concerns separately. When two reviewers contradict each other (one says "too complex," another says "too simple"), note this explicitly and respond to each on its own terms.

## Limitations

- This approach works best when the source document is available for evidence retrieval. Without it, responses rely on the user's verbal summary and may lack specific citations.
- The two-strategy framework (Clarification vs. Justification) covers the vast majority of rebuttal scenarios but does not handle meta-concerns like "this paper is out of scope for this venue" which require a different kind of argument.
- For multi-round review discussions (e.g., second-round reviewer responses), the framework applies iteratively but each round needs the full conversation history to avoid contradicting earlier responses.
- The quality of decomposition depends on the review quality. Poorly written reviews with interleaved concerns in a single sentence are harder to decompose cleanly.

## Reference

**Paper:** Han, P., Yu, Y., Xu, J., & You, J. (2026). *DRPG: An Agentic Framework for Academic Rebuttal.* arXiv:2601.18081.
https://arxiv.org/abs/2601.18081v1

**Key takeaway:** The Planner's two-strategy classification (Clarification vs. Justification) is the highest-leverage component -- choosing the right response direction matters more than generating eloquent prose. See Section 3.3 and the ablation in Section 5.3 of the paper.

**Code:** https://github.com/ulab-uiuc/DRPG-RebuttalAgent