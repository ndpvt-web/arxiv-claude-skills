---
name: "from-sparse-decisions-dense"
description: "Build content moderation and safety classification systems using multi-attribute trajectory reasoning instead of binary labels. Decomposes monolithic safe/unsafe decisions into structured reasoning chains (evidence grounding, modality assessment, risk mapping, policy decision, response generation) with multi-head reward scoring. Use when asked to: 'build a content moderation pipeline', 'classify harmful content with explanations', 'create a safety filter with reasoning traces', 'design a multi-attribute content scorer', 'implement explainable content moderation', 'add dense safety reasoning to a classifier'."
---

# From Sparse Decisions to Dense Reasoning: Multi-Attribute Trajectory Moderation

This skill enables Claude to build content moderation and safety classification systems that replace brittle binary (safe/unsafe) labels with structured, multi-stage reasoning trajectories. Based on the UniMod paradigm, the approach decomposes each moderation decision into five explicit stages -- evidence grounding, modality assessment, risk mapping, policy decision, and response generation -- then scores outputs along multiple safety dimensions (quality, privacy, bias, toxicity, legal risk) using a multi-head reward model. This eliminates shortcut learning where classifiers latch onto superficial features, and produces explainable, auditable moderation decisions.

## When to Use

- When the user asks to build a content moderation API or pipeline that needs to explain *why* content was flagged, not just return a binary label
- When implementing safety classifiers for multimodal inputs (text + images) and the user needs per-modality risk assessment
- When designing a reward model or scoring system that evaluates generated content across multiple safety dimensions simultaneously
- When the user wants to reduce training data requirements for moderation systems by using structured supervision instead of raw label volume
- When building a policy-aware content filter that must map flagged content to specific policy categories (hate speech, self-harm, CSAM, etc.)
- When refactoring an existing binary classifier into a chain-of-thought moderation system with interpretable intermediate outputs
- When the user needs to train or fine-tune a model for safety tasks and wants to avoid reward hacking or shortcut convergence

## Key Technique

