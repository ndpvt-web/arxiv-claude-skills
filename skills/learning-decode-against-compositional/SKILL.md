---
name: "learning-decode-against-compositional"
description: "Detect and mitigate compositional hallucinations in video multimodal LLM outputs using triple-pathway contrastive decoding (TriCD). Use when: 'check this video QA pipeline for hallucinations', 'build a video understanding evaluation harness', 'add contrastive decoding to my VLLM', 'reduce hallucination in video captioning', 'evaluate my model on compositional video reasoning', 'implement perturbation-based calibration for video models'."
---

# Compositional Hallucination Detection and Mitigation for Video LLMs

This skill teaches Claude to implement the TriCD (Triple-pathway Contrastive Decoding) framework from the paper "Learning to Decode Against Compositional Hallucination in Video Multimodal Large Language Models." TriCD addresses a critical blind spot in video multimodal LLMs: compositional hallucinations that arise when a model reasons incorrectly across multiple interacting spatial and temporal factors simultaneously (e.g., confusing both the object performing an action AND the temporal order of events). The technique works by running three parallel inference pathways -- original, negatively perturbed, and saliency-enhanced -- then calibrating final logits to suppress hallucination-prone reasoning while amplifying visually grounded evidence.

## When to Use

- When building or improving a video question-answering (VQA) pipeline and the model confabulates details about object interactions, temporal sequences, or camera movements
- When implementing contrastive decoding for any vision-language model to reduce hallucination at inference time without retraining the base model
- When constructing an evaluation benchmark for video understanding that needs to test compositional reasoning (not just isolated fact recall)
- When designing adversarial QA datasets with trap options like "All are correct" or "None of the above" to stress-test model reasoning
- When adding perturbation-based robustness testing to a video ML pipeline (temporal shuffling, spatial masking, motion blur)
- When implementing saliency-guided visual grounding to focus a model's attention on relevant objects and motions in video frames

## Key Technique: Triple-Pathway Contrastive Decoding

**The Core Insight.** Standard contrastive decoding uses two pathways (original vs. corrupted). TriCD adds a third saliency-enhanced pathway, creating a push-pull calibration: the negative pathway identifies what the model hallucinates when visual evidence is degraded, the positive pathway identifies what the model attends to when visual evidence is sharpened, and the original pathway serves as baseline. The final logit at each decoding step is:

```
q_t = q_t^o + alpha_1 * (q_t^p - q_t^o) + alpha_2 * (q_t^o - q_t^n)
```

where `q_t^o` is the original logit, `q_t^p` is the saliency-enhanced logit, `q_t^n` is the perturbed-negative logit, `alpha_1=0.8` amplifies grounded evidence, and `alpha_2=0.4` suppresses hallucination patterns. This formula applies directed residual corrections that widen the margin between grounded reasoning and hallucinated reasoning.

**Adaptive Perturbation Controller (APC).** Rather than using fixed corruptions, TriCD learns which perturbations to apply per video via reinforcement learning. The APC selects from eight operations -- temporal frame shuffling, spatial region masking, frame dropping, motion blur, color jittering, contrast modification, and others -- using a dual-stage cross-attention mechanism over video hidden states and tool description embeddings. Tools are sampled via Bernoulli distributions during RL training, enabling multiple simultaneous perturbations.

**Saliency-Guided Enhancement (SGE).** The positive pathway computes spatial saliency from DINOv2/v3 CLS-token attention weights (capturing foreground object importance) and temporal saliency from Farneback optical flow with Gaussian + band-pass filtering (isolating intentional motion). A learnable gating network fuses these into per-token weights: `X_v' = w_sal * X_v`, anchoring the model to critical objects and their dynamic interactions.

## Step-by-Step Workflow

### 1. Classify the hallucination type taxonomy

Categorize the video understanding task against the eight-type taxonomy: Object (entity presence/identity), Scene (environment/setting), Event (causal occurrences), Action (physical movement), Relation (spatial/logical interactions), Attribute (color/size/material), Temporal (chronological order/duration), and Camera (cinematic lens dynamics). Determine whether the task involves isolated (single-type) or compositional (multi-type) reasoning.

### 2. Set up the three-pathway inference architecture

