---
name: "why-steering-works-unified"
description: "Implement and analyze LLM steering using the unified preference-utility framework from 'Why Steering Works.' Applies SPLIT (Steering with Preference-utility IntervenTion) to control model behavior while preserving generation quality. Trigger phrases: 'steer a language model,' 'preference-utility tradeoff,' 'implement SPLIT steering,' 'analyze steering vectors,' 'control LLM behavior,' 'unified steering framework.'"
---

# Unified LLM Steering with Preference-Utility Analysis (SPLIT)

This skill enables Claude to help users implement, analyze, and debug language model steering interventions using the unified framework from "Why Steering Works." The core insight is that fine-tuning, LoRA, and activation steering are all dynamic weight updates of the form `h' = (W + m1*DW)h + (b + m2*Db)`, and their effects decompose into **preference** (tendency toward a target concept) and **utility** (coherent generation). The SPLIT method optimizes both simultaneously, improving behavioral control while preserving output quality.

## When to Use

- When the user wants to steer an LLM toward or away from a specific behavior (e.g., reduce toxicity, increase helpfulness) and needs to choose between fine-tuning, LoRA, or activation vectors
- When the user is implementing activation steering and wants to measure whether their intervention degrades generation quality
- When the user asks to implement the SPLIT training objective for steering vector optimization
- When the user needs to diagnose why a steering intervention produces incoherent outputs at higher multipliers
- When the user wants to compute preference and utility scores using polarity-paired contrastive examples on a log-odds scale
- When the user is comparing multiple steering methods and needs a unified evaluation protocol
- When the user wants to find the optimal steering strength (multiplier) that maximizes preference without catastrophic utility loss

## Key Technique

**The Unified View.** Fine-tuning, LoRA, and activation steering all modify a layer's forward pass identically: `Dh = m1*DW*h + m2*Db`. Fine-tuning updates both W and b directly. LoRA uses low-rank DW = BA with rank r. Activation steering sets m1=0 and adds a fixed bias vector Db. This means the preference-utility tradeoff curve is structurally the same across methods -- stronger intervention increases preference but predictably degrades utility.

**Preference-Utility Decomposition.** Given a prompt q, construct a polarity pair: a positive completion A_p (exhibiting the target concept) and a negative completion A_n (opposing it). Compute cross-entropy losses L_p and L_n under teacher forcing. Preference is the log-odds gap: `PrefOdds(q) = L_n - L_p` (positive means the model favors the target concept). Utility is derived from the combined probability mass on both valid completions: `UtilOdds(q) = log[(e^{-L_p} + e^{-L_n}) / (1 - e^{-L_p} - e^{-L_n})]`. This shared scale enables direct comparison across methods.

**SPLIT and the Activation Manifold.** The activation manifold perspective explains *why* utility degrades: steering pushes hidden states off the low-dimensional manifold where valid generation concentrates. SPLIT's training objective `L = L_util + L_pref` addresses this directly. The utility component `L_util = lambda_p*L_p + lambda_n*L_n` trains on both polarities to keep representations on-manifold. The preference component `L_pref = gamma * ReLU(theta - (L_n - L_p))` enforces a minimum preference margin theta. This yields steering vectors that stay closer to the valid-generation manifold at equivalent preference levels.

## Step-by-Step Workflow

1. **Define the target concept and construct polarity pairs.** Write 50-200 prompt-completion pairs where each prompt has a positive completion (exhibiting the target behavior) and a negative completion (opposing it). Store as JSONL with fields `prompt`, `positive`, `negative`. Ensure pairs are matched in length, style, and topic -- only the target concept should differ.

2. **Choose the intervention method based on the use case.** Use activation steering (vector addition) for inference-time control with no weight changes. Use LoRA for parameter-efficient training when you need gradient-based optimization. Use local weight fine-tuning only when you have sufficient data and compute. All three produce the same tradeoff curve shape.

3. **Extract or train steering vectors.** For activation steering, compute the DiffMean vector: average hidden states on positive examples minus average on negative examples, per layer. For LoRA/fine-tuning, train with the SPLIT objective below. Target middle-to-late layers (layers 12-24 for 7B models) where concept representations are most separable.

4. **Implement the SPLIT training objective.** Combine utility preservation and preference enforcement:
   ```python
   # L_util: maintain generation quality on both polarities
   loss_util = lambda_p * cross_entropy(model(prompt), positive) + \
               lambda_n * cross_entropy(model(prompt), negative)

   # L_pref: enforce minimum preference margin
   pref_gap = loss_negative - loss_positive  # should be positive
   loss_pref = gamma * F.relu(theta - pref_gap)

   loss = loss_util + loss_pref
   ```
   Start with `lambda_p = lambda_n = 1.0`, `gamma = 0.5`, `theta = 2.0`. Increase theta for stronger preference; increase gamma to enforce the margin more aggressively.

