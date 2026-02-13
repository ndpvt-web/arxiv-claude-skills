---
name: "open-tutorai-open-source-platform"
description: "Build personalized AI tutoring systems with structured onboarding, four-layer prompt architecture, adaptive lesson generation, 3D avatar integration, and learner analytics. Use when: 'build an AI tutor', 'create a personalized learning platform', 'design an educational chatbot with avatars', 'generate adaptive lesson plans', 'build a learner onboarding flow', 'create an intelligent tutoring system'."
---

# Open TutorAI: Building Personalized AI Tutoring Platforms

This skill enables Claude to design and implement personalized AI tutoring systems following the Open TutorAI architecture. The core technique is a four-layer compositional prompting system that decomposes tutoring into chained, context-aware prompts — combined with structured learner onboarding, adaptive lesson generation, 3D avatar integration, and embedded learning analytics. This produces tutoring platforms that adapt communication style, reasoning strategy, and content pacing to individual learner profiles.

## When to Use

- When the user asks to build an AI tutor, educational chatbot, or intelligent tutoring system
- When designing a learner onboarding flow that captures goals, preferences, and learning context
- When constructing multi-layer system prompts for pedagogical AI assistants
- When building adaptive lesson plan generators that decompose topics into sequenced micro-sessions
- When integrating 3D avatars or animated characters into an educational web interface
- When implementing learner analytics dashboards tracking engagement and cognitive metrics
- When creating role-based educational platforms serving learners, educators, and parents

## Key Technique: Four-Layer Compositional Prompting

Open TutorAI's central innovation is a **four-layer prompt architecture** that separates pedagogical concerns into composable layers, each injected dynamically based on learner context:

1. **Global Context Layer** — Defines the AI's pedagogical role, tutoring intent, tone constraints, and output format rules. This layer is static per assistant instance and establishes the tutor's persona.
2. **Instructional Logic Layer** — Encodes the teaching action sequence: topic introduction, progressive concept buildup, real-world examples, practice tasks, and quiz generation. This layer controls *what* the tutor teaches and in *what order*.
3. **Adaptive Variable Layer** — Dynamically injects learner-specific data: learning objectives, preferred communication style, current progress, quiz performance, and engagement scores. This layer makes each interaction personalized.
4. **Post-Interaction Layer** — Manages follow-up: quiz feedback, content review for incorrect answers, advancement decisions, and next-session planning.

The key insight is **compositional prompting** — complex tutoring tasks are decomposed into smaller chained prompts rather than handled by a single monolithic prompt. A lesson plan prompt chains into a micro-lesson prompt, which chains into a quiz prompt, which chains into a feedback prompt. Each chain carries forward the learner's adaptive variables, making the system responsive without losing pedagogical structure.

The assistant-generation pipeline further personalizes this by selecting **reasoning strategies** (deductive, inductive, analogical, causal, abductive) based on the subject domain and learner profile, and adapting **communication style** dynamically when the learner struggles.

## Step-by-Step Workflow

### 1. Design the Learner Profile Schema

Define a structured schema capturing onboarding data. At minimum, include:

```json
{
  "learner_id": "uuid",
  "display_name": "string",
  "learning_objectives": [{ "topic": "string", "description": "string", "goal_type": "mastery | exploration | certification" }],
  "preferred_style": "visual | textual | conversational | mixed",
  "experience_level": "beginner | intermediate | advanced",
  "planning_horizon": { "start_date": "ISO-8601", "end_date": "ISO-8601" },
  "uploaded_materials": ["file_ref"],
  "engagement_metrics": { "form_completion_rate": 0.0, "avg_time_per_step_seconds": 0, "quiz_scores": [] }
}
```

### 2. Build the Structured Onboarding Flow

Create a multi-step onboarding UI that collects learner data sequentially:
- Step 1: Learning objective entry (topic + free-text description)
- Step 2: Goal type selection (mastery, exploration, certification)
- Step 3: Course material upload (PDFs, notes, syllabi for RAG ingestion)
- Step 4: Start/end date planning
- Step 5: Avatar selection and customization
- Track timestamps per step to compute engagement metrics from the start.

### 3. Construct the Four-Layer System Prompt

Assemble the system prompt by concatenating four distinct sections:

```
[GLOBAL CONTEXT]
You are a personal tutor specializing in {topic}. Your role is to guide the
learner through structured micro-lessons. Maintain a {tone} tone. Always
provide examples before abstractions. Never exceed the learner's current level.

[INSTRUCTIONAL LOGIC]
Follow this session structure for each 25-minute micro-lesson:
1. Brief introduction connecting to the previous session (2 min)
2. Core concept explanation with one real-world analogy (8 min)
3. Guided practice with scaffolded hints (10 min)
4. Summary and preview of next session (3 min)
5. Optional 5-question quiz with immediate feedback (2 min)

[ADAPTIVE VARIABLES]
Learner: {display_name}
Level: {experience_level}
Current objective: {current_objective}
Previous quiz score: {last_score}%
Preferred reasoning: {reasoning_strategy}
Sessions completed: {session_count}/{total_sessions}

[POST-INTERACTION]
After each response: assess learner understanding from their reply.
If score < 60%: review the concept with a different analogy before advancing.
If score >= 80%: offer to skip to the next topic or deepen current topic.
Generate a 1-sentence progress note for the analytics dashboard.
```

