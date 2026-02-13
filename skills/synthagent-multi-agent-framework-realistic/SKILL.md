---
name: "synthagent-multi-agent-framework-realistic"
description: "Build multi-agent pipelines that generate realistic synthetic patient profiles by integrating epidemiological data, medical claims, literature evidence, and personality models. Use when asked to 'simulate patients', 'generate synthetic medical data', 'build a patient simulation pipeline', 'create virtual patient cohorts', 'model disease progression with agents', or 'synthesize clinical profiles'."
---

# SynthAgent: Multi-Agent Framework for Realistic Patient Simulation

This skill enables Claude to design and implement multi-agent systems (MAS) that generate high-fidelity synthetic patient profiles. Based on the SynthAgent framework (AAAI 2026), the approach uses a five-agent sequential pipeline — Summarizer, Generator, Augmenter, Evaluator, Refiner — where each agent has a specialized role that transforms raw epidemiological and clinical data into clinically coherent, behaviorally rich virtual patients. The technique generalizes beyond healthcare: any domain requiring realistic synthetic entities built from heterogeneous data sources can use this architecture.

## When to Use

- When the user asks to generate synthetic patient records or clinical cohorts for research, testing, or simulation
- When building a multi-agent pipeline where each agent performs a distinct transformation stage (summarize, generate, augment, evaluate, refine)
- When the user needs to combine multiple heterogeneous data sources (surveys, claims data, literature, epidemiological statistics) into unified synthetic profiles
- When simulating longitudinal disease progression, treatment response, or behavioral dynamics over time
- When the user wants to enrich generated profiles with personality trait models (HEXACO, Big Five, or similar) that modulate simulated behavior
- When creating privacy-preserving synthetic datasets that maintain statistical fidelity to real populations
- When the user asks to evaluate synthetic data quality using LLM-as-a-Judge scoring across multiple dimensions

## Key Technique

SynthAgent's core insight is **staged specialization**: rather than asking a single LLM call to generate a complete synthetic patient from scratch, the work is decomposed into five sequential agents, each with a narrow mandate and access to specific data sources. The Summarizer agent matches demographic and comorbidity distributions against real population data (e.g., NHANES, BRFSS) to create a statistically grounded blueprint. The Generator agent then expands this blueprint into a full narrative profile spanning demographics, medical history, symptoms, lab values, treatments, psychological traits, and a multi-year disease timeline. The Augmenter agent queries external literature (e.g., PubMed case reports) filtered by demographic match, injecting evidence-based clinical detail that a purely statistical generator would miss.

The second critical innovation is **quality control as an agent**. The Evaluator agent audits each profile across ten dimensions (demographics, medical history, current conditions, symptoms, habits, lab values, treatments, psychological scales, role-play profile, disease timeline) using three criteria: Information Sufficiency, Logical Consistency, and Medical Plausibility. Issues are classified by severity (major, moderate, minor). The Refiner agent then performs targeted edits to resolve flagged issues while preserving narrative coherence. This evaluate-then-refine loop is what separates high-fidelity outputs from naive generation.

The third innovation is **personality-driven behavioral modeling**. Each patient is assigned scores on validated personality instruments (HEXACO six-factor model, Reinforcement Sensitivity Theory, Temperament and Character Inventory). These personality dimensions modulate simulated adherence patterns, emotional reactivity, and lifestyle decisions — producing diverse behavioral trajectories even among patients with identical clinical profiles.

## Step-by-Step Workflow

1. **Define the target population and comorbidity matrix.** Specify the condition(s) of interest, comorbidity combinations, demographic distributions (age, sex, ethnicity, socioeconomic status), and the number of profiles to generate. Structure this as a configuration object or YAML file.

2. **Curate and pre-process heterogeneous data sources.** Collect at least three complementary data types: (a) population survey data for demographic and health baselines, (b) longitudinal claims or EHR data for disease trajectories and treatment patterns, (c) epidemiological prevalence statistics for sampling priors. Standardize these into a unified schema with consistent coding (ICD-10, SNOMED, LOINC as appropriate).

