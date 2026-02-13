---
name: "domain-adaptation-synthetic-data-fine-tuning-germa"
description: "Generate difficulty-graded synthetic QA datasets from authoritative domain documents (laws, regulations, standards) and fine-tune LLMs for specialized question answering. Triggers: 'fine-tune model on legal text', 'generate synthetic QA from statutes', 'domain-adapt LLM for specialized knowledge', 'create training data from regulations', 'build legal QA dataset', 'adapt model to German law'"
---

# Domain Adaptation through Synthetic Data: Difficulty-Graded QA Generation and Fine-Tuning

This skill enables Claude to build end-to-end pipelines that adapt general-purpose LLMs to specialized domains -- particularly legal, regulatory, and compliance domains -- by generating difficulty-graded synthetic question-answer pairs directly from authoritative source documents (statutes, regulations, standards), filtering them with an automated LLM reviewer, and fine-tuning models with LoRA. The technique comes from Bashir et al. (2026), who demonstrated that difficulty-graded synthetic data dramatically outperforms naive single-prompt generation, achieving up to +39.6% accuracy improvement on German legal QA.

## When to Use

- When the user wants to fine-tune an LLM on a specialized knowledge domain (law, medicine, finance, compliance) but lacks labeled training data
- When the user asks to generate synthetic QA pairs from legal statutes, regulatory codes, technical standards, or policy documents
- When the user needs to build a domain-specific QA system grounded in authoritative texts
- When the user wants to adapt an open model (Llama, Gemma, Mistral) to answer questions about a specific body of regulations or rules
- When the user asks about creating training data from structured legal or regulatory documents without human annotators
- When the user wants to implement a multi-level difficulty curriculum for synthetic data generation

## Key Technique: Difficulty-Graded Synthetic QA Generation

The core insight is that naive single-prompt QA generation ("read this passage and produce questions") yields low-diversity, surface-level data that can actually *degrade* model performance. The paper found that standard instruction-style generation caused a -7.6 point drop on one benchmark. The fix is **difficulty-graded generation**: producing four complementary question types per source passage, each targeting a different reasoning depth.

**The four difficulty levels are:**
1. **Clause-centric (L1):** Direct comprehension -- "What does Section X state about Y?" Tests whether the model can retrieve and restate provisions accurately.
2. **Client-style paraphrase (L2):** Natural language reformulations without legal jargon or section numbers -- "Can my landlord raise rent during the first year?" Tests whether the model handles real-world phrasing.
3. **Scenario application (L3):** Fact patterns requiring the model to apply a provision -- "Alice signed a contract but the seller delivered damaged goods. What are her rights?" Tests legal reasoning.
4. **Multi-provision reasoning (L4):** Complex scenarios requiring synthesis across multiple sections. Tests cross-reference and integration ability.

**Automated quality filtering** uses an LLM reviewer that checks three criteria against the source text only: (a) the question is specific and answerable from the passage, (b) the answer is fully supported by the statute with no external assumptions, and (c) the pair is not redundant with others from the same passage. This filtering removed ~25% of generated pairs (31,777 down to 23,905) and was critical to data quality.

**Fine-tuning** uses LoRA (rank 16, alpha 32) on attention and MLP modules with AdamW_8bit, learning rate 1e-5, 7 epochs. This is parameter-efficient and runs on a single A100 in ~10 hours for an 8B model.

## Step-by-Step Workflow

1. **Collect and normalize source documents.** Parse authoritative texts into a structured format: `{document_id, section_id, section_text}`. For legal texts, extract law identifier (e.g., "BGB"), section number (e.g., "Section 433"), and full section text. Ensure each section is a self-contained unit.

2. **Design level-specific generation prompts.** Write four prompt templates (L1-L4), each instructing the generator to produce QA pairs at a specific difficulty. Enforce these constraints in every prompt:
   - Answers must cite the specific document and section
   - No external knowledge beyond the provided text
   - Output as structured JSON: `{"qa_pairs": [{"question": "...", "answer": "..."}]}`
   - Cap redundancy (e.g., max 3 pairs per level per section)

3. **Generate QA pairs per level.** For each source section, call the generator model (GPT-4 or equivalent) once per difficulty level. Pass only the section text and the level-specific prompt. Collect all outputs into a unified dataset tagged with source section and difficulty level.

4. **Apply automated LLM-based filtering.** For each generated pair, run a reviewer prompt that receives only the source section text and the QA pair. The reviewer checks three binary criteria:
   - **Answerability:** Is the question specific and answerable from this text alone?
   - **Support:** Is the answer fully grounded in the statute, with no external assumptions?
   - **Non-redundancy:** Is this pair meaningfully different from other pairs generated for this section?
   Discard any pair that fails any criterion. Expect ~20-30% rejection rate.

