---
name: "regular-variational-latent-reasoning"
description: "Compress verbose chain-of-thought reasoning into compact latent state representations guided by rendered visual summaries, based on the ReGuLaR paper. Use when: 'compress my reasoning', 'latent reasoning over this problem', 'reduce CoT verbosity', 'efficient multi-step reasoning', 'render-guided reasoning compression', 'reason with fewer tokens'."
---

# ReGuLaR: Variational Latent Reasoning Guided by Rendered Chain-of-Thought

This skill implements the ReGuLaR agent reasoning pattern — compressing explicit chain-of-thought into compact latent state summaries, using rendered structured representations as compression guides. Instead of emitting long verbose reasoning chains, the agent produces a full internal reasoning trace, renders it into a structured visual summary (table, diagram, or compressed format), extracts the dense semantic core from that rendering, and then uses only that compressed state to drive subsequent reasoning steps and final answers. This yields dramatically fewer output tokens while preserving (or improving) reasoning quality, directly applying the insight from ReGuLaR that rendered representations regularize latent compression with minimal information loss.

## When to Use

- When a user asks you to solve a multi-step math, logic, or coding problem and you need to reason through many intermediate steps but want concise output
- When debugging a complex issue requiring long chains of deduction — compress intermediate findings into latent state summaries between phases
- When orchestrating multi-agent workflows where passing full reasoning chains between agents wastes context — compress to latent states instead
- When a user explicitly asks for "compressed reasoning", "latent reasoning", or "think step by step but keep it short"
- When solving problems that require 5+ reasoning steps and you want to avoid context window bloat in downstream processing
- When building pipelines that chain LLM calls and need to minimize token transfer between stages
- When the user asks you to implement a reasoning-compression layer in their AI application

## Key Technique

**The Redundancy Problem.** Standard chain-of-thought generates explicit text for every reasoning step. For a 10-step math problem, this can produce 500+ tokens of intermediate reasoning. Most of these tokens are scaffolding ("Now I need to...", "Let me calculate...", "Therefore...") rather than information-carrying content. ReGuLaR addresses this by formulating reasoning as a variational process: each reasoning step is compressed into a latent state z_k sampled from a posterior distribution conditioned on all previous states.

**Rendered Guidance.** The critical insight is that naive compression (just "think internally") degrades badly because the model loses track of what information to preserve. ReGuLaR solves this by rendering the explicit reasoning chain as a structured visual artifact — literally converting text to an image — and extracting dense visual-semantic features from it. These features serve as the prior distribution that regularizes compression: the latent state must capture everything that the visual rendering captures, but in far fewer dimensions. The KL divergence between the posterior (what the model wants to encode) and the prior (what the rendered summary captures) acts as the compression guide.

**Practical Application.** In an agent context, we adapt this as a two-pass pattern: (1) generate full reasoning internally, (2) render it into a structured summary format (markdown table, state diagram, or compressed notation), (3) extract the essential semantic content from that rendering, and (4) use only that compressed representation going forward. At inference time, the trained agent can skip the rendering step entirely — the compression becomes internalized. For agent orchestration, this means passing compact state objects between agents instead of full reasoning transcripts.

## Step-by-Step Workflow

1. **Receive the problem and identify reasoning depth.** Count the expected reasoning steps. If fewer than 3 steps, use standard direct reasoning. If 3+ steps, activate the latent reasoning pattern.

2. **Generate the full explicit reasoning chain internally.** Work through the problem step by step with complete detail. Do not suppress any reasoning — capture every intermediate result, assumption, and deduction. This is your "uncompressed" chain.

3. **Segment the reasoning into K phases.** Divide the chain into K logical segments (where K is typically 2-5), each covering one coherent sub-problem or deduction stage. Each segment should be self-contained enough to summarize independently.

4. **Render each segment into a structured summary.** For each segment, produce a compressed structured representation:
   - For math/logic: a state tuple `(known_facts, derived_result, carry_forward)`
   - For code debugging: a state tuple `(hypothesis, evidence, conclusion)`
   - For multi-step plans: a state tuple `(completed, current_state, next_action)`

   Format these as a compact table or structured block — this is the "rendering" step that acts as the compression guide.

5. **Extract the latent state from each rendering.** Reduce each structured summary to its minimal information-carrying core. Strip all scaffolding language. Keep only values, relationships, and conclusions. This is your latent state z_k.

6. **Chain latent states for final answer generation.** Compose the sequence of latent states [z_1, z_2, ..., z_K] into a single compact reasoning summary. Generate the final answer conditioned only on the original question and this compressed state chain — not the full verbose reasoning.

7. **Validate compression quality.** Check that the final answer matches what the full reasoning chain would produce. If the compressed path yields a different answer, the compression lost critical information — expand the failing segment and re-compress with finer granularity.

