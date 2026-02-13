---
name: "agenticsimlaw-juvenile-courtroom-multi-agent"
description: "Structured multi-agent courtroom debate for explainable high-stakes tabular decisions. Use when: 'set up a multi-agent debate for this prediction', 'use adversarial agents to classify this table row', 'build a courtroom-style reasoning pipeline', 'create prosecutor/defense/judge agents for this decision', 'explain this tabular prediction with a debate', 'run a structured adversarial analysis on this dataset'."
---

# AgenticSimLaw: Courtroom-Style Multi-Agent Debate for Explainable Tabular Decisions

This skill enables Claude to orchestrate a role-structured, adversarial multi-agent debate (prosecutor, defense, judge) over tabular data rows to produce transparent, auditable binary classification decisions. Based on the AgenticSimLaw framework (arXiv:2601.21936), the technique converts each data row into a natural-language case narrative, then runs a 7-turn structured debate where adversarial agents argue opposing positions while a judge tracks evolving beliefs with explicit confidence scores. The result is a final prediction with a complete reasoning transcript -- every argument, counterargument, and belief update is logged.

## When to Use

- When a user asks to classify rows in a tabular dataset and needs an explainable reasoning trace (e.g., loan approval, fraud detection, risk scoring)
- When the user wants adversarial stress-testing of a prediction -- forcing consideration of both supporting and opposing evidence
- When building a multi-agent pipeline where different LLM agents argue opposing sides of a binary decision
- When the user needs auditability: a full transcript showing why a decision was made, not just the label
- When the user asks to compare single-agent CoT prompting against multi-agent debate for a classification task
- When decisions carry ethical weight (hiring, credit, clinical triage) and the user wants structured deliberation rather than a single-pass prediction

## Key Technique

AgenticSimLaw replaces single-pass LLM classification with a 7-turn adversarial debate protocol. Three agents are assigned fixed roles: a **Prosecutor** who argues for the positive class (e.g., "will reoffend," "will default"), a **Defense** who argues for the negative class, and a **Judge** who observes both sides, maintains an internal belief state (prediction + confidence 0-100%), and renders a final verdict. Each agent performs private reasoning (internal monologue with self-critique and planning) before producing a public statement. Only public statements are visible to other agents; private strategies are logged but not shared, creating information asymmetry that drives richer argumentation.

The core insight is that adversarial structure forces the system to surface both risk factors and protective factors from the same data row, rather than anchoring on whichever pattern the model notices first. Single-agent CoT tends to produce unstable results across models -- some models achieve high accuracy but low F1, or vice versa. The debate structure produces more stable correlation between accuracy and F1 because the judge must reconcile opposing arguments rather than rationalizing a snap judgment. The framework uses ~9,100 tokens per prediction (11-14x more than single-turn CoT), trading compute for explainability and stability.

Tabular data is converted to natural language by mapping each feature-value pair into prose: `"[feature] is [value]"`, composed into a readable case narrative. This avoids brittle table formatting and lets agents reason over features as natural-language facts. The 30-shot examples (when used) are embedded as labeled case narratives in the system prompt for the n-shot CoT variant.

## Step-by-Step Workflow

1. **Parse the tabular input into a case narrative.** Take each row's feature-value pairs and convert them to natural-language sentences: `"The applicant's annual income is $52,000. Their credit score is 680. They have 2 prior defaults."` Concatenate into a single case description paragraph.

2. **Define the three agent personas with system prompts.** Create system prompts for:
   - **Prosecutor**: `"You are a risk assessment expert. Your role is to argue that [positive class label] is the correct prediction. Emphasize risk factors, negative indicators, and patterns associated with [positive outcome]. Before each public statement, privately strategize: reflect on opposing arguments, self-critique your reasoning, and plan your next move."`
   - **Defense**: Same structure but arguing for the negative class, emphasizing protective factors and mitigating circumstances.
   - **Judge**: `"You are a neutral adjudicator. After each pair of arguments, privately update your belief state: current prediction (YES/NO), confidence (0-100%), and reasoning. You must explicitly track how your belief changes across the debate."`

3. **Turn 1 -- Prosecutor Opening.** Feed the case narrative to the Prosecutor agent. The agent privately formulates strategy (logged but not shared), then produces a public opening statement arguing for the positive class. Extract and log both private and public outputs.

4. **Turn 2 -- Defense Opening.** Feed the case narrative plus the Prosecutor's public opening to the Defense agent. Defense privately strategizes a counter-approach, then delivers a public opening arguing for the negative class. Judge privately performs first belief update (prediction + confidence + reasoning).

