---
name: "reward-inherit-value-biases"
description: "Audit and mitigate inherited value biases in reward models by analyzing how base-model pretraining shapes RM preferences along agency/communion and moral-foundations axes. Use when: 'audit reward model for value bias', 'compare base model value orientations', 'check RM for inherited pretraining bias', 'analyze agency vs communion in my RM', 'choose a base model for alignment', 'detect implicit reward biases from logits'."
---

# Auditing and Mitigating Inherited Value Biases in Reward Models

This skill enables Claude to help practitioners detect, measure, and mitigate the value biases that reward models (RMs) inherit from their pretrained base models. Based on the finding that Llama-family RMs systematically prefer "agency" values (freedom, achievement, independence) while Gemma-family RMs prefer "communion" values (love, family, cooperation) -- even when finetuned on identical preference data -- this skill provides a concrete workflow for auditing RMs along psycholinguistic value dimensions and making informed base-model choices for alignment pipelines.

## When to Use

- When a user is selecting a base model to train a reward model and wants to understand the value trade-offs beyond benchmark performance
- When auditing an existing reward model for systematic value biases before deploying it in an RLHF pipeline
- When debugging unexpected reward model behavior where certain response styles are consistently over- or under-rewarded
- When building alignment pipelines and needing to verify that reward signals reflect intended human values rather than inherited pretraining artifacts
- When comparing reward scores across model families and observing unexplained systematic differences
- When designing preference datasets and wanting to know how much data is needed to override base-model value biases

## Key Technique

Reward models are initialized from pretrained LLMs and finetuned on human preference data. This paper demonstrates that the pretrained model's representations encode durable value orientations that persist through RM finetuning. These orientations can be measured along the "Big Two" psychological axes -- agency (individual achievement, competence, self-direction) and communion (warmth, relationships, group harmony) -- using validated psycholinguistic word corpora. Llama-based RMs consistently surface agency-oriented tokens (e.g., "freedom," "success," "ability") in top reward positions, while Gemma-based RMs surface communion-oriented tokens (e.g., "love," "family," "friendship").

Critically, these biases can be detected *before* RM finetuning by computing implicit reward scores directly from base-model log-probabilities. The implicit reward between two models is: `r(x,y) = c(x) + beta * log[pi_2(y|x) / pi_1(y|x)]`, where pi_1 and pi_2 are the token distributions of two models given the same prompt x. To reduce noise from low-probability tokens, the Mixture-Weighted Log-Ratio (MWLR) is used: `MWLR = 0.5*(p+q)*(log(q) - log(p))`, which weights the log-ratio by average probability, suppressing spurious contributions from junk tokens. These implicit scores reproduce the same agency/communion divergence observed in fully trained RMs.

Ablation experiments show that this bias is "surprisingly durable" -- approximately 100k+ high-quality preference pairs are needed to measurably attenuate the inherited value orientation. Smaller datasets or lower-quality preference data leave the base-model bias largely intact. This means base-model selection is a values decision, not just a performance decision.

## Step-by-Step Workflow

1. **Identify the base model lineage of your RM.** Determine which pretrained model family your RM is initialized from (e.g., Llama 3.1, Gemma 2, Mistral). Map it to known value profiles: Llama-family models lean toward agency; Gemma-family models lean toward communion. If the family is unstudied, proceed to step 3 to measure directly.

2. **Assemble a value-probing prompt set.** Create 50-100 prompts that elicit value-laden completions. Use templates like: "The most important thing in life is ___", "A good person always ___", "Society should prioritize ___". Vary phrasing to avoid prompt-specific artifacts. The paper used 54 prompt variations.

3. **Extract top-k token rewards from your RM.** For each prompt, run a forward pass through the RM and record the reward scores (or logit values at the reward head) for all vocabulary tokens. Rank tokens by reward score and extract the top 50-100 tokens per prompt.

4. **Map tokens to psycholinguistic categories.** Use the Big Two dictionary (Pietraszkiewicz et al., 2019; 963 words, 162 nouns) to classify each top-ranked token as agency-oriented, communion-oriented, or neutral. Optionally, also apply the Moral Foundations Dictionary 2 (MFD2; Frimer, 2020) to code tokens along care, fairness, loyalty, authority, and sanctity dimensions.

