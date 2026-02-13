---
name: "isd-agent-bench-comprehensive-benchmark-evaluating"
description: "Build and evaluate LLM-based Instructional Design agents using the ADDIE framework, Context Matrix scenario generation, and multi-judge evaluation. Triggers: 'design a course using ADDIE', 'build an instructional design agent', 'evaluate my ISD pipeline', 'create training program with learning objectives', 'benchmark educational content generation', 'generate instructional scenarios with context variables'"
---

# ISD-Agent-Bench: Building Theory-Grounded Instructional Design Agents

This skill enables Claude to design, build, and evaluate LLM-based agents that automate Instructional Systems Design (ISD) using the ADDIE framework (Analysis, Design, Development, Implementation, Evaluation). The core insight from ISD-Agent-Bench is that integrating classical ISD theories with ReAct-style reasoning — specifically using coarse-grained phase-level tools rather than fine-grained sub-step tools — achieves the highest performance (86.49 vs. 82.96 for fine-grained). This skill teaches how to construct Context Matrix scenarios for systematic coverage, implement theory-grounded ISD agents, and apply multi-judge evaluation protocols to eliminate LLM-as-judge bias.

## When to Use

- When the user asks to build an agent or pipeline that generates course curricula, training programs, or lesson plans
- When designing educational content that must systematically cover analysis, design, development, implementation, and evaluation phases
- When the user wants to generate diverse instructional scenarios by crossing learner characteristics, domains, delivery modes, and constraints
- When evaluating the quality of LLM-generated educational materials using structured rubrics
- When building a ReAct-style agent that needs to follow a multi-phase structured methodology (not limited to education — the pattern generalizes)
- When the user needs to set up multi-judge LLM evaluation with inter-rater reliability checks

## Key Technique

**Context Matrix Scenario Generation.** The paper's Context Matrix framework crosses 51 contextual variables across 5 categories (Learner Characteristics, Institutional Context, Educational Domain, Delivery Mode, Constraints) with 33 ISD sub-steps derived from ADDIE. This produces combinatorially diverse scenarios that stress-test an agent's ability to adapt instructional design to specific contexts. Scenarios are generated via a five-stage pipeline: seed collection from domain literature, stratified sampling of variable combinations, LLM-based content generation, two-layer quality control (rule-based field validation + LLM consistency review), and targeted augmentation of underrepresented categories.

**React-ADDIE: Coarse-Grained Theory + ReAct Reasoning.** The top-performing architecture (React-ADDIE) wraps each ADDIE phase as a single tool call rather than decomposing into 14 fine-grained sub-step tools. At each phase, the agent generates a reasoning thought, executes the phase tool, and observes the result — standard ReAct loop. The key finding: coarse-grained decomposition (5 tools) outperforms fine-grained (14 tools) because it lets the LLM handle intra-phase reasoning natively rather than forcing rigid sub-step sequences. This principle — match tool granularity to the LLM's natural reasoning span — applies broadly to agent design.

**Multi-Judge Evaluation Protocol.** To avoid single-model bias, evaluation uses median aggregation across judges from different providers (e.g., GPT-4o, Gemini, Solar). A two-stage scoring process first determines categorical status (absent/weak/moderate/good/excellent), then assigns bounded numerical scores within that category's range. This achieves 0.905 mean inter-judge reliability with only ±0.06 systematic bias.

## Step-by-Step Workflow

### Building an ISD Agent

1. **Define the ADDIE phase structure.** Create 5 phase-level tool functions: `analyze(scenario)`, `design(analysis_output)`, `develop(design_output)`, `implement(development_output)`, `evaluate(implementation_output)`. Each tool takes the previous phase's output as context. Do NOT decompose into 14+ sub-step tools — coarse-grained outperforms fine-grained.

