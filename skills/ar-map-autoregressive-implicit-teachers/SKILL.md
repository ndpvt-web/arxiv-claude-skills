---
name: ar-map-autoregressive-implicit-teachers
description: >
  Transfer preference alignment from autoregressive LLMs to diffusion language models via
  weight scaling. Implements the AR-MAP framework: train DPO on an AR model, extract the
  preference task vector, and merge it into a diffusion model with an amplified scaling
  coefficient to overcome spectral shadowing. Use when: "align a diffusion language model",
  "transfer DPO weights to a diffusion model", "scale LoRA adapters across architectures",
  "AR-MAP preference transfer", "merge alignment into DLLM", "fix high-variance DLLM training".
---

# AR-MAP: Transferring Preference Alignment from Autoregressive to Diffusion Language Models

This skill enables Claude to guide users through the AR-MAP transfer learning framework, which aligns Diffusion Large Language Models (DLLMs) by extracting preference knowledge from autoregressive LLMs and injecting it via scaled weight arithmetic. Instead of running expensive, high-variance DPO directly on diffusion models (where log-probabilities require intractable ELBO estimation), AR-MAP trains standard DPO on an autoregressive sibling model, computes the preference task vector (the weight delta), and merges it into the diffusion model with an amplified scaling factor gamma that compensates for spectral shadowing -- the phenomenon where the diffusion adaptation's large weight magnitude drowns out the smaller alignment signal.

## When to Use

- When a user wants to preference-align a diffusion language model (Dream, SDAR, or similar) but direct DPO on the DLLM produces unstable or high-variance results
- When transferring LoRA adapters trained on an autoregressive model (e.g., Qwen) to a structurally related diffusion model
- When the user asks how to choose the right scaling coefficient (gamma) for merging alignment weights into a diffusion model
- When implementing multi-aspect preference alignment (helpfulness, truthfulness, math reasoning) across model families that share architectural parameters
- When debugging why a diffusion model loses capabilities after naive weight merging (likely spectral shadowing -- gamma is too low or too high)
- When the user needs to set up a DPO-then-merge pipeline that avoids computing ELBO-based likelihoods during training

## Key Technique

**The Core Insight: Task Vector Arithmetic Across Paradigms.** AR-MAP exploits the fact that diffusion LLMs (like Dream-7B and SDAR-8B) are derived from autoregressive base models through continued pre-training, preserving identical architectural parameters (attention heads, hidden dimensions, layer count). This means the weight spaces are related by a structured offset. The method defines two vectors: (1) the *preference task vector* `tau_pref = W_AR_aligned - W_AR`, capturing what DPO learned about human preferences, and (2) the *diffusion structural vector* `tau_diffusion = W_DLLM - W_AR`, capturing the AR-to-diffusion adaptation. The aligned DLLM weights are then: `W_DLLM_aligned = W_AR + tau_diffusion + gamma * tau_pref`, which simplifies to `W_DLLM + gamma * tau_pref`.

**Why Gamma Must Be Greater Than 1 (Spectral Shadowing).** Proposition 3.1 from the paper proves that `||tau_pref||` is much smaller than `||tau_diffusion||` (by a factor epsilon << 1). Without amplification, the preference signal is trapped in an epsilon-neighborhood of the diffusion weights and has negligible effect on generation. The scaling factor gamma (typically 2-6) amplifies the alignment signal above the noise floor. However, gamma is task-sensitive: mathematical reasoning collapses beyond gamma > 2-3, while open-ended helpfulness benefits from gamma ~ 4.

**Automated Gamma Search.** AR-MAP uses a two-phase reward-based search: Phase 1 sweeps gamma in increments of 2 over a coarse range, Phase 2 fine-tunes around the best candidate at gamma-1. The objective maximizes pairwise reward accuracy: `gamma_hat = argmax_gamma (1/|B|) * sum(I[r_gamma(x, y_w) > r_gamma(x, y_l)])` evaluated on ~4096 preference samples.

## Step-by-Step Workflow

1. **Verify architectural compatibility.** Confirm that the target DLLM (e.g., Dream-7B, SDAR-8B) was derived from the same AR base model you plan to use as the teacher. Check that layer count, hidden dimension, attention head count, and vocabulary size match exactly. If they diverge, this method will not work.

2. **Prepare multi-aspect DPO training data.** Curate preference pairs for each alignment dimension: helpfulness (from HelpSteer2/UltraFeedback), truthfulness (from TruthfulQA-style data), and mathematical reasoning (from competition-math datasets). Format each as `{"prompt": x, "chosen": y_w, "rejected": y_l}`. Aim for ~10K samples per aspect.

