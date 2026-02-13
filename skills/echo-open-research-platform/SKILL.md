---
name: "echo-open-research-platform"
description: "Build and configure ECHO-style research platforms for running reproducible user studies comparing chat-based AI and web search interactions. Use when: 'set up a user study platform', 'build a chat vs search experiment', 'log participant interactions with LLMs', 'create a research workflow with surveys and tasks', 'export user study interaction traces', 'configure an IRB-compliant experiment with pre/post questionnaires'."
---

# ECHO: Open Research Platform for Evaluating Chat, Human Behavior, and Outcomes

This skill enables Claude to build, configure, and extend research platforms following the ECHO architecture — a serverless, three-layer system for running reproducible mixed-method user studies that compare human interaction with conversational AI (LLMs) and web search engines. ECHO's design integrates consent flows, configurable surveys, chat and search task sessions with fine-grained interaction logging, writing/judgment tasks, and structured data export — all orchestrated through an admin dashboard that requires minimal coding.

## When to Use

- When the user needs to build a web-based platform for running user studies involving chat with LLMs (OpenAI, Claude, Gemini) or web search (Brave, Bing, Google)
- When the user asks to scaffold a research experiment workflow with consent, background surveys, task sessions, and post-task evaluations
- When the user needs fine-grained interaction logging — capturing typing timestamps, click events, search result rankings, chat turn ratings, and survey responses in real time
- When the user wants to create a configurable admin dashboard where researchers can define study flows without writing code
- When the user needs to export structured CSV datasets of participant interactions for downstream analysis
- When the user is building an IRB-compliant participant interface with consent forms, demographic questionnaires, and in-situ popup surveys
- When the user wants to compare information-seeking behavior across chat-based and search-based paradigms in a controlled study

## Key Technique

ECHO uses a **serverless three-layer architecture**: (1) a React + TailwindCSS frontend split into an administrator dashboard and a participant interface, (2) a Firebase backend using Firestore for real-time data storage and Firebase Authentication for user management, and (3) an external API layer that integrates LLM providers (OpenAI, Anthropic Claude, Google Gemini) and search APIs (Brave Search by default). This architecture eliminates the need for dedicated servers or clusters — the entire platform runs with Firebase credentials and API keys.

The core innovation is a **configurable six-stage study workflow** that researchers assemble through the admin dashboard: (1) background survey, (2) pre-task questionnaire, (3) main task (chat or search session), (4) post-task questionnaire, (5) experience survey, (6) end-of-study survey. Administrators can enable, disable, or reorder these stages via drag-and-drop. Survey instruments support four editing modes — View, Reorder, Form (interactive GUI), and JSON (direct import/export) — letting researchers range from no-code to full-config approaches.

What distinguishes ECHO from ad-hoc study setups is its **unified interaction trace logging**. For chat sessions, it captures unique turn identifiers, full text content, typing start/end timestamps, and per-turn user ratings. For search sessions, it captures query text, typing timestamps, result list counts, clicked URLs with timestamps, and result rankings. An in-situ survey system triggers dynamic popup questionnaires based on configurable conditions (after N prompts, after N responses, at intervals, or before task submission), enabling measurement without disrupting the participant's flow.

## Step-by-Step Workflow

1. **Scaffold the three-layer project structure.** Create a monorepo with two React apps (`admin-dashboard/` and `participant-app/`), a shared Firebase configuration module, and an API integration layer. Use `create-react-app` or Vite with TailwindCSS. Initialize Firebase with Firestore and Authentication.

2. **Configure Firebase backend and authentication.** Set up a Firestore database with collections for `studies`, `participants`, `sessions`, `interactions`, and `responses`. Enable Firebase Authentication for participant and administrator login. Store all study configurations as Firestore documents.

3. **Build the administrator dashboard with study workflow configuration.** Implement a drag-and-drop interface for the six default study stages. Each stage should be toggleable (enabled/disabled) and reorderable. Store the stage configuration as a JSON document in Firestore under the study record:
   ```json
   {
     "studyId": "study_001",
     "stages": [
       {"id": "consent", "enabled": true, "order": 0},
       {"id": "background_survey", "enabled": true, "order": 1},
       {"id": "pre_task", "enabled": true, "order": 2},
       {"id": "main_task", "enabled": true, "order": 3, "type": "chat"},
       {"id": "post_task", "enabled": true, "order": 4},
       {"id": "experience_survey", "enabled": true, "order": 5},
       {"id": "end_survey", "enabled": true, "order": 6}
     ],
     "llmProvider": "openai",
     "searchProvider": "brave",
     "inSituTriggers": {"afterPrompts": 3, "beforeSubmission": true}
   }
   ```