3. **Implement the Summarizer agent.** This agent takes the target population spec and outputs a foundational blueprint per patient. It samples demographics from population distributions, assigns comorbidity combinations based on epidemiological prevalence, and matches against real records (survey or claims) to anchor the profile in observed data. Output: a structured JSON blueprint with demographics, condition list, and matched reference records.

4. **Implement the Generator agent.** This agent receives the blueprint and expands it into a complete patient profile. It generates: full medical history, current condition status, symptom presentation, lab values within clinically plausible ranges, treatment plan, personality trait scores (using HEXACO or equivalent framework), and a multi-year disease timeline showing progression and interventions. Output: a comprehensive patient record as structured JSON with narrative sections.

5. **Implement the Augmenter agent.** This agent enriches profiles with literature evidence. For each patient, it queries a medical literature source (PubMed API, clinical guidelines database, or a pre-built evidence corpus) filtered by the patient's demographics and conditions. It integrates relevant findings — symptom patterns, behavioral characteristics, treatment outcomes from documented cases — into the patient record. Output: the augmented patient profile with citations.

6. **Implement the Evaluator agent.** This agent audits each profile across defined quality dimensions using a structured rubric. Score each dimension on three criteria: Information Sufficiency (are required fields complete and detailed?), Logical Consistency (do elements contradict each other?), and Medical Plausibility (are values and timelines clinically realistic?). Classify any issues found as major (breaks clinical logic), moderate (implausible but not contradictory), or minor (missing optional detail). Output: a scored audit report with flagged issues.

7. **Implement the Refiner agent.** This agent receives the profile plus the audit report and performs targeted corrections. It resolves major issues first, then moderate, then minor — editing only the flagged sections while preserving the rest of the narrative. Output: the final corrected patient profile.

8. **Orchestrate the pipeline.** Wire the five agents into a sequential pipeline where each agent's output feeds the next. Implement retry logic: if the Evaluator flags major issues after refinement, loop back through Generator → Augmenter → Evaluator → Refiner (with a max iteration cap of 2-3 cycles).

9. **Evaluate cohort-level quality.** After generating the full cohort, compute diversity metrics: embed all profiles (excluding demographics) using a text embedding model, measure mean Euclidean distance from each embedding to the cohort centroid, and visualize with t-SNE. Check that patients with identical comorbidity profiles still show meaningful behavioral and clinical variation.

10. **Export and validate.** Output profiles in the downstream-required format (FHIR JSON, CSV, or structured documents). Run a final statistical validation comparing generated cohort distributions against source population data for key variables (BMI distribution, condition prevalence, age distribution, treatment patterns).

## Concrete Examples

**Example 1: Generating a synthetic obesity cohort with mental health comorbidities**

User: "I need to generate 50 synthetic patient profiles for obesity patients with comorbid depression and anxiety for a clinical decision support system test."

Approach:
1. Define target: 50 patients, primary condition obesity (BMI 30+), comorbidities depression + anxiety, demographically representative of US adult population
2. Set up data sources: NHANES public-use files for anthropometric/demographic baselines, CDC BRFSS for prevalence rates, PubMed API for case report augmentation
3. Build the five-agent pipeline:

```python
# Agent pipeline configuration
pipeline_config = {
    "target_population": {
        "n_patients": 50,
        "primary_condition": "obesity",
        "comorbidities": ["major_depressive_disorder", "generalized_anxiety_disorder"],
        "demographics": {
            "age_range": [25, 70],
            "sex_distribution": {"male": 0.42, "female": 0.58},
            "source": "NHANES_2017_2023"
        }
    },
    "agents": {
        "summarizer": {
            "role": "Create foundational blueprints by matching NHANES records and sampling from epidemiological priors",
            "data_sources": ["nhanes_processed.parquet", "cdc_brfss_prevalence.json"],
            "output_schema": "patient_blueprint.json"
        },
        "generator": {
            "role": "Expand blueprints into full profiles with medical history, labs, treatments, personality traits, and disease timelines",
            "personality_model": "HEXACO",
            "timeline_years": 5
        },
        "augmenter": {
            "role": "Enrich profiles with PubMed case report evidence filtered by patient demographics",
            "api": "pubmed_efetch",
            "max_articles_per_patient": 5
        },
        "evaluator": {
            "role": "Audit profiles across 10 dimensions with severity-classified issue flagging",
            "dimensions": ["demographics", "medical_history", "conditions", "symptoms",
                          "habits", "lab_values", "treatments", "psychological_scales",
                          "behavioral_profile", "disease_timeline"],
            "criteria": ["information_sufficiency", "logical_consistency", "medical_plausibility"]
        },
        "refiner": {
            "role": "Resolve flagged issues by severity priority while preserving narrative coherence"
        }
    }
}
```

4. Run pipeline, generating each patient through all five stages
5. Evaluate diversity: embed profiles, compute centroid distances, visualize with t-SNE
6. Export as FHIR-compatible JSON bundles

Output: 50 clinically coherent patient records, each containing demographics, 5-year medical timeline, current lab values, treatment plan, HEXACO personality scores, and behavioral adherence patterns — with cohort-level statistics matching real-world obesity+depression+anxiety prevalence ratios.

**Example 2: Building a reusable multi-agent synthetic data pipeline**

User: "Help me architect a multi-agent system for generating synthetic clinical trial participants. I want it to be modular so I can swap data sources and target different conditions."

Approach:
1. Design abstract agent interfaces following the SynthAgent five-stage pattern
2. Implement each agent as a class with pluggable data source connectors

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class PatientBlueprint:
    patient_id: str
    demographics: dict
    conditions: list[str]
    matched_references: list[dict]

@dataclass
class PatientProfile:
    blueprint: PatientBlueprint
    medical_history: list[dict]
    current_conditions: list[dict]
    symptoms: list[dict]
    lab_values: dict
    treatments: list[dict]
    personality_traits: dict  # HEXACO scores
    disease_timeline: list[dict]
    literature_evidence: list[dict]
    audit_report: dict | None = None

class SynthAgentBase(ABC):
    @abstractmethod
    def process(self, input_data) -> any:
        pass

class SummarizerAgent(SynthAgentBase):
    """Matches demographics and comorbidities against population data to create blueprints."""
    def __init__(self, population_data_source, prevalence_data_source):
        self.pop_data = population_data_source
        self.prev_data = prevalence_data_source

    def process(self, target_spec: dict) -> PatientBlueprint:
        # 1. Sample demographics from population distributions
        # 2. Assign comorbidities based on prevalence priors
        # 3. Match against real records for anchoring
        ...

class GeneratorAgent(SynthAgentBase):
    """Expands blueprints into full patient profiles with personality and timelines."""
    def __init__(self, llm_client, personality_model="HEXACO"):
        self.llm = llm_client
        self.personality_model = personality_model

    def process(self, blueprint: PatientBlueprint) -> PatientProfile:
        # Prompt the LLM with blueprint + personality framework
        # to generate a complete, coherent patient narrative
        ...

class AugmenterAgent(SynthAgentBase):
    """Enriches profiles with literature-sourced clinical evidence."""
    def __init__(self, literature_api, max_articles=5):
        self.lit_api = literature_api
        self.max_articles = max_articles

    def process(self, profile: PatientProfile) -> PatientProfile:
        # Query literature filtered by patient demographics and conditions
        # Integrate relevant findings into profile
        ...