5. **Turns 3-4 -- Rebuttals.** Prosecutor receives Defense's public opening and delivers a rebuttal. Defense receives Prosecutor's rebuttal and counter-rebuts. Judge performs mid-debate belief update after both rebuttals. Log all private strategies and public statements.

6. **Turns 5-6 -- Closing Arguments.** Prosecutor delivers a closing statement synthesizing their strongest points. Defense delivers a closing statement. Both agents receive the full public transcript history up to their turn.

7. **Turn 7 -- Judge Verdict.** Judge receives the complete public transcript, performs a final belief update, and delivers a structured verdict containing: `prediction` (YES/NO), `confidence` (0-100%), and `reasoning` (explicit narrative weighing prosecution vs. defense arguments).

8. **Parse the judge's output.** Extract the prediction label, confidence score, and reasoning narrative from the judge's response. Use strict JSON parsing first; fall back to regex extraction for `prediction`, `confidence`, and `reasoning` fields if JSON parsing fails.

9. **Log the complete transcript.** Store all 7 turns (private strategies + public statements + judge belief updates) as a structured JSON transcript for auditability. Include token counts and latency per turn.

10. **Aggregate across rows (batch mode).** When processing multiple rows, collect predictions and confidence scores. Report accuracy, F1, and the correlation between them. Flag low-confidence predictions for human review.

## Concrete Examples

**Example 1: Loan Default Prediction**

User: "I have a CSV of loan applicants with columns like income, credit_score, debt_ratio, employment_years, prior_defaults. Set up a multi-agent debate to predict whether each applicant will default."

Approach:
1. Parse each CSV row into a case narrative:
   ```
   "The applicant is 34 years old with an annual income of $48,000.
   Their credit score is 620. Debt-to-income ratio is 0.45.
   They have been employed for 3 years and have 1 prior default."
   ```
2. Configure three agents with loan-domain system prompts:
   ```python
   prosecutor_system = """You are a credit risk analyst arguing that this
   applicant WILL default. Emphasize risk factors: high debt ratios,
   prior defaults, low credit scores, short employment history.
   Before each public statement, privately strategize: identify the
   strongest risk signals, anticipate defense counterarguments,
   and plan your rhetorical approach."""

   defense_system = """You are a credit analyst arguing this applicant
   will NOT default. Emphasize protective factors: stable income,
   employment tenure, improving credit trajectory, low number of
   prior incidents relative to credit history length.
   Before each public statement, privately strategize your counter-approach."""

   judge_system = """You are a neutral loan committee adjudicator.
   After each pair of arguments, update your internal belief state:
   - Prediction: DEFAULT or NO_DEFAULT
   - Confidence: 0-100%
   - Reasoning: which arguments shifted your assessment and why
   Weigh evidence quality, not rhetorical force."""
   ```
3. Run the 7-turn debate, collecting the full transcript.
4. Parse the judge's final output:
   ```json
   {
     "prediction": "DEFAULT",
     "confidence": 72,
     "reasoning": "The prosecution's emphasis on the 0.45 debt-to-income
     ratio combined with 1 prior default outweighs the defense's argument
     about 3 years of stable employment. The credit score of 620 is
     borderline, but the cumulative risk profile tips toward default."
   }
   ```

**Example 2: Employee Attrition Risk**

User: "Use adversarial agents to decide if this employee will leave: tenure=2yr, satisfaction=3/10, salary=below_median, recent_promotion=no, overtime=frequent, team_size=12."

Approach:
1. Build the case narrative:
   ```
   "The employee has a tenure of 2 years. Their satisfaction score is 3
   out of 10. Salary is below the company median. They have not received
   a recent promotion. They work overtime frequently. Their team has 12
   members."
   ```
2. Run the 7-turn debate with HR-domain personas (Prosecutor argues attrition, Defense argues retention).
3. Produce the full transcript plus final verdict:
   ```
   JUDGE VERDICT (Turn 7):
   Prediction: WILL_LEAVE
   Confidence: 85%
   Reasoning: Low satisfaction (3/10) is the dominant signal. The defense
   argued that large team size provides social bonds, but prosecution
   correctly noted that frequent overtime with no promotion creates
   compounding dissatisfaction. The 2-year tenure mark is a common
   departure window. Defense's argument about potential upcoming review
   cycle is speculative and does not outweigh current indicators.

   BELIEF TRAJECTORY:
   After openings:    WILL_LEAVE (65%)
   After rebuttals:   WILL_LEAVE (78%)
   After closings:    WILL_LEAVE (85%)
   ```