4. **Implement the survey configuration system with four editing modes.** Build a survey editor component supporting: (a) View mode for read-only preview, (b) Reorder mode for drag-and-drop question ordering, (c) Form mode with a GUI for adding Likert scales, multiple choice, and open-ended questions, and (d) JSON mode for direct import/export of survey instrument definitions.

5. **Build the participant interface with consent flow.** Create an IRB-compliant consent page with customizable text, acknowledgment checkboxes, and a demographic questionnaire. Route participants through the configured study stages sequentially, storing progress in Firestore so sessions can be resumed.

6. **Implement the chat task interface.** Build a three-panel layout: task description on the left, central conversation area with markdown rendering and code highlighting, and an optional note-taking sidebar on the right. Integrate LLM APIs (OpenAI, Claude, Gemini) with streaming responses. Add an in-situ rating mechanism for each chat turn (e.g., thumbs up/down or Likert scale).

7. **Implement the search task interface.** Mirror the chat layout but replace the conversation area with a search box and SERP display showing titles, URLs, and snippet abstracts. Integrate a search API (Brave Search by default). Log clicked results with their ranking position and timestamps.

8. **Build the interaction trace logger.** Create a logging service that writes to Firestore in real time. For chat: log turn ID, full text, typing start/end timestamps, and user ratings. For search: log query text, typing timestamps, result count, clicked URLs, click timestamps, and result rankings. Attach participant and session IDs to every logged event.

9. **Implement in-situ popup surveys.** Build a survey overlay component triggered by configurable conditions: after N prompts submitted, after N responses received, after N search queries, at timed intervals, or before task submission. Store trigger configuration per study and log all popup responses with timestamps.

10. **Build the data export pipeline.** Add a "View Responses" section to the admin dashboard that queries Firestore and exports structured CSV files for: participant demographics, pre/post-task responses, complete chat histories with timestamps, search queries with click data, in-situ survey responses, and participant notes.

## Concrete Examples

**Example 1: Scaffolding a new ECHO-style research platform**

User: "I need to build a platform for running user studies that compare how people find information using ChatGPT vs Google Search. It needs consent forms, surveys, and interaction logging."

Approach:
1. Create a Vite + React + TailwindCSS monorepo with `apps/admin` and `apps/participant` directories
2. Set up Firebase project with Firestore collections: `studies`, `participants`, `sessions`, `chatInteractions`, `searchInteractions`, `surveyResponses`
3. Build the admin dashboard with a study creation form that lets the researcher configure the LLM provider (OpenAI), search provider (Brave), and enable/disable study stages
4. Build the participant flow: consent -> background survey -> pre-task questionnaire -> chat OR search session -> post-task questionnaire -> end survey
5. Integrate OpenAI streaming API for the chat condition and Brave Search API for the search condition
6. Wire up Firestore logging for all interaction events

Output structure:
```
echo-platform/
  apps/
    admin/           # React admin dashboard
      src/
        components/
          StudyConfigurator.jsx
          SurveyEditor.jsx
          ResponseExporter.jsx
    participant/     # React participant interface
      src/
        components/
          ConsentFlow.jsx
          ChatTask.jsx
          SearchTask.jsx
          SurveyStep.jsx
          InSituPopup.jsx
  shared/
    firebase.js      # Firestore + Auth config
    logger.js        # Interaction trace logger
    apiClients.js    # LLM and Search API wrappers
  firestore.rules
  package.json
```

**Example 2: Adding fine-grained interaction logging to an existing chat interface**

User: "I already have a chat interface that talks to Claude. I need to add ECHO-style interaction logging that captures typing timestamps, turn ratings, and exports to CSV."

Approach:
1. Create a logging service module that writes interaction events to Firestore
2. Instrument the chat input to capture `typingStart` (onFocus or first keydown) and `typingEnd` (on submit) timestamps
3. Add a per-turn rating widget (1-5 stars or thumbs up/down) that logs the rating with the turn ID
4. Build an export endpoint that queries all interactions for a session and writes CSV

Logger implementation pattern:
```javascript
// logger.js
import { collection, addDoc, serverTimestamp } from 'firebase/firestore';

export async function logChatInteraction(db, sessionId, data) {
  await addDoc(collection(db, 'chatInteractions'), {
    sessionId,
    turnId: data.turnId,
    role: data.role,           // 'user' or 'assistant'
    content: data.content,
    typingStartedAt: data.typingStartedAt,
    typingEndedAt: data.typingEndedAt,
    userRating: data.userRating || null,
    createdAt: serverTimestamp(),
  });
}

export async function logSearchInteraction(db, sessionId, data) {
  await addDoc(collection(db, 'searchInteractions'), {
    sessionId,
    queryText: data.queryText,
    typingStartedAt: data.typingStartedAt,
    typingEndedAt: data.typingEndedAt,
    resultCount: data.resultCount,
    clickedUrl: data.clickedUrl || null,
    clickedRank: data.clickedRank || null,
    clickedAt: data.clickedAt || null,
    createdAt: serverTimestamp(),
  });
}
```

