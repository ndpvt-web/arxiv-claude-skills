---
name: "lingxidiagbench-multi-agent-framework-benchmarking"
description: "Build multi-agent benchmarking systems with role-separated agents (simulator, interviewer, evaluator) for structured multi-turn dialogue evaluation. Inspired by the LingxiDiagBench psychiatric consultation framework. Trigger phrases: 'build a multi-agent benchmark', 'create a doctor-patient simulation', 'evaluate LLMs on multi-turn dialogue', 'benchmark diagnostic reasoning', 'multi-agent consultation framework', 'simulate structured interviews with LLMs'"
---

# Multi-Agent Framework for Benchmarking LLMs on Structured Multi-Turn Dialogue

This skill teaches Claude to design and implement multi-agent benchmarking systems where specialized agents (simulator, interviewer, judge) interact in structured multi-turn dialogues to evaluate LLM capabilities. The core technique comes from LingxiDiagBench, which separates the patient simulation, clinical interviewing, and diagnostic evaluation into distinct agent roles -- each with constrained prompts and measurable outputs -- then compares static (pre-recorded dialogue) versus dynamic (live agent interaction) evaluation modes to surface gaps in information-gathering strategy.

## When to Use

- When the user wants to benchmark LLMs on tasks requiring multi-turn information gathering (medical consultation, customer support triage, investigative interviews)
- When building a simulation harness where one LLM plays an information-holder (patient, customer, witness) and another plays an information-seeker (doctor, support agent, interviewer)
- When the user needs to evaluate whether an LLM can ask the right questions, not just answer them
- When designing an LLM-as-a-Judge pipeline to score dialogue quality alongside task accuracy
- When comparing static evaluation (score LLMs on fixed dialogues) with dynamic evaluation (let LLMs conduct their own dialogues)
- When creating synthetic datasets of multi-turn conversations aligned to real-world demographic and categorical distributions

## Key Technique

**Role-Separated Multi-Agent Architecture.** The framework decomposes evaluation into three distinct agent roles. The *Simulator Agent* holds ground-truth case information and responds to questions in character -- constrained to brief, naturalistic answers without volunteering unsolicited details. The *Interviewer Agent* conducts a structured multi-turn dialogue using one of several information-gathering strategies (free-form, decision-tree-guided, or RAG-enhanced phased protocols). The *Evaluator Agent* independently scores both the dialogue quality and the final task output (e.g., a diagnosis). This separation prevents information leakage and lets you measure each component independently.

**Static vs. Dynamic Evaluation Modes.** A key insight from the paper is that LLMs that perform well on pre-recorded dialogues (static evaluation) often perform worse when they must gather information themselves (dynamic evaluation). This gap reveals whether a model's reasoning ability is bottlenecked by poor questioning strategy. Implementing both modes in your benchmark exposes this distinction: static mode tests reasoning-from-evidence, while dynamic mode tests the full loop of question planning, evidence collection, and reasoning.

**Phased Consultation with RAG Enhancement.** The best-performing interviewer strategy follows a five-phase protocol: (1) Screening -- 3-5 broad questions about chief complaints and duration; (2) Assessment -- 5-8 questions on core symptoms and functional impact; (3) Deep-dive -- 3-6 targeted questions on specifics and underlying causes; (4) Risk Assessment -- 2-4 safety-critical checks; (5) Closure -- 1-2 confirmation questions. At each phase, a RAG component retrieves relevant diagnostic guidelines for the top-3 candidate categories, steering the interviewer toward discriminating questions. This phased approach improved end-to-end accuracy by ~5% over unstructured questioning.

## Step-by-Step Workflow

1. **Define the domain taxonomy.** Enumerate the categories your benchmark must distinguish (e.g., 12 ICD-10 psychiatric categories, product issue types, legal case types). Specify the classification granularity levels -- binary, multi-class, and fine-grained -- since LLM performance degrades dramatically with category count (92% binary vs. 28% 12-way in the paper).

2. **Design the case profile schema.** Create a structured format for ground-truth cases that the Simulator Agent will embody. Include demographic fields, category labels, key information points, and response-style constraints. Example fields: `age`, `gender`, `primary_category`, `secondary_category`, `key_facts[]`, `emotional_tone`, `verbosity_level`.