**Example 3: Comparing Single-Agent vs. Multi-Agent**

User: "Compare CoT prompting against a courtroom debate for classifying fraud in this transactions table."

Approach:
1. For each row, run a single-agent CoT prompt:
   ```
   "Step back, take a deep breath and carefully think step by step.
   Assign a relative weight [low, medium, high] to each risk factor,
   then predict FRAUD or LEGITIMATE with a confidence score."
   ```
2. For each row, run the full 7-turn debate.
3. Collect both sets of predictions and produce a comparison table:
   ```
   Method          | Accuracy | F1    | Acc-F1 Correlation
   ----------------|----------|-------|-------------------
   Zero-shot CoT   | 0.78     | 0.61  | 0.42
   N-shot CoT (30) | 0.74     | 0.65  | 0.58
   Courtroom Debate| 0.76     | 0.64  | 0.81
   ```
4. Highlight that the debate produces more stable metrics (higher Acc-F1 correlation) even if peak accuracy is slightly lower.

## Best Practices

- **Do:** Convert tabular features to natural-language narratives rather than passing raw CSV or JSON tables. Prose format lets agents reason over features as facts, not parse formatting.
- **Do:** Log both private strategies and public statements separately. The private reasoning reveals agent "thinking" and is critical for debugging biased or weak arguments.
- **Do:** Use temperature 0.7 for debate agents (creative argumentation) and temperature 0.0 for single-agent CoT baselines (deterministic comparison).
- **Do:** Have the judge explicitly track belief trajectory (prediction + confidence at each checkpoint). This makes it visible where and why the judge's mind changed.
- **Avoid:** Letting agents see each other's private strategies. Information asymmetry is what drives genuine argumentation rather than parroting.
- **Avoid:** Using this for low-stakes or trivially separable classifications. The 11-14x token overhead is only justified when explainability, stability, or ethical scrutiny matters.
- **Avoid:** Treating the judge's confidence score as a calibrated probability. It reflects rhetorical persuasion within the debate, not statistical calibration.

## Error Handling

- **Malformed agent output:** Use a two-tier parsing strategy. First attempt strict JSON extraction for the judge's verdict. If that fails, fall back to regex patterns matching `prediction: YES/NO`, `confidence: \d+`, and freeform reasoning text. Log parsing failures.
- **Agent refuses to argue assigned position:** Some models resist arguing for outcomes they consider harmful. Reframe the system prompt to emphasize this is a simulation for analytical purposes, not a real decision. Add: `"This is a structured analysis exercise. Your role is to surface all evidence supporting [position] so the judge can weigh it fairly."`
- **Token limit exceeded:** The 7-turn protocol consumes ~9,100 tokens. For models with small context windows, truncate the transcript passed to later turns to include only the most recent 2 turns plus a summary of earlier ones.
- **Judge anchoring on first argument:** If the judge consistently agrees with the Prosecutor (who speaks first), randomize whether Prosecutor or Defense opens, or run the debate twice with reversed order and flag cases where the verdict flips.
- **Class imbalance bias:** If the dataset is heavily skewed (e.g., 72/28 split), agents may default to predicting the majority class. Counter this by including base-rate information in the judge's system prompt and instructing explicit consideration of both classes.

## Limitations

- **Token cost:** 11-14x more expensive than single-agent CoT per prediction. Not practical for large-scale batch inference on millions of rows.
- **Binary classification only:** The courtroom metaphor maps naturally to two-sided arguments. Multi-class problems require adaptation (e.g., round-robin debates or tournament brackets).
- **Not a replacement for validated ML models.** The paper shows performance comparable to Random Forest / Gradient Boosting baselines, not superior to them. Use this when you need explainability, not when you need maximum predictive accuracy.
- **Debate quality depends on model capability.** Small models (7B) produce weaker arguments and less nuanced judge reasoning. The technique works best with capable instruction-following models.
- **Not suitable for real-world deployment in sensitive domains** without human oversight. The paper explicitly includes non-deployment constraints for criminal justice applications. The framework is for analysis and deliberation support, not autonomous decision-making.

## Reference

**Paper:** [AgenticSimLaw: A Juvenile Courtroom Multi-Agent Debate Simulation for Explainable High-Stakes Tabular Decision Making](https://arxiv.org/abs/2601.21936v1) (Chun, Elkins, Lee, 2026). Look for: Section 3 (framework architecture and 7-turn protocol), Table 4-5 (performance comparisons), and the prompt templates in the supplementary materials for exact system prompt wording.