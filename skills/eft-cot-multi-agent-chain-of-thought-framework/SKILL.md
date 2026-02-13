---
name: "eft-cot-multi-agent-chain-of-thought-framework"
description: "Build multi-agent emotion-focused therapy (EFT) reasoning pipelines for empathetic mental health Q&A systems. Uses a bottom-up three-stage chain-of-thought: Embodied Perception, Cognitive Exploration, and Narrative Intervention with eight specialized agents. Trigger phrases: 'build an EFT chatbot', 'emotion-focused therapy agent', 'empathetic counseling system', 'multi-agent mental health pipeline', 'somatic-aware therapy bot', 'EFT-CoT reasoning chain'."
---

# EFT-CoT: Multi-Agent Chain-of-Thought Framework for Emotion-Focused Therapy

This skill enables Claude to design and implement multi-agent systems that process mental health queries using Emotion-Focused Therapy (EFT) principles. Unlike CBT-based approaches that apply top-down rational restructuring ("your thought is distorted, replace it"), EFT-CoT follows a bottom-up trajectory: first grounding in the client's bodily and emotional experience, then exploring cognitive patterns, and finally restructuring through narrative. The framework decomposes this into eight specialized agents operating across three stages, producing chain-of-thought reasoning that is both interpretable and deeply empathetic.

## When to Use

- When the user asks to build a mental health Q&A system, counseling chatbot, or emotional support agent
- When implementing a multi-agent pipeline that needs to process emotional or psychological content with empathy
- When designing chain-of-thought reasoning for therapy-informed responses (especially moving beyond simple CBT reframing)
- When the user wants to fine-tune or prompt-engineer an LLM for empathetic, structured therapeutic dialogue
- When building a system that must distinguish between surface emotions and underlying core emotional experiences
- When creating dataset pipelines (like EFT-Instruct) for distilling therapeutic reasoning into training data
- When the user asks to implement somatic awareness, emotion scheme analysis, or narrative restructuring in an AI system

## Key Technique

**Bottom-Up vs. Top-Down Therapy Reasoning.** Most LLM-based mental health systems use Cognitive Behavioral Therapy (CBT), which works top-down: identify a distorted thought, challenge it rationally, replace it. This often feels dismissive because it skips the client's felt, embodied experience. EFT-CoT reverses this. It starts from the body (somatic sensations, primary emotions), moves through cognitive exploration (what beliefs and emotion schemes underlie the feeling), and only then constructs a narrative intervention. This mirrors how emotions actually form: sensation precedes cognition, and cognition precedes narrative meaning-making.

**Eight Specialized Agents Across Three Stages.** The framework partitions the therapeutic reasoning into discrete, auditable steps handled by purpose-built agents. Stage 1 (Embodied Perception) uses agents for somatic awareness mapping (detecting body-referenced language like "my chest feels tight") and adaptive emotion assessment (classifying primary vs. secondary vs. instrumental emotions). Stage 2 (Cognitive Exploration) deploys agents for emotion scheme analysis, core belief extraction, and maladaptive pattern identification. Stage 3 (Narrative Intervention) uses agents for narrative restructuring, adaptive coping generation, and response synthesis. Each agent produces structured intermediate output that feeds the next, creating an interpretable chain-of-thought.

**Chain-of-Thought Distillation.** The EFT-Instruct methodology shows how to build training data: take authentic mental health texts, run them through the multi-agent pipeline to generate step-by-step reasoning traces, then use those traces to fine-tune a single model that internalizes the multi-agent reasoning. This lets you deploy a single model that reasons as if it had eight agents, reducing inference cost while preserving reasoning depth.

## Step-by-Step Workflow

1. **Parse the input for emotional content.** Extract the user's stated problem, any emotion words (explicit: "I feel anxious"; implicit: "I can't breathe when I think about it"), and situational context. Separate the presenting complaint from embedded emotional signals.

2. **Run Somatic Awareness Mapping (Agent 1).** Scan the input for body-referenced language: physical sensations ("tightness in my chest," "stomach in knots," "can't stop shaking"), somatic metaphors ("drowning," "weight on my shoulders"), and autonomic markers (sleep disruption, appetite changes). Output a structured somatic map:
   ```json
   {
     "somatic_signals": ["chest tightness", "shallow breathing"],
     "somatic_metaphors": ["feels like drowning"],
     "autonomic_markers": ["insomnia", "loss of appetite"]
   }
   ```

