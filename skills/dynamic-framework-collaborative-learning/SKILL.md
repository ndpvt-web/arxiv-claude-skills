---
name: "dynamic-framework-collaborative-learning"
description: "Build AI-moderated collaborative learning platforms with LLM-driven discussion facilitation, adaptive feedback, and participation balancing. Use when: 'build a collaborative learning app', 'create an AI discussion moderator', 'add adaptive feedback to my education platform', 'implement a group discussion system with AI', 'build a real-time classroom discussion tool', 'add participation balancing to my learning platform'."
---

# Dynamic Framework for Collaborative Learning

This skill enables Claude to build AI-moderated collaborative learning platforms where an LLM acts as a dynamic discussion moderator — posing questions drawn from a content dataset via RAG, managing turn-taking across participants, delivering personalized feedback, and adapting prompts in real time based on learner behavior. The architecture follows a three-tier modular design (ReactJS frontend, Flask/Socket.IO backend, LangChain-managed LLM layer) drawn from Tahir et al.'s framework, which demonstrated mean response latencies of 1.84 seconds and effective moderation of diverse participant behaviors including passive, toxic, off-topic, and highly engaged learners.

## When to Use

- When building a real-time group discussion platform where an AI moderator facilitates structured conversations
- When adding adaptive feedback to an existing education or training application
- When implementing participation balancing so quiet users get prompted and dominant users yield turns
- When creating a RAG-powered question retrieval system that feeds discussion prompts from a content dataset
- When designing moderation logic that handles toxic, off-topic, or disengaged participants without shutting them down
- When building a Socket.IO-based chat backend that integrates LLM responses into multi-user sessions
- When constructing persona-aware prompt pipelines that adapt based on conversation history and learner profiles

## Key Technique

The core innovation is treating the LLM not as a chatbot but as a **structured discussion moderator** with an explicit behavioral contract. The system prompt defines numbered responsibilities: welcome participants, present a passage, pose questions one at a time, ensure every student responds before revealing answers, provide constructive feedback without premature disclosure, and manage turn-taking by name (e.g., "What do you think, [name]?"). This transforms open-ended LLM generation into a predictable pedagogical loop.

