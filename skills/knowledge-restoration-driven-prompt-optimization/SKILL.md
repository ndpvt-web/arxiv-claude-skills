---
name: "knowledge-restoration-driven-prompt-optimization"
description: |
  Iteratively optimize LLM prompts for information extraction tasks using self-evaluation feedback loops.
  Applies the KRPO framework: extract structured data, restore it to natural language, score semantic
  consistency via NLI, then generate textual gradients to refine the prompt. Includes relation
  canonicalization to deduplicate and normalize extracted schemas.
  Trigger phrases:
  - "Extract relations from text and optimize the prompt"
  - "Build a self-improving extraction pipeline"
  - "Optimize my prompt for triplet extraction"
  - "Extract knowledge graph triples from unstructured text"
  - "Set up iterative prompt refinement with feedback"
  - "Canonicalize extracted relations across documents"
---

# Knowledge Restoration-driven Prompt Optimization (KRPO)

This skill enables Claude to build **self-improving information extraction pipelines** that iteratively refine their own prompts through a feedback loop. Rather than relying on a static prompt for extracting structured data (entities, relations, triplets) from text, KRPO projects extracted structures back into natural language, scores semantic fidelity against the source, and uses those scores as "textual gradients" to rewrite the prompt. The result is a prompt that gets measurably better with each batch of data processed -- without any manual tuning.

## When to Use

- When the user needs to extract (subject, relation, object) triplets from unstructured text at scale
- When a fixed extraction prompt produces inconsistent or incomplete results and needs systematic improvement
- When building a knowledge graph from documents without a predefined ontology or schema
- When extracted relations are noisy, redundant, or inconsistently named across documents (e.g., "born in", "birthplace", "place of birth" all appearing)
- When the user wants to evaluate extraction quality without labeled ground truth
- When building any LLM pipeline that would benefit from automatic prompt refinement based on output quality signals

## Key Technique

**The core insight:** Instead of guessing whether an extraction prompt is good, KRPO measures it. Given an extracted triplet like `(Pontiac Rageous, assembly, Detroit)`, the system asks an LLM to restore it back to a natural language sentence: *"The Pontiac Rageous was assembled in Detroit."* It then uses Natural Language Inference (NLI) to check whether the original source text entails that restored sentence. If the source says "produced in Detroit, Michigan" but the triplet claims "assembly" as the relation, the NLI score reveals the semantic drift. These scores become feedback that drives prompt rewrites.

**Textual gradients** replace numerical backpropagation with language-based optimization. The system generates three layers of feedback: (1) per-sample critique identifying what went wrong with each extraction, (2) batch-level patterns aggregating recurring failure modes, and (3) concrete prompt edit instructions. This mirrors gradient descent but operates entirely in natural language -- the "gradient" is a textual description of how the prompt should change, and the "update step" is an LLM rewriting the prompt accordingly.

**Relation canonicalization** solves the vocabulary explosion problem in open-domain extraction. A memory bank stores canonical relation names. When a new relation is extracted, a cross-encoder scores its semantic similarity against existing canonical relations. If a close match exists (e.g., "birthplace" matches "placeOfBirth"), the extracted relation is mapped to the canonical form. If no match exists, the memory expands. This produces a clean, deduplicated schema without requiring one upfront.

## Step-by-Step Workflow

1. **Define the extraction prompt template.** Write an initial prompt that instructs the LLM to extract (subject, relation, object) triplets from a given text passage. Include format instructions (e.g., JSON array of triplets) and 1-2 few-shot examples. This is the prompt that will be iteratively improved.

2. **Batch the input corpus.** Split input documents into batches of 5-15 samples. Each batch will produce extraction results, feedback, and a prompt update. Smaller batches iterate faster; larger batches produce more stable gradients.

3. **Extract triplets with the current prompt.** For each text in the batch, call the LLM with the current extraction prompt. Parse the output into structured triplets: `[(subject, relation, object), ...]`. Handle malformed outputs by retrying or discarding.

4. **Restore triplets to natural language.** For each extracted triplet, prompt the LLM: *"Convert the following triplet into a single natural language sentence that preserves its exact meaning: (subject: X, relation: Y, object: Z)."* This produces a hypothesis sentence for NLI evaluation.

5. **Score semantic consistency via NLI.** For each (original_text, restored_sentence) pair, run NLI classification. Map outcomes to scores: **entailment = +1.0** (triplet is correct), **neutral = 0.0** (triplet is plausible but unsupported), **contradiction = -0.5** (triplet is wrong). Sum scores across the batch to get an aggregate quality metric.

6. **Generate textual gradients.** Prompt the LLM with the batch results, NLI scores, and the current extraction prompt. Ask it to: (a) identify specific failure patterns (missed entities, hallucinated relations, incomplete triplets), (b) explain why the current prompt led to those failures, and (c) propose concrete prompt edits to fix them.