**Example 3: Configuring in-situ popup surveys for a within-subjects study**

User: "I'm running a study where each participant does both a chat task and a search task. I need popup surveys that appear after every 3 prompts during chat and after every 2 queries during search, plus a final one before they submit."

Approach:
1. Define trigger configurations per task condition in the study config
2. Build a counter-based trigger system that tracks prompts/queries per session
3. Create a reusable popup survey component that renders configurable questions
4. Log popup responses with the trigger context (which prompt/query number triggered it)

Configuration:
```json
{
  "conditions": {
    "chat": {
      "inSituTriggers": [
        {"type": "afterPrompts", "every": 3, "surveyId": "mid_task_chat"},
        {"type": "beforeSubmission", "surveyId": "final_reflection"}
      ]
    },
    "search": {
      "inSituTriggers": [
        {"type": "afterQueries", "every": 2, "surveyId": "mid_task_search"},
        {"type": "beforeSubmission", "surveyId": "final_reflection"}
      ]
    }
  }
}
```

## Best Practices

- **Do:** Store all study configurations as structured JSON in Firestore so they are versioned, shareable, and reproducible across research teams.
- **Do:** Log interaction events with `serverTimestamp()` rather than client-side timestamps to avoid clock skew across participants.
- **Do:** Design the survey editor to support both GUI (Form mode) and raw JSON editing — researchers with technical skills prefer JSON import/export for replicating instruments across studies.
- **Do:** Use Firebase Authentication with anonymous sign-in for participants to avoid collecting personally identifiable login credentials while still tracking session continuity.
- **Avoid:** Hard-coding LLM or search provider integrations. Use an adapter pattern so providers can be swapped (OpenAI to Claude, Brave to Bing) by changing a config value, not rewriting code.
- **Avoid:** Logging raw API keys or participant email addresses in interaction trace collections. Keep credentials in environment variables and PII in a separate, access-controlled collection.

## Error Handling

- **LLM API failures during chat sessions:** Implement retry with exponential backoff (3 attempts, 1s/2s/4s delays). Display a non-disruptive inline error to the participant ("Response delayed, retrying...") rather than crashing the session. Log the failure event with the error code for analysis.
- **Search API rate limits:** Cache recent search results in Firestore for 5 minutes. If the API returns 429, serve cached results and log the rate-limit event. Alert the administrator dashboard.
- **Firestore write failures on interaction logs:** Buffer interaction events in browser localStorage as a fallback. Retry writes when connectivity resumes. Never silently drop logged events — they are the primary research data.
- **Participant session interruption:** Store session progress (current stage, current task step) in Firestore. On page reload or reconnection, resume from the last completed stage rather than restarting the study.
- **Survey configuration validation:** Validate survey JSON against a schema before saving. Reject configurations with missing question IDs, invalid Likert scale ranges, or duplicate option values. Surface validation errors in the admin dashboard.

## Limitations

- ECHO's serverless Firebase architecture works well for studies with tens to low hundreds of concurrent participants, but may hit Firestore write rate limits (1 write/second per document) under heavy concurrent load. For large-scale studies (1000+ simultaneous participants), consider a dedicated backend with batch writes.
- The platform captures behavioral traces (clicks, timestamps, ratings) but does not natively support physiological or eye-tracking data. Researchers needing gaze data or biometric signals must integrate external tools.
- Chat sessions depend on third-party LLM APIs, so response quality, latency, and availability are outside the platform's control. Model updates by providers can change behavior between study waves.
- The within-platform search experience uses API-returned results, not a full browser-based search. This means participants cannot interact with live web pages from search results, which limits ecological validity for studies of real browsing behavior.
- Survey instruments are self-report measures, subject to the usual limitations of self-report data (social desirability bias, recall errors, satisficing).

## Reference

Liu, J., Dinesh, N., & Yu, R. (2026). ECHO: An Open Research Platform for Evaluation of Chat, Human Behavior, and Outcomes. *SIGIR '26*. [arXiv:2602.10295](https://arxiv.org/abs/2602.10295v1) — Read for the full architecture diagram, Firestore collection schemas, admin dashboard screenshots, and detailed descriptions of the six-stage configurable study workflow with in-situ survey trigger conditions.