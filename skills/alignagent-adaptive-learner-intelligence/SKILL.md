---
name: "alignagent-adaptive-learner-intelligence"
description: |
  Build multi-agent adaptive learning systems that diagnose knowledge gaps and recommend targeted resources.
  Implements the ALIGNAgent framework: Skill Gap Agent (proficiency estimation + concept-level diagnostic reasoning)
  and Recommender Agent (preference-aware resource retrieval aligned to deficiencies).
  Trigger phrases:
  - "Build an adaptive learning system"
  - "Create a personalized tutoring agent"
  - "Diagnose student knowledge gaps from quiz data"
  - "Build a skill gap analyzer for learners"
  - "Create an educational recommender that adapts to student performance"
  - "Implement a multi-agent pipeline for personalized education"
---

# ALIGNAgent: Adaptive Learner Intelligence for Gap Identification and Next-step Guidance

This skill enables Claude to build multi-agent educational systems that close the loop between assessment, diagnosis, and intervention. Based on the ALIGNAgent framework, the approach processes learner performance data (quiz scores, gradebook records, preferences) through a Skill Gap Agent that computes topic-level proficiency scores and performs concept-level diagnostic reasoning to identify specific misconceptions, then routes deficiencies to a Recommender Agent that retrieves preference-aware learning materials. The key innovation is the continuous feedback loop: interventions occur *before* advancing to subsequent topics, creating an iterative remediation cycle rather than a one-shot recommendation.

## When to Use

- When building a personalized learning platform that needs to diagnose what a student doesn't know and why
- When the user wants to analyze quiz or assessment data to compute per-topic proficiency scores
- When creating a tutoring system that adapts resource recommendations to individual learning gaps
- When implementing a multi-agent pipeline where one agent diagnoses and another recommends
- When the user needs to classify learner proficiency (High/Medium/Low) from gradebook data and validate against exam performance
- When designing formative assessment cycles that loop diagnosis into intervention before progression

## Key Technique

**Two-Agent Architecture with Continuous Feedback Loop.** ALIGNAgent separates concerns into a Skill Gap Agent and a Recommender Agent connected by a structured handoff. The Skill Gap Agent computes topic-level proficiency as `rho_t = (1/|I_t|) * sum(g_i for i in I_t)` where `I_t` is the set of assessment items for topic `t` and `g_i` is the normalized score on item `i`. It then applies a mastery threshold `tau` to produce a gap set `G(s) = {(t, rho_t) | rho_t < tau}`, sorted by ascending proficiency so the weakest areas get attention first. Beyond this quantitative layer, the agent performs concept-level diagnostic reasoning -- analyzing patterns in incorrect answers, distractor selection frequency, and cross-item reasoning to produce natural-language explanations of *why* a learner is struggling (e.g., "confuses AVL rotation direction" rather than just "low score on AVL trees").

**Preference-Aware Resource Retrieval.** The Recommender Agent constructs structured queries from the gap set and learner preferences (video vs. text, self-paced vs. structured), retrieves candidate resources, validates them against modality preferences and topic relevance, and returns up to K resources per gap. This is diagnostic-driven retrieval -- the query is shaped by the specific misconception, not just the broad topic.

**Formative Loop, Not One-Shot.** The critical design principle is that the pipeline re-executes after intervention. New assessment data updates the proficiency vector, and topics that remain below threshold trigger fresh recommendations. This makes ALIGNAgent a cycle, not a pipeline -- each iteration refines the learner model.

## Step-by-Step Workflow

1. **Define the domain schema.** Create a topic taxonomy for your subject area as a structured map: `{topic_id: {name, prerequisite_topics[], concept_tags[]}}`. Each assessment item must be tagged with its topic and concept(s).

2. **Ingest learner data.** Parse three input streams into a unified learner profile: (a) gradebook records with per-item scores normalized to [0, 1], (b) quiz/assessment responses including selected answers (not just correctness), and (c) learner preferences (modality, pacing, prior stated goals).

3. **Compute topic-level proficiency.** For each topic `t`, calculate `rho_t = mean(normalized_scores for items tagged with t)`. Assemble the full proficiency vector `rho(s) = [rho_t1, rho_t2, ..., rho_tn]`.

4. **Classify proficiency levels.** Map each `rho_t` to a categorical label using thresholds: High (`rho_t >= 0.8`), Medium (`0.5 <= rho_t < 0.8`), Low (`rho_t < 0.5`). These thresholds should be configurable.