Structure the inference pipeline with three parallel forward passes through the frozen VLLM:
- **Original pass**: Standard `logit(y_t | V, T, y_<t)` with unmodified video `V` and text prompt `T`
- **Negative pass**: `logit(y_t | V^-, T, y_<t)` with perturbed video `V^-`
- **Positive pass**: `logit(y_t | X_v', T, y_<t)` with saliency-reweighted visual tokens `X_v'`

```python
class TriCDDecoder:
    def __init__(self, vllm_model, alpha1=0.8, alpha2=0.4):
        self.model = vllm_model
        self.alpha1 = alpha1
        self.alpha2 = alpha2
        self.apc = AdaptivePerturbationController()
        self.sge = SaliencyGuidedEnhancement()

    def decode_step(self, video_frames, text_tokens, past_tokens):
        logits_orig = self.model(video_frames, text_tokens, past_tokens)
        neg_frames = self.apc.perturb(video_frames)
        logits_neg = self.model(neg_frames, text_tokens, past_tokens)
        enhanced_vis = self.sge.enhance(video_frames)
        logits_pos = self.model(enhanced_vis, text_tokens, past_tokens)
        calibrated = (logits_orig
                      + self.alpha1 * (logits_pos - logits_orig)
                      + self.alpha2 * (logits_orig - logits_neg))
        return calibrated
```

### 3. Implement the Adaptive Perturbation Controller

Build a perturbation selector that takes video hidden states and chooses which corruption operations to apply. Use cross-attention between learnable query tokens and tool description embeddings derived from the VLLM's text encoder. Apply selected perturbations to construct `V^-`:

```python
PERTURBATION_OPS = {
    "temporal_shuffle": lambda frames: frames[torch.randperm(len(frames))],
    "spatial_mask": lambda frames: mask_random_regions(frames, ratio=0.3),
    "frame_drop": lambda frames: drop_frames(frames, drop_rate=0.2),
    "motion_blur": lambda frames: apply_motion_blur(frames, kernel=7),
    "color_jitter": lambda frames: torchvision.transforms.ColorJitter(0.4, 0.4, 0.4, 0.1)(frames),
    "contrast_mod": lambda frames: adjust_contrast(frames, factor=0.5),
}

class AdaptivePerturbationController(nn.Module):
    def __init__(self, hidden_dim=768, n_tools=6):
        super().__init__()
        self.query = nn.Parameter(torch.randn(1, hidden_dim))
        self.cross_attn = nn.MultiheadAttention(hidden_dim, 8)
        self.tool_proj = nn.Linear(hidden_dim, n_tools)

    def select_tools(self, video_hidden):
        attn_out, _ = self.cross_attn(self.query, video_hidden, video_hidden)
        probs = torch.sigmoid(self.tool_proj(attn_out))  # per-tool Bernoulli
        return probs

    def perturb(self, frames, video_hidden):
        probs = self.select_tools(video_hidden)
        selected = (probs > 0.5) if not self.training else (torch.bernoulli(probs) > 0)
        result = frames.clone()
        for i, (name, op) in enumerate(PERTURBATION_OPS.items()):
            if selected[0, i]:
                result = op(result)
        return result
```

### 4. Implement the Saliency-Guided Enhancement module

Compute spatial saliency from a pretrained DINO model's CLS attention and temporal saliency from optical flow, then fuse them with a learned gate:

```python
class SaliencyGuidedEnhancement(nn.Module):
    def __init__(self, hidden_dim=768):
        super().__init__()
        self.gate = nn.Sequential(nn.Linear(hidden_dim * 2, 1), nn.Sigmoid())

    def spatial_saliency(self, frames, dino_model):
        # Extract CLS-to-patch attention from DINO's final layer
        with torch.no_grad():
            attn = dino_model.get_last_selfattention(frames)
        cls_attn = attn[:, :, 0, 1:]  # CLS token attending to patches
        return cls_attn.mean(dim=1)  # average over heads

    def temporal_saliency(self, frames):
        # Farneback optical flow between consecutive frames
        flows = compute_optical_flow(frames)  # (T-1, H, W)
        smoothed = gaussian_filter(flows, sigma=1.5)
        bandpassed = temporal_bandpass(smoothed, low=0.5, high=5.0)
        return bandpassed.abs().mean(dim=-1)

    def enhance(self, visual_tokens, spatial_sal, temporal_sal):
        combined = torch.cat([spatial_sal, temporal_sal], dim=-1)
        beta = self.gate(combined)  # fusion weight in [0, 1]
        w_sal = beta * spatial_sal + (1 - beta) * temporal_sal
        return w_sal.unsqueeze(-1) * visual_tokens
```

