---
name: "optimal-turkish-subword-strategies"
description: "Design and evaluate subword tokenizers for Turkish and other morphologically rich languages (MRLs) using the vocabulary-corpus-success triad framework. Covers tokenizer family selection (BPE, WordPiece, Unigram, morphology-level, character-level), vocabulary sizing, corpus coupling, and morphology-aware diagnostic evaluation. Use when: 'build a Turkish tokenizer', 'evaluate my tokenizer for agglutinative languages', 'choose vocabulary size for Turkish BERT', 'diagnose tokenizer morphological quality', 'compare BPE vs WordPiece for Turkish', 'optimize subword segmentation for MRLs'."
---

# Optimal Turkish Subword Strategies

This skill enables Claude to guide practitioners through designing, training, and evaluating subword tokenizers specifically optimized for Turkish and other morphologically rich languages (MRLs). Drawing from the "subwords manifest" framework (Altinok, 2026), it applies the vocabulary-corpus-success triad: jointly varying vocabulary size, tokenizer training corpus size, and tokenizer family to find optimal configurations. The skill also implements a morphology-aware diagnostic toolkit that goes beyond fertility and token counts to measure boundary-level F1, lemma atomicity, over/under-segmentation, and affix coverage -- connecting intrinsic tokenizer quality to downstream NLP performance.

## When to Use

- When the user needs to **train a subword tokenizer** for Turkish or another agglutinative language (Finnish, Hungarian, Korean, Kazakh, etc.)
- When choosing between **BPE, WordPiece, Unigram, morphology-aware, or character-level** tokenization for an MRL
- When deciding on **vocabulary size** (8K--64K+) for a Turkish language model and needs evidence-based guidance
- When the user wants to **diagnose why a tokenizer underperforms** on Turkish morphology tasks
- When evaluating whether a tokenizer **preserves morpheme boundaries**, lemma integrity, or suffix coverage
- When building a **Turkish BERT, GPT, or encoder model** and needs to make the tokenization design choice
- When comparing a custom tokenizer against established baselines like BERTurk

## Key Technique

### The Vocabulary-Corpus-Success Triad

Prior tokenizer studies typically fix the training corpus and sweep vocabulary size, or compare tokenizer families without controlling for data. This framework treats tokenizer design as a three-variable optimization: **(1) vocabulary size**, **(2) tokenizer training corpus size**, and **(3) tokenizer family**. All three must be jointly varied to draw valid conclusions. A 32K vocabulary trained on 1GB of text behaves fundamentally differently than one trained on 80GB, even with the same algorithm.

### Morphology-Aware Diagnostics

Standard tokenizer evaluation uses coarse metrics like fertility (average subwords per word) and unknown-token rate. These miss what matters for agglutinative languages: whether the tokenizer respects morpheme boundaries. The diagnostic toolkit introduced here measures:

- **Boundary micro/macro F1**: Does the tokenizer split words at actual morpheme boundaries? Computed by treating predicted subword boundaries as a classification problem against gold morphological analysis.
- **Lemma atomicity**: Is the lemma (root word) kept as a single token, or shattered into pieces? Measured as the single-token rate for lemmas.
- **Over-segmentation index**: Ratio of predicted subwords to gold morphemes -- values >> 1.0 mean the tokenizer splits too aggressively.
- **Under-segmentation index**: The inverse -- values >> 1.0 mean the tokenizer merges morphemes that should be separate.
- **Affix coverage and atomicity**: What fraction of common Turkish suffixes appear as standalone vocabulary items vs. being split or merged?

### Why This Matters for Downstream Tasks

Character-level tokenizers achieve near-perfect morphological fidelity but struggle on semantic tasks (NLI, STS) because they produce very long sequences that dilute contextual signal. Word-level tokenizers fragment in MRLs because productive agglutination creates massive effective vocabularies. Subword tokenizers (BPE/WordPiece) offer the best overall trade-off, but their quality depends critically on vocabulary size and training data volume. Morphology-aware tokenizers can outperform on syntax-heavy tasks (POS, dependency parsing) but may not help on pure semantic tasks.

## Step-by-Step Workflow

### 1. Characterize the Target Language's Morphology

