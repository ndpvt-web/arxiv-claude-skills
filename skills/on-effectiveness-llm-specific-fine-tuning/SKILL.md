---
name: "on-effectiveness-llm-specific-fine-tuning"
description: "Build and evaluate AI-generated text detectors using LLM-specific fine-tuning strategies. Covers corpus construction, per-LLM vs per-family vs generic detector training, token-level classification heads, and ensemble evaluation. Use when: 'detect AI-generated text', 'build an AI text detector', 'fine-tune a model for AI detection', 'classify text as human or AI', 'train a per-LLM detector', 'evaluate AI text detection accuracy'."
---

This skill enables Claude to guide users through building high-accuracy AI-generated text detectors using the LLM-specific fine-tuning methodology from Gromadzki et al. (2026). The core insight is that freezing an LLM backbone and training only a token-level classification head—using either a single generic corpus or per-model/per-family splits—can reach 99.6% token-level accuracy while supporting 8K+ token context windows. Claude can apply this to architect detection pipelines, construct balanced training corpora, choose between generic vs. ensemble strategies, and evaluate detector performance across multiple generator LLMs.

## When to Use

- When the user wants to build a classifier that distinguishes human-written text from AI-generated text
- When the user needs to fine-tune an existing LLM (e.g., Phi-4, Qwen, Llama) as a binary text detector
- When the user asks how to construct a balanced corpus of human and AI-generated text for detection training
- When the user wants to compare generic (one-model-fits-all) vs. per-LLM vs. per-LLM-family detection strategies
- When the user needs to evaluate a detector across multiple generator models (cross-model generalization)
- When the user is building a content moderation or academic integrity pipeline that needs AI text detection
- When the user asks about token-level vs. document-level AI text classification trade-offs

## Key Technique

The paper's core architecture takes a pretrained causal LLM, freezes its backbone entirely, and attaches a linear classification head with sigmoid activation to each token's hidden representation. Because causal LLMs encode only left-context, each token's prediction reflects everything preceding it. To incorporate global document context, the final token's representation (which sees the full input) is concatenated with every other token's representation before the classification head. This produces a per-token human/AI probability, which can be averaged across all tokens to yield a document-level score. Training uses binary cross-entropy with Adam (betas 0.9/0.999), a linear learning rate schedule with 20% warmup, and runs for 5 epochs updating only the classification layer.

The critical finding is about training data strategy. The authors test three paradigms: (1) **Generic** — one detector trained on a balanced mix from all 21 generator LLMs, (2) **Per LLM** — 21 separate detectors each specialized to one generator, aggregated via ensemble, and (3) **Per LLM Family** — 6 detectors, one per model family (Llama, Phi, Mistral, Qwen, Falcon, plus one for proprietary). Counter-intuitively, the single generic Phi-4-based detector (14B params) achieves 99.6% accuracy, decisively outperforming the Per LLM ensemble (71.3%) which requires 416B total parameters. The Per LLM Family ensemble (69B params, 80.6%) falls in between. The lesson: a well-constructed generic training corpus with balanced representation across generators and domains matters more than model specialization.

The corpus design is equally important. Human text spans 10 domains (blogs, essays, news, Reddit, tweets, academic Q&A, etc.) totaling 1B tokens. AI text (1.9B tokens) is generated using four randomized sampling presets (deterministic temp=0.0, balanced temp=0.5, creative temp=0.7, highly creative temp=1.0) to capture the full range of generation styles. The training split uses 366K samples / 100M tokens, balanced by class, by generator model, and with genre proportions matching the overall corpus distribution.

## Step-by-Step Workflow

1. **Select a base LLM for the detector backbone.** Choose a model with strong language understanding and long context support. Phi-4 (14B) was the top performer in the paper; Qwen2.5-14B and Phi-3-medium also performed well. Smaller models (3B) work but sacrifice 1-3% accuracy.

2. **Assemble human-authored text across multiple domains.** Collect text from at least 5 distinct genres (news articles, social media, academic writing, creative writing, conversational). Aim for diversity in register, length, and topic. Use established datasets like XSum, WritingPrompts, Reddit comments, and news corpora.

3. **Generate AI text using target LLMs with varied sampling parameters.** For each human text sample, prompt the target LLM(s) to produce a corresponding AI text. Use a three-message format: system instruction setting the domain context, a user message providing the writing prompt or context, and a response starter. Randomly assign one of four sampling presets per generation:
   - Deterministic: `temperature=0.0, top_p=1.0, top_k=-1`
   - Balanced: `temperature=0.5, top_p=0.95, top_k=100`
   - Creative: `temperature=0.7, top_p=0.9, top_k=50`
   - Highly Creative: `temperature=1.0, top_p=0.95, top_k=30`

