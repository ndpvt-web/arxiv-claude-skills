---
name: "toward-cognitive-supersensing-multimodal"
description: "Apply Cognitive Supersensing to multimodal reasoning tasks -- augmenting text-only chain-of-thought with latent visual reasoning steps that maintain spatial and structural information. Use when: 'reason about this visual puzzle', 'solve this spatial reasoning problem', 'analyze this abstract pattern', 'build a visual reasoning pipeline', 'implement visuospatial chain-of-thought', 'add visual imagination to my MLLM agent'."
---

# Cognitive Supersensing for Multimodal Reasoning

This skill teaches Claude to apply the Cognitive Supersensing paradigm from Li et al. (2026) when building or improving multimodal AI systems that must reason about visual content. The core insight: text-only chain-of-thought fails on tasks requiring mental rotation, pattern induction, spatial simulation, or geometric transformation. Cognitive Supersensing introduces a **Latent Visual Imagery Prediction (LVIP) head** that generates internal visual reasoning states alongside text, grounding each reasoning step in a continuous visual embedding rather than forcing spatial logic through discrete tokens. This skill guides you in architecting pipelines, training loops, and inference strategies that integrate visual latent reasoning with standard language-based CoT.

## When to Use

- When building or improving a multimodal LLM pipeline that must solve abstract visual reasoning tasks (Raven's Progressive Matrices, Bongard problems, ARC-AGI, spatial QA)
- When a user's MLLM produces correct text reasoning but wrong final answers on visuospatial tasks, indicating the text bottleneck described in the paper
- When implementing a training pipeline that needs to jointly optimize text generation and visual representation prediction
- When designing a reinforcement learning stage for multimodal reasoning that requires reward grounding in visual similarity rather than text-match alone
- When the user wants to evaluate cognitive capabilities (fluid intelligence, crystallized intelligence, visuospatial cognition, mental simulation, visual routines) of an MLLM
- When building an inference-time strategy that samples multiple reasoning chains and selects the best via evidence scoring

## Key Technique

**The Text Bottleneck Problem.** Standard MLLM reasoning encodes everything -- including spatial relations, rotations, and geometric patterns -- as text tokens. But many visual subroutines (mentally rotating a shape, simulating physical dynamics, inducing rules from pattern matrices) are naturally expressed as continuous geometric transformations, not discrete token sequences. Forcing these through language loses structural fidelity and introduces reasoning errors.

**Latent Visual Imagery Prediction (LVIP).** The paper's solution adds a lightweight two-layer MLP head on top of the LLM's hidden states. During training, this head predicts the latent visual embedding of the correct answer image. The training objective combines standard autoregressive cross-entropy loss with an MSE loss between the predicted and target visual embeddings: `L = L_CE + beta * MSE(h_pred, h_target)`. This joint optimization forces the model to build internal visual representations that are grounded in actual visual content, not just text descriptions of that content.

**Three-Stage Training.** (1) A teacher MLLM generates reasoning chains filtered for correctness, creating an augmented dataset. (2) Supervised fine-tuning jointly trains text generation and LVIP embedding prediction. (3) Reinforcement learning via Generative Flow Networks samples diverse reasoning trajectories, scored by a composite reward combining answer evidence and LVIP representation grounding. The RL reward is `R = alpha * R_answer + gamma * R_lvip`, where `R_lvip = -||h_pred - h_target||^2`. A sparse anchor-based interpolation reduces the cost of reward computation during RL by evaluating the scorer only at stride intervals and linearly interpolating between anchors.

## Step-by-Step Workflow

1. **Audit the reasoning failure mode.** Collect examples where text-only CoT produces plausible reasoning but wrong answers. Classify failures into visuospatial categories: rotation errors, pattern completion failures, spatial relation mistakes, simulation inaccuracies. This tells you whether LVIP will help.

2. **Prepare the visual encoder.** Select a pre-trained vision encoder (e.g., SigLIP, CLIP ViT-L). Extract visual features for all images in the dataset, including answer-option images. Store these as target embeddings `h_target` for LVIP supervision.

3. **Generate reasoning chains (Stage I).** Use a capable teacher MLLM to produce step-by-step rationales for each training example. Filter outputs: keep only chains whose predicted answer matches ground truth. This creates dataset `D_chain` of (image, question, reasoning_chain, answer) tuples.

4. **Implement the LVIP head.** Add a two-layer MLP that takes the LLM's hidden states at answer-option token positions, applies average pooling across option tokens, and predicts a visual embedding: `h_pred = MLP(avg_pool(H_option))`. The target is the pre-extracted embedding of the correct answer image.

5. **Define the joint loss function.** Combine autoregressive text loss with LVIP MSE loss: `L = -sum(log q(x_t | x_<t)) + beta * MSE(h_pred, h_target)`. Start with `beta = 0.1` and tune based on validation performance. Too high a beta degrades text fluency; too low fails to ground visual representations.

6. **Run supervised fine-tuning (Stage II).** Train on `D_chain` with the joint loss. Use standard practices: learning rate ~2e-5, cosine schedule, gradient accumulation for large batches. Monitor both text generation quality and LVIP MSE convergence independently.

7. **Implement RL with GFlowNet sampling (Stage III).** Define the composite reward: `R = alpha * log q_frozen(y | X, Z) + gamma * (-||h_pred - h_target||^2)`. The first term uses a frozen copy of the Stage II model as an answer-evidence scorer. The second term grounds reasoning in visual similarity. Use sparse anchor interpolation (stride lambda = 4-8 tokens) to reduce scorer evaluations.

8. **Build the inference pipeline.** At inference, sample N diverse reasoning chains from the trained policy. For each chain, compute a length-normalized evidence score: `S_i = (1 / (|Z_i| + |y_i|)) * log q_frozen(y_i | X, Z_i)`. Select the answer with the highest score. N=8 to 16 is a reasonable starting point.

9. **Evaluate across cognitive dimensions.** Test on five axes: fluid intelligence (novel rule induction), crystallized intelligence (leveraging learned knowledge), visuospatial cognition (3D spatial reasoning), mental simulation (inferring hidden dynamics), and visual routines (efficient visual search). This decomposition reveals which capabilities improved and which need further work.

10. **Iterate on beta and reward weights.** If the model hallucinates visual features, increase `beta`. If text reasoning degrades, decrease it. If RL collapses to a single reasoning mode, increase GFlowNet diversity pressure. Track the ratio of LVIP loss to CE loss as a diagnostic signal.

## Concrete Examples

**Example 1: Adding LVIP to an existing MLLM training pipeline**

User: "I have a LLaVA-based model fine-tuned for visual QA but it fails on Raven's Progressive Matrices. How can I add visual reasoning grounding?"

Approach:
1. Extract SigLIP embeddings for all answer-option images in the Raven dataset
2. Add a two-layer MLP head (hidden_dim=1024) after the LLM backbone
3. Modify the training loop to compute joint loss:

```python
# LVIP head: predicts visual embedding from LLM hidden states
class LVIPHead(nn.Module):
    def __init__(self, hidden_dim, visual_dim):
        super().__init__()
        self.mlp = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim),
            nn.GELU(),
            nn.Linear(hidden_dim, visual_dim),
        )

    def forward(self, hidden_states, option_mask):
        # Average pool over answer-option token positions
        h_opt = (hidden_states * option_mask.unsqueeze(-1)).sum(1)
        h_opt = h_opt / option_mask.sum(1, keepdim=True).clamp(min=1)
        return self.mlp(h_opt)

# Joint loss computation
ce_loss = compute_autoregressive_loss(logits, labels)
h_pred = lvip_head(hidden_states, option_token_mask)
lvip_loss = F.mse_loss(h_pred, target_visual_embedding)
total_loss = ce_loss + beta * lvip_loss
```

Output: The model now jointly optimizes text generation and visual grounding. On Raven-style tasks, expect 6-10 point accuracy improvement from LVIP alone before RL.

---

**Example 2: Building an inference pipeline with evidence-based selection**

User: "My multimodal model sometimes gets the right reasoning but picks the wrong answer. How do I improve answer selection?"

Approach:
1. Freeze a copy of the fine-tuned model as an evidence scorer
2. Sample multiple reasoning chains at inference time
3. Score each chain and select the best answer:

```python
def evidence_scored_inference(model, scorer, image, question, n_samples=12):
    candidates = []
    for _ in range(n_samples):
        # Sample a reasoning chain with temperature > 0
        chain, answer = model.generate(image, question, temperature=0.7)
        # Compute length-normalized evidence score
        log_prob = scorer.score(image, chain, answer)
        score = log_prob / (len(chain) + len(answer))
        candidates.append((answer, score, chain))

    # Group by answer, aggregate scores
    answer_scores = defaultdict(list)
    for answer, score, chain in candidates:
        answer_scores[answer].append(score)

    # Select answer with highest mean evidence
    best_answer = max(answer_scores, key=lambda a: sum(answer_scores[a]) / len(answer_scores[a]))
    return best_answer
```

Output: This majority-voting-with-evidence approach typically improves accuracy by 3-5 points over greedy decoding on cognitive reasoning benchmarks.

---

**Example 3: Evaluating an MLLM across cognitive dimensions**

User: "I want to benchmark my vision-language model on cognitive reasoning, not just standard VQA."

Approach:
1. Organize evaluation into five cognitive dimensions following the CogSense-Bench structure
2. Source tasks for each dimension:

```
Cognitive Dimension          | Task Sources                    | What It Tests
-----------------------------|--------------------------------|---------------------------
Fluid Intelligence           | Bongard, RAVEN, ARC-AGI        | Novel rule induction
Crystallized Intelligence    | Science diagrams, world-knowledge VQA | Learned knowledge application
Visuospatial Cognition       | 3D shape matching, mental rotation    | Spatial/geometric reasoning
Mental Simulation            | Physics prediction, hidden dynamics   | Hypothetico-deductive inference
Visual Routines              | Visual search, attention binding      | Efficient visual processing
```

3. Run evaluation per dimension, compute accuracy breakdowns, and identify the weakest axis for targeted improvement

Output: A radar chart or table showing per-dimension accuracy, revealing whether your model's bottleneck is spatial reasoning (add LVIP), knowledge retrieval (more data), or simulation (more RL training).

## Best Practices

- **Do:** Start with Stage II (SFT + LVIP) before adding RL. The joint supervised loss alone yields meaningful gains and is much easier to debug than the full three-stage pipeline.
- **Do:** Use a frozen pre-trained visual encoder for target embeddings. Training the encoder end-to-end during LVIP optimization causes representation drift that destabilizes the MSE target.
- **Do:** Monitor LVIP MSE loss independently from CE loss. A diverging MSE with decreasing CE means the model is ignoring visual grounding -- increase beta.
- **Do:** Use sparse anchor interpolation during RL (stride 4-8) to keep training cost manageable. Full-density reward evaluation is 4-8x more expensive with negligible accuracy gain.
- **Avoid:** Setting beta too high (>1.0) initially. This causes the model to prioritize visual embedding prediction over coherent text generation, producing garbled reasoning chains.
- **Avoid:** Using greedy decoding at inference for cognitive tasks. The evidence-scoring approach with N sampled chains consistently outperforms greedy by exploiting reasoning diversity.
- **Avoid:** Applying this technique to tasks where text-only CoT is already sufficient (standard factual VQA, OCR, captioning). LVIP adds overhead and complexity; reserve it for genuinely visuospatial tasks.

## Error Handling

- **LVIP loss diverges during training:** The target embeddings may be on a different scale than the MLP output. Add a LayerNorm before the final MLP projection, or normalize both predicted and target embeddings to unit length before computing MSE.
- **RL reward collapse (all chains get similar scores):** The frozen scorer may be too confident. Lower the scorer's softmax temperature to spread out probability mass, or increase the GFlowNet exploration parameter.
- **Answer-evidence scorer disagrees with LVIP reward:** When `R_answer` and `R_lvip` point to different answers, the model receives conflicting gradients. Diagnose by logging reward components separately. If persistent, reduce gamma to let answer evidence dominate while still providing mild visual grounding.
- **Stage I generates low-quality chains:** If the teacher MLLM produces chains with <30% ground-truth alignment, switch to a stronger teacher or use human-written rationales for the hardest examples.

## Limitations

- **Requires answer-option images.** The LVIP head predicts embeddings of answer images, so the technique applies most directly to multiple-choice visual reasoning tasks. Open-ended generation tasks need an adapted formulation where the target embedding comes from a generated or retrieved reference image.
- **Visual encoder dependency.** The quality of LVIP grounding is bounded by the visual encoder's representation quality. If the encoder cannot distinguish fine-grained visual differences relevant to the task, LVIP supervision provides a noisy signal.
- **Computational cost of three-stage training.** The full pipeline (chain generation, SFT, RL) requires 3x the training budget of standard fine-tuning. For resource-constrained settings, Stage II alone captures most of the benefit.
- **Not a replacement for perception improvements.** If the model fails because it cannot *see* relevant visual features (low resolution, occluded objects), LVIP cannot compensate. It improves *reasoning about* what is perceived, not perception itself.
- **Benchmark-specific evaluation.** The CogSense-Bench results are strong (73.8% vs GPT-5.2's 40.3%), but generalization to arbitrary real-world cognitive tasks is not yet established beyond EMMA math/science subsets.

## Reference

Li, B., Shen, Y., Liu, Y., Xu, Y., & Liu, J. (2026). *Toward Cognitive Supersensing in Multimodal Large Language Model.* arXiv:2602.01541v1. [https://arxiv.org/abs/2602.01541v1](https://arxiv.org/abs/2602.01541v1)

Key sections to read: Section 3 (LVIP head architecture and joint loss formulation), Section 4 (GFlowNet RL with sparse anchor interpolation), Section 5.3 (ablation showing LVIP contributes 6-10 point gains before RL).