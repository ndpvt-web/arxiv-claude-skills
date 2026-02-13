---
name: "note2chat-improving-multi-turn-clinical"
description: "Build structured multi-turn clinical history-taking agents and diagnostic chatbots using the Note2Chat framework: convert unstructured notes/documents into decision-tree-guided dialogues, implement single-turn reasoning chains for iterative information gathering, and train LLMs for systematic diagnostic interviews. Use when: 'build a clinical interview chatbot', 'convert medical notes to dialogues', 'create a diagnostic decision tree from notes', 'implement multi-turn history taking', 'structured information gathering agent', 'note-driven dialogue generation pipeline'."
---

# Note2Chat: Structured Multi-Turn Clinical History Taking from Notes

This skill enables Claude to build systems that convert unstructured documents (medical notes, case reports, knowledge bases) into structured multi-turn dialogues for training information-gathering agents. The core technique from Note2Chat (AAAI 2026) is a three-phase pipeline: (1) generate a diagnostic decision tree from source documents, (2) synthesize realistic multi-turn dialogues guided by that tree, and (3) refine the agent through supervised fine-tuning, trajectory sampling, and preference optimization. The key insight is reframing iterative questioning as a sequence of independent single-turn reasoning problems, where each turn produces a summary, a reasoning plan, and a targeted next question.

## When to Use

- When building a chatbot that must systematically gather information from a user through multi-turn dialogue (clinical intake, troubleshooting, qualification screening)
- When converting unstructured notes, case files, or documentation into training dialogues for a conversational agent
- When implementing a diagnostic or triage system that narrows down possibilities through iterative questioning
- When designing a decision-tree-guided dialogue generation pipeline to produce synthetic training data
- When creating a patient/client simulator for training or evaluation purposes
- When adding structured reasoning (summary + plan + action) to each turn of a multi-turn agent

## Key Technique

**Note-to-Decision-Tree-to-Dialogue Pipeline.** Instead of requiring expensive human-annotated dialogue data, Note2Chat starts from widely available unstructured notes (e.g., discharge summaries containing chief complaint, history of present illness, and diagnosis). An LLM first analyzes the note and generates a JSON decision tree that encodes branching diagnostic logic -- each node contains clinical criteria, yes/no branches, and terminal diagnoses with confidence levels. This decision tree then guides a second LLM call that produces a realistic multi-turn dialogue where a doctor agent asks focused questions and a patient agent responds strictly from note contents. A critic pass then refines the dialogue by identifying missing facts (inserting `add_turn` actions) and logical inconsistencies (issuing `revise_turn` actions).

**Single-Turn Reasoning Paradigm.** The breakthrough design reframes the entire multi-turn history taking as a sequence of independent single-turn problems. At each turn, the doctor agent receives the conversation so far and produces three outputs: (1) a third-person clinical summary of information gathered, (2) a reasoning plan explaining which conditions the next question could help rule in or out, and (3) the actual question or differential diagnosis. This decomposition enables local supervision at each turn, dynamic adaptation to unexpected patient responses, and dramatically improved sample efficiency because each turn becomes an independent training example.

**Three-Stage Fine-Tuning.** Stage 1 performs supervised fine-tuning (SFT) on the synthesized dialogues. Stage 2 uses self-augmented trajectory sampling -- the SFT model generates multiple response trajectories, which are quality-filtered into preference pairs. Stage 3 applies Direct Preference Optimization (DPO) to align the model toward higher-quality diagnostic conversations. This pipeline achieved +16.9 F1 and +21.0 Top-1 diagnostic accuracy over GPT-4o.

## Step-by-Step Workflow

1. **Extract structured fields from source documents.** Parse each note/document into canonical sections: the presenting problem (chief complaint), detailed narrative (history of present illness), and ground-truth outcome (diagnosis). Store as structured records with a unique ID, e.g., `{note_id, chief_complaint, history_of_present_illness, final_diagnosis}`.

2. **Generate a decision tree from each record.** Prompt an LLM to produce a JSON decision tree that incorporates symptoms from the history and organizes potential diagnoses hierarchically. Each node should have: a clinical criterion, yes/no branches, and leaf nodes with a diagnosis and confidence. This tree serves as the structural backbone for dialogue generation.

3. **Synthesize a multi-turn dialogue guided by the decision tree.** Using the decision tree as context, prompt an LLM to generate a doctor-patient conversation where: the patient opens with the chief complaint or a single clear symptom, the doctor asks one focused question per turn (following SOCRATES: Site, Onset, Character, Radiation, Associated symptoms, Timing, Exacerbating/relieving factors, Severity), and the patient responds strictly from note content ("I don't know" for missing info). End with five ranked differential diagnoses.