5. **Enforce train/test contamination control.** Assign all QA pairs from a given source section to exactly one data split. Never allow pairs from the same section to appear in both training and evaluation sets.

6. **Format the dataset for fine-tuning.** Convert filtered QA pairs into the instruction-tuning chat format expected by your target model (e.g., Llama chat template or Gemma IT format). Include a system prompt establishing the domain role.

7. **Configure LoRA fine-tuning.** Set up parameter-efficient training with these proven hyperparameters as a starting point:
   - LoRA rank: 16, alpha: 32, dropout: 0.05
   - Target modules: attention projections + MLP layers
   - Max sequence length: 2048 tokens
   - Learning rate: 1e-5 with linear scheduler, warmup ratio 0.1
   - Optimizer: AdamW_8bit, weight decay: 0.01
   - Precision: bf16
   - Epochs: 5-7

8. **Train and monitor.** Launch fine-tuning using Unsloth, PEFT, or similar frameworks. Monitor training loss for convergence. Training an 8B model on ~24K pairs takes approximately 9-10 hours on a single A100.

9. **Evaluate on held-out domain benchmarks.** Test on both open-ended QA (using an LLM judge like GPT-4 for factual correctness) and multiple-choice QA (exact accuracy). Also run general benchmarks (ARC, MMLU) to verify the model hasn't catastrophically forgotten general knowledge.

10. **Iterate on data composition.** If results are weak at a specific difficulty level, increase the proportion of that level in training data. The paper found L3 (scenario) pairs had the highest filtering rejection rate (~37%), so generating more raw L3 pairs compensates for losses.

## Concrete Examples

**Example 1: Building a German Legal QA Fine-Tuning Pipeline**

User: "I have the full text of the German Civil Code (BGB). I want to fine-tune Llama 3.1 8B to answer legal questions about it."

Approach:
1. Parse the BGB into sections -- each `Section XXX` becomes a record with `{law: "BGB", section: "Section 433", text: "..."}`
2. Write four generation prompts. For Level 2 (client-style):
```
You are a German legal data generator. Given the following statute section,
generate 2-3 questions that a non-lawyer client might naturally ask,
WITHOUT using section numbers or legal terminology in the question.
The answer MUST cite the exact section (e.g., "Gemaess Section 433 BGB...")
and be fully supported by the provided text only.

Statute: {section_text}

Output as JSON: {"qa_pairs": [{"question": "...", "answer": "..."}]}
```
3. Generate ~32K pairs across all four levels using GPT-4
4. Filter with the LLM reviewer -- expect ~24K surviving pairs
5. Fine-tune with LoRA (rank 16, alpha 32, lr 1e-5, 7 epochs)
6. Evaluate on held-out BGB QA test set

Expected improvement: +20 points on open-ended BGB QA (from ~39% to ~59% for Llama 8B).

**Example 2: Adapting to a Non-Legal Specialized Domain (Medical Guidelines)**

User: "I want to use this technique to adapt a model for answering questions about clinical practice guidelines."

Approach:
1. Normalize guidelines into sections: `{guideline: "AHA-HF-2024", section: "4.2", text: "..."}`
2. Adapt the four difficulty levels to the medical domain:
   - L1 (Guideline-centric): "What does the AHA guideline recommend for Stage C heart failure?"
   - L2 (Patient-style): "My doctor said my heart isn't pumping well enough. What treatments should I expect?"
   - L3 (Clinical scenario): "A 67-year-old with EF 35% and NYHA Class III symptoms presents to clinic..."
   - L4 (Multi-guideline reasoning): Scenarios requiring synthesis across multiple sections or guidelines
3. Adjust the reviewer criteria: replace "statute support" with "guideline support" -- answer must be grounded in the cited guideline section
4. Generate, filter, and fine-tune using the same LoRA configuration
5. Evaluate with domain-specific QA benchmarks

**Example 3: Generating the Filtering Reviewer Prompt**

User: "How do I implement the automated QA filter?"

Output -- the reviewer prompt:
```
You are a quality reviewer for domain-specific QA pairs.
Given a source passage and a generated question-answer pair,
evaluate the pair on three criteria. Respond with JSON.

Source passage:
{section_text}

Question: {question}
Answer: {answer}

Evaluate:
1. answerability: Is the question specific and answerable from this passage alone? (true/false)
2. support: Is the answer fully supported by this passage with no external assumptions or hallucinated facts? (true/false)
3. non_redundancy: Is this pair meaningfully different from these existing pairs for the same passage? (true/false)
   Existing pairs: {existing_pairs_json}

Output: {"answerability": bool, "support": bool, "non_redundancy": bool, "keep": bool}
Set "keep" to true only if ALL three criteria are true.
```

