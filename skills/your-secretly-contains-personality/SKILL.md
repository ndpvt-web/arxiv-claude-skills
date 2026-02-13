---
name: "your-secretly-contains-personality"
description: "Extract and activate persona-specialized subnetworks from LLMs using activation-guided pruning and contrastive masking. Use when: 'build a persona extraction pipeline', 'find personality subnetworks in a model', 'create contrastive persona masks', 'implement activation-guided pruning for personas', 'extract opposing personality circuits from weights', 'build MBTI subnetwork discovery tool'."
---

This skill enables Claude to build data pipelines, analysis tools, and prompt-engineering workflows based on the Persona Subnetwork Discovery method from "Your Language Model Secretly Contains Personality Subnetworks" (ICLR 2026). The core insight: LLMs already contain separable persona-specialized circuits in their weights, discoverable through activation statistics on small calibration sets. By computing per-neuron importance scores (weight magnitude x activation magnitude) and applying top-K masking, you can isolate lightweight binary masks that produce strong persona alignment — without any training. For opposing persona pairs (introvert vs. extrovert), a contrastive pruning variant maximizes parameter disentanglement. This skill teaches you to implement these pipelines, design calibration datasets, compute and store masks, and build persona-switching inference systems.

## When to Use

- When the user asks to extract or discover personality subnetworks from a pretrained language model
- When building a pipeline to create persona-specific model variants without fine-tuning
- When implementing activation-guided structured pruning (Wanda-style) for behavioral specialization
- When designing calibration datasets for persona dimensions (MBTI, Big Five, custom role taxonomies)
- When the user wants to compare persona encoding across model layers (mechanistic interpretability)
- When building a system that switches between personas at inference time using lightweight binary masks
- When implementing contrastive pruning to separate opposing behavioral traits in a single model

## Key Technique

**Activation-Guided Persona Masking.** For each persona *p*, run a small calibration dataset D_p (128+ examples) through the model and collect mean activation magnitudes per neuron per layer: A_p[j] = E[|h_j(x)|] over D_p. Then compute importance scores for each weight: S_ij = |w_ij| * A_p[j]. This combines weight magnitude (structural importance) with activation frequency (persona-specific usage). Apply row-wise top-K selection: for each output neuron, keep the K = (1 - sparsity) * n highest-scoring input connections. The result is a binary mask M_p that, when applied element-wise to the weight matrices (y = (W * M_p)x + b), produces a sparse subnetwork specialized for persona p. Typical sparsity: 40-60%. Masks are stored as binary tensors and applied to all Linear layers in attention (Q, K, V, O projections) and MLP blocks; embeddings and LM head remain unpruned.

**Contrastive Pruning for Opposing Personas.** When working with binary-opposing persona pairs (e.g., introvert vs. extrovert), standard masking produces overlapping subnetworks. Contrastive pruning explicitly maximizes separation: compute normalized importance scores for both personas, then calculate divergence C_ij = |S_ij_normalized_p+ - S_ij_normalized_p-|. Parameters with high divergence are allocated to whichever persona activates them more, producing disjoint masks. An alternative formulation uses standardized activation differences: S_ij = |w_ij| * relu((mu_p+ - mu_p-) / sqrt(sigma_p+ + sigma_p- + eps)). On benchmarks, contrastive pruning outperforms prompting by 10-25 percentage points on persona classification tasks while preserving >98% of general capability (MMLU, HellaSwag).

**Why This Matters for Practitioners.** The method is entirely training-free — mask computation takes minutes, not hours. It demonstrates that persona behavior is structurally encoded: MBTI I/E and F/T dimensions show ~1.3% and ~1.1% differential mask ratios (fraction of weights unique to one pole), while N/S and J/P show ~0.75%, suggesting some personality dimensions are more strongly encoded than others. This has direct implications for building controllable persona systems, interpretability research, and efficient model personalization.

## Step-by-Step Workflow

1. **Define the persona taxonomy.** Specify the target personas as a structured schema — MBTI dimensions, Big Five traits, character roles, or custom behavioral axes. For binary-opposing pairs, explicitly list both poles (e.g., {introvert, extrovert}).