8. **Present the output.** Show the user the final answer with the compressed reasoning trail. Optionally include the rendered state summaries if the user wants to see the reasoning process.

## Concrete Examples

**Example 1: Multi-step math problem with reasoning compression**

User: "Solve this step by step but keep your reasoning compact: A store sells apples at $2 each. They offer 10% off for orders of 5+, and an additional 5% loyalty discount. If someone with loyalty status buys 8 apples, what do they pay?"

Approach:
1. Full internal reasoning: base cost = 8 * $2 = $16, volume discount = 10% off $16 = $1.60 off = $14.40, loyalty discount = 5% off $14.40 = $0.72 off = $13.68
2. Segment into K=2 phases: (pricing, discounts)
3. Render as latent states:

```
z_1: {items: 8, unit: $2, base: $16}
z_2: {discounts: [0.10, 0.05], method: sequential, result: $16 * 0.90 * 0.95}
```

Output:
```
Latent reasoning trail:
  z_1: 8 × $2 = $16
  z_2: $16 × 0.90 × 0.95 = $13.68

Answer: $13.68
```

**Example 2: Debugging a failing test with compressed diagnostic reasoning**

User: "My test_user_auth test is failing with a 403 on the /api/profile endpoint. Help me debug this."

Approach:
1. Generate full diagnostic chain internally (check auth middleware, token generation, route guards, CORS, session config...)
2. Segment into K=3 phases: (symptom analysis, hypothesis formation, root cause)
3. Render compressed states:

```
z_1: {symptom: 403_on_profile, context: auth_test, implies: token_valid_but_unauthorized}
z_2: {hypotheses: [role_mismatch, missing_scope, expired_token], evidence: 403≠401 → not_auth_failure → authorization_issue}
z_3: {root_cause: role_check, location: middleware/auth.js:47, fix: add_role_to_test_fixture}
```

Output:
```
Diagnostic summary:
  403 (not 401) means authentication succeeded but authorization failed.
  The test user fixture is likely missing the required role/scope.

Check: middleware/auth.js:47 — the role guard expects "user" role.
Fix: Add `role: "user"` to your test fixture's user object.
```

**Example 3: Compressing reasoning for multi-agent handoff**

User: "I'm building a pipeline where Agent A analyzes requirements and Agent B writes code. How do I compress A's reasoning before passing to B?"

Approach:
1. Identify the inter-agent communication bottleneck
2. Apply ReGuLaR pattern: Agent A produces full analysis, renders to structured spec, passes only the latent state

Output implementation:
```python
# Agent A: Full reasoning → compressed latent state
def compress_reasoning(full_analysis: str, k_segments: int = 3) -> dict:
    """
    ReGuLaR-inspired compression: segment reasoning into phases,
    render each to structured summary, extract latent states.
    """
    segments = segment_by_logical_phase(full_analysis, k=k_segments)
    latent_states = []
    for seg in segments:
        # "Render" step: convert verbose reasoning to structured form
        rendered = {
            "conclusion": extract_conclusion(seg),
            "key_facts": extract_facts(seg),
            "constraints": extract_constraints(seg),
        }
        # "Extract" step: reduce to minimal latent state
        latent = {
            k: v for k, v in rendered.items() if v  # drop empty fields
        }
        latent_states.append(latent)
    return {
        "original_query": query,
        "latent_chain": latent_states,
        "final_spec": synthesize_spec(latent_states),
    }

# Agent B receives only the compressed state, not the full reasoning
agent_b_input = compress_reasoning(agent_a_output)
```

**Example 4: Implementing a ReGuLaR-style training pipeline**

User: "I want to train a model that reasons in latent space using the ReGuLaR approach."

Approach:
1. Set up the data pipeline: CoT dataset → segmented reasoning → rendered images → visual features
2. Build the VAE module on top of a frozen LLM with LoRA
3. Train with the three-component loss