2. **Encode the 33 sub-steps as phase prompts, not separate tools.** Within each phase tool's system prompt, enumerate the sub-steps that phase must address:
   - **Analysis (10):** Problem identification, gap analysis, performance analysis, needs prioritization, learner analysis, context analysis, initial learning goals, subordinate skills identification, entry behaviors, task review
   - **Design (8):** Objectives refinement (SMART format), assessment plan, content selection, instructional strategy, non-instructional strategy, media selection, activities & time allocation, storyboard design
   - **Development (5):** Learner materials, instructor manual, operator manual, assessment tools, expert review plan
   - **Implementation (4):** Instructor orientation, system check, prototype execution, operations monitoring plan
   - **Evaluation (6):** Pilot data collection plan, formative improvements, summative assessment, effectiveness analysis, adoption decision criteria, program improvement recommendations

3. **Implement the ReAct loop for each phase.** For each phase p, the agent: (a) generates a thought about what this phase requires given the scenario context, (b) calls the phase tool with the scenario and all prior phase outputs as history, (c) observes the result and appends it to the trajectory history before moving to the next phase.

4. **Construct the scenario input using Context Matrix variables.** Every scenario must specify: learner age/education/expertise/role, institutional type, educational domain, delivery mode, class size, duration, assessment type, technology availability, and budget level. Use SMART-format learning goals.

5. **Chain phase outputs sequentially.** Pass the full accumulated history to each phase: `h_p = h_(p-1) ∪ {(thought_p, tool_p, output_p)}`. This ensures later phases reference earlier decisions (e.g., evaluation references the objectives set during design).