5. **Identify skill gaps.** Apply mastery threshold `tau` (default: 0.7) to produce the gap set `G(s) = {(t, rho_t) | rho_t < tau}`. Sort gaps by ascending `rho_t` so the weakest topics are prioritized.

6. **Run concept-level diagnostic reasoning.** For each topic in the gap set, analyze the learner's incorrect responses: which distractors were selected, which concepts those distractors test, and whether errors cluster around a specific misconception. Produce a natural-language diagnostic per gap (e.g., "Consistently confuses DFS with BFS traversal order; correctly identifies base cases but fails on recursive step for graph cycles").

7. **Construct retrieval queries.** For each diagnosed gap, build a structured query combining: topic name, specific misconception description, and learner modality preference. Example: `{topic: "AVL Trees", misconception: "rotation direction after double rotation", preference: "video tutorial"}`.

8. **Retrieve and filter resources.** Search for candidate resources matching each query. Filter by: (a) topic relevance to the specific misconception, (b) modality alignment with learner preferences, (c) content quality signals. Return up to K resources per gap, ranked by alignment score.

9. **Generate learner summary.** Produce a structured output containing: strengths (topics with High proficiency), gaps (topics below threshold with diagnostic explanations), recommended actions (ordered resource list per gap), and next assessment targets.

10. **Close the feedback loop.** After the learner engages with resources and completes a new assessment, re-run steps 3-9 with updated data. Track proficiency delta per topic to measure intervention effectiveness and adjust recommendations accordingly.

## Concrete Examples

**Example 1: Analyzing quiz performance in a Python programming course**

User: "I have quiz data for 30 students across 5 Python topics. Help me build a system that identifies each student's weak areas and recommends resources."

Approach:
1. Parse the quiz CSV with columns: `student_id, quiz_id, topic, question_id, score, max_score`
2. Compute per-student, per-topic proficiency scores
3. Apply threshold to identify gaps
4. Generate diagnostic reasoning per gap
5. Match resources to gaps and preferences

Output structure:
```json
{
  "student_id": "S042",
  "proficiency_vector": {
    "variables_types": 0.92,
    "control_flow": 0.78,
    "functions": 0.45,
    "data_structures": 0.33,
    "file_io": 0.60
  },
  "proficiency_labels": {
    "variables_types": "High",
    "control_flow": "Medium",
    "functions": "Low",
    "data_structures": "Low",
    "file_io": "Medium"
  },
  "skill_gaps": [
    {
      "topic": "data_structures",
      "proficiency": 0.33,
      "diagnostic": "Correctly uses lists but fails when indexing nested structures. Confuses dict key access with list indexing. Missed all questions involving dictionary comprehensions.",
      "resources": [
        {"title": "Python Data Structures - Nested Collections", "type": "video", "url": "..."},
        {"title": "Dictionary Comprehension Practice Problems", "type": "interactive", "url": "..."}
      ]
    },
    {
      "topic": "functions",
      "proficiency": 0.45,
      "diagnostic": "Understands function definition syntax but struggles with return vs print distinction. Errors cluster around scope-related questions -- treats local variables as global.",
      "resources": [
        {"title": "Python Scoping Rules Explained", "type": "article", "url": "..."},
        {"title": "Functions and Return Values - Step by Step", "type": "video", "url": "..."}
      ]
    }
  ],
  "strengths": ["variables_types"],
  "next_assessment": ["data_structures", "functions"]
}
```

**Example 2: Building the multi-agent system in code**

User: "Implement the ALIGNAgent two-agent pipeline as a Python module."

Approach:
1. Define data models for learner profiles, assessments, and proficiency vectors
2. Implement the Skill Gap Agent as a class with proficiency computation and LLM-based diagnostic reasoning
3. Implement the Recommender Agent with query construction and resource retrieval
4. Wire them together in a pipeline that accepts assessment data and returns a learner summary

