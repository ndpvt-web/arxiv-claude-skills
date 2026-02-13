---
name: "do-reasoning-ask-questions"
description: "Information-theoretic question-asking framework for disambiguating user intent through structured yes/no questions. Uses a three-agent architecture (Seeker, Oracle, Pruner) grounded in Shannon entropy to maximize information gain per question turn. Trigger phrases: 'clarify ambiguous request', 'ask better questions', 'narrow down requirements', 'disambiguate user intent', 'information-gathering dialogue', 'structured requirements elicitation'."
---

# Information-Theoretic Question-Asking for Requirement Disambiguation

This skill enables Claude to systematically resolve ambiguity in user requests by asking maximally informative yes/no questions, modeled on the Seeker-Oracle-Pruner triad from Pedrozo et al. (2026). Instead of asking vague open-ended questions or making assumptions, Claude constructs a hypothesis space of plausible user intents, computes which binary question would eliminate the most candidates (maximizing information gain in bits), asks that question, prunes the space based on the answer, and repeats until the intent is unambiguous. This transforms requirement gathering from guesswork into a principled search process.

## When to Use

- When a user's request maps to multiple valid implementations and you need to determine which one they want (e.g., "add authentication" could mean session-based, JWT, OAuth, or magic links).
- When debugging a reported issue and the bug description is ambiguous -- systematically narrow down the root cause through targeted diagnostic questions.
- When a user asks to "build a feature" but hasn't specified key design decisions (data model, API shape, UI behavior, error handling strategy).
- When refactoring code and there are multiple valid target architectures -- use binary questions to converge on the user's preferred outcome.
- When a configuration or deployment request has many possible environments, settings, or constraints that the user hasn't specified.
- When triaging a large set of possible solutions and you need to efficiently identify the user's constraints before proposing one.

## Key Technique

The core insight from the paper is that **asking questions should be treated as an information-theoretic optimization problem**. Given a set of N plausible interpretations of a user's request (the hypothesis space), the current uncertainty is H = log2(N) bits. The optimal yes/no question is the one that splits the hypothesis space as close to 50/50 as possible, yielding ~1 bit of information gain per turn. This is the same principle behind binary search, but applied to intent disambiguation.

The framework uses three cooperating roles. The **Seeker** generates candidate yes/no questions and selects the one with highest expected information gain. The **Oracle** (the user, in our case) answers truthfully. The **Pruner** eliminates hypotheses inconsistent with the answer, shrinking the candidate space. The key behavioral finding: reasoning through candidates explicitly (chain-of-thought) dramatically improves question quality -- models with CoT achieved 0.93 win rates versus 0.10 without it in partially observable settings, where the questioner cannot see the full hypothesis space and must infer it from conversation history alone.

A practical takeaway for coding agents: **generate multiple candidate questions before selecting one**, and explicitly reason about how many hypotheses each question would eliminate. Smaller models in the study compensated for limited capacity by exploring ~9.7 candidate questions per turn versus ~7.6 for larger models, but larger models selected higher-IG candidates. The lesson: always brainstorm several possible questions, then pick the one that maximally partitions the remaining possibilities.

## Step-by-Step Workflow

