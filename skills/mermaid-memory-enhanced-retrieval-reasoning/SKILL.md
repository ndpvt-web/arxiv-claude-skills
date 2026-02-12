---
name: "mermaid-memory-enhanced-retrieval-reasoning"
description: "Memory-enhanced multi-agent retrieval and reasoning for veracity assessment and fact-checking. Use when: 'verify this claim', 'fact-check these statements', 'check if this is true', 'assess the veracity of', 'cross-reference these claims', 'build a fact-checking pipeline'."
---

# MERMAID: Memory-Enhanced Retrieval and Reasoning for Veracity Assessment

This skill enables Claude to implement the MERMAID framework -- a multi-agent system that decomposes claims into structured triplets, retrieves evidence through an iterative Thought-Action-Observation loop, and maintains a persistent evidence memory across claims. The key innovation is coupling retrieval and reasoning into a single ReAct-style cycle while reusing previously gathered evidence via an entity-indexed memory store, eliminating redundant searches and improving verification consistency.

## When to Use

- When the user asks to verify factual claims, fact-check statements, or assess whether information is true or false
- When processing a batch of related claims that may share overlapping entities or evidence (e.g., checking multiple statements from the same article)
- When building an automated fact-checking or claim verification pipeline
- When the user wants to decompose a complex statement into verifiable sub-claims and check each one
- When assessing the veracity of LLM-generated responses against real-world knowledge
- When the user needs a transparent, traceable reasoning chain showing how a verdict was reached

## Key Technique

MERMAID uses four coordinated components: a **Decomposer agent** that parses claims into structured knowledge triplets `(subject, relation, object, attributions)` plus topical keywords; an **Executor agent** that runs a ReAct-style `{Thought -> Action -> Observation}` loop to gather and reason over evidence; a **Toolset** providing granular search capabilities (Wikipedia article/section/fact retrieval, web search, scholarly databases); and a **Memory Module** that persists evidence in an entity-indexed key-value store across claims.

The critical insight is the memory module. Before the Executor begins reasoning on a new claim, the system extracts entities from the decomposition and queries memory for any previously retrieved evidence about those entities. This recalled evidence is injected into the initial prompt as a "warm start," letting the agent skip redundant lookups. After verification completes, newly gathered evidence is committed back to memory keyed by its associated entities. In benchmarks this reduced total tool calls by 16-30%, with the largest gains on claims requiring longer evidence chains.

The Executor's ReAct loop runs for up to `T_max` steps (typically 5). At each step the agent produces a thought (reasoning about what evidence is needed), selects an action (a specific tool call or the terminal `Answer` action), and receives an observation (tool output). The full trajectory becomes the human-readable rationale for the final verdict, providing transparency that static retrieval-then-classify pipelines lack.

## Step-by-Step Workflow

1. **Receive and normalize the claim.** Accept the raw claim text. If the input contains multiple claims, split them into individual statements for sequential processing so the memory module can accumulate evidence across them.

2. **Decompose the claim into structured triplets.** Extract `(subject, relation, object)` triplets and topical keywords from each claim. For example, "Marie Curie won two Nobel Prizes in different scientific disciplines" yields triplets like `(Marie Curie, won, Nobel Prize, {count: two, qualifier: different disciplines})` and keywords `["Nobel Prize", "Marie Curie", "physics", "chemistry"]`.

3. **Extract entity keys for memory lookup.** Collect all unique subjects and objects from the triplets to form the entity set `E_c`. These serve as lookup keys into the evidence memory.

4. **Recall prior evidence from memory.** Query the memory store for each entity in `E_c`. Aggregate all matching evidence entries into `M_c`. If evidence exists, include it in the initial context as pre-gathered facts to avoid re-searching.

5. **Initialize the Executor's ReAct loop.** Construct the initial prompt `P_0` containing: the original claim, the structured decomposition, recalled evidence `M_c`, and an empty chat history `H_0`.

6. **Execute the Thought-Action-Observation cycle.** For each step `t` (up to `T_max = 5`):
   - **Thought**: Reason about what information is still missing to verify the claim given current evidence.
   - **Action**: Select a retrieval tool (web search, Wikipedia lookup, scholarly search) with specific query parameters, OR select the `Answer` action to terminate with a verdict.
   - **Observation**: Execute the tool and capture its output. Append `(thought_t, action_t, observation_t)` to the chat history `H_t`.

