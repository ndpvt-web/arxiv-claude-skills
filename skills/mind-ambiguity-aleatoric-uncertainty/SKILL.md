---
name: "mind-ambiguity-aleatoric-uncertainty"
description: "Detect ambiguous user queries in safety-critical QA systems using aleatoric uncertainty probes on LLM hidden states, then route to clarification or answering. Use when: 'detect ambiguous medical questions', 'add clarification to my QA pipeline', 'build a safe medical chatbot', 'probe LLM hidden states for uncertainty', 'triage vague user queries before answering', 'implement clarify-before-answer logic'."
---

# Aleatoric Uncertainty Detection for Safe QA: Clarify-Before-Answer

This skill teaches Claude how to implement the AU-Probe "Clarify-Before-Answer" framework from the paper *Mind the Ambiguity* (WWW 2026). The core idea: ambiguous user queries in high-stakes domains (medical, legal, financial) cause LLMs to give wrong answers silently. Instead of answering blindly, you train a lightweight linear probe on the LLM's internal hidden states to detect whether a query is underspecified, then route ambiguous queries to a clarification step before generating an answer. The probe requires no LLM fine-tuning and no multiple forward passes — just a single inference pass with a linear classifier on the hidden states.

## When to Use This Skill

- When building a medical, legal, or financial QA system that must handle vague or incomplete user questions safely
- When a user asks to add ambiguity detection or clarification logic to an existing LLM-based chatbot
- When implementing a triage layer that decides whether to answer directly or ask follow-up questions
- When training a linear probe on LLM hidden states to classify input properties (ambiguity, topic, safety)
- When constructing contrastive datasets of clear vs. ambiguous question pairs for probe training
- When evaluating whether an LLM pipeline handles underspecified inputs gracefully (benchmarking with CV-MedBench)
- When the user wants to reduce hallucination risk by detecting inputs the model is uncertain about

## Key Technique: Representation Engineering for Aleatoric Uncertainty

**Aleatoric uncertainty (AU)** is the irreducible uncertainty that comes from the *input itself* being underspecified — not from the model lacking knowledge (that would be epistemic uncertainty). A question like "What medication helps with chest pain?" is ambiguous because chest pain has dozens of causes. The correct answer depends on context the user hasn't provided.

The paper's central discovery is that **AU is linearly encoded in LLM hidden states**. When you feed a clear question and its ambiguous counterpart through the same LLM and extract hidden states from intermediate layers, the difference between them lies along a consistent linear direction. This means a simple logistic regression or linear SVM trained on hidden-state vectors can classify whether an input is ambiguous with high accuracy (AUROC scores well above 0.8 across multiple LLMs).

This insight enables **AU-Probe**: a frozen linear classifier that sits on top of an LLM's hidden states at a chosen layer. During inference, you run the input through the LLM once, extract the hidden state at the probe layer, pass it through AU-Probe, and get an ambiguity score. If the score exceeds a threshold, the system asks the user for clarification instead of generating an answer. This "Clarify-Before-Answer" pipeline achieved a 9.48% average accuracy improvement over standard QA baselines across four open LLMs.

## Step-by-Step Workflow

### Phase 1: Build the Contrastive Training Dataset

1. **Collect domain-specific question pairs.** For each clear, well-specified question in your domain, create (or use an LLM to generate) an ambiguous variant that removes key clinical/contextual details. Each pair shares the same correct answer and a unique `id`. Label clear questions `0` and ambiguous questions `1`. Structure as:
   ```json
   {"id": 42, "input": "What is the first-line treatment for...", "output": "B", "label": 0}
   {"id": 42, "input": "What medicine helps with that condition?", "output": "B", "label": 1}
   ```

2. **Split into train/test sets** preserving pair integrity — both members of a pair must be in the same split. Use 80/20 or load the pre-built CV-MedBench dataset:
   ```python
   from datasets import load_dataset
   ds = load_dataset("yaokunl/CV-MedBench", "cv_medqa")
   train_clear = ds["train_clear"]
   train_vague = ds["train_vague"]
   ```

### Phase 2: Extract Hidden States

3. **Run each question through the target LLM and capture hidden states.** Use a hook or the `output_hidden_states=True` flag in HuggingFace Transformers. Extract the hidden state of the **last token** at your chosen probe layer (middle-to-upper layers tend to be most informative — start with layer `L * 0.6` to `L * 0.8` where `L` is total layers):
   ```python
   from transformers import AutoModelForCausalLM, AutoTokenizer
   import torch

   model = AutoModelForCausalLM.from_pretrained(model_name, output_hidden_states=True)
   tokenizer = AutoTokenizer.from_pretrained(model_name)

   inputs = tokenizer(question, return_tensors="pt")
   with torch.no_grad():
       outputs = model(**inputs)

   # Extract hidden state at chosen layer, last token position
   probe_layer = int(model.config.num_hidden_layers * 0.7)
   hidden = outputs.hidden_states[probe_layer][0, -1, :]  # shape: (hidden_dim,)
   ```