1. **Enumerate the hypothesis space.** Upon receiving an ambiguous request, list all plausible interpretations. Structure them hierarchically if possible (like the paper's Region > Country > State > City taxonomy). For a coding task, this might be: framework choice > architecture pattern > specific library > configuration variant.

2. **Compute current entropy.** Count the number of live hypotheses N. Your current uncertainty is log2(N) bits. This tells you roughly how many perfect binary questions you need (e.g., 16 hypotheses = 4 questions minimum).

3. **Generate 5-10 candidate yes/no questions.** Each question should be answerable with yes or no. Think about which dimension of the hypothesis space has the most variance -- target that dimension first. Prefer questions about categories (high-level splits) before specific instances.

4. **Score each candidate by expected information gain.** For each question, estimate how many hypotheses would remain under "yes" vs "no." The ideal question splits the space 50/50. Compute expected IG = H_before - (p_yes * H_after_yes + p_no * H_after_no), where p_yes and p_no are the fraction of hypotheses consistent with each answer.

5. **Ask the highest-IG question.** Present it clearly to the user as a yes/no question. Avoid compound questions -- one binary decision per turn.

6. **Prune the hypothesis space.** Based on the user's answer, eliminate all inconsistent hypotheses. Update N and recompute entropy.

7. **Check termination condition.** If only one hypothesis remains (or the remaining hypotheses are equivalent for implementation purposes), proceed to implementation. If entropy is still high, return to step 3.

8. **Summarize what you've learned.** Before implementing, state back to the user the specific interpretation you've converged on, giving them a chance to correct any misunderstanding.

9. **Implement with confidence.** With ambiguity resolved, proceed with the specific implementation. Reference the disambiguation decisions in code comments only where they affect non-obvious design choices.

10. **Track cumulative information gain.** If the user provides new requirements mid-implementation that reintroduce ambiguity, repeat the process on the new hypothesis space rather than guessing.

## Concrete Examples

**Example 1: Disambiguating an Authentication Feature Request**

```
User: "Add authentication to my Express app"

Step 1 - Enumerate hypothesis space (8 candidates):
  Auth method: [session-based, JWT, OAuth2, magic-link]
  Storage: [database, in-memory]
  = 8 combinations, H = 3.0 bits

Step 2 - Generate candidate questions, scored by IG:
  Q1: "Do you need token-based auth (JWT/OAuth) rather than
       session-based?" → splits 4/4 → IG ≈ 1.0 bit
  Q2: "Do you need third-party login (Google, GitHub, etc.)?"
       → splits 2/6 → IG ≈ 0.81 bits
  Q3: "Is this for an API (no browser sessions) or a web app
       with browser sessions?" → splits 4/4 → IG ≈ 1.0 bit
  Q4: "Do you need persistent sessions across server restarts?"
       → splits 4/4 → IG ≈ 1.0 bit

Step 3 - Ask Q3 (tied for highest, and most intuitive):
  Claude: "Is this for a stateless API (consumed by mobile/SPA
  clients) or a traditional web app with server-side sessions?"
  User: "It's an API for a React frontend."

Step 4 - Prune: eliminate session-based options → 4 remaining
  [JWT+db, JWT+memory, OAuth2+db, OAuth2+memory], H = 2.0 bits

Step 5 - Next best question:
  Claude: "Do users need to log in via third-party providers
  like Google or GitHub?"
  User: "No, just email and password."

Step 6 - Prune: eliminate OAuth2 → 2 remaining, H = 1.0 bit
  Claude: "Should auth tokens survive server restarts (stored in
  a database) or is in-memory sufficient for development?"
  User: "Database, this is for production."

Step 7 - Converged: JWT + database storage for email/password auth.
  Total questions: 3, Total IG: 3.0 bits (optimal for 8 hypotheses).

Step 8 - Confirm and implement:
  Claude: "I'll implement JWT authentication with email/password,
  storing refresh tokens in your database. I'll add login, register,
  and token refresh endpoints to your Express app."
```

**Example 2: Debugging an Ambiguous Bug Report**

```
User: "The app crashes when users submit the form"

Step 1 - Enumerate hypothesis space:
  Crash type: [uncaught exception, infinite loop, OOM, network timeout]
  Trigger: [all inputs, specific input patterns, race condition, auth-expired]
  Location: [client-side, server-side, database]
  = ~48 combinations, H ≈ 5.6 bits

Step 2 - Start with highest-level split:
  Claude: "Does the crash happen on the client side (browser error/
  white screen) or server side (5xx response)?"
  User: "Server side, we get a 500 error."
  → Prune client-side hypotheses, ~16 remaining, H ≈ 4.0 bits

  Claude: "Does it happen for every form submission, or only for
  certain inputs?"
  User: "Only when the description field is very long."
  → Prune to input-dependent server errors, ~4 remaining, H ≈ 2.0 bits

  Claude: "Does your server log show a database error or an
  application-level exception?"
  User: "Database error -- something about packet size."
  → Converged: database packet size limit on large text input.

Result: 3 questions to pinpoint root cause from 48 possibilities.
  Claude proceeds to check max_allowed_packet or equivalent config
  and the column type (VARCHAR vs TEXT).
```

**Example 3: Narrowing Down a Refactoring Target**

```
User: "Refactor the data layer to be cleaner"

Step 1 - Hypothesis space:
  Target: [separate concerns, reduce duplication, improve types,
           add caching, switch ORM, add repository pattern]
  Scope: [single module, full data layer]
  = 12 combinations, H ≈ 3.6 bits

Step 2 - Questions:
  Claude: "Is the main pain point about code organization (files/
  modules are tangled) or about runtime behavior (performance,
  error handling)?"
  User: "Organization -- everything is in one huge file."

  Claude: "Do you want to keep the current data access approach
  (raw queries / current ORM) and just restructure the files, or
  also change how data is accessed?"
  User: "Just restructure, keep the queries as-is."

  → Converged in 2 questions: split monolithic data file into
    separate modules by domain concern, preserving existing queries.
```

## Best Practices

- **Do:** Structure your hypothesis space hierarchically. High-level category questions first (framework vs. library, client vs. server), then drill into specifics. This mirrors the paper's five-level taxonomy and maximizes early information gain.
- **Do:** Generate multiple candidate questions internally before selecting one. Explicitly reason about the split ratio each question achieves. A 50/50 split is ideal; a 90/10 split wastes a turn.
- **Do:** Keep questions strictly binary (yes/no). Compound questions ("Is it A or B, and does it also do C?") muddle the information gain calculation and confuse users.
- **Do:** Track your running hypothesis count and entropy. State it internally so you know how many more questions you need. If you have 2 hypotheses left, you need exactly 1 more question -- don't ask 3.
- **Avoid:** Asking open-ended questions when binary ones would be more efficient. "What framework do you want?" has unbounded answer space. "Are you using React?" eliminates half the possibilities cleanly.
- **Avoid:** Asking about implementation details before resolving high-level architectural ambiguity. Don't ask "Should the button be blue or green?" when you haven't established whether there's a button at all.
- **Avoid:** Asking more than 5 questions in a row without providing value. If the hypothesis space is too large (>32 candidates), offer your best-guess interpretation with a brief rationale and ask the user to correct it, rather than running a 5+ question interrogation.

## Error Handling

- **User gives ambiguous answers to yes/no questions** (e.g., "sort of" or "it depends"): Treat this as a signal that your question was poorly framed. Rephrase to target a more concrete, observable distinction. Split the ambiguous answer into two new sub-hypotheses and continue.
- **Hypothesis space was incomplete** (user's actual intent wasn't in your initial list): When an answer eliminates ALL remaining hypotheses, acknowledge the gap, ask one open-ended question to discover the missing category, rebuild the hypothesis space, and resume binary questioning.
- **User gets impatient with questions**: If the user signals they want you to "just do it," pick the most likely hypothesis based on codebase context and available evidence. State your assumption explicitly so they can correct if wrong.
- **Contradictory answers**: If a new answer conflicts with a previous one, flag the contradiction to the user. Re-ask the conflicting question with the context of both answers to resolve it.

## Limitations

- This technique is most valuable when the hypothesis space has 4+ genuinely distinct alternatives. For requests with only 2 plausible interpretations, a single direct question is faster than the full framework.
- Binary questions work poorly for continuous parameters (e.g., "how many items should the pagination show?"). Use direct questions for numerical or free-text values.
- The information gain calculation assumes roughly uniform priors over hypotheses. If your codebase strongly suggests one interpretation (e.g., the project already uses JWT everywhere), weight your priors accordingly and skip questions whose answers are near-certain.
- In partially observable settings (you haven't read the full codebase), your hypothesis space may be incomplete. Read relevant code before constructing the hypothesis space to avoid missing the correct interpretation entirely.
- This approach adds conversational overhead. For trivial or unambiguous requests, skip it entirely and proceed directly to implementation.

## Reference

Pedrozo, D. M., Soares, T. W. de L., & de Oliveira, B. L. M. (2026). *Do Reasoning Models Ask Better Questions? A Formal Information-Theoretic Analysis on Multi-Turn LLM Games.* [arXiv:2601.17716](https://arxiv.org/abs/2601.17716v1). Key takeaway: Explicit chain-of-thought reasoning over candidate questions dramatically improves disambiguation efficiency (0.93 vs 0.10 win rate), and the optimal strategy is to generate multiple candidate questions, score them by expected information gain (Shannon entropy reduction), and select the one that most evenly splits the hypothesis space.