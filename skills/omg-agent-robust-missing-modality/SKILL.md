---
name: "omg-agent-robust-missing-modality"
description: >
  Build robust multimodal pipelines that handle missing or incomplete data using the OMG-Agent
  coarse-to-fine agentic workflow: a three-stage Semantic Planner → Evidence Retriever → Executor
  architecture that decouples reasoning from synthesis. Use this skill when building systems that
  must tolerate absent modalities (text, audio, video, images) or when designing agent pipelines
  that fuse partial observations into complete outputs.
  Trigger phrases: "handle missing modality", "incomplete multimodal data", "robust to missing inputs",
  "generate missing features", "multimodal imputation", "coarse-to-fine agent pipeline"
---

# OMG-Agent: Robust Missing Modality Generation with Coarse-to-Fine Agentic Workflows

This skill teaches Claude to design and implement agentic pipelines that gracefully handle missing or incomplete multimodal data. Based on the OMG-Agent framework, the core insight is **Semantic-Detail Decoupling**: instead of training a single end-to-end model to both reason about what's missing and synthesize the missing signal (which creates a structural conflict called Semantic-Detail Entanglement), you decompose the task into three explicit stages -- planning what to generate, retrieving evidence to ground the plan, and executing synthesis under dual constraints. This approach is applicable whenever you build systems that ingest multiple data streams (text + images, audio + video, sensor fusion) and must remain robust when some streams are absent or degraded.

## When to Use This Skill

- When building a multimodal ML pipeline that must tolerate missing inputs at inference time (e.g., a sentiment analysis system where audio or video may be unavailable)
- When designing an agent workflow that needs to reconstruct or impute absent data modalities from partial observations
- When a user asks to make an existing multimodal system robust to data incompleteness without retraining from scratch
- When implementing retrieval-augmented generation where retrieved evidence should guide synthesis rather than be directly pasted in
- When refactoring a monolithic generation model into a staged pipeline to improve fidelity and reduce hallucination
- When the user needs a "deliberate-then-act" architecture where an LLM plans what to generate before a specialized model executes

## Key Technique: Decoupled Coarse-to-Fine Agentic Workflow

**The core problem.** End-to-end models that reconstruct missing modalities face Semantic-Detail Entanglement: the same network must simultaneously perform logical reasoning (what *should* the missing data contain?) and signal synthesis (what exact features should be produced?). These are fundamentally different computational tasks -- reasoning operates on abstract semantics while synthesis operates on dense signal representations. Forcing both into one forward pass causes hallucination and low fidelity.

**The OMG-Agent solution** decomposes the task into three cooperating stages that mirror a human "deliberate-then-act" cognitive process:

1. **Semantic Planner** -- An MLLM (e.g., Qwen2.5-Omni) performs Progressive Contextual Reasoning over available modalities. It autoregressively generates a structured semantic plan `S = {c1, ..., cL}` where each constraint conditions on observations and all prior constraints. A re-ranking mechanism scores candidate plans by balancing log-likelihood, semantic consistency (cosine similarity between plan embedding and observation features), and schema validity. This converts ambiguous partial inputs into a deterministic, structured specification.

2. **Evidence Retriever** -- A non-parametric module that queries an external feature bank using a fused embedding of observations and the semantic plan. It uses sparse attention over Top-K retrieved entries to produce an aggregated evidence vector. "Non-parametric" means the evidence consists of real feature vectors from a knowledge base, not hallucinated outputs from model weights. This grounds abstract plan semantics in concrete, verifiable data.

3. **Retrieval-Injected Executor** -- A conditional diffusion model (or analogous generator) that receives dual constraints: the semantic plan is injected via cross-attention in deep layers (controlling global structure), while retrieved evidence is injected via lightweight adapters in shallow layers (controlling local detail). An instruction-following loss enforces both constraints: `L_plan = 1 - cos(g(Y_hat), c_S)` for semantic alignment and `L_evi = ||phi(Y_hat) - A(E)||` for detail fidelity.

## Step-by-Step Workflow

### 1. Audit available and missing modalities
Enumerate all expected input modalities (e.g., language L, vision V, acoustics A). Create a binary availability mask indicating which modalities are present. Log the missing pattern explicitly (e.g., `{L: present, V: missing, A: present}`).