class EvaluatorAgent(SynthAgentBase):
    """Audits profiles for sufficiency, consistency, and plausibility."""
    DIMENSIONS = ["demographics", "medical_history", "conditions", "symptoms",
                  "habits", "lab_values", "treatments", "psych_scales",
                  "behavioral_profile", "disease_timeline"]

    def process(self, profile: PatientProfile) -> dict:
        # Score each dimension on 3 criteria
        # Classify issues as major/moderate/minor
        return {"scores": {...}, "issues": [...], "overall": float}

class RefinerAgent(SynthAgentBase):
    """Resolves flagged issues by severity priority."""
    def process(self, profile: PatientProfile) -> PatientProfile:
        # Fix major issues first, then moderate, then minor
        # Preserve unaffected narrative sections
        ...

class SynthAgentPipeline:
    def __init__(self, summarizer, generator, augmenter, evaluator, refiner, max_refine_cycles=2):
        self.agents = [summarizer, generator, augmenter, evaluator, refiner]
        self.max_cycles = max_refine_cycles

    def generate_patient(self, target_spec: dict) -> PatientProfile:
        blueprint = self.agents[0].process(target_spec)
        profile = self.agents[1].process(blueprint)
        profile = self.agents[2].process(profile)
        for _ in range(self.max_cycles):
            audit = self.agents[3].process(profile)
            if not audit["issues"]:
                break
            profile.audit_report = audit
            profile = self.agents[4].process(profile)
        return profile
```

3. This modular design lets users swap `population_data_source` for different survey datasets, change the `literature_api` to different knowledge bases, and target any condition by modifying the `target_spec`.

**Example 3: Adding personality-driven behavioral simulation**

User: "I have generated patient profiles but they all behave the same way in our treatment adherence simulation. How do I make them more diverse?"

Approach:
1. Assign validated personality scores to each patient using the HEXACO model
2. Map personality dimensions to behavioral modifiers

```python
HEXACO_BEHAVIOR_MAP = {
    "conscientiousness": {
        "high": {"medication_adherence_modifier": 1.3, "appointment_attendance": 0.95},
        "low":  {"medication_adherence_modifier": 0.6, "appointment_attendance": 0.55}
    },
    "emotionality": {
        "high": {"stress_eating_risk": 1.5, "anxiety_flare_frequency": 1.4},
        "low":  {"stress_eating_risk": 0.7, "anxiety_flare_frequency": 0.6}
    },
    "extraversion": {
        "high": {"social_support_seeking": 1.4, "group_therapy_engagement": 1.3},
        "low":  {"social_support_seeking": 0.5, "group_therapy_engagement": 0.4}
    },
    "agreeableness": {
        "high": {"provider_trust": 1.3, "treatment_plan_compliance": 1.2},
        "low":  {"provider_trust": 0.7, "treatment_plan_compliance": 0.6}
    },
    "honesty_humility": {
        "high": {"self_report_accuracy": 1.4, "symptom_disclosure": 1.3},
        "low":  {"self_report_accuracy": 0.6, "symptom_disclosure": 0.5}
    },
    "openness": {
        "high": {"alternative_therapy_receptivity": 1.4, "lifestyle_change_adoption": 1.2},
        "low":  {"alternative_therapy_receptivity": 0.5, "lifestyle_change_adoption": 0.6}
    }
}

def apply_personality_modifiers(patient_profile, base_behavior):
    """Modulate behavioral simulation parameters based on HEXACO personality scores."""
    modified = base_behavior.copy()
    for dimension, score in patient_profile["personality_traits"]["hexaco"].items():
        level = "high" if score > 3.5 else "low"
        if dimension in HEXACO_BEHAVIOR_MAP:
            for behavior, modifier in HEXACO_BEHAVIOR_MAP[dimension][level].items():
                if behavior in modified:
                    modified[behavior] *= modifier
    return modified