4. **Refine the dialogue with a critic pass.** Run a critic prompt that checks for: (a) missing facts from the original note not yet elicited, generating `add_turn` actions with open-ended questions to naturally surface them; (b) logical inconsistencies where the doctor references unmentioned information or the patient volunteers unsolicited details, generating `revise_turn` corrections. Apply all actions to produce the refined dialogue.

5. **Add single-turn reasoning annotations.** For each doctor turn in the refined dialogue, generate three components: a third-person clinical summary of all information gathered so far, a question-planning rationale explaining which conditions the next question helps differentiate, and the actual question. For the final turn, replace question planning with differential-diagnosis planning explaining why each candidate diagnosis is included.

6. **Build SFT training data.** Format each annotated turn as an independent training example: system prompt (doctor role instructions) + conversation history up to that point as input, and the summary + reasoning + action as the target output. This creates N training examples from a single N-turn dialogue.

7. **Fine-tune with SFT (Stage 1).** Train the base model on the synthesized single-turn examples. Use a medical-capable base model (e.g., Qwen2.5-7B-Instruct or equivalent).

8. **Generate trajectory samples and filter (Stage 2).** Run the SFT model on held-out cases to generate multiple response trajectories per turn. Score each trajectory for clinical accuracy and dialogue quality. Create preference pairs: high-scoring trajectories as "chosen," low-scoring as "rejected."

9. **Apply DPO alignment (Stage 3).** Fine-tune the SFT model using Direct Preference Optimization on the preference pairs to push the model toward higher-quality clinical questioning patterns.

10. **Evaluate with semantic matching.** Use meaning-based (not lexical) matching to evaluate: (a) information coverage -- which note sentences appear in the dialogue, matched by turn number; (b) diagnostic accuracy -- whether candidate diagnoses match or are subtypes of the ground truth. Report F1 for coverage and Top-K accuracy for diagnosis.

## Concrete Examples

**Example 1: Building a clinical intake chatbot from discharge notes**

User: "I have a dataset of 5,000 discharge summaries with chief complaint, HPI, and diagnosis fields. Help me build a pipeline to generate training dialogues for a clinical history-taking chatbot."

Approach:
1. Parse CSV into structured records: `{note_id, chief_complaint, history_of_present_illness, final_diagnosis}`
2. For each record, call LLM with decision-tree generation prompt:
   ```
   Analyze this medical note and produce a diagnostic decision tree in JSON.
   Incorporate symptoms from the History of Present Illness.
   Each node: {criterion, yes_branch, no_branch}.
   Leaf nodes: {diagnosis, confidence}.
   Chief Complaint: {cc}
   HPI: {hpi}
   Diagnosis: {dx}
   ```
3. Feed decision tree + note into dialogue generation prompt with constraints:
   - Patient opens with chief complaint only
   - Doctor asks one question per turn, no jargon
   - Patient responds only from note content
   - Ends with 5 ranked differential diagnoses
4. Run critic pass to insert missing-fact turns and fix inconsistencies
5. Annotate each doctor turn with summary + reasoning plan
6. Export as JSONL training examples, one per turn

Output structure per example:
```json
{
  "note_id": "12345",
  "decision_tree": {"criterion": "chest pain present?", "yes": {...}, "no": {...}},
  "dialogue": [
    {"role": "patient", "content": "I've been having chest pain for two days."},
    {"role": "doctor", "summary": "Patient presents with 2-day chest pain.",
     "reasoning": "Need to differentiate cardiac vs musculoskeletal. Ask about character.",
     "content": "Can you describe what the pain feels like?"},
    {"role": "patient", "content": "It's a sharp, stabbing pain."}
  ],
  "diagnoses": ["1. Costochondritis", "2. Acute coronary syndrome", ...]
}
```

**Example 2: Implementing the single-turn reasoning paradigm for a diagnostic agent**

User: "I have a fine-tuned medical LLM. I want to add structured reasoning to each turn so it produces a summary, plan, and question instead of just a question."

Approach:
1. Define the doctor system prompt with explicit output structure:
   ```
   You are a medical interviewer. At each turn, output exactly three sections:
   **Summary:** A third-person clinical summary of all information gathered.
   **Plan:** Which conditions your next question helps rule in/out.
   **Question:** One focused, jargon-free question to the patient.
   When ready to diagnose, replace Plan/Question with:
   **Diagnosis Plan:** Why each candidate is considered.
   **Preliminary Diagnoses:** Five ranked diagnoses.
   ```
2. Maintain conversation history as a rolling context window
3. After each patient response, feed full history + system prompt to the model
4. Parse the structured output into components for logging and supervision
5. Use the summary field to detect information saturation (no new facts = time to diagnose)

Output per turn:
```
**Summary:** 45-year-old male presenting with 2-day sharp chest pain,
worse with deep breathing, no radiation, no prior cardiac history.

**Plan:** Sharp pain worsening with breathing suggests pleuritic cause.
Asking about recent illness/cough helps differentiate pneumonia,
pleurisy, and PE from cardiac causes.

**Question:** Have you had any cough, fever, or shortness of breath recently?
```

