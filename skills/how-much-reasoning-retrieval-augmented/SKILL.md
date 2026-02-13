---
name: "how-much-reasoning-retrieval-augmented"
description: "Build contamination-aware hybrid RAG evaluation pipelines that couple knowledge graphs with text retrieval for multi-hop reasoning benchmarks. Use when: 'build a RAG benchmark', 'evaluate multi-hop reasoning', 'create hybrid KG-text retrieval pipeline', 'detect parametric recall vs genuine reasoning', 'generate multi-hop QA from knowledge graphs', 'benchmark RAG system with contamination control'."
---

# HybridRAG-Bench: Contamination-Aware Multi-Hop Reasoning Evaluation over Hybrid Knowledge

This skill enables Claude to build evaluation pipelines that rigorously test whether retrieval-augmented generation (RAG) systems perform genuine multi-hop reasoning or merely recall answers memorized during pretraining. The core technique, from the HybridRAG-Bench framework, constructs benchmarks by coupling unstructured text with structured knowledge graphs derived from time-bounded scientific literature, then generates QA pairs grounded in explicit reasoning paths that require both modalities to answer.

## When to Use

- When the user wants to **build a benchmark** to evaluate a RAG system's reasoning vs. memorization
- When the user needs to **construct a knowledge graph from a document corpus** and align it with text chunks for hybrid retrieval
- When the user asks to **generate multi-hop QA pairs** with explicit reasoning paths over a knowledge base
- When the user wants to **detect data contamination** in LLM evaluations by controlling temporal boundaries of knowledge sources
- When the user is **comparing RAG architectures** (text-only, KG-only, hybrid) and needs a principled evaluation framework
- When the user asks to **evaluate whether a model actually reasons** or just pattern-matches from training data

## Key Technique

HybridRAG-Bench addresses a critical blind spot in RAG evaluation: most benchmarks overlap with LLM pretraining data, so high scores may reflect memorization rather than retrieval-and-reasoning ability. The framework solves this with a four-stage pipeline: (1) collect documents from a time-bounded window that postdates the target LLM's training cutoff, (2) build a hybrid knowledge representation coupling text chunks with a structured knowledge graph, (3) sample reasoning paths through the KG and generate QA pairs that require both graph traversal and text comprehension, and (4) validate quality via LLM-as-judge.

The hybrid knowledge coupling is the key innovation. Each entity and relation in the KG is grounded in supporting text spans from the corpus. Questions are constructed so that answering them requires integrating structured relational facts (from KG paths like `v0 -> r1 -> v1 -> r2 -> v2`) with contextual information only available in the unstructured text. This makes it impossible to solve questions through either modality alone. The framework generates six question types of increasing difficulty: single-hop, conditional single-hop, multi-hop (k>=2), hard multi-hop (through high-degree entities), counterfactual (minimal perturbations), and open-ended synthesis.

Contamination control works by restricting the source corpus to a configurable time window `[T_start, T_end]` that postdates the evaluated model's training cutoff. Experiments showed that models trained on data overlapping with benchmark sources scored 79.6% accuracy, while models without that exposure dropped to 24.4% -- proving that without temporal controls, benchmarks measure recall, not reasoning.

## Step-by-Step Workflow

1. **Define the evaluation scope.** Choose a domain (e.g., `cs.AI`, `cs.LG`, policy documents, biomedical literature) and a time window `[T_start, T_end]` that postdates the training cutoff of every model you intend to evaluate. For recent models with mid-2025 cutoffs, use papers from late 2025 onward.

2. **Collect the source corpus.** Fetch documents from the chosen domain within the time window. For arXiv, use the API to retrieve papers by subject category and date range. Store full text, metadata (title, authors, date, categories), and any structured data (tables, equations) as separate artifacts.

3. **Extract entities and relations from the corpus.** Use an LLM to identify candidate entities (concepts, methods, datasets, metrics) and relations (uses, outperforms, extends, applied-to) from each document. Normalize entity mentions using joint embeddings over entity type, name, and description. Use HNSW-based similarity search with a configurable threshold to merge duplicates or create new KG nodes.

4. **Build the knowledge graph in a graph database.** Import extracted entities as nodes and relations as edges into Neo4j (or equivalent). Attach supporting text spans to each node and edge so every KG element is grounded in corpus evidence. Run `APOC` procedures for indexing and embed entities/relations for downstream retrieval.

5. **Sample reasoning paths through the KG.** Generate paths of the form `p = (v0 -> r1 -> v1 -> r2 -> ... -> rk -> vk)` where `vk` is the answer entity. For hard multi-hop paths, preferentially select routes through high-degree nodes. Retrieve the supporting text spans for each entity and relation along the path.

