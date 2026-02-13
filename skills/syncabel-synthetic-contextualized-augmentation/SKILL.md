---
name: "syncabel-synthetic-contextualized-augmentation"
description: "Generate synthetic training data for biomedical entity linking using LLM-based contextualized augmentation. Use when: 'generate synthetic training data for entity linking', 'augment biomedical NER data', 'link mentions to UMLS/SNOMED concepts', 'create synthetic biomedical annotations', 'build entity linking pipeline with limited labeled data', 'normalize medical entities to knowledge base codes'."
---

# SynCABEL: Synthetic Contextualized Augmentation for Biomedical Entity Linking

This skill enables Claude to implement the SynCABEL framework for building biomedical entity linking (BEL) systems that overcome the scarcity of expert-annotated training data. The core technique uses large language models to generate context-rich synthetic training examples for every concept in a target knowledge base (UMLS, SNOMED-CT, MedDRA, etc.), then trains a decoder-only model with guided inference to map entity mentions in text to their canonical knowledge base identifiers. This approach achieves state-of-the-art performance across multilingual biomedical benchmarks while requiring up to 60% less human-annotated data.

## When to Use

- When the user needs to normalize biomedical entity mentions in clinical text, literature, or EHRs to standard knowledge base codes (UMLS CUIs, SNOMED-CT codes, MedDRA terms)
- When building an entity linking system but only a small number of expert-annotated examples are available
- When the user asks to generate synthetic training data for biomedical NLP tasks
- When linking rare or unseen medical concepts that lack annotated training examples
- When the user needs a multilingual biomedical entity linking pipeline (English, French, Spanish, or other languages with KB coverage)
- When expanding coverage of a supervised entity linker to new concept vocabularies without additional manual annotation
- When evaluating entity linking systems and standard exact-match metrics are insufficient (LLM-as-a-judge clinical validity assessment)

## Key Technique

**The data scarcity problem.** Supervised biomedical entity linking requires training examples mapping mentions-in-context to knowledge base concept IDs. Most KBs contain hundreds of thousands of concepts, but annotated corpora cover only a small fraction. SynCABEL solves this by using an LLM (e.g., Llama-3-70B) to generate three synthetic training examples per KB concept, each consisting of a realistic biomedical sentence containing a natural mention of that concept. The prompts combine: (1) a structured task description requiring two-step generation (first select a mention variant, then compose a contextualized sentence), (2) five random examples drawn from any available human-annotated data to establish style, and (3) the target concept's metadata—title, semantic group, definitions, and synonyms from the KB.

**Adaptive concept representation.** Rather than always using the KB-preferred name as the training target, SynCABEL selects the synonym most semantically similar to the mention surface form using cosine similarity of embeddings. Ambiguous synonyms (those shared across concepts within the same semantic group) are filtered out. This produces training targets that are closer to how entities actually appear in text, reducing the distribution gap between training and inference.

**Guided decoding at inference.** The fine-tuned decoder-only model (Llama-3-8B) generates concept identifiers autoregressively. At inference time, a trie built from all KB synonyms constrains token generation at each step to valid branches, guaranteeing that every output maps to a real KB concept. Semantic group filtering further narrows the trie to concepts matching the detected entity type, dramatically reducing the search space.

## Step-by-Step Workflow

1. **Define the target knowledge base and scope.** Identify which KB (UMLS, SNOMED-CT, MedDRA, custom ontology) and which subset of concepts (e.g., UMLS semantic types, SNOMED branches) the linker must cover. Extract concept IDs, preferred names, synonyms, definitions, and semantic groups into a structured format (JSON or TSV).

2. **Prepare the synthetic generation prompt template.** Construct a prompt in the target language with three blocks: (a) a task description instructing the LLM to first choose a mention surface form for the given concept, then write a realistic biomedical sentence using that mention; (b) a slot for 5 randomly sampled human-annotated examples (if available) showing the expected format; (c) the target concept's metadata (title, semantic group, definitions, all synonyms).

3. **Generate synthetic training examples at scale.** For each concept in the KB, call the LLM (Llama-3-70B or comparable) three times with the prompt template, sampling different random few-shot examples each time. Parse outputs to extract the generated mention and sentence. Store as triples: `(sentence, mention_span, concept_id)`.

4. **Build the adaptive concept representation mapping.** For each concept, embed all its synonyms using a biomedical sentence encoder (e.g., SapBERT). Remove synonyms that are ambiguous within the same semantic group. At training time, for each synthetic or human-annotated example, select the synonym with the highest cosine similarity to the mention surface form as the target label.

5. **Construct the training input format.** Format each example as: `context_left [mention] {semantic_group} context_right [SEP] [mention] is <target_synonym>`. The model learns to generate the target synonym autoregressively given the mention-in-context.

