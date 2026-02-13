---
name: "logicscore-fine-grained-logic-evaluation"
description: "Evaluate the logical integrity of LLM-generated multi-hop answers using Horn Rule backward chaining. Scores Completeness (gap-free reasoning), Conciseness (no redundant steps), and Determinateness (answer entailment). Use when: 'evaluate my QA pipeline logic', 'check reasoning chain completeness', 'score answer conciseness', 'find deductive gaps in generated answers', 'audit multi-hop reasoning quality', 'LogicScore evaluation'."
---

# LogicScore: Fine-grained Logic Evaluation of Multi-hop Answers

This skill enables Claude to evaluate the **global logical integrity** of long-form, attributed question-answering outputs using the LogicScore framework. Instead of merely checking whether individual claims are supported by sources (local attribution), LogicScore decomposes answers into atomic propositions structured as Horn Rules, then applies backward chaining to measure three dimensions: whether the reasoning chain is **complete** (no deductive gaps), **concise** (no redundant propositions), and **determinate** (the long-form answer uniquely entails the short answer). This directly addresses "attribution myopia" -- the phenomenon where LLMs produce factually grounded yet logically incoherent responses.

## When to Use

- When the user asks to evaluate a multi-hop QA system's reasoning quality beyond simple factual accuracy
- When auditing whether an LLM-generated answer contains deductive gaps, circular reasoning, or broken logic chains
- When building or benchmarking retrieval-augmented generation (RAG) pipelines and needing logic-level metrics
- When the user wants to score the conciseness of chain-of-thought or step-by-step answers (ratio of useful vs. total reasoning steps)
- When checking if a long-form answer deterministically implies the final short answer without relying on external knowledge
- When diagnosing why an LLM answer "looks correct" but has flawed reasoning (high attribution score, low logic score)

## Key Technique

LogicScore reformulates answer evaluation as a **Horn Rule verification problem**. Given a question Q, a short answer SA, and a long-form answer LA, the framework first decomposes LA into atomic propositions P = {P1, P2, ..., Pn}. Each proposition is parsed into a (subject, relation, object) triple. The short answer SA is treated as the consequent of a Horn clause: `SA <- P1 AND P2 AND ... AND Pn`. The key insight is that valid multi-hop reasoning forms a **chain of entity bridges** -- each proposition's object becomes the next proposition's subject, connecting the question entity to the answer entity.

The **backward verification algorithm** starts from propositions containing the gold answer entity and recursively searches backward through the remaining propositions by matching bridge entities, attempting to reach the question entity. If it succeeds, it identifies the **minimal sufficient set** (Pmin) -- the smallest subset of propositions forming a complete deductive path. This directly yields all three metrics: Completeness is binary (did backward chaining reach the question entity?), Conciseness is |Pmin|/|P| (what fraction of propositions were actually needed?), and Determinateness tests whether the long-form answer alone -- with no external knowledge -- uniquely entails the short answer.

This approach catches three specific failure modes that traditional attribution metrics miss: **circular reasoning** (chains that loop back to their premises), **broken chains** (missing entity bridges between sequential hops), and **deviated reasoning** (retrieved information that fails to synthesize across documents into a coherent path).

## Step-by-Step Workflow

1. **Extract the QA components.** Parse the input into three elements: the question Q (with its query entity), the short/gold answer SA, and the long-form answer LA (the full reasoning text to evaluate).

2. **Decompose LA into atomic propositions.** Break the long-form answer into the smallest meaningful factual claims P = {P1, P2, ..., Pn}. Each proposition should express a single relationship. For attributed answers, preserve citation markers linking each proposition to its source document.

3. **Parse propositions into triples.** Convert each Pi into a structured (subject, relation, object) triple. Identify the key entities in each proposition -- these will serve as potential bridge nodes in the reasoning chain.

4. **Partition propositions.** Split P into two sets: Pu (propositions containing the gold answer entity) and Pc (all remaining contextual propositions). Pu serves as the starting point for backward chaining.

5. **Run backward chaining from answer to question.** For each proposition in Pu, extract the bridge entity (the non-answer entity). Search Pc for a proposition whose object matches this bridge entity. Update the bridge entity to that proposition's subject. Repeat until either: (a) the bridge entity matches the question entity (success), or (b) no matching proposition is found (failure).

