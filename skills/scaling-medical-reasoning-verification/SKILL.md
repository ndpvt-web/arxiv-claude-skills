---
name: "scaling-medical-reasoning-verification"
description: >
  Build agentic verification pipelines that iteratively retrieve external evidence to fact-check
  LLM reasoning traces, based on the Med-TIV framework. Combines tool-augmented verification
  with reinforcement learning and adaptive curriculum training.
  Use when: "build a reasoning verifier", "verify medical reasoning with retrieval",
  "fact-check LLM outputs against a corpus", "iterative retrieval verification pipeline",
  "agentic RL verification framework", "tool-integrated reasoning checker"
---

# Scaling Medical Reasoning Verification via Tool-Integrated RL

This skill enables Claude to build **agentic verification systems** that iteratively query external knowledge bases to fact-check LLM reasoning traces. Based on the Med-TIV (Medical Tool-Integrated reasoning Verifier) framework, the core idea is to train or orchestrate a verifier agent that — instead of producing a single scalar score — generates structured reasoning interleaved with search queries, retrieves evidence from a dense corpus, and synthesizes a justified verification judgment. The technique generalizes beyond medicine to any domain where reasoning must be grounded in retrievable facts.

## When to Use

- When the user wants to **verify LLM-generated reasoning** against an authoritative corpus (medical, legal, scientific, financial)
- When building a **retrieval-augmented verification pipeline** that goes beyond single-pass RAG by allowing the verifier to issue multiple adaptive queries
- When implementing a **reward model or judge** for RLHF/GRPO that needs to produce explainable, evidence-backed scores rather than opaque scalars
- When designing an **adaptive curriculum** for RL training that filters trivially easy and impossibly hard examples
- When the user asks to build a system that **fact-checks clinical reasoning**, drug interactions, diagnostic chains, or treatment plans against PubMed or similar corpora
- When implementing a **best-of-N selection pipeline** where a verifier ranks multiple candidate reasoning traces by correctness

## Key Technique

**Med-TIV** replaces monolithic reward models with an agentic verifier that interleaves reasoning with tool calls. Given a question `q` and a candidate reasoning trace `τ`, the verifier generates structured output in a loop: it thinks about potential errors in `<think>` blocks, formulates a natural-language search query in `<search>` blocks, receives top-k retrieved documents, then continues reasoning. After up to T iterations (typically 2), it emits a final judgment in an `<answer>` block. This iterative retrieval lets the verifier adaptively seek evidence for the specific claims it finds questionable, rather than retrieving everything upfront.

