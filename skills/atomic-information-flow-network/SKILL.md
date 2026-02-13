---
name: "atomic-information-flow-network"
description: >
  Trace and attribute RAG system responses back to specific tools and sources using Atomic Information Flow (AIF) --
  a network flow model that decomposes outputs into atoms and computes precise attribution scores.
  Use when: "trace which tools contributed to this RAG response", "attribute this answer to its sources",
  "debug why my RAG pipeline returned wrong information", "compress RAG context without losing accuracy",
  "build attribution into my multi-agent system", "score tool contribution in my retrieval pipeline".
---

# Atomic Information Flow: Network Flow Attribution for RAG Systems

This skill enables Claude to apply **Atomic Information Flow (AIF)** -- a graph-based network flow model from Gao et al. (2026) -- to decompose RAG system outputs into minimal semantic units called *atoms*, construct a directed flow graph tracing information from tool/source nodes through LLM processing to the final response, and compute precise attribution scores. This lets you debug RAG pipelines, identify which tools or documents actually contributed to an answer, detect hallucinated content (atoms with no source), and compress context by cutting low-contribution tools via minimum-cut analysis.

## When to Use

- When the user asks to **trace which documents or tools contributed** to a RAG-generated answer
- When building a **multi-agent or multi-tool RAG pipeline** and needing attribution transparency
- When debugging **incorrect RAG responses** by identifying which source introduced the error
- When the user wants to **compress retrieval context** by removing low-contribution documents while preserving answer quality
- When implementing **groundedness scoring** to measure what fraction of a response is backed by retrieved sources
- When designing a **tool orchestration system** that needs auditable information provenance
- When the user asks to detect **hallucinated content** in a RAG response (atoms not traceable to any tool)

## Key Technique

AIF models a RAG pipeline as a **directed graph** `G = (V, E)` where nodes represent orchestration components (a super-source for the query, tool/document nodes, LLM processing nodes, and a super-sink for the final response) and edges represent causal information flow. The fundamental unit is the **atom** -- a minimal, self-contained snippet of semantic information that cannot be further decomposed without losing meaning. Tool outputs and LLM responses are each decomposed into multisets of atoms. Flow through the graph assigns atom counts to edges, subject to a relaxed conservation law: incoming flow plus any new atoms generated at a node must be greater than or equal to outgoing flow, with slack representing atoms the LLM filtered out.

The power of this framing is that **attribution becomes a flow measurement problem**. For each atom in the final response, you trace it back through the graph to its source tool(s). This yields four core metrics: **Groundedness (RAP)** -- fraction of response atoms sourced from tools; **Tool Consumption (RAR)** -- fraction of available tool atoms that actually reached the response; **Tool Contribution (TUP_t)** -- what fraction of the response came from tool `t`; and **Tool Usage (TUR_t)** -- what fraction of tool `t`'s atoms were consumed. A response with low RAP likely contains hallucinations. A tool with high TUP_t is a critical source. A tool with zero TUP_t can be cut from context.

For context compression, AIF applies the **minimum-cut principle**: define each tool node's capacity proportional to its probability of being utilized, then find the minimum cut that separates the query source from the response sink. Tools on the cut side can be safely removed. In practice, a fine-tuned small model (e.g., Gemma3-4B) learns this compression policy from AIF-generated labels, achieving 87.5% token reduction while maintaining 82.7% accuracy on HotPotQA.

## Step-by-Step Workflow

1. **Map the RAG pipeline into a directed graph.** Identify every component: the user query (super-source `s_0`), each retrieval tool or document (`V_tool`), each LLM call (`V_llm`), and the final response (super-sink `t_0`). Draw directed edges following the actual data flow -- e.g., `s_0 -> retriever -> doc_1, doc_2, doc_3 -> LLM_synthesizer -> t_0`.

2. **Decompose tool outputs into atoms.** For each tool node, break its output into minimal self-contained information units. Use structured prompting: instruct the LLM to "decompose the following text into atomic facts -- each atom should be a single claim or datum that stands alone." Apply map-reduce if outputs exceed context windows: chunk, decompose in parallel, then merge and deduplicate.

3. **Decompose the final response into atoms.** Apply the same atomic decomposition to the response text at the super-sink. Each response atom `r_j` is a single claim or fact in the answer.

4. **Match response atoms to source atoms.** For each response atom `r_j`, search the flattened pool of all tool atoms `U_flat` for semantic matches using an LLM matcher or embedding similarity. Record the mapping `r_j -> {source_tool_id, source_atom_id}`. An atom with no match is flagged as **unsourced** (potential hallucination or LLM-generated reasoning).