6. **Record the minimal sufficient set.** If backward chaining succeeds, collect all propositions traversed into Pmin. This is the smallest subset forming a complete deductive path from question entity to answer entity.

7. **Compute Completeness.** Score = 1 if Pmin is non-empty (a valid path exists), 0 otherwise. A score of 0 indicates a deductive gap -- classify it as circular, broken, or deviated based on where chaining failed.

8. **Compute Conciseness.** Score = |Pmin| / |P|. A score of 1.0 means every proposition was logically necessary. Lower scores indicate redundant or off-topic information. If Completeness = 0, Conciseness defaults to 0.

9. **Compute Determinateness.** Present only Q and LA (no external context) and attempt to derive the short answer. If the derived answer matches SA exactly, score = 1. Otherwise score = 0. This tests whether the reasoning is self-contained.

10. **Report results with diagnostics.** Output all three scores with a diagnostic breakdown: which propositions formed the minimal chain, which were redundant, and where any deductive gaps occurred.

## Concrete Examples

**Example 1: Evaluating a 2-hop QA answer**

```
User: Evaluate this multi-hop QA answer for logical integrity.

Question: "Who is the director of the film that stars the actor born in Springfield?"
Short Answer: "Christopher Nolan"
Long-form Answer: "The actor born in Springfield is John Smith [Doc1].
John Smith starred in the film Inception [Doc2]. The director of Inception
is Christopher Nolan [Doc3]. Springfield is located in Illinois [Doc4]."

Approach:
1. Decompose into propositions:
   P1: (John Smith, born in, Springfield) [Doc1]
   P2: (John Smith, starred in, Inception) [Doc2]
   P3: (Inception, directed by, Christopher Nolan) [Doc3]
   P4: (Springfield, located in, Illinois) [Doc4]

2. Partition:
   Pu = {P3} (contains answer entity "Christopher Nolan")
   Pc = {P1, P2, P4}

3. Backward chain from P3:
   - P3 bridge entity: "Inception"
   - Search Pc: P2 has object "Inception" -> match
   - P2 bridge entity: "John Smith"
   - Search Pc: P1 has subject "John Smith" and relates to "Springfield" -> match
   - "Springfield" matches the question entity -> SUCCESS

4. Pmin = {P1, P2, P3}

Output:
  Completeness:    1    (valid chain: P1 -> P2 -> P3)
  Conciseness:     0.75 (3 of 4 propositions needed; P4 is redundant)
  Determinateness: 1    (LA alone entails "Christopher Nolan")
  Redundant:       P4 ("Springfield is located in Illinois")
  Diagnosis:       Answer is complete and determinate but includes
                   one off-topic proposition about Springfield's location.
```

**Example 2: Detecting a broken reasoning chain**

```
User: Check if this answer has any deductive gaps.

Question: "What country is the birthplace of the author of Harry Potter?"
Short Answer: "United Kingdom"
Long-form Answer: "J.K. Rowling wrote Harry Potter [Doc1].
The United Kingdom has a population of 67 million [Doc2]."

Approach:
1. Decompose:
   P1: (J.K. Rowling, wrote, Harry Potter) [Doc1]
   P2: (United Kingdom, has population, 67 million) [Doc2]

2. Partition:
   Pu = {P2} (contains "United Kingdom")
   Pc = {P1}

3. Backward chain from P2:
   - P2 bridge entity: "67 million" (or "United Kingdom" as subject)
   - Search Pc for entity match: P1 has entities "J.K. Rowling"
     and "Harry Potter" -- neither matches
   - Chain BROKEN: no bridge between P1 and P2

4. Pmin = {} (empty)

Output:
  Completeness:    0    (BROKEN chain)
  Conciseness:     0    (defaults to 0 when incomplete)
  Determinateness: 0    (LA does not state where Rowling was born)
  Gap type:        Broken -- missing proposition linking
                   "J.K. Rowling" to "United Kingdom" (e.g.,
                   "J.K. Rowling was born in the United Kingdom")
  Diagnosis:       The answer establishes authorship but never connects
                   the author to a birthplace. A critical hop is missing.
```

**Example 3: Scoring a RAG pipeline's batch output**