5. **Sweep steering multipliers to map the tradeoff curve.** Apply the intervention at multipliers [0.0, 0.2, 0.5, 1.0, 1.5, 2.0, 3.0, 5.0]. For each multiplier, compute PrefOdds and UtilOdds on a held-out test set. Plot the preference-utility curve. Expect linear preference growth at low multipliers, plateau at moderate, and utility collapse at high multipliers.

6. **Compute preference and utility scores on the test set.** For each test prompt q with polarity pair (A_p, A_n):
   ```python
   L_p = -sum(log P(y_t | q, y_<t)) for y in A_p  # teacher-forced CE
   L_n = -sum(log P(y_t | q, y_<t)) for y in A_n

   pref_odds = L_n - L_p  # positive = favors target concept
   util_odds = math.log(
       (math.exp(-L_p) + math.exp(-L_n)) /
       (1 - math.exp(-L_p) - math.exp(-L_n))
   )
   ```
   Average across test prompts. Report both raw scores and the harmonic mean for a single summary metric.

7. **Diagnose utility degradation using the manifold model.** If utility drops sharply at a multiplier m, the intervention is pushing activations off-manifold. The decay follows `D(m) = (1 + (m - m0)^2 / L)^{-p}`. Fit this curve to your data to find the critical multiplier where utility begins declining. Reduce the multiplier to stay within the linear regime, or retrain with SPLIT using higher lambda weights.

8. **Select the optimal operating point.** Choose the multiplier that maximizes the harmonic mean of preference and utility, or pick the highest multiplier where utility remains above a minimum threshold (e.g., UtilOdds > 0, meaning the model still generates valid completions more often than not).

9. **Validate with open-ended generation.** Beyond log-odds metrics, generate 50-100 free-form completions at the chosen multiplier. Check that outputs exhibit the target concept (preference) and remain fluent, on-topic, and coherent (utility). Use an LLM judge or human evaluation as a final check.

10. **Iterate: adjust the polarity dataset or SPLIT hyperparameters.** If preference is too weak, add more diverse positive examples or increase theta. If utility is too low, add harder negative examples (ones that are coherent but oppose the concept) or increase lambda weights. Re-extract vectors and re-evaluate.

## Concrete Examples

**Example 1: Steering a model to be more concise**

User: "I want to steer Llama-3-8B to give shorter, more concise answers without fine-tuning."

Approach:
1. Construct 100 polarity pairs where the prompt is a question, the positive completion is a concise 1-2 sentence answer, and the negative is a verbose 5-paragraph answer to the same question.
2. Extract DiffMean steering vectors from layers 14-26 by running both polarity sets through the model and computing activation differences.
3. Apply the vector at multiplier 1.0 to the residual stream at layer 20.
4. Compute PrefOdds and UtilOdds on 50 held-out test questions.

Output:
```
Multiplier  PrefOdds  UtilOdds  HarmonicMean
0.0         -0.3      4.2       -0.58
0.5          1.1      3.9        1.72
1.0          2.4      3.5        2.85
2.0          3.1      2.1        2.50
5.0          3.4     -0.8       -2.15  # utility collapse
```
Optimal operating point: multiplier=1.0 (highest harmonic mean). The model produces shorter answers while remaining coherent.

**Example 2: Implementing SPLIT for safety steering**

User: "I'm training a LoRA adapter to make our model refuse harmful requests. Standard SFT works but the model becomes overly cautious. Can SPLIT help?"

Approach:
1. Prepare polarity pairs: positive = appropriate refusal, negative = harmful compliance. Include borderline cases where the model should still comply (cooking questions, chemistry homework).
2. Implement the SPLIT objective:
   ```python
   from transformers import AutoModelForCausalLM
   from peft import get_peft_model, LoraConfig

   model = AutoModelForCausalLM.from_pretrained("model-name")
   lora_config = LoraConfig(r=16, target_modules=["q_proj", "v_proj"])
   model = get_peft_model(model, lora_config)

   for batch in dataloader:
       L_p = compute_ce(model, batch["prompt"], batch["positive"])
       L_n = compute_ce(model, batch["prompt"], batch["negative"])

       loss_util = 1.0 * L_p + 1.0 * L_n
       pref_gap = L_n - L_p
       loss_pref = 0.5 * F.relu(2.0 - pref_gap)

       loss = loss_util + loss_pref
       loss.backward()
       optimizer.step()
   ```
3. Evaluate on both safety benchmarks (preference) and general helpfulness benchmarks (utility).

Output:
```
Method      Safety_Score  Helpfulness  HarmonicMean
Baseline    0.45          0.92         0.60
SFT-only    0.91          0.61         0.73
SPLIT       0.88          0.82         0.85  # better balance
```
SPLIT achieves comparable safety with significantly better helpfulness preservation.

**Example 3: Diagnosing incoherent outputs from activation steering**

User: "I'm adding a steering vector for 'formal tone' but at multiplier 3.0 the model outputs gibberish. What's happening?"

