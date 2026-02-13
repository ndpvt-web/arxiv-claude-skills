---
name: cognitively-diverse-multiple-choice-question
description: >
  Generate high-quality multiple-choice questions at controlled cognitive levels using the ReQUESTA multi-agent framework.
  Decomposes MCQ authoring into planning, generation, evaluation, and post-processing stages with specialized agents
  targeting text-based (recall), inferential (synthesis), and main idea (abstraction) comprehension.
  Trigger phrases: "generate MCQs from this text", "create quiz questions at different difficulty levels",
  "make multiple choice questions for this reading", "build an assessment from this passage",
  "create comprehension questions", "generate exam items from this content"
---

# Cognitively Diverse Multiple-Choice Question Generation (ReQUESTA)

This skill enables Claude to generate psychometrically rigorous multiple-choice questions from any expository or academic text by implementing the ReQUESTA framework. Instead of producing questions in a single pass (which tends to yield easy, surface-level items with implausible distractors), this approach decomposes MCQ authoring into specialized subtasks: planning what to assess, generating questions at three distinct cognitive levels, evaluating each item against a quality checklist, iteratively refining failures, and post-processing for presentation consistency. The result is questions that are harder, more discriminative, and better aligned with genuine comprehension.

## When to Use

- When the user provides a passage, article, textbook chapter, or technical document and asks for quiz or exam questions
- When the user wants questions at specific cognitive levels (factual recall, inference, main idea)
- When the user needs assessment items with plausible, well-crafted distractors -- not obviously wrong answers
- When building reading comprehension assessments, course quizzes, or practice exams
- When the user asks to "create a quiz" or "generate test questions" from supplied content
- When the user wants to evaluate whether learners truly understand material vs. just recognizing surface details
- When generating MCQs for training data, educational apps, or certification prep materials

## Key Technique

**Why single-pass MCQ generation fails.** When an LLM generates MCQs in one shot, it defaults to easy text-based recall questions with distractors that are obviously wrong (different length, different register, or semantically distant from the correct answer). Research shows single-pass items average ~90% accuracy -- they don't differentiate learners. The core insight of ReQUESTA is that MCQ quality emerges from *workflow design*, not model capability.

**The hybrid multi-agent decomposition.** ReQUESTA separates stochastic tasks (planning, generation, evaluation) from deterministic tasks (text segmentation, task routing, formatting). A Planner agent first analyzes the text to extract key concepts, implicit inferences, and overarching themes, then produces a structured generation plan mapping content targets to cognitive levels. A Controller routes each subtask to one of three specialized Question Generators -- text-based, inferential, or main idea -- each scoped to its cognitive focus. An Evaluator then assesses each item against a quality checklist (stem clarity, answer-stem alignment, distractor plausibility, linguistic consistency). Failed items loop back to their generator with targeted revision feedback until they pass or hit a termination limit.

**Distractor quality as the differentiator.** The framework's largest advantage is in distractor generation. By constraining each generator's cognitive scope and applying post-hoc option-shortening (detecting and rewriting options that are disproportionately long -- a telltale sign of the correct answer), the output achieves balanced option lengths, consistent syntactic structure, and semantically plausible wrong answers. This is what makes items genuinely challenging rather than trick-detectable.

## Step-by-Step Workflow

1. **Preprocess the input text.** Segment the passage into coherent units at sentence or paragraph boundaries. Identify the domain, register, and approximate reading level. If the text exceeds ~2000 words, divide it into sections that can each support 3-5 questions.

2. **Plan the assessment.** Adopt the Planner role: summarize each segment, extract explicit key facts (for text-based items), identify implicit relationships requiring cross-sentence integration (for inferential items), and distill overarching themes or central arguments (for main idea items). Output a structured plan as JSON mapping each concept to a cognitive level and source segment:
   ```json
   {
     "items": [
       {"id": 1, "level": "text-based", "segment": 2, "target_concept": "definition of X"},
       {"id": 2, "level": "inferential", "segment": "1+3", "target_concept": "relationship between X and Y"},
       {"id": 3, "level": "main-idea", "segment": "all", "target_concept": "central argument about Z"}
     ]
   }
   ```

3. **Generate text-based questions.** For each text-based plan item, write a question targeting explicit, directly stated information. The correct answer must be verifiable by a single sentence or clause. Distractors must use vocabulary and phrasing from the same passage to remain plausible.

