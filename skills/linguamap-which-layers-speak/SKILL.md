---
name: "linguamap-which-layers-speak"
description: "Diagnose and fix multilingual LLM failures using layer-localized analysis and selective fine-tuning. Use when: 'my model responds in English instead of French', 'fix language consistency in multilingual LLM', 'which layers control language output', 'selective layer fine-tuning for multilingual', 'logit lens language analysis', 'LLM answers in wrong language'"
---

# LinguaMap: Layer-Localized Multilingual Diagnosis and Selective Fine-Tuning

This skill enables Claude to diagnose why an LLM produces responses in the wrong language and prescribe a minimal, targeted fix. It applies the LinguaMap technique: use logit lens probing and cross-lingual similarity to identify the three-phase internal structure of transformer layers (early semantic alignment, middle task reasoning, late language generation), then selectively fine-tune only the final 3-5% of parameters responsible for language control -- achieving 98%+ language consistency without sacrificing task accuracy.

## When to Use

- When a user reports their multilingual LLM responds in the wrong language (e.g., prompted in Japanese but answers in English)
- When building a fine-tuning pipeline for multilingual models and wanting to minimize compute by targeting only language-critical layers
- When diagnosing whether a multilingual failure is a **task accuracy** problem or a **language consistency** problem
- When implementing logit lens or hidden-state probing to understand what different transformer layers encode
- When a user needs to evaluate multilingual model behavior across code-switched, monolingual, and bilingual prompt scenarios
- When adapting a pretrained English-dominant model (Qwen, Bloom, LLaMA, etc.) to reliably output in a target language

## Key Technique

LLMs have a three-phase internal structure for multilingual processing. **Early layers** (roughly the first 10-15% of layers) align inputs from any language into a shared semantic space -- hidden states for equivalent sentences in different languages become nearly identical. **Middle layers** (the bulk, ~15-85%) perform task reasoning in this language-agnostic space, maintaining cross-lingual cosine similarity above 0.97. **Late layers** (the final 5-15%) break this alignment and drive language-specific token generation -- this is where the model "decides" which language to output.

Two distinct failure modes exist. The **multilingual transfer bottleneck** occurs when the model responds in the correct language but gets the task wrong (semantic understanding failed to transfer). The **language consistency bottleneck** occurs when the model gets the task right but responds in English instead of the target language (language generation layers default to the dominant pretraining language). The second failure is far more common and far cheaper to fix.

The fix: freeze all layers except the final ones responsible for language control, then fine-tune only those layers on language-specific instruction data. For Bloom-7.1B this means the last 1 layer; for Qwen-3-32B the last 2-3 layers. This updates only 3-5% of parameters, uses a fraction of the GPU hours of full fine-tuning, and achieves statistically identical language consistency (98-100%) to full-scope fine-tuning.

## Step-by-Step Workflow

### Phase 1: Diagnose the Failure Mode

1. **Classify the failure type.** Run the model on 50-100 test prompts in the target language. For each response, label two binary attributes: (a) is the task answer correct? (b) is the response in the correct language? If mostly correct-task-wrong-language, you have a language consistency bottleneck (proceed to selective fine-tuning). If wrong-task-correct-language, you have a transfer bottleneck (need broader fine-tuning or different data).

2. **Design a four-scenario evaluation set.** Create test prompts in four configurations to isolate the failure:
   - **Monolingual Direct**: Instruction and content entirely in the target language
   - **Code-Switched**: Instruction in English, question content in the target language
   - **Bilingual Answer**: Correct answer options presented in both target language and English
   - **English Distractor**: Include plausible but incorrect English answer options alongside correct target-language answers

3. **Run logit lens probing to locate language emergence.** At each transformer layer `l`, project the hidden state through the model's unembedding matrix to get pseudo-logits: `z_t^(l) = U @ h^(l)`. Decode the top-k tokens via argmax, then run language detection (e.g., `langdetect` or `fasttext`) on the decoded text. Plot the probability of the target language vs. English across layers. The crossover point where target language probability exceeds English marks the onset of language-specific generation.

4. **Compute cross-lingual hidden-state similarity.** For parallel sentence pairs (same meaning, different languages), extract mean-pooled hidden states at each layer. Compute cosine similarity: `sim(l) = cos(h_bar_l^(lang_A), h_bar_l^(lang_B))`. The layer where similarity drops sharply from the middle-layer plateau (~0.97-0.99) to divergent values (<0.8) marks the boundary between shared reasoning and language-specific generation.

### Phase 2: Selective Fine-Tuning

5. **Determine the fine-tuning boundary.** Use the two convergence indicators from steps 3-4 to identify the first "language-critical" layer. For most models, this is roughly the last 5-15% of layers (e.g., layers 55-63 of a 64-layer model, or layer 30 of a 30-layer model).

