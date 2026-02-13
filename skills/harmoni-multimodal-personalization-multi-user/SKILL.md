---
name: "harmoni-multimodal-personalization-multi-user"
description: "Build multi-user personalization pipelines with per-user profile tracking, multimodal perception, and LLM-driven contextual response generation following the HARMONI architecture. Use when: 'build a multi-user chatbot with memory', 'personalize responses per user', 'track user profiles across sessions', 'add speaker identification to my app', 'create a system that remembers each user differently', 'build an assistive robot conversation system'."
---

# HARMONI: Multi-User Personalization with LLM-Driven User Modeling

This skill teaches Claude to design and implement multi-user personalization systems based on the HARMONI framework. HARMONI decomposes personalized interaction into four cooperating modules -- perception, world modeling, user modeling, and generation -- enabling applications where multiple users interact with a single system and each receives contextually grounded, memory-aware, ethically filtered responses. The core insight is that separating user identity resolution, profile maintenance, environmental context, and response generation into distinct pipeline stages yields dramatically better personalization than monolithic LLM prompting.

## When to Use

- When building a chatbot or assistant that must track and personalize for multiple distinct users across sessions
- When adding persistent user profiles (preferences, history, demographic features) to an LLM-powered application
- When implementing speaker identification or user resolution in a multimodal (video, audio, text) pipeline
- When designing a system that must update user knowledge incrementally from each conversation turn
- When the user asks to build a nursing home assistant, reception bot, or any socially assistive system serving repeat visitors
- When adding ethical guardrails (PII filtering, safety constraints) to a personalized response pipeline
- When the user needs a FastAPI-based conversation backend with per-user memory and session management

## Key Technique

HARMONI's central contribution is a **four-module pipeline** that processes each interaction turn through distinct stages rather than stuffing everything into a single LLM prompt. At time `t`, the system constructs an observation tuple `o = (query, user_profile, world_state)` and generates a response `r ~ policy(o)` while simultaneously updating the user profile `p_{t+1} ~ update_policy(p_t, query)`. This parallel generation-and-update pattern keeps latency low while ensuring the profile is always current.

**User modeling** is the differentiating module. Each user has a persistent profile containing: facial/biometric embeddings for re-identification, demographic attributes, affective state history, extracted facts from past interactions, and semantic embeddings of prior conversations. Profile retrieval uses cosine similarity between the current user's embedding and stored profiles (using models like InsightFace for faces or sentence transformers for text). Profile *updates* happen via a secondary LLM call that extracts new facts from the current turn and merges them into the existing profile -- this is the "online memory updating" that outperforms static RAG approaches.

**Ethical filtering** is embedded in the generation module rather than applied as a post-hoc filter. The generation prompt includes explicit instructions to avoid surfacing PII, to respect user-defined privacy preferences stored in their profile, and to apply safety constraints. This "ethics-by-design" approach scored 82.4/100 on the System Usability Scale in real-world nursing home trials.

## Step-by-Step Workflow

1. **Define the user profile schema.** Create a data model with fields for: unique ID, display name, embedding vector (for re-identification), demographic attributes (age, gender -- optional), extracted facts (list of strings), conversation history summary, affective state, privacy preferences, and timestamps. Store as JSON documents or database rows.

2. **Implement the perception/identity resolution layer.** For text-only systems, resolve identity via session tokens or login. For multimodal systems, use face detection (YOLOv8-Face), active speaker detection (lip motion via 68-point facial landmarks), and facial encoding (InsightFace buffalo_l) to match against the user database. For audio, transcribe with Whisper and attribute to the identified speaker.

3. **Build the user profile retrieval function.** Given an identified user (by embedding or ID), fetch their profile from the database. Use semantic similarity search (cosine distance on embeddings from a model like `EmbeddingGemma-300m` or `all-MiniLM-L6-v2`) to retrieve the most relevant facts from their history relative to the current query.

4. **Implement the world modeling context builder.** Maintain a short-term session buffer (last N turns of conversation) and a long-term environment state (location, time, other users present). Combine these into a `world_state` string that gets injected into the generation prompt.

5. **Build the parallel generation-and-update pipeline.** On each turn, fire two concurrent LLM calls: (a) the response generator, which receives `(query, user_profile_summary, world_state)` and produces the reply; (b) the profile updater, which receives `(current_profile, query, response)` and returns a JSON patch of new facts to merge into the profile.