3. **Build the Simulator Agent with behavioral constraints.** Write a system prompt that instructs the LLM to role-play the case profile with strict rules: answer only what is asked, use colloquial language, express appropriate uncertainty when the case profile lacks specific details, and keep responses brief (calibrate token limits from real-world data distributions).

4. **Implement Interviewer Agent strategies at multiple levels.** Create at least three interviewer variants: (a) *Free-form* -- the LLM decides what to ask with no external guidance; (b) *Structured* -- follow a decision tree or checklist of topics; (c) *RAG-enhanced phased* -- follow a multi-phase protocol where each phase retrieves relevant reference material (guidelines, knowledge base entries) for the top candidate categories to inform the next set of questions.

5. **Build the phased consultation controller.** Implement a state machine or turn counter that tracks which phase the interviewer is in (Screening -> Assessment -> Deep-dive -> Risk -> Closure). At each phase transition, invoke the RAG retrieval to update the interviewer's context with relevant guidelines for the current top-3 candidate categories.

6. **Create the Evaluator Agent with separate scoring dimensions.** Design two independent evaluation pipelines: (a) *Task accuracy* -- compare the final classification output against ground truth using accuracy, macro-F1, and weighted-F1; (b) *Dialogue quality* -- use an LLM-as-a-Judge to score along 5-6 rubric dimensions (e.g., clinical appropriateness, thoroughness, ethical conduct, communication quality), each on a defined scale with anchor descriptions.

7. **Implement static evaluation mode.** Feed pre-recorded complete dialogues directly to the Evaluator Agent. This tests pure reasoning-from-evidence without the information-gathering step. Use a fixed test set (e.g., 1,000 cases) for reproducible comparison across models.

8. **Implement dynamic evaluation mode.** Wire the Simulator and Interviewer Agents into a conversation loop with a maximum turn count. After the dialogue concludes, pass the full transcript to the Evaluator Agent. Compare results against the static baseline to quantify the information-gathering gap.

9. **Generate synthetic case data at scale.** Use an LLM to synthesize case profiles that match real-world demographic and categorical distributions. Validate distribution alignment using Jensen-Shannon divergence between synthetic and reference distributions for each demographic/category axis. Target thousands of cases to enable statistically meaningful comparisons.

10. **Run comparative analysis across models and strategies.** Execute all combinations of (model x interviewer-strategy x evaluation-mode) and report results in a structured table. Focus on three key comparisons: (a) static vs. dynamic accuracy gap per model; (b) strategy effectiveness ranking in dynamic mode; (c) correlation between dialogue quality scores and task accuracy (expect it to be moderate, not strong).

## Concrete Examples

**Example 1: Building a multi-agent medical consultation benchmark**

User: "I want to benchmark GPT-4 and Claude on psychiatric diagnosis using multi-turn patient interviews."

Approach:
1. Define the case schema with ICD-10 categories and patient profile fields
2. Create a Simulator Agent with a constrained patient prompt
3. Implement three Interviewer strategies (free-form, symptom-tree, RAG-phased)
4. Run both static and dynamic evaluations
5. Score with LLM-as-a-Judge on clinical quality dimensions

