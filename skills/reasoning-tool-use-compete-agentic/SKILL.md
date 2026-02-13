---
name: "reasoning-tool-use-compete-agentic"
description: "Diagnose and fix interference between reasoning and tool-use in agentic AI systems using LEAS attribution and DART-style decoupled adaptation. Use when: 'my agent is worse at reasoning after adding tools', 'tool calls are degrading chain-of-thought', 'diagnose reasoning vs tool-use interference', 'separate reasoning and action adapters', 'build a DART-style agentic pipeline', 'why does joint training hurt my agent'."
---

# Reasoning/Tool-Use Interference Diagnosis and Decoupled Agentic Design

This skill enables Claude to diagnose, quantify, and resolve the interference that occurs when an agentic system jointly trains or jointly executes reasoning (chain-of-thought, planning, reflection) and tool-use (API calls, search queries, code execution) through shared parameters. The core insight comes from the DART framework: reasoning and tool-use induce misaligned gradient directions during optimization, and decoupling them into separate adaptation pathways yields ~6-19% improvements in downstream task accuracy. Claude applies this principle at design time -- structuring agentic pipelines, prompt architectures, and fine-tuning configurations so that reasoning and action capabilities do not compete for the same representational capacity.

## When to Use

- When a user reports that adding tool-use capabilities to an LLM agent degraded its reasoning quality, or vice versa
- When designing a fine-tuning pipeline for an agent that must both reason (multi-step CoT, planning) and act (search, code execution, API calls)
- When building a single-model agent that needs to match multi-agent system performance where reasoning and tool-use are handled by separate models
- When the user asks to analyze whether reasoning and tool-use are interfering in their training runs (gradient conflict diagnosis)
- When architecting LoRA or adapter-based fine-tuning for agentic tasks and deciding how to partition adapter modules
- When a multi-hop QA or retrieval-augmented generation system underperforms despite strong individual capabilities

## Key Technique

**The Problem -- Gradient Interference:** When a single model is trained via reinforcement learning to both reason (generate chain-of-thought tokens) and use tools (generate search queries, API calls), the gradient updates for these two capabilities point in nearly orthogonal directions. Empirically, reasoning-token gradients and tool-use-token gradients show cosine similarity near zero (angles approaching 90 degrees), while same-capability gradients remain well-aligned. This means joint optimization forces the model into a compromise direction that is suboptimal for both capabilities -- a phenomenon the paper terms "training interference."

**LEAS -- Quantifying the Interference:** The Linear Effect Attribution System (LEAS) provides a principled way to measure this. It trains six model variants: a base model, a reasoning-only specialist (gradient-masked to only update on reasoning tokens), a tool-only specialist, a jointly-trained unified model, and two hybrid compositions. By fitting a linear model z = x^T * lambda over these variants' per-question correctness, LEAS extracts interaction coefficients. A negative lambda_23 (reasoning x tool-use interaction) confirms destructive interference; a positive value would indicate synergy. Across benchmarks, LEAS consistently finds negative interaction terms.

**DART -- The Fix:** Disentangled Action Reasoning Tuning freezes the base model and attaches two separate low-rank adaptation (LoRA) modules: one for reasoning tokens (theta_r = {B_r, A_r}) and one for tool-use tokens (theta_a = {B_a, A_a}). A token-level router classifies each generated token as reasoning or action based on special delimiter tokens (e.g., `<search>` opens tool-use, `</search>` closes it). During both forward pass and backpropagation, each token activates only its corresponding adapter. This zeroes out the cross-capability interaction term entirely, eliminating interference while keeping the efficiency of a single model. With rank r=8 per adapter (total parameters comparable to a single r=16 LoRA), DART achieves 6-19% gains over joint training and matches dedicated two-agent systems.

## Step-by-Step Workflow

1. **Classify token regions in your agent's output format.** Audit your agent's generation template and explicitly label which tokens are "reasoning" (thinking, planning, reflecting, summarizing evidence) and which are "action" (tool calls, search queries, API invocations, structured outputs). Use delimiter tokens or structured tags -- e.g., `<think>...</think>` for reasoning, `<tool_call>...</tool_call>` for actions.

2. **Measure baseline interference (LEAS-inspired diagnosis).** If you have training infrastructure, create three fine-tuning runs: (a) reasoning-only by masking gradients on action tokens, (b) tool-only by masking gradients on reasoning tokens, (c) joint training on all tokens. Compare per-task accuracy. If (c) underperforms the better of (a) or (b) on either capability, you have confirmed interference.

3. **Compute gradient alignment if possible.** During a training run, periodically extract flattened gradient vectors for reasoning-token batches and tool-token batches separately. Compute cosine similarity: `cos(g_reason, g_tool)`. Values near 0 or negative confirm orthogonal/opposing updates -- the signal that decoupling will help.