6. **Construct the generation prompt with ethical guardrails.** The system prompt must include: the user's profile summary, the world context, explicit instructions to never reveal raw PII, instructions to respect the user's stored privacy preferences, and safety constraints appropriate to the domain (e.g., medical advice boundaries for healthcare settings).

7. **Implement the profile merge/update logic.** Take the extracted facts from the updater LLM and merge them into the stored profile: append new facts, update changed attributes, increment interaction counters, and update the last-seen timestamp. Use deduplication (semantic similarity between new and existing facts) to avoid profile bloat.

8. **Wire the pipeline into a FastAPI application.** Create endpoints for: `POST /interact` (accepts user ID + query, returns response), `GET /users/{id}/profile` (returns current profile for transparency), and `POST /users/identify` (accepts an image/audio clip, returns matched user ID). Use WebSocket for real-time video/audio streaming if needed.

9. **Add observability and explainability.** Log each turn's observation tuple, the retrieved profile subset, and the generated response. Expose updated profiles through the UI so operators can verify the system's understanding of each user.

10. **Evaluate with the HARMONI metrics.** Measure user modeling accuracy (do extracted facts match ground truth?), personalization quality (LLM-as-judge scoring using a separate model like Mixtral-8x7B), dialogue coherence (ROUGE-1/2/L between turns), and user satisfaction (SUS questionnaire for live deployments).

## Concrete Examples

**Example 1: Multi-user chatbot with persistent memory**

User: "Build me a FastAPI chatbot that remembers different users and personalizes responses based on their history."

Approach:
1. Define a `UserProfile` Pydantic model with fields: `id`, `name`, `facts` (list of strings), `preferences` (dict), `history_summary` (str), `last_seen` (datetime).
2. Use SQLite + JSON columns (or a vector DB like ChromaDB) for profile storage.
3. On each `/interact` call, retrieve the user profile, build the observation tuple, and run two parallel LLM calls.
4. Return the response and persist the updated profile.

Output:
```python
from pydantic import BaseModel
from datetime import datetime

class UserProfile(BaseModel):
    id: str
    name: str
    facts: list[str] = []          # "prefers formal tone", "allergic to peanuts"
    preferences: dict = {}          # {"language": "en", "privacy": "strict"}
    history_summary: str = ""       # rolling summary of past interactions
    affective_state: str = "neutral"
    last_seen: datetime = None

class ObservationTuple(BaseModel):
    query: str
    user_profile: UserProfile
    world_state: dict               # {"time": "morning", "location": "lobby", "other_users": ["Alice"]}

# Generation prompt template
GENERATION_SYSTEM_PROMPT = """You are a personalized assistant. Use the user profile to tailor your response.
User profile: {profile_summary}
Context: {world_state}
Rules:
- Never reveal raw personal data back to the user
- Respect privacy level: {privacy_level}
- If the user has expressed preferences, follow them
- Do not make assumptions beyond what is in the profile"""

# Profile update prompt template
UPDATE_SYSTEM_PROMPT = """Extract new facts about the user from this conversation turn.
Current profile facts: {existing_facts}
User said: {query}
Assistant replied: {response}
Return a JSON object: {"new_facts": [...], "updated_preferences": {}, "affective_state": "..."}
Only include genuinely new information. Do not repeat existing facts."""
```

**Example 2: Adding speaker identification to a video conferencing assistant**

User: "I have a video feed with multiple people. I want to identify who is speaking and maintain separate conversation threads per person."

Approach:
1. Use YOLOv8-Face for face detection on each frame.
2. Use dlib's 68-point face landmarks to detect lip motion and identify the active speaker.
3. Encode detected faces with InsightFace and match against stored embeddings using cosine similarity (threshold 0.6).
4. Route the transcribed speech (Whisper) to the matched user's profile.
5. Maintain per-user conversation buffers and generate personalized responses.

Output:
```python
import numpy as np
from insightface.app import FaceAnalysis

face_app = FaceAnalysis(name="buffalo_l", providers=["CPUExecutionProvider"])
face_app.prepare(ctx_id=0)

def identify_speaker(frame: np.ndarray, user_db: dict[str, np.ndarray]) -> str | None:
    """Match a face in the frame against known user embeddings."""
    faces = face_app.get(frame)
    if not faces:
        return None
    # Select the face with highest lip motion score (active speaker)
    active_face = select_active_speaker(faces)  # uses 68-point landmarks
    embedding = active_face.embedding
    best_match, best_score = None, 0.0
    for user_id, stored_emb in user_db.items():
        score = np.dot(embedding, stored_emb) / (
            np.linalg.norm(embedding) * np.linalg.norm(stored_emb)
        )
        if score > best_score:
            best_match, best_score = user_id, score
    return best_match if best_score > 0.6 else None
```