7. **Generate the verdict and rationale.** When the `Answer` action is selected (or `T_max` is reached), produce a structured output: the veracity label (e.g., True / False / Partially True / Unverifiable), a confidence score, and the full reasoning trace as a human-readable rationale.

8. **Validate and commit new evidence to memory.** Review all evidence gathered during the ReAct loop. Filter out irrelevant or low-quality observations. Index valid evidence by the associated entities and store it in the persistent memory for future claims.

9. **Process the next claim.** Move to the next claim in the batch, repeating from step 2. The memory now contains evidence from all previously processed claims, enabling cross-claim reuse.

10. **Return the consolidated results.** Aggregate verdicts across all claims with their rationales, and optionally report memory statistics (cache hits, total searches avoided).

## Concrete Examples

**Example 1: Single claim verification**

```
User: Fact-check this claim: "Tesla's Cybertruck was the best-selling electric
pickup truck in the US in 2025."

Approach:
1. Decompose: (Tesla Cybertruck, was, best-selling electric pickup truck,
   {market: US, year: 2025})
   Keywords: ["Cybertruck", "electric pickup truck", "US sales", "2025"]

2. Memory recall: No prior evidence (first claim). M_c = {}

3. ReAct loop:
   Step 1:
     Thought: I need 2025 US electric pickup truck sales data. Let me search
              for Cybertruck sales figures and competing models.
     Action: web_search("Cybertruck US sales 2025 electric pickup truck")
     Observation: [Retrieved sales data showing Cybertruck units sold]

   Step 2:
     Thought: I have Cybertruck numbers but need competitor data (Ford F-150
              Lightning, Rivian R1T, Chevy Silverado EV) to confirm "best-selling."
     Action: web_search("electric pickup truck US sales 2025 F-150 Lightning
              Rivian R1T Silverado EV")
     Observation: [Retrieved competitor sales figures]

   Step 3:
     Thought: I now have Cybertruck and competitor sales. Cybertruck led in
              total units. The claim is supported.
     Action: Answer

4. Verdict: TRUE
   Rationale: Based on 2025 US sales data, the Cybertruck sold [X] units,
   exceeding the F-150 Lightning ([Y] units) and Rivian R1T ([Z] units).

5. Memory update: Store sales evidence under entities
   ["Tesla Cybertruck", "F-150 Lightning", "Rivian R1T"]
```

**Example 2: Batch claims with cross-claim memory reuse**

```
User: Check these three claims from an article:
  1. "CRISPR was invented by Jennifer Doudna and Emmanuelle Charpentier."
  2. "Doudna and Charpentier received the 2020 Nobel Prize in Chemistry."
  3. "CRISPR has been approved by the FDA for treating sickle cell disease."

Approach:
Claim 1:
  Decompose: (CRISPR, invented by, Jennifer Doudna AND Emmanuelle Charpentier)
  Memory recall: Empty. Start fresh.
  ReAct: Search Wikipedia for CRISPR history -> find that Doudna and
    Charpentier developed the CRISPR-Cas9 gene editing method (2012), but
    "invented" oversimplifies -- earlier work by Mojica, Zhang, etc. contributed.
  Verdict: PARTIALLY TRUE
  Memory update: Store CRISPR history, Doudna bio, Charpentier bio.

Claim 2:
  Decompose: (Doudna AND Charpentier, received, 2020 Nobel Prize in Chemistry)
  Memory recall: HIT -- Doudna and Charpentier evidence already stored from
    Claim 1. The recalled evidence already mentions Nobel Prize context.
  ReAct: Memory provides substantial context. One confirmatory search suffices.
    Step 1: web_search("2020 Nobel Prize Chemistry") -> confirms award.
    Step 2: Answer.
  Verdict: TRUE
  Searches saved: ~2 tool calls avoided via memory warm start.

Claim 3:
  Decompose: (CRISPR, approved by, FDA, {application: treating sickle cell})
  Memory recall: HIT on "CRISPR" -- general CRISPR evidence from Claim 1.
  ReAct: Need specific FDA approval data.
    Step 1: web_search("FDA CRISPR sickle cell disease approval")
    -> Finds Casgevy (exagamglogene autotemcel) FDA-approved Dec 2023.
    Step 2: Answer.
  Verdict: TRUE (with nuance: the approved therapy is Casgevy, a CRISPR-based
    treatment, not "CRISPR" as a generic tool)
```