### 2. Define the structured semantic plan schema
Design a JSON or dataclass schema that the Semantic Planner must output. The schema should capture modality-specific attributes at a semantic level. For example, for missing video in a sentiment task: `{"facial_expression": str, "head_movement": str, "gaze_direction": str, "emotional_valence": float}`. The schema constrains the planner's output space and prevents open-ended hallucination.

### 3. Implement Progressive Contextual Reasoning
Feed available modalities into an MLLM with a prompt that instructs it to reason step-by-step about the missing modality. Each reasoning step should condition on prior conclusions. Structure the prompt so the model first identifies what can be inferred from available data, then progressively narrows the semantic description of the missing modality.

```python
plan_prompt = f"""Given the following partial observations:
- Text transcript: "{transcript}"
- Audio features: [available/missing]
- Video features: [available/missing]

Reason step-by-step about what the missing {missing_modality} likely contains.
Step 1: What emotional tone does the text convey?
Step 2: What prosodic/visual features would align with this tone?
Step 3: Output a structured plan following this schema: {schema}
"""
```

### 4. Score and re-rank candidate plans
Generate N candidate plans (e.g., N=5 with temperature sampling). Score each using: `score = log_prob - lambda_s * (1 - cosine_sim(plan_embed, observation_embed)) - gamma * schema_violations`. Select the highest-scoring plan. Typical hyperparameters: `lambda_s=0.3`, `gamma=0.1`.

### 5. Build or connect to an external evidence bank
Construct a feature bank from training data or an external corpus. Each entry should contain real feature vectors (not raw data) indexed for similarity search. Use FAISS or a similar vector index. The bank provides non-parametric grounding -- the system retrieves actual observed features rather than generating them from scratch.

### 6. Retrieve and aggregate evidence with sparse attention
Project the semantic plan and available observations into a query vector: `q = sigma(W_q * [obs_embed; plan_embed] + b_q)`. Retrieve Top-K entries (K=10 is a good default). Compute attention weights over retrieved entries and aggregate: `E = sum(alpha_i * v_i)` where `alpha_i = softmax(similarity_scores)`.

### 7. Inject plan and evidence at different abstraction levels
Design the executor so that semantic-level guidance (the plan) enters at deep/abstract layers and detail-level guidance (evidence) enters at shallow/concrete layers. For transformer-based generators, use cross-attention for the plan and additive adapters for evidence. This orthogonalizes semantics and details in the feature space.

### 8. Apply dual-constraint loss during training or fine-tuning
Add two auxiliary losses: (a) `L_plan`: cosine similarity between the output's semantic embedding and the plan embedding, ensuring the output follows the planned semantics; (b) `L_evi`: L1 distance between the output's detail features and a projection of the evidence, ensuring fidelity to retrieved details. Weight both at `lambda=0.1`.

### 9. Validate under systematic missing patterns
Test every combination of missing modalities. For three modalities, this means 7 patterns (all single-missing, all double-missing, all-missing). Measure both task performance (accuracy, F1) and reconstruction quality (MSE, cosine similarity). The system should degrade gracefully, not catastrophically.

### 10. Expose the pipeline as a fault-tolerant API
Wrap the three stages behind an API that accepts partial inputs and returns complete multimodal representations. The API should detect which modalities are missing, route through the pipeline, and return both the reconstructed features and the semantic plan (for interpretability).

## Concrete Examples

**Example 1: Sentiment analysis with missing video**

```
User: "Build a multimodal sentiment classifier that works even when
the video stream drops out. We have text transcripts and audio features
but video is intermittently unavailable."

Approach:
1. Define availability mask: {text: True, audio: True, video: False}
2. Define video plan schema:
   {"facial_action_units": list[float],  # 35-dim FAU vector description
    "head_pose": str,                     # e.g., "slight_nod", "neutral"
    "expression_category": str,           # e.g., "smile", "frown"
    "intensity": float}                   # 0.0-1.0
3. Prompt an MLLM with text + audio features:
   "The speaker says 'I absolutely loved it' with rising intonation
    and 0.8 positive valence in audio. Reason about likely facial
    expression and output the video plan schema."
4. Plan output: {"facial_action_units": "AU6+AU12 activation (smile)",
                  "head_pose": "slight_nod",
                  "expression_category": "genuine_smile",
                  "intensity": 0.85}
5. Retrieve Top-10 similar video feature vectors from the training
   bank where text sentiment and audio prosody matched.
6. Aggregate retrieved features with attention weights.
7. Feed plan + evidence into executor to produce 35-dim FAU +
   74-dim visual feature vector for the downstream classifier.

Output: The classifier receives a complete (text, audio, video_reconstructed)
tuple and achieves ~2.6 points higher accuracy than baselines at 70% video
missing rate.
```

