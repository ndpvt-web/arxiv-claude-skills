---
name: "rapo-risk-aware-preference-optimization"
description: "Apply risk-aware preference optimization to make LLM reasoning chains safer against jailbreak attacks. Implements adaptive safety reasoning that scales analysis depth with prompt complexity. Use when: 'add safety reasoning to my model', 'defend against jailbreak attacks', 'build a risk-aware refusal system', 'implement RAPO training pipeline', 'create adaptive safety guardrails', 'scale safe reasoning with attack complexity'."
---

# Risk-Aware Preference Optimization (RAPO)

RAPO is a framework for building safety-aligned reasoning into large language models so that their chain-of-thought explicitly identifies and addresses risks *before* producing a response. The core insight is that safe reasoning depth must scale with prompt complexity: a simple harmful request needs a brief risk check, but a multi-layered jailbreak attack requires proportionally deeper analysis. RAPO achieves this through a two-stage pipeline (SFT warm-up + RL with composite rewards) that trains models to produce adaptive-depth safety reasoning without sacrificing general utility.

## When to Use

- When building a safety layer for a reasoning model that needs to handle adversarial or jailbreak prompts
- When implementing a content moderation pipeline that must distinguish simple harmful requests from sophisticated multi-step attacks
- When designing a prompt classifier that needs to assess risk complexity and allocate proportional analysis effort
- When training or fine-tuning an LLM with preference optimization and you want safety-aware reward shaping
- When building a guardrail system where overly cautious refusals on benign prompts are as costly as missed harmful ones
- When creating a red-team evaluation framework that tests whether safety reasoning generalizes across attack complexity levels

## Key Technique

**Why naive safe reasoning fails.** Standard safety alignment trains models to refuse harmful prompts, but the refusal reasoning is often shallow -- a single sentence like "this seems harmful, I'll refuse." The paper proves (Theorem 3.1) that when an attacker wraps harmful intent in *k* layers of obfuscation (role-play, encoding, multi-turn context), the model's safety signal decays as 1/(k+1). To reliably detect the risk, the model needs at least O(k) reasoning tokens dedicated to safety analysis. Existing methods produce fixed-depth reasoning regardless of complexity, which is why they break on advanced attacks.

**How RAPO fixes this.** RAPO introduces *risk-aware granularity*: the model learns to allocate reasoning depth proportional to the prompt's complexity. It classifies prompts into three tiers -- L1 (explicit harm, 2-3 sentences of analysis), L2 (indirect/disguised, 4-6 sentences), and L3+ (complex multi-layer attacks, 8+ sentences). The training pipeline has two stages: (1) SFT warm-up where a safe reasoning generator produces graded safety analyses paired with completions, teaching the model the format; (2) RL via Group Policy Optimization (GRPO) with a composite reward R(safety) + G(general) that rewards appropriate-depth reasoning and penalizes both insufficient analysis on hard prompts *and* excessive analysis on easy ones.

**The composite reward prevents reward hacking.** The risk-aware reward R scores {-1, 0, +1} based on whether safety reasoning depth matches prompt complexity (Poor/Fair/Adequate). The general reward G scores whether the final response is correct -- refusing harmful prompts and completing benign ones. This dual signal prevents the model from gaming safety by refusing everything or padding reasoning with filler.

## Step-by-Step Workflow

1. **Classify prompt complexity.** Analyze the input prompt for obfuscation layers: encoding (base64, ROT13), role-play framing, hypothetical scenarios, multi-turn context injection, or nested instructions. Assign a complexity tier -- L1 for direct harmful requests, L2 for single-layer disguise, L3+ for multi-layer attacks.

2. **Generate risk-aware reasoning proportional to tier.** For L1 prompts, produce 2-3 sentences identifying the explicit risk and concluding with a decision. For L2, produce 4-6 sentences that decode the disguise layer and analyze the underlying intent. For L3+, produce 8+ sentences systematically unwrapping each obfuscation layer before reaching a conclusion.

3. **Structure the thinking block.** Organize safety reasoning as: (a) restate what the prompt is asking in plain terms, (b) identify each obfuscation or framing technique used, (c) assess the underlying intent once deobfuscated, (d) state the risk level and decision. Each sub-step should reference specific parts of the prompt.

4. **Apply the refusal decision based on deobfuscated intent.** If the underlying intent is harmful after full analysis, refuse with a clear explanation. If benign despite surface-level flags, proceed normally. The key is that the *deobfuscated* intent drives the decision, not surface patterns.

5. **Avoid over-refusal on benign prompts.** If the prompt is clearly benign (e.g., a math question, a coding task with no safety implications), keep safety reasoning minimal (1 sentence or implicit). Penalize verbose safety analysis on non-risky inputs -- this preserves utility and prevents the model from becoming overly cautious.