2. **Build calibration datasets.** For each persona, assemble 128-500 prompt-response pairs that exhibit the target behavior. Format as (x_i, y_i) tuples. Use existing personality questionnaire corpora, roleplay dialogue datasets, or generate synthetic examples via prompted LLM completions. Pad/truncate to fixed length (512 tokens max).

3. **Implement the forward-pass activation collector.** Hook into every Linear layer in the transformer (attention Q/K/V/O projections and MLP up/down/gate projections). For each calibration sample, record the input activation vector h(x) per layer. Compute running mean of absolute activation magnitudes: A_p[j] = mean(|h_j|) across all samples for persona p.

4. **Compute importance scores.** For each weight matrix W in each Linear layer, compute the element-wise score matrix: S_ij = |W_ij| * A_p[j]. This is a single element-wise multiplication — no gradients needed.

5. **Apply row-wise top-K masking.** For each row i of the score matrix (each output neuron), select the top K = floor((1 - rho) * n_cols) entries. Set M_ij = 1 for selected entries, 0 otherwise. Use sparsity ratio rho between 0.4 and 0.6 (start with 0.5).

6. **[For opposing pairs] Apply contrastive pruning.** Normalize importance scores column-wise for each persona: S_tilde_ij = S_ij / sum_k(S_ik). Compute divergence: C_ij = |S_tilde_ij_p+ - S_tilde_ij_p-|. Assign each parameter to the persona with the larger normalized score. Prune low-divergence parameters from both masks.

7. **Store masks as sparse binary tensors.** Save one mask file per persona per layer. Use compressed sparse formats (CSR/CSC) or bitmask arrays to minimize storage. A 7B model at 50% sparsity needs ~3.5GB of binary mask data (1 bit per parameter).

8. **Implement masked inference.** At inference time, load the target persona mask and apply element-wise: output = (W * M_p) @ input + bias. For persona switching, swap the mask tensor — no weight modification needed. Optionally use soft gating: G = M_p + gamma * (1 - M_p) with gamma in [0, 0.1) for smoother transitions.

9. **Validate persona alignment.** Run standardized personality questionnaires (MBTI 93-question instrument, BFI-44) through the masked model. Score responses and verify the target persona emerges. Compare against prompting-only and RAG baselines.

10. **Profile general capability retention.** Evaluate the masked model on standard benchmarks (MMLU, HellaSwag, TruthfulQA) to confirm degradation stays below 2% at 50-60% sparsity.

## Concrete Examples

**Example 1: Building an MBTI Persona Extraction Pipeline**

User: "I want to extract introvert and extrovert subnetworks from Llama-3-8B and compare their activation patterns."

Approach:
1. Create two calibration datasets: 128 prompt-response pairs reflecting introverted communication style (reserved, reflective, prefers writing) and 128 for extroverted style (outgoing, expressive, prefers talking).
2. Implement activation hooks on all 32 transformer layers.
3. Run calibration data through the model, collecting per-layer activation statistics.
4. Compute importance scores and generate binary masks at 50% sparsity.
5. Apply contrastive pruning to maximize I/E separation.
6. Analyze differential mask ratios per layer to identify where introversion vs. extroversion is most strongly encoded.