```python
from dataclasses import dataclass, field
from typing import Optional
import json

@dataclass
class AssessmentItem:
    question_id: str
    topic: str
    concepts: list[str]
    score: float          # normalized to [0, 1]
    selected_answer: str  # the actual answer chosen
    correct_answer: str

@dataclass
class LearnerProfile:
    student_id: str
    preferences: dict     # e.g., {"modality": "video", "pacing": "self-paced"}
    assessments: list[AssessmentItem] = field(default_factory=list)

@dataclass
class SkillGap:
    topic: str
    proficiency: float
    diagnostic: str       # natural-language explanation
    misconceptions: list[str]

class SkillGapAgent:
    def __init__(self, mastery_threshold: float = 0.7, llm_client=None):
        self.tau = mastery_threshold
        self.llm = llm_client

    def compute_proficiency(self, profile: LearnerProfile) -> dict[str, float]:
        topic_scores: dict[str, list[float]] = {}
        for item in profile.assessments:
            topic_scores.setdefault(item.topic, []).append(item.score)
        return {t: sum(s) / len(s) for t, s in topic_scores.items()}

    def identify_gaps(self, proficiency: dict[str, float]) -> list[tuple[str, float]]:
        gaps = [(t, rho) for t, rho in proficiency.items() if rho < self.tau]
        return sorted(gaps, key=lambda x: x[1])  # weakest first

    def diagnose(self, profile: LearnerProfile, gaps: list[tuple[str, float]]) -> list[SkillGap]:
        diagnosed = []
        for topic, rho in gaps:
            incorrect = [a for a in profile.assessments
                         if a.topic == topic and a.score < 1.0]
            # Use LLM for concept-level diagnostic reasoning
            prompt = self._build_diagnostic_prompt(topic, incorrect)
            reasoning = self.llm.generate(prompt, temperature=0)
            diagnosed.append(SkillGap(
                topic=topic, proficiency=rho,
                diagnostic=reasoning["explanation"],
                misconceptions=reasoning["misconceptions"]
            ))
        return diagnosed

    def _build_diagnostic_prompt(self, topic: str, incorrect_items: list) -> str:
        items_desc = json.dumps([{
            "question_id": i.question_id,
            "concepts": i.concepts,
            "selected": i.selected_answer,
            "correct": i.correct_answer
        } for i in incorrect_items], indent=2)
        return f"""Analyze these incorrect responses for topic "{topic}".
Identify specific misconceptions by examining which distractors were chosen and what conceptual errors they reveal.

Incorrect responses:
{items_desc}

Return JSON with:
- "explanation": one paragraph describing the pattern of errors
- "misconceptions": list of specific misconception strings"""


class RecommenderAgent:
    def __init__(self, max_resources_per_gap: int = 3, search_fn=None):
        self.k = max_resources_per_gap
        self.search = search_fn

    def recommend(self, gaps: list[SkillGap], preferences: dict) -> dict[str, list]:
        recommendations = {}
        for gap in gaps:
            query = self._build_query(gap, preferences)
            candidates = self.search(query)
            filtered = [r for r in candidates
                        if self._matches_preference(r, preferences)]
            recommendations[gap.topic] = filtered[:self.k]
        return recommendations

    def _build_query(self, gap: SkillGap, preferences: dict) -> str:
        modality = preferences.get("modality", "any")
        return f"{gap.topic} {gap.misconceptions[0]} {modality} tutorial"

    def _matches_preference(self, resource: dict, preferences: dict) -> bool:
        modality = preferences.get("modality")
        if not modality or modality == "any":
            return True
        return resource.get("type", "").lower() == modality.lower()


class ALIGNAgentPipeline:
    def __init__(self, mastery_threshold=0.7, max_resources=3,
                 llm_client=None, search_fn=None):
        self.skill_gap_agent = SkillGapAgent(mastery_threshold, llm_client)
        self.recommender_agent = RecommenderAgent(max_resources, search_fn)

    def run(self, profile: LearnerProfile) -> dict:
        proficiency = self.skill_gap_agent.compute_proficiency(profile)
        gaps = self.skill_gap_agent.identify_gaps(proficiency)
        diagnosed = self.skill_gap_agent.diagnose(profile, gaps)
        resources = self.recommender_agent.recommend(diagnosed, profile.preferences)

        return {
            "student_id": profile.student_id,
            "proficiency": proficiency,
            "strengths": [t for t, rho in proficiency.items() if rho >= 0.8],
            "gaps": [{
                "topic": g.topic,
                "proficiency": g.proficiency,
                "diagnostic": g.diagnostic,
                "misconceptions": g.misconceptions,
                "resources": resources.get(g.topic, [])
            } for g in diagnosed],
            "next_assessment_targets": [g.topic for g in diagnosed]
        }
```

**Example 3: Retrofitting an existing quiz app with adaptive diagnostics**