### 5. Train the APC and SGE via REINFORCE

Freeze the base VLLM. Use a binary reward signal (+1 correct, -1 incorrect) with exponential moving average baseline to reduce variance:

```python
def train_tricd_policy(tricd, dataloader, optimizer, ema_factor=0.99):
    baseline = 0.0
    for video, question, answer in dataloader:
        prediction = tricd.decode(video, question)
        reward = 1.0 if prediction == answer else -1.0
        log_prob = tricd.apc.log_prob() + tricd.sge.log_prob()
        loss = -log_prob * (reward - baseline)
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        baseline = ema_factor * baseline + (1 - ema_factor) * reward
```

### 6. Build adversarial evaluation questions

When constructing QA pairs for benchmarking, include trap options ("All are correct", "None of the above") and ensure compositional questions combine multiple hallucination types. For example, a question testing both Temporal + Action might ask about the order in which specific actions occur.

### 7. Evaluate with the fine-grained taxonomy

Report accuracy broken down by: (a) isolated vs. compositional, (b) each of the 8 hallucination types, (c) question format (Yes/No vs. multiple-choice). This reveals whether improvements come from better spatial reasoning, temporal reasoning, or both.

## Concrete Examples

**Example 1: Adding contrastive decoding to an existing video QA model**

User: "My VideoLLaMA model hallucinates about the order of events in videos. How can I fix this without retraining?"

Approach:
1. Wrap the existing model in a TriCD decoder with the base model frozen
2. Implement temporal frame shuffling and frame dropping as negative perturbations (these target temporal hallucinations specifically)
3. Compute temporal saliency via optical flow to build the positive pathway
4. Apply the calibration formula with `alpha1=0.8, alpha2=0.4`
5. Train only the APC and SGE parameters on a small labeled VQA set using REINFORCE

Output:
```python
from tricd import TriCDDecoder, AdaptivePerturbationController, SaliencyGuidedEnhancement

tricd = TriCDDecoder(
    vllm_model=my_videollama,
    perturbation_ops=["temporal_shuffle", "frame_drop"],  # target temporal errors
    alpha1=0.8,
    alpha2=0.4,
)
# Train APC+SGE on 500 labeled video QA pairs (base model stays frozen)
train_tricd_policy(tricd, my_train_loader, lr=1e-4, epochs=3)
# At inference: calibrated decoding with no base model changes
answer = tricd.decode(video_frames, question_text)
```
Expected improvement: ~10% accuracy gain on temporal reasoning questions.

**Example 2: Building a compositional hallucination evaluation benchmark**

User: "I need to evaluate whether my video model can handle complex multi-factor reasoning, not just simple recognition."

Approach:
1. Define the 8-type taxonomy: Object, Scene, Event, Action, Relation, Attribute, Temporal, Camera
2. For each video, generate single-type (isolated) and multi-type (compositional) questions
3. Add adversarial answer options to prevent shortcut reasoning
4. Score models on both isolated accuracy and compositional accuracy to measure the degradation gap

Output:
```python
benchmark_schema = {
    "video_id": "v_001",
    "domain": "real_world",  # or "ai_generated"
    "questions": [
        {
            "type": "S_MCQA",
            "hallucination_types": ["action"],
            "question": "What action does the person perform after picking up the cup?",
            "options": [
                "A) Drinks from it",
                "B) Places it on the shelf",
                "C) Hands it to another person",
                "D) None of the above"  # adversarial trap
            ],
            "answer": "B"
        },
        {
            "type": "C_MCQA",
            "hallucination_types": ["action", "temporal", "relation"],
            "question": "Which sequence of interactions between the two people is correct?",
            "options": [
                "A) Person A waves, then Person B approaches and shakes hands",
                "B) Person B approaches first, then both wave simultaneously",
                "C) All are correct",  # adversarial trap
                "D) None of the above"
            ],
            "answer": "A"
        }
    ]
}
```