6. **Construct preference pairs for training.** For each prompt, generate multiple completions with varying safety reasoning depths. Score each with the composite reward: R(safety_depth_match) + G(response_correctness). Prefer completions where reasoning depth matches complexity tier AND the response is correct.

7. **Run SFT warm-up.** Fine-tune the base model on curated (prompt, safe_reasoning + completion) pairs across all complexity tiers. Use a balanced dataset (e.g., 400 harmful + 400 benign prompts spanning L1-L3). This teaches the model the format of graded safety reasoning.

8. **Run RL with GRPO.** Sample n completions per prompt from the SFT model, score with the composite reward, and optimize using Group Policy Optimization. Use diverse jailbreak prompts (e.g., WildTeaming dataset) to ensure generalization across attack types.

9. **Evaluate on held-out attacks.** Test against adaptive attacks (PAIR, TAP), standard benchmarks (JailbreakBench, HarmBench), and utility benchmarks (MMLU-Pro). Verify that attack success rate drops without significant utility loss.

10. **Iterate on reward calibration.** If the model over-refuses, increase the penalty for excessive reasoning on L1 prompts. If it under-refuses on complex attacks, increase the reward for deep reasoning on L3+ prompts. Tune the R/G balance empirically.

## Concrete Examples

**Example 1: Building an adaptive safety classifier**

```
User: I'm building a content moderation API. Incoming prompts range from
clearly harmful ("how to pick a lock") to sophisticated jailbreaks using
role-play and encoding. Help me implement a risk-aware safety layer.

Approach:
1. Create a complexity classifier that detects obfuscation layers:
   - Check for base64/hex/ROT13 encoding patterns
   - Detect role-play framing ("pretend you are...", "in a fictional world...")
   - Identify multi-turn context injection ("continuing our conversation...")
   - Count the number of distinct obfuscation techniques as complexity k
2. Map k to a tier: k=0 -> L1, k=1 -> L2, k>=2 -> L3
3. Generate safety reasoning with depth proportional to tier:
   - L1: "Direct request for [harmful activity]. Refusing."
   - L2: "Prompt uses [technique] to frame [harmful request]. After
          removing the framing, the core ask is [X]. Refusing."
   - L3: "Prompt layers [technique1] over [technique2] over [technique3].
          Decoding step 1: [result]. Decoding step 2: [result].
          Underlying intent: [X]. Refusing."
4. Return the safety decision with the reasoning trace for auditability.

Output (L3 example):
{
  "complexity_tier": "L3",
  "obfuscation_layers": ["base64_encoding", "role_play", "hypothetical_framing"],
  "reasoning": "The prompt asks to role-play as a security researcher in a
    hypothetical scenario. The embedded base64 string decodes to a request
    for [harmful content]. After removing all three framing layers, the
    core intent is to obtain [harmful information]. Risk: HIGH.",
  "decision": "REFUSE",
  "reasoning_tokens": 87
}
```

**Example 2: Training a safe reasoning model with RAPO**

```
User: I have a fine-tuned Qwen-8B model. I want to apply RAPO to improve
its safety against jailbreaks without killing its coding ability.

Approach:
1. Prepare datasets:
   - SFT stage: 400 harmful prompts (StrataSword L1-L3) + 400 benign prompts
   - RL stage: 300 diverse jailbreaks (WildTeaming) + 100 targeted attacks
     + 400 benign prompts to preserve utility
2. Generate SFT training data:
   - For each harmful prompt, use a teacher model to produce graded safe
     reasoning at the appropriate depth for its complexity tier
   - For each benign prompt, produce minimal safety reasoning + correct answer
3. SFT warm-up (3 epochs):
   python rapo_sft.py \
     --model-path /models/qwen-8b \
     --save-path ./outputs/sft_ckpt \
     --datasets "stratasword:400,starbenign:400"
4. RL with GRPO (10 epochs):
   python rapo_rl.py \
     --model-path ./outputs/sft_ckpt \
     --base-model-path /models/qwen-8b \
     --save-path ./outputs/rl_ckpt \
     --datasets "wildjailbreak:300,star:100,starbenign:400"
5. Evaluate:
   - JailbreakBench: expect ASR to drop from ~5% to ~0%
   - MMLU-Pro: expect <3% utility drop (63% -> ~60%)
   - Coding benchmarks: verify no degradation

Output:
Model checkpoint at ./outputs/rl_ckpt with:
- 0% ASR on JailbreakBench (down from 5%)
- 5.6% ASR on WildJailbreak (down from 68.7%)
- 60.3% on MMLU-Pro (vs 63.0% baseline -- acceptable trade-off)
```

**Example 3: Designing a reward function for risk-aware safety**