7. **Apply the prompt update.** Feed the textual gradient and current prompt to the LLM with instructions to rewrite the prompt incorporating the suggested improvements. Preserve working elements; only change what the feedback targets. Store the previous prompt version for rollback if the next batch scores lower.

8. **Iterate across batches.** Repeat steps 3-7 for each batch. Track the aggregate NLI score per batch. Stop when scores plateau across 2-3 consecutive batches or a maximum iteration count is reached.

9. **Canonicalize relations.** After extraction is complete (or during extraction if running online), deduplicate relations. For each unique relation string, compute semantic similarity against a canonical relation memory. If similarity exceeds a threshold (e.g., 0.85), map to the canonical form. Otherwise, add as a new canonical relation. Use a cross-encoder or sentence embeddings for similarity.

10. **Output the final knowledge graph.** Emit the deduplicated triplets with canonical relations as the final structured output (JSON, CSV, or Neo4j-compatible format). Include the optimized prompt for reuse on future documents.

## Concrete Examples

**Example 1: Extracting knowledge from product descriptions**

```
User: "I have 200 product descriptions and need to extract structured
       relationships like manufacturer, materials, dimensions, and origin
       country. My current prompt misses a lot of info."

Approach:
1. Start with the user's existing extraction prompt as the seed.
2. Batch the 200 descriptions into groups of 10.
3. Run extraction on batch 1 with the seed prompt.
   Sample input: "The Oakley Holbrook sunglasses, designed in California
   and manufactured in Italy, feature O-Matter plastic frames with
   Plutonite polycarbonate lenses."
   Initial output:
     [("Oakley Holbrook", "designed in", "California"),
      ("Oakley Holbrook", "material", "plastic")]
   -- Missing: manufacturer location, lens material, frame material name.

4. Restore each triplet to natural language:
   - "The Oakley Holbrook was designed in California." -> NLI: entailment (+1.0)
   - "The Oakley Holbrook is made of plastic." -> NLI: neutral (0.0)
     (Source says "O-Matter plastic frames" not just "plastic")

5. Generate textual gradient:
   "The prompt fails to instruct extraction of manufacturing location
   vs. design location. It also loses specificity on materials --
   extracting 'plastic' instead of 'O-Matter plastic'. Add instructions
   to distinguish design/manufacturing locations and preserve full
   material names."

6. Updated prompt now includes: "Extract the full material name
   (e.g., 'O-Matter plastic' not just 'plastic'). Distinguish between
   design location and manufacturing location as separate relations."

7. Batch 2 extraction with updated prompt yields:
     [("Oakley Holbrook", "designLocation", "California"),
      ("Oakley Holbrook", "manufacturingLocation", "Italy"),
      ("Oakley Holbrook", "frameMaterial", "O-Matter plastic"),
      ("Oakley Holbrook", "lensMaterial", "Plutonite polycarbonate")]
   NLI scores: all entailment (+4.0 vs +1.0 previously).

8. After 20 batches, canonicalize: "made in" -> "manufacturingLocation",
   "produced in" -> "manufacturingLocation", "built with" -> "material".

Output: 200 product descriptions -> ~900 canonicalized triplets, JSON format.
Optimized prompt saved for reuse on new product data.
```

**Example 2: Building a knowledge graph from news articles**

```
User: "Extract entity relationships from these 50 news articles about
       tech acquisitions. I need acquirer, target, price, and date."

Approach:
1. Write seed prompt: "Extract all (subject, relation, object) triplets
   about corporate acquisitions from the following text. Output as JSON."

2. Batch into groups of 10 articles.

3. Batch 1 extraction on: "Microsoft completed its $69 billion
   acquisition of Activision Blizzard on October 13, 2023, after
   securing regulatory approval from the UK's CMA."
   Output:
     [("Microsoft", "acquired", "Activision Blizzard"),
      ("Microsoft", "acquisition price", "$69 billion")]
   Missing: date, regulatory body.

4. Knowledge restoration + NLI:
   - "Microsoft acquired Activision Blizzard." -> entailment (+1.0)
   - "Microsoft's acquisition price was $69 billion." -> entailment (+1.0)
   Aggregate: +2.0, but 2 triplets from a 4-fact sentence = low recall.

5. Textual gradient: "Prompt does not instruct extraction of temporal
   information or regulatory entities. Add: 'Include dates, regulatory
   bodies, and approval statuses as separate triplets.'"

6. After 3 iterations, extraction of the same text yields:
     [("Microsoft", "acquired", "Activision Blizzard"),
      ("acquisition", "value", "$69 billion"),
      ("acquisition", "completionDate", "October 13, 2023"),
      ("acquisition", "regulatoryApproval", "UK CMA")]
   NLI: all entailment (+4.0).

7. Canonicalize across all articles: "bought", "acquired", "took over"
   all map to "acquired". "deal value", "acquisition price", "price tag"
   all map to "acquisitionValue".

Output: Structured acquisition knowledge graph with 4 canonical relation
types, exported as Neo4j-compatible CSV.
```

