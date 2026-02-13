---
name: "can-truly-embody-human"
description: "Evaluate and improve personality-behavior alignment in LLM simulations of human social interactions. Uses the BFI-IRP evaluation framework from Kwon et al. (2026) to audit whether personality-prompted agents reproduce real human conflict patterns. Trigger phrases: 'validate personality simulation', 'audit LLM personality alignment', 'dispute resolution simulation', 'personality-prompted agent evaluation', 'BFI behavior alignment check', 'test if my agent acts human'"
---

# Personality-Behavior Alignment Evaluation for LLM Simulations

This skill enables Claude to audit, evaluate, and improve LLM-based simulations where agents are prompted with personality traits to replicate human behavior in social interactions such as negotiation, mediation, and dispute resolution. It applies the BFI-IRP evaluation framework from Kwon et al. (AAAI 2026), which provides interpretable metrics comparing personality-driven strategic behavior and conflict outcomes between human data and LLM-generated dialogues. The core insight: personality-prompted LLMs systematically diverge from human trait-behavior relationships in predictable, measurable ways — and this skill teaches you how to detect, quantify, and mitigate those divergences.

## When to Use

- When building multi-agent simulations that assign Big Five personality traits to LLM agents (negotiation bots, social simulations, role-playing systems)
- When a user asks to validate whether their personality-prompted agents behave like real humans would
- When designing dispute resolution, mediation, or negotiation systems powered by LLMs
- When auditing an existing agent framework for psychological realism before deploying in socially impactful contexts (legal, HR, education)
- When a user wants to benchmark how different LLMs respond to personality prompts in conflict scenarios
- When constructing personality-conditioned datasets that need to parallel human behavioral distributions

## Key Technique