4. **Balance the corpus rigorously.** Ensure equal token counts between human and AI classes, equal representation across generator models, and genre proportions matching overall distribution. Cap samples at 8192 tokens. Split into train (roughly 60%), validation (20%), and test (20%).

5. **Implement the token-level classification head.** Add a `nn.Linear(hidden_size * 2, 1)` layer followed by `sigmoid`. For each token position `i`, concatenate `[hidden_state[i]; hidden_state[-1]]` — the token's own representation with the final token's global representation — then pass through the head.

6. **Freeze the LLM backbone and train only the classification head.** Use Adam optimizer with `lr` in the 1e-4 to 5e-4 range, `betas=(0.9, 0.999)`, binary cross-entropy loss, linear LR schedule with 20% warmup, for 5 epochs.

7. **Evaluate at token level and document level.** Compute per-token accuracy, precision, recall, and F1. For document-level scores, average the per-token predictions across the full sample and threshold at 0.5.

8. **If targeting known generators, consider per-family ensembles.** Group models by family (e.g., all Llama variants together, all Qwen together). Train one detector per family. At inference, run all family detectors and take the maximum confidence score. This trades some accuracy for attribution capability.

9. **Benchmark against the DeBERTa baseline.** Compare your detector against `desklib/ai-text-detector-v1.01` (430M DeBERTa-v3-large) as a strong open-source baseline (88.0% accuracy on the paper's benchmark).

10. **Deploy with confidence calibration.** Use the validation set to calibrate the sigmoid threshold — 0.5 is default but domain-specific tuning (e.g., higher threshold for low-false-positive requirements) improves practical utility.

## Concrete Examples

**Example 1: Building a generic AI text detector with Phi-4**

User: "I want to fine-tune Phi-4 to detect AI-generated text. How should I set this up?"

Approach:
1. Load `microsoft/phi-4` with frozen weights using `model.requires_grad_(False)`
2. Add classification head:
   ```python
   class AITextDetector(nn.Module):
       def __init__(self, base_model, hidden_size):
           super().__init__()
           self.base = base_model
           self.head = nn.Linear(hidden_size * 2, 1)

       def forward(self, input_ids, attention_mask):
           with torch.no_grad():
               outputs = self.base(input_ids, attention_mask=attention_mask,
                                   output_hidden_states=True)
           hidden = outputs.hidden_states[-1]  # (batch, seq, hidden)
           global_repr = hidden[:, -1:, :].expand_as(hidden)  # final token
           combined = torch.cat([hidden, global_repr], dim=-1)
           logits = self.head(combined).squeeze(-1)  # (batch, seq)
           return torch.sigmoid(logits)
   ```
3. Train on balanced corpus with BCE loss for 5 epochs, linear warmup 20%
4. At inference, average per-token predictions for document score

Output: A detector achieving ~99% accuracy on mixed-generator test sets, supporting inputs up to 8192 tokens.

**Example 2: Constructing a balanced multi-domain training corpus**

User: "How do I build a training dataset for AI text detection that covers multiple LLMs?"

Approach:
1. Collect human texts from diverse sources:
   ```
   Sources: XSum (news summaries), WritingPrompts (creative fiction),
            Reddit (informal), arXiv abstracts (academic), blog posts
   Target: ~500K samples, ~50M tokens human text
   ```
2. For each human sample, generate AI equivalents using 5+ LLMs:
   ```python
   models = ["meta-llama/Llama-3.1-8B-Instruct",
             "microsoft/Phi-4",
             "Qwen/Qwen2.5-7B-Instruct",
             "mistralai/Mistral-Nemo-Instruct-2407",
             "tiiuae/Falcon3-7B-Instruct"]

   sampling_presets = [
       {"temperature": 0.0, "top_p": 1.0},    # deterministic
       {"temperature": 0.5, "top_p": 0.95},    # balanced
       {"temperature": 0.7, "top_p": 0.9},     # creative
       {"temperature": 1.0, "top_p": 0.95},    # highly creative
   ]
   # Randomly assign a preset per generation
   ```
3. Balance the dataset:
   ```python
   # Ensure equal human/AI token counts
   # Ensure equal representation per generator model
   # Ensure genre proportions match overall distribution
   # Cap at 8192 tokens per sample
   ```

Output: A corpus with ~100M tokens split 50/50 human/AI, covering 5+ generators, 5+ domains, and 4 sampling temperature regimes.

**Example 3: Evaluating per-LLM-family ensemble vs. generic detector**

User: "Should I train one detector or separate detectors for each LLM family?"

Approach:
1. Train a single generic detector on the full mixed corpus
2. Train per-family detectors (one for Llama-family texts, one for Phi-family, etc.)
3. For ensemble inference, run all family detectors and take max confidence:
   ```python
   def ensemble_predict(text, family_detectors):
       scores = [det.predict(text) for det in family_detectors.values()]
       return max(scores)  # highest AI probability from any family detector
   ```
4. Compare on held-out test set:
   ```
   Generic Phi-4:         99.6% token accuracy, single model (14B params)
   Per-Family Ensemble:   80.6% sample accuracy, 6 models (69B params total)
   Per-LLM Ensemble:      71.3% sample accuracy, 21 models (416B params total)
   ```

Output: The generic approach wins on raw accuracy and efficiency. Use per-family only when you need attribution (identifying which model family generated the text) or when your deployment targets a known, narrow set of generators.

## Best Practices

- **Do** freeze the LLM backbone entirely. Training only the classification head (a single linear layer) prevents catastrophic forgetting and reduces compute by orders of magnitude while achieving top accuracy.
- **Do** include multiple sampling temperature regimes in your AI-generated training data. Deterministic (temp=0) text is trivially detectable; creative (temp=0.7-1.0) text is harder. Your detector needs exposure to both.
- **Do** balance your corpus at three levels: human/AI class balance, cross-generator balance, and cross-domain balance. Imbalance at any level degrades generalization.
- **Do** concatenate the final token's hidden state with each token's representation. This global context signal is critical for causal LLMs that otherwise only see left-context per token.
- **Avoid** training per-LLM ensembles unless you specifically need generator attribution. A single well-trained generic detector outperforms ensembles on overall accuracy while using far fewer parameters.
- **Avoid** evaluating only at document level. Token-level metrics reveal where the detector struggles (e.g., opening sentences vs. middle paragraphs) and enable partial-document detection for hybrid human/AI text.

## Error Handling

- **Low accuracy on a specific generator:** If the detector underperforms on one LLM's outputs, check whether that LLM is underrepresented in training data. Rebalance or add more samples from that generator.
- **High false positive rate on formal human text:** Formal, structured human writing (legal documents, scientific abstracts) can trigger false positives. Add more formal-register human text to training data and consider domain-specific thresholds.
- **Out-of-memory on long inputs:** With 14B parameter models and 8K context, memory is significant. Use gradient checkpointing during training, and at inference truncate or chunk inputs at 8192 tokens and average predictions across chunks.
- **Ensemble disagreement:** When per-family detectors disagree sharply, the text may be from an unseen model family. Fall back to the generic detector's prediction rather than relying on max-confidence aggregation.
- **Poor cross-domain generalization:** If the detector works on news but fails on code or poetry, the training corpus likely lacks those domains. The technique is corpus-dependent — expand coverage to fix generalization gaps.

## Limitations

- The method assumes access to a pretrained LLM as backbone (minimum ~3B parameters for competitive results; 14B recommended). This rules out low-resource deployment scenarios.
- Detection accuracy is tied to corpus coverage. Text from LLMs not represented in training (or from fundamentally different architectures) may evade detection. The authors note that fine-tuning for every available LLM (~700K on HuggingFace) is infeasible.
- The approach was evaluated on AI text generated by prompting human-written samples with uniform parameters. Real-world AI text with iterative editing, human-AI collaboration, or paraphrasing-based evasion is not covered.
- Token-level classification requires running the full LLM backbone at inference, making real-time detection expensive compared to lightweight classifiers like DeBERTa (430M params, 88% accuracy vs. Phi-4's 14B params, 99.6% accuracy). The accuracy-efficiency tradeoff is significant.
- The method detects statistical patterns, not semantic AI-ness. Adversarial techniques (targeted paraphrasing, style transfer) can degrade performance.

## Reference

Gromadzki, M., Wroblewska, A., & Kaliska, A. (2026). *On the Effectiveness of LLM-Specific Fine-Tuning for Detecting AI-Generated Text.* arXiv:2601.20006v1. [https://arxiv.org/abs/2601.20006v1](https://arxiv.org/abs/2601.20006v1)

Key takeaway: A single generic fine-tuned Phi-4 with a frozen backbone and token-level classification head achieves 99.6% accuracy — outperforming specialized per-LLM ensembles that cost 30x more compute. Code: [github.com/Michal1337/llm-ai-text-detection](https://github.com/Michal1337/llm-ai-text-detection). Dataset: [huggingface.co/datasets/Majkel1337/Detect-AI](https://huggingface.co/datasets/Majkel1337/Detect-AI).