3. **Train DPO on the autoregressive base model using LoRA.** Use LlamaFactory or an equivalent DPO trainer. Configure LoRA with rank 16, scaling factor 16, applied to query/key/value projection layers. Train for 3 epochs with learning rate 5e-7, batch size 128, beta=0.1. This produces the AR-aligned adapter weights.

4. **Extract the preference task vector.** Compute `tau_pref = W_AR_aligned - W_AR`. In practice with LoRA, this is simply the LoRA adapter weights themselves (since the base model weights cancel out). Save the adapter checkpoint.

5. **Run the gamma search to find the optimal scaling coefficient.** Prepare a validation set of ~4096 preference pairs. For each candidate gamma in [1, 2, 4, 6, 8], temporarily merge `W_DLLM + gamma * tau_pref`, compute reward accuracy on the validation set, and select the gamma with highest pairwise discrimination. Then refine with a fine-grained sweep at gamma_best +/- 1 in steps of 0.5.

6. **Merge the scaled adapter into the diffusion model.** Execute the merge: `theta_new = theta_DLLM + gamma_hat * tau_pref`. Use the model-specific merge script (e.g., `merge-lora-dream.py` for Dream, `merge-lora-sdar.py` for SDAR). Pass the `--weight` flag with the selected gamma value.

7. **Evaluate across alignment benchmarks.** Run the merged model on: AlpacaEval (helpfulness, GPT-4 judged), TruthfulQA (truthfulness), GSM8K/MATH (mathematical reasoning), Arena-Hard (general capability), and IFEval (instruction following). Compare against the unaligned DLLM baseline and any direct-DPO DLLM results.

8. **Diagnose and iterate.** If helpfulness scores are low, increase gamma (try 4-6). If math reasoning degrades, decrease gamma (try 2-3). If all metrics drop, verify that the architectural parameters truly match and that the LoRA adapter was trained on the correct AR base. Run the gamma search again with a larger validation set if results are noisy.

9. **Deploy the aligned DLLM.** The merged model can be served with standard inference frameworks. No special alignment-time infrastructure is needed at inference -- the preference knowledge is baked into the weights.

## Concrete Examples

**Example 1: Aligning Dream-7B for Helpfulness**

User: "I have Dream-7B (a diffusion language model) and Qwen2.5-7B (its AR base). I want to make Dream more helpful using DPO but direct training is unstable. How do I use AR-MAP?"

Approach:
1. Confirm Dream-7B derives from Qwen2.5-7B (both have 28 attention heads, 3584 hidden dim, 28 layers)
2. Train DPO on Qwen2.5-7B with HelpSteer2 preference data using LoRA (rank=16, alpha=16, 3 epochs, lr=5e-7)
3. Save the LoRA adapter -- this is tau_pref
4. Run gamma search on 4096 helpfulness preference pairs; expect optimal gamma ~ 4-6
5. Merge into Dream-7B: `python merge-lora-dream.py --base_model /models/dream-7b --lora_adapter /adapters/qwen-helpful --output /models/dream-7b-helpful --weight 6.0`
6. Evaluate on AlpacaEval

Output:
```
# Gamma search results (helpfulness):
gamma=1: 52.3% pairwise accuracy
gamma=2: 61.7% pairwise accuracy
gamma=4: 68.9% pairwise accuracy
gamma=6: 70.2% pairwise accuracy  <-- selected
gamma=8: 67.1% pairwise accuracy (overshooting)

# AlpacaEval results:
Dream-7B (unaligned):     34.2% win rate
Dream-7B + AR-MAP (γ=6):  58.7% win rate
Dream-7B + Direct DPO:    41.3% win rate (high variance across runs)
```

**Example 2: Multi-Aspect Alignment of SDAR-8B**

User: "I need SDAR-8B aligned for helpfulness, truthfulness, AND math. Do I train three separate adapters or one combined?"

Approach:
1. Train three separate LoRA adapters on Qwen2.5-7B, one per aspect (dpo_helpful.json, dpo_truthful.json, dpo_math.json)
2. Run gamma search independently for each aspect -- math will need a lower gamma (~2-3) than helpfulness (~4-6)
3. Merge adapters sequentially or average the task vectors before scaling: `tau_combined = alpha_h * tau_helpful + alpha_t * tau_truthful + alpha_m * tau_math`
4. Merge into SDAR-8B with the composite vector

Output:
```
# Per-aspect optimal gamma values:
Helpfulness: γ=4 (open-ended generation benefits from strong signal)
Truthfulness: γ=3 (moderate signal avoids hallucination regression)
Math reasoning: γ=2 (fragile capability, collapses at higher gamma)

# SDAR-8B multi-aspect results:
             Helpful  Truthful  GSM8K   Avg
Unaligned:    35.1     42.3     67.8   48.4
AR-MAP:       59.2     58.7     71.2   63.0
Direct DPO:   44.1     49.8     63.2   52.4  (unstable, ELBO variance)
```