**Example 3: Implementing saliency-guided visual grounding for video captioning**

User: "My video captioning model describes objects that aren't actually important to the scene. How do I focus it on what matters?"

Approach:
1. Extract spatial saliency maps using DINO's CLS attention over video frames
2. Extract temporal saliency using optical flow to identify frames with meaningful motion
3. Fuse with a learned gate and reweight visual tokens before feeding to the LLM decoder
4. The model now attends preferentially to foreground objects and intentional motion

Output:
```python
sge = SaliencyGuidedEnhancement(hidden_dim=768)
dino = load_dino_v2()

for batch in video_loader:
    spatial_sal = sge.spatial_saliency(batch.frames, dino)
    temporal_sal = sge.temporal_saliency(batch.frames)
    enhanced_tokens = sge.enhance(batch.visual_tokens, spatial_sal, temporal_sal)
    # Feed enhanced_tokens to your caption decoder instead of raw visual_tokens
    caption = caption_model.generate(enhanced_tokens, batch.prompt)
```

## Best Practices

- **Do** keep the base VLLM frozen during TriCD training. Only the lightweight APC and SGE modules are trained, preserving the model's general capabilities while adding hallucination resistance.
- **Do** use multiple perturbation operations simultaneously. The RL-trained APC learns that combining temporal shuffling with spatial masking is more effective than either alone for compositional hallucinations.
- **Do** include adversarial answer options ("All are correct", "None of the above") in evaluation sets. Models that rely on surface-level pattern matching will fail these, revealing true compositional reasoning ability.
- **Do** report results broken down by isolated vs. compositional hallucination types. Aggregate accuracy hides the most important signal: how much worse the model gets when multiple reasoning factors interact.
- **Avoid** setting `alpha2` higher than `alpha1`. The positive pathway correction should dominate; the negative suppression is a secondary regularizer. The paper finds `alpha1=0.8, alpha2=0.4` optimal.
- **Avoid** using fixed perturbation strategies for all videos. The whole point of the adaptive controller is that different videos need different corruptions -- a static scene needs spatial perturbation while an action-heavy clip needs temporal perturbation.

## Error Handling

- **Three-pass latency**: TriCD requires three forward passes per decoding step. If latency is critical, cache the APC's perturbation selection per video (it only depends on the video hidden states, not on the decoding step) and precompute the saliency maps offline.
- **Degenerate perturbations**: If the APC converges to always selecting the same perturbation(s), add entropy regularization to the Bernoulli probabilities to encourage exploration during RL training.
- **Saliency collapse**: If the SGE gate `beta` saturates to 0 or 1, the module ignores one saliency source entirely. Monitor `beta` during training and add a regularization term penalizing extreme values.
- **Reward sparsity**: The binary +1/-1 reward can cause high-variance gradients. The exponential moving average baseline is essential. If training is unstable, increase the EMA factor (e.g., 0.999) or use a larger batch size.
- **OOM on long videos**: Three-pass decoding triples memory for KV cache. Use frame subsampling or sliding-window attention to keep memory bounded.

## Limitations

- TriCD adds inference overhead (3x forward passes) that may be unacceptable for real-time applications. It is best suited for offline evaluation or quality-critical pipelines.
- The technique is demonstrated on multiple-choice and yes/no QA formats. Its effectiveness on open-ended video captioning or long-form generation is not validated in the paper.
- The APC and SGE require a small labeled training set with correct answers for RL optimization. Zero-shot application without any training data is not supported.
- Camera-based hallucinations (a novel type in this work) require videos with meaningful cinematographic variation. For static webcam footage, this hallucination axis is irrelevant.
- The method assumes the base VLLM's visual encoder can extract meaningful spatial and temporal features. If the backbone's visual encoding is fundamentally weak, contrastive decoding cannot compensate.

## Reference

**Paper**: [Learning to Decode Against Compositional Hallucination in Video Multimodal Large Language Models](https://arxiv.org/abs/2602.00559v1) (Xing et al., 2026). Look for Section 3 (TriCD framework architecture), Section 4.2 (APC/SGE implementation), and Table 2 (per-type accuracy breakdown showing where compositional hallucinations are worst).