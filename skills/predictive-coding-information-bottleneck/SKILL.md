---
name: "predictive-coding-information-bottleneck"
description: >
  Build lightweight hallucination detection pipelines using Predictive Coding surprise signals
  and Information Bottleneck perturbation testing. Implements the PCIB framework: extract interpretable
  signals (Uptake, Stress, Conflict, Falsifiability) from LLM outputs, train sub-1M-parameter classifiers,
  and deploy real-time hallucination filters for RAG systems.
  Trigger phrases: "detect hallucinations", "hallucination detection pipeline", "verify LLM output",
  "build a hallucination filter", "RAG output quality scoring", "factual grounding check"
---

# Predictive Coding & Information Bottleneck Hallucination Detection

This skill enables Claude to build production-grade hallucination detection systems based on the PCIB framework. Instead of relying on expensive 70B+ LLM judges or slow retrieval loops, PCIB extracts four interpretable signals from LLM outputs -- Uptake (prediction error), Stress (semantic stability), Conflict (logical consistency), and Falsifiability (confident contradiction) -- then feeds them into a lightweight Random Forest classifier. The result is a sub-1M-parameter detector achieving 0.87 AUROC at 5ms inference, using 75x less training data than comparable methods.

## When to Use

- When building a hallucination detection layer for a RAG pipeline that needs to flag unfaithful answers before they reach users
- When implementing real-time quality gates on LLM-generated content (customer support, medical summaries, legal analysis)
- When the user wants interpretable hallucination scores with per-signal explanations, not just a binary pass/fail
- When designing a two-tier verification system where a fast first-pass filter reduces load on expensive LLM judges
- When creating labeled datasets for hallucination detection with minimal annotation effort (the framework works with as few as 200 balanced samples)
- When auditing an existing LLM system to understand *why* specific outputs are hallucinated (entity-level analysis, grounding strength, perturbation fragility)

## Key Technique

The PCIB framework is grounded in two neuroscience-inspired principles. **Predictive Coding** treats the LLM as a hierarchical prediction machine: when given context, a grounded answer aligns with the model's updated beliefs, while a hallucination requires the model to suppress provided context in favor of prior biases. This is measured as **Uptake** -- the KL divergence between the answer distribution conditioned on context+question versus question alone. High Uptake means the context meaningfully shaped the answer (likely factual); low Uptake means the model ignored the context (likely hallucinated).

The **Information Bottleneck** principle posits that factual claims are robust compressed representations, while hallucinated claims are noise that degrades under perturbation. PCIB tests this by paraphrasing extracted claims at temperature 0.7, then measuring how much the NLI entailment probability shifts (**Stress** via Jensen-Shannon divergence) and whether contradictions emerge (**Conflict** via NLI contradiction probability). The insight: a true fact survives rephrasing; a hallucination crumbles.

Three enhancements sharpen these base signals for production use. **Entity-Focused Uptake** weights the KL divergence toward named entities (people, dates, numbers) since hallucinations cluster in high-value tokens. **Context Adherence** combines inverse stress with context richness to measure grounding strength. **Falsifiability Score** augments conflict detection with linguistic confidence markers -- flagging statements that say "definitely" or "certainly" while contradicting the source. A critical negative finding: **Rationalization** (consistency across reasoning traces) does NOT help, because LLMs generate coherent justifications even for false premises (sycophancy). Do not waste compute on chain-of-thought self-verification for hallucination detection.

## Step-by-Step Workflow

1. **Define the input triple (Q, C, A).** Structure every detection query as a Question, Context (source document or retrieved passages), and Answer (LLM-generated response). If the user provides only an answer, prompt them for the source context -- detection without grounding context is unreliable.

2. **Compute Uptake (Predictive Coding surprise).** Generate token-level log-probabilities for the answer twice: once conditioned on question only `P(A|Q)`, once conditioned on question+context `P(A|Q,C)`. Compute `U = KL(P(A|Q,C) || P(A|Q))`. Higher values indicate the context informed the answer (factual); near-zero values suggest the model ignored context (hallucination risk).

3. **Extract claims from the answer.** Use the LLM to decompose the answer into atomic, independently verifiable claims. For example, "Einstein published relativity in 1905 while working at the patent office" becomes two claims: (a) Einstein published relativity in 1905, (b) he was working at the patent office.