### 4. Implement the Lesson Plan Generator

Build an endpoint that takes a topic and learner profile, then calls the LLM to decompose the topic into a sequence of micro-lessons:
- Each session targets one concept with clear prerequisites
- Sessions follow a dependency graph (e.g., "variables" before "loops")
- Output a JSON array of session objects with title, objectives, estimated duration, and prerequisites

### 5. Build the Conversational Engine with Chained Prompts

Implement prompt chaining: the lesson plan prompt produces a session outline, which feeds into a micro-lesson delivery prompt, which feeds into a quiz generation prompt, which feeds into a feedback prompt. Pass the learner's adaptive variables through each chain. Use the LLM's structured output mode to enforce consistent formatting.

### 6. Integrate the Avatar and 3D Environment (Optional)

If building an immersive interface:
- Render a 3D classroom scene using Three.js or similar
- Synchronize avatar lip-sync and gestures with TTS audio output
- Display LLM-generated content on a virtual board with intelligent pagination
- Embed a semi-transparent chat panel within the 3D space for text fallback

### 7. Implement Role-Based Access Control

Create three interface tiers:
- **Learner view**: Tutoring chat, lesson progress, avatar interaction, quiz history
- **Educator view**: Analytics dashboard, content configuration, learner performance heatmaps
- **Parent view**: High-level progress summary, engagement patterns, milestone notifications

Enforce via backend authorization tokens and conditionally rendered frontend components.

### 8. Build the Learning Analytics Pipeline

Track and compute:
- Form completion rates and time-per-step from onboarding
- Quiz scores over time (trend analysis)
- Session engagement duration
- Cognitive engagement score (composite of interaction depth, response quality, quiz performance)
- Generate actionable feedback summaries per learner per week

### 9. Deploy with Modular Backend

Structure the backend as three layers:
- **Frontend**: React/Next.js serving onboarding, chat, dashboard, and 3D views
- **Core Logic**: Conversational engine, lesson planner, analytics processor, RAG engine for uploaded materials
- **Infrastructure**: LLM provider abstraction (supporting both local Ollama and cloud OpenAI), containerized with Docker

### 10. Test Adaptive Behavior End-to-End

Verify that changing a learner's profile (e.g., switching from beginner to intermediate, or changing goal type) produces measurably different tutoring output: different reasoning strategies, adjusted pacing, and appropriate content depth.

## Concrete Examples

**Example 1: Building a personalized math tutor API**

User: "Build me an API for a personalized math tutor that adapts to each student's level."

Approach:
1. Define a learner profile schema with fields for math_level, current_topic, quiz_history, and preferred_explanation_style
2. Create a `/onboard` POST endpoint that accepts learner goals and returns a generated lesson plan
3. Build the four-layer system prompt: global context sets the math tutor persona, instructional logic defines the lesson flow (concept -> example -> practice -> quiz), adaptive variables inject the learner's current level and topic, post-interaction manages quiz scoring and advancement logic
4. Create a `/chat` POST endpoint that accepts learner_id + message, loads the learner profile, assembles the layered prompt, and streams the LLM response
5. Create a `/progress` GET endpoint returning quiz scores, session count, and engagement metrics

Output structure:
```python
# POST /onboard
{
  "learner_id": "abc-123",
  "lesson_plan": [
    {"session": 1, "topic": "Order of Operations", "prereqs": [], "objectives": ["Apply PEMDAS to multi-step expressions"]},
    {"session": 2, "topic": "Fractions Basics", "prereqs": [1], "objectives": ["Add and subtract fractions with unlike denominators"]},
    {"session": 3, "topic": "Fraction Operations", "prereqs": [2], "objectives": ["Multiply and divide fractions"]}
  ]
}

# POST /chat
{
  "response": "Let's work through order of operations step by step. Consider: 3 + 4 x 2. Which operation do you think we should do first?",
  "session_progress": {"current_step": "guided_practice", "concept": "PEMDAS"},
  "analytics_note": "Learner engaged with introductory example; awaiting response to assess baseline understanding."
}
```

**Example 2: Designing a learner onboarding flow for a language learning app**

User: "Design the onboarding flow for an AI language tutor that personalizes lessons."

Approach:
1. Map out 5 onboarding steps: target language + native language, proficiency self-assessment, learning goal (travel, business, academic), preferred session length, sample interaction to calibrate actual level
2. Define the profile schema capturing all inputs plus computed engagement metrics (time spent on each step, completeness)
3. Build the adaptive variable layer that translates onboarding data into prompt parameters: `difficulty_level`, `vocabulary_domain`, `grammar_focus`, `conversation_style`
4. Design the prompt chain: onboarding data -> lesson plan generator -> first session prompt with calibration questions