4. **Generate inferential questions.** For each inferential plan item, write a question requiring integration of information across multiple sentences or paragraphs. The correct answer should not appear verbatim in the text. Distractors should represent partially correct inferences or common misinterpretations.

5. **Generate main idea questions.** For each main idea plan item, write a question assessing understanding of the overarching theme, central argument, or primary purpose. Distractors should represent subsidiary points, overgeneralizations, or plausible-but-wrong framings of the text's purpose.

6. **Apply self-critique to each generated item.** Before external evaluation, check each question against three diagnostic prompts: (a) Is the stem clear and unambiguous? (b) Is the correct answer clearly the best option without being obvious? (c) Are distractors plausible, relevant, and distinct from each other?

7. **Evaluate against the quality checklist.** Score each item on: stem clarity, answer-stem alignment, distractor plausibility, distractor linguistic consistency (similar length/structure across all options), distractor semantic uniqueness (each wrong answer represents a different misconception), and absence of cuing (no "all of the above," no absolute terms like "always/never" that signal wrong answers).

8. **Revise failing items.** For any item that fails evaluation, return it to the appropriate generator with specific feedback (e.g., "Distractor C is too short relative to other options" or "The stem is ambiguous between answers A and B"). Regenerate and re-evaluate. Allow up to 2 revision cycles before accepting or flagging.

9. **Shorten and balance options.** Scan all items for length imbalances among answer options. If one option is significantly longer than the others (a common cue that it is correct), rewrite it for conciseness while preserving meaning. Ensure all four options in each question have comparable word counts and syntactic complexity.

10. **Format and finalize.** Shuffle option order so the correct answer isn't consistently in one position. Apply consistent labeling (A/B/C/D). Group questions by cognitive level or interleave them based on user preference. Include an answer key with the cognitive level and source segment for each item.

## Concrete Examples

**Example 1: Academic passage on photosynthesis**

User: "Generate 6 MCQs from this biology textbook section on photosynthesis, covering different difficulty levels."

Approach:
1. Segment the passage into light reactions, Calvin cycle, and overall significance sections
2. Plan: 2 text-based (chlorophyll absorption spectrum, ATP production location), 2 inferential (why C4 plants outperform C3 in hot climates, relationship between light and dark reactions), 2 main idea (primary function of photosynthesis in ecosystems, why photosynthesis is described as a two-stage process)
3. Generate, self-critique, evaluate, and balance options

Output:
```
## Assessment: Photosynthesis (6 items)

**Q1 [Text-Based]**
According to the passage, chlorophyll a absorbs light most efficiently in which wavelength ranges?
A. Green and yellow wavelengths (500-580 nm)
B. Blue-violet and red wavelengths (430-660 nm)
C. Ultraviolet and infrared wavelengths (below 380, above 750 nm)
D. Orange and yellow-green wavelengths (580-620 nm)

**Correct: B** | Source: Paragraph 2, sentence 3

**Q2 [Inferential]**
Based on the passage's discussion of CO2 fixation pathways, which of the following best explains why C4 plants maintain higher photosynthetic rates than C3 plants in hot, arid environments?
A. C4 plants have more chloroplasts per cell than C3 plants
B. C4 plants use a preliminary carbon fixation step that concentrates CO2 near RuBisCO, reducing photorespiration when stomata close
C. C4 plants absorb a broader spectrum of light than C3 plants
D. C4 plants require less water per molecule of glucose synthesized during the Calvin cycle

**Correct: B** | Source: Integration of paragraphs 4-5

**Q3 [Main Idea]**
Which statement best captures the central argument the author makes about photosynthesis in this section?
A. Photosynthesis is primarily important as the mechanism by which plants grow and reproduce
B. Photosynthesis functions as the foundational energy-conversion process linking solar radiation to nearly all biological energy systems
C. Photosynthesis represents an evolutionary adaptation unique to terrestrial plant species
D. Photosynthesis is best understood as a chemical reaction that converts carbon dioxide into organic sugars

**Correct: B** | Source: Passage-wide synthesis
```

**Example 2: Technical documentation quiz**

User: "Create comprehension questions for this API documentation on OAuth 2.0 authorization flows."

Approach:
1. Segment into: grant types overview, authorization code flow, token refresh, security considerations
2. Plan: text-based items on protocol specifics, inferential items on when to choose which flow, main idea on OAuth's design philosophy
3. Generate with technical accuracy, using plausible misconfiguration scenarios as distractors