```

3. This produces meaningfully different behavioral trajectories: a low-conscientiousness, high-emotionality patient will show poor medication adherence and frequent stress-eating episodes, while a high-conscientiousness, low-emotionality patient will follow treatment plans consistently — even when both share the same clinical diagnosis.

## Best Practices

- **Do:** Anchor synthetic profiles in real population statistics. Always ground the Summarizer agent's outputs in actual survey data (NHANES, BRFSS, or equivalent) to ensure demographic and prevalence distributions are realistic.
- **Do:** Keep agents narrowly specialized. Each agent should have one clear job. Combining generation and evaluation into a single agent degrades both tasks.
- **Do:** Classify evaluation issues by severity and resolve in priority order. Major issues (clinical contradictions) must be fixed before moderate issues (implausible values) which come before minor issues (missing optional detail).
- **Do:** Measure cohort-level diversity, not just individual quality. Use embedding-based distance metrics to verify that generated patients show meaningful variation even within the same comorbidity group.
- **Avoid:** Generating all profile fields in a single LLM call. The staged pipeline exists because no single prompt can simultaneously ensure statistical grounding, literature accuracy, and internal consistency.
- **Avoid:** Skipping the Augmenter stage. Literature evidence integration is what separates clinically authentic profiles from statistically plausible but shallow ones.
- **Avoid:** Running more than 2-3 evaluate-refine cycles. Diminishing returns set in quickly, and excessive refinement can introduce new inconsistencies.

## Error Handling

- **Contradictory lab values and diagnoses:** If the Evaluator flags lab values inconsistent with stated conditions (e.g., normal HbA1c with diagnosed uncontrolled diabetes), the Refiner should adjust lab values to plausible ranges for the condition severity, not remove the diagnosis.
- **Personality-behavior mismatches:** If personality trait scores produce behavioral modifiers that contradict the clinical timeline (e.g., high conscientiousness but documented non-adherence), resolve by either adjusting the personality score or adding a contextual explanation (life stressor, job loss) that accounts for the deviation.
- **Literature API failures:** If PubMed or the literature source is unavailable, the Augmenter should degrade gracefully — flag the profile as "un-augmented" and proceed. Do not block the pipeline.
- **Demographic distribution drift:** After generating the full cohort, if statistical validation shows the cohort diverges from target distributions (e.g., age skew), re-generate underrepresented demographic segments rather than post-hoc reweighting.
- **Evaluation score below threshold:** If a profile scores below a defined quality threshold (e.g., mean < 60/100) after max refine cycles, discard and regenerate from a new blueprint rather than forcing fixes on a fundamentally flawed profile.

## Limitations

- This framework is designed for **synthetic data generation for research and testing**, not for clinical decision-making. Generated profiles should never be treated as real patient records.
- The quality of output is bounded by the quality and representativeness of input data sources. Biases in NHANES or claims data will propagate into synthetic profiles.
- Personality trait modeling uses population-level heuristics mapping traits to behaviors. Individual human behavior is far more context-dependent and variable than these modifiers capture.
- The five-agent pipeline introduces latency and cost: each patient passes through five LLM calls minimum. For large cohorts (1000+ patients), budget for significant API costs and implement batching.
- Disease progression timelines are narrative constructions, not epidemiologically calibrated survival models. For precise outcome modeling, use dedicated simulation tools (microsimulation, discrete event simulation) calibrated to clinical trial data.
- The evaluate-refine loop improves internal consistency but cannot detect subtle clinical errors that fall outside the LLM evaluator's medical knowledge.

## Reference

**Paper:** [SynthAgent: A Multi-Agent LLM Framework for Realistic Patient Simulation](https://arxiv.org/abs/2602.08254v1) — Aghaee, Asgarian, Jeon (AAAI 2026 Workshop on Health Intelligence). Key sections to study: the five-agent architecture design (Summarizer/Generator/Augmenter/Evaluator/Refiner), the HEXACO personality integration approach, the 10-dimension quality evaluation rubric, and the embedding-based diversity analysis methodology.