Output:
```python
# Pseudo-implementation of the core pipeline
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3-8B")
tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3-8B")

# Step 1: Collect activations per persona
activation_stats = {}  # {persona: {layer_name: mean_abs_activation}}

def hook_fn(persona, layer_name):
    def fn(module, input, output):
        x = input[0].detach()
        if layer_name not in activation_stats[persona]:
            activation_stats[persona][layer_name] = []
        activation_stats[persona][layer_name].append(x.abs().mean(dim=(0, 1)))
    return fn

for persona, dataset in [("introvert", introvert_data), ("extrovert", extrovert_data)]:
    activation_stats[persona] = {}
    hooks = []
    for name, module in model.named_modules():
        if isinstance(module, torch.nn.Linear) and "embed" not in name and "lm_head" not in name:
            hooks.append(module.register_forward_hook(hook_fn(persona, name)))
    for batch in dataset:
        inputs = tokenizer(batch, return_tensors="pt", padding=True, truncation=True, max_length=512)
        with torch.no_grad():
            model(**inputs)
    for h in hooks:
        h.remove()
    # Average activation magnitudes
    for layer_name in activation_stats[persona]:
        activation_stats[persona][layer_name] = torch.stack(
            activation_stats[persona][layer_name]
        ).mean(dim=0)

# Step 2: Compute importance scores and masks
masks = {}
sparsity = 0.5
for persona in ["introvert", "extrovert"]:
    masks[persona] = {}
    for name, module in model.named_modules():
        if isinstance(module, torch.nn.Linear) and name in activation_stats[persona]:
            W = module.weight.data.abs()
            A = activation_stats[persona][name]
            S = W * A.unsqueeze(0)  # importance scores
            k = int((1 - sparsity) * S.shape[1])
            topk_indices = S.topk(k, dim=1).indices
            mask = torch.zeros_like(S)
            mask.scatter_(1, topk_indices, 1.0)
            masks[persona][name] = mask

# Step 3: Analyze differential mask ratios
for name in masks["introvert"]:
    diff = (masks["introvert"][name] != masks["extrovert"][name]).float().mean()
    print(f"{name}: {diff.item():.4f} differential ratio")
```

**Example 2: Contrastive Pruning for Role-Playing Characters**

User: "Build me a tool that creates Sherlock Holmes vs. Watson persona masks using contrastive pruning."

Approach:
1. Curate calibration data: 200 dialogue excerpts for each character from the canon, formatted as prompt-completion pairs.
2. Collect activation statistics for both personas.
3. Apply contrastive pruning: normalize scores per persona, compute divergence, assign parameters to the character with higher activation.
4. Store disjoint masks.
5. Validate by generating responses to deduction-style prompts.

Output:
```python
# Contrastive pruning core logic
def contrastive_prune(scores_a, scores_b, sparsity=0.5):
    """Produce disjoint masks for opposing personas."""
    # Normalize column-wise
    s_a_norm = scores_a / (scores_a.sum(dim=0, keepdim=True) + 1e-8)
    s_b_norm = scores_b / (scores_b.sum(dim=0, keepdim=True) + 1e-8)

    # Divergence per parameter
    divergence = (s_a_norm - s_b_norm).abs()

    # Assign to persona with larger normalized score
    a_dominant = s_a_norm >= s_b_norm  # boolean mask

    # Within each persona's allocation, keep top-K by divergence
    k = int((1 - sparsity) * scores_a.shape[1])

    mask_a = torch.zeros_like(scores_a)
    mask_b = torch.zeros_like(scores_b)

    for i in range(scores_a.shape[0]):
        a_indices = a_dominant[i].nonzero(as_tuple=True)[0]
        b_indices = (~a_dominant[i]).nonzero(as_tuple=True)[0]

        if len(a_indices) > k:
            topk = divergence[i, a_indices].topk(k).indices
            a_indices = a_indices[topk]
        if len(b_indices) > k:
            topk = divergence[i, b_indices].topk(k).indices
            b_indices = b_indices[topk]

        mask_a[i, a_indices] = 1.0
        mask_b[i, b_indices] = 1.0

    return mask_a, mask_b
```

**Example 3: Prompt-Engineering Workflow Inspired by Persona Subnetworks**

User: "I don't have access to model weights. How can I use these findings to improve my persona prompts?"

Approach:
1. The paper shows I/E and F/T dimensions are encoded more strongly than N/S and J/P. Use this to prioritize which personality axes to specify in prompts.
2. Structure prompts to activate the same circuits the subnetworks isolate — be specific about behavioral manifestations, not abstract labels.
3. Layer persona cues: early tokens set the persona context (like early transformer layers encoding persona), middle tokens provide task framing, final tokens specify output format.