**Example 3: Implementing a fact-checking pipeline in code**

```
User: Build me a Python fact-checking module using the MERMAID pattern.

Approach:
1. Create a ClaimDecomposer class that uses an LLM to extract triplets
   and keywords from raw claim text.
2. Create an EvidenceMemory class with a dict-based key-value store indexed
   by entity strings, supporting recall(entities) and update(entities, evidence).
3. Create an Executor class implementing the ReAct loop with configurable
   tools (web search, Wikipedia API, etc.) and T_max parameter.
4. Create a MermaidPipeline class that orchestrates:
   decompose -> memory_recall -> execute_react_loop -> generate_verdict ->
   memory_update
5. Support batch processing with memory persisting across claims.

Output structure:
@dataclass
class VerificationResult:
    claim: str
    verdict: str           # TRUE / FALSE / PARTIALLY_TRUE / UNVERIFIABLE
    confidence: float
    rationale: str         # Full reasoning trace
    evidence: list[str]    # Key evidence snippets
    search_count: int      # Number of tool calls used
    memory_hits: int       # Evidence items recalled from memory
```

## Best Practices

- **Do:** Process related claims in sequence so the memory module accumulates shared evidence. Order claims by topical similarity when possible to maximize cache hits.
- **Do:** Set `T_max` to 5 for general claims. Increase to 7-8 for claims requiring multi-hop reasoning (e.g., "X happened because of Y, which was caused by Z").
- **Do:** Include the full ReAct trajectory in your rationale output. Transparency in the reasoning chain is what distinguishes this approach from opaque classifiers.
- **Do:** Validate evidence quality before committing to memory. Discard tool outputs that returned errors, empty results, or clearly irrelevant content.
- **Avoid:** Treating the decomposition as optional. Structured triplets are essential for both targeted retrieval queries and entity-based memory indexing.
- **Avoid:** Overfilling the initial prompt with recalled memory. If `M_c` exceeds context limits, prioritize evidence most relevant to the current claim's triplets and discard tangential entries.

## Error Handling

- **Decomposition produces malformed triplets:** Fall back to keyword extraction from the raw claim and proceed with keyword-based memory lookup and search queries. Log the decomposition failure for review.
- **All retrieval tools return empty results:** After exhausting retries, mark the claim as `UNVERIFIABLE` with a rationale explaining that no supporting or refuting evidence could be found. Do not guess.
- **Memory recall returns excessive or irrelevant evidence:** The current entity-matching approach uses string-based keywords, not semantic similarity. If recalled evidence exceeds a reasonable threshold (e.g., >10 items), rank by recency and relevance to the claim's topical keywords, then truncate.
- **Executor reaches `T_max` without selecting `Answer`:** Force verdict generation from accumulated evidence in the chat history. Flag the result with lower confidence and note that the reasoning loop was truncated.
- **Conflicting evidence retrieved:** Report the conflict explicitly in the rationale. Assess source reliability (e.g., primary sources over secondary, recent over outdated) and produce a nuanced verdict (PARTIALLY_TRUE or include caveats).

## Limitations

- **String-based memory matching:** The entity-indexed memory uses keyword matching, not semantic similarity. Paraphrased entities (e.g., "US" vs. "United States") will miss cache hits. Normalize entities where possible.
- **No temporal reasoning:** The memory does not track evidence freshness. Stale evidence about rapidly changing facts (stock prices, election results) may produce incorrect verdicts if not re-verified.
- **Single-claim decomposition granularity:** Very complex claims with nested conditionals or multiple temporal scopes may not decompose cleanly into flat triplets.
- **Tool dependency:** Verdict quality depends directly on the availability and accuracy of search tools. Claims about niche or non-English topics may suffer from retrieval gaps.
- **Not suitable for opinion or subjective claims:** The framework is designed for factual veracity, not for assessing opinions, predictions, or value judgments.

## Reference

[MERMAID: Memory-Enhanced Retrieval and Reasoning with Multi-Agent Iterative Knowledge Grounding for Veracity Assessment](https://arxiv.org/abs/2601.22361v1) -- Cao et al., 2026. Focus on Section 3 (framework architecture), Figure 2 (system diagram), and Section 4.4 (memory ablation study showing 16-30% search reduction).