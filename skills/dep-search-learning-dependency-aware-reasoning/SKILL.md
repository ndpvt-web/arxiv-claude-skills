---
name: "dep-search-learning-dependency-aware-reasoning"
description: "Dependency-aware multi-step reasoning with persistent memory for complex questions requiring information retrieval across multiple sources. Use when: 'answer this multi-hop question', 'research this topic step by step', 'find information that depends on other lookups', 'break down this complex question', 'trace the reasoning chain for this query', 'search and synthesize across multiple sources'."
---

# Dep-Search: Dependency-Aware Reasoning with Persistent Memory

This skill enables Claude to tackle complex multi-hop questions by decomposing them into dependency-aware sub-questions organized as a directed acyclic graph (DAG), retrieving information in topological order, and storing intermediate findings in a persistent memory buffer for reuse. Based on the Dep-Search framework (Liu et al., 2026), this approach replaces implicit "chain of thought" reasoning with explicit structured operations: **Decompose**, **Retrieve**, **Memory Access**, and **Conclude**. The result is more reliable answers to questions where the answer to one sub-question is a prerequisite for formulating the next.

## When to Use

- When a user asks a question requiring 2+ retrieval steps where later searches depend on earlier results (e.g., "What university did the director of Inception attend?")
- When building a research agent or search pipeline that must track dependencies between information needs
- When answering questions that require synthesizing facts from multiple documents or APIs
- When the user asks to "break down" or "decompose" a complex question into sub-questions
- When implementing a retrieval-augmented generation (RAG) system that must handle multi-hop reasoning
- When prior retrieved context is too long and needs to be summarized into reusable facts before proceeding

## Key Technique

**Dependency-Aware Decomposition.** Instead of generating sub-questions sequentially (where each implicitly depends on the previous), Dep-Search decomposes the original question into K sub-questions forming a DAG. Each sub-question explicitly declares which prior sub-questions it depends on. For example, "What is the population of the capital of France?" decomposes into: (1) "What is the capital of France?" and (2) "What is the population of [result of step 1]?" -- where step 2 explicitly depends on step 1. Sub-questions are then resolved in topological order, guaranteeing prerequisites are satisfied before dependent steps execute.

**Persistent Memory Buffer.** Dep-Search maintains an LRU (least-recently-used) memory buffer (capacity ~20 entries) that accumulates compressed facts across reasoning steps. After each retrieval, a **Conclude** operation summarizes the retrieved documents into short, reusable fact sentences stored in memory. Later steps can access these facts via embedding similarity search combined with recency weighting, avoiding redundant re-retrieval. This is critical when context windows would otherwise overflow with raw retrieved documents.

**Reward-Driven Optimization.** The framework uses GRPO (Group Relative Policy Optimization) with a composite reward: R = R_answer - 0.1 * R_retrieval - 0.05 * R_decomposition. This penalizes excessive retrieval calls (>10) and over-decomposition (>8 sub-questions), training the model to be both accurate and efficient. For application without RL training, the structured format itself provides the key benefit -- you can implement the decomposition and memory pattern directly.

## Step-by-Step Workflow

1. **Analyze the question for multi-hop structure.** Determine whether the question requires information that can only be found by first resolving a prerequisite question. Single-hop questions don't need this framework -- just answer directly.

2. **Decompose into sub-questions with explicit dependencies.** Write out each sub-question and annotate which prior steps it depends on. Use the format: `Sub-Q K (depends on: step N, step M): [question text]`. Aim for the minimum number of sub-questions needed -- penalize over-decomposition.

3. **Build the dependency DAG and determine topological order.** Identify which sub-questions have no dependencies (roots) and which depend on others. Resolve root questions first, then proceed to dependent questions only after their prerequisites are resolved.

4. **For each sub-question in topological order, check memory first.** Before performing any new retrieval, check whether previously stored conclusions already answer the current sub-question. If a stored fact resolves the sub-question, use it directly and skip retrieval.

5. **Retrieve information for unresolved sub-questions.** Issue a targeted search query for the current sub-question, incorporating resolved dependency values. For example, if step 1 resolved "capital of France = Paris", step 2's query becomes "population of Paris" rather than "population of capital of France".

6. **Substitute resolved dependencies into query templates.** Replace placeholder references in dependent sub-questions with actual resolved values before querying. This produces more specific, effective searches.

7. **Conclude: compress retrieved context into reusable facts.** After each successful retrieval, extract 1-3 concise fact sentences from the retrieved documents and store them in the memory buffer. Evict oldest entries if buffer exceeds capacity.

8. **Repeat steps 4-7 for each sub-question in topological order.** Track which sub-questions are resolved and which are still pending. If a retrieval fails, note the failure in memory to avoid repeating the same failed strategy.

