---
name: "benchmarking-pairwise-causal-discovery"
description: >
  Detect and extract pairwise causal relationships from text using structured
  prompting strategies (zero-shot, CoT, few-shot ICL, Least-to-Most, ReAct).
  Built on the unified evaluation framework from Anuyah et al. (2026) covering
  12 datasets across biomedical and multi-domain contexts. Use when the user
  asks to "find causal relationships in text", "extract cause and effect from
  sentences", "detect causation vs correlation", "build a causal extraction
  pipeline", "benchmark causal reasoning", or "annotate causal links in a
  corpus".
---

# Pairwise Causal Discovery from Text

This skill enables Claude to detect whether text contains causal relationships and extract the exact cause-effect phrase pairs, using the structured prompting and evaluation framework from Anuyah et al. (2026). The core technique decomposes causal reasoning into two sequential competencies — **Causal Detection** (binary classification: causal vs. non-causal) and **Causal Extraction** (span identification of cause and effect phrases) — then applies difficulty-aware prompting strategies calibrated to the complexity of the input text along three dimensions: marker presence, sentence scope, and pair cardinality.

## When to Use

- When a user asks to identify cause-and-effect relationships in biomedical abstracts, clinical notes, or research papers
- When building an NLP pipeline that needs to distinguish causation from correlation, hypotheticals, or reported claims
- When annotating a corpus for causal relations and needing consistent labeling guidelines
- When evaluating or benchmarking an LLM's causal reasoning capabilities across datasets
- When extracting structured causal knowledge graphs from unstructured text
- When a user provides sentences or paragraphs and asks "what causes what here?"
- When designing prompts for causal reasoning tasks and needing empirically validated prompt templates

## Key Technique

**Two-stage pairwise causal discovery (PCD)** treats causation detection as a strict prerequisite for extraction. Stage 1 classifies text as "Causal" or "Non-Causal" using a role-framed prompt with explicit definitions that separate factual causal assertions from correlations, hypotheticals, and reported claims. Only text classified as causal proceeds to Stage 2, where cause and effect spans are extracted as minimal contiguous phrases excluding causal cue words (e.g., "causes", "leads to").

**Difficulty-aware prompt selection** is the actionable insight. The paper's three-dimensional taxonomy categorizes inputs by: (1) **Marker presence** — explicit markers like "causes" or "results in" vs. implicit unmarked causation; (2) **Sentence scope** — intra-sentential (within one sentence) vs. inter-sentential (spanning multiple sentences); (3) **Pair cardinality** — single cause-effect pair vs. multiple pairs. Simple explicit single-sentence cases work well with zero-shot or basic CoT prompts. Implicit or multi-sentence cases benefit from Least-to-Most decomposition (first find the cause, then use it as context to find the effect). Multi-pair cases require ReAct-style reasoning with Thought/Action/Observation loops.

**Evaluation uses approximate semantic matching**, not exact string match. Predicted cause-effect spans are compared to gold spans using sentence-embedding cosine similarity, mapped through a calibrated 10-band binomial scoring scheme derived from 4,000 human-annotated samples (96.72% accuracy). For multi-pair outputs, Hungarian 1:1 matching aligns predicted pairs to gold pairs before scoring. This prevents penalizing valid paraphrases while maintaining strict alignment requirements.

## Step-by-Step Workflow

1. **Classify the input text's difficulty** along three dimensions: Does it contain explicit causal markers (words like "causes", "leads to", "results in", "due to", "as a result")? Does the potential causal relationship span one sentence or multiple? Might there be more than one cause-effect pair?

2. **Run Causal Detection** with this prompt template, adapting the framing based on domain:
   ```
   You are a causal reasoning expert. Your task is to identify if the text
   contains a factual causal relationship. A 'Causal' relationship is one
   where a cause is stated as a fact to produce an effect. A 'Non-Causal'
   relationship includes correlations, reported claims ('studies show'),
   hypotheticals, or negated effects. Analyze the following text.
   Text: {INPUT_TEXT}
   Does this text contain a causal relationship? Answer only 'Causal' or 'Non-Causal'.
   ```

3. **Select the prompting strategy** for extraction based on the difficulty classification:
   - Explicit + single-sentence + single-pair: Use **zero-shot or CoT** ("Let's think step by step")
   - Implicit or multi-sentence: Use **Least-to-Most decomposition** — first ask "What is the cause?", then use that answer to ask "Given that the cause is X, what is the effect?"
   - Multiple causal pairs: Use **ReAct framework** — alternate Thought/Action/Observation steps to enumerate pairs sequentially
   - Any complexity: Add **3 canonical few-shot examples** demonstrating the target nuance (implicit causation, multi-sentence spans, or multiple pairs as appropriate)