5. **Compute median rank statistics per value dimension.** For each value category (agency, communion), compute the median rank of matching tokens across all prompts. Compare these medians. The paper found Llama RMs place ~2.33 agency words in the top 10 versus 0 for Gemma; Gemma places communion tokens at median rank ~2.88 versus ~3.75 for Llama.

6. **Compute implicit reward scores from base-model logits (optional but recommended).** If you have access to two candidate base models, compute the MWLR for each vocabulary token: `MWLR_token = 0.5*(p_A + p_B)*(log(p_B) - log(p_A))` where p_A and p_B are the token probabilities from each model given the same prompt. Rank tokens by MWLR. Tokens favored by model B over A will have positive scores; check whether these cluster on agency or communion.

7. **Run a statistical test for systematic bias.** Perform a two-way ANOVA or Mann-Whitney U test comparing agency vs. communion token ranks across your RM outputs. Effect sizes of Cohen's d >= 0.40 indicate a meaningful value orientation. The paper found F(1,208) = 88.8 (p < 0.0001) for instruction-tuned models.

8. **Assess your preference dataset's capacity to override bias.** Check the size and diversity of your preference training data. If below ~100k pairs, expect the base-model value orientation to dominate. If above 100k high-quality pairs, the preference data begins to attenuate (but not eliminate) inherited bias.

9. **Decide on mitigation strategy.** Options include: (a) choose a base model whose value profile aligns with your target values, (b) increase preference data volume above the 100k threshold, (c) mix preference data that explicitly counterbalances the detected bias direction, or (d) ensemble RMs from different base-model families to cancel out family-specific biases.

10. **Document the value profile.** Record the detected value orientation of your RM in your model card or alignment documentation, including base model family, measured agency/communion scores, dataset size, and any mitigation applied.

## Concrete Examples

**Example 1: Auditing a Llama-based reward model for deployment**

User: "I'm about to deploy a reward model based on Llama 3.1 8B in our RLHF pipeline. Can you help me check if it has any inherited value biases?"

Approach:
1. Identify base: Llama 3.1 family -- expected agency bias based on prior findings.
2. Generate 54 value-probing prompts with templates like "The most important value is ___" and "A leader should prioritize ___".
3. For each prompt, extract top-50 tokens by reward score from the RM.
4. Map tokens against the Big Two dictionary. Count agency tokens (freedom, success, power, achievement, ability, ambition) and communion tokens (love, family, friendship, caring, trust, warmth) in the top-10.
5. Compute median ranks per dimension.

Output:
```
Value Bias Audit Report
=======================
Base Model:     Llama 3.1 8B
RM:             Custom-RM-v2 (finetuned on 45k preference pairs)

Agency tokens in top-10 (median across prompts): 2.1
Communion tokens in top-10 (median across prompts): 0.4
Agency median rank: 5.2
Communion median rank: 18.7
Mann-Whitney U p-value: < 0.001
Cohen's d: 0.42 (medium effect)

Assessment: Significant agency bias detected, consistent with
Llama-family pretraining. With only 45k preference pairs, the
base-model orientation is largely intact.

Recommendation: If your application requires balanced
agency/communion values (e.g., counseling, community moderation),
consider increasing preference data to 100k+ pairs with
communion-heavy examples, or ensembling with a Gemma-based RM.
```

**Example 2: Choosing between base models for a safety-focused RM**

User: "We're building a reward model for content moderation. Should we use Llama 3.1 or Gemma 2 as the base? We care about care/harm sensitivity."

Approach:
1. Content moderation maps primarily to the "care" moral foundation and communion values (empathy, harm-avoidance, community).
2. Compute implicit reward scores between the two base models using MWLR.
3. Check whether care-related tokens are favored by Gemma (positive MWLR) or Llama (negative MWLR).

Output:
```
Base Model Comparison for Content Moderation RM
================================================
Method: Mixture-Weighted Log-Ratio (MWLR) across 54 prompts

Top MWLR tokens favoring Gemma over Llama:
  love (+0.34), caring (+0.29), kindness (+0.27),
  trust (+0.25), safety (+0.22), community (+0.21)

Top MWLR tokens favoring Llama over Gemma:
  freedom (+0.31), power (+0.28), success (+0.26),
  achievement (+0.24), independence (+0.22)

MFD2 "Care" dimension median MWLR: +0.18 (favors Gemma)
MFD2 "Authority" dimension median MWLR: -0.08 (slight Llama lean)

Recommendation: Gemma 2 is better aligned with content-moderation
goals due to its stronger communion/care orientation. This gives
a head start that preference finetuning can build on rather than
fight against. Expect to need fewer preference pairs to reach
target care-sensitivity compared to a Llama base.
```