6. **Prepare language-specific training data.** Collect 500 examples per subject/domain per target language. Format as instruction-response pairs where the instruction is in the target language and the response must also be in the target language. Use an 80/20 train/validation split.

7. **Freeze all layers except the identified final layers.** In your training script, iterate over model parameters and set `requires_grad = False` for all layers before the boundary. Only the final 1-3 layers (plus the language modeling head) should remain trainable.

8. **Fine-tune with conservative hyperparameters.** Use AdamW optimizer, learning rate 1e-5, OneCycleLR scheduler, batch size 16, for 1-3 epochs. Monitor validation loss and language consistency on the four-scenario eval set after each epoch. Stop when language consistency exceeds 98%.

9. **Validate across all four scenarios.** Re-run the four-scenario evaluation from step 2. Confirm that: (a) language consistency is 98%+ across all scenarios, (b) task accuracy has not degraded compared to the base model, (c) the model handles code-switched inputs without reverting to English.

10. **Compare against full fine-tuning baseline (optional).** If compute allows, run full fine-tuning on the same data to verify that selective fine-tuning matches its performance. This validates the layer-localization hypothesis for your specific model.

## Concrete Examples

**Example 1: Diagnosing a Japanese language consistency failure in Qwen**

User: "My Qwen-3-32B model answers MMLU questions correctly but always responds in English even when I prompt in Japanese. How do I fix this?"

Approach:
1. Confirm this is a language consistency bottleneck (correct task, wrong language) -- not a transfer bottleneck
2. Run logit lens analysis across all 64 layers. Expect to see English dominating token predictions until very late layers, with Japanese never surpassing English probability
3. Check cross-lingual similarity: English-Japanese hidden states will remain aligned (>0.97) through layer ~55, then diverge -- but diverge toward English output
4. Selectively fine-tune layers 62-64 (last 2-3 layers, ~3% of parameters) on 500 Japanese instruction-response pairs per MMLU subject
5. Use AdamW, lr=1e-5, batch_size=16, 2 epochs

Output:
```python
# Freeze all layers except the last 3
for name, param in model.named_parameters():
    param.requires_grad = False

for name, param in model.named_parameters():
    if any(f"layers.{i}." in name for i in range(61, 64)):
        param.requires_grad = True
    if "lm_head" in name:
        param.requires_grad = True

trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
total = sum(p.numel() for p in model.parameters())
print(f"Trainable: {trainable/1e9:.1f}B / {total/1e9:.1f}B ({100*trainable/total:.1f}%)")
# Expected: ~1.5B / 32B (4.7%)
```

Result: Language consistency jumps from ~35% to 99%+ for Japanese, task accuracy unchanged.

**Example 2: Implementing logit lens language probing**

User: "How do I implement the logit lens analysis from LinguaMap to see which layers in my model produce which language?"

Approach:
1. Hook into each transformer layer to capture hidden states
2. Project each hidden state through the unembedding matrix
3. Decode top tokens and run language detection
4. Plot language probabilities across layers

Output:
```python
import torch
from langdetect import detect_langs
from transformers import AutoModelForCausalLM, AutoTokenizer

def linguamap_logit_lens(model, tokenizer, input_text, device="cuda"):
    """Track language probability at each transformer layer."""
    inputs = tokenizer(input_text, return_tensors="pt").to(device)
    hidden_states = []

    # Forward pass capturing all hidden states
    with torch.no_grad():
        outputs = model(**inputs, output_hidden_states=True)

    unembed = model.lm_head.weight  # [vocab_size, hidden_dim]
    lang_probs_per_layer = []

    for layer_idx, h in enumerate(outputs.hidden_states):
        # Project hidden state to vocabulary space
        logits = h[:, -1, :] @ unembed.T  # pseudo-logits at last position
        top_tokens = torch.topk(logits, k=20).indices[0]
        decoded = tokenizer.decode(top_tokens)

        # Detect language distribution
        try:
            langs = detect_langs(decoded)
            lang_dict = {str(l.lang): l.prob for l in langs}
        except:
            lang_dict = {"unknown": 1.0}

        lang_probs_per_layer.append({
            "layer": layer_idx,
            "langs": lang_dict,
            "top_tokens": tokenizer.decode(top_tokens[:5])
        })

    return lang_probs_per_layer

# Usage: find the language crossover layer
results = linguamap_logit_lens(model, tokenizer, "日本の首都はどこですか？")
for r in results:
    ja_prob = r["langs"].get("ja", 0)
    en_prob = r["langs"].get("en", 0)
    print(f"Layer {r['layer']:2d}: ja={ja_prob:.2f} en={en_prob:.2f} | {r['top_tokens']}")
```

**Example 3: Building the four-scenario evaluation protocol**

User: "How do I properly evaluate multilingual performance? My model seems fine on simple tests but fails in production."

