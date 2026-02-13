---
name: "diffusion-lms-approximate-optimal"
description: "Search and retrieve research on Calibrated Adaptive Length (CAL) for diffusion language model infilling. Helps users understand and implement training-free length prediction for DLM code/text infilling tasks. Triggers: 'diffusion model infilling', 'optimal infilling length', 'CAL method for DLMs', 'adaptive length code infilling', 'denoising confidence calibration', 'mask length prediction'"
---

# Diffusion LM Infilling with Calibrated Adaptive Length (CAL)

This skill enables Claude to help users understand, implement, and debug the CAL (Calibrated Adaptive Length) method from Liu et al. (2026), a training-free technique that allows diffusion language models (DLMs) to automatically discover the correct number of tokens to generate for infilling tasks. The core insight is that a DLM's first-step denoising confidence contains a hidden signal (the "Oracle Peak") near the true completion length, which can be revealed by calibrating out a systematic length-dependent bias. This eliminates the need to manually specify infilling length -- the biggest practical limitation of DLM infilling.

## When to Use

- When a user asks how to determine the right number of mask tokens for diffusion model infilling
- When implementing code completion or fill-in-the-middle with DLMs like LLaDA, DiffuCoder, or DreamCoder
- When a user wants to improve Pass@1 rates on code infilling benchmarks (HumanEval-Infilling, etc.)
- When building text infilling pipelines (story completion, abstract gap-filling, review generation) with DLMs
- When a user asks about confidence calibration or length bias in masked language models
- When debugging poor infilling quality caused by mismatched mask lengths in diffusion models
- When a user wants a training-free method to make DLM infilling adaptive without fine-tuning

## Key Technique

**The Problem.** Diffusion language models generate text by iteratively denoising a fully masked sequence. For infilling (filling a gap between a prefix and suffix), the user must pre-specify how many mask tokens to allocate. Too few tokens and the model truncates the completion; too many and it pads with garbage. This length mismatch is the dominant failure mode for DLM infilling.

**The Discovery.** The authors found two phenomena in the *first-step denoising confidence* -- the average max-probability across masked positions after a single forward pass from a fully masked state. First, a local **Oracle Peak**: calibrated confidence reaches a maximum near the ground-truth length, signaling semantic completeness. Second, a systematic **Length Bias**: raw confidence monotonically decays as mask length increases (longer spans are harder to predict), following a double-exponential curve `B(L) = a*exp(-b*L) + c*exp(-d*L) + e`. This bias obscures the Oracle Peak in raw confidence.

**The Method (CAL).** Fit the bias function `B(L)` once on a small calibration set (~100 samples), excluding lengths near ground-truth to avoid contamination. At inference, compute calibrated confidence `Phi_c(L) = Phi(L) / B(L)` for candidate lengths using single forward passes. Use bidirectional hill-climbing from an initial length estimate, stepping by `DeltaL` and stopping after `D` consecutive non-improvements. The peak of `Phi_c` gives the predicted length, then run full denoising at that length. Cost: 11-18 forward passes overhead, yielding up to 47.7% Pass@1 improvement on code infilling.

## Step-by-Step Workflow

1. **Identify the DLM and task.** Determine which diffusion language model is being used (LLaDA, DiffuCoder, DreamCoder, DAEDAL) and whether the task is code infilling or text infilling. This dictates hyperparameter choices.

2. **Prepare the bias calibration dataset.** Collect ~100 representative samples with known ground-truth completions. For code, HumanEval-Infilling demo split works well. For text, use domain-matched examples (stories, abstracts, reviews).

3. **Measure raw confidence at multiple lengths.** For each calibration sample, construct masked sequences at lengths `[1, 2, 4, 6, 12, 16, 24, 32, 48, 64, 96, 128]`. Run a single forward pass per length and compute `Phi(L) = (1/L) * sum(max_v p_j(v))` over masked positions `j`.

4. **Exclude oracle-contaminated points.** For each sample, remove measurements within `+/- 4` tokens of the ground-truth length. This prevents the Oracle Peak from distorting the bias fit.

5. **Fit the double-exponential bias function.** Using scipy's `curve_fit` with initial params `[1.0, 1.8, 0.6, 0.05, 0.3]` and bounds `(0, [1.0, 2.0, 1.0, 0.5, 0.5])`, fit `B(L) = a*exp(-b*L) + c*exp(-d*L) + e` to the aggregated confidence-vs-length data. Weight by sample count per length.