The verifier is trained via **Dr.GRPO** (a group-relative policy optimization variant) using only trace-level supervision — a single binary label per reasoning trace (correct/incorrect), with no step-level annotations. The reward function is `R = R_c × R_f`, where `R_c` is correctness (does the verifier's judgment match the ground truth label?) and `R_f` is format compliance (are XML tags well-formed?). An **adaptive curriculum** filters training batches to retain only instances where sampled verification trajectories show reward variance — removing cases the verifier always gets right or always gets wrong. This focuses training compute on the learning frontier.

The retrieval backend uses **MedCPT** embeddings over a FAISS index of ~24 million snippets from PubMed abstracts and medical textbooks. Each search returns the top-3 documents. The approach achieves an **8x reduction in sampling budget** compared to scalar reward models for best-of-N selection, because evidence-grounded verification is far more sample-efficient than majority voting or ungrounded scoring.

## Step-by-Step Workflow

1. **Define the verification domain and corpus.** Identify the knowledge base your verifier will query (e.g., PubMed for medical, case law for legal, documentation for code). Prepare a dense retrieval index: encode documents with a domain-specific encoder (MedCPT for biomedical, or a general encoder like `all-MiniLM-L6-v2`) and build a FAISS index.

2. **Design the structured verification format.** Define XML-tag schemas the verifier must follow:
   ```
   <think>Analyze claim about drug interaction...</think>
   <search>metformin contraindications renal impairment</search>
   [Retrieved documents inserted here]
   <think>Evidence confirms the trace is incorrect because...</think>
   <answer>incorrect</answer>
   ```
   Set a maximum number of search iterations (2 is effective; more adds latency without proportional gains).

3. **Build the retrieval tool interface.** Implement a function that accepts a natural-language query string, encodes it, performs approximate nearest-neighbor search on the FAISS index, and returns top-k document snippets (k=3 recommended). Wrap this as a callable tool the verifier agent can invoke.

4. **Prepare trace-level training data.** Collect (question, reasoning_trace, correctness_label) triplets. Labels are binary: 1 if the trace reaches the correct answer without factual errors, 0 otherwise. You do NOT need step-level annotations — trace-level labels suffice.

5. **Implement the reward function.** Compute `R = R_c × R_f`:
   - `R_c = 1` if the verifier's extracted judgment matches the ground truth label, else `0`
   - `R_f = 1` if output follows the XML schema, `0.25` if minor violations (e.g., extra answer tags), `0` if malformed
   - Final reward is the product, ensuring format compliance is a prerequisite for credit

6. **Apply adaptive curriculum filtering.** For each training batch, sample G verification trajectories per instance. Retain only instances where at least two trajectories receive different rewards (i.e., reward variance > 0). Discard trivially solved and impossible cases. Resample to maintain a fixed training set size (~20K instances).

7. **Train with Dr.GRPO (or PPO/GRPO variant).** Run group-relative policy optimization:
   - Sample G trajectories per instance from the current policy
   - Compute per-trajectory rewards and group-normalize advantages
   - Apply clipped policy gradient update (clip range: ε_low=0.2, ε_high=0.3)
   - Iterate for T_max training rounds (2 rounds is sufficient)

8. **Deploy as a verification agent.** At inference, feed the verifier a (question, candidate_trace) pair. The verifier autonomously decides what to search, retrieves evidence, and outputs a justified judgment. Use this for:
   - Best-of-N selection: generate N candidate answers, verify each, pick the one judged correct
   - Filtering: reject traces the verifier flags as incorrect
   - Explanation: surface the retrieved evidence and verifier reasoning to end users

9. **Evaluate and calibrate.** Measure verification accuracy on a held-out set. Track both precision (are flagged-correct traces actually correct?) and recall (are incorrect traces caught?). Adjust the retrieval top-k and max iterations if needed.

## Concrete Examples

**Example 1: Building a medical reasoning verifier with PubMed retrieval**

User: "I want to build a system that fact-checks medical QA reasoning traces against PubMed. The model sometimes hallucinates drug interactions."

Approach:
1. Download PubMed abstracts via the MedRAG collection (~24M snippets)
2. Encode with MedCPT and build a FAISS `IndexFlatIP` (or `IndexIVFFlat` for scale)
3. Define the verification prompt template with `<think>`, `<search>`, `<answer>` tags
4. Implement the search tool:
   ```python
   import faiss
   import numpy as np
   from transformers import AutoModel, AutoTokenizer

   class MedicalSearchTool:
       def __init__(self, index_path, docs, encoder_name="ncbi/MedCPT-Query-Encoder"):
           self.tokenizer = AutoTokenizer.from_pretrained(encoder_name)
           self.encoder = AutoModel.from_pretrained(encoder_name)
           self.index = faiss.read_index(index_path)
           self.docs = docs  # list of document strings

       def search(self, query: str, top_k: int = 3) -> list[str]:
           inputs = self.tokenizer(query, return_tensors="pt", truncation=True, max_length=512)
           emb = self.encoder(**inputs).last_hidden_state[:, 0, :].detach().numpy()
           faiss.normalize_L2(emb)
           scores, indices = self.index.search(emb, top_k)
           return [self.docs[i] for i in indices[0]]
   ```
5. Wire into a verification loop that parses `<search>` tags, calls the tool, injects results, and continues generation
6. Train the verifier with GRPO using trace-level labels from MedQA

Output: A verifier that, given a reasoning trace claiming "Metformin is safe in severe renal failure," searches PubMed, retrieves contraindication evidence, and outputs `<answer>incorrect</answer>` with cited justification.

**Example 2: Adaptive curriculum for RL training of a code review verifier**

User: "I'm training a code review agent with RL. Some examples are too easy (obvious bugs) and some are too hard (subtle race conditions). How do I focus training on the useful examples?"

Approach:
1. For each (code_snippet, review_trace, label) training instance, sample G=8 verification trajectories from the current policy
2. Compute rewards for each trajectory
3. Filter: keep only instances where `max(rewards) != min(rewards)` — the model is uncertain
4. Discard instances where all 8 trajectories agree (too easy or too hard)
5. Resample filtered set to maintain fixed batch size

```python
def adaptive_curriculum_filter(instances, policy, G=8):
    """Retain only instances with reward variance across G sampled trajectories."""
    filtered = []
    for inst in instances:
        rewards = [compute_reward(policy.sample(inst)) for _ in range(G)]
        if max(rewards) != min(rewards):  # non-zero variance
            filtered.append(inst)
    # Resample to target size if needed
    if len(filtered) < TARGET_SIZE:
        filtered = resample_with_replacement(filtered, TARGET_SIZE)
    return filtered[:TARGET_SIZE]
```

Output: Training converges faster because gradient signal comes from instances at the model's decision boundary, not from already-mastered or impossibly hard cases.

**Example 3: Best-of-N selection with 8x fewer samples**

User: "I'm using majority voting over 64 samples to pick the best medical answer. It's expensive. Can I do better?"

Approach:
1. Instead of 64 samples with majority vote, generate N=8 candidate reasoning traces
2. Run each through the Med-TIV-style verifier with iterative retrieval
3. Select the candidate the verifier judges as correct with highest confidence
4. If multiple candidates pass, use retrieval evidence overlap as a tiebreaker

```python
def verified_best_of_n(question, generator, verifier, search_tool, n=8):
    candidates = [generator.generate(question) for _ in range(n)]
    verified = []
    for trace in candidates:
        judgment, evidence, reasoning = verifier.verify(
            question, trace, search_tool, max_iterations=2
        )
        if judgment == "correct":
            verified.append((trace, evidence, reasoning))
    if verified:
        return verified[0]  # first verified-correct candidate
    return candidates[0]  # fallback to first candidate
```

Output: Equivalent or better accuracy to majority-vote@64, using only 8 samples — an 8x reduction in compute cost.

## Best Practices

- **Do:** Use domain-specific encoders for retrieval (MedCPT for biomedical, legal-BERT for law). Generic embeddings degrade retrieval precision in specialized domains.
- **Do:** Cap search iterations at 2-3. Diminishing returns set in quickly, and more iterations increase latency without meaningful accuracy gains.
- **Do:** Multiply correctness and format rewards (`R_c × R_f`) rather than adding them. This ensures the verifier gets zero credit for correct judgments delivered in malformed output.
- **Do:** Refresh the curriculum filter every training iteration with fresh trajectory samples. The learning frontier shifts as the policy improves.
- **Avoid:** Step-level annotations. The whole point of trace-level supervision is that it scales — you only need a binary label per trace, not per-step correctness marks.
- **Avoid:** Single-pass retrieval before verification. The key insight is that the verifier discovers what it needs to look up *during* analysis, not before. Pre-retrieval misses claims the verifier hasn't yet identified as questionable.

## Error Handling

| Failure Mode | Cause | Mitigation |
|---|---|---|
| Verifier loops without issuing `<answer>` | Format reward too lenient | Enforce max iterations; set `R_f = 0` for missing answer tag |
| Retrieval returns irrelevant documents | Query too vague or corpus mismatch | Log retrieved docs during training; add query specificity heuristic |
| Curriculum filter removes all instances | Policy too deterministic (all trajectories agree) | Increase sampling temperature or reduce G; check for reward function bugs |
| Verifier always says "correct" | Reward signal too sparse or imbalanced labels | Balance correct/incorrect traces in training data; verify reward computation |
| FAISS index OOM on large corpus | Index doesn't fit in RAM | Use `IndexIVFPQ` with product quantization; shard across machines |

## Limitations

- **Retrieval ceiling.** The verifier can only catch errors that have contradicting evidence in the corpus. Novel findings, recent discoveries not yet indexed, or reasoning errors that are "plausible but wrong" in ways the corpus doesn't address will be missed.
- **Domain transfer.** A verifier trained on medical traces does not generalize to legal or financial verification without retraining on domain-specific data and swapping the retrieval corpus.
- **Latency.** Each verification requires 2-3 retrieval round-trips plus LLM generation. For real-time applications, this adds meaningful latency compared to a single-pass reward model.
- **Trace-level labels are noisy.** A trace can reach the correct answer via incorrect reasoning. Trace-level binary labels cannot distinguish this case, which caps verification quality.
- **Base model size.** Results in the paper use 7-8B parameter models. Smaller models may lack the reasoning capacity to formulate effective search queries during verification.

## Reference

**Paper:** [Scaling Medical Reasoning Verification via Tool-Integrated Reinforcement Learning](https://arxiv.org/abs/2601.20221v1) (Zhang et al., 2026)

**Key takeaway:** Look at Section 3 for the Dr.GRPO training algorithm with adaptive curriculum, and Section 4.3 for the ablation showing that iterative retrieval outperforms single-pass retrieval by a significant margin. Table 2 shows the 8x sampling efficiency gain over scalar reward models.