Approach:
1. Build four prompt variants for each test question
2. Measure both task accuracy and language consistency separately
3. Identify which scenario triggers failure

Output:
```python
def build_four_scenarios(question, choices, answer, target_lang="fr"):
    """Generate four evaluation scenarios for a single question."""
    return {
        "monolingual_direct": {
            # Everything in target language
            "prompt": f"Répondez à la question suivante en français.\n\n{question}\n{choices}",
            "expected_lang": target_lang
        },
        "code_switched": {
            # Instruction in English, content in target language
            "prompt": f"Answer the following question in French.\n\n{question}\n{choices}",
            "expected_lang": target_lang
        },
        "bilingual_answer": {
            # Correct answer available in both languages
            "prompt": f"Répondez en français.\n\n{question}\n{choices}\n(English: {answer})",
            "expected_lang": target_lang
        },
        "english_distractor": {
            # Include tempting English wrong answers
            "prompt": f"Répondez en français.\n\n{question}\n{choices}\nAlternatively: {wrong_en_answer}",
            "expected_lang": target_lang
        }
    }

def evaluate_multilingual(model, scenarios):
    """Score both task accuracy and language consistency."""
    results = {"task_correct": 0, "lang_correct": 0, "total": 0}
    for scenario in scenarios:
        response = model.generate(scenario["prompt"])
        detected_lang = detect(response)
        results["total"] += 1
        results["task_correct"] += int(is_answer_correct(response, scenario))
        results["lang_correct"] += int(detected_lang == scenario["expected_lang"])

    return {
        "task_accuracy": results["task_correct"] / results["total"],
        "lang_consistency": results["lang_correct"] / results["total"]
    }
```

## Best Practices

- **Do:** Always diagnose before fine-tuning. Run logit lens analysis first to confirm the failure is in late layers (language consistency bottleneck). If the problem is in middle layers (task reasoning), selective late-layer tuning won't help.
- **Do:** Include all four prompt scenarios in evaluation. Models often pass monolingual tests but fail on code-switched inputs -- production traffic is rarely purely monolingual.
- **Do:** Keep training data small and balanced. 500 examples per subject per language is sufficient. More data does not meaningfully improve results for this targeted intervention.
- **Do:** Fine-tune the LM head alongside the final layers. The language modeling head directly maps hidden states to vocabulary tokens and is critical for language control.
- **Avoid:** Fine-tuning early or middle layers for language consistency problems. This wastes compute and risks degrading the shared semantic representations that enable cross-lingual transfer.
- **Avoid:** Using more than 3 epochs. The selective fine-tuning converges fast; over-training causes the model to lose task accuracy while the language consistency gain plateaus after epoch 1-2.

## Error Handling

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Language consistency improves but task accuracy drops | Fine-tuning too many layers or too many epochs | Reduce to last 1 layer only; reduce to 1 epoch |
| Logit lens shows no clear language crossover | Model may lack target language in pretraining vocabulary | Check tokenizer coverage for target language; this method won't work if the language is severely underrepresented |
| Cross-lingual similarity is low even in early layers | Input sentences are not semantically parallel | Use verified parallel corpora (e.g., FLORES, Tatoeba) for probing |
| Fine-tuning has no effect on language consistency | Wrong layers identified; failure may be in attention patterns not captured by logit lens | Try expanding the fine-tuning window by 2-3 more layers; consider full attention layer analysis |
| Works for some languages but not others | Typologically distant languages (e.g., Arabic, Hindi) diverge earlier | Run per-language logit lens analysis; distant languages may need a wider fine-tuning window |

## Limitations

- **Requires access to model internals.** This technique needs hidden state access at every layer, making it inapplicable to API-only models (GPT-4, Claude, Gemini). It works only with open-weight models where you can hook into intermediate layers.
- **Language must exist in pretraining data.** If the target language has minimal representation in the model's pretraining corpus, the late layers have no language-specific patterns to amplify. The method fixes *consistency*, not *capability*.
- **Validated on specific architectures.** Results are demonstrated on Qwen-3-32B and Bloom-7.1B. The three-phase structure likely generalizes to other decoder-only transformers but exact layer boundaries will differ.
- **Does not fix vocabulary gaps.** If the tokenizer fragments target-language text into many subword tokens (poor coverage), language quality will be limited regardless of fine-tuning.
- **Single-language output only.** The method optimizes for consistent output in one target language. Intentional code-switching or mixed-language output requires a different approach.

## Reference

**Paper:** [LinguaMap: Which Layers of LLMs Speak Your Language and How to Tune Them?](https://arxiv.org/abs/2601.20009v1) (Ben Tamo et al., 2026)

**Key insight to look for:** Section 4's logit lens and cross-lingual similarity plots that visually demonstrate the three-phase structure, and Section 5's ablation showing that fine-tuning only the last 1-3 layers matches full fine-tuning at 3-5% of the parameter cost.