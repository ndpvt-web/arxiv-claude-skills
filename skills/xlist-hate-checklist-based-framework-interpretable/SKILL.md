---
name: "xlist-hate-checklist-based-framework-interpretable"
description: "Decompose hate speech detection into a checklist of ten concept-level binary questions answered independently by an LLM, then aggregate results via a lightweight decision tree for interpretable, cross-dataset-robust classification. Use when asked to: 'build a hate speech detector', 'create an interpretable content moderation system', 'detect hateful content with explainability', 'classify toxic text with audit trails', 'implement checklist-based text classification', 'make a robust hate speech pipeline'."
---

# xList-Hate: Checklist-Based Diagnostic Framework for Interpretable Hate Speech Detection

This skill enables Claude to build hate speech detection systems that decompose the classification task into ten independent, concept-level binary questions grounded in international legal frameworks and platform policies. Instead of predicting "hate/not-hate" directly (which overfits to dataset-specific definitions), each input text is scored along explicit diagnostic dimensions -- protected-group targeting, dehumanization, slur usage, incitement, speaker endorsement, etc. -- producing a binary feature vector that a lightweight decision tree aggregates into a final, fully auditable prediction. The approach yields cross-dataset robustness (relative AUC typically >95% under domain shift vs. 85-95% for supervised baselines) and interpretable decision paths showing exactly which factors drove each prediction.

## When to Use

- When the user asks to build a hate speech or toxic content classifier that must be **explainable** or auditable
- When building a content moderation pipeline that needs to generalize across platforms, languages, or annotation schemes without retraining
- When the user wants to detect hateful content and understand **why** each item was flagged (e.g., "was it a slur? dehumanization? incitement?")
- When implementing a classification system that must be robust to annotation noise and inconsistent labeling across datasets
- When the user needs a modular text classification architecture where the semantic reasoning (LLM) is cleanly separated from the decision policy (decision tree)
- When asked to create a content moderation system with fine-grained factor-level analysis for bias auditing or policy compliance

## Key Technique

**Diagnostic decomposition, not direct classification.** Traditional hate speech detectors ask a model "Is this hate speech?" and fine-tune on a single dataset's labels. This couples semantic understanding with dataset-specific annotation conventions, causing brittleness under domain shift. xList-Hate instead asks ten independent binary questions -- each grounded in widely shared normative criteria from the UN, Council of Europe, UNESCO, Meta, YouTube, X, and Reddit policies -- producing a diagnostic vector z(t) in {0,1}^10. The LLM never sees the word "hate speech" or makes a final classification; it only answers factual concept-level questions like "Does this text target individuals or groups based on inherent characteristics?" and "Does this text dehumanize or vilify the targeted group?"

**Two-layer architecture with clean separation of concerns.** The first layer (LLM diagnostic) handles nuanced semantic judgment through ten parallel, stateless inference calls -- no conversational state, no cross-question contamination. Each call uses a structured prompt with an explicit factor statement, a guiding rationale, and few-shot examples. The second layer (decision tree aggregator) maps the 1,024 possible binary vector configurations to a final label using a scikit-learn `DecisionTreeClassifier` trained on a small labeled set. This separation means the semantic layer transfers across datasets while only the lightweight aggregator adapts to a new labeling regime.

**Interpretability by construction.** Every prediction traces to an explicit conjunction of satisfied predicates. For instance: "q3=Yes (slur detected) AND q9=Yes (speaker endorses) AND q4=Yes (dehumanization present) -> Hate speech." Feature importance analysis across the decision tree reveals which factors dominate (typically dehumanization, threats, slurs, and incitement), enabling systematic bias diagnosis and policy alignment verification.

## Step-by-Step Workflow