User: "I have a Flask quiz app that stores results in PostgreSQL. Add adaptive gap analysis after each quiz submission."

Approach:
1. Read the existing quiz submission endpoint and database schema
2. Add a post-submission hook that triggers the Skill Gap Agent
3. Store proficiency vectors in a new `learner_proficiency` table
4. Add a `/api/student/<id>/gaps` endpoint that returns diagnosed gaps with recommendations
5. Implement proficiency delta tracking across quiz attempts

```python
# Added to existing Flask app -- new endpoint
@app.route("/api/student/<student_id>/gaps", methods=["GET"])
def get_student_gaps(student_id):
    # Fetch assessment history from DB
    assessments = db.session.query(QuizResponse).filter_by(
        student_id=student_id
    ).all()

    profile = LearnerProfile(
        student_id=student_id,
        preferences=get_learner_preferences(student_id),
        assessments=[to_assessment_item(a) for a in assessments]
    )

    pipeline = ALIGNAgentPipeline(
        mastery_threshold=0.7,
        llm_client=openai_client,
        search_fn=search_educational_resources
    )
    result = pipeline.run(profile)

    # Store updated proficiency for delta tracking
    store_proficiency_snapshot(student_id, result["proficiency"])

    return jsonify(result)
```

## Best Practices

- **Do:** Tag every assessment item with both a topic and fine-grained concept labels. The diagnostic reasoning quality depends entirely on concept-level metadata -- without it, you get "low score on Topic X" instead of actionable misconception identification.
- **Do:** Set LLM temperature to 0 for proficiency estimation and diagnostic reasoning. The paper found deterministic outputs produce more consistent and reproducible classifications.
- **Do:** Sort gaps by ascending proficiency and address them in order. The weakest areas benefit most from early intervention, and prerequisite topics often gate understanding of downstream topics.
- **Do:** Track proficiency deltas across assessment cycles. A gap that persists after intervention signals that the recommended resources were ineffective and the diagnostic may need revision.
- **Avoid:** Using only aggregate scores without analyzing selected answers. The concept-level diagnostic reasoning requires knowing *which wrong answer* was chosen, not just that the answer was wrong.
- **Avoid:** Recommending resources based solely on topic match. The Recommender Agent must incorporate the specific misconception and the learner's modality preference -- a generic "learn about trees" video won't address a specific AVL rotation confusion.

## Error Handling

- **Sparse assessment data:** When a topic has fewer than 3 items, proficiency estimates are unreliable. Flag these with a confidence indicator and require additional assessment before diagnosing specific misconceptions.
- **LLM diagnostic hallucination:** The diagnostic reasoning agent may invent misconceptions not supported by the response data. Validate diagnostics by checking that cited concepts actually appear in the incorrect items' concept tags.
- **Broken or irrelevant resources:** Web-retrieved resources may be dead links or off-topic. Implement URL validation and content-relevance checking before presenting resources to learners.
- **Threshold sensitivity:** A mastery threshold that is too high floods learners with gaps; too low misses real deficiencies. Start with `tau=0.7` and calibrate by comparing against held-out exam performance.
- **Cold start problem:** New learners with no assessment history cannot be profiled. Provide a brief diagnostic pre-assessment or default to the most common misconceptions for the cohort.

## Limitations

- The framework requires assessment items tagged with topic and concept metadata. Unstructured or untagged quiz data cannot be processed without a manual or LLM-assisted tagging step.
- Validated only on small cohorts (11-14 students) in undergraduate CS courses. Effectiveness in other domains, age groups, or at scale is unverified.
- Resource recommendation quality depends on the availability of modality-specific materials for niche misconceptions. Obscure topics may yield poor or no matches.
- The proficiency formula uses simple averaging, which doesn't account for question difficulty, recency of attempts, or item discrimination. More sophisticated models (IRT, BKT) may be needed for high-stakes applications.
- Concept-level diagnostic reasoning via LLMs can be inconsistent across models. GPT-4o significantly outperformed other models in the paper's evaluation.

## Reference

[ALIGNAgent: Adaptive Learner Intelligence for Gap Identification and Next-step guidance](https://arxiv.org/abs/2601.15551v1) -- Focus on Section 3 (Framework Architecture), Algorithm 1 (Skill Gap identification), Algorithm 2 (Resource Recommendation), and Tables 1-2 (evaluation results showing 0.87-0.90 precision with GPT-4o agents).