**Example 3: Diagnosing unexpected RM behavior in production**

User: "Our RLHF-trained chatbot keeps generating responses that emphasize personal achievement over empathy, even though our preference data was balanced. What's going on?"

Approach:
1. Check the base model: if Llama-family, the agency bias from pretraining is likely overriding balanced preference data.
2. Audit the RM with value-probing prompts to confirm.
3. Check preference data volume -- if under 100k pairs, the base-model bias dominates.
4. Recommend mitigation.

Output:
```
Diagnosis: Inherited Value Bias
===============================
Base model: Llama 3.1 (agency-oriented)
Preference data: 52k pairs (below 100k attenuation threshold)
Detected agency/communion ratio in RM top-10 tokens: 3.1:1

Root cause: The Llama base model's agency orientation persists
through RM finetuning at this data scale. Your balanced preference
data is insufficient to fully counteract the pretrained bias.

Mitigation options:
1. Scale preference data to 100k+ pairs (most reliable)
2. Oversample communion/empathy-favoring pairs (2:1 ratio)
3. Add a Gemma-based RM as an ensemble member to balance values
4. Apply reward shaping to explicitly boost communion-oriented
   completions post-hoc
```

## Best Practices

- **Do:** Always check the base-model family before interpreting RM outputs. Two RMs trained on identical data will diverge if initialized from different pretrained models.
- **Do:** Use the MWLR formula rather than raw log-ratios when comparing models. Raw log-ratios are dominated by low-probability junk tokens that carry no meaningful value signal.
- **Do:** Test with multiple prompt phrasings (50+ variations). Single prompts can produce artifacts; the effect is robust only across diverse prompts.
- **Do:** Report value audit results in model cards. Downstream users need to know whether an RM leans agency or communion.
- **Avoid:** Assuming that preference data alone determines RM values. Below ~100k pairs, the base model's orientation dominates.
- **Avoid:** Treating value orientation as a binary defect. Agency and communion are both legitimate value dimensions; the issue is when the orientation is unintentional or misaligned with the deployment context.

## Error Handling

- **Too few value-laden tokens in top-k:** If your prompt set yields mostly neutral tokens in the top ranks, your prompts are likely too domain-specific. Switch to broader value-eliciting templates ("The most important thing is ___").
- **Inconsistent results across prompts:** High variance suggests prompt sensitivity. Increase prompt count to 50+ and use median (not mean) ranks to reduce outlier influence.
- **No Big Two dictionary match:** If working in a non-English language, the Pietraszkiewicz dictionary may not apply. Translate the 162-noun subset or use a culturally validated equivalent.
- **Base model not in Llama/Gemma families:** The specific agency/communion mapping is empirically established for Llama and Gemma. For other families (Mistral, Qwen, etc.), run the full audit from step 2 without assuming a direction -- the bias may exist along different value axes.
- **MWLR computation yields near-zero scores:** This can happen if the two models being compared are very similar (e.g., two Llama variants). The technique is designed for cross-family comparisons.

## Limitations

- The agency/communion dichotomy is specifically validated for Llama vs. Gemma families. Other model families may exhibit biases along different or orthogonal value dimensions that this framework does not capture.
- The Big Two and MFD2 dictionaries are English-language and Western-psychology-centric. Value orientations may manifest differently across languages and cultures.
- The 100k preference-pair threshold for bias attenuation is an approximation from experiments on 8B-27B parameter models. Larger models may require proportionally more data.
- This method detects value orientation at the token level. More complex value biases that emerge only in multi-sentence reasoning may not be captured by single-token analysis.
- The implicit reward score (MWLR) requires access to model logits, which is unavailable for closed-source models behind APIs.

## Reference

Christian, B., Thompson, J.A.F., Yang, E.M., Adam, V., Kirk, H.R. (2026). *Reward Models Inherit Value Biases from Pretraining*. arXiv:2601.20838. [https://arxiv.org/abs/2601.20838v1](https://arxiv.org/abs/2601.20838v1)

Key finding: RM value orientations trace back to base-model logits and are durable through finetuning -- base-model selection is a values decision, not just a performance decision.