Output:
```markdown
## Prompt Design Guided by Subnetwork Findings

### Weak (targets weakly-encoded dimensions):
"You are an INTJ personality type. Answer the following question."

### Strong (targets strongly-encoded dimensions with behavioral specifics):
"You are deeply introverted and analytical. You prefer to think before
speaking, find social gatherings draining, and express yourself through
careful written communication rather than spontaneous speech. You make
decisions based on logical frameworks rather than emotional responses.

Given the following scenario, respond in character:"

### Why this works:
The paper shows I/E divergence (1.34% differential mask ratio) is nearly
2x stronger than N/S (0.75%). Prompts targeting I/E and F/T activate
more distinct neural pathways. Behavioral descriptions (not just labels)
better activate the same circuits that persona subnetworks isolate.
```

## Best Practices

- **Do:** Start with 128 calibration samples and increase only if validation scores are unstable. The paper shows diminishing returns beyond 500 samples.
- **Do:** Apply masks only to attention and MLP Linear layers. Keep embeddings and the language model head unpruned — they encode shared vocabulary knowledge, not persona-specific behavior.
- **Do:** Use contrastive pruning whenever working with binary-opposing personas. Standard masking produces 70-80% mask overlap; contrastive pruning reduces this to disjoint allocations.
- **Do:** Validate both persona alignment AND general capability. A mask that scores high on personality questionnaires but tanks MMLU is useless.
- **Avoid:** Using sparsity above 0.6 for persona masks. Beyond 60% sparsity, general task performance degrades noticeably (>2% on MMLU).
- **Avoid:** Treating all personality dimensions as equally extractable. The paper shows some axes (I/E, F/T) are encoded ~2x more strongly than others (N/S, J/P). Set expectations accordingly.

## Error Handling

- **Mask produces incoherent text:** Sparsity is too high. Reduce rho from 0.5 to 0.4, or switch to soft gating (gamma=0.05) instead of hard binary masking.
- **Persona alignment is weak despite correct pipeline:** Calibration data may be noisy. Audit examples for consistency — a dataset with mixed introvert/extrovert signals will produce a blurred mask. Filter calibration pairs to those with unambiguous persona signals.
- **Contrastive masks are heavily imbalanced (one persona gets 80%+ of parameters):** The calibration sets are imbalanced in activation intensity. Normalize activation statistics per-persona before computing divergence scores.
- **Out-of-memory during activation collection:** Process calibration data in small batches (8-16 samples) and maintain running mean/variance statistics rather than storing all raw activations.
- **Masked model repeats itself or degenerates:** Pruned subnetwork may have lost critical generation-quality circuits. Restore the top MLP layer (layer 0 and final layer tend to be critical) by setting their masks to all-ones.

## Limitations

- Requires access to model weights — this technique cannot be applied to API-only models (GPT-4, Claude). For API-only scenarios, use the prompt-engineering insights from Example 3 instead.
- Mask quality depends heavily on calibration data quality. Garbage in, garbage out — poorly curated persona examples produce masks that encode noise rather than personality.
- The method finds correlational subnetworks, not necessarily causal ones. Some masked parameters may be incidental to persona behavior rather than driving it.
- Storage overhead scales linearly with persona count. Each persona requires one binary mask per Linear layer (~3.5GB for a 7B model at 50% sparsity, though this compresses well).
- Evaluated primarily on MBTI and character roleplay. Generalization to nuanced behavioral axes (e.g., communication formality, humor style, domain expertise) is plausible but unvalidated.
- Subnetwork extraction is model-specific. Masks computed for Llama-3-8B do not transfer to Mistral-7B. Each model requires its own calibration pass.

## Reference

**Paper:** "Your Language Model Secretly Contains Personality Subnetworks" — Ye et al., ICLR 2026.
[https://arxiv.org/abs/2602.07164v1](https://arxiv.org/abs/2602.07164v1)
Look for: Section 3 (activation-guided masking algorithm), Section 4 (contrastive pruning formulation), Table 2 (differential mask ratios by MBTI dimension), Tables 4-5 (persona classification results vs. baselines).
**Code:** [https://github.com/Ruimeng-Ye/Persona.git](https://github.com/Ruimeng-Ye/Persona.git)