Output structure:
```python
# Case profile schema
case_profile = {
    "case_id": "EMR-00421",
    "age": 34,
    "gender": "female",
    "primary_diagnosis": "F32",  # Depressive Episode
    "comorbidity": "F41",        # Anxiety Disorder
    "key_symptoms": ["insomnia", "low_mood", "concentration_difficulty", "appetite_loss"],
    "duration_months": 6,
    "severity": "moderate",
    "verbosity": "low"           # Brief, reluctant responses
}

# Simulator Agent system prompt
PATIENT_SYSTEM_PROMPT = """You are role-playing a psychiatric outpatient based on the following profile.
Rules:
- Answer ONLY what the doctor asks. Do not volunteer extra information.
- Keep responses under 50 tokens. Use everyday language.
- If the profile does not specify an answer, say you are unsure or do not remember.
- Do not use clinical terminology unless the doctor uses it first.
- Show emotional tone consistent with your condition but do not exaggerate.

Profile: {case_profile_json}
"""

# Interviewer Agent (RAG-phased strategy)
DOCTOR_SYSTEM_PROMPT = """You are a psychiatrist conducting an initial consultation.
Follow this five-phase protocol:
Phase 1 (Screening, 3-5 questions): Ask about chief complaints, onset, and duration.
Phase 2 (Assessment, 5-8 questions): Assess core symptoms and functional impairment.
Phase 3 (Deep-dive, 3-6 questions): Explore specific symptom details and triggers.
Phase 4 (Risk, 2-4 questions): Screen for self-harm and safety concerns.
Phase 5 (Closure, 1-2 questions): Confirm key findings and ask if anything was missed.

Retrieved guidelines for current top candidates:
{rag_retrieved_guidelines}

After the consultation, provide your differential diagnosis with ICD-10 codes.
"""

# Evaluator Agent rubric
JUDGE_RUBRIC = {
    "clinical_appropriateness": {"scale": "1-6", "anchors": {
        1: "Questions are irrelevant or harmful",
        3: "Questions are clinically reasonable but miss key areas",
        6: "Questions systematically cover all relevant diagnostic criteria"
    }},
    "thoroughness": {"scale": "1-6", "anchors": {...}},
    "ethical_conduct": {"scale": "1-6", "anchors": {...}},
    "communication_quality": {"scale": "1-6", "anchors": {...}},
    "information_gathering_efficiency": {"scale": "1-6", "anchors": {...}},
}
```

**Example 2: Customer support triage benchmark (domain adaptation)**

User: "Adapt this multi-agent framework to benchmark LLMs on customer support ticket resolution through multi-turn chat."

Approach:
1. Replace psychiatric categories with support ticket categories (billing, technical, account, shipping)
2. Replace patient profiles with customer profiles (issue type, history, sentiment)
3. Adapt the five-phase protocol to support workflows (greeting -> problem identification -> troubleshooting -> resolution -> satisfaction check)
4. Use RAG to retrieve relevant knowledge base articles instead of clinical guidelines

Output structure:
```python
# Domain-adapted case profile
customer_profile = {
    "case_id": "TKT-8832",
    "customer_tier": "premium",
    "issue_category": "billing_dispute",
    "sub_category": "double_charge",
    "key_facts": ["charged twice on Feb 3", "amount: $49.99", "card ending 4421"],
    "sentiment": "frustrated",
    "verbosity": "medium",
    "red_herrings": ["mentions unrelated shipping delay"]
}

# Adapted phase protocol for support agent
SUPPORT_PHASES = {
    "greeting": {"turns": 1, "goal": "Establish rapport and ask for issue summary"},
    "identification": {"turns": "3-5", "goal": "Identify exact issue, account, and timeline"},
    "troubleshooting": {"turns": "3-6", "goal": "Verify details, check systems, gather evidence"},
    "resolution": {"turns": "1-3", "goal": "Propose and confirm resolution"},
    "closure": {"turns": 1, "goal": "Confirm satisfaction and offer further help"},
}

# Evaluation: task accuracy + dialogue quality
# Task accuracy: Did the agent correctly classify the issue and propose the right resolution?
# Dialogue quality: Scored on empathy, efficiency, accuracy, policy compliance
```

**Example 3: Comparing static vs. dynamic evaluation to find information-gathering gaps**

User: "My LLM scores 85% on classifying support tickets from transcripts but seems worse in live chat. How do I measure this gap?"

Approach:
1. Run static evaluation: feed 500 pre-recorded chat transcripts to the LLM and measure classification accuracy
2. Run dynamic evaluation: let the same LLM conduct 500 live chats with a Simulator Agent, then classify
3. Compare the accuracy gap to quantify the information-gathering deficit
4. Analyze which question-asking strategies close the gap

Output:
```
Static Evaluation Results (n=500):
  Binary classification:  87.2% accuracy
  4-class classification: 61.4% accuracy
  8-class classification: 38.7% accuracy

Dynamic Evaluation Results (n=500):
  Binary classification:  79.8% accuracy  (gap: -7.4%)
  4-class classification: 48.2% accuracy  (gap: -13.2%)
  8-class classification: 27.1% accuracy  (gap: -11.6%)

Strategy Comparison (dynamic, 8-class):
  Free-form:    27.1% accuracy
  Checklist:    31.4% accuracy  (+4.3%)
  RAG-phased:   35.8% accuracy  (+8.7%)

Key Insight: The static-dynamic gap widens with category count,
confirming that information-gathering quality is the bottleneck,
not reasoning ability. RAG-phased strategy recovers ~75% of the gap.
```