4. **Design decoupled adapter architecture.** Attach two separate LoRA modules to each transformer layer of your frozen base model. Use rank r=8 per adapter as a starting point. Wire a token-level router that reads your delimiter tokens to select which adapter processes each token during forward pass.

5. **Implement gradient isolation.** During backpropagation, apply a binary mask m_t per token: reasoning adapter parameters receive gradients only from reasoning tokens, action adapter parameters only from action tokens. This prevents cross-contamination: `grad(theta_r) = sum_t [grad_t * I(t in T_reason)]`.

6. **Configure RL training with GRPO or PPO.** Use group-relative policy optimization with rollout batch size 256, gradient batch size 64, learning rate 1e-6, KL coefficient 0.001. Limit tool-use budget to B=3-4 actions per episode to prevent degenerate tool-spamming.

7. **For prompt-only (no fine-tuning) scenarios, apply the decoupling principle architecturally.** Structure your agentic pipeline so that reasoning steps and tool-use steps are handled by separate system prompts, separate context windows, or separate API calls. Pass only the conclusion of reasoning to the tool-use stage, and only tool results back to the reasoning stage -- never raw mixed context.

8. **Evaluate on multi-hop benchmarks.** Test on tasks requiring both capabilities: HotpotQA, 2WikiMultiHopQA, Musique, or your domain equivalent. Measure exact match and F1. Compare against a joint/unified baseline to confirm the decoupling benefit.

9. **Tune adapter rank sensitivity.** Sweep r in {4, 8, 16, 32}. The paper found minimal variation, so prefer the smallest rank that maintains performance -- this minimizes parameter count and memory.

10. **Monitor for capability drift.** After deployment, periodically re-run LEAS-style diagnosis. As data distributions shift, previously benign interactions can become adversarial. Re-balance adapter capacity if one capability degrades.

## Concrete Examples

**Example 1: Diagnosing tool-use degradation in a search-augmented QA agent**

```
User: "I fine-tuned Qwen2.5-7B with GRPO to do search-augmented QA.
It learned to call the search API well, but its reasoning over search
results got worse compared to the base model. What's going on?"

Approach:
1. Identify the symptom: joint training improved tool-use (search calls)
   but degraded reasoning (synthesizing results). This is textbook
   reasoning/tool-use interference.
2. Confirm with LEAS-style diagnosis:
   - Train a reasoning-only variant: mask gradients on all tokens inside
     <search>...</search> tags. Evaluate on multi-hop QA.
   - Train a tool-only variant: mask gradients on all tokens outside
     <search>...</search> tags. Evaluate search recall.
   - Compare both against the joint model.
3. If reasoning-only variant outperforms joint model on reasoning metrics,
   interference is confirmed.
4. Prescribe DART: attach separate LoRA adapters for reasoning and search
   tokens, freeze the backbone, retrain with gradient isolation.

Output:
"Your agent exhibits reasoning/tool-use gradient interference. The joint
GRPO updates for search-query generation and answer-reasoning are pulling
model weights in orthogonal directions. Here's what to do:

1. Split your single LoRA (r=16) into two adapters (r=8 each):
   - reasoning_lora: receives gradients only from tokens outside <search> tags
   - tool_lora: receives gradients only from tokens inside <search> tags
2. Freeze the base model weights entirely.
3. Route tokens during forward pass: when generating inside <search>,
   activate tool_lora; otherwise activate reasoning_lora.
4. Keep all other GRPO hyperparameters the same.

Expected improvement: 13-19% exact match on NQ/HotpotQA based on
comparable setups in the literature."
```

**Example 2: Designing a prompt-only decoupled agent (no fine-tuning)**

```
User: "I'm building an agent with Claude that searches the web and
reasons over results. I can't fine-tune. How do I apply DART principles?"

Approach:
1. Apply the decoupling principle at the prompt/architecture level.
2. Separate the reasoning and tool-use into distinct phases with
   isolated context.
3. Prevent mixed-capability interference by not asking the model to
   simultaneously reason AND decide tool calls.

Output:
"Use a two-phase generation loop per step:

Phase 1 -- REASONING (isolated context):
  System prompt focuses purely on analysis and planning.
  Input: previous observations + current question
  Output: structured analysis and a decision: 'need more info' or 'ready to answer'
  Do NOT include tool-call syntax in this phase.

Phase 2 -- ACTION (isolated context):
  System prompt focuses purely on tool selection and query formulation.
  Input: the reasoning conclusion from Phase 1 (summary only, not full CoT)
  Output: a single tool call with parameters

Loop: Feed tool results back into Phase 1 as new observations.

This mirrors DART's architectural separation without fine-tuning. The key
is never asking the model to produce reasoning tokens and tool-call tokens
in the same generation -- that's where interference occurs even at
inference time due to attention pattern conflicts."
```