Determine the morphological complexity: is the language agglutinative (Turkish, Finnish), fusional (Russian, German), or polysynthetic (Inuktitut)? For Turkish, identify that productive suffixation can generate thousands of surface forms from a single lemma (e.g., "ev" -> "evlerinizden" = ev+ler+iniz+den).

### 2. Prepare a Morphologically Annotated Gold Standard

Obtain or build a gold morphological analysis dataset for evaluation. For Turkish, use the dataset at `turkish-nlp-suite/turkish-morph-analysis` on Hugging Face, or generate analyses using Zeyrek (Python) or the BOUN Treebank. Split into lemma, inflected common nouns, inflected verbs, and mixed-category subsets.

### 3. Select Tokenizer Families to Compare

Train at minimum three tokenizer variants under matched conditions:
- **BPE** (via SentencePiece or HuggingFace tokenizers): the most common baseline
- **WordPiece** (via HuggingFace tokenizers): used by BERT-family models
- **Morphology-aware**: segment using Zeyrek/spaCy Turkish first, then apply subword on each morpheme
- **Character-level baseline**: each character is a token with `##` continuation markers

### 4. Design a Vocabulary-Corpus Grid

Create a systematic grid varying both dimensions:
- **Vocabulary sizes**: 8K, 16K, 32K, 48K, 64K
- **Corpus sizes**: 1GB, 5GB, 20GB, 50GB+ (subsets of OSCAR Turkish or similar)
- Train each (vocab_size, corpus_size, tokenizer_family) combination

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

vocab_sizes = [8_000, 16_000, 32_000, 48_000, 64_000]
corpus_files = {
    "1gb": ["turkish_1gb.txt"],
    "5gb": ["turkish_5gb.txt"],
    "20gb": ["turkish_20gb.txt"],
}

for vsize in vocab_sizes:
    for corpus_name, files in corpus_files.items():
        tokenizer = Tokenizer(models.WordPiece(unk_token="[UNK]"))
        tokenizer.pre_tokenizer = pre_tokenizers.Whitespace()
        trainer = trainers.WordPieceTrainer(
            vocab_size=vsize,
            special_tokens=["[UNK]", "[CLS]", "[SEP]", "[PAD]", "[MASK]"]
        )
        tokenizer.train(files, trainer)
        tokenizer.save(f"tokenizer_wp_{vsize}_{corpus_name}.json")
```

### 5. Compute Morphology-Aware Diagnostics

For each trained tokenizer, run the full diagnostic suite against the gold morphological dataset:

```python
def compute_boundary_f1(predicted_boundaries, gold_boundaries):
    """Boundary-level micro F1 for a single word."""
    tp = len(predicted_boundaries & gold_boundaries)
    fp = len(predicted_boundaries - gold_boundaries)
    fn = len(gold_boundaries - predicted_boundaries)
    precision = tp / (tp + fp + 1e-8)
    recall = tp / (tp + fn + 1e-8)
    return 2 * precision * recall / (precision + recall + 1e-8)

def over_segmentation_index(n_predicted, n_gold):
    """Ratio of predicted subwords to gold morphemes. >1.0 = over-segmented."""
    return n_predicted / (n_gold + 1e-8)

def lemma_atomicity(tokenizer, lemmas):
    """Fraction of lemmas encoded as a single token."""
    single = sum(1 for l in lemmas if len(tokenizer.encode(l).tokens) == 1)
    return single / len(lemmas)

def affix_coverage(tokenizer_vocab, frequent_suffixes):
    """Fraction of common suffixes present as standalone vocab items."""
    return sum(1 for s in frequent_suffixes if s in tokenizer_vocab) / len(frequent_suffixes)