The feedback mechanism operates on full conversation history rather than individual messages. After a discussion round, the system analyzes the entire chat log plus student identifiers to generate per-student feedback — identifying patterns like passivity (encouraged to ask questions), toxicity (acknowledged frustration, redirected constructively), off-topic drift (praised creativity, linked back to core topic), and strong engagement (refined to build on peers' ideas). This history-aware approach produces feedback that feels contextual rather than generic.

Scalability is achieved through three configurable parameters: `max_students` (room capacity), `max_tokens` (conversation window — trimmed after 5000 tokens to manage context), and `min_qa_pairs` (minimum question-answer pairs per content unit). The content layer is dataset-agnostic: any dataset with passages and Q&A pairs plugs in via a retrieval module that randomly selects content instances, supporting RAG without requiring a vector database for simple use cases.

## Step-by-Step Workflow

1. **Define the moderator system prompt** with explicit numbered responsibilities: greeting participants, presenting content, posing questions sequentially, managing turn-taking by participant name, providing feedback without revealing answers prematurely, and summarizing after all participants respond. Store this as a configurable template with placeholders for `{student_names}`, `{passage}`, and `{question}`.

2. **Set up the content retrieval module** — load your dataset of passages and Q&A pairs (JSON, CSV, or database), implement a retrieval function that selects content instances filtered by `min_qa_pairs`, and format them as context for the moderator prompt. For simple cases, random selection works; for curriculum-aligned use, add topic filtering or difficulty ranking.

3. **Build the Flask backend with Socket.IO** — create room management (unique room IDs, `max_students` capacity tracking, participant name registry), message routing (client → server → LLM → broadcast), and session state (current passage, current question index, which students have responded this round).

4. **Integrate LangChain for prompt management and conversational state** — initialize a conversation chain with the moderator system prompt, feed each incoming student message as a human turn, and manage the message window by trimming to `max_tokens` (default 5000) to prevent context overflow while preserving recent discussion context.

5. **Implement the discussion loop** in the backend: (a) moderator introduces itself and the passage, (b) poses the first question, (c) collects responses tracking which participants have answered, (d) after each response, moderator provides targeted feedback and optionally prompts the next silent participant by name, (e) once all participants respond, reveals the answer and transitions to the next question.

6. **Build participation tracking** — maintain a per-room dictionary mapping participant names to response counts and last-active timestamps. Before generating the moderator's next message, inject a participation summary into the prompt context (e.g., "Ethan has not responded to the last 2 questions"). The moderator's system prompt instructs it to direct questions to under-participating students.

7. **Implement behavioral intervention patterns** in the system prompt with explicit handling instructions: for toxicity, acknowledge the emotion and redirect ("I hear your frustration — let's channel that into the discussion"); for off-topic remarks, validate and bridge ("That's creative — how might it connect to [topic]?"); for passivity, scaffold with recaps ("To catch you up, we were discussing X — what's your take?").

8. **Build the ReactJS frontend** with these components: a room creation/join view (generates meeting ID, collects participant name), a waiting room (shows joined participants, starts when capacity reached), a chat interface (message list with sender labels, input field, Socket.IO connection), and a feedback panel (displayed after discussion concludes with per-student summaries).

9. **Generate per-student feedback** at discussion end — send the full conversation history plus the student name list to the LLM with a feedback-specific prompt that asks it to analyze each student's participation pattern, identify strengths, and suggest one concrete improvement. Return structured feedback (JSON with student name keys) and render per-student cards in the frontend.

10. **Add configuration and observability** — expose `max_students`, `max_tokens`, and `min_qa_pairs` as environment variables or admin settings. Log moderator latency per response, track participation metrics per session, and store conversation transcripts for post-session review and system refinement.

## Concrete Examples

**Example 1: Building a Reading Comprehension Discussion Platform**

User: "Build a collaborative reading discussion app where an AI moderator leads 4 students through passage-based questions."

Approach:
1. Create a Flask app with Socket.IO, defining a `Room` class tracking `room_id`, `students[]`, `current_passage`, `current_question_idx`, and `responses{}`.
2. Write the moderator system prompt:

```
You are a discussion moderator for a group reading activity.
Students in this session: {student_names}.

Your responsibilities:
1. Welcome students and create an inclusive atmosphere.
2. Present the passage below and read it with the group.
3. Ask questions one at a time from the provided Q&A pairs.
4. After posing a question, ensure EVERY student responds before revealing the answer.
   If a student hasn't responded, prompt them by name: "What do you think, [name]?"
5. Provide constructive feedback on each response without revealing the correct answer.
6. After all students respond, share the correct answer and transition to the next question.
7. Handle disruptions with empathy — acknowledge feelings, then redirect to the topic.

Passage: {passage}
Questions: {questions}
```

3. Implement the retrieval module loading FairytaleQA-format JSON:
```python
import random

def get_discussion_content(dataset, min_qa_pairs=3):
    eligible = [item for item in dataset if len(item["qa_pairs"]) >= min_qa_pairs]
    return random.choice(eligible)
```

4. Wire Socket.IO events: `join_room`, `send_message`, `request_feedback`.
5. On `send_message`, append to conversation history, invoke LangChain chain, broadcast moderator response.

Output: A working multi-user discussion app where the AI moderator sequences through questions, calls on quiet students, and provides feedback.

---

**Example 2: Adding Adaptive Moderation to an Existing Chat App**

User: "I have a Socket.IO chat app for study groups. Add an AI moderator that balances participation and handles disruptive behavior."

Approach:
1. Add a participation tracker to the existing backend:
```python
participation = {}  # {room_id: {student_name: {"count": 0, "last_active": None}}}

def get_participation_summary(room_id):
    stats = participation.get(room_id, {})
    total = sum(s["count"] for s in stats.values())
    summary = []
    for name, s in stats.items():
        pct = (s["count"] / total * 100) if total > 0 else 0
        summary.append(f"{name}: {s['count']} messages ({pct:.0f}%)")
    return "\n".join(summary)
```

2. Inject participation context into the LLM prompt before each moderator turn:
```python
context = f"Participation so far:\n{get_participation_summary(room_id)}\n"
context += "Direct your next question to the least active student."
```

3. Add behavioral intervention rules to the system prompt:
```
Special handling:
- If a student uses aggressive language, respond: "I understand this can be
  frustrating. Let's focus on [topic] — your perspective matters here."
- If a student goes off-topic for 2+ messages, say: "Interesting thought!
  How might that connect to our current question about [topic]?"
- If a student hasn't spoken in 2+ rounds, say: "We'd love to hear from you,
  [name]. To recap, we're discussing [summary]. What are your thoughts?"
```

4. Implement token trimming to keep context manageable:
```python
def trim_history(messages, max_tokens=5000):
    total = sum(len(m["content"].split()) for m in messages)
    while total > max_tokens and len(messages) > 2:
        removed = messages.pop(1)  # keep system prompt at index 0
        total -= len(removed["content"].split())
    return messages
```

Output: The existing chat app now has an AI moderator that tracks who's talking, prompts quiet students, de-escalates conflict, and redirects off-topic drift.

---

**Example 3: Generating Per-Student Feedback Reports**

User: "After a group discussion ends, I want to show each student personalized feedback on their participation."

Approach:
1. Collect the full conversation transcript and student list.
2. Send a feedback generation prompt:
```python
feedback_prompt = f"""Analyze this discussion transcript and generate feedback
for each student. For each student, provide:
- participation_level: "high", "medium", or "low"
- strengths: one specific thing they did well (with quote)
- improvement: one concrete suggestion for next time
- encouragement: a brief motivational note

Students: {student_names}
Transcript:
{transcript}

Respond in JSON format with student names as keys."""
```

3. Parse the JSON response and render per-student feedback cards in the frontend:
```jsx
function FeedbackCard({ name, feedback }) {
  return (
    <div className="feedback-card">
      <h3>{name}</h3>
      <span className={`badge ${feedback.participation_level}`}>
        {feedback.participation_level} participation
      </span>
      <p><strong>Strength:</strong> {feedback.strengths}</p>
      <p><strong>Next time:</strong> {feedback.improvement}</p>
      <p className="encouragement">{feedback.encouragement}</p>
    </div>
  );
}
```

Output: Each student sees a personalized feedback card after the discussion ends, with specific quotes from their contributions and actionable suggestions.

## Best Practices

- **Do** define moderator responsibilities as an explicit numbered list in the system prompt — this produces more predictable turn-taking and question sequencing than vague role descriptions.
- **Do** track participation quantitatively (message count, recency) and inject that data into the prompt context so the LLM makes informed turn-management decisions rather than guessing.
- **Do** trim conversation history to a token window (default 5000) to maintain response latency under 2.5 seconds while preserving enough context for coherent moderation.
- **Do** make the content retrieval layer dataset-agnostic — accept any format with passages and Q&A pairs so the platform works across subjects.
- **Avoid** letting the moderator reveal correct answers before all participants have responded — enforce this explicitly in the system prompt and validate in application logic.
- **Avoid** binary toxicity detection (block/allow) — instead, instruct the moderator to acknowledge the underlying emotion and redirect constructively, which preserves engagement.
- **Avoid** hardcoding participant counts — use `max_students` as a configurable parameter and design the turn-tracking logic to handle variable group sizes.

## Error Handling

- **LLM returns off-format response**: When expecting structured output (JSON feedback), wrap the LLM call in a retry with a stricter prompt ("You must respond in valid JSON only"). Parse with error handling and fall back to a generic feedback template if parsing fails after 2 retries.
- **Token limit exceeded**: If conversation history exceeds `max_tokens`, trim from the oldest non-system messages. Log when trimming occurs so administrators can adjust the threshold.
- **Student disconnects mid-discussion**: Update the participant roster, inject a context message ("Note: [name] has left the discussion"), and adjust the moderator's turn-taking to skip the absent student.
- **Moderator reveals answer prematurely**: Add a post-processing check that compares the moderator's response against known answers before broadcasting. If a match is detected and not all students have responded, regenerate with an appended instruction: "Do not reveal the answer yet."
- **Socket.IO connection drops**: Implement reconnection with session recovery — store conversation state server-side keyed by room ID so reconnecting clients receive the full history.
- **High latency under load**: If response time exceeds 3 seconds, reduce `max_tokens` window or switch to a faster model variant. Monitor p95 latency per room.

## Limitations

- The participation balancing is prompt-driven, not algorithmic — the LLM may not consistently direct questions to the quietest participant despite instructions. For strict fairness guarantees, implement a server-side turn queue that overrides LLM decisions.
- The framework was evaluated with simulated student personas (GPT-driven bots), not real users. Real classroom dynamics — simultaneous messages, varied response times, off-platform distractions — may challenge the sequential turn-taking model.
- Token trimming discards older context, which means the moderator may lose track of earlier contributions in long discussions. For sessions exceeding 30 minutes, consider periodic LLM-generated summaries injected as synthetic context.
- Content retrieval uses random selection by default, which does not account for difficulty progression or prerequisite knowledge. Production deployments should add topic sequencing and difficulty scoring.
- The system assumes a single moderator per room. Scaling to large groups (20+ participants) would require sub-group management or multiple concurrent moderator instances, which the current architecture does not address.

## Reference

Tahir, H., Faisal, F., Alnajjar, F., Taj, M. I., & Gordon, L. (2026). "Dynamic Framework for Collaborative Learning: Leveraging Advanced LLM with Adaptive Feedback Mechanisms." [arXiv:2601.21344](https://arxiv.org/abs/2601.21344v1) | [IEEE](https://ieeexplore.ieee.org/document/11118419). Focus on Section III (System Architecture) for the three-tier modular design, Section IV for the moderator prompt structure and discussion loop, and Appendix B for the full system prompt template.