Implementation in Python:
```python
import json
from openai import OpenAI

client = OpenAI()

def filter_qa_pair(section_text: str, question: str, answer: str,
                   existing_pairs: list[dict]) -> bool:
    prompt = REVIEWER_TEMPLATE.format(
        section_text=section_text,
        question=question,
        answer=answer,
        existing_pairs_json=json.dumps(existing_pairs, ensure_ascii=False)
    )
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"},
        temperature=0.0,
    )
    result = json.loads(response.choices[0].message.content)
    return result.get("keep", False)
```

## Best Practices

- **Do:** Generate all four difficulty levels for every source section. The diversity across levels is what drives the improvement -- L1 and L2 alone are insufficient.
- **Do:** Enforce strict grounding constraints in generation prompts. Every answer must cite the specific source section. This prevents hallucination from propagating into training data.
- **Do:** Use passage-level split isolation. All QA pairs from a given source section go into the same split (train OR test, never both). This prevents data leakage.
- **Do:** Start with the paper's LoRA hyperparameters (rank 16, alpha 32, lr 1e-5) as a baseline. They were validated across two model families.
- **Avoid:** Single-prompt "standard instruction" generation. The paper showed this can actively hurt performance (up to -7.6 points). Always use difficulty-graded generation.
- **Avoid:** Skipping the filtering step. Unfiltered data contains ~25% low-quality pairs that dilute training signal. The LLM reviewer is cheap relative to the quality gain.
- **Avoid:** Fine-tuning for more than 7-10 epochs. The paper used 7 epochs; overfitting on synthetic data risks memorizing generation artifacts rather than learning domain knowledge.

## Error Handling

- **Generator produces malformed JSON:** Wrap generation calls in try/catch with retry logic. Set `response_format={"type": "json_object"}` when using OpenAI models. For open models, use constrained decoding or regex extraction as fallback.
- **Filtering rejects too many pairs (>50%):** Review your generation prompts -- they may be too unconstrained. Tighten grounding requirements or reduce the per-section pair cap. Also check that the reviewer isn't being overly strict on non-redundancy when pairs target genuinely different aspects.
- **Fine-tuned model degrades on general benchmarks:** Reduce epochs (try 3-4) or lower LoRA rank to 8. Run ARC/MMLU checks after each epoch to find the sweet spot before general knowledge degrades.
- **Low improvement on scenario-level (L3/L4) questions:** These are hardest to generate well and have the highest rejection rate (~37%). Generate 2x more raw L3/L4 pairs to compensate, or use a stronger generator model for these levels.
- **Source documents are too long for context window:** Split documents at natural boundaries (sections, articles, clauses). Each chunk should be self-contained. For cross-reference scenarios (L4), pass multiple related sections as combined context.

## Limitations

- **Requires authoritative structured source texts.** The technique works best when domain knowledge is codified in clearly delineated sections (statutes, guidelines, standards). It is less effective for unstructured knowledge domains like creative writing or informal expertise.
- **Generator quality ceiling.** The synthetic QA pairs can only be as good as the model generating them. Using a weak generator produces low-quality pairs that filtering cannot fully rescue.
- **Language and domain transfer is not automatic.** The paper validated this for German legal text. Applying to other languages or domains (medical, financial) requires adapting the difficulty levels and reviewer criteria to domain conventions.
- **Does not replace RAG for rapidly changing knowledge.** Fine-tuning bakes knowledge into weights. For regulations that change frequently, combine this approach with retrieval-augmented generation.
- **Evaluation requires domain expertise or a strong LLM judge.** Open-ended QA evaluation relies on GPT-4-level judgment. For domains where even GPT-4 lacks expertise, you need domain-expert validation of a sample.

## Reference

Bashir, A. H., Khalid, M. R., Cvejoski, K., Birr, J., & Berghaus, J. (2026). *Domain-Adaptation through Synthetic Data: Fine-Tuning Large Language Models for German Law.* arXiv:2601.14160v1. https://arxiv.org/abs/2601.14160v1

Key takeaway: Difficulty-graded QA generation (4 levels from direct comprehension to multi-provision reasoning) with LLM-based filtering produces synthetic training data that yields +12 to +40 point accuracy gains on domain QA, while naive single-prompt generation can hurt performance.