4. **Store hidden-state vectors with their labels.** Save as numpy arrays or torch tensors. You need vectors for both the clear and ambiguous versions of each question in the training set.

### Phase 3: Train the AU-Probe

5. **Train a linear classifier on the extracted hidden states.** A logistic regression is sufficient — the paper shows the AU direction is linear. Use scikit-learn or a single-layer PyTorch module:
   ```python
   from sklearn.linear_model import LogisticRegression
   from sklearn.metrics import roc_auc_score
   import numpy as np

   X_train = np.stack(all_train_vectors)  # (N, hidden_dim)
   y_train = np.array(all_train_labels)   # 0=clear, 1=ambiguous

   probe = LogisticRegression(max_iter=1000, C=1.0)
   probe.fit(X_train, y_train)

   # Evaluate
   X_test = np.stack(all_test_vectors)
   y_test = np.array(all_test_labels)
   auroc = roc_auc_score(y_test, probe.predict_proba(X_test)[:, 1])
   print(f"AU-Probe AUROC: {auroc:.4f}")
   ```

6. **Select the operating threshold.** Use the test set to pick a probability threshold that balances false clarifications (annoying the user) against missed ambiguity (unsafe answers). Plot precision-recall or use Youden's J statistic on the ROC curve. A typical range is 0.5–0.7.

### Phase 4: Deploy the Clarify-Before-Answer Pipeline

7. **Wire the probe into the inference path.** On each incoming query:
   - Tokenize and run a single forward pass through the LLM with hidden states enabled
   - Extract the hidden state at the probe layer (last token)
   - Pass through AU-Probe to get an ambiguity score
   - If score >= threshold: generate a clarification request instead of an answer
   - If score < threshold: proceed with standard answer generation

8. **Generate targeted clarification requests.** When ambiguity is detected, do not just say "Can you clarify?" — identify *what* is underspecified. Prompt the LLM with the original query plus a system instruction like: "The following medical question is ambiguous. Identify what specific information is missing and ask the user to provide it." Return this as the response.

9. **After clarification, re-run the pipeline.** Concatenate the user's clarification with the original query and pass through AU-Probe again. If still ambiguous, request further clarification. If clear, generate the answer.

10. **Monitor and recalibrate.** Track the rate of clarification triggers, user satisfaction, and downstream answer accuracy. Re-train the probe periodically as the LLM is updated or the domain shifts.

## Concrete Examples

**Example 1: Medical QA Chatbot with Ambiguity Detection**

User: "Build a medical QA system that asks for clarification when questions are vague."

Approach:
1. Load a medical LLM (e.g., BioMistral-7B) and CV-MedBench from HuggingFace
2. Extract hidden states from layer 22 (of 32) for all train_clear and train_vague splits
3. Train a logistic regression probe on the extracted vectors
4. Build a FastAPI endpoint that accepts questions, runs AU-Probe, and either asks for clarification or answers

Output structure:
```python
# POST /ask {"question": "What helps with chest pain?"}
# AU-Probe score: 0.82 (above threshold 0.6)
{
  "action": "clarify",
  "message": "Your question about chest pain is too broad to answer safely. Could you specify: (1) Is this acute or chronic chest pain? (2) What is the patient's age and medical history? (3) Are there accompanying symptoms like shortness of breath or radiating arm pain?"
}

# POST /ask {"question": "What is the first-line treatment for stable angina in a 55-year-old male with no contraindications to beta-blockers?"}
# AU-Probe score: 0.18 (below threshold 0.6)
{
  "action": "answer",
  "message": "The first-line treatment for stable angina is a beta-blocker (e.g., metoprolol or atenolol) combined with sublingual nitroglycerin for acute symptom relief..."
}
```

**Example 2: Adding a Safety Layer to an Existing Chatbot**

User: "I have a LangChain-based medical chatbot. Add an ambiguity detection step before it answers."

Approach:
1. Insert a custom LangChain `RunnablePassthrough` step before the LLM chain
2. In this step, run the user query through the same LLM backbone (single forward pass), extract hidden states
3. Load pre-trained AU-Probe weights, score the query
4. Branch the chain: high AU score triggers a clarification prompt template; low score proceeds to the answer chain

```python
from langchain_core.runnables import RunnableBranch, RunnableLambda

def score_ambiguity(query: str) -> dict:
    hidden = extract_hidden_state(model, tokenizer, query, layer=probe_layer)
    score = au_probe.predict_proba(hidden.reshape(1, -1))[0, 1]
    return {"query": query, "au_score": score}

def is_ambiguous(state: dict) -> bool:
    return state["au_score"] >= THRESHOLD

chain = (
    RunnableLambda(score_ambiguity)
    | RunnableBranch(
        (is_ambiguous, clarification_chain),
        answer_chain  # default
    )
)
```