The paper introduces a two-layer evaluation framework. The first layer measures **final outcomes**: negotiation payoff scores, acceptance rates, and walk-away rates. The second layer measures **strategic behavior** using the Interests-Rights-Power (IRP) framework, which classifies each utterance in a dialogue into one of eight strategy types grouped into four categories — Cooperative (Proposal, Concession, Interests, Positive Expectations), Neutral (Facts, Procedural), Competitive (Power, Rights), and Residual. From these classifications, four IRP metrics are computed: IRP Ratio (relative frequency of each strategy), IRP Reciprocity (how often a speaker mirrors their partner's strategy), Escalation Ratio (competitive responses to non-competitive turns), and De-escalation Ratio (non-competitive responses to competitive turns).

The critical finding is that personality-behavior mappings in LLMs are systematically distorted compared to humans. In human data, **neuroticism** is the strongest predictor of strategic outcomes — it reduces offer acceptance and predicts avoidant behavior. But LLMs show weak neuroticism effects while being oversensitive to **extraversion** and **agreeableness**. LLMs also exhibit temporal inflexibility: humans follow a structured progression (Facts early, then Interest/Proposal/Concession, then Residual), while LLMs show flat trajectories with premature concession behavior. These divergences mean that even well-prompted personality agents may produce psychologically invalid simulations.

The mitigation approach involves: (1) measuring alignment gaps using regression models that predict behavioral metrics from personality traits, (2) comparing coefficient magnitudes and significance between human baselines and LLM outputs, and (3) applying corrective weighting or prompt adjustments to reduce the most impactful divergences. The paper uses the regression model `DV_k = B0 + sum(Bj * SELF_j) + sum(Gj * PARTNER_j) + B11 * Position` where DV_k is any behavioral metric, SELF/PARTNER are BFI trait scores, and Position encodes role assignment.

## Step-by-Step Workflow

1. **Define the simulation scenario and roles.** Specify the dispute context (e.g., buyer/seller, employer/employee, landlord/tenant), the issues under negotiation (at least 3 distinct issues), and the role assignments. Each role needs clear issue preferences with numeric importance weights.

2. **Construct personality profiles using the BFI polarity-degree scale.** For each agent, assign Big Five traits (Extraversion, Conscientiousness, Agreeableness, Neuroticism, Openness) on a six-point polarity scale: `+++` (very high), `++` (high), `+` (slightly high), `-` (slightly low), `--` (low), `---` (very low). Sample from empirical human distributions to ensure realistic trait combinations.

3. **Generate personality prompts using the adjective-intensity method.** For each trait level, select 3 adjectives from Goldberg's (1992) bipolar inventory. Modulate with intensity words: "very [adj]" for extreme levels, bare "[adj]" for moderate, "a bit [adj]" for slight. The full prompt includes 15 adjectives (3 per trait). Keep the personality section minimal — do not over-engineer trait expression through behavioral instructions.

4. **Link personality to issue importance weights.** Apply empirically grounded mappings: weight Apology/relationship-repair issues by agreeableness score. Assign remaining issue importances randomly or through domain-specific mappings. This ensures trait-outcome coupling matches human patterns.

5. **Run the simulation with controlled parameters.** Set maximum dialogue turns (25 is the paper's standard), use default model temperatures (1.0), and enforce multi-issue engagement (agents must negotiate across at least 3 issues). Include termination conditions: ACCEPT, WALK-AWAY, or NO-AGREEMENT at turn limit.

6. **Classify each utterance using the IRP taxonomy.** Tag every subject-verb sequence in each turn as one of 8 types: Proposal, Concession, Interests, Positive Expectations (Cooperative), Facts, Procedural (Neutral), Power, Rights (Competitive), or Residual. A single turn can contain multiple strategy types. Use an LLM classifier validated to A-Kappa >= 0.80.

7. **Compute the seven behavioral metrics.** Calculate: (a) Score — inner product of agreed allocation and issue preferences; (b) Accept rate; (c) Walk-away rate; (d) IRP Ratio per strategy type; (e) IRP Reciprocity; (f) Escalation Ratio; (g) De-escalation Ratio. Aggregate across all dialogues.

8. **Run regression analysis to extract personality-behavior coefficients.** Fit OLS for continuous metrics and logistic regression for binary outcomes. Independent variables: self traits, partner traits, and role position. Compare coefficient magnitudes, signs, and significance against the known human baselines from the paper.

9. **Identify alignment gaps using the divergence checklist.** Flag these known LLM failure modes: (a) neuroticism showing weak or no effect on outcomes, (b) agreeableness or extraversion showing disproportionately strong effects on scores, (c) flat temporal strategy trajectories instead of Facts-first progression, (d) premature concession behavior, (e) rigid escalation patterns instead of context-sensitive adaptation.

10. **Apply corrective adjustments and re-evaluate.** For each identified gap, apply targeted fixes: adjust personality prompt wording for underexpressed traits, add temporal scaffolding instructions ("begin by establishing facts before making proposals"), reduce over-reactive escalation with explicit de-escalation heuristics, or re-weight issue importance mappings. Re-run steps 6-9 to verify improvement.

## Concrete Examples

**Example 1: Auditing a Customer Service Negotiation Bot**

User: "I built a negotiation agent that takes Big Five personality inputs and simulates customer complaint resolution. How do I check if the personality actually affects behavior realistically?"

Approach:
1. Run 200+ simulations with personality profiles sampled from a normal distribution across all five BFI dimensions
2. Classify every agent utterance using the IRP taxonomy (Cooperative/Neutral/Competitive/Residual)
3. Compute the seven behavioral metrics per dialogue
4. Regress each metric on the five personality traits plus partner traits
5. Compare results against human baselines

Output:
```
ALIGNMENT AUDIT RESULTS
=======================
Metric: Negotiation Score
  Human baseline:    No significant trait predictors
  Your agent:        Agreeableness B=-3.12 (p<.001) — DIVERGENT
  Issue:             Agent over-penalizes agreeable personalities

Metric: Escalation Ratio
  Human baseline:    Balanced, context-sensitive (~0.3-0.5)
  Your agent:        0.72 for low-agreeableness profiles — DIVERGENT
  Issue:             Agent escalates too aggressively for disagreeable traits

Metric: Strategy Temporal Progression
  Human baseline:    Facts(early) → Proposals(mid) → Concessions(late)
  Your agent:        Concessions appear in turn 2 — DIVERGENT
  Issue:             Agent skips factual grounding phase

Recommendations:
  1. Add temporal scaffolding: "In the first 5 turns, focus on
     sharing facts and understanding the situation"
  2. Reduce agreeableness weight on score calculations
  3. Add escalation dampening for low-agreeableness profiles
```

**Example 2: Building a Personality-Prompted Dispute Resolution Dataset**

User: "I need to generate 500 LLM dispute dialogues with matched personality profiles for training a mediation classifier."

Approach:
1. Define the dispute scenario with 3+ negotiable issues and role-specific preferences
2. Sample 500 personality profile pairs from the empirical BFI distribution
3. Construct adjective-based personality prompts (15 adjectives per agent, intensity-modulated)
4. Link agreeableness to apology/relationship issue importance weights
5. Run simulations with 25-turn max, temperature=1.0, multi-issue engagement enforced
6. Validate behavioral distributions against human reference data

Output:
```python
# Personality profile generation
import numpy as np

BFI_TRAITS = ["EXT", "CON", "AGR", "NEU", "OPE"]
POLARITY_SCALE = ["---", "--", "-", "+", "++", "+++"]

# Adjective bank (3 per trait pole, from Goldberg 1992)
ADJECTIVES = {
    "EXT+": ["outgoing", "sociable", "talkative"],
    "EXT-": ["reserved", "quiet", "withdrawn"],
    "AGR+": ["sympathetic", "warm", "cooperative"],
    "AGR-": ["cold", "harsh", "uncooperative"],
    # ... (full bank for all 10 poles)
}

INTENSITY = {"+++": "very", "++": "", "+": "a bit",
             "-": "a bit", "--": "", "---": "very"}

def build_personality_prompt(profile: dict) -> str:
    """Generate 15-adjective personality description."""
    adjectives = []
    for trait, level in profile.items():
        pole = "+" if level in ["+", "++", "+++"] else "-"
        key = f"{trait}{pole}"
        modifier = INTENSITY[level]
        for adj in ADJECTIVES[key]:
            desc = f"{modifier} {adj}".strip()
            adjectives.append(desc)
    return "You are " + ", ".join(adjectives) + "."

def weight_issues(profile: dict, base_weights: dict) -> dict:
    """Adjust issue importance by agreeableness."""
    agr_level = ["+", "++", "+++"].index(profile["AGR"]) + 1 \
                if profile["AGR"] in ["+", "++", "+++"] else 0
    weights = base_weights.copy()
    weights["apology"] += agr_level * 2.13  # empirical coefficient
    return weights
```

**Example 3: Comparing Two LLMs for Personality Fidelity**

User: "I'm choosing between GPT-4o and Claude for a legal mediation simulator. Which one handles personality prompts more realistically?"

Approach:
1. Run 250 matched simulations on each model with identical personality profiles and scenarios
2. Compute all seven behavioral metrics for both
3. Regress metrics on BFI traits for each model separately
4. Compare coefficient patterns against human baselines from the paper

Output:
```
MODEL COMPARISON: Personality-Behavior Alignment
=================================================

                        Human   GPT-4o   Claude   Gemini
Neuroticism→Score       strong  weak     weak     weak
Agreeableness→Score     none    strong*  moderate moderate
Escalation sensitivity  medium  high*    low      medium
De-escalation bias      medium  low      high*    medium
Facts usage (early)     high    low*     moderate low*
Concession timing       late    early*   early*   early*
IRP Reciprocity         0.45    0.62*    0.38     0.51

* = significant divergence from human baseline

Summary:
- Claude: Best on competitive reciprocity and facts usage;
  over-indexes on de-escalation (too accommodating)
- GPT-4o: Most sensitive to trait prompts but over-escalates
  and shows strongest agreeableness-score coupling
- Gemini: Highest Power/Rights usage; worst strategy diversity

Recommendation: Claude with temporal scaffolding prompts and
reduced de-escalation bias for this use case.
```

## Best Practices

- **Do:** Sample personality profiles from empirical BFI distributions rather than using extreme or uniform values. Realistic trait combinations produce more meaningful behavioral variation.
- **Do:** Validate your IRP utterance classifier independently before using it at scale. Target A-Kappa >= 0.80 across all strategy categories.
- **Do:** Always compare against human behavioral baselines, not just against other LLMs. Two LLMs agreeing does not mean either is human-like.
- **Do:** Include partner personality traits as independent variables in regression analysis — human conflict behavior is shaped by the dyad, not just the individual.
- **Avoid:** Over-engineering personality prompts with explicit behavioral instructions. The point is to test whether the model inherently represents personality-driven behavior, not to script it.
- **Avoid:** Treating any single LLM's personality response as ground truth. The paper shows every tested model diverges from human patterns in different ways.
- **Avoid:** Deploying personality-prompted agents in high-stakes social applications (legal mediation, counseling, HR) without first running this evaluation framework and demonstrating acceptable alignment on all seven metrics.

## Error Handling

- **Low strategy classification agreement:** If your IRP classifier falls below A-Kappa 0.80, use majority-vote ensembles of 3+ classifier runs or switch to human annotation for a validation subset.
- **Insufficient simulation count:** Regression coefficients become unreliable below ~200 dialogues. If you see non-significant results across all traits, increase sample size before concluding there is no personality effect.
- **Degenerate dialogues:** LLMs sometimes produce dialogues that terminate in 1-2 turns (immediate acceptance) or loop without progress. Filter these out — they indicate prompt issues, not personality effects. Check that multi-issue engagement constraints are being enforced.
- **Trait collinearity:** Big Five traits are theoretically independent but can correlate in sampled data. Check variance inflation factors (VIF < 5) in your regression models; re-sample profiles if collinearity is high.
- **Temporal analysis failures:** If strategy progression plots look noisy, bin turns into phases (early: turns 1-8, mid: 9-17, late: 18-25) rather than analyzing turn-by-turn.

## Limitations

- The framework is validated on a single dispute scenario (jersey purchase). Generalization to other conflict types (workplace, family, international) requires additional human baseline data.
- Only closed-source LLMs were tested. Open-source models may show different divergence patterns.
- The IRP taxonomy covers strategic behavior but not emotional expression, tone, or pragmatic nuance — all of which contribute to perceived personality in human interactions.
- The adjective-based personality prompting method intentionally uses minimal behavioral instruction. Systems that use detailed persona backstories or few-shot behavioral examples may show different alignment patterns.
- Human baselines come from a specific cultural and demographic context (the KODIS dataset). Personality-behavior mappings vary across cultures, and this framework does not account for cross-cultural differences.

## Reference

Kwon, D., Shrestha, K., Han, B., Lin, S., & Hale, J. (2026). *Can LLMs Truly Embody Human Personality? Analyzing AI and Human Behavior Alignment in Dispute Resolution.* AAAI 2026 (Special Track: AISI). [arXiv:2602.07414](https://arxiv.org/abs/2602.07414v1) — Read for the full regression tables comparing human vs. LLM personality-behavior coefficients across all seven metrics, the complete IRP taxonomy with annotation guidelines, and the temporal strategy progression analysis.