6. **Apply theory-specific alignment checks.** After generation, verify: (a) assessments directly derive from stated objectives (Dick & Carey alignment principle), (b) the design is problem-centered with real-world task anchoring (Merrill's First Principles), (c) content-activity pairings match the demonstration-application pattern.

### Evaluating ISD Agent Output

7. **Score using the ADDIE rubric (70% weight).** Rate each of the 33 sub-steps on a 0-10 scale across 5 levels: Absent [0-2], Poor [3-4], Satisfactory [5-6], Good [7-8], Excellent [9-10]. Aggregate to 13 items with phase weights: Analysis 25%, Design 25%, Development 20%, Implementation 15%, Evaluation 15%.

8. **Score the agent trajectory (30% weight).** Evaluate: tool correctness (did it pick the right tools?), argument accuracy (correct parameters?), redundancy avoidance (no unnecessary calls?), result utilization (did it use tool outputs effectively?). Each criterion is 25 points.

9. **Use multi-judge median aggregation.** Run evaluation through 3+ LLM judges from different providers. Apply two-stage scoring: first categorical status, then bounded numerical score. Take the median across judges as the final score. Verify inter-judge reliability exceeds 0.75 (good) for at least 90% of items.

10. **Compute difficulty-stratified results.** Weight scenario difficulty by: learning goals complexity (0.25), learner expertise (0.25), resource constraints (0.20), duration (0.20), budget (0.10). Report performance separately for Easy/Medium/Hard terciles to check robustness.

## Concrete Examples

**Example 1: Building a Course Design Agent**

User: "Build me an agent that generates complete training programs for corporate onboarding."

Approach:
1. Define the scenario context variables: Learner (age: 20s-30s, education: bachelor+, expertise: beginner in company processes, role: office worker), Institution (corporate), Domain (business), Delivery (blended), Class size (medium 10-30), Duration (mid 2-4 weeks), Assessment (formative + project-based), Tech (provided), Budget (medium).

2. Implement the 5-phase React-ADDIE pipeline:

```python
class ReactADDIEAgent:
    def __init__(self, llm_client):
        self.llm = llm_client
        self.history = []

    def run_phase(self, phase_name, phase_prompt, scenario, prior_outputs):
        thought = self.llm.generate(
            f"Think about what {phase_name} requires for this scenario:\n"
            f"{scenario}\n\nPrior phases:\n{prior_outputs}"
        )
        result = self.llm.generate(
            f"{phase_prompt}\n\nScenario: {scenario}\n"
            f"Your reasoning: {thought}\nPrior outputs: {prior_outputs}\n"
            f"Produce the complete {phase_name} phase output."
        )
        self.history.append({
            "phase": phase_name,
            "thought": thought,
            "output": result
        })
        return result

    def generate_program(self, scenario):
        phases = [
            ("Analysis", ANALYSIS_PROMPT),   # covers 10 sub-steps
            ("Design", DESIGN_PROMPT),       # covers 8 sub-steps
            ("Development", DEVELOPMENT_PROMPT),  # covers 5 sub-steps
            ("Implementation", IMPLEMENTATION_PROMPT),  # covers 4 sub-steps
            ("Evaluation", EVALUATION_PROMPT),  # covers 6 sub-steps
        ]
        outputs = {}
        for name, prompt in phases:
            outputs[name] = self.run_phase(name, prompt, scenario, outputs)
        return outputs
```

3. Each phase prompt embeds its sub-steps as a checklist the LLM must address. For example, the Analysis prompt includes: "You must address: (1) Problem identification — what performance gap exists? (2) Gap analysis — current vs. desired state..."

Output: A complete training program document with 5 sections, each covering all sub-steps, with assessments aligned to objectives and content matched to learner context.

**Example 2: Generating Diverse Test Scenarios via Context Matrix**

User: "I need 100 diverse instructional design scenarios to test my educational content generator."

Approach:
1. Define the Context Matrix with the 5 categories and their variables:

```python
CONTEXT_MATRIX = {
    "learner": {
        "age": ["teens", "20s", "30s", "40+"],
        "education": ["high_school", "bachelor", "master", "adult_learner"],
        "expertise": ["beginner", "intermediate", "advanced"],
        "role": ["student", "professional", "teacher"]
    },
    "institution": ["k12", "university", "corporate", "vocational", "public_nonprofit"],
    "domain": ["language", "math", "science", "IT", "AI", "medical", "business"],
    "delivery": ["classroom", "online_sync", "online_async", "blended", "mobile", "VR"],
    "constraints": {
        "class_size": ["small_1-10", "medium_10-30", "large_30+"],
        "duration": ["short_<1wk", "mid_2-4wk", "long_1-6mo"],
        "assessment": ["formative", "summative", "project_based"],
        "tech": ["provided", "BYOD", "limited"],
        "budget": ["low", "medium", "high"]
    }
}
```

2. Use stratified sampling to select 100 combinations ensuring balanced coverage across all categories. Avoid over-representing any single variable.

3. For each combination, generate a scenario with an LLM prompt:

```
Given this context: {variables}
Generate an instructional design scenario with:
- A specific educational goal in SMART format
- Concrete constraints and prerequisites
- Realistic learner background details
- Available resources matching the budget/tech level
```

4. Apply two-layer validation: (a) rule-based check that all required fields are present and constraint combinations are logically consistent (e.g., "VR delivery" + "limited tech" should be flagged), (b) LLM-based review for narrative coherence.

Output: 100 validated scenarios in structured JSON, each with context variables, SMART goals, constraints, and learner profiles, stratified roughly equally across Easy/Medium/Hard difficulty.

**Example 3: Multi-Judge Evaluation of Generated Course Content**

User: "Evaluate my LLM-generated lesson plans using a reliable scoring method."

Approach:
1. Define the two-stage scoring rubric per ADDIE sub-step:

```python
SCORING_STAGES = {
    "stage_1_status": ["absent", "weak", "moderate", "good", "excellent"],
    "stage_2_bounds": {
        "absent": [0.0, 2.0],
        "weak": [1.0, 3.9],
        "moderate": [4.0, 6.9],
        "good": [7.0, 8.9],
        "excellent": [9.0, 10.0]
    }
}
```

2. Send each lesson plan to 3+ LLM judges from different providers. Each judge first categorizes each sub-step, then assigns a numerical score within the category's bounds.

3. Compute median scores across judges. Check inter-judge reliability: calculate ICC or Krippendorff's alpha across all items. Flag any item where judges disagree by more than one category level for manual review.

4. Aggregate using phase weights (A:25%, D:25%, Dev:20%, I:15%, E:15%) for the ADDIE component (70% of final), plus trajectory quality (30% of final) if evaluating an agent pipeline.

Output: Per-item scores, phase-level aggregates, overall score, and inter-judge reliability metrics. Example: "Analysis: 8.2/10, Design: 7.8/10, Development: 7.1/10, Implementation: 6.9/10, Evaluation: 7.5/10. Overall: 7.64/10. Inter-judge reliability: 0.91."

## Best Practices

- **Do:** Use coarse-grained phase-level tools (5 tools for 5 ADDIE phases) rather than fine-grained sub-step tools. The LLM handles intra-phase reasoning better when given the full phase context in a single call.
- **Do:** Always pass accumulated phase history to subsequent phases so later outputs reference earlier decisions. Evaluation must reference the objectives from Design; Development must reference the strategies from Design.
- **Do:** Use at least 3 judges from different LLM providers for evaluation, with median aggregation. Single-judge evaluation introduces systematic bias of up to ±0.06 points.
- **Do:** Apply two-stage scoring (categorical status first, bounded numerical score second) to reduce score drift and anchor judgments.
- **Avoid:** Decomposing ISD into 14+ micro-tools with rigid sequencing. This adds multi-step overhead without improving output quality (82.96 vs. 86.49 for coarse-grained).
- **Avoid:** Generating scenarios without stratified sampling across context variables. Unbalanced coverage masks weaknesses in specific learner/domain/delivery combinations.
- **Avoid:** Skipping the alignment check between objectives and assessments. The strongest signal of quality is objective-assessment alignment (effect size d=0.32 for theory-based agents).

## Error Handling

- **Inconsistent context variable combinations:** If a generated scenario pairs contradictory constraints (e.g., "VR simulation" + "limited technology"), flag during rule-based validation and either resample or adjust the conflicting variable.
- **Phase output drift:** If a later phase contradicts decisions from an earlier phase (e.g., evaluation criteria don't match design objectives), re-run the drifting phase with explicit instructions to reference the earlier phase output. Include the specific objectives in the re-run prompt.
- **Low inter-judge reliability on specific items:** If any evaluation item scores below 0.75 reliability, examine the rubric wording for that sub-step. Ambiguous criteria (e.g., "adequate media selection") should be made concrete (e.g., "media selection matches delivery mode and technology constraints").
- **Scenario generation quality failures:** If LLM-generated scenarios lack specificity or realism, enrich the generation prompt with domain-specific seed content from relevant literature or real course catalogs.

## Limitations

- The ADDIE framework is comprehensive but heavyweight. For informal or micro-learning content (e.g., a single tutorial or quick reference), the full 33 sub-step evaluation is overkill. Consider using only the Analysis and Design phases.
- Multi-judge evaluation requires 3+ LLM API calls per item per judge, which scales linearly with scenario count. The paper's full evaluation cost ~$8,900 for ~108K judge calls.
- The Context Matrix covers educational domains well but may not capture niche training contexts (military, emergency response, artistic apprenticeships) without extending the variable taxonomy.
- ReAct-ADDIE performance was validated on GPT-5-mini, Gemini-3-Flash, and Solar-Pro3. Results may vary with smaller or less capable models that struggle with long-context phase reasoning.
- Theory-based advantages are most pronounced for complex, multi-constraint scenarios. For simple single-topic lessons, the difference between theory-grounded and baseline agents is marginal.

## Reference

**ISD-Agent-Bench: A Comprehensive Benchmark for Evaluating LLM-based Instructional Design Agents** — Jeon et al., 2026. [arXiv:2602.10620](https://arxiv.org/abs/2602.10620v1). Key sections: Section 3 for Context Matrix construction, Section 4.5 for React-ADDIE architecture, Section 5 for multi-judge protocol details, Table 4 for comparative agent performance.