4. **Compute Stress (Information Bottleneck stability).** For each claim, generate K=5 paraphrases at temperature 0.7. Run an NLI model (e.g., `cross-encoder/nli-deberta-v3-base`) on both original and paraphrased claims against the context. Compute `S = mean(JS_divergence(p_original, p_paraphrased))` across all claims. High stress = fragile claim = hallucination signal.

5. **Compute Conflict (logical consistency).** Using the same NLI model, measure `C = max(P(contradiction | A, c_paraphrased))` across all paraphrased claim variants. A high conflict score means at least one perturbation reveals a logical inconsistency.

6. **Compute Entity-Focused Uptake.** Run spaCy NER on the answer. Reweight the base Uptake: `U_entity = U_base * (1 + 2.0 * |entities| / |tokens|)`. This amplifies the signal when the answer contains many named entities, which are the tokens most likely to be hallucinated.

7. **Compute Context Adherence.** Combine stress and context length: `A_context = (1 / (1 + S)) * min(1, |context_words| / 200)`. Low stress with rich context indicates strong grounding; high stress with thin context flags weak grounding.

8. **Compute Falsifiability Score.** Count definitive language markers ("definitely", "certainly", "clearly", "always", "never") and hedging markers ("possibly", "maybe", "perhaps", "might") in the answer. Compute `F = C_base * (1 + 0.1 * (n_definitive - n_hedge))`. High confidence + high conflict = likely hallucination.

9. **Assemble feature vector and classify.** Stack all signals into a vector `x = [U, S, C, U_entity, A_context, F, ESI]` where `ESI` is the geometric mean of (1-S) and (1-C) as an evidence sufficiency index. Train or apply a Random Forest classifier (scikit-learn, 100 estimators) on this 7-8 dimensional vector.

10. **Return interpretable results.** Output the binary prediction AND the per-signal breakdown so users can understand *why* something was flagged. For production, set a threshold on the RF probability (e.g., >0.6 = likely hallucination) and route flagged outputs to human review or a heavier LLM judge.

## Concrete Examples

**Example 1: Building a hallucination detector for a RAG chatbot**

User: "I have a RAG system answering questions from company docs. Build me a hallucination detection pipeline that flags bad answers before they reach customers."

Approach:
1. Define the pipeline entry point accepting (question, retrieved_context, generated_answer) triples
2. Implement signal extraction using the LLM API and an NLI model
3. Train the classifier on a small labeled set, deploy as a scoring microservice