5. **Compute attribution metrics.** Using the atom mappings, calculate:
   - `RAP = |matched_response_atoms| / |total_response_atoms|` (groundedness)
   - `RAR = |matched_response_atoms| / |total_tool_atoms|` (tool consumption)
   - `TUP_t = |response_atoms_from_tool_t| / |total_response_atoms|` (per-tool contribution)
   - `TUR_t = |response_atoms_from_tool_t| / |atoms_in_tool_t|` (per-tool usage rate)

6. **Optionally inject relevance signals.** Score each atom with metadata like relevance-to-query, freshness, or authority. Compute weighted variants of the metrics (e.g., `RAP@K` filtering to atoms above relevance threshold `K`). Note: the paper found unweighted `RAP` (K=0) was actually the strongest predictor of answer correctness.

7. **Identify the minimum cut for context compression.** Label each tool node as "retained" if `TUP_t > 0` (it contributed atoms to the response) or "pruned" if `TUP_t = 0`. These labels form training data for a lightweight compression model that predicts, given a query and tool set, which tools to keep.

8. **Build or apply the compression policy.** Fine-tune a small model on the binary labels from step 7 (tool retained vs. pruned per query). At inference time, this model predicts which tools to keep, and the rest are masked out before the final LLM call -- dramatically reducing token count.

9. **Surface attribution results.** Present the flow graph, per-tool contribution scores, and any unsourced atoms to the user or downstream system. Use this for debugging, auditing, or as feedback signal for retrieval tuning.

## Concrete Examples

**Example 1: Debugging a wrong answer in a multi-document QA pipeline**

```
User: My RAG system answered "The Eiffel Tower was built in 1901" but the
correct answer is 1889. I have 5 retrieved documents. Which one caused the error?

Approach:
1. Decompose each of the 5 document outputs into atoms:
   - Doc 1 atoms: ["Eiffel Tower is in Paris", "Located on Champ de Mars"]
   - Doc 2 atoms: ["Tower completed in 1889", "Designed by Gustave Eiffel"]
   - Doc 3 atoms: ["1901 World's Fair held in Buffalo NY", "Pan-American Exposition"]
   - Doc 4 atoms: ["Eiffel Tower height is 330m", "Made of iron"]
   - Doc 5 atoms: ["Paris hosted 1889 World's Fair", "Tower built for the exposition"]

2. Decompose the response into atoms:
   - R1: "The Eiffel Tower was built in 1901"

3. Match R1 against source atoms:
   - Semantic match: R1 partially matches Doc 3's "1901 World's Fair" (year 1901)
     and Doc 2/5's Eiffel Tower construction context
   - The LLM conflated the 1901 date from Doc 3 with the construction topic

4. Compute attribution:
   - TUP_doc3 = 1.0 (the erroneous date came from Doc 3)
   - RAP = 0.5 (partially grounded -- "Eiffel Tower" is sourced, but "1901" is misattributed)

Output:
  Attribution Report:
  - Response atom "built in 1901" traces to Doc 3 (1901 World's Fair)
  - The LLM incorrectly merged the year from Doc 3 with construction context from Docs 2/5
  - Fix: Improve retrieval precision or add a re-ranking step to filter Doc 3
    as irrelevant to the construction date question
```

**Example 2: Compressing context in a 10-tool agent pipeline**

```
User: My agent calls 10 tools (web search, calculator, calendar, weather, etc.)
but the responses are slow due to large context. Help me identify which tools
I can drop.

Approach:
1. Collect a sample of 50 representative queries and their tool outputs
2. For each query, decompose all tool outputs and the final response into atoms
3. Match response atoms to tool atoms across all 50 queries
4. Compute average TUP_t for each tool across the sample:

   Tool             Avg TUP_t   Avg TUR_t   Queries Contributing
   web_search       0.45        0.12        48/50
   calculator       0.22        0.85        20/50
   calendar         0.15        0.60        15/50
   weather          0.08        0.30        8/50
   stock_ticker     0.05        0.20        5/50
   dictionary       0.03        0.90        3/50
   maps             0.01        0.05        2/50
   translator       0.01        0.10        1/50
   unit_converter   0.00        0.00        0/50
   timezone         0.00        0.00        0/50

5. Apply minimum-cut analysis: tools with TUP_t = 0 are safe to always prune.
   Tools with TUP_t < 0.03 are candidates for conditional pruning.

Output:
  Compression Recommendation:
  - Always remove: unit_converter, timezone (zero contribution across all queries)
  - Conditionally remove: maps, translator, dictionary (< 3% contribution)
  - Always keep: web_search, calculator, calendar (> 15% contribution)
  - Expected context reduction: ~40% tokens with < 2% accuracy impact
```

**Example 3: Building attribution into a new RAG system**