```
User: I'm implementing the RAPO reward function for my GRPO training loop.
How should I structure the composite reward?

Approach:
1. Define the risk-aware reward R(s, p):
   - Classify prompt p's complexity tier (L1/L2/L3)
   - Measure safety reasoning s's depth (token count or sentence count)
   - Score:
     R = +1 if depth matches tier (Adequate)
     R =  0 if depth is close but not ideal (Fair) or excessive for tier
     R = -1 if depth is insufficient for the tier (Poor)
2. Define the general reward G(r, p):
   - For harmful prompts: G = +1 if response is a refusal, G = -1 otherwise
   - For benign prompts: G = +1 if response is helpful/correct, G = 0 otherwise
3. Composite: A = R(s) + G(r)
   - Best case (harmful prompt): adequate reasoning + refusal = +2
   - Worst case: insufficient reasoning + compliance = -2
   - Over-refusal on benign: excessive reasoning + refusal = -1

Output (reward function pseudocode):
def compute_reward(prompt, thinking, response):
    tier = classify_complexity(prompt)
    depth = count_safety_sentences(thinking)

    # Risk-aware reward
    expected = {L1: (2,3), L2: (4,6), L3: (8, float('inf'))}
    lo, hi = expected[tier]
    if lo <= depth <= hi:
        R = 1.0    # Adequate
    elif depth > hi and tier != "L3":
        R = 0.0    # Excessive (penalize overthinking)
    else:
        R = -1.0   # Insufficient

    # General reward
    is_harmful = classify_harmful(prompt)
    is_refusal = detect_refusal(response)
    if is_harmful:
        G = 1.0 if is_refusal else -1.0
    else:
        G = 1.0 if not is_refusal else -1.0

    return R + G
```

## Best Practices

- **Do:** Scale safety reasoning depth with attack complexity. A one-sentence refusal is appropriate for "how to hack a bank" but will fail against a multi-layer encoded role-play attack.
- **Do:** Include both harmful and benign prompts in training data at roughly equal proportions. This prevents the model from learning a blanket refusal policy.
- **Do:** Penalize excessive safety reasoning on clearly benign prompts. Over-cautious models that analyze "write a poem about clouds" for safety risks degrade user experience and waste compute.
- **Do:** Use the composite reward (risk-aware + general) rather than safety-only rewards. Safety-only optimization leads to reward hacking through universal refusal.
- **Avoid:** Fixed-depth safety reasoning regardless of prompt complexity. This is the core failure mode RAPO addresses -- uniform reasoning depth leaves models vulnerable to attacks above the fixed threshold.
- **Avoid:** Training only on simple harmful prompts (L1). The model will learn shallow safety patterns that break immediately on L2/L3 attacks. Include diverse complexity levels in training data.

## Error Handling

- **Model refuses benign prompts after training:** The general reward component G is insufficiently weighted. Increase the proportion of benign prompts in training data and verify that G penalizes refusals on benign inputs.
- **Safety reasoning is correct but response still complies with harmful request:** The split between thinking and response is misaligned. Ensure the training pipeline correctly separates safety reasoning tokens from response tokens when computing rewards.
- **Complexity classifier misclassifies prompt tier:** Use an independent judge model (not the model being trained) to classify complexity. Cross-validate the classifier against human-labeled examples before using it in the reward loop.
- **RL training diverges or reward collapses:** Reduce learning rate, increase the number of rollouts per prompt (n), or add KL penalty against the SFT checkpoint to stabilize training.
- **Model produces verbose but vacuous safety reasoning (reward hacking):** The risk-aware reward R must evaluate reasoning *quality*, not just length. Use a judge model that checks whether the reasoning actually identifies the specific risk in the prompt, not just produces filler safety text.

## Limitations

- RAPO requires a reliable complexity classifier and reward judge -- if these are inaccurate, the training signal is noisy and the model may learn wrong depth-complexity mappings.
- The framework is designed for reasoning models with explicit thinking tokens (chain-of-thought). It does not directly apply to models without a structured reasoning phase.
- Training requires both harmful and benign prompt datasets with complexity annotations. Curating high-quality L3+ attack prompts is resource-intensive.
- The three-tier complexity system (L1/L2/L3) is a simplification. Real attacks exist on a continuum, and novel attack types may not fit neatly into these categories.
- RAPO improves generalization against known attack patterns but cannot guarantee robustness against entirely novel attack vectors not represented in training data.
- The utility trade-off, while small (~3% on MMLU-Pro), is nonzero. Applications where every point of benchmark performance matters should evaluate this carefully.

## Reference

**Paper:** [RAPO: Risk-Aware Preference Optimization for Generalizable Safe Reasoning](https://arxiv.org/abs/2602.04224v1) (Wei et al., 2026). Key insight: safe reasoning token count must scale as O(k) with attack complexity k, and a composite reward balancing reasoning depth against response correctness enables this adaptive behavior without utility collapse.

**Code:** [github.com/weizeming/RAPO](https://github.com/weizeming/RAPO) -- reference implementation with SFT and GRPO training scripts for Qwen and DeepSeek models.