1. **Define the ten diagnostic questions.** Use the canonical xList-Hate checklist organized into four categories:
   - *Core conditions*: (q1) protected-group targeting, (q2) derogatory/hostile content
   - *Realizations*: (q3) explicit slur usage, (q4) dehumanization/vilification, (q5) blame/scapegoating
   - *Severity signals*: (q6) advocacy of discrimination/exclusion, (q7) threats/wishes of harm, (q8) incitement/endorsement of violence
   - *Contextual qualifiers*: (q9) speaker endorsement vs. quotation/satire, (q10) likely perceived harm to target group

2. **Construct the prompt template for each question.** Use a system message establishing the expert role and a user message presenting the text and question. Never ask the model to classify hate speech directly. Format:
   ```
   System: "You are an expert in hate speech detection. Your task is to answer
   the following question for the given text: {question}. For internal
   consistency, here is a guiding rationale: {rationale}. Please output your
   answer at the end in the format <a>Yes</a> or <a>No</a>."

   User: "Text: {text}. Answer the question: {question}.
   Please output your answer in the format <a>Yes|No</a>."
   ```

3. **Write guiding rationales for each question.** Each rationale is a 1-2 sentence scope clarification. For example, q4's rationale: "Dehumanization includes comparing the targeted group to animals, insects, diseases, or subhuman entities, or portraying them as monstrous, criminal by nature, or existentially threatening." Include 2-3 few-shot examples (one positive, one negative, optionally one borderline) per question.

4. **Implement parallel binary inference.** For each input text, fire all ten question prompts independently (no shared conversational state). Parse the `<a>Yes</a>` or `<a>No</a>` tags from responses. If a response is unparseable, use log-probability forcing: append the opening `<a>` token and query log-probabilities of "Yes" vs. "No" to force binary resolution.

5. **Assemble the diagnostic vector.** Encode each answer as a binary feature: `z(t) = [q1, q2, ..., q10]` where each qi is 0 or 1. This is the diagnostic representation of the input text.

6. **Train the decision tree aggregator.** Use `sklearn.tree.DecisionTreeClassifier` with `min_weight_fraction_leaf=0.01` on a labeled dataset where inputs are the 10-dimensional binary vectors and labels are the dataset's hate/not-hate annotations. This tree learns which factor combinations map to "hate" under the specific labeling regime.

7. **Extract and log decision paths.** For each prediction, export the conjunction of predicates (e.g., "q1=1 AND q4=1 AND q9=1 -> hate") as the explanation. Compute feature importance scores across the tree to identify which diagnostic dimensions dominate.

8. **Evaluate cross-dataset robustness.** Train the aggregator on one dataset, test on others. Measure relative AUC (OOD AUC / in-domain AUC * 100). Expect values above 95% for the checklist approach, compared to 85-95% for direct supervised classification.

9. **Analyze disagreement cases.** When the checklist disagrees with ground truth, inspect the diagnostic vector to determine whether the error stems from (a) an LLM misanswer on a specific question, (b) an aggregation boundary case, or (c) an annotation artifact in the dataset. This attribution is the core interpretability advantage.

10. **Iterate on the checklist.** Add domain-specific questions (e.g., for a platform that prohibits glorification of self-harm) or remove questions that are never informative for a particular use case. The modular structure supports this without retraining the LLM layer.

## Concrete Examples

**Example 1: Building a hate speech detection API**

User: "Build me a hate speech detection system that explains its decisions."

Approach:
1. Define the ten diagnostic questions as a JSON config:
```python
CHECKLIST = [
    {
        "id": "q1",
        "question": "Does the text target individuals or groups based on inherent or protected characteristics such as race, ethnicity, religion, gender, sexual orientation, disability, or national origin?",
        "rationale": "Protected targeting means the text singles out people for who they are, not what they have done. Criticism of specific actions or policies does not qualify.",
        "examples": [
            {"text": "Those people from X country are all thieves", "answer": "Yes"},
            {"text": "The senator's tax policy is terrible", "answer": "No"}
        ]
    },
    {
        "id": "q2",
        "question": "Does the text convey hostility, contempt, or animus toward the targeted group or its members?",
        "rationale": "Hostile tone includes ridicule, disgust, demonization, or expressions of superiority. Neutral factual mentions of a group do not qualify.",
        "examples": [
            {"text": "These disgusting animals should go back where they came from", "answer": "Yes"},
            {"text": "The census shows demographic shifts in the region", "answer": "No"}
        ]
    },
    # ... q3 through q10 following the same structure
]
```