4. **Extract cause-effect spans** using structured JSON output formatting:
   ```
   Extract all cause-effect pairs from the text as JSON.
   Rules: Select the smallest contiguous phrases that capture each cause
   and effect. Exclude causal cue words ('because', 'causes', 'leads to')
   from the spans. Decompose compound relations into separate pairs.
   Output format: [{"cause": "...", "effect": "..."}]
   ```

5. **Validate extracted spans** against the annotation principles: spans must be factual assertions (not hypothetical or reported), both cause and effect must be clearly identifiable, and ambiguous cases should default to non-causal.

6. **Score extraction quality** when gold labels are available: compute sentence-embedding cosine similarity between predicted and gold spans, apply Hungarian matching for multi-pair alignment, and map cosine scores to binary correctness using the 10-band calibration (threshold >= 0.85 for high confidence).

7. **Aggregate results** across the difficulty taxonomy to identify systematic weaknesses — track performance separately for explicit vs. implicit, intra vs. inter-sentential, and single vs. multi-pair.

8. **Iterate on prompt design** by adding domain-specific few-shot examples for the weakest categories, prioritizing implicit and inter-sentential cases where baseline LLM performance is lowest.

## Concrete Examples

**Example 1: Simple biomedical sentence (explicit, single-sentence, single-pair)**

User: "Extract the causal relationship from: 'Smoking causes lung cancer through chronic inflammation of bronchial tissue.'"

Approach:
1. Classify difficulty: explicit marker ("causes"), single sentence, single pair
2. Detection: "Causal" — factual assertion with explicit marker
3. Use zero-shot extraction (simple case)
4. Extract spans excluding the cue word "causes"

Output:
```json
[{"cause": "Smoking", "effect": "lung cancer through chronic inflammation of bronchial tissue"}]
```

Note: "causes" is excluded from both spans per the minimal-span principle.

**Example 2: Implicit multi-sentence causation (implicit, inter-sentential, single-pair)**

User: "Find causal links in: 'The patient discontinued statin therapy in March. By June, LDL cholesterol levels had risen to 195 mg/dL, well above the clinical threshold.'"

Approach:
1. Classify difficulty: no explicit causal marker, spans two sentences, likely single pair
2. Detection: "Causal" — the temporal sequence and domain knowledge imply factual causation
3. Use Least-to-Most decomposition for this implicit inter-sentential case:
   - Step A: "What event or action is the likely cause?" -> "The patient discontinued statin therapy"
   - Step B: "Given that cause, what is the stated effect?" -> "LDL cholesterol levels had risen to 195 mg/dL"
4. Validate: factual assertion, both spans identifiable, no hypothetical language

Output:
```json
[{"cause": "The patient discontinued statin therapy", "effect": "LDL cholesterol levels had risen to 195 mg/dL"}]
```

**Example 3: Multiple causal pairs (explicit, single-sentence, multi-pair)**

User: "Extract all causal relationships from: 'Deforestation leads to soil erosion, which in turn causes river sedimentation, while industrial runoff results in water acidification.'"

Approach:
1. Classify difficulty: explicit markers ("leads to", "causes", "results in"), single sentence, multiple pairs
2. Detection: "Causal" — multiple factual causal assertions
3. Use ReAct-style enumeration for multi-pair extraction:
   - Thought: The sentence contains a causal chain (deforestation -> soil erosion -> river sedimentation) and a separate causal link (industrial runoff -> water acidification)
   - Action: Decompose the chain into individual pairs
   - Observation: Three distinct pairwise relations identified
4. Decompose compound relations into separate pairs

Output:
```json
[
  {"cause": "Deforestation", "effect": "soil erosion"},
  {"cause": "soil erosion", "effect": "river sedimentation"},
  {"cause": "industrial runoff", "effect": "water acidification"}
]
```

**Example 4: Non-causal filtering (correlation, not causation)**

User: "Is there a causal relationship in: 'Studies have shown that coffee consumption is associated with lower rates of Parkinson's disease'?"

Approach:
1. Detection prompt applied
2. Key signals: "Studies have shown" (reported claim), "associated with" (correlation, not causation)
3. Per annotation guidelines: correlations and reported claims are non-causal

Output:
```
Non-Causal. This text describes a reported correlation ("studies have shown",
"associated with"), not a factual causal assertion. No cause-effect pairs extracted.
```

**Example 5: Misleading marker (appears causal but is not)**