**Example 2: Multimodal document processing with missing OCR**

```
User: "Our document understanding pipeline takes images + OCR text + layout
features. Sometimes OCR fails entirely on scanned documents. How do I handle
the missing text modality?"

Approach:
1. Availability mask: {image: True, ocr_text: False, layout: True}
2. Define text plan schema:
   {"document_type": str,       # "invoice", "receipt", "letter"
    "key_fields": list[str],    # expected field names
    "language": str,
    "text_density": str}        # "sparse", "moderate", "dense"
3. Semantic Planner examines image + layout features:
   "Given a document image with tabular layout (3 columns, 12 rows)
    and a header region at top, infer the document type and expected text
    content structure."
4. Plan: {"document_type": "invoice", "key_fields": ["date", "amount",
          "vendor", "line_items"], "language": "en", "text_density": "moderate"}
5. Retrieve text embeddings from similar invoice templates in the evidence bank.
6. Executor generates a plausible text feature vector (e.g., 768-dim BERT
   embedding) that the downstream model uses for classification/extraction.
7. The downstream model processes (image, reconstructed_text_features, layout)
   as if OCR had succeeded.

Output: Document understanding accuracy drops only 3-5% instead of 20%+ when
OCR completely fails.
```

**Example 3: Implementing the three-stage pipeline in Python**

```python
# Skeleton implementation of the OMG-Agent pipeline

from dataclasses import dataclass
from typing import Optional
import numpy as np

@dataclass
class SemanticPlan:
    constraints: dict          # structured plan output
    embedding: np.ndarray      # dense vector for retrieval/loss
    confidence: float          # re-ranking score

@dataclass
class ModalityInput:
    text: Optional[np.ndarray]    # None if missing
    audio: Optional[np.ndarray]
    video: Optional[np.ndarray]

class SemanticPlanner:
    """Stage 1: MLLM-driven planning with progressive reasoning."""

    def __init__(self, llm_client, schema: dict, n_candidates: int = 5):
        self.llm = llm_client
        self.schema = schema
        self.n_candidates = n_candidates

    def plan(self, inputs: ModalityInput) -> SemanticPlan:
        available = self._describe_available(inputs)
        missing = self._identify_missing(inputs)

        candidates = []
        for _ in range(self.n_candidates):
            plan = self.llm.generate(
                prompt=self._build_progressive_prompt(available, missing),
                schema=self.schema,
                temperature=0.7
            )
            score = self._rerank(plan, inputs)
            candidates.append((plan, score))

        best_plan, best_score = max(candidates, key=lambda x: x[1])
        return SemanticPlan(
            constraints=best_plan,
            embedding=self._embed_plan(best_plan),
            confidence=best_score
        )

    def _rerank(self, plan, inputs, lambda_s=0.3, gamma=0.1):
        log_prob = self._compute_log_prob(plan)
        sem_sim = cosine_sim(self._embed_plan(plan), self._embed_obs(inputs))
        schema_penalty = self._check_schema(plan)
        return log_prob - lambda_s * (1 - sem_sim) - gamma * schema_penalty


class EvidenceRetriever:
    """Stage 2: Non-parametric retrieval with sparse attention."""

    def __init__(self, feature_bank, top_k: int = 10):
        self.bank = feature_bank  # FAISS index + feature store
        self.top_k = top_k

    def retrieve(self, obs_embed: np.ndarray, plan: SemanticPlan) -> np.ndarray:
        query = self._project_query(obs_embed, plan.embedding)
        indices, scores = self.bank.search(query, k=self.top_k)
        features = self.bank.get_features(indices)
        weights = softmax(scores)
        return np.sum(weights[:, None] * features, axis=0)


class RetrievalInjectedExecutor:
    """Stage 3: Dual-constrained synthesis."""

    def __init__(self, generator_model):
        self.model = generator_model

    def execute(self, plan: SemanticPlan, evidence: np.ndarray) -> np.ndarray:
        # Plan injected at deep layers (global semantics)
        # Evidence injected at shallow layers (local details)
        return self.model.generate(
            plan_condition=plan.embedding,
            evidence_condition=evidence
        )


# Full pipeline
class OMGAgentPipeline:
    def __init__(self, planner, retriever, executor):
        self.planner = planner
        self.retriever = retriever
        self.executor = executor

    def reconstruct(self, inputs: ModalityInput) -> np.ndarray:
        plan = self.planner.plan(inputs)
        obs_embed = encode_available(inputs)
        evidence = self.retriever.retrieve(obs_embed, plan)
        return self.executor.execute(plan, evidence)
```