2. Implement the parallel diagnostic inference:
```python
import asyncio
from typing import list
import re

async def diagnose_text(text: str, llm_client, checklist: list[dict]) -> list[int]:
    """Run all checklist questions in parallel, return binary vector."""
    async def ask_question(q: dict) -> int:
        system_msg = (
            f"You are an expert in hate speech detection. Your task is to "
            f"answer the following question for the given text: {q['question']}. "
            f"For internal consistency, here is a guiding rationale: {q['rationale']}. "
            f"Please output your answer at the end in the format <a>Yes</a> or <a>No</a>."
        )
        user_msg = (
            f"Text: {text}\n\nAnswer the question: {q['question']}\n"
            f"Please output your answer in the format <a>Yes|No</a>."
        )
        response = await llm_client.chat(system=system_msg, user=user_msg)
        match = re.search(r"<a>(Yes|No)</a>", response)
        if match:
            return 1 if match.group(1) == "Yes" else 0
        # Fallback: log-probability forcing
        return await force_binary(llm_client, system_msg, user_msg, response)

    results = await asyncio.gather(*[ask_question(q) for q in checklist])
    return list(results)
```

3. Train the decision tree on labeled diagnostic vectors:
```python
from sklearn.tree import DecisionTreeClassifier, export_text
import numpy as np

# X_train: (N, 10) binary matrix from diagnose_text on training set
# y_train: (N,) hate/not-hate labels
tree = DecisionTreeClassifier(min_weight_fraction_leaf=0.01)
tree.fit(X_train, y_train)

# Export human-readable rules
feature_names = [f"q{i+1}" for i in range(10)]
print(export_text(tree, feature_names=feature_names))
```

Output (decision path for a prediction):
```
Prediction: HATE SPEECH
Diagnostic vector: [1, 1, 1, 1, 0, 0, 0, 0, 1, 1]
Decision path:
  q1=1 (targets protected group)        -> YES
  q2=1 (conveys hostility)              -> YES
  q3=1 (contains slur)                  -> YES
  q9=1 (speaker endorses, not quoting)  -> YES
  => Leaf: hate (confidence: 0.87)
```

**Example 2: Cross-dataset content moderation**

User: "I have a model trained on the Davidson dataset but need it to work on HateXplain data too without retraining."

Approach:
1. Run `diagnose_text` on both datasets using the same LLM and checklist -- the diagnostic layer is dataset-agnostic
2. Train a decision tree on Davidson's diagnostic vectors and labels
3. Evaluate on HateXplain's diagnostic vectors -- the binary representation abstracts away dataset-specific annotation artifacts
4. Compare relative AUC against a BERT model fine-tuned on Davidson and tested on HateXplain

Output:
```
Cross-dataset evaluation (train: Davidson, test: HateXplain):

                  In-domain AUC    OOD AUC    Relative AUC
xList-Hate:           0.89          0.86         96.6%
BERT fine-tuned:      0.93          0.79         84.9%

The checklist approach loses 3.4% under domain shift vs. 15.1% for BERT.
```

**Example 3: Auditing flagged content**

User: "A post was flagged as hate speech. Show me exactly why."

Approach:
1. Run `diagnose_text` on the flagged post
2. Extract the decision tree path for the resulting vector
3. Present factor-by-factor analysis