6. **Combine human-annotated and synthetic data.** Merge any available human-annotated training examples with the full synthetic dataset. Human examples anchor stylistic fidelity; synthetic examples provide broad KB coverage. If no human data exists, use synthetic data alone (performance degrades but remains viable).

7. **Fine-tune a decoder-only model.** Fine-tune Llama-3-8B (or similar) on the combined dataset using standard causal language modeling loss, masking the input prefix so loss is computed only on the target synonym tokens. Use LoRA or QLoRA for parameter efficiency if resources are constrained.

8. **Build the guided decoding trie.** Construct a prefix trie from all (filtered) synonyms in the KB, keyed by token IDs from the model's tokenizer. Optionally partition the trie by semantic group so that at inference time, only concepts matching the detected entity type are candidates.

9. **Run inference with constrained generation.** For each mention-in-context at test time, format the input identically to training, then generate with the trie constraining each token step. The output is a valid KB synonym, which maps deterministically to a concept ID.

10. **Evaluate with exact match and LLM-as-a-judge.** Compute Recall@1 against gold concept IDs. For clinical applications, implement an LLM-as-a-judge protocol: present the gold concept and predicted concept to a capable LLM and ask whether the prediction is clinically equivalent, to capture cases where different codes represent the same clinical reality.

## Concrete Examples

**Example 1: Generating Synthetic Data for UMLS Entity Linking**

User: "I have a small annotated dataset of 500 clinical notes with UMLS CUI annotations. I need to build an entity linker covering all of UMLS ST21pv (over 300K concepts). How do I augment my data?"

Approach:
1. Export the ST21pv concept list with CUIs, preferred names, synonyms, definitions, and semantic types from UMLS.
2. Build a generation prompt:
```
Task: You are generating biomedical training data. Given a medical concept,
(1) select one natural mention form from the concept's synonyms or a
    paraphrase that a clinician might use, then
(2) write a realistic 1-2 sentence clinical passage containing that mention.

Format your output as:
Mention: <selected surface form>
Sentence: <passage with the mention in context>

Examples from the dataset:
- Concept: Myocardial Infarction (C0027051, Disorder)
  Mention: heart attack
  Sentence: The patient presented to the ER with a suspected heart attack after
  reporting severe chest pain radiating to the left arm.
[...4 more random examples...]

Now generate for this concept:
- Title: {concept_title}
- Semantic Group: {semantic_group}
- Definition: {definition}
- Synonyms: {synonym_list}
```
3. Generate 3 examples per concept using Llama-3-70B (or via API). For 300K concepts, this yields ~900K synthetic training pairs.
4. Combine with the 500 real annotations and fine-tune Llama-3-8B.

Output:
```json
{
  "concept_id": "C0027051",
  "generated_examples": [
    {
      "mention": "MI",
      "sentence": "Troponin levels were markedly elevated, confirming the diagnosis of MI in the 68-year-old male."
    },
    {
      "mention": "myocardial infarction",
      "sentence": "Following an acute myocardial infarction, the patient was started on dual antiplatelet therapy."
    },
    {
      "mention": "heart attack",
      "sentence": "Family history is significant for a heart attack in his father at age 55."
    }
  ]
}
```

**Example 2: Building a Guided Decoding Trie for SNOMED-CT**

User: "I've fine-tuned my model. How do I ensure it only outputs valid SNOMED-CT codes at inference?"

