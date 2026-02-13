---
name: "culturally-grounded-personas-characterization"
description: >
  Generate and evaluate culturally-grounded LLM personas using World Values Survey variables,
  Inglehart-Welzel Cultural Map positioning, and Moral Foundations Theory profiling.
  Use this skill when the user asks to:
  - "create culturally diverse personas for user testing"
  - "build a persona pipeline grounded in real survey data"
  - "evaluate cultural bias in LLM-generated personas"
  - "map synthetic personas onto the Inglehart-Welzel cultural map"
  - "generate moral profiles for different cultural configurations"
  - "simulate culturally conditioned survey responses"
---

# Culturally Grounded Persona Characterization

This skill enables Claude to generate synthetic personas that are systematically conditioned on real-world cultural value dimensions derived from the World Values Survey (WVS), then evaluate those personas through three complementary lenses: positioning on the Inglehart-Welzel Cultural Map, demographic-level consistency with human survey data, and moral profiling via Moral Foundations Theory. The technique comes from Greco, La Cava, and Tagarelli (2026), who showed that LLM personas conditioned on interpretable WVS-derived variables produce culturally structured response patterns that broadly track human group differences -- but with important failure modes that this skill also helps detect.

## When to Use

- When the user needs to generate a set of culturally diverse synthetic personas for product testing, UX research, or market analysis
- When building a pipeline that must produce persona responses aligned with specific cultural value profiles (e.g., "traditional-survival" vs. "secular-self-expression")
- When evaluating whether an LLM exhibits cultural bias by comparing its persona outputs against known WVS distributions
- When the user wants to simulate how different cultural groups might respond to moral dilemmas, policy proposals, or product concepts
- When creating a DevOps automation that batch-generates and validates culturally-conditioned personas at scale
- When building a testing harness that checks LLM cultural alignment across demographic slices

## Key Technique

The core insight is that cultural variation in LLM personas can be systematically controlled and measured using two orthogonal frameworks. First, the **Inglehart-Welzel Cultural Map** defines a two-dimensional space: Traditional vs. Secular-rational values (y-axis) and Survival vs. Self-expression values (x-axis). Countries and cultural groups occupy stable positions on this map based on aggregated WVS responses. By conditioning an LLM persona on WVS-derived variables (religious importance, national pride, authority attitudes, trust, tolerance, post-materialist priorities), you can steer where the persona lands on this map and verify whether it lands where expected.

Second, **Moral Foundations Theory (MFT)** provides five (or six) moral dimensions -- Care/Harm, Fairness/Cheating, Loyalty/Betrayal, Authority/Subversion, Sanctity/Degradation, and optionally Liberty/Oppression -- that vary systematically across cultures. The paper administers the Moral Foundations Questionnaire (MFQ-30) to LLM personas and analyzes whether the resulting moral profiles match the culture-to-morality mapping predicted by the persona's cultural conditioning. This creates a closed-loop validation: generate persona from cultural variables, administer moral questionnaire, check whether moral profile aligns with the cultural position.

The practical pipeline is: (1) select WVS conditioning variables, (2) construct a persona prompt embedding those values, (3) generate responses to both WVS-style and MFQ-style items, (4) compute Inglehart-Welzel coordinates from WVS responses, (5) score MFQ foundations, (6) validate alignment between cultural position and moral profile.

## Step-by-Step Workflow

1. **Define the cultural conditioning variables.** Select from these WVS-derived dimensions: importance of religion, importance of family, national pride, confidence in institutions (government, military, press), interpersonal trust, tolerance of outgroups (immigrants, homosexuality), attitude toward authority, gender role attitudes, post-materialist vs. materialist priorities, and economic redistribution views. Each variable should have a clear scale (e.g., 1-4 or 1-10).

2. **Choose target cultural profiles.** Decide which cultural configurations to generate. Use the Inglehart-Welzel Cultural Map zones as anchors: Protestant Europe (high secular, high self-expression), Confucian (high secular, mid survival), African-Islamic (high traditional, high survival), Latin America (mid traditional, mid self-expression), English-speaking (mid secular, high self-expression), etc.

3. **Construct the persona system prompt.** Build a prompt that embeds the conditioning variables as concrete character attributes. Do NOT use vague cultural labels like "act Japanese." Instead, specify the value positions directly:

   ```
   You are a persona with the following value profile:
   - Religion is very important in your life (9/10)
   - You have strong national pride (8/10)
   - You believe children should learn obedience over independence
   - You have low interpersonal trust (3/10)
   - You prioritize economic security over personal freedom
   - You hold traditional gender role views
   - You have low tolerance for social nonconformity
   Respond to all questions from this perspective consistently.
   ```

4. **Administer the WVS item battery.** Present 15-25 WVS questions to the persona covering the key index variables. Use the standard WVS question wording. Collect responses on the original WVS scales (typically 1-4 or 1-10). Store responses as structured JSON.