**Example 3: Configuring LoRA adapters for a DART-style training run**

```
User: "Give me the concrete configuration for DART-style training
on a Qwen2.5-3B model for search-augmented QA."

Output:
# DART Configuration

## Model Setup
base_model: Qwen/Qwen2.5-3B-Instruct
freeze_backbone: true

## Adapter Configuration
reasoning_adapter:
  type: lora
  rank: 8
  alpha: 16
  target_modules: [q_proj, k_proj, v_proj, o_proj]
  token_filter: "outside_tool_tags"  # receives gradients only for reasoning tokens

tool_adapter:
  type: lora
  rank: 8
  alpha: 16
  target_modules: [q_proj, k_proj, v_proj, o_proj]
  token_filter: "inside_tool_tags"   # receives gradients only for tool tokens

## Token Routing
tool_open_tags: ["<search>"]
tool_close_tags: ["</search>"]
# Tokens between open/close -> tool_adapter
# All other tokens -> reasoning_adapter

## GRPO Training
rollout_batch_size: 256
gradient_batch_size: 64
learning_rate: 1.0e-6
kl_coefficient: 0.001
max_tool_calls_per_episode: 4
top_k_passages: 3
reward: exact_match  # binary reward on final answer correctness

## Gradient Masking
# During backprop, binary mask m_t gates per-token gradient:
# grad(reasoning_adapter) += grad_t * I(t not in tool_tags)
# grad(tool_adapter) += grad_t * I(t in tool_tags)
# Cross-adapter gradients are strictly zero.
```

## Best Practices

- **Do:** Use explicit delimiter tokens (`<think>`, `<search>`, `<code>`) to create unambiguous boundaries between reasoning and action regions. The router's accuracy depends entirely on clean token classification.
- **Do:** Start with equal rank (r=8) for both adapters and only increase if one capability is clearly bottlenecked. The paper found rank insensitivity across r in {8, 16, 32}.
- **Do:** Limit tool-use budget (B=3-4 calls per episode) during RL training to prevent the agent from learning to spam tools instead of reasoning.
- **Do:** Apply this principle even without fine-tuning -- separate reasoning and action into distinct API calls or prompt phases to avoid interference in attention patterns.
- **Avoid:** Training a single LoRA on interleaved reasoning + tool-use tokens. This is the exact setup that causes gradient interference.
- **Avoid:** Using very high LoRA rank for one adapter and very low for the other. Asymmetric capacity creates a different kind of imbalance.
- **Avoid:** Letting tool-use results leak into the reasoning adapter's gradient path. Strict masking is essential -- partial isolation still permits interference.

## Error Handling

- **Router misclassification:** If delimiter tokens are missing or malformed, the router assigns tokens to the wrong adapter. Validate that all generated outputs contain properly nested open/close tags before training. Add a tag-repair preprocessing step.
- **Degenerate tool-use:** If the agent learns to call tools on every turn without reasoning, reduce the tool budget B or increase the KL penalty. This is a reward hacking failure mode, not an interference issue.
- **Adapter collapse:** If one adapter's weights converge to near-zero, its capability is being absorbed by the other. Check that gradient masking is correctly implemented -- both adapters should receive non-trivial gradient magnitudes.
- **OOM during dual-adapter training:** Two r=8 LoRA adapters use the same memory as one r=16. If still constrained, reduce rank to r=4 -- performance impact is minimal.

## Limitations

- The DART framework has been validated primarily on search-augmented QA tasks (NQ, TriviaQA, HotpotQA, etc.). Its effectiveness on agents with many diverse tools (code execution + web search + file I/O + database queries) is not yet established.
- Token-level routing requires clean delimiter-based classification. For agents where reasoning and tool-use are deeply interleaved at the sub-sentence level (e.g., inline code generation within a reasoning chain), the binary partition breaks down.
- LEAS assumes linear interaction effects. Highly nonlinear capability interactions (where reasoning quality depends on which specific tool was called) may not be captured by the linear attribution model.
- The approach targets training-time interference. At inference time with a frozen model, the decoupled adapters help, but prompt-level interference from mixed context can still occur.
- Results are demonstrated on 3B and 7B parameter models. Scaling behavior to much larger models (70B+) where capacity constraints are less severe is unknown -- interference may diminish naturally with scale.

## Reference

**Paper:** "Reasoning and Tool-use Compete in Agentic RL: From Quantifying Interference to Disentangled Tuning" -- Li et al., 2026. [arXiv:2602.00994v1](https://arxiv.org/abs/2602.00994v1)

**What to look for:** Section 3 for LEAS formulation and the six-model experimental design; Section 4 for DART architecture details including the token-level router and gradient masking equations; Table 1 for benchmark results showing 6-19% improvements; Appendix G for rank sensitivity analysis.