**The core insight**: Binary moderation labels (safe/unsafe) create a sparse supervision signal that lets models learn shortcuts -- e.g., flagging any image with skin tones as unsafe, or any mention of weapons as harmful regardless of context. UniMod replaces this with *dense* supervision by requiring the model to produce a structured reasoning trajectory before its final decision. Each stage constrains the search space for the next, so the model cannot skip to a conclusion without grounding it in evidence. This sequential decomposition reduces sample complexity from exponential (searching the full decision space) to stepwise (searching within each stage's logical subspace).

**Multi-head reward scoring**: Instead of a single reward signal, the UniRM component uses a shared backbone with five parallel scoring heads: quality/compliance, privacy protection, bias mitigation, toxicity avoidance, and legal risk assessment. Each head computes `r_k = sigmoid(w_k^T * h)` where `h` is the shared representation. Heads are kept independent via soft orthogonal regularization (`sum of (cos_sim(w_i, w_j))^2` with penalty lambda=0.05) and stochastic scheduling that randomizes which head updates first each epoch. The final reward is an additive aggregate `R = sum(w_k * r_k)`, which preserves reward variance and prevents the collapse-to-zero problem that multiplicative aggregation causes.

**Practical upshot**: This approach achieves state-of-the-art multimodal moderation with under 40% of the training data used by competing methods, because structured trajectories extract far more learning signal per sample than flat labels.

## Step-by-Step Workflow

1. **Define your safety taxonomy**: Enumerate the risk categories your system must detect (e.g., violence, hate speech, sexual content, self-harm, misinformation, privacy violations). Map each to a policy document or ruleset. This becomes the "risk mapping" vocabulary.

2. **Design the trajectory schema**: Create a structured output format with five sequential fields. Use XML tags or JSON keys:
   - `<evidence>`: Specific quotes, regions, or features from the input that are safety-relevant
   - `<modality>`: Which input modalities (text, image, audio, video) contain the flagged content
   - `<risk>`: Which taxonomy categories apply, with severity (low/medium/high/critical)
   - `<decision>`: Allow, flag, or block -- with the policy rule that justifies it
   - `<response>`: The user-facing moderation message or explanation

3. **Build the trajectory generation pipeline**: For each input, generate the full five-stage trajectory rather than a single label. If training a model, use a teacher ensemble (multiple strong models vote on each stage, majority consensus wins for categorical stages, embedding similarity for free-text evidence). If building with an LLM, use structured prompting that forces sequential stage completion.

4. **Implement multi-head scoring**: Create separate scoring functions for each safety dimension (quality, privacy, bias, toxicity, legal). Each scorer evaluates the `<response>` stage output on its dimension, returning a score in [-1, 1]. Keep scorer weights orthogonal -- if using learned heads, add the cosine-similarity penalty term to your loss.

5. **Aggregate scores additively**: Combine dimension scores as `R = sum(w_k * r_k)` with configurable per-dimension weights. Do NOT use multiplicative aggregation -- it collapses reward variance when any single head is uncertain.

6. **Decouple training objectives**: If fine-tuning a model, use head-wise weight subspace decoupling (orthogonal regularization on head weight vectors) and stochastic head scheduling (random permutation of head update order each epoch) to prevent gradient interference between safety dimensions.

7. **Validate intermediate stages, not just final decisions**: Evaluate accuracy at each trajectory stage independently. A system that gets the right final answer but wrong evidence or wrong risk category is still fragile -- it will fail on distribution shifts.

8. **Implement cascading early-exit**: For production latency, allow high-confidence cases to exit early. If evidence grounding finds zero safety-relevant features with high confidence, skip to "allow" without full risk mapping. Reserve the full trajectory for ambiguous cases.

9. **Build feedback loops**: Log full trajectories for human review. When reviewers disagree with a decision, annotate *which stage* went wrong (bad evidence extraction? wrong risk category? correct risk but wrong policy application?). This targeted feedback is far more valuable than flipping a binary label.

10. **Test against shortcut scenarios**: Create adversarial test cases designed to trigger shortcuts -- benign content with surface-level "dangerous" keywords, harmful content disguised in neutral language, images with misleading context. Verify the system's evidence grounding catches the true signal.

## Concrete Examples

**Example 1: Building a moderation API with trajectory reasoning**

User: "Build me a content moderation function that takes text and an optional image URL, and returns a structured safety assessment instead of just safe/unsafe."

Approach:
1. Define the trajectory schema as a Python dataclass or TypedDict
2. Implement each stage as a separate function that feeds into the next
3. Return the full trajectory as a JSON response

Output:
```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class Severity(Enum):
    NONE = "none"
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class Decision(Enum):
    ALLOW = "allow"
    FLAG = "flag"
    BLOCK = "block"

@dataclass
class ModerationTrajectory:
    evidence: list[str]          # Specific excerpts/regions flagged
    modality_flags: dict[str, bool]  # {"text": True, "image": False}
    risk_categories: dict[str, Severity]  # {"violence": "none", "hate_speech": "high"}
    decision: Decision
    policy_rule: str             # Which policy triggered the decision
    explanation: str             # Human-readable justification

RISK_TAXONOMY = [
    "violence", "hate_speech", "sexual_content",
    "self_harm", "misinformation", "privacy_violation",
]

def moderate(text: str, image_url: Optional[str] = None) -> ModerationTrajectory:
    # Stage 1: Evidence grounding
    evidence = extract_safety_evidence(text, image_url)

    # Stage 2: Modality assessment
    modality_flags = {
        "text": any(e["source"] == "text" for e in evidence),
        "image": any(e["source"] == "image" for e in evidence),
    }

    # Stage 3: Risk mapping
    risk_categories = {}
    for category in RISK_TAXONOMY:
        risk_categories[category] = classify_risk(evidence, category)

    # Stage 4: Policy decision
    max_risk = max(risk_categories.values(), key=lambda s: s.value)
    decision, policy_rule = apply_policy(risk_categories, max_risk)

    # Stage 5: Response generation
    explanation = generate_explanation(evidence, risk_categories, decision, policy_rule)

    return ModerationTrajectory(
        evidence=[e["text"] for e in evidence],
        modality_flags=modality_flags,
        risk_categories={k: v.value for k, v in risk_categories.items()},
        decision=decision,
        policy_rule=policy_rule,
        explanation=explanation,
    )
```

**Example 2: Multi-head reward scorer for evaluating moderation quality**

User: "I have a moderation model that generates responses. I need a scoring system that evaluates responses across multiple safety dimensions, not just accuracy."

Approach:
1. Implement separate scoring heads sharing a common embedding
2. Add orthogonal regularization to prevent head interference
3. Aggregate scores additively with configurable weights

Output:
```python
import torch
import torch.nn as nn

class MultiHeadRewardModel(nn.Module):
    DIMENSIONS = ["quality", "privacy", "bias", "toxicity", "legal"]

    def __init__(self, backbone_dim: int = 768, lambda_ortho: float = 0.05):
        super().__init__()
        self.lambda_ortho = lambda_ortho
        self.heads = nn.ModuleDict({
            dim: nn.Linear(backbone_dim, 1) for dim in self.DIMENSIONS
        })
        self.dim_weights = nn.Parameter(torch.ones(len(self.DIMENSIONS)) / len(self.DIMENSIONS))

    def forward(self, hidden_state: torch.Tensor) -> dict:
        scores = {}
        for dim in self.DIMENSIONS:
            scores[dim] = torch.sigmoid(self.heads[dim](hidden_state)).squeeze(-1)

        # Additive aggregation (NOT multiplicative)
        aggregate = sum(
            self.dim_weights[i] * scores[dim]
            for i, dim in enumerate(self.DIMENSIONS)
        )
        scores["aggregate"] = aggregate
        return scores

    def orthogonal_loss(self) -> torch.Tensor:
        """Penalize cosine similarity between head weight vectors."""
        weights = [self.heads[d].weight.squeeze() for d in self.DIMENSIONS]
        loss = torch.tensor(0.0, device=weights[0].device)
        for i in range(len(weights)):
            for j in range(i + 1, len(weights)):
                cos_sim = torch.cosine_similarity(weights[i], weights[j], dim=0)
                loss += cos_sim ** 2
        return self.lambda_ortho * loss

    def total_loss(self, predictions: dict, targets: dict, active_dim: str) -> torch.Tensor:
        """SSSL: single-sample single-label -- only one dimension has a label per sample."""
        mse = nn.functional.mse_loss(predictions[active_dim], targets[active_dim])
        return mse + self.orthogonal_loss()
```

**Example 3: Converting a binary classifier into trajectory-based moderation**

User: "I have an existing content filter that returns True/False. How do I refactor it to use the trajectory approach for better accuracy and explainability?"

Approach:
1. Wrap the existing classifier as one stage (the decision stage)
2. Add the four missing stages around it
3. Use the trajectory to validate and override the original classifier's shortcuts

Output:
```python
# Before: binary classifier
def is_harmful(text: str) -> bool:
    return classifier.predict(text) > 0.5

# After: trajectory-wrapped moderation
def moderate_with_trajectory(text: str, image: bytes | None = None) -> dict:
    trajectory = {}

    # Stage 1: Evidence grounding -- extract what's actually concerning
    trajectory["evidence"] = extract_evidence(text, image)

    # Stage 2: Modality assessment
    trajectory["modality"] = {
        "text_flagged": bool(trajectory["evidence"]["text_spans"]),
        "image_flagged": bool(trajectory["evidence"].get("image_regions")),
    }

    # Stage 3: Risk mapping -- categorize, don't just score
    trajectory["risks"] = map_to_taxonomy(trajectory["evidence"])

    # Stage 4: Policy decision -- use original classifier PLUS trajectory context
    raw_score = classifier.predict(text)
    has_grounded_evidence = len(trajectory["evidence"]["text_spans"]) > 0
    has_mapped_risk = any(r["severity"] != "none" for r in trajectory["risks"])

    # Override shortcuts: high score but no evidence = likely false positive
    if raw_score > 0.5 and not has_grounded_evidence:
        trajectory["decision"] = "allow"
        trajectory["override_reason"] = "classifier_flagged_but_no_evidence_found"
    elif has_grounded_evidence and has_mapped_risk:
        trajectory["decision"] = "block"
    else:
        trajectory["decision"] = "allow"

    # Stage 5: Explanation
    trajectory["explanation"] = build_explanation(trajectory)

    return trajectory
```

## Best Practices

- **Do** always generate evidence *before* making a decision. The trajectory order matters: grounding first prevents confirmation bias where the model decides early and then cherry-picks evidence.
- **Do** use additive aggregation for multi-head scores. Multiplicative aggregation causes reward collapse when any single dimension is uncertain (scores near 0.5 multiply down toward zero).
- **Do** validate each trajectory stage independently during evaluation. A correct final decision with wrong intermediate reasoning is a latent failure waiting to happen.
- **Do** apply orthogonal regularization (lambda ~0.05) between scoring head weights to prevent them from collapsing into measuring the same thing.
- **Avoid** skipping the modality assessment stage even for text-only inputs. Explicitly recording "image: not present" prevents the model from hallucinating visual evidence.
- **Avoid** training all reward heads simultaneously on every sample. Use stochastic head scheduling -- randomly select which heads update on each batch to prevent gradient interference.

## Error Handling

- **Evidence grounding finds nothing but content is genuinely harmful**: This indicates your evidence extraction is too narrow. Fall back to the full-text/full-image as evidence and flag for human review. Log these cases to improve extraction coverage.
- **Multiple risk categories fire at high severity**: This is correct behavior, not an error. Aggregate scores additively and let the policy layer decide which category takes precedence. Provide all flagged categories in the output.
- **Reward heads converge to identical scores**: Check that orthogonal regularization is active and lambda is non-zero. Inspect head weight cosine similarities -- they should stay below 0.3. Increase lambda or add dropout between the shared backbone and heads.
- **Trajectory stages contradict each other** (e.g., evidence says safe, risk says high): Treat contradictions as low-confidence cases. Route to human review and log the inconsistency. This is a key advantage of trajectory reasoning -- binary classifiers hide these internal contradictions.

## Limitations

- **Latency**: Five-stage trajectory generation is slower than single-pass binary classification. For real-time use cases with millions of requests per second, use trajectory reasoning only for borderline cases (classifier score between 0.3-0.7) and fast-path clear cases.
- **Taxonomy dependency**: Risk mapping quality depends entirely on how well your safety taxonomy covers actual harms. Novel harm categories (e.g., new forms of AI-generated misinformation) require taxonomy updates and retraining.
- **Not a replacement for human review**: Dense trajectories make moderation *auditable* and *debuggable*, but they don't eliminate the need for human oversight on edge cases and policy evolution.
- **Training data for trajectories is expensive**: Generating five-stage annotations requires either strong teacher models or skilled human annotators. The UniTrace consensus approach (majority vote from multiple models) helps but requires access to multiple capable VLMs.
- **Single-modality inputs still benefit but less dramatically**: The biggest gains are on multimodal content where binary classifiers struggle most. For text-only tasks, the improvement over strong binary classifiers is smaller.

## Reference

[From Sparse Decisions to Dense Reasoning (arXiv:2602.02536)](https://arxiv.org/abs/2602.02536v1) -- Focus on Section 3 (trajectory decomposition and the three lemmas justifying it), Section 4 (UniRM multi-head architecture with orthogonal regularization), and Table 2 (ablation showing each trajectory stage's contribution to final accuracy).