Output:
```
Post: "Typical [group] behavior, they're like cockroaches infesting our country"

Factor-level analysis:
  q1  Protected targeting:      YES - references ethnic/national group
  q2  Hostile tone:             YES - contempt and disgust expressed
  q3  Slur usage:               NO  - no explicit slur, but pejorative framing
  q4  Dehumanization:           YES - "cockroaches" strips human status
  q5  Scapegoating:             YES - "infesting our country" blames group
  q6  Exclusion advocacy:       YES - implies group should be removed
  q7  Threat of harm:           NO  - no direct threat articulated
  q8  Violence incitement:      NO  - no call to violent action
  q9  Speaker endorsement:      YES - speaker expresses own view
  q10 Perceived harm:           YES - reasonable group member would feel threatened

Decision path: q1=1 -> q4=1 -> q9=1 -> q2=1 => HATE SPEECH (confidence: 0.92)
Primary factors: dehumanization + protected targeting + speaker endorsement
```

## Best Practices

- **Do:** Keep each question prompt fully independent -- no shared conversation context, no "given your previous answers." Cross-question contamination defeats the purpose of diagnostic decomposition.
- **Do:** Use the `<a>Yes</a>` / `<a>No</a>` XML tag format to constrain output parsing. Fall back to log-probability forcing only when tag extraction fails.
- **Do:** Retrain only the decision tree (not the LLM prompts) when adapting to a new dataset or policy regime. The semantic layer should stay stable.
- **Do:** Log the full diagnostic vector alongside every prediction for downstream auditing and bias analysis.
- **Avoid:** Asking the LLM "Is this hate speech?" anywhere in the prompt. The framework's power comes from never exposing the final classification concept to the semantic layer.
- **Avoid:** Using deep or ensemble classifiers as the aggregator. The decision tree's interpretability is a core design requirement, not a limitation to optimize away.
- **Avoid:** Omitting the guiding rationale from prompts. Without it, LLMs interpret questions inconsistently, especially on borderline cases like satire and counter-speech.

## Error Handling

- **Unparseable LLM output:** If the `<a>` tag pattern is not found, append `<a>` to the response and query log-probabilities over "Yes" and "No" tokens. If log-probs are unavailable (e.g., API limitation), retry the prompt once with a simplified instruction. If still unparseable, default to 0 (conservative) and log a warning.
- **All-zero diagnostic vector:** If all ten questions return "No," the text is almost certainly not hate speech. However, flag it for human review if the aggregator still predicts "hate" -- this indicates a degenerate tree leaf.
- **Inconsistent q9 (endorsement) answers:** Satire, sarcasm, and counter-speech are the hardest for LLMs to judge. When q9 flips the final prediction, surface this to the user as a low-confidence case requiring human review.
- **Decision tree overfitting on small datasets:** If the training set is under ~500 labeled diagnostic vectors, increase `min_weight_fraction_leaf` to 0.05 or use cross-validation to select tree depth.

## Limitations

- **Latency:** Ten parallel LLM calls per input text adds latency and cost compared to a single fine-tuned model inference. This is a throughput-interpretability tradeoff best suited for moderation queues rather than real-time filtering.
- **LLM quality dependency:** The diagnostic layer is only as good as the LLM's ability to answer nuanced concept-level questions. Smaller models may struggle with sarcasm detection (q9) and perceived harm assessment (q10).
- **English-centric normative grounding:** The ten questions are derived from English-language legal frameworks and platform policies. Hate speech in other languages may involve cultural dimensions not captured by these ten factors.
- **Not a replacement for in-domain fine-tuning when distribution is stable.** Supervised models will typically outperform on in-domain benchmarks. The checklist approach trades peak in-domain accuracy for cross-dataset robustness and interpretability.
- **Decision tree capacity:** With only 10 binary features, the aggregator has limited capacity to capture complex feature interactions. This is intentional for interpretability but means some nuanced cases will be misclassified.

## Reference

- **Paper:** [xList-Hate: A Checklist-Based Framework for Interpretable and Generalizable Hate Speech Detection](https://arxiv.org/abs/2602.05874v1) (Giron et al., 2026)
- **Key takeaway:** Look at Section 3 for the full checklist derivation from normative sources, Section 4 for prompt templates and log-probability forcing, and Table 2 for cross-dataset relative AUC results showing the robustness advantage over supervised baselines.