**Example 3: Debugging a Failed Merge**

User: "I merged the DPO adapter into my diffusion model with gamma=1 and saw zero improvement. What went wrong?"

Approach:
1. Diagnose spectral shadowing: gamma=1 means the preference signal magnitude is ~epsilon times the diffusion structural vector -- it's being drowned out
2. Verify with a quick check: compute `||tau_pref||` and `||tau_diffusion||` norms. If the ratio is < 0.1, gamma must be >> 1
3. Re-run with gamma=4 as a starting point, then use the two-phase search

Output:
```python
import torch

tau_pref = torch.load("adapter_weights.pt")
tau_diff = torch.load("diffusion_delta.pt")  # W_DLLM - W_AR

norm_pref = sum(p.norm() for p in tau_pref.values())
norm_diff = sum(p.norm() for p in tau_diff.values())
ratio = norm_pref / norm_diff

print(f"||tau_pref|| / ||tau_diffusion|| = {ratio:.4f}")
# Typical output: 0.03 -- confirming spectral shadowing
# Recommendation: start gamma search at 1/ratio ≈ 33, but practically 2-8 works
# because LoRA constrains the adapter to a low-rank subspace
```

## Best Practices

- **Do:** Always run the two-phase gamma search rather than guessing. The optimal gamma varies significantly by task (math=2-3, helpfulness=4-6) and is not transferable between domains.
- **Do:** Verify architectural parameter parity (heads, hidden dim, layers, vocab) between the AR teacher and DLLM student before attempting any transfer. Mismatches cause silent corruption.
- **Do:** Use separate LoRA adapters per alignment aspect. Combining helpfulness and math into a single training run dilutes both signals and prevents per-aspect gamma tuning.
- **Do:** Evaluate on both alignment metrics AND general capability benchmarks (Arena-Hard, IFEval) to detect capability regression from over-scaling.
- **Avoid:** Setting gamma > 3 for mathematical reasoning tasks. The paper shows sharp performance collapse beyond this threshold -- math capabilities are structurally fragile.
- **Avoid:** Applying AR-MAP when the DLLM was NOT derived from the AR base model via continued pre-training. If the weight spaces are unrelated, task vector arithmetic is meaningless.

## Error Handling

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Zero improvement after merge | Gamma too low (spectral shadowing) | Increase gamma; run the automated search |
| Catastrophic capability loss | Gamma too high; alignment signal overshoots | Reduce gamma by 50%; check per-aspect sensitivity |
| Math scores collapse while helpfulness improves | Single gamma applied to all aspects | Use separate adapters with per-aspect gamma values |
| Merge script fails with dimension mismatch | DLLM architecture differs from AR base | Verify head count, hidden dim, layer count match exactly |
| High variance in gamma search results | Validation set too small or unrepresentative | Increase validation set to 4096+ samples; ensure balanced chosen/rejected pairs |
| LoRA adapter loads but produces gibberish | Adapter was trained on a different base model checkpoint | Retrain DPO on the exact AR base that the DLLM was derived from |

## Limitations

- **Requires shared lineage.** AR-MAP only works when the DLLM was created from the AR model through continued pre-training. It cannot align arbitrary diffusion models with arbitrary AR teachers.
- **No guarantee of Pareto improvement.** Amplifying one alignment dimension (e.g., helpfulness at gamma=6) can degrade others (e.g., math reasoning). Multi-aspect alignment requires careful per-dimension tuning.
- **Gamma is not universal.** The optimal scaling coefficient changes across model sizes, training data, and task domains. Results from Dream-7B do not transfer to SDAR-8B without re-running the search.
- **Dependent on AR-DPO quality.** The transferred alignment is only as good as the AR teacher's DPO training. Noisy or biased preference data propagates through the pipeline.
- **Does not address inference-time diffusion control.** AR-MAP aligns the model's weights but does not modify the diffusion sampling process itself. Inference-time guidance techniques remain complementary.

## Reference

**Paper:** [AR-MAP: Are Autoregressive Large Language Models Implicit Teachers for Diffusion Large Language Models?](https://arxiv.org/abs/2602.02178v2) (Lin et al., 2026)
**Code:** [github.com/AMAP-ML/AR-MAP](https://github.com/AMAP-ML/AR-MAP)
**Key sections:** Section 3.2 for the spectral shadowing proposition and weight scaling formula; Section 4.2 for the gamma search algorithm; Table 3 for per-task gamma sensitivity analysis.