## Best Practices

- **Do:** Define explicit schemas for the Semantic Planner's output. Unconstrained generation leads to the same hallucination problems the framework is designed to solve.
- **Do:** Use real feature vectors in the evidence bank, not raw data. The retriever should return embeddings that can be directly injected into the executor, not text or pixels that need re-encoding.
- **Do:** Inject plan and evidence at different depth levels in the executor. Plan controls structure (deep layers); evidence controls detail (shallow layers). Mixing them at the same level reintroduces entanglement.
- **Do:** Test all combinatorial missing patterns, not just single-modality absence. The system should handle cascading failures gracefully.
- **Avoid:** End-to-end training of the entire three-stage pipeline without stage-wise pretraining. Each stage should be verified independently before joint optimization.
- **Avoid:** Over-relying on the planner's confidence score alone. Cross-check plan plausibility against retrieved evidence -- if the top retrieved entries strongly contradict the plan, re-plan.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| All modalities missing | Availability mask is all-False | Fall back to unconditional prior from evidence bank; flag output as low-confidence |
| Planner produces invalid schema | Schema validation fails | Retry with lower temperature; if persistent, use template-based fallback plan |
| Evidence bank returns low-similarity results | Max retrieval score below threshold | Reduce evidence weight in executor; rely more heavily on plan conditioning |
| Executor output violates plan constraints | `L_plan` loss exceeds threshold at inference | Apply iterative refinement: re-run executor with stronger plan conditioning weight |
| Modality encoder fails on corrupted input | Exception during feature extraction | Treat that modality as missing and enter the reconstruction pipeline |

## Limitations

- **Requires a feature bank.** The evidence retriever needs a pre-built index of real feature vectors. For novel domains with no existing data, this stage cannot function, and the pipeline degrades to plan-only generation.
- **Planner quality depends on the MLLM.** If the underlying language model has poor understanding of the target domain (e.g., specialized medical imaging), the semantic plan will be unreliable. Domain-specific fine-tuning or few-shot examples are necessary.
- **Latency overhead.** Three sequential stages (plan → retrieve → execute) add latency compared to a single forward pass. Not suitable for hard real-time applications (<10ms budget) without aggressive optimization.
- **Feature-level reconstruction only.** This framework reconstructs intermediate feature representations, not raw signals (e.g., it produces a 768-dim embedding, not actual video frames). If raw signal reconstruction is needed, an additional decoder stage is required.
- **Assumes modality absence is detectable.** The pipeline relies on knowing which modalities are missing. It does not handle silently corrupted inputs that appear present but contain garbage data.

## Reference

**Paper:** [OMG-Agent: Toward Robust Missing Modality Generation with Decoupled Coarse-to-Fine Agentic Workflows](https://arxiv.org/abs/2602.04144v1) (Dai et al., 2026)

**Key takeaway:** The paper's central contribution is demonstrating that explicitly decoupling semantic reasoning from detail synthesis via a three-stage agentic workflow (plan → retrieve → execute) consistently outperforms end-to-end approaches, especially under extreme data missingness (70%+ missing rates). Look for Section 3 (methodology) for the Progressive Contextual Reasoning formulation and Section 4 (experiments) for ablation studies showing each stage's contribution.