6. **Generate QA pairs from reasoning paths.** Prompt an LLM with: (a) the sampled reasoning path with relational constraints, (b) the textual evidence spans for each path element, and (c) in-context exemplars for each question type. Generate questions across all six types. Instruct the model to abstain when evidence is insufficient for a well-posed question.

7. **Validate generated QA pairs.** Apply LLM-as-judge quality control: check answerability (the question can be answered solely from the provided hybrid context), faithfulness (the answer is supported by evidence), deduplication, and normalization. Remove or regenerate failing pairs.

8. **Run baseline and target models.** Evaluate at least four configurations: (a) LLM-only (IO, CoT, self-consistency), (b) text-only RAG (BM25 or dense retrieval), (c) KG-only (1-hop neighbor injection), and (d) hybrid KG-RAG methods. Record accuracy broken down by question type and hop count.

9. **Analyze results for genuine reasoning vs. parametric recall.** Compare LLM-only scores against RAG-augmented scores. If LLM-only performance is high, the benchmark likely overlaps with pretraining data -- tighten the time window. Genuine reasoning is evidenced when hybrid methods substantially outperform both text-only and KG-only baselines, especially on multi-hop and hard multi-hop questions.

10. **Iterate the benchmark.** As new models are released with later training cutoffs, shift `T_start` forward to maintain contamination control. Re-run the pipeline on the updated corpus to produce a fresh, uncontaminated benchmark.

## Concrete Examples

**Example 1: Building a RAG benchmark for an AI research domain**

```
User: I want to evaluate whether my RAG pipeline actually reasons over
recent AI papers or just recalls memorized answers. Build me a benchmark.

Approach:
1. Set domain to cs.AI, cs.LG with T_start=2025-10 and T_end=2026-02
   (postdating GPT-4o/Claude training cutoffs).
2. Fetch ~500 recent arXiv papers via the arXiv API:
   ```python
   import arxiv
   search = arxiv.Search(
       query="cat:cs.AI OR cat:cs.LG",
       max_results=500,
       sort_by=arxiv.SortCriterion.SubmittedDate
   )
   papers = [p for p in search.results()
             if p.published >= datetime(2025, 10, 1)]
   ```
3. Extract entities (models, datasets, metrics, techniques) and relations
   (outperforms, uses, extends, evaluated-on) using structured prompts:
   ```
   Extract all technical entities and their relationships from this passage.
   Format: (entity1, relation, entity2, supporting_quote)
   ```
4. Load into Neo4j, sample 2-hop and 3-hop reasoning paths.
5. Generate ~800 QA pairs across six question types.
6. Validate with LLM-as-judge, yielding ~650 clean pairs.
7. Run evaluations: LLM-only, dense-retrieval RAG, KG-only, hybrid.

Output:
- benchmark.json with 650 QA pairs, each annotated with:
  - question, answer, question_type, reasoning_path, supporting_text_spans
- eval_results.json with per-model, per-question-type accuracy
- contamination_report.md showing LLM-only baseline as sanity check
```

**Example 2: Detecting parametric recall in an existing RAG system**

```
User: My RAG system scores 85% on our internal QA benchmark but I suspect
the LLM already knows most answers. How do I test this?

Approach:
1. Run the LLM backbone alone (no retrieval) on the same benchmark.
   If LLM-only scores >60%, contamination is likely.
2. Construct a contamination-controlled version:
   - Identify the LLM's training cutoff date.
   - Rebuild the QA set using only documents published AFTER that date.
   - Re-extract KG, re-sample paths, re-generate questions.
3. Compare scores:
   - Original benchmark: LLM-only 72%, RAG 85% (+13)
   - Time-controlled benchmark: LLM-only 28%, RAG 61% (+33)
4. The time-controlled benchmark reveals the RAG system adds +33 points
   of genuine retrieval value, vs. only +13 on the contaminated set.

Output:
- Side-by-side accuracy table showing contaminated vs. clean benchmarks
- Per-question-type breakdown revealing which reasoning types benefit
  most from retrieval (typically multi-hop and conditional)
- Recommendation: adopt time-controlled evaluation going forward
```

**Example 3: Generating hybrid multi-hop QA pairs from a custom corpus**