```
User: I have 50 multi-hop QA outputs from my RAG pipeline. Score them
      for logical quality and identify the weakest reasoning patterns.

Approach:
1. For each QA pair, run the backward chaining evaluation as above
2. Aggregate scores across the batch:

Output (summary table):
  Metric           Mean    Median  Std Dev
  Completeness     0.72    1.00    0.45
  Conciseness      0.54    0.60    0.23
  Determinateness  0.68    1.00    0.47

  Failure breakdown (14 incomplete answers):
    Broken chains:   9  (64%) -- missing bridge propositions
    Circular:        3  (21%) -- self-referential loops
    Deviated:        2  (14%) -- irrelevant document synthesis

  Recommendation: The pipeline retrieves relevant documents (high attribution)
  but fails to synthesize cross-document entity bridges in 28% of cases.
  Priority fix: Add explicit bridge-entity extraction in the reasoning prompt.
```

## Best Practices

- **Do:** Decompose into the finest-grained atomic propositions possible. Each proposition should express exactly one (subject, relation, object) relationship. Coarser decomposition hides deductive gaps.
- **Do:** Preserve entity identity consistently across propositions. Use canonical entity names so backward chaining can match bridge entities reliably (e.g., "J.K. Rowling" not "she" or "the author").
- **Do:** Run Determinateness in a closed-world setting -- provide only the question and long-form answer, explicitly blocking any external knowledge or retrieval, to test self-containment.
- **Do:** Classify failure modes (circular, broken, deviated) rather than reporting only binary completeness. The failure type directly informs which part of the pipeline to fix.
- **Avoid:** Evaluating single-hop or factoid QA with this framework. LogicScore is designed for multi-hop reasoning where the chain structure matters. For single-hop, standard attribution metrics suffice.
- **Avoid:** Treating high Conciseness as inherently good. A conciseness score of 1.0 with completeness of 0 means the answer is both minimal and wrong. Always interpret conciseness conditional on completeness.
- **Avoid:** Using Determinateness alone as a quality signal. An answer can be determinate (entails the right answer) while having a broken reasoning chain (low completeness) if it simply states the answer without justification.

## Error Handling

- **Ambiguous entity resolution:** When propositions use pronouns or indirect references, coreference resolution may fail during triple extraction. Resolve by first running a coreference pass to replace pronouns with canonical entity names before decomposition.
- **Multiple valid chains:** Some answers support more than one backward-chaining path to the question entity. In this case, select the shortest valid chain as Pmin and report alternative paths as supplementary information.
- **Partial entity matches:** Bridge entities may partially overlap (e.g., "Inception film" vs. "Inception"). Use fuzzy entity matching with a threshold (e.g., token overlap > 0.8) rather than exact string matching.
- **Very long answers:** Answers with 20+ propositions can create combinatorial search during backward chaining. Prune by first filtering to propositions sharing entities with Pu or the question, then chain within this reduced set.
- **LLM decomposition errors:** When using an LLM to decompose LA into propositions, it may merge or split claims incorrectly. Validate by checking that each proposition contains exactly one relational claim and that the union of propositions covers all information in LA.

## Limitations

- LogicScore assumes reasoning follows a **linear chain structure** (entity bridges forming a path). Answers requiring tree-shaped or parallel reasoning (e.g., comparison questions needing two independent chains) may not be fully captured by a single backward chain.
- The framework is designed for **entity-centric multi-hop QA** where each hop resolves an entity. Questions requiring numerical reasoning, temporal ordering, or set operations may not decompose cleanly into Horn Rules.
- **Determinateness evaluation** relies on LLM re-inference, which introduces model-dependent variance. The same long-form answer may score differently depending on which LLM performs the re-derivation.
- Conciseness penalizes **all non-chain propositions equally**, whether they provide useful context (e.g., disambiguation) or are truly irrelevant. Human readers may value some "redundant" context.
- The backward chaining algorithm is **greedy** -- it takes the first matching bridge entity. In adversarial or ambiguous cases, this may miss the correct chain or select a spurious one.

## Reference

**Paper:** [LogicScore: Fine-grained Logic Evaluation of Conciseness, Completeness, and Determinateness in Attributed Question Answering](https://arxiv.org/abs/2601.15050v3) -- Yan et al., 2026. Look for: Section 3 (Horn Rule formulation and backward verification algorithm), Section 4 (metric definitions), and Table 2 (failure mode taxonomy: circular, broken, deviated).

**Code:** [github.com/zhichaoyan11/LogicScore](https://github.com/zhichaoyan11/LogicScore) -- Python pipeline with 5 stages covering generation, decomposition, backward chaining, citation evaluation, and factuality scoring.