6. **Set search hyperparameters.** For code infilling: `DeltaL=1, D=4, L_max=64` (single-line) or `128` (multi-line). For text infilling: `DeltaL=1, D=2`. Choose initial length `L_init` from `{4, 8, 16, 32}` for code or `{2, 4, 8}` for text.

7. **Run bidirectional hill-climbing at inference.** Starting from `L_init`, probe both increasing and decreasing directions. At each candidate length `L`, compute `Phi_c(L) = Phi(L) / B(L)`. Track the best calibrated confidence. Stop a direction after `D` consecutive non-improvements.

8. **Select the optimal length and decode.** The length `L_hat` with the highest `Phi_c` becomes the final mask length. Run the full iterative denoising process of the DLM at this length to produce the completion.

9. **Validate results.** For code, check syntactic correctness and test passage. For text, measure BLEU/ROUGE against references. Compare against the fixed-length baseline to confirm the adaptive length improved quality.

10. **Transfer the bias function across model variants.** The fitted `B(L)` generalizes across models of the same family (e.g., LLaDA-Base to LLaDA-Instruct) without re-fitting, so re-use it when switching between base and instruction-tuned variants.

## Concrete Examples

**Example 1: Code Infilling with Unknown Completion Length**

User: "I'm using DiffuCoder for fill-in-the-middle code completion but my completions are getting cut off or padded with nonsense. How do I pick the right mask length?"

Approach:
1. Explain that the core issue is fixed mask length -- DLMs require pre-specifying how many tokens to generate, and mismatch causes truncation or padding.
2. Recommend implementing CAL: fit a bias function on ~100 HumanEval samples, then use calibrated confidence to search for optimal length at inference.
3. Provide the bias fitting code structure:

```python
import numpy as np
from scipy.optimize import curve_fit

def bias_func(L, a, b, c, d, e):
    return a * np.exp(-b * L) + c * np.exp(-d * L) + e

# Collect raw confidence at test lengths (excluding oracle neighborhood)
test_lengths = [1, 2, 4, 6, 12, 16, 24, 32, 48, 64, 96, 128]
# avg_confidences = [...] (measured from calibration set)

popt, _ = curve_fit(
    bias_func, test_lengths, avg_confidences,
    p0=[1.0, 1.8, 0.6, 0.05, 0.3],
    bounds=(0, [1.0, 2.0, 1.0, 0.5, 0.5]),
    maxfev=10000
)
```

4. Provide the inference-time search:

```python
def cal_search(model, prefix, suffix, bias_params, L_init=16, delta=1, D=4, L_max=128):
    best_L, best_score = L_init, calibrated_conf(model, prefix, suffix, L_init, bias_params)

    for direction in [+1, -1]:
        L = L_init + direction * delta
        consecutive_misses = 0
        while 1 <= L <= L_max and consecutive_misses < D:
            score = calibrated_conf(model, prefix, suffix, L, bias_params)
            if score > best_score:
                best_L, best_score = L, score
                consecutive_misses = 0
            else:
                consecutive_misses += 1
            L += direction * delta
    return best_L

def calibrated_conf(model, prefix, suffix, L, bias_params):
    # Construct: [prefix_tokens] + [MASK]*L + [suffix_tokens]
    # Single forward pass, get logits at masked positions
    # Phi(L) = mean of max softmax prob at each masked position
    # Return Phi(L) / bias_func(L, *bias_params)
    ...
```

Output: The user gets a working CAL implementation that replaces their fixed-length mask with an adaptive search, typically improving Pass@1 by 20-47% on code infilling tasks.

---

**Example 2: Text Infilling for Story Completion**

User: "I want to use LLaDA to fill in missing sentences in short stories (ROCStories dataset). The fixed length of 10 tokens works for some but not others."

Approach:
1. Note that text infilling has a smoother confidence landscape than code, so use tighter tolerance `D=2`.
2. Fit bias function on a subset of ROCStories with known completions. Use the same double-exponential model.
3. At inference, run CAL with `L_init=4, DeltaL=1, D=2, L_max=64`.
4. Expect BLEU-2 improvement of up to 8.5% and ROUGE-L up to 9.9% over fixed-length.

Output: Per-story adaptive lengths -- short stories get 5-8 tokens, longer completions get 15-25 tokens, each matched to the semantic gap rather than a one-size-fits-all constant.