Output structure:
```python
# pcib_detector.py
import numpy as np
from scipy.special import kl_div, rel_entr
from scipy.spatial.distance import jensenshannon
from transformers import pipeline
import spacy

nlp = spacy.load("en_core_web_sm")
nli_model = pipeline("text-classification", model="cross-encoder/nli-deberta-v3-base")

def compute_uptake(llm_client, question, context, answer):
    """KL divergence: P(A|Q,C) vs P(A|Q) at token level."""
    logprobs_with_ctx = llm_client.logprobs(prompt=f"{question}\n{context}\n", completion=answer)
    logprobs_without_ctx = llm_client.logprobs(prompt=f"{question}\n", completion=answer)
    return sum(rel_entr(np.exp(logprobs_with_ctx), np.exp(logprobs_without_ctx)))

def compute_stress_and_conflict(llm_client, nli_pipe, context, claims, k=5, temp=0.7):
    """Perturbation testing via paraphrasing + NLI."""
    stresses, conflicts = [], []
    for claim in claims:
        paraphrases = [llm_client.paraphrase(claim, temperature=temp) for _ in range(k)]
        orig_entail = nli_pipe(f"{context} [SEP] {claim}")[0]
        para_scores = [nli_pipe(f"{context} [SEP] {p}")[0] for p in paraphrases]
        # JS divergence between original and paraphrased entailment distributions
        stresses.append(np.mean([jensenshannon(
            [orig_entail['score'], 1 - orig_entail['score']],
            [ps['score'], 1 - ps['score']]
        ) for ps in para_scores]))
        # Max contradiction probability across perturbations
        contra_scores = [nli_pipe(f"{claim} [SEP] {p}", hypothesis="contradiction")
                         for p in paraphrases]
        conflicts.append(max(cs['score'] for cs in contra_scores))
    return np.mean(stresses), max(conflicts)

def compute_entity_uptake(uptake, answer):
    doc = nlp(answer)
    entity_ratio = len(doc.ents) / max(len(doc), 1)
    return uptake * (1 + 2.0 * entity_ratio)

def compute_falsifiability(conflict, answer):
    definitive = sum(1 for w in ["definitely","certainly","clearly","always","never"]
                     if w in answer.lower())
    hedges = sum(1 for w in ["possibly","maybe","perhaps","might","likely"]
                 if w in answer.lower())
    return conflict * (1 + 0.1 * (definitive - hedges))

def score(question, context, answer, llm_client):
    uptake = compute_uptake(llm_client, question, context, answer)
    claims = llm_client.extract_claims(answer)
    stress, conflict = compute_stress_and_conflict(llm_client, nli_model, context, claims)
    entity_uptake = compute_entity_uptake(uptake, answer)
    context_adherence = (1 / (1 + stress)) * min(1, len(context.split()) / 200)
    falsifiability = compute_falsifiability(conflict, answer)
    esi = np.sqrt((1 - stress) * (1 - conflict))
    return {
        "uptake": uptake, "stress": stress, "conflict": conflict,
        "entity_uptake": entity_uptake, "context_adherence": context_adherence,
        "falsifiability": falsifiability, "esi": esi,
        "features": [uptake, stress, conflict, entity_uptake,
                      context_adherence, falsifiability, esi]
    }
```

**Example 2: Training the classifier on minimal labeled data**

User: "I have 200 labeled examples of hallucinated vs. faithful answers. Train a detector."

Approach:
1. Extract PCIB features for all 200 examples
2. Train a Random Forest with cross-validation
3. Export the model for production inference

```python
# train_pcib_classifier.py
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score
import joblib, json

# Load labeled data: list of {"question", "context", "answer", "label": 0|1}
with open("labeled_data.json") as f:
    data = json.load(f)

# Extract features for each sample
X = np.array([score(d["question"], d["context"], d["answer"], llm)["features"]
              for d in data])
y = np.array([d["label"] for d in data])

# Train Random Forest -- the best-performing architecture per the paper
clf = RandomForestClassifier(n_estimators=100, max_depth=10, random_state=42)
auroc_scores = cross_val_score(clf, X, y, cv=5, scoring="roc_auc")
print(f"Cross-validated AUROC: {np.mean(auroc_scores):.4f} +/- {np.std(auroc_scores):.4f}")

clf.fit(X, y)
joblib.dump(clf, "pcib_hallucination_detector.pkl")

# Feature importance ranking
feature_names = ["uptake", "stress", "conflict", "entity_uptake",
                 "context_adherence", "falsifiability", "esi"]
for name, imp in sorted(zip(feature_names, clf.feature_importances_), key=lambda x: -x[1]):
    print(f"  {name}: {imp:.3f}")
```

**Example 3: Two-tier production deployment**

User: "I need hallucination detection at scale -- millions of queries per day. Full LLM judge is too expensive."

Approach:
1. Deploy PCIB as a fast first-pass filter (5ms, <$0.001 per query)
2. Route only uncertain cases (RF probability between 0.4-0.6) to an LLM judge
3. This eliminates ~80% of queries from expensive verification

```
Architecture:

  LLM Answer ──> PCIB Scorer (5ms, <1M params)
                      │
            ┌─────────┼──────────┐
            ▼         ▼          ▼
        Score < 0.4   0.4-0.6   Score > 0.6
        (Factual)   (Uncertain)  (Hallucination)
            │         │          │
            ▼         ▼          ▼
        Pass through  LLM Judge  Flag + block
                     (5s, 70B)
```

Configuration:
```python
PCIB_THRESHOLD_LOW = 0.4   # Below: pass as factual
PCIB_THRESHOLD_HIGH = 0.6  # Above: flag as hallucination
# Between: escalate to expensive LLM judge

def route_query(pcib_score):
    if pcib_score < PCIB_THRESHOLD_LOW:
        return "pass"
    elif pcib_score > PCIB_THRESHOLD_HIGH:
        return "flag_hallucination"
    else:
        return "escalate_to_llm_judge"
```