**Example 3: Building a patient simulator for evaluation**

User: "I need a simulated patient that responds realistically based on a case file, for testing my diagnostic chatbot."

Approach:
1. Parse the case file into a structured description
2. Set up the patient system prompt with strict constraints:
   ```
   You are a patient. Your case: {case_description}
   Rules:
   - No medical terminology. Describe symptoms in everyday language.
   - Only reveal information from your case when asked. Never volunteer extra details.
   - Say "I don't know" if asked about something not in your case.
   - If asked a repeated question, say "You already asked me that."
   - Keep responses to one sentence.
   - Never reveal you are an AI or reference the case file.
   ```
3. Wire the patient simulator against the doctor agent in a turn-taking loop
4. Log each exchange and evaluate information coverage against the original note
5. Score using semantic sentence matching: for each fact in the case, check if it appeared in a dialogue turn (match by meaning, return turn number or -1)

## Best Practices

- **Do:** Generate the decision tree BEFORE the dialogue. The tree provides structural scaffolding that prevents the conversation from wandering or missing key differential criteria.
- **Do:** Enforce the "one question per turn" constraint strictly. Multi-question turns are unrealistic and degrade training quality because they conflate multiple reasoning steps.
- **Do:** Use the critic/refinement pass. Raw generated dialogues consistently miss 15-30% of facts from the source note. The critic pass with `add_turn` and `revise_turn` actions closes this gap.
- **Do:** Decompose into single-turn reasoning examples for training. This multiplies your effective training data by N (number of turns) and enables per-turn quality control.
- **Avoid:** Letting the patient simulator fabricate symptoms not in the source note. This introduces noise that degrades both training data quality and evaluation validity.
- **Avoid:** Using lexical/string matching for evaluation. Clinical language has extensive synonymy ("heart attack" vs "myocardial infarction" vs "MI"). Always use meaning-based semantic matching.
- **Avoid:** Skipping Stage 2 (trajectory sampling). SFT alone produces a model that generates plausible but often suboptimal question sequences. The preference pairs from trajectory sampling are critical for the DPO stage to work.

## Error Handling

- **Decision tree generation fails or produces invalid JSON:** Retry with a more explicit JSON schema in the prompt. Add a validation step that checks for required fields (criterion, branches, leaf diagnoses) and rejects malformed trees. Fall back to a flat list of differential diagnoses if tree generation consistently fails for a particular note.
- **Dialogue ends prematurely (insufficient information gathered):** Compare the number of note sentences covered by the dialogue against a threshold (e.g., 70%). If below threshold, re-run generation with the critic feedback appended, or extend the dialogue by prompting "continue gathering history."
- **Patient simulator hallucinates facts not in the note:** This is the most common failure mode. Mitigate by adding an automated check: extract all factual claims from patient responses and verify each against the source note. Flag and regenerate turns with hallucinated content.
- **Single-turn reasoning produces empty or generic summaries:** The summary should grow monotonically in information content. If a turn's summary doesn't add new facts relative to the previous turn's summary, flag the turn for regeneration or trigger the diagnosis phase.
- **DPO training diverges or shows no improvement:** Ensure preference pairs have sufficient quality gap. Filter trajectory pairs where the chosen and rejected responses are too similar (e.g., by cosine similarity of their clinical summaries).

## Limitations

- The pipeline assumes source documents contain sufficient detail to generate meaningful decision trees. Sparse or poorly written notes produce shallow trees and uninformative dialogues.
- Decision-tree generation works well for differential-diagnosis-style reasoning but is less suited for open-ended information gathering without a finite set of possible outcomes.
- The single-turn reasoning paradigm adds latency per turn (summary + plan + question vs. just question) and increases token usage. Not suitable for latency-critical real-time applications without optimization.
- Generated dialogues inherit biases from the source notes and the LLM used for generation. Medical notes from a single institution may not generalize across populations or practice patterns.
- Evaluation via semantic matching requires a capable LLM as judge, introducing its own error rate. There is no fully automated ground-truth evaluation for clinical dialogue quality.

## Reference

**Paper:** [Note2Chat: Improving LLMs for Multi-Turn Clinical History Taking Using Medical Notes](https://arxiv.org/abs/2601.21551v1) (AAAI 2026)
**Code:** [github.com/zhentingsheng/Note2Chat](https://github.com/zhentingsheng/Note2Chat)
**Key insight:** Converting unstructured notes into decision-tree-guided dialogues, then decomposing multi-turn history taking into independent single-turn reasoning problems (summary + plan + action), dramatically improves diagnostic accuracy and training efficiency over direct multi-turn fine-tuning.