```

### 6. Run Downstream Evaluation

Evaluate on both semantic and syntactic tasks to capture the full trade-off:
- **Semantic**: TrGLUE (SST-2 sentiment, MNLI NLI, STS-B similarity), Turkish WikiNER
- **Syntactic**: BOUN Treebank (POS accuracy, UAS/LAS for dependencies, morphological feature accuracy)

Use matched model architectures (same hidden size, layers, training budget) across all tokenizer variants.

### 7. Analyze the Triad Interactions

Plot heatmaps of (vocab_size x corpus_size) for each metric. Look for:
- The point where larger vocabulary stops helping (diminishing returns)
- The minimum corpus size needed for a given vocabulary to be well-estimated
- Whether morphology-aware tokenizers close the gap at smaller vocab/corpus sizes

### 8. Select the Optimal Configuration

Choose the tokenizer that maximizes your priority task while maintaining acceptable morphological quality. Use boundary F1 > 0.7 and over-segmentation index < 2.0 as minimum morphological quality thresholds.

### 9. Validate Against Established Baselines

Compare your chosen tokenizer against BERTurk's tokenizer (32K WordPiece). If your configuration does not outperform on at least one axis (morphological fidelity or downstream task), reconsider.

### 10. Package and Document

Save the tokenizer with its training configuration, diagnostic scores, and downstream results. Use HuggingFace tokenizers format for interoperability.

## Concrete Examples

**Example 1: Choosing Vocabulary Size for a Turkish Sentiment Model**

User: "I'm training a Turkish BERT for sentiment analysis. Should I use 32K or 64K vocabulary?"

Approach:
1. Check the training corpus size. If < 5GB, a 64K vocabulary will have many under-trained tokens -- prefer 32K.
2. Train both 32K and 64K WordPiece tokenizers on the same corpus.
3. Compute fertility on a held-out set. Turkish typically needs fertility 1.5--2.5 for good performance. If 64K gives fertility < 1.3, it is under-segmenting (keeping very long tokens as single units that won't generalize).
4. Compute lemma atomicity. If 32K keeps > 80% of frequent lemmas atomic, it is sufficient.
5. For sentiment (a semantic task), the 32K tokenizer likely wins because it produces shorter sequences with denser semantic content per position.

Output:
```
Recommendation: Use 32K vocabulary.
- Fertility: 32K=1.87, 64K=1.41
- Lemma atomicity: 32K=0.82, 64K=0.91
- SST-2 accuracy: 32K=86.1%, 64K=85.4%
- 32K produces better sequence-level representations for sentiment.
  The slight atomicity advantage of 64K does not compensate.
```

**Example 2: Diagnosing Poor NER Performance on Turkish**

User: "My Turkish NER model gets F1=0.55. The English version of the same architecture gets 0.89. What's wrong?"

Approach:
1. Check the tokenizer's over-segmentation index on entity-heavy text. Turkish entities with suffixes (e.g., "Istanbul'dan" = from Istanbul) often get split incorrectly.
2. Compute boundary F1 against gold morphological analysis -- if < 0.5, the tokenizer is destroying morpheme structure.
3. Check affix coverage: Turkish case suffixes (-dan, -den, -da, -de, -nin, -nun) must be in the vocabulary as standalone tokens for NER to work.
4. If affix coverage < 60%, retrain with larger vocabulary or use morphology-aware pre-segmentation.

Output:
```
Diagnosis: Tokenizer over-segments entity suffixes.
- Over-segmentation index: 3.2 (target: < 2.0)
- Boundary F1: 0.38 (target: > 0.7)
- Case suffix coverage: 4/12 common suffixes in vocab (33%)

Fix: Retrain tokenizer with 48K vocab on >= 10GB corpus,
or apply Zeyrek morphological pre-segmentation before BPE.
Expected improvement: F1 0.55 -> ~0.72
```

**Example 3: Morphology-Aware vs. Standard Tokenizer for Dependency Parsing**

User: "Should I use a morphology-aware tokenizer for my Turkish dependency parser?"

Approach:
1. Train two tokenizers: standard 32K WordPiece and morphology-aware (Zeyrek segmentation + WordPiece on morphemes).
2. Evaluate on BOUN Treebank: POS accuracy, UAS, LAS, morphological feature micro-accuracy.
3. Character-level models achieve near-ceiling morphological accuracy (~96%) but low UAS/LAS (~65/57) due to long sequences overwhelming the attention mechanism.
4. Morphology-aware tokenizers typically improve POS (+1-2%) and morphological accuracy (+5-15%) over plain WordPiece while maintaining comparable UAS/LAS.

Output:
```
Results on BOUN Treebank:
                    POS     UAS     LAS     Morph-Acc