User: "Detect causality in: 'The cause of the rash is unknown.'"

Approach:
1. Contains the word "cause" — an apparent marker
2. However, the sentence explicitly states the cause is unknown — no factual causal link is asserted
3. Per annotation guidelines: ambiguous or unknown causation defaults to non-causal

Output:
```
Non-Causal. Although the word "cause" appears, the sentence states the cause is
unknown. No factual causal relationship is asserted.
```

## Best Practices

**Do:**
- Always run detection before extraction — extraction on non-causal text produces hallucinated pairs (the paper found ~4.5% hallucination rate)
- Use the difficulty taxonomy (marker/scope/cardinality) to select the right prompting strategy rather than applying one strategy uniformly
- Include the explicit definition of "non-causal" categories (correlation, hypothetical, reported claim, negated effect) in detection prompts — this reduces false positives significantly
- Extract minimal contiguous spans and exclude causal cue words from both cause and effect phrases
- Decompose compound causal chains (A causes B causes C) into individual pairwise relations
- Use semantic similarity (cosine >= 0.85) rather than exact string match when evaluating extraction quality

**Avoid:**
- Applying zero-shot prompts to implicit or inter-sentential cases — performance drops dramatically; use Least-to-Most or CoT+FICL instead
- Treating "associated with", "linked to", or "correlated with" as causal markers — these indicate correlation
- Extracting causal pairs from hypothetical or conditional statements ("if X then Y might cause Z") unless the user specifically requests it
- Using unconstrained output formats — always request structured JSON to prevent parsing failures
- Assuming a single prompting strategy works across all difficulty levels — the paper shows no single strategy dominates
- Swapping cause and effect direction — ~4% of LLM errors are directionality swaps; verify the cause temporally/mechanistically precedes the effect

## Error Handling

| Error | Frequency | Cause | Fix |
|-------|-----------|-------|-----|
| Missing relations | 35.7% of errors | Multi-pair texts with incomplete extraction | Use ReAct enumeration; explicitly instruct "list ALL pairs" |
| Incorrect spans | 6.1% | Span boundaries include cue words or excess context | Reinforce minimal-span rule; add span-boundary examples to few-shot |
| Hallucinated pairs | 4.5% | Model invents causal links not in the text | Enforce detection gate; add "only extract pairs directly stated in the text" |
| Swapped cause/effect | 4.1% | Directionality confused in passive or complex syntax | Add explicit direction check: "Does the cause produce/enable the effect?" |
| False positive detection | Common | Correlation or reported claim classified as causal | Add non-causal examples to few-shot prompt; include explicit non-causal definitions |
| Empty extraction on causal text | Common | Implicit causation with no surface markers | Switch from zero-shot to Least-to-Most decomposition with sub-questions |

## Limitations

- **Implicit causation remains hard**: Even the best model achieved only ~49.6% on detection. Implicit causal links (no surface markers) show substantially lower accuracy than explicit ones. Manual review is recommended for high-stakes applications.
- **Inter-sentential scope degrades performance**: Causal links spanning multiple sentences are frequently missed. For long documents, consider sentence-windowing (2-3 sentence sliding windows) to keep spans manageable.
- **Domain transfer is not guaranteed**: Prompts optimized on biomedical text may not transfer to financial, legal, or technical domains without domain-specific few-shot examples.
- **Causal chains vs. pairwise**: This framework decomposes chains into pairwise relations. It does not model transitivity, confounders, feedback loops, or causal graphs directly.
- **Evaluation requires embeddings**: The semantic similarity scoring needs a sentence embedding model. Exact-match evaluation will undercount correct extractions that use different wording.
- **The best open-source models plateau below 50%** on the composite metrics (C_detect and C_extract) — this is a genuinely difficult task and outputs should be treated as assistive, not authoritative.

## Reference

Anuyah, S., Shajee-Mohan, S., Chauhan, A.-S., & Chakraborty, S. (2026). *Benchmarking LLMs for Pairwise Causal Discovery in Biomedical and Multi-Domain Contexts*. arXiv:2601.15479. [Paper](https://arxiv.org/abs/2601.15479) | [Code](https://github.com/sydneyanuyah/CausalDiscovery)

Key sections to consult: Table 1 (causal vs. non-causal examples with difficulty axes), Table 2 (difficulty taxonomy with 8 extraction sub-tasks), Section 4 (all five prompt templates: zero-shot, FICL, CoT, Least-to-Most, ReAct), Section 5 (per-task results by prompting strategy), Table 7 (error analysis with frequency breakdown).