5. **Compute Inglehart-Welzel coordinates.** Calculate the Traditional/Secular-rational score and Survival/Self-expression score from the WVS responses using the standard index construction:
   - **Traditional vs. Secular-rational**: Aggregate responses on God importance, national pride, respect for authority, abortion justifiability, and autonomy vs. obedience emphasis.
   - **Survival vs. Self-expression**: Aggregate responses on happiness, homosexuality tolerance, petition signing experience, interpersonal trust, and post-materialist priorities.
   - Normalize each index to a [-2, 2] range for map positioning.

6. **Administer the Moral Foundations Questionnaire (MFQ-30).** Present the 30-item MFQ to the persona. This consists of 15 relevance items ("When deciding whether something is right or wrong, to what extent is X relevant?") and 15 judgment items ("Please indicate agreement with: Y"). Score each of the five foundations (Care, Fairness, Loyalty, Authority, Sanctity) by averaging the relevant items (0-5 scale).

7. **Build the culture-to-morality mapping.** Compare the persona's moral profile against expected patterns:
   - Traditional-survival profiles should score high on Authority, Loyalty, Sanctity and lower on Care, Fairness (relative)
   - Secular-self-expression profiles should score high on Care, Fairness and lower on Authority, Sanctity, Loyalty (relative)
   - Flag any persona whose moral profile contradicts its cultural position by more than 1 standard deviation on any foundation.

8. **Validate demographic consistency.** If the persona is conditioned on demographic attributes (age, gender, education, income), compare its WVS response distributions against the actual WVS wave 7 data for the corresponding demographic slice. Use Jensen-Shannon divergence or chi-squared tests per item to quantify alignment.

9. **Generate the characterization report.** Output a structured report containing: the persona's cultural conditioning variables, its Inglehart-Welzel map coordinates (with the target zone), its MFQ-30 foundation scores, alignment flags, and demographic consistency metrics if applicable.

10. **Automate for batch runs.** Wrap steps 3-9 in a parameterized pipeline (Python script, CI job, or API automation) that accepts a JSON specification of cultural profiles and produces a CSV/JSON dataset of characterized personas with validation scores.

## Concrete Examples

**Example 1: Generate and validate a "Protestant Europe" persona**

User: "Create a persona representing Protestant European values and verify it lands in the right zone on the Inglehart-Welzel map."

Approach:
1. Set conditioning variables: religion importance=2/10, national pride=4/10, interpersonal trust=7/10, homosexuality tolerance=9/10, post-materialist priorities=high, authority respect=low, gender equality=high, autonomy emphasis over obedience.
2. Construct persona system prompt embedding these values as character traits.
3. Administer 20 WVS items covering both IW dimensions.
4. Compute scores: Traditional/Secular-rational = +1.4, Survival/Self-expression = +1.6.
5. Administer MFQ-30. Expected: high Care (4.2) and Fairness (4.0), low Authority (1.8) and Sanctity (1.5).
6. Validate: coordinates fall in Protestant Europe zone (+1.0 to +2.0 on both axes). Moral profile matches secular-self-expression pattern.

Output:
```json
{
  "persona_id": "protestant_europe_01",
  "conditioning": {
    "religion_importance": 2, "national_pride": 4,
    "trust": 7, "tolerance": 9, "post_materialist": true
  },
  "inglehart_welzel": {
    "traditional_secular": 1.4,
    "survival_self_expression": 1.6,
    "target_zone": "Protestant Europe",
    "in_zone": true
  },
  "moral_foundations": {
    "care": 4.2, "fairness": 4.0, "loyalty": 2.5,
    "authority": 1.8, "sanctity": 1.5
  },
  "alignment_flags": []
}
```

**Example 2: Batch cultural bias audit across six cultural zones**

User: "Build a pipeline that generates 5 personas per Inglehart-Welzel zone and checks whether GPT-4 produces culturally consistent responses."

Approach:
1. Define six target zones with variable ranges: Protestant Europe, English-speaking, Confucian, Latin America, Orthodox, African-Islamic.
2. For each zone, create 5 persona specs with slight variation in conditioning variables (e.g., religion importance 1-3 for Protestant Europe, 7-10 for African-Islamic).
3. Generate a Python script that loops over specs, constructs prompts, calls the LLM API, administers WVS + MFQ batteries, computes scores.
4. Collect results into a DataFrame. Flag personas outside their target zone or with contradictory moral profiles.
5. Output a summary table showing zone hit rate and average foundation scores per zone.

Output:
```
Zone                 | Hit Rate | Care | Fair | Loyal | Auth | Sanct
---------------------|----------|------|------|-------|------|------
Protestant Europe    | 5/5      | 4.1  | 3.9  | 2.4   | 1.9  | 1.6
English-speaking     | 4/5      | 3.8  | 3.7  | 2.9   | 2.5  | 2.1
Confucian            | 3/5      | 3.2  | 3.0  | 3.5   | 3.8  | 3.0
Latin America        | 4/5      | 3.6  | 3.3  | 3.1   | 3.0  | 3.4
Orthodox             | 3/5      | 3.0  | 2.8  | 3.3   | 3.5  | 3.2
African-Islamic      | 4/5      | 2.9  | 2.6  | 3.8   | 4.1  | 4.3

Flagged: confucian_03 (secular score too low), orthodox_02 (trust too high)
```