```
User: I'm building a RAG system for legal document Q&A. I need to cite which
specific documents support each claim in the response.

Approach:
1. Define the flow graph:
   s_0 (query) -> retriever -> [doc_1, doc_2, ..., doc_n] -> LLM_synthesizer -> t_0

2. After generating the response, decompose into atoms:
   Response: "Under Section 12(a), sellers are strictly liable for
   misstatements in a registration statement (Smith v. Jones, 2019).
   The statute of limitations is one year from discovery."

   Atoms:
   - R1: "Under Section 12(a), sellers are strictly liable for misstatements
          in a registration statement"
   - R2: "Smith v. Jones (2019) supports this"
   - R3: "The statute of limitations is one year from discovery"

3. Match each atom to source documents:
   - R1 -> Doc 4 (Securities Act text), confidence: 0.95
   - R2 -> Doc 7 (case law digest), confidence: 0.88
   - R3 -> Doc 4 (Securities Act text), confidence: 0.91

4. Generate inline citations:
   "Under Section 12(a), sellers are strictly liable for misstatements
   in a registration statement [Doc 4] (Smith v. Jones, 2019 [Doc 7]).
   The statute of limitations is one year from discovery [Doc 4]."

   Groundedness (RAP): 1.0 -- all atoms are sourced
   Unsourced atoms: none
```

## Best Practices

- **Do:** Decompose at the right granularity -- atoms should be single claims or facts, not sentences or paragraphs. A good test: if removing any word changes the meaning, it's atomic.
- **Do:** Use the flattened unordered pool for matching (not per-tool sequential matching) to avoid order bias when attributing response atoms to sources.
- **Do:** Track multi-hop attribution -- a response atom may trace through multiple LLM nodes back to multiple tools. Record the full path, not just the final source.
- **Do:** Use `RAP` (groundedness at K=0) as the primary quality signal. The paper found unweighted groundedness was a stronger predictor of answer correctness than relevance-weighted variants.
- **Avoid:** Treating atoms as fungible across tools. Two tools may produce atoms with identical text but different provenance -- track source IDs, not just content.
- **Avoid:** Over-relying on high relevance thresholds (K) for attribution filtering. The paper paradoxically found that high K actually weakened correlation with answer correctness.
- **Avoid:** Running atomic decomposition on very large outputs without chunking first. Use map-reduce: chunk the output, decompose each chunk in parallel, then merge and deduplicate atoms.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Response atom matches multiple tools equally | Ambiguous provenance | Record all matching sources with confidence scores; flag for human review if attribution must be unique |
| Very low RAP score (< 0.3) | Heavy hallucination or LLM reasoning steps | Distinguish between unsourced atoms that are hallucinated vs. legitimate LLM inference chains; consider adding reasoning nodes to the graph |
| Atom decomposition produces too many fine-grained atoms | Decomposer is over-splitting | Add a merging pass that combines atoms sharing the same subject-predicate structure; validate with human spot-checks |
| Tool with high TUR_t but low TUP_t | Tool produces many atoms but few reach the response | Tool output may be partially relevant; consider re-ranking or filtering tool output before feeding to the LLM |
| Compression model prunes a tool that was needed | Training data insufficient for that query type | Increase training sample size for underrepresented query categories; add a confidence threshold below which tools are kept by default |

## Limitations

- **Decomposition cost:** Atomic decomposition requires LLM calls for every tool output and every response. For real-time systems, consider pre-computing tool atom libraries or using a fine-tuned lightweight decomposer.
- **Semantic matching is imperfect:** Atom-to-atom matching relies on LLM or embedding similarity, which can miss paraphrased or implicit information flow. Attribution precision degrades when the LLM heavily rephrases source content.
- **No retrieval-side modeling:** AIF as published models only the Tool-to-Response flow. The Query-to-Tool edge (retrieval relevance) is not yet covered -- you must handle retrieval quality separately.
- **Multi-hop chains are hard:** When information passes through multiple LLM nodes (e.g., agent chains), atoms may be transformed at each hop. The relaxed flow conservation allows slack but doesn't precisely model semantic transformation.
- **Integer Multicommodity Flow is NP-hard:** The exact flow optimization is computationally intractable for large graphs. The practical approach uses heuristic matching rather than solving the full optimization.
- **Best validated on factoid QA:** The paper's evaluation focused on datasets like HotPotQA and MuSiQue. Attribution quality on open-ended generation, creative tasks, or code generation is unvalidated.

## Reference

**Paper:** Gao, Zhou, Sun, Huang, Yoo. "Atomic Information Flow: A Network Flow Model for Tool Attributions in RAG Systems." arXiv:2602.04912v1, 2026.
**Link:** https://arxiv.org/abs/2602.04912v1
**Key takeaway:** Look for Algorithm 1 (atom extraction), Algorithm 3 (response atom assignment), Table 1 (the four flow heuristic metrics), and Section 7 (minimum-cut compression) -- these are the directly implementable components.