Approach:
1. Load all SNOMED-CT concept synonyms and their associated concept IDs.
2. Tokenize every synonym using the model's tokenizer.
3. Build a prefix trie where each path from root to leaf represents a tokenized synonym.
4. At each generation step, mask logits for tokens that would leave the trie (i.e., tokens that don't continue any valid synonym prefix).

```python
from collections import defaultdict

class ConceptTrie:
    def __init__(self):
        self.root = {}
        self.synonym_to_concept = {}

    def add_synonym(self, token_ids: list[int], concept_id: str, synonym: str):
        node = self.root
        for tid in token_ids:
            if tid not in node:
                node[tid] = {}
            node = node[tid]
        node["__concept__"] = concept_id
        self.synonym_to_concept[synonym] = concept_id

    def get_valid_next_tokens(self, prefix_token_ids: list[int]) -> list[int]:
        node = self.root
        for tid in prefix_token_ids:
            if tid not in node:
                return []
            node = node[tid]
        return [k for k in node.keys() if k != "__concept__"]

    def get_concept_at(self, token_ids: list[int]) -> str | None:
        node = self.root
        for tid in token_ids:
            if tid not in node:
                return None
            node = node[tid]
        return node.get("__concept__")
```

5. During generation, after each token, call `get_valid_next_tokens` and set all other logits to `-inf`. When `get_concept_at` returns a concept ID, generation is complete.

**Example 3: Low-Resource Multilingual Entity Linking**

User: "I need to link Spanish clinical entities to SNOMED-CT but have zero annotated Spanish data."

Approach:
1. Extract the Spanish SNOMED-CT release (concept IDs, Spanish synonyms, definitions, semantic tags).
2. Write the generation prompt entirely in Spanish, using the same structure (task description, concept metadata).
3. Generate 3 synthetic examples per concept using a multilingual LLM (Llama-3-70B handles Spanish well).
4. Fine-tune Llama-3-8B on synthetic-only data.
5. Build the guided decoding trie from Spanish synonyms.
6. At inference, the model links Spanish mentions to SNOMED-CT codes despite having no human annotations.

Expected performance: ~60-65% Recall@1 on Spanish clinical text with synthetic-only training (the paper reports 62.6% on SPACCC with synthetic augmentation, rising to 67.0% when combined with human data).

## Best Practices

- **Do:** Generate in the same language as your target text. SynCABEL's synthetic examples must match the linguistic register of the deployment domain. Clinical notes need clinical-sounding synthetic sentences, not Wikipedia-style prose.
- **Do:** Filter ambiguous synonyms before building the trie. If "cold" maps to both "Common Cold" and "Cold Temperature" within the same semantic group, remove it to prevent systematic confusion.
- **Do:** Use the adaptive synonym selection (cosine similarity) rather than always mapping to preferred names. This closes the distribution gap between how concepts appear in text versus how they're named in the KB.
- **Do:** Include concept definitions and semantic group labels in generation prompts. These metadata fields steer the LLM toward domain-appropriate language and reduce hallucinated mentions.
- **Avoid:** Generating synthetic data for concepts without definitions or synonyms. The LLM lacks sufficient grounding and will produce noisy examples. The paper found only 4.5-6.5% of UMLS concepts had usable definitions, limiting gains on that KB.
- **Avoid:** Skipping guided decoding at inference. Without trie-constrained generation, the model may output free-form text that doesn't map to any KB concept, wasting compute and producing unlinkable predictions.

## Error Handling

- **LLM generates off-topic or hallucinated mentions.** Validate that the generated mention string appears (or approximately appears) in the concept's synonym list or is a reasonable paraphrase. Discard examples where the mention has zero lexical overlap with any synonym.
- **Trie produces no valid next tokens.** This occurs when the input context leads the model toward an unexpected prefix. Fall back to the top-k unconstrained predictions and post-hoc filter against the KB. Log these cases for prompt or training data refinement.
- **Tokenizer mismatch between training and trie construction.** Always build the trie with the exact same tokenizer and settings (including special tokens, padding side) used during fine-tuning. A single token ID mismatch invalidates the entire constrained decoding.
- **Semantic group misclassification.** If the upstream NER system assigns the wrong semantic group, the filtered trie excludes the correct concept entirely. Implement a fallback that searches the unfiltered trie when the filtered trie yields low-confidence predictions.
- **Memory issues with large KBs.** A trie over 300K+ concepts with multiple synonyms each can consume significant RAM. Use a compressed trie (MARISA-trie or LOUDS encoding) or shard by semantic group and load lazily.

## Limitations

- Requires a knowledge base with structured metadata (synonyms, definitions, semantic groups). Poorly curated or flat ID-only vocabularies provide insufficient signal for synthetic generation.
- Synthetic data quality depends heavily on the generation LLM's biomedical knowledge. Smaller or less capable LLMs produce noisier examples, especially for rare or highly specialized concepts.
- The autoregressive generation approach is slower at inference than bi-encoder retrieval methods. For applications requiring sub-10ms linking of thousands of mentions, a bi-encoder with SynCABEL-augmented training may be more practical.
- Guided decoding ties the model to a fixed KB version. Adding new concepts requires rebuilding the trie and ideally re-fine-tuning (or at minimum, expanding the trie and accepting lower accuracy on new entries).
- Evaluation via exact concept ID match underestimates real-world accuracy. The LLM-as-a-judge protocol helps but introduces its own noise. Plan for both metrics in any serious evaluation.

## Reference

**Paper:** [SynCABEL: Synthetic Contextualized Augmentation for Biomedical Entity Linking](https://arxiv.org/abs/2601.19667v1) (Remaki et al., 2026). Focus on Section 3 for the synthetic generation prompt structure, Section 4 for the adaptive concept representation and guided decoding details, and Table 2 for benchmark results showing +9 point gains on unseen entities.