**Example 3: Ethical personalization for a healthcare reception system**

User: "Build an intake assistant for a clinic that remembers returning patients but strictly controls what information it reveals."

Approach:
1. Define privacy tiers in the user profile: `"strict"` (never reference past visits aloud), `"moderate"` (acknowledge returning status only), `"open"` (can reference specific past interactions).
2. Store medical-adjacent facts separately with an access control flag.
3. In the generation prompt, inject the privacy tier and enforce it with explicit instructions.
4. Run a post-generation PII filter that strips any names, dates of birth, or medical terms that shouldn't be spoken aloud.

Output:
```python
HEALTHCARE_SYSTEM_PROMPT = """You are a clinic reception assistant.
Patient privacy level: {privacy_level}

STRICT RULES:
- privacy=strict: Do NOT reference any past visits or stored information. Treat as new visitor.
- privacy=moderate: You may say "Welcome back" but do not mention specifics.
- privacy=open: You may reference past appointment types and preferences.
- NEVER speak aloud: diagnoses, medications, insurance details, dates of birth.
- If unsure whether information is safe to share, do not share it.

Patient profile (for internal use only, do NOT read aloud):
{profile_summary}

Current context: {world_state}
"""
```

## Best Practices

- **Do:** Store user profiles as structured documents (JSON/Pydantic models) with explicit field schemas -- not as unstructured text blobs. This enables targeted updates and retrieval.
- **Do:** Run profile updates in parallel with response generation to keep latency low. The update does not need to complete before the response is sent.
- **Do:** Implement fact deduplication using semantic similarity (cosine > 0.85 threshold) before appending new facts to a profile. Profiles grow unbounded otherwise.
- **Do:** Include the user's privacy preferences directly in the generation prompt as hard constraints, not suggestions.
- **Avoid:** Stuffing the entire user profile into the LLM context. Retrieve only the top-k most relevant facts (by semantic similarity to the current query) to stay within token limits.
- **Avoid:** Using a single monolithic prompt that handles identification, memory, and generation simultaneously. The pipeline decomposition is what makes HARMONI outperform naive approaches.
- **Avoid:** Storing raw conversation transcripts as the user profile. Always extract and summarize facts -- raw transcripts consume storage and degrade retrieval quality.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| User not recognized (new user or low confidence match) | Cosine similarity below threshold (< 0.6) | Create a new profile; optionally ask for name/confirmation |
| Profile retrieval returns empty or stale data | Check `last_seen` timestamp or empty `facts` list | Fall back to generic (non-personalized) response; log for review |
| Profile update LLM returns malformed JSON | JSON parse error on updater response | Retry once with stricter prompt; skip update for this turn if retry fails |
| Ethical filter triggers (PII detected in response) | Regex or NER scan on generated text | Regenerate with stronger constraints; log the violation |
| Multiple speakers detected simultaneously | More than one active lip motion detected | Queue speakers; process the highest-confidence speaker first |
| Profile bloat (hundreds of facts) | Fact count exceeds configured threshold | Run periodic summarization to compress old facts into a condensed summary |

## Limitations

- **Speaker identification requires visual input.** Text-only systems must rely on session tokens or explicit login -- the multimodal perception module cannot function without camera/audio feeds.
- **Profile quality depends on LLM extraction accuracy.** If the updater LLM hallucinates facts or misattributes statements in group conversations, the profile degrades over time. Regular human review of profiles is recommended for high-stakes domains.
- **Latency scales with user count.** Embedding similarity search against a large user database (>1000 users) requires a vector index (FAISS, ChromaDB) rather than brute-force comparison.
- **Ethical guardrails are prompt-based, not provably safe.** The system relies on LLM instruction-following for PII filtering. For regulated domains (healthcare, finance), add a deterministic post-processing filter as a second layer.
- **The framework assumes cooperative users.** Adversarial scenarios (impersonation, deliberate misinformation) are not addressed. Add verification steps for sensitive operations.

## Reference

[HARMONI: Multimodal Personalization of Multi-User Human-Robot Interactions with LLMs](https://arxiv.org/abs/2601.19839v1) -- Focus on Section 3 (framework architecture), Section 4 (the observation-tuple formulation and parallel update mechanism), and Section 5 (evaluation metrics for user modeling accuracy and personalization quality). Reference implementation: [github.com/hamedR96/HARMONI](https://github.com/hamedR96/HARMONI).