**Example 3: CI pipeline for persona cultural drift detection**

User: "Add a CI check that verifies our LLM-based persona service hasn't drifted culturally after a model update."

Approach:
1. Create a reference dataset: 30 personas (5 per zone) with known IW coordinates and MFQ scores from the current model version.
2. Write a pytest suite that re-generates each persona against the updated model.
3. For each persona, assert IW coordinates are within 0.3 of reference values and MFQ scores within 0.5.
4. Integrate into CI as a `cultural_alignment_test` job that runs on model-update PRs.

Output (`test_cultural_alignment.py`):
```python
import pytest
from persona_pipeline import generate_persona, compute_iw, score_mfq

REFERENCE = load_json("reference_personas.json")

@pytest.mark.parametrize("persona_spec", REFERENCE)
def test_cultural_position_stability(persona_spec):
    persona = generate_persona(persona_spec["conditioning"])
    iw = compute_iw(persona["wvs_responses"])
    assert abs(iw["trad_sec"] - persona_spec["ref_trad_sec"]) < 0.3
    assert abs(iw["surv_self"] - persona_spec["ref_surv_self"]) < 0.3

@pytest.mark.parametrize("persona_spec", REFERENCE)
def test_moral_profile_stability(persona_spec):
    persona = generate_persona(persona_spec["conditioning"])
    mfq = score_mfq(persona["mfq_responses"])
    for foundation in ["care", "fairness", "loyalty", "authority", "sanctity"]:
        assert abs(mfq[foundation] - persona_spec[f"ref_{foundation}"]) < 0.5
```

## Best Practices

- **Do:** Use specific WVS variable values (numeric scales) in persona prompts rather than cultural labels like "Japanese" or "American." Labels activate stereotypes; variables activate value positions.
- **Do:** Always administer both WVS and MFQ instruments to get cross-validation between cultural position and moral profile. One without the other leaves blind spots.
- **Do:** Include multiple personas per cultural zone with variable ranges to test consistency rather than relying on a single exemplar.
- **Do:** Use the standard WVS question wording verbatim. Paraphrasing changes response distributions.
- **Avoid:** Conditioning on country name alone. Country is a confound that mixes culture, politics, economics, and stereotypes. Condition on value dimensions instead.
- **Avoid:** Treating LLM persona responses as ground truth for any cultural group. These are synthetic approximations useful for testing pipelines, not substitutes for real survey data.
- **Avoid:** Comparing raw MFQ scores across different LLMs without normalization. Different models have different response scale biases (e.g., GPT-4 tends toward moderate scores, Claude toward slightly higher variance).

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Persona lands outside all IW zones | Conditioning variables are contradictory (e.g., high religion + high tolerance + low authority) | Check variable coherence against actual WVS country profiles before prompting |
| MFQ scores are flat across all foundations | LLM defaults to "moderate agreement" on all items | Add a re-prompting step that asks the persona to justify extreme positions, or increase temperature |
| Persona ignores conditioning and gives "balanced" answers | System prompt is too weak or user turn overrides it | Move conditioning into a reinforced system prompt with explicit instructions to maintain the value profile consistently |
| Demographic consistency test fails on all items | Wrong WVS wave data used as reference, or demographic slice too narrow | Use WVS Wave 7 (2017-2022) data and ensure demographic slices have n>50 in the reference |
| Batch pipeline produces identical personas for different specs | Temperature too low or prompt differences too subtle | Increase temperature to 0.7-1.0 and ensure conditioning variables differ by at least 3 points on key dimensions |

## Limitations

- LLM personas are **approximations**, not simulations. They reflect training data patterns, not actual cultural cognition. Do not use them to make claims about real cultural groups.
- The Inglehart-Welzel map is a macro-level tool designed for country-level aggregates. Individual-level persona positioning is an extrapolation that may not be meaningful for edge cases.
- Moral Foundations Theory is one of several competing moral psychology frameworks. MFQ-30 scores are useful for structured comparison but do not capture all moral reasoning dimensions.
- LLMs exhibit **sycophancy** -- they may adjust responses to match what they infer the prompter wants rather than maintaining the conditioned value profile. Repeated administration can reveal this drift.
- This technique works best for broad cultural positioning (which quadrant of the IW map) and relative moral profile shapes. It is not precise enough for fine-grained cultural distinctions (e.g., distinguishing Danish from Swedish value profiles).
- Results are model-dependent. A pipeline validated on one LLM version must be re-validated after model updates, as cultural priors shift with training data changes.

## Reference

Greco, C. M., La Cava, L., & Tagarelli, A. (2026). *Culturally Grounded Personas in Large Language Models: Characterization and Alignment with Socio-Psychological Value Frameworks*. arXiv:2601.22396. [https://arxiv.org/abs/2601.22396](https://arxiv.org/abs/2601.22396)

Look for: The WVS variable selection rationale (Section 3), persona prompt construction (Section 4), IW index computation methodology (Section 5), and the culture-to-morality mapping analysis (Section 6) which reveals which cultural zones LLMs reproduce faithfully vs. where they collapse into Western-default responses.