## Best Practices

- **Do:** Constrain the Simulator Agent's verbosity strictly. Real interviewees give terse, incomplete answers. Overly cooperative simulators inflate dynamic evaluation scores and hide information-gathering deficits.
- **Do:** Run both static and dynamic evaluation on the same test cases to produce a paired comparison. The gap between them is the most informative metric your benchmark produces.
- **Do:** Use multi-model consensus for LLM-as-a-Judge scoring (e.g., average scores from 3 different judge models) to reduce single-model bias in quality assessment.
- **Do:** Validate synthetic data distributions against real-world reference distributions using Jensen-Shannon divergence before running evaluations at scale.
- **Avoid:** Assuming high dialogue quality scores imply high task accuracy. The paper found only moderate correlation -- a model can ask well-structured questions and still reach wrong conclusions.
- **Avoid:** Testing only binary classification. Performance degrades dramatically with category count (92% -> 28% in the paper). Always include fine-grained multi-class tasks to reveal the true ceiling.

## Error Handling

- **Simulator produces out-of-character responses:** Add a validation layer that checks each simulator response against the case profile for factual consistency. If the response mentions facts not in the profile, regenerate with a reminder prompt. Score this with an "accuracy" dimension (adherence to ground truth).
- **Interviewer gets stuck in a loop:** Implement a maximum turn limit per phase and a global turn limit. If the interviewer repeats the same question category, inject a phase-transition prompt or force-advance to the next phase.
- **Judge scores are inconsistent across runs:** Use temperature=0 for judge inference, provide detailed scoring anchors for each rubric point, and require the judge to output chain-of-thought reasoning before the numeric score. Average across multiple judge models.
- **RAG retrieves irrelevant guidelines:** Use a re-ranking step after initial retrieval. Filter retrieved documents by relevance score threshold. Limit context to top-3 candidate categories to avoid overwhelming the interviewer's context window.
- **Distribution mismatch in synthetic data:** Compute per-category and per-demographic KL divergence between synthetic and reference sets. Regenerate underrepresented categories with targeted sampling until divergence drops below threshold.

## Limitations

- **Dynamic evaluation is expensive.** Each case requires a full multi-turn dialogue (15-25 LLM calls per case across simulator + interviewer + evaluator). Budget for 50-100x the cost of static evaluation per case.
- **LLM-as-a-Judge has moderate inter-rater reliability.** The paper shows dialogue quality scores correlate only moderately with task accuracy. Do not use judge scores as the sole evaluation metric -- always pair with ground-truth task accuracy.
- **Simulator fidelity is bounded by prompt engineering.** LLM-based simulators cannot perfectly replicate real human behavior (pauses, contradictions, emotional volatility). Benchmark results represent an upper bound on real-world dynamic performance.
- **Language and domain specificity.** LingxiDiagBench targets Chinese psychiatric consultation. Adapting to other languages or domains requires rebuilding case profiles, phase protocols, RAG knowledge bases, and judge rubrics from domain-specific data.
- **Comorbidity and ambiguous cases are hard.** Binary classification is a solved problem for current LLMs; the real challenge is multi-label and differential diagnosis. Design your benchmark to emphasize these hard cases.

## Reference

- **Paper:** [LingxiDiagBench: A Multi-Agent Framework for Benchmarking LLMs in Chinese Psychiatric Consultation and Diagnosis](https://arxiv.org/abs/2602.09379v2) -- Read Section 3 for the three-agent architecture and five-phase consultation protocol, Section 4 for the static vs. dynamic evaluation comparison, and Table 7 for the strategy ablation results.
- **Code & Data:** [github.com/Lingxi-mental-health/LingxiDiagBench](https://github.com/Lingxi-mental-health/LingxiDiagBench) -- See `src/doctor/` for interviewer agent implementations, `src/patient/` for simulator agents, and `evaluation/` for the judge pipeline.