3. **Run Adaptive Emotion Assessment (Agent 2).** Classify the emotions detected into EFT categories:
   - **Primary adaptive**: healthy responses to the situation (e.g., grief after loss)
   - **Primary maladaptive**: deep-seated painful emotions from old wounds (e.g., shame from childhood neglect)
   - **Secondary**: reactive emotions covering the primary ones (e.g., anger masking hurt)
   - **Instrumental**: emotions used to influence others (e.g., performative helplessness)
   Output the classification with evidence from the text.

4. **Run Emotion Scheme Analysis (Agent 3).** Identify the underlying emotion schemes -- the learned patterns connecting situations, bodily sensations, cognitions, and action tendencies. For example: "When criticized (situation) -> chest tightens (body) -> I'm not good enough (cognition) -> withdraw (action)."

5. **Run Core Belief Extraction (Agent 4).** From the emotion schemes, extract the core beliefs driving the maladaptive patterns. These are typically absolute statements: "I am unlovable," "The world is unsafe," "I must be perfect to be accepted." Distinguish between beliefs the client states explicitly and those inferred from patterns.

6. **Run Maladaptive Pattern Identification (Agent 5).** Map how the core beliefs, emotion schemes, and somatic responses form self-reinforcing cycles. Identify the specific avoidance strategies, emotional suppression patterns, or repetitive interpersonal scripts the client is stuck in.

7. **Run Narrative Restructuring (Agent 6).** Construct an alternative narrative that: (a) validates the primary emotion ("It makes sense that you feel this way"), (b) names the somatic experience ("That tightness you feel is your body's alarm system"), (c) gently reframes the core belief through the emotional experience rather than through logic alone, and (d) connects to the client's own stated values or desires.

8. **Run Adaptive Coping Generation (Agent 7).** Generate concrete, emotion-informed coping strategies that address the specific somatic, emotional, and cognitive patterns identified. These should be grounded in EFT techniques: emotion naming, somatic grounding exercises, empty-chair adaptations, or compassionate self-talk scripts tied to the identified core beliefs.

9. **Run Response Synthesis (Agent 8).** Compose the final response by weaving together outputs from all agents into a coherent, warm, professional therapeutic reply. The response must: lead with empathic attunement (reflecting the felt experience), include psychoeducation where appropriate, offer the narrative reframe, and close with actionable next steps. Maintain appropriate boundaries (not diagnosing, encouraging professional help for severe cases).

10. **Validate the chain-of-thought.** Review the full reasoning trace for: (a) logical consistency across stages, (b) no premature cognitive restructuring before embodied acknowledgment, (c) empathy depth (does the response feel like it truly heard the person?), and (d) appropriate safety guardrails (suicide risk screening, crisis resource provision if indicated).

## Concrete Examples

**Example 1: Building an EFT-CoT Pipeline in Python**

User: "I want to build a mental health chatbot that uses emotion-focused therapy instead of CBT. Can you help me design the multi-agent pipeline?"

Approach:
1. Define agent interfaces with typed inputs/outputs
2. Implement the three-stage pipeline with inter-agent message passing
3. Use structured prompts for each agent role