WordPiece-32K      92.6    81.5    74.6    30.8
Morph-aware-32K    93.8    80.9    74.1    45.2
Character-level    91.6    65.2    57.2    96.2

Recommendation: Use morphology-aware tokenizer.
+15pp morphological accuracy with minimal syntax cost.
Character-level is only justified if morphological fidelity
is the sole objective.
```

## Best Practices

**Do:**
- Always jointly vary vocabulary size AND training corpus size -- never sweep one while holding the other fixed
- Compute boundary F1 and lemma atomicity as primary intrinsic metrics, not just fertility
- Use Zeyrek (`pip install zeyrek`) for gold morphological analysis of Turkish text
- Test on both semantic (NLI, sentiment) and syntactic (POS, parsing) tasks -- tokenizers that excel on one may fail on the other
- Set minimum morphological quality thresholds (boundary F1 > 0.7, over-segmentation < 2.0) before optimizing for downstream performance

**Avoid:**
- Using fertility alone as a tokenizer quality metric -- it tells you nothing about morphological fidelity
- Training a 64K+ vocabulary on < 5GB of text -- the long tail of tokens will be severely under-estimated
- Assuming character-level tokenization "solves" agglutination -- it achieves high morphological scores but fails on semantic and syntactic tasks due to sequence length explosion
- Using word-level tokenization for Turkish -- productive agglutination creates effectively infinite vocabulary, leading to massive OOV rates and NER F1 as low as 0.5

## Error Handling

| Problem | Symptom | Solution |
|---------|---------|----------|
| Vocabulary too large for corpus | Many single-occurrence tokens, fertility < 1.2 | Reduce vocab size or increase training corpus |
| Vocabulary too small | Fertility > 3.0, high continuation rate | Increase vocab size to at least 32K |
| Morphological pre-segmentation errors | Boundary F1 drops despite morph-aware tokenizer | Validate Zeyrek/spaCy output quality; filter to high-confidence analyses |
| Gold standard mismatch | Metrics look unreasonably low | Ensure gold morphological analysis matches the text domain (news vs. social media) |
| Sequence length overflow | OOM errors during training | Character-level and very small vocabularies produce long sequences; increase max_length or switch to subword |

## Limitations

- The framework is most thoroughly validated on **Turkish**. While the principles (triad optimization, morphology-aware diagnostics) transfer to other MRLs, specific vocabulary size recommendations may not.
- Morphology-aware tokenization requires a **high-quality morphological analyzer** (Zeyrek for Turkish). Languages without mature analyzers cannot use the morph-aware pipeline directly.
- The diagnostic toolkit requires **gold morphological annotations**, which are expensive to produce for new languages or domains.
- Results are based on **BERT-scale models** (110M parameters). Scaling laws for tokenizer choice at larger model sizes (1B+) may differ.
- Character-level findings assume standard Transformer architectures. Architectures designed for long sequences (e.g., Mamba, RWKV) may change the character-level trade-off.

## Reference

**Paper**: Altinok, D. (2026). "Optimal Turkish Subword Strategies at Scale: Systematic Evaluation of Data, Vocabulary, Morphology Interplay." arXiv:2602.06942v1. [https://arxiv.org/abs/2602.06942v1](https://arxiv.org/abs/2602.06942v1)

Look for: Section 4 (morphology-aware diagnostic toolkit definitions), Section 6 (WordPiece vocabulary-corpus grid results), and Section 7 (downstream evaluation tables).

**Code & Data**:
- Evaluation code and tokenizer pipelines: [github.com/turkish-nlp-suite/Turkish-subwords-research](https://github.com/turkish-nlp-suite/Turkish-subwords-research)
- Pretrained tokenizers and models: [huggingface.co/collections/turkish-nlp-suite/turkish-subwords-research](https://huggingface.co/collections/turkish-nlp-suite/turkish-subwords-research)
- Morphological analysis dataset: [huggingface.co/datasets/turkish-nlp-suite/turkish-morph-analysis](https://huggingface.co/datasets/turkish-nlp-suite/turkish-morph-analysis)