**Example 3: Benchmarking with CV-MedBench**

User: "Evaluate how well my LLM handles ambiguous medical questions."

Approach:
1. Load CV-MedBench test splits (both clear and vague)
2. Run the LLM on both splits and measure accuracy on each
3. Calculate the accuracy drop between clear and vague inputs — this is the "ambiguity gap"
4. Train an AU-Probe and measure AUROC, ECE, and Brier score for ambiguity detection quality

```bash
# Using the AU-Med codebase
git clone https://github.com/yaokunliu/AU-Med.git && cd AU-Med
pip install -r requirements.txt
bash run/medqa.sh  # runs evaluation on MedQA subset
python scripts/analyze.py  # outputs AUROC, ECE, Brier scores to CSV
```

Expected output:
```
| Model         | Clear Acc | Vague Acc | Gap   | AU-Probe AUROC |
|---------------|-----------|-----------|-------|----------------|
| BioMistral-7B | 0.72      | 0.58      | -0.14 | 0.87           |
| Llama-3-8B    | 0.68      | 0.55      | -0.13 | 0.84           |
```

## Best Practices

- **Do:** Use the last token's hidden state for probe input — it aggregates the most context about the full query.
- **Do:** Sweep multiple layers when training the probe (e.g., layers 60%–80% depth). The optimal layer varies by model architecture.
- **Do:** Generate *specific* clarification questions that name the missing information, not generic "please clarify" responses.
- **Do:** Evaluate with AUROC (discrimination), ECE (calibration), and Brier score (overall quality) — accuracy alone is misleading for imbalanced ambiguity detection.
- **Avoid:** Using output token probabilities or entropy alone for ambiguity detection — these conflate aleatoric and epistemic uncertainty. The hidden-state probe isolates aleatoric uncertainty specifically.
- **Avoid:** Setting the clarification threshold too low, which causes the system to ask for clarification on clear questions and frustrates users. Calibrate on held-out data with domain experts.
- **Avoid:** Fine-tuning the LLM itself for ambiguity detection — the whole point of AU-Probe is that a frozen linear classifier suffices, keeping the approach lightweight and modular.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Low AUROC (<0.70) on probe | Wrong layer selected, or insufficient contrast in training data | Sweep all layers to find the one with maximum probe AUROC; ensure clear/vague pairs have genuine information gaps, not just paraphrases |
| High false clarification rate | Threshold too low; probe over-detects ambiguity | Raise the threshold; check if the training data contains clear questions that are stylistically similar to vague ones |
| Probe doesn't generalize to new domains | Training data too narrow | Add domain-specific clear/vague pairs to the training set; consider layer-wise probing on the new domain |
| OOM when extracting hidden states | Full hidden states for all layers stored simultaneously | Extract only the target layer's hidden state; use `torch.no_grad()` and offload to CPU immediately |
| Clarification questions are too generic | LLM not prompted to identify specific missing info | Add a system prompt that instructs the LLM to list exactly which clinical details (age, symptoms, duration, history) are absent |

## Limitations

- **Linear probe assumption:** The method assumes AU is linearly separable in hidden-state space. For some model architectures or highly nuanced ambiguity, a nonlinear probe may be needed — but this trades off simplicity.
- **Domain transfer:** A probe trained on medical data will not transfer well to legal or financial QA without retraining on domain-appropriate clear/vague pairs.
- **Single-model binding:** The probe is tied to a specific LLM's layer dimensions and representation space. Changing the LLM (or even its quantization level) requires retraining the probe.
- **Aleatoric only:** This approach detects input ambiguity, not model knowledge gaps. A perfectly clear question about an obscure topic will pass the probe but may still get a wrong answer due to epistemic uncertainty.
- **Clarification adds latency:** The pipeline introduces one round-trip of user interaction for ambiguous queries. In time-critical scenarios, consider whether the safety benefit justifies the delay.
- **Requires access to hidden states:** Not applicable to closed-source LLM APIs (e.g., GPT-4) that don't expose intermediate representations. Only works with open-weight models where you control inference.

## Reference

**Paper:** Liu et al., "Mind the Ambiguity: Aleatoric Uncertainty Quantification in LLMs for Safe Medical Question Answering," The Web Conference 2026 (WWW 2026). [arXiv:2601.17284](https://arxiv.org/abs/2601.17284v1)

Look for: Section 3 (representation engineering analysis showing AU is linearly encoded), Section 4 (AU-Probe architecture and Clarify-Before-Answer pipeline), and Table 2 (layer-wise AUROC results showing which layers encode ambiguity best).

**Code:** [github.com/yaokunliu/AU-Med](https://github.com/yaokunliu/AU-Med) | **Dataset:** [huggingface.co/datasets/yaokunl/CV-MedBench](https://huggingface.co/datasets/yaokunl/CV-MedBench)