9. **Synthesize the final answer from memory.** Once all sub-questions are resolved, combine the accumulated facts from memory into a coherent answer to the original question. Cite which sub-question each fact came from.

10. **Validate dependency satisfaction.** Before finalizing, verify that every sub-question that had dependencies actually used the resolved values from those dependencies. Flag any gaps in the reasoning chain.

## Concrete Examples

**Example 1: Multi-hop factual question**

User: "What is the birth year of the person who directed the first Marvel Cinematic Universe film?"

Approach:
```
Decompose:
  Sub-Q 1 (depends on: none): What was the first Marvel Cinematic Universe film?
  Sub-Q 2 (depends on: step 1): Who directed [result of step 1]?
  Sub-Q 3 (depends on: step 2): What is the birth year of [result of step 2]?

Topological order: 1 -> 2 -> 3

Step 1: Retrieve "first Marvel Cinematic Universe film"
  -> Retrieved: Iron Man (2008)
  -> Conclude & store: "The first MCU film was Iron Man, released in 2008."

Step 2: Check memory -> no info on director. Retrieve "director of Iron Man 2008"
  -> Retrieved: Jon Favreau directed Iron Man.
  -> Conclude & store: "Jon Favreau directed Iron Man (2008)."

Step 3: Check memory -> no birth year. Retrieve "Jon Favreau birth year"
  -> Retrieved: Jon Favreau was born October 19, 1966.
  -> Conclude & store: "Jon Favreau was born in 1966."

Final answer: 1966. Jon Favreau, who directed Iron Man (the first MCU film),
was born in 1966.
```

**Example 2: Research task with shared dependencies**

User: "Compare the GDP per capita of the countries where Tesla and Toyota are headquartered."

Approach:
```
Decompose:
  Sub-Q 1 (depends on: none): Where is Tesla headquartered?
  Sub-Q 2 (depends on: none): Where is Toyota headquartered?
  Sub-Q 3 (depends on: step 1): What is the GDP per capita of [result of step 1]?
  Sub-Q 4 (depends on: step 2): What is the GDP per capita of [result of step 2]?
  Sub-Q 5 (depends on: step 3, step 4): Compare [result of step 3] and [result of step 4].

Topological order: {1, 2} (parallel) -> {3, 4} (parallel) -> 5

Steps 1-2 (no dependencies, resolve in parallel):
  Step 1: Retrieve -> Tesla is headquartered in Austin, Texas, USA.
    Conclude & store: "Tesla HQ: Austin, Texas, USA."
  Step 2: Retrieve -> Toyota is headquartered in Toyota City, Japan.
    Conclude & store: "Toyota HQ: Toyota City, Japan."

Steps 3-4 (depend on 1 and 2 respectively):
  Step 3: Retrieve "GDP per capita of USA"
    -> Conclude & store: "USA GDP per capita: ~$85,000 (2025)."
  Step 4: Retrieve "GDP per capita of Japan"
    -> Conclude & store: "Japan GDP per capita: ~$34,000 (2025)."

Step 5: Synthesize from memory -- all 4 facts available.
  Final answer: Tesla (USA) has a GDP per capita of ~$85,000 vs Toyota
  (Japan) at ~$34,000. The USA's GDP per capita is roughly 2.5x Japan's.
```

**Example 3: Building a search agent with dependency tracking**

User: "Build me a function that answers multi-hop questions using this dependency-aware approach."

Approach:
```python
from dataclasses import dataclass, field
from collections import deque

@dataclass
class MemoryEntry:
    fact: str
    step_id: int
    query: str

class DepSearchReasoner:
    def __init__(self, search_fn, max_memory=20):
        self.search_fn = search_fn  # callable: query -> list[str]
        self.memory: deque[MemoryEntry] = deque(maxlen=max_memory)
        self.resolved: dict[int, str] = {}

    def decompose(self, question: str) -> list[dict]:
        """Returns list of {id, question_template, depends_on: list[int]}."""
        # LLM call to decompose question into sub-questions with dependencies
        ...

    def topological_order(self, sub_questions: list[dict]) -> list[int]:
        """Return sub-question IDs in dependency-respecting order."""
        from graphlib import TopologicalSorter
        graph = {sq["id"]: set(sq["depends_on"]) for sq in sub_questions}
        return list(TopologicalSorter(graph).static_order())

    def substitute_deps(self, template: str, depends_on: list[int]) -> str:
        """Replace [step N] placeholders with resolved values."""
        query = template
        for dep_id in depends_on:
            query = query.replace(f"[step {dep_id}]", self.resolved[dep_id])
        return query

    def check_memory(self, query: str) -> str | None:
        """Search memory for a fact that answers the query."""
        # Embedding similarity search over self.memory entries
        ...

    def conclude(self, step_id: int, query: str, docs: list[str]) -> str:
        """Summarize retrieved docs into a reusable fact sentence."""
        # LLM call to extract key fact from docs
        fact = ...
        self.memory.append(MemoryEntry(fact=fact, step_id=step_id, query=query))
        return fact

    def answer(self, question: str) -> str:
        sub_qs = self.decompose(question)
        order = self.topological_order(sub_qs)
        sq_map = {sq["id"]: sq for sq in sub_qs}

        for step_id in order:
            sq = sq_map[step_id]
            query = self.substitute_deps(sq["question_template"], sq["depends_on"])

            cached = self.check_memory(query)
            if cached:
                self.resolved[step_id] = cached
                continue

            docs = self.search_fn(query)
            fact = self.conclude(step_id, query, docs)
            self.resolved[step_id] = fact

        return self.synthesize(question, self.resolved)
```