---

**Example 3: Debugging Why CAL Isn't Finding the Right Length**

User: "I implemented CAL but the predicted lengths are consistently too short. What's going wrong?"

Approach:
1. Check bias function fitting: plot raw confidence vs. length and the fitted curve. If the fit is poor (R^2 < 0.95), the calibration will be unreliable.
2. Verify oracle exclusion: ensure lengths within `[L* - 4, L* + 4]` are excluded from bias fitting. Including them contaminates the bias estimate.
3. Check if `L_init` is too low and the hill-climbing terminates before reaching the true peak. Try a larger `L_init` (e.g., 32 for code, 8 for text).
4. Inspect the calibrated confidence curve for a specific example: plot `Phi_c(L)` across all lengths. If it's monotonically decreasing, the bias function may be over-correcting -- re-fit with more calibration data.
5. Ensure the forward pass uses the correct masking scheme: the entire infilling region should be [MASK] tokens, with prefix and suffix as context.

Output: Diagnosis pinpoints whether the issue is in bias fitting, search initialization, or masking, with concrete fix for each case.

## Best Practices

- **Do:** Fit the bias function once per model family and reuse it across variants (base/instruct). The bias is a property of the architecture, not the fine-tuning.
- **Do:** Use `D=4` for code (sharp Oracle Peaks need room to confirm) and `D=2` for text (smoother landscape, early stopping saves compute).
- **Do:** Exclude the oracle neighborhood `[L* - 4, L* + 4]` when fitting bias. This is critical for clean calibration.
- **Do:** Profile the search cost. Expect 11-18 forward passes per inference. Each is a single denoising step, much cheaper than full iterative decoding.
- **Avoid:** Using CAL with autoregressive models. The technique is specific to diffusion/masked language models that denoise from a fully masked state. AR models don't have this confidence signal.
- **Avoid:** Setting `L_max` too high without reason. Unnecessary large search ranges waste forward passes. Use domain knowledge: single-line code rarely exceeds 64 tokens; multi-line rarely exceeds 128.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Bias fit fails to converge | Too few calibration samples or extreme outliers | Increase to 200+ samples; remove outliers beyond 3 sigma |
| Oracle Peak not visible after calibration | Bias function poorly fitted or task has high ambiguity | Re-fit with more length test points; for highly ambiguous text, fall back to fixed length at the mode of training distribution |
| Search always returns `L_init` | Hill-climbing stuck at local optimum or `D` too small | Try multiple `L_init` values and take the best; increase `D` to 6 |
| Predicted length is wildly wrong (off by 3x+) | Prefix/suffix tokenization mismatch or masking error | Verify that mask tokens are placed exactly in the infilling region and context tokens are unmasked |
| Calibrated confidence is flat (no peak) | All forward passes return near-uniform distributions | Check model loading -- this usually indicates the model isn't processing context correctly |

## Limitations

- **DLM-only.** CAL exploits the masked denoising framework of diffusion language models. It does not apply to autoregressive models (GPT, LLaMA) or encoder-only models (BERT).
- **Compute overhead.** The 11-18 extra forward passes per inference add latency. For real-time applications, this may be prohibitive. Batch the confidence probes where possible.
- **Text infilling gains are modest.** Due to the inherent ambiguity of natural language (multiple valid completions of different lengths), Oracle Peaks in text are diffuse. Code infilling benefits far more (47.7% vs 8.5%).
- **Requires calibration data.** While training-free, CAL still needs ~100 samples with known completions to fit the bias function. For novel domains with no reference data, bias fitting is unreliable.
- **Single contiguous span.** CAL addresses single-span infilling. Multi-span or scattered mask infilling (e.g., filling multiple blanks simultaneously) is not directly supported.
- **Token-level granularity.** The search operates at token level (`DeltaL=1`), which is appropriate for code but could be made coarser (word-level) for long text spans to reduce search cost.

## Reference

**Paper:** Liu, Yang, Su. "Diffusion LMs Can Approximate Optimal Infilling Lengths Implicitly." arXiv:2602.00476v1, Jan 2026.
**Code:** https://github.com/NiuHechang/Calibrated_Adaptive_Length
**Key insight to look for:** Section 3's analysis of first-step denoising confidence showing the Oracle Peak phenomenon, and Algorithm 1's bidirectional hill-climbing search with calibrated confidence.