Output:
```python
from dataclasses import dataclass, field
from enum import Enum

class EmotionType(Enum):
    PRIMARY_ADAPTIVE = "primary_adaptive"
    PRIMARY_MALADAPTIVE = "primary_maladaptive"
    SECONDARY = "secondary"
    INSTRUMENTAL = "instrumental"

@dataclass
class SomaticMap:
    signals: list[str]        # "chest tightness", "jaw clenching"
    metaphors: list[str]      # "drowning", "walls closing in"
    autonomic: list[str]      # "insomnia", "nausea"

@dataclass
class EmotionAssessment:
    emotions: dict[str, EmotionType]  # emotion_name -> category
    primary_target: str               # the core emotion to process
    evidence: dict[str, str]          # emotion -> textual evidence

@dataclass
class EmotionScheme:
    trigger: str        # "being criticized"
    body_response: str  # "chest tightens, breathing shallows"
    cognition: str      # "I'm not good enough"
    action_tendency: str # "withdraw and isolate"

@dataclass
class CoreBelief:
    belief: str          # "I am fundamentally flawed"
    source: str          # "explicit" or "inferred"
    supporting_schemes: list[str]

@dataclass
class EFTChainOfThought:
    """Complete reasoning trace from all eight agents."""
    somatic_map: SomaticMap
    emotion_assessment: EmotionAssessment
    emotion_schemes: list[EmotionScheme]
    core_beliefs: list[CoreBelief]
    maladaptive_cycles: list[str]
    narrative_reframe: str
    coping_strategies: list[str]
    final_response: str

# Agent base with LLM call
class EFTAgent:
    def __init__(self, role: str, system_prompt: str, llm_client):
        self.role = role
        self.system_prompt = system_prompt
        self.llm = llm_client

    def run(self, context: dict) -> dict:
        prompt = self._build_prompt(context)
        return self.llm.chat(self.system_prompt, prompt, response_format="json")

    def _build_prompt(self, context: dict) -> str:
        raise NotImplementedError

# Stage 1: Embodied Perception
class SomaticAwarenessAgent(EFTAgent):
    def _build_prompt(self, context):
        return (
            f"Analyze this text for body-referenced language, somatic "
            f"metaphors, and autonomic markers.\n\n"
            f"Client text: {context['user_input']}\n\n"
            f"Return JSON with keys: signals, metaphors, autonomic"
        )

class AdaptiveAssessmentAgent(EFTAgent):
    def _build_prompt(self, context):
        return (
            f"Classify emotions in this text into EFT categories: "
            f"primary_adaptive, primary_maladaptive, secondary, instrumental.\n\n"
            f"Client text: {context['user_input']}\n"
            f"Somatic map: {context['somatic_map']}\n\n"
            f"Identify which emotion is the primary target for processing."
        )

# Pipeline orchestrator
class EFTCoTPipeline:
    def __init__(self, llm_client):
        self.agents = {
            "somatic": SomaticAwarenessAgent("somatic", SOMATIC_PROMPT, llm_client),
            "assessment": AdaptiveAssessmentAgent("assessment", ASSESS_PROMPT, llm_client),
            # ... remaining 6 agents follow the same pattern
        }
        self.stage_order = [
            ["somatic", "assessment"],           # Stage 1: Embodied Perception
            ["schemes", "beliefs", "patterns"],  # Stage 2: Cognitive Exploration
            ["narrative", "coping", "synthesis"], # Stage 3: Narrative Intervention
        ]

    def run(self, user_input: str) -> EFTChainOfThought:
        context = {"user_input": user_input}
        for stage in self.stage_order:
            for agent_name in stage:
                result = self.agents[agent_name].run(context)
                context[agent_name] = result
        return self._build_cot(context)
```

**Example 2: Designing EFT-Instruct Training Data Pipeline**

User: "I have 50,000 mental health forum posts. How do I create an EFT-Instruct dataset to fine-tune a model?"

Approach:
1. Run each post through the full EFT-CoT pipeline to generate reasoning traces
2. Format as instruction-following examples with the chain-of-thought as the target
3. Apply quality filtering

Output:
```python
# EFT-Instruct dataset generation
def generate_eft_instruct(posts: list[str], pipeline: EFTCoTPipeline) -> list[dict]:
    dataset = []
    for post in posts:
        cot = pipeline.run(post)
        example = {
            "instruction": (
                "You are an emotion-focused therapy counselor. "
                "Respond to the following with empathy and therapeutic depth."
            ),
            "input": post,
            "chain_of_thought": {
                "stage_1_embodied_perception": {
                    "somatic_map": cot.somatic_map.__dict__,
                    "emotion_assessment": cot.emotion_assessment.__dict__,
                },
                "stage_2_cognitive_exploration": {
                    "emotion_schemes": [s.__dict__ for s in cot.emotion_schemes],
                    "core_beliefs": [b.__dict__ for b in cot.core_beliefs],
                    "maladaptive_cycles": cot.maladaptive_cycles,
                },
                "stage_3_narrative_intervention": {
                    "narrative_reframe": cot.narrative_reframe,
                    "coping_strategies": cot.coping_strategies,
                },
            },
            "output": cot.final_response,
        }
        dataset.append(example)
    return dataset

# Quality filter: discard traces with empty somatic maps or missing beliefs
def quality_filter(dataset):
    return [
        ex for ex in dataset
        if len(ex["chain_of_thought"]["stage_1_embodied_perception"]["somatic_map"]["signals"]) > 0
        and len(ex["chain_of_thought"]["stage_2_cognitive_exploration"]["core_beliefs"]) > 0
    ]
```

**Example 3: Single-Model Inference with Internalized EFT-CoT**

User: "I've already fine-tuned my model on EFT-Instruct data. How should I prompt it at inference time to get the full chain-of-thought?"

Approach:
1. Use a structured prompt that cues each stage of the reasoning chain
2. Parse the structured output into the three stages
3. Extract the final response while retaining the trace for interpretability