```python
# Core training loop structure (PyTorch pseudocode)
class ReGuLaRModel(nn.Module):
    def __init__(self, base_llm, visual_encoder, d_hidden=2048, d_visual=1280):
        super().__init__()
        self.llm = base_llm  # frozen, with LoRA adapters
        self.latent_head = nn.Sequential(
            nn.Linear(d_hidden, d_hidden),
            nn.GELU(),
            nn.Linear(d_hidden, d_hidden * 2),  # outputs (mu, log_sigma)
        )
        self.visual_adapter = nn.Sequential(
            nn.Linear(d_visual, d_hidden),
            nn.GELU(),
            nn.Linear(d_hidden, d_hidden),  # maps visual features to prior
        )
        self.visual_encoder = visual_encoder  # frozen (e.g., SigLIP/CLIP)

    def compute_loss(self, question, cot_segments, rendered_images, answer):
        latent_states = []
        kl_total = 0.0

        for k, (seg, img) in enumerate(zip(cot_segments, rendered_images)):
            # Posterior: conditioned on question + previous latent states
            h_k = self.llm.encode(question, latent_states)
            mu_k, log_sigma_k = self.latent_head(h_k).chunk(2, dim=-1)

            # Prior: from rendered visual features
            v_k = self.visual_encoder(img).mean(dim=1)  # mean pool
            z_hat_k = self.visual_adapter(v_k)

            # Reparameterization trick
            z_k = mu_k + torch.exp(log_sigma_k) * torch.randn_like(mu_k)
            latent_states.append(z_k)

            # KL divergence: posterior vs visual-guided prior
            kl_k = 0.5 * ((mu_k - z_hat_k)**2 + log_sigma_k.exp()**2
                          - 2 * log_sigma_k - 1).sum(dim=-1)
            kl_total += kl_k.mean()

        # Reconstruction: can model recover reasoning from latent states?
        recon_loss = self.llm.decode_loss(latent_states, cot_segments)

        # Answer generation: final answer from question + latent chain
        answer_loss = self.llm.decode_loss(latent_states, answer)

        return answer_loss + recon_loss + kl_total
```

Key hyperparameters from the paper:
- LoRA: r=128, alpha=32
- Optimizer: AdamW, lr=1e-4, weight_decay=0.01
- Visual encoder: SAM-Base + CLIP-Large (DeepSeek-OCR), 512x512 input
- Rendering: Verdana 9pt, A4 layout (595x842 points), 72 DPI
- Training: 8x A100 GPUs, GSM8K-Aug dataset (385k samples)

## Best Practices

- **Do:** Segment reasoning at natural logical boundaries (sub-problem transitions, hypothesis shifts), not at arbitrary token counts. The quality of segmentation directly determines compression quality.
- **Do:** Validate compressed output against full reasoning on a sample of problems before deploying in production. ReGuLaR achieves ~35% accuracy on GSM8K with 1B params — compression is lossy, so measure the loss.
- **Do:** Use the rendering step as an explicit checkpoint — if you cannot render a reasoning segment into a clean structured summary, the reasoning itself may be muddled.
- **Do:** Start with K=3-5 segments for most problems. The paper shows strong results even at K=1 (extreme compression), but more segments preserve more information.
- **Avoid:** Compressing reasoning for safety-critical decisions where every intermediate step must be auditable. Latent reasoning trades interpretability for efficiency.
- **Avoid:** Using this pattern for simple 1-2 step problems — the overhead of segmentation and rendering exceeds the savings from compression.
- **Avoid:** Passing raw unstructured text as the "rendered" representation. The key insight is that structured rendering (tables, tuples, diagrams) creates a better compression target than prose summaries.

## Error Handling

- **Compression loses critical information:** If the final answer from compressed reasoning differs from full reasoning, bisect the segments to find where information was lost. Split that segment into finer-grained sub-segments and re-compress.
- **Rendering fails to capture semantics:** If your structured summary misses a key relationship (e.g., a conditional dependency between variables), switch from flat tuple rendering to a dependency graph or decision tree format.
- **KL divergence explodes during training:** This indicates the posterior and prior have diverged severely. Reduce the KL weight (beta in beta-VAE), increase rendering resolution, or use a warmup schedule for the KL term.
- **Latent states collapse to identical values:** The model is ignoring the conditioning. Ensure each segment contains genuinely different information and that the visual encoder is frozen (not co-adapted with the LLM).

## Limitations

- Latent reasoning is inherently less interpretable than explicit CoT. Users who need to audit every reasoning step should use standard CoT instead.
- The technique is most effective for structured reasoning (math, logic, code) where intermediate states can be cleanly summarized. Open-ended creative reasoning compresses poorly.
- Training a full ReGuLaR model requires paired data: explicit CoT chains plus rendered visual representations. Generating this training data adds preprocessing overhead.
- The paper demonstrates results primarily on 1B-3B parameter models. Scaling behavior to larger models (70B+) is not yet established.
- Visual rendering quality matters: the paper found Verdana 9pt on A4 layout optimal. Suboptimal rendering configurations degrade the prior quality.
- As an agent pattern (without model fine-tuning), the compression is heuristic rather than learned — it approximates but does not replicate the trained VAE's compression quality.

## Reference

**Paper:** [ReGuLaR: Variational Latent Reasoning Guided by Rendered Chain-of-Thought](https://arxiv.org/abs/2601.23184v1) — Wang et al., 2026. Look for: Section 3 (VAE formulation and ELBO derivation), Section 3.2 (rendering pipeline and visual encoder architecture), Table 1 (main results showing 30%+ improvement over COCONUT/CoLaR), and Table 3 (molecular captioning showing latent reasoning outperforming 300+ token CoT).

**Code:** [https://github.com/FanmengWang/ReGuLaR](https://github.com/FanmengWang/ReGuLaR) — Reference implementation with training scripts, data processing pipeline, and evaluation code.