**Example 3: Iterative prompt optimization for biomedical text**

```
User: "My extraction prompt for biomedical abstracts only gets 40% of
       the drug-gene interactions. Help me improve it systematically."

Approach:
1. Take the user's current prompt as seed. Run on 5 sample abstracts.
2. For each extracted triplet like ("metformin", "inhibits", "mTOR"),
   restore to: "Metformin inhibits mTOR."
3. NLI against source: If source says "metformin activates AMPK, which
   in turn inhibits mTOR signaling", the direct claim scores neutral (0.0)
   because the relationship is indirect.
4. Textual gradient: "Prompt conflates direct and indirect interactions.
   Add instruction: 'Specify whether the interaction is direct or
   mediated by another entity. For indirect interactions, extract the
   full chain.'"
5. After 5 optimization rounds:
   - Precision improves (fewer hallucinated direct interactions)
   - Recall improves (prompt now instructs extraction of pathway chains)
   - User's extraction rate rises from ~40% to ~65%+ on held-out samples

Output: Optimized biomedical extraction prompt + relation canonicalization
mapping (e.g., "inhibits"/"suppresses"/"downregulates" -> "negativelyRegulates").
```

## Best Practices

- **Do:** Track NLI scores per batch and plot them. A rising score curve confirms the optimization is working. If scores plateau early, the seed prompt may need manual restructuring before automated refinement can help.
- **Do:** Keep a version history of every prompt iteration. If a prompt update causes regression (lower NLI scores), roll back to the previous version and try a different textual gradient.
- **Do:** Use the canonicalization memory as a living artifact. Export it alongside the knowledge graph so downstream consumers know the schema. Seed it with a few obvious canonical relations to guide early convergence.
- **Do:** Run NLI with a dedicated model (e.g., a fine-tuned NLI classifier) rather than prompting the same extraction LLM, to avoid self-confirmation bias.
- **Avoid:** Optimizing on a single sample. Textual gradients from individual examples overfit to edge cases. Always aggregate feedback across a batch of at least 5 samples before updating the prompt.
- **Avoid:** Running too many optimization iterations without checking for diminishing returns. Empirically, 15-25 batches suffice for most domains. Beyond that, gains are marginal and the prompt may start overfitting to the training distribution.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| LLM returns malformed triplets (not valid JSON) | Prompt format instructions unclear | Add explicit JSON schema to the prompt with a concrete example. Wrap extraction call with a retry + JSON repair step. |
| NLI scores all neutral | Restored sentences are too vague | Improve the restoration prompt to produce specific, detailed sentences rather than generic summaries. |
| Prompt gets worse after update | Textual gradient was too aggressive | Roll back to previous prompt version. Constrain the update instruction: "Make minimal targeted edits only." |
| Relation canonicalization maps distinct relations together | Similarity threshold too low | Raise the threshold (e.g., 0.85 -> 0.92) or add negative examples to the canonicalization prompt. |
| Extraction misses implicit relationships | Seed prompt only targets explicit statements | Add instruction in the prompt: "Extract both explicitly stated and strongly implied relationships." |
| Optimization loop diverges (scores oscillate) | Contradictory feedback across batches | Increase batch size to smooth gradients. Consider using majority-vote across multiple gradient samples. |

## Limitations

- **Requires multiple LLM calls per sample.** Each text needs extraction, restoration, NLI, and gradient generation -- at minimum 4 LLM calls per sample per iteration. Cost scales with corpus size multiplied by iteration count.
- **NLI is an imperfect proxy for correctness.** Entailment checks whether the source supports the triplet, not whether the triplet captures all information. High NLI scores can coexist with low recall.
- **Open-domain relations resist full canonicalization.** Without a target ontology, the canonical memory can still grow large. Domain-specific extraction with a fixed schema will always be cleaner.
- **The method assumes the source text is truthful.** If the source contains errors or contradictions, NLI-based evaluation will validate incorrect triplets.
- **Textual gradients are noisy.** Unlike numerical gradients, language-based feedback can be vague, contradictory, or hallucinated. Aggregating across larger batches mitigates but does not eliminate this.

## Reference

**Paper:** [Knowledge Restoration-driven Prompt Optimization: Unlocking LLM Potential for Open-Domain Relational Triplet Extraction](https://arxiv.org/abs/2601.15037v1) (Jing et al., 2026). Look for: the NLI scoring formula (Section 3.2), the three-layer textual gradient generation process (Section 3.3), and the cross-encoder canonicalization mechanism (Section 3.4). Benchmark results on WebNLG, REBEL, and Wiki-NRE show +1.2 to +5.6 F1 improvement over baselines.