Output:
```python
EFT_INFERENCE_PROMPT = """You are an emotion-focused therapy counselor. Analyze the client's
message using bottom-up EFT reasoning, then respond.

## Stage 1: Embodied Perception
- Somatic signals: [identify body-referenced language]
- Emotion classification: [primary adaptive/maladaptive, secondary, instrumental]

## Stage 2: Cognitive Exploration
- Emotion schemes: [trigger -> body -> cognition -> action patterns]
- Core beliefs: [underlying absolute beliefs]
- Maladaptive cycles: [self-reinforcing patterns]

## Stage 3: Narrative Intervention
- Validation: [reflect the felt experience]
- Reframe: [emotion-grounded narrative shift]
- Coping: [concrete strategies tied to identified patterns]

## Response
[Final empathetic therapeutic response]

Client message: {user_message}"""

def eft_inference(model, user_message: str) -> dict:
    prompt = EFT_INFERENCE_PROMPT.format(user_message=user_message)
    raw = model.generate(prompt)
    # Parse structured output into stages for audit trail
    stages = parse_eft_stages(raw)
    return {
        "reasoning_trace": stages,
        "response": stages["response"],
    }
```

## Best Practices

- **Do:** Always complete Stage 1 (Embodied Perception) before moving to Stage 2 (Cognitive Exploration). The entire value of EFT-CoT is that body and emotion come first. Skipping to cognitive analysis reproduces the CBT pattern the framework is designed to avoid.
- **Do:** Distinguish between primary and secondary emotions in every analysis. If a user says "I'm so angry at myself," the anger is likely secondary -- the primary emotion underneath may be shame, grief, or fear. The agents must surface this.
- **Do:** Include safety guardrails in the Response Synthesis agent. Screen for crisis indicators (suicidal ideation, self-harm, abuse) and prepend crisis resources before any therapeutic content when detected.
- **Do:** Make the chain-of-thought visible and auditable. One of EFT-CoT's key advantages is interpretability -- clinicians or reviewers can inspect each agent's reasoning. Never collapse the trace into a black box.
- **Avoid:** Jumping to advice or cognitive reframing before validating the emotional experience. Responses like "Have you tried thinking about it differently?" without first acknowledging the felt experience will score poorly on empathy depth.
- **Avoid:** Using the framework for diagnosis. EFT-CoT generates empathetic therapeutic responses, not clinical assessments. Always include disclaimers and encourage professional consultation for serious conditions.

## Error Handling

- **Empty somatic map:** Some inputs contain no body-referenced language. In this case, the Somatic Awareness Agent should output an explicit "no somatic signals detected" flag, and the pipeline should still proceed -- not all clients express somatically, and the absence itself is informative.
- **Ambiguous emotion classification:** When the Adaptive Assessment Agent cannot confidently classify an emotion, it should output multiple candidates ranked by confidence rather than forcing a single label. Downstream agents should work with the top candidate while noting uncertainty.
- **Crisis detection:** If any agent detects indicators of imminent self-harm or suicidal ideation, the pipeline should short-circuit: skip remaining agents, output crisis resources immediately, and flag the interaction for human review.
- **Circular reasoning in schemes:** Core belief extraction can produce tautologies ("I feel bad because I believe I'm bad"). The Maladaptive Pattern Agent should detect circular schemes and probe for the originating experience or memory that anchors the belief.
- **Overly long inputs:** For extended client narratives, chunk the input and run Stage 1 on each chunk, then merge somatic maps before proceeding to Stage 2 with the aggregated context.

## Limitations

- EFT-CoT is designed for mental health question answering and emotional support, not for clinical treatment. It should never replace a licensed therapist.
- The framework assumes the input contains enough emotional or experiential content to analyze. Factual or purely logistical queries ("What are the side effects of sertraline?") will produce empty or forced reasoning traces -- route these to a standard Q&A system instead.
- The eight-agent pipeline has high inference cost when run as separate LLM calls. For production, consider the EFT-Instruct distillation approach to internalize the reasoning into a single model.
- The somatic awareness stage relies on the client's self-reported language. Clients who are alexithymic (difficulty identifying emotions) or culturally less expressive about bodily sensations may produce minimal somatic data, reducing Stage 1's effectiveness.
- Evaluation metrics like "empathy depth" and "structural professionalism" are inherently subjective. Human evaluation remains necessary for deployment validation.

## Reference

**Paper:** [EFT-CoT: A Multi-Agent Chain-of-Thought Framework for Emotion-Focused Therapy](https://arxiv.org/abs/2601.17842v1) (Du et al., 2026). Key sections: the three-stage agent architecture (Section 3), EFT-Instruct dataset construction methodology (Section 4), and ablation studies confirming that removing any agent degrades empathy and professionalism scores (Section 5).