Output:
```
Onboarding Step 1/5: "What language do you want to learn?"
  -> Captures: target_language, native_language

Onboarding Step 2/5: "How would you rate your current level?"
  -> Captures: self_assessed_level (A1-C2 CEFR scale)

Onboarding Step 3/5: "What's your main goal?"
  -> Captures: goal_type (travel | business | academic | casual)

Onboarding Step 4/5: "How long do you want each session?"
  -> Captures: session_duration_minutes (15 | 25 | 40)

Onboarding Step 5/5: Calibration conversation (3 exchanges in target language)
  -> Computes: actual_level, vocabulary_range, grammar_accuracy

Generated Adaptive Variables:
  difficulty_level: "A2" (self-assessed B1 but calibration shows A2)
  vocabulary_domain: "travel"
  grammar_focus: "present tense, basic questions"
  conversation_style: "casual, encouraging"
```

**Example 3: Adding analytics to an existing chatbot tutor**

User: "I have an AI tutoring chatbot but no analytics. How should I add learner tracking?"

Approach:
1. Instrument the existing chat endpoint to log: session_id, timestamp, message_length, response_time, topic_tag
2. After each quiz interaction, store: question_id, learner_answer, correct_answer, time_to_answer, is_correct
3. Compute per-learner weekly metrics: sessions_completed, avg_quiz_score, engagement_duration, topic_progression_rate
4. Build a cognitive engagement score: weighted composite of quiz_accuracy (40%), interaction_depth (30%), consistency (30%)
5. Add a post-interaction layer to the system prompt that generates a one-line analytics note after each exchange
6. Create an educator dashboard endpoint returning aggregated metrics with trend indicators

Output:
```json
{
  "learner_id": "abc-123",
  "week": "2026-W06",
  "sessions_completed": 4,
  "avg_quiz_score": 72,
  "quiz_score_trend": "+8% vs last week",
  "engagement_duration_minutes": 94,
  "cognitive_engagement_score": 0.68,
  "top_strength": "Real-world application problems",
  "area_for_review": "Abstract formula derivation",
  "recommendation": "Increase analogical reasoning examples for formula topics"
}
```

## Best Practices

- **Do:** Keep the four prompt layers in separate templates so you can modify instructional logic without touching the global context or adaptive variables. This modularity is the core architectural advantage.
- **Do:** Timestamp every onboarding step and interaction. Engagement timing data is as valuable as quiz scores for detecting disengagement early.
- **Do:** Use structured output (JSON mode) for lesson plan generation and quiz grading to ensure downstream components can parse results reliably.
- **Do:** Implement the reasoning strategy selector — choosing between deductive, inductive, and analogical reasoning based on subject matter makes a measurable difference in learner comprehension.
- **Avoid:** Putting all personalization into a single monolithic system prompt. The four-layer separation exists specifically to prevent prompt bloat and enable independent updates.
- **Avoid:** Skipping the calibration step in onboarding. Learners routinely overestimate their level; a short interaction-based assessment prevents mismatched content from the first session.
- **Avoid:** Hard-coding lesson sequences. Use dependency graphs so the system can reorder or skip sessions based on demonstrated mastery.

## Error Handling

- **LLM generates off-topic content**: The instructional logic layer should include guardrails ("Stay within the scope of {current_topic}. If the learner asks unrelated questions, acknowledge briefly and redirect."). Validate output against the current session's topic tag.
- **Learner profile is incomplete**: Default missing adaptive variables to conservative values (beginner level, conversational style, no prerequisites assumed). Flag incomplete profiles in the educator dashboard.
- **Quiz parsing fails**: If the LLM produces malformed quiz JSON, retry with a stricter schema prompt. Fall back to free-text feedback if structured output fails twice.
- **Avatar sync lag**: If TTS/avatar rendering falls behind, degrade gracefully to text-only mode with a notice. Never block the tutoring flow on avatar rendering.
- **RAG retrieval returns irrelevant material**: Set a similarity threshold for uploaded course materials. If no chunk exceeds the threshold, rely on the LLM's parametric knowledge and note the gap in the educator dashboard.

## Limitations

- The four-layer prompting system adds latency compared to single-prompt approaches. For real-time conversational tutoring, optimize by caching the global context and instructional logic layers and only recomputing the adaptive variable layer per turn.
- Avatar integration (3D rendering, TTS sync) requires significant frontend engineering and may not be feasible for all projects. The text-based tutoring path works independently and should be built first.
- Learning analytics quality depends on interaction volume. Engagement scores are unreliable with fewer than 5 sessions per learner.
- The system assumes learners interact honestly during onboarding and quizzes. Adversarial or disengaged inputs will produce inaccurate learner profiles. Consider adding consistency checks across sessions.
- Compositional prompt chaining can accumulate errors if intermediate outputs are malformed. Validate the output of each chain step before feeding it to the next.

## Reference

**Paper**: [Open TutorAI: An Open-source Platform for Personalized and Immersive Learning with Generative AI](https://arxiv.org/abs/2602.07176v1) (El Hajji et al., 2026). Focus on Section 3 for the four-layer prompt architecture and assistant-generation pipeline, and Section 4 for the learning analytics module and engagement scoring methodology.