## Best Practices

- **Do:** Annotate dependencies explicitly -- write `(depends on: step N)` for every sub-question. Implicit dependencies are the primary failure mode of naive chain-of-thought decomposition.
- **Do:** Check memory before every retrieval. Redundant searches waste tokens and may introduce contradictory information from different sources.
- **Do:** Keep concluded facts atomic -- one fact per sentence. "Paris is the capital of France" is better than a paragraph about French geography. Atomic facts are easier to match during memory lookup.
- **Do:** Substitute resolved values into dependent queries before searching. Searching "population of the capital of France" is strictly worse than "population of Paris".
- **Avoid:** Over-decomposing simple questions. If a question can be answered in 1-2 steps, don't force a 6-step DAG. The Dep-Search reward function penalizes decomposition beyond 8 sub-questions for good reason.
- **Avoid:** Storing raw retrieved documents in memory. Always compress through the Conclude step. Raw documents consume memory capacity without the precision of extracted facts.

## Error Handling

- **Retrieval returns no results:** Record the failed query in memory (to avoid retrying it), reformulate the query using different terms, and retry. If a sub-question cannot be resolved after 2 attempts, note the gap explicitly in the final answer.
- **Circular dependencies detected:** If the dependency graph contains cycles, the decomposition is invalid. Re-decompose the question, ensuring the DAG is acyclic. Use topological sort validation before proceeding.
- **Memory buffer overflow:** When the LRU buffer is full and a new fact must be stored, the oldest entry is evicted. If evicted facts are still needed, they will require re-retrieval. For very long reasoning chains (>20 steps), consider increasing buffer capacity.
- **Dependency resolution produces ambiguous value:** When step N resolves to multiple possible values (e.g., a person with multiple roles), propagate all candidates and branch the reasoning, or ask the user to disambiguate.
- **Contradictory facts in memory:** When two stored facts conflict, flag the contradiction, retrieve a third source to arbitrate, and store the corrected fact with a note about the discrepancy.

## Limitations

- **Single-hop questions gain nothing.** This framework adds overhead for questions that can be answered with one retrieval step. Only apply it when genuine multi-hop dependencies exist.
- **Decomposition quality is the bottleneck.** If sub-questions are poorly formulated or dependencies are misidentified, the entire reasoning chain degrades. The DAG is only as good as the decomposition step.
- **No native parallel execution in sequential contexts.** While the DAG may show independent sub-questions that could be resolved in parallel, sequential execution environments process them one at a time. The dependency structure still helps by avoiding unnecessary ordering constraints.
- **Memory capacity is finite.** For extremely long research tasks with dozens of sub-questions, the 20-entry LRU buffer will evict early facts. Adjust capacity based on task complexity.
- **Assumes retrievable knowledge.** If the answer requires reasoning, computation, or knowledge not present in any searchable source, the retrieve-and-conclude loop will not help.

## Reference

Liu, Y., Peng, X., Yan, Z., Shen, Y., & Xu, W. (2026). *Dep-Search: Learning Dependency-Aware Reasoning Traces with Persistent Memory*. arXiv:2601.18771v1. [https://arxiv.org/abs/2601.18771v1](https://arxiv.org/abs/2601.18771v1)

Key sections to read: Section 3 (framework architecture with Decompose/Retrieve/Memory/Conclude operations), Section 3.3 (persistent memory buffer with LRU eviction), Section 4 (GRPO training with composite reward R = R_ans - 0.1*R_ret - 0.05*R_dec), and Table 2 (results across 7 QA benchmarks showing ~3-point average improvement over HierSearch).