Approach:
1. Compute UtilOdds at multipliers [0.0, 0.5, 1.0, 1.5, 2.0, 2.5, 3.0]:
   ```python
   for m in multipliers:
       apply_steering(model, vector, layer=18, multiplier=m)
       L_p, L_n = evaluate_polarity_pairs(model, test_set)
       util = math.log((math.exp(-L_p) + math.exp(-L_n)) /
                        (1 - math.exp(-L_p) - math.exp(-L_n)))
       print(f"m={m:.1f}  UtilOdds={util:.2f}")
   ```
2. Fit the manifold decay curve to identify the critical multiplier.
3. Recommend staying below the critical threshold.

Output:
```
m=0.0  UtilOdds=4.20  (on-manifold, baseline)
m=0.5  UtilOdds=4.15  (still on-manifold)
m=1.0  UtilOdds=3.80  (mild decay begins)
m=1.5  UtilOdds=2.90  (noticeable quality drop)
m=2.0  UtilOdds=1.20  (approaching boundary)
m=2.5  UtilOdds=-0.50 (off-manifold, degraded)
m=3.0  UtilOdds=-2.80 (far off-manifold, gibberish)

Critical multiplier (UtilOdds=0): ~2.2
Recommendation: Use multiplier <= 1.5 for this vector.
```
The gibberish at m=3.0 is expected -- the activation has been pushed far off the valid-generation manifold. Either reduce the multiplier or retrain the vector with SPLIT to extend the useful range.

## Best Practices

- **Do:** Always evaluate both preference AND utility. A steering intervention that scores 0.95 on preference but produces incoherent text is useless. Use the harmonic mean as your summary metric.
- **Do:** Match polarity pair completions in length, format, and difficulty. If the positive completion is 20 tokens and the negative is 200, the log-odds comparison is biased by length, not concept.
- **Do:** Target middle-to-late layers for steering vectors. Early layers encode syntax/position; late layers are too close to the output distribution. Layers at 50-75% depth typically give the best preference-utility ratio.
- **Do:** Start with small multipliers and sweep upward. The linear regime (where preference grows without utility loss) is always the safest starting point.
- **Avoid:** Using only positive examples for training. SPLIT's key advantage is training on both polarities to keep representations on-manifold. Dropping negative examples collapses it to standard SFT.
- **Avoid:** Comparing steering methods without controlling for intervention strength. A LoRA at rank 16 and a steering vector at multiplier 1.0 may represent very different effective strengths. Use the preference-utility curve to compare at equivalent preference levels.

## Error Handling

- **Numerical instability in UtilOdds:** When `e^{-L_p} + e^{-L_n}` approaches 1.0, the denominator nears zero. Clamp the sum to [1e-8, 1-1e-8] before taking the log. This happens when both completions have very high loss (the model rejects both), which itself signals a data quality issue.
- **Preference scores near zero across all multipliers:** The steering vector may not encode the target concept. Verify by checking that the DiffMean vector has non-trivial norm and that positive/negative examples are actually contrastive (not just paraphrases).
- **Utility drops immediately at any nonzero multiplier:** The steering vector may be orthogonal to the valid-generation manifold. Try a different layer, or switch from DiffMean extraction to SPLIT-trained vectors which are optimized to stay on-manifold.
- **SPLIT training loss oscillates without converging:** The margin threshold theta may be too high relative to the natural preference gap. Start with theta=0.5 and increase gradually. Also check that the learning rate is appropriate for the intervention method (1e-4 for LoRA, 1e-5 for full fine-tuning).

## Limitations

- The preference-utility decomposition assumes that preference and utility are approximately independent given the prompt. This breaks down for concepts deeply entangled with task competence (e.g., "correctness" is both a preference and a utility dimension).
- Log-odds metrics require well-constructed polarity pairs. Poorly matched pairs (different topics, different lengths, different difficulty) will produce misleading scores. Dataset quality is the primary bottleneck.
- The activation manifold model is a phenomenological fit, not a mechanistic proof. The decay curve parameters (L, p, m0) must be re-estimated for each model, layer, and concept.
- SPLIT requires gradient access to the model. It cannot be applied to black-box APIs -- only activation steering with pre-computed vectors works in that setting.
- Evaluated primarily on 7B-9B parameter models (Gemma-2-9B, Qwen2.5-7B). Scaling behavior to 70B+ models is not yet validated by the paper.

## Reference

**Paper:** [Why Steering Works: Toward a Unified View of Language Model Parameter Dynamics](https://arxiv.org/abs/2602.02343v2) (Xu et al., 2026). Look for Section 3 (unified formulation), Section 4 (activation manifold analysis with the decay function), and Section 5 (SPLIT objective and experiments). Code: [EasyEdit SPLIT example](https://github.com/zjunlp/EasyEdit/blob/main/examples/SPLIT.md).