Output:
```
**Q1 [Text-Based]**
According to the documentation, what is the primary purpose of the "state" parameter in the authorization code flow?
A. To encrypt the authorization code during transit between the authorization server and the client
B. To prevent cross-site request forgery by binding the request to the user's session
C. To specify which scopes the client application is requesting from the resource owner
D. To identify the redirect URI registered with the authorization server

**Correct: B** | Source: Section 3.1, "Security Parameters"

**Q2 [Inferential]**
A developer is building a single-page JavaScript application that cannot securely store a client secret. Based on the documentation's comparison of grant types, which flow should they implement and why?
A. Authorization Code with PKCE, because it replaces the client secret with a dynamically generated code verifier
B. Client Credentials grant, because it eliminates the need for user interaction during authentication
C. Implicit grant, because it issues tokens directly without an intermediate authorization code
D. Resource Owner Password Credentials, because it allows the application to authenticate with only a username and password

**Correct: A** | Source: Integration of Sections 2.1, 2.3, and 4.2
```

**Example 3: Generating questions with user-specified constraints**

User: "I need 4 inferential-only questions from this history passage about the Industrial Revolution. Make the distractors really hard to distinguish."

Approach:
1. Plan exclusively for inferential items -- identify cross-paragraph relationships, cause-effect chains, and comparative claims that require synthesis
2. Craft distractors that each represent a partially valid but incomplete inference
3. Apply extra scrutiny to distractor plausibility during evaluation -- each wrong answer should be defensible from one paragraph but contradicted by another

Output format: 4 questions where each distractor is grounded in real passage content but represents an incomplete or misweighted inference, with an explanation of why each distractor is wrong appended to the answer key.

## Best Practices

- **Do:** Create the generation plan before writing any questions. The plan-first approach prevents clustering around the most salient paragraph and ensures cognitive level diversity.
- **Do:** Write distractors drawn from the source text's own vocabulary. Plausible wrong answers use the same register and terminology as the correct answer.
- **Do:** Check option length balance after generation. If the correct answer is 15 words and distractors average 6 words, rewrite until comparable.
- **Do:** Vary the position of the correct answer across items. Shuffle after generation rather than trying to vary during generation.
- **Avoid:** Generating all questions in a single prompt. The decomposed approach (plan, then generate by cognitive level, then evaluate) produces measurably better items than single-pass generation.
- **Avoid:** Using "all of the above," "none of the above," or absolute qualifiers ("always," "never") in options -- these are test-taking cues that reduce item validity.
- **Avoid:** Writing inferential questions that can be answered from a single sentence. If the answer is directly stated, it's text-based regardless of how the stem is phrased.

## Error Handling

- **Passage too short (< 3 sentences):** Inform the user that meaningful MCQ generation requires sufficient content. Offer to generate 1-2 text-based items only, noting that inferential and main idea items need more material.
- **Ambiguous correct answer after self-critique:** If two options could defensibly be correct, revise the stem to add a qualifying phrase that disambiguates, or replace the weaker option with a clearly distinct distractor.
- **Distractor collapse (two distractors say the same thing differently):** Flag during evaluation. Replace one with a distractor targeting a different misconception.
- **Option length imbalance persists after revision:** If the correct answer is inherently more complex, lengthen the distractors by adding qualifying clauses rather than shortening the correct answer and losing precision.
- **User-requested cognitive level doesn't match content:** If the passage is purely factual with no argumentation, main idea questions will be forced. Inform the user and suggest substituting additional inferential items.

## Limitations

- Works best on expository and argumentative text (textbooks, articles, documentation). Narrative fiction and poetry require different question design principles not covered by this framework.
- The three cognitive levels (text-based, inferential, main idea) map to reading comprehension. For procedural knowledge (how-to guides), application-level questions need a different framing.
- Distractor quality depends on passage density. Sparse passages with few concepts yield limited distractor material -- the framework cannot manufacture plausible wrong answers from thin content.
- This framework generates 4-option MCQs. Other formats (true/false, matching, fill-in-the-blank, constructed response) are outside its scope.
- Without actual learner response data, psychometric properties (difficulty, discrimination) are estimated heuristically. Real calibration requires field testing.

## Reference

[Cognitively Diverse Multiple-Choice Question Generation: A Hybrid Multi-Agent Framework with Large Language Models](https://arxiv.org/abs/2602.03704v1) -- Tian et al. (2026). Focus on Section 3 (framework architecture), Figure 1 (agent pipeline diagram), and Section 5.2 (expert evaluation rubric for distractor quality).