```
User: I have 200 policy documents about EU AI regulation. Generate
multi-hop questions that require both text retrieval and KG traversal.

Approach:
1. Parse the 200 documents, chunk into ~500-token passages with overlap.
2. Extract entities: regulations (AI Act), organizations (EC, ENISA),
   concepts (high-risk AI, conformity assessment), dates, thresholds.
3. Extract relations: regulates, requires, exempts, defines, amends.
4. Build KG with ~2000 nodes and ~5000 edges in Neo4j.
5. Sample 3-hop paths, e.g.:
   AI_Act -> requires -> conformity_assessment -> applies_to -> high_risk_system -> defined_by -> Annex_III
6. Generate question from this path + supporting text:
   "Under the EU AI Act, what conformity assessment procedure applies
    to systems classified as high-risk under Annex III?"
7. The answer requires: (a) KG traversal to link AI Act -> conformity
   assessment -> high-risk -> Annex III, and (b) text retrieval to get
   the specific procedural details not captured in the KG structure.

Output:
- 400 QA pairs with reasoning_path annotations
- Neo4j database with the policy KG
- Evaluation harness ready to test RAG pipelines against the dataset
```

## Best Practices

**Do:**
- Always establish a contamination baseline by running the LLM without retrieval first. If LLM-only accuracy exceeds 50%, your benchmark is likely contaminated.
- Ground every KG entity and relation in explicit supporting text spans from the corpus. This ensures questions genuinely require hybrid reasoning.
- Generate questions across all six types (single-hop, conditional, multi-hop, hard multi-hop, counterfactual, open-ended) to diagnose specific reasoning failures.
- Use HNSW-based similarity search with tuned thresholds for entity alignment during KG construction to avoid duplicate nodes that fragment reasoning paths.
- Track the fact recovery rate (recovered facts / verifiable facts in corpus) as a KG quality metric. Aim for >70%.

**Avoid:**
- Do not use documents published before the evaluated model's training cutoff. This is the single most common mistake and renders the benchmark meaningless.
- Do not evaluate only overall accuracy. Break results down by question type and hop count -- aggregate numbers hide reasoning deficits on hard multi-hop questions.
- Do not skip the LLM-as-judge validation step. Unvalidated QA pairs introduce noise that obscures real performance differences between methods.
- Do not assume KG-only or text-only baselines are sufficient comparisons. The diagnostic value comes from comparing hybrid methods against both single-modality baselines.

## Error Handling

- **Low fact recovery rate (<50%):** The entity/relation extraction prompts may be too restrictive. Widen extraction by lowering confidence thresholds or using a larger LLM for extraction. Inspect missed facts to identify systematic gaps (e.g., implicit relations the extractor misses).
- **LLM-only baseline too high:** The time window overlaps with pretraining data. Shift `T_start` forward by 3-6 months and regenerate the benchmark.
- **Generated questions are trivial or unanswerable:** The reasoning paths may be too short or disconnected from supporting text. Increase minimum path length to 2 hops and enforce that each hop has grounding text. Re-run validation with stricter answerability criteria.
- **KG has too many duplicate entities:** Lower the HNSW similarity threshold for entity merging (e.g., from 0.85 to 0.75) or add type-constrained matching to reduce false negatives.
- **Neo4j performance degrades on large KGs:** Add indexes on entity name and type properties. Use APOC periodic commit for bulk imports. For KGs exceeding 1M nodes, partition by document cluster.

## Limitations

- The framework assumes access to a corpus with clear temporal metadata (like arXiv submission dates). For domains without reliable timestamps, contamination control is weakened.
- KG construction quality depends heavily on the extraction LLM. Domain-specific terminology (bioinformatics, legal language) may require fine-tuned extractors or manual schema design.
- The six question types are comprehensive but may not cover all reasoning patterns relevant to specific applications (e.g., numerical reasoning, spatial reasoning).
- Generating a full benchmark is computationally expensive: entity extraction, KG construction, path sampling, QA generation, and validation each require multiple LLM calls across the corpus.
- The framework evaluates retrieval-augmented reasoning but does not prescribe how to improve it. It is a diagnostic tool, not a training methodology.
- Counterfactual questions test robustness but may not reflect real-world query distributions.

## Reference

**Paper:** Lin et al., "How Much Reasoning Do Retrieval-Augmented Models Add beyond LLMs? A Benchmarking Framework for Multi-Hop Inference over Hybrid Knowledge" (arXiv:2602.10210, 2026). Look for: the four-stage pipeline architecture (Section 3), the six question types with generation methodology (Section 3.3), contamination analysis via temporal controls (Table 1), and the comparative evaluation of 10+ RAG methods across three domains (Tables 3-4).

**Code:** [github.com/junhongmit/HybridRAG-Bench](https://github.com/junhongmit/HybridRAG-Bench) -- reference implementation with Neo4j integration, arXiv fetcher, KG construction, QA generation, and evaluation harness.