## Best Practices

- **Do:** Always include source context (C) alongside the question and answer. Detection without grounding context produces unreliable signals -- Uptake fundamentally requires the context comparison.
- **Do:** Weight entity-rich answers more aggressively. Hallucinations cluster in named entities, dates, and numbers. The `alpha=2.0` entity weighting in Entity-Focused Uptake reflects this empirical finding.
- **Do:** Use a balanced training set even if your production data is imbalanced. The paper achieved strong results with exactly 50/50 hallucinated/factual splits on just 200 samples. Oversample or undersample to balance.
- **Do:** Expose per-signal scores to downstream consumers, not just the binary label. The interpretability of individual signals (Stress, Conflict, Falsifiability) is a core advantage over black-box judges.
- **Avoid:** Using chain-of-thought self-verification (Rationalization) for hallucination detection. The paper's negative result shows LLMs generate coherent reasoning for false premises. Multiple reasoning traces will agree with each other even when wrong.
- **Avoid:** Skipping the perturbation step to save compute. Stress and Conflict (the Information Bottleneck signals) are the most discriminative features. Without them, you lose the core detection mechanism.

## Error Handling

- **NLI model unavailable or too slow:** Fall back to a theory-guided unsupervised score using only Uptake and a simple entailment check. The unsupervised baseline achieves 0.80 AUROC without any NLI model, using only log-probability signals.
- **LLM API does not expose token log-probabilities:** Some APIs (e.g., certain Claude endpoints) do not return logprobs. In this case, approximate Uptake by prompting the LLM to rate its own confidence with and without context, or use embedding cosine similarity between context and answer as a proxy.
- **Claim extraction produces too many or too few claims:** If the LLM returns zero claims, treat the entire answer as a single claim. If it returns more than 20, sample the 10 most entity-dense claims to keep perturbation cost bounded.
- **Paraphrase generation produces near-duplicates:** If JS divergence is near-zero across all paraphrases, increase temperature to 0.9 or use a different paraphrasing model. Near-duplicate paraphrases make the Stress signal uninformative.
- **Class imbalance in production:** The classifier was trained on balanced data. If production traffic is 95% factual, calibrate the RF probability threshold upward (e.g., 0.7 instead of 0.5) to control false positive rate.

## Limitations

- **Context-dependent only.** PCIB requires source context to compute Uptake and grounding signals. It cannot detect hallucinations in open-ended generation without a reference document (e.g., creative writing, brainstorming).
- **NLI model ceiling.** The Stress and Conflict signals are bounded by the NLI model's accuracy. Domain-specific content (legal, medical, scientific) may need a fine-tuned NLI model rather than a general-purpose one.
- **Small evaluation scale.** The paper evaluates on n=200 from HaluBench. Performance on larger, domain-specific benchmarks is not validated. Expect variance when applying to specialized domains.
- **English-centric.** Entity extraction (spaCy), linguistic markers (hedge/definitive words), and NLI models are English-focused. Multilingual deployment requires adapted components for each language.
- **Latency of perturbation step.** While the classifier itself runs in 5ms, generating K=5 paraphrases per claim requires LLM calls. For a 5-claim answer, that is 25 LLM calls for paraphrasing plus 25 NLI calls. Batch these aggressively or reduce K for latency-sensitive applications.
- **Sycophancy blind spot.** The Rationalization negative result means this framework cannot catch hallucinations that are internally consistent across multiple reasoning paths. If an LLM consistently fabricates the same false fact, perturbation testing on claims (not reasoning) is the only viable signal.

## Reference

- **Paper:** [Predictive Coding and Information Bottleneck for Hallucination Detection in Large Language Models](https://arxiv.org/abs/2601.15652v1) (Bhatt, 2026)
- **Key insight:** Domain knowledge encoded in signal architecture (Predictive Coding surprise + Information Bottleneck perturbation) provides 75x better data efficiency than scaling LLM judges, achieving 0.87 AUROC with <1M parameters and 200 training samples.