---
name: "eurollm-22b-technical-report"
description: >
  Build multilingual LLM training pipelines using the EuroLLM methodology: balanced tokenizer design for 35+ languages,
  multi-phase curriculum training, quality-tiered data filtering with EuroFilter, and instruction tuning with best-of-N
  selection. Use when the user asks to "train a multilingual model," "build a European language pipeline,"
  "design a multilingual tokenizer," "filter multilingual web data," "create a curriculum training schedule,"
  or "instruction-tune a model for multiple languages."
---

This skill enables Claude to design and implement multilingual LLM training pipelines following the EuroLLM-22B methodology. EuroLLM-22B is a 22.6B-parameter dense transformer trained on ~4 trillion tokens across 35 languages (all 24 EU official languages plus 11 additional), using a three-phase curriculum strategy, a 128K-vocabulary BPE tokenizer balanced for cross-lingual fertility, quality-tiered data filtering via a custom EuroFilter classifier, and instruction tuning with reward-model-guided best-of-N selection across ~10.6M multilingual examples.

## When to Use

- When the user wants to train or fine-tune a multilingual language model covering European (or other diverse) languages
- When designing a BPE tokenizer that balances vocabulary allocation across high-resource and low-resource languages
- When building a multilingual data pipeline with quality filtering, deduplication, and language identification
- When implementing a multi-phase curriculum training schedule (warmup -> plateau -> annealing -> final decay)
- When curating instruction-tuning datasets that span dozens of languages with reward-model selection
- When filtering parallel corpora using CometKiwi or Bicleaner quality thresholds
- When extending a model's context window mid-training via RoPE theta scaling
- When deciding data mixing ratios between English, multilingual, code, and math sources

## Key Technique

EuroLLM's core insight is **quality-tiered curriculum training** combined with **balanced multilingual data curation**. Rather than training on a single static data mix, the pipeline divides ~4T tokens across three phases of progressively increasing data quality. Phase 1 (3.6T tokens) uses the broadest data at baseline quality. Phase 2 (400B tokens) anneals the learning rate while shifting the mix toward higher-quality multilingual content. Phase 3 (100B tokens) decays the learning rate to zero using only the highest-quality data and simultaneously extends the context window from 4K to 32K tokens by scaling RoPE theta from 10^4 to 10^6.

The **EuroFilter** classifier is central to this quality tiering. It is a fine-tuned mDeBERTa model trained on machine-translated FineWeb-Edu labels that assigns educational quality scores (0-5) to multilingual web documents. Combined with KenLM perplexity filtering, Bicleaner thresholds (0.5-0.6) for parallel data, CometKiwi-22 scores (>=0.7) for translations, and heuristic filters (minimum 200 characters, removal of boilerplate patterns), this creates three distinct quality tiers that map directly to training phases.

For instruction tuning, EuroLLM uses a **best-of-N response selection** strategy: public instruction datasets are re-generated using multiple strong models (DeepSeek-v3, Qwen2.5, Llama3.1-70B, Tulu-3), and the best response is selected using the Skywork-Gemma2-27B reward model. This produces a 10.6M-example multilingual SFT corpus (EuroBlocks) that is trained for 55 epochs with cosine learning rate decay at 1e-5.

## Step-by-Step Workflow

1. **Define language coverage and resource tiers.** Classify target languages into high-resource (English, German, French, Spanish, Italian), medium-resource (Dutch, Polish, Portuguese, Romanian, Czech), and low-resource (Maltese, Irish, Latvian, Estonian, etc.). This determines upsampling ratios and data sourcing strategy.

2. **Train a balanced BPE tokenizer with 128K vocabulary.** Sample training text from all target languages with explicit oversampling of low-resource languages. Measure cross-lingual fertility (tokens-per-word) and iterate until low-resource languages achieve fertility within 1.5x of English. Use byte-level BPE fallback to guarantee full Unicode coverage.

3. **Assemble and filter the pretraining corpus.** Source web data from FineWeb-Edu (English, score >2), Nemotron-CC (high-quality split), RedPajama-v2 (high-resource EU), HPLT/MADLAD-400/CulturaX/mC4 (medium and low-resource). Source parallel data from Europarl, ParaCrawl, CCMatrix, WikiMatrix, OPUS. Source code from The Stack/Algebraic-Stack and math from FineMath/GSM8k/MATH.

4. **Apply the EuroFilter quality pipeline.** For each multilingual document: (a) run language identification, (b) apply heuristic filters (remove documents <200 chars, >40% uppercase, containing "lorem ipsum" or excessive JavaScript/curly brackets), (c) run KenLM perplexity scoring per language, (d) score educational quality 0-5 with fine-tuned mDeBERTa, (e) partition into three quality tiers (low/medium/high) for curriculum phases.

5. **Filter parallel data with translation quality metrics.** Apply Bicleaner with threshold 0.5 (0.6 for Portuguese) and CometKiwi-22 with threshold 0.7. Format as bidirectional pairs (en->xx and xx->en) to support both translation directions.

6. **Configure the three-phase training schedule.** Phase 1: 3.6T tokens with 10% linear warmup to peak LR (3e-4), then constant plateau. Data mix: ~60% English, ~20% multilingual, ~20% code/math. Phase 2: 400B tokens, linear LR decay to 10% of peak (3e-5). Shift mix toward higher-quality multilingual and reduce web data proportion. Phase 3: 100B tokens, LR decays to zero. Use only highest-quality tier. Upsample long-context data (30B tokens books + 30B tokens code) and scale RoPE theta to 1e6 for 32K context.

7. **Set architecture and training hyperparameters.** For a 22B model: 54 layers, d_model=6144, FFN=16384, 48 attention heads, 8 KV heads (GQA), SwiGLU activation, RMSNorm (pre-layer), no tied embeddings. Train in bfloat16 on Megatron-LM with tensor and pipeline parallelism.

8. **Curate the instruction-tuning dataset (EuroBlocks).** Collect seed instructions from OpenHermes, Aya Expanse, HelpSteer-2, Magpie. For each instruction, generate N responses using 3-4 strong models. Score all responses with a reward model (e.g., Skywork-Gemma2-27B) and select the highest-scoring response. Include ~2M STEM samples. Remove chain-of-thought reasoning traces for the non-reasoning variant.

9. **Run supervised fine-tuning.** Train on the full 10.6M-example SFT corpus for 55 epochs with: max context 32768, LR=1e-5, cosine scheduler, 125 warmup steps, bfloat16 mixed precision. Use Axolotl with Liger-Kernel optimizations (fused RoPE, RMSNorm, GLU, cross-entropy).

10. **Evaluate across multilingual benchmarks.** Test on MMLU-Pro, GSM8K, MATH, ARC, HellaSwag (reasoning), IFEval (instruction following), and COMET-22 (translation). Use LLM-as-judge evaluation with 3+ judge models (mean aggregation) rather than regex-based extraction for generation tasks.

## Concrete Examples

**Example 1: Designing a multilingual data pipeline for 10 EU languages**

User: "I want to train a 7B model covering English, German, French, Spanish, Italian, Polish, Dutch, Czech, Romanian, and Portuguese. Help me design the data pipeline."

Approach:
1. Classify: English = high-resource anchor; DE/FR/ES/IT = high-resource EU; PL/NL/CS/RO/PT = medium-resource EU
2. Source web data per language: FineWeb-Edu (EN), RedPajama-v2 (DE/FR/ES/IT), HPLT + CulturaX (PL/NL/CS/RO/PT)
3. Source parallel data: Europarl (all 10), ParaCrawl (EN<->each), CCMatrix for supplementary pairs
4. Build quality filter: fine-tune mDeBERTa on FineWeb-Edu labels translated to each target language
5. Apply three-tier filtering: Tier 1 (score 0-2), Tier 2 (score 2-4), Tier 3 (score 4-5)

Output (data mix config):
```yaml
phase_1:  # 80% of total tokens
  english_web: 0.50
  eu_high_web: 0.20  # DE/FR/ES/IT at 5% each
  eu_mid_web: 0.10   # PL/NL/CS/RO/PT at 2% each
  parallel: 0.05
  code: 0.10
  math: 0.05
  quality_tier: [1, 2, 3]

phase_2:  # 15% of total tokens, annealing
  english_web: 0.35
  eu_high_web: 0.25
  eu_mid_web: 0.15
  parallel: 0.10
  code: 0.10
  math: 0.05
  quality_tier: [2, 3]

phase_3:  # 5% of total tokens, decay to zero
  english_web: 0.25
  eu_high_web: 0.25
  eu_mid_web: 0.20
  parallel: 0.10
  code: 0.10
  math: 0.05
  long_context: 0.05
  quality_tier: [3]
```

**Example 2: Implementing EuroFilter for multilingual quality scoring**

User: "How do I build a quality classifier like EuroFilter for my multilingual web data?"

Approach:
1. Start with FineWeb-Edu English annotations (educational scores 0-5)
2. Translate a representative sample (~500K documents) to each target language using a strong MT system
3. Fine-tune mDeBERTa-v3-base on translated labels for multilingual classification
4. Apply the trained classifier to score all web documents per language

Output (training script sketch):
```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
from datasets import load_dataset

# Load FineWeb-Edu scored data, translated to target languages
dataset = load_dataset("eurofilter-training-data")

model = AutoModelForSequenceClassification.from_pretrained(
    "microsoft/mdeberta-v3-base",
    num_labels=6,  # scores 0-5
    problem_type="single_label_classification"
)
tokenizer = AutoTokenizer.from_pretrained("microsoft/mdeberta-v3-base")

# Fine-tune with cross-lingual transfer
# After training, score documents:
def score_document(text: str) -> int:
    inputs = tokenizer(text, truncation=True, max_length=512, return_tensors="pt")
    logits = model(**inputs).logits
    return logits.argmax(dim=-1).item()

# Partition into quality tiers
def assign_tier(score: int) -> str:
    if score >= 4: return "high"
    if score >= 2: return "medium"
    return "low"
```

**Example 3: Best-of-N instruction tuning dataset curation**

User: "I have 50K multilingual instructions. How do I apply best-of-N selection to build a high-quality SFT dataset?"

Approach:
1. For each instruction, generate N=4 responses from diverse models
2. Score each response with a reward model
3. Select the highest-scoring response per instruction

Output (pipeline sketch):
```python
from vllm import LLM
from transformers import pipeline

# Models for response generation
generators = [
    LLM("deepseek-ai/DeepSeek-V3"),
    LLM("Qwen/Qwen2.5-72B-Instruct"),
    LLM("meta-llama/Llama-3.1-70B-Instruct"),
]
reward_model = pipeline("text-classification", model="Skywork/Skywork-Reward-Gemma-2-27B-v0.2")

def best_of_n(instruction: str, n_per_model: int = 2) -> dict:
    candidates = []
    for gen in generators:
        for _ in range(n_per_model):
            response = gen.generate(instruction, sampling_params={"temperature": 0.7})
            score = reward_model(f"{instruction}\n{response}")
            candidates.append({"response": response, "score": score})

    best = max(candidates, key=lambda x: x["score"])
    return {"instruction": instruction, "response": best["response"], "score": best["score"]}

# Apply to full dataset, then filter: keep only score >= threshold
sft_dataset = [best_of_n(inst) for inst in instructions]
```

## Best Practices

- **Do:** Measure tokenizer fertility (tokens per word) for every target language before finalizing the vocabulary. Low-resource languages with fertility >2x English will underperform significantly.
- **Do:** Use progressive quality tiering across training phases. Reserve your highest-quality data for the final annealing phase where the model has the most capacity to absorb it.
- **Do:** Include bidirectional parallel data (en->xx AND xx->en) to develop robust translation capabilities in both directions.
- **Do:** Use LLM-as-judge with multiple judge models (mean aggregation) for evaluating multilingual generation, as regex-based extraction correlates poorly with human judgment on non-English text.
- **Avoid:** Training with a single static data mix for the entire run. Curriculum-based phase transitions meaningfully improve final quality.
- **Avoid:** Extending context length from the start of training. Train at 4K context first, then scale RoPE theta and upsample long documents in the final phase -- this is cheaper and equally effective.
- **Avoid:** Including chain-of-thought reasoning traces in SFT data unless you specifically want a reasoning model. EuroLLM found that removing reasoning traces improved general instruction following.

## Error Handling

- **Language ID misclassification:** Low-resource languages with limited training data for LangID models can be misclassified. Cross-validate with multiple LangID tools (e.g., fastText + CLD3) and use agreement thresholds.
- **EuroFilter score miscalibration:** Quality scores transfer imperfectly across languages via translation. Validate calibration per language by sampling 200+ documents per tier and checking manually.
- **Degenerate parallel data:** Parallel corpora often contain misaligned pairs, duplicated source-target, or machine-translated noise. Apply both Bicleaner (alignment quality) AND CometKiwi (translation quality) filters -- using only one is insufficient.
- **Phase transition instability:** Abrupt changes in data mix between training phases can cause loss spikes. Use a short (1-2% of phase tokens) linear interpolation window at phase boundaries.
- **SFT overfitting at 55 epochs:** High epoch counts require careful monitoring. Track validation loss per language and apply early stopping per-language if a specific language degrades.

## Limitations

- The three-phase curriculum strategy requires knowing your total compute budget upfront. It is not easily adapted to open-ended continued pretraining.
- EuroFilter relies on FineWeb-Edu labels translated via MT, which introduces annotation noise proportional to MT quality for each language. Truly low-resource languages (Maltese, Irish) get the least reliable quality scores.
- The 128K vocabulary tokenizer is designed for European + select Asian languages. Extending to African, Indic, or Southeast Asian language families requires retraining the tokenizer from scratch.
- Best-of-N instruction tuning scales linearly in inference cost with N and the number of generator models. For large instruction sets (>1M), this can exceed pretraining cost.
- The methodology assumes access to hundreds of GPUs (400 H100s on MareNostrum5). Adapting the multi-phase curriculum to smaller-scale training (<100B tokens) requires recalibrating phase ratios.

## Reference

**Paper:** [EuroLLM-22B: Technical Report](https://arxiv.org/abs/2602.05879v1) (Ramos et al., 2026). Look for: Section 2 (tokenizer fertility analysis), Section 3 (three-phase data curriculum with quality tiers), Section 4 (EuroFilter classifier details), and Section 5 (best-of-N instruction tuning with EuroBlocks).