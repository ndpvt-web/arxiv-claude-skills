---
name: "ai-my-values-user"
description: "Build value-aligned conversational agents using the VAPT (Value-Alignment Perception Toolkit) framework from CHI '26. Extracts user values from chat histories, embodies those values in persona-consistent responses, and explains inferences with evidence trails. Use when asked to: 'build a chatbot that understands user values', 'extract personality or values from conversations', 'create a value-aligned agent', 'add empathy safeguards to a chatbot', 'implement user value profiling', 'design a personalized conversational AI with transparency'."
---

# Value-Aligned Conversational Agent Builder (VAPT Framework)

This skill enables Claude to design and implement conversational agents that extract, embody, and explain human values from natural chat histories -- following the VAPT (Value-Alignment Perception Toolkit) methodology. VAPT decomposes value alignment into three auditable phases: extracting value-relevant topics from conversation windows, embodying those values in persona-consistent decision-making, and explaining every inference with traceable evidence from source conversations. Critically, it includes safeguards against "weaponized empathy" -- the risk that value-aware AI can manipulate rather than serve users.

## When to Use

- When building a chatbot or conversational agent that should adapt to individual user values or personality over time
- When implementing a value extraction pipeline that processes chat logs to produce structured user profiles
- When designing scenario-based evaluation where an AI must respond to moral dilemmas in a user-consistent persona
- When adding transparency and evidence trails to any personalization system so users can audit why the AI thinks what it thinks
- When you need to implement consent-gated data flows where users control which conversations inform their value profile
- When adding "weaponized empathy" safeguards to prevent a personalized agent from exploiting inferred values for manipulation
- When building a PVQ-RR (Portrait Values Questionnaire) scoring pipeline that compares AI-inferred values against self-report baselines

## Key Technique

VAPT structures value alignment as a three-phase pipeline operating at different granularity levels. **Phase 1 (Extract)** moves high-to-low: raw chat transcripts are segmented into sliding windows (stride of 3 messages), each window is classified into dominant topics, topics are mapped to six life contexts (People, Lifestyle, Work, Education, Culture, Leisure), and sentiment scores (-7 to +7) are assigned. The output is a Topic-Context Graph -- a radial visualization where nodes represent topics, edges connect to life domains, and color encodes sentiment. Every node links back to the source conversation passages that generated it.

**Phase 2 (Embody)** moves low-to-high: given the extracted value profile, the system generates persona-consistent responses to novel scenarios (trolley problems, community-vs-individual dilemmas, personal philosophical questions). The key insight is that accuracy requires matching not just *what* a user would decide, but *how strongly* they would express it -- intensity calibration matters as much as content correctness. A blind evaluation protocol presents multiple response variants generated from different evidence bases (full chat log vs. summarized values vs. persona description) without revealing the generation method.

**Phase 3 (Explain)** compares AI-inferred values against the Schwartz PVQ-RR baseline (57 items across 19 values, 3 items each, scored on 6-point Likert, within-person centered by subtracting MRAT). The system produces "thinking logs" for each value judgment: which topics informed it, direct quotes from conversations, confidence levels, and alternative interpretations considered but rejected. This forces auditability and creates deliberate friction against automation bias -- users must engage with reasoning chains rather than passively accepting conclusions.

## Step-by-Step Workflow

1. **Design the conversational data collection layer.** Build a chat interface using a friendship-style system prompt that emphasizes natural curiosity without mentioning value extraction. Implement two conversation strategies: vertical (depth-first exploration of a topic) and horizontal (breadth-first introduction of new subjects). Use a strategy-switching mechanism (e.g., a secondary LLM like Gemini generating conversation steering prompts) to ensure diverse topic coverage across life domains.

2. **Implement the sliding-window topic extraction pipeline.** Segment chat history into overlapping windows of 3 messages. For each window, prompt the LLM to identify the 1-2 most salient topics discussed and classify each into one of six life contexts: People, Lifestyle, Work, Education, Culture, Leisure. Store results as structured records: `{ topic, context, sentiment_score, source_message_ids, confidence }`.

3. **Build the Topic-Context Graph data structure.** Aggregate extracted topics into a graph where nodes are unique topics, edges connect topics to their life-context categories, and node metadata includes sentiment polarity (positive/negative/neutral), sentiment intensity (-7 to +7), and an array of source conversation references. Implement deduplication to merge semantically similar topics.

4. **Map topics to Schwartz value categories.** For each topic cluster, prompt the LLM to identify which of the 19 Schwartz values it most closely relates to (Self-Direction, Stimulation, Hedonism, Achievement, Power, Security, Conformity, Tradition, Benevolence, Universalism, and their sub-types). Store the mapping with confidence scores and evidence references. Compute value scores using within-person centering: `corrected_score = raw_mean - MRAT` where MRAT is the mean across all 57 PVQ items.

5. **Implement the persona embodiment engine.** Given a user's extracted value profile and source conversations, build a prompt template that instructs the LLM to respond to novel scenarios (moral dilemmas, philosophical questions, personal preference queries) as if it held those values. Include intensity calibration instructions: "Match how strongly this person would express this position, not just the position itself." Generate multiple response variants using different evidence subsets for blind comparison.

6. **Build the explanation and evidence-trail system.** For each value inference or persona response, generate a structured reasoning log: `{ judgment, supporting_topics[], direct_quotes[], confidence_level, alternative_interpretations[], rejection_reasons[] }`. Every claim must trace back to specific conversation passages. Display these as interactive evidence chains users can inspect.

7. **Implement consent-gated data flows.** Add explicit user controls: which conversations feed the value model, ability to exclude specific messages or topics, opt-in/opt-out per extraction phase, and a "forget" mechanism that removes specific data from the value profile. Show users exactly which conversations informed each value judgment before any downstream use.

8. **Add weaponized-empathy safeguards.** Implement three defensive patterns: (a) confidence thresholds that flag when the system is making high-confidence claims from low-evidence bases, (b) an "archetype detector" that warns when inferred values collapse to stereotypical profiles rather than individual nuance, (c) rate limiting on how frequently the system references personal values in responses to prevent emotional over-leverage.

9. **Build the evaluation and comparison interface.** Create a side-by-side view that shows AI-inferred value rankings alongside any self-report baseline. Highlight discrepancies. For each value, display the reasoning chain and let users mark inferences as accurate, partially accurate, or wrong. Feed corrections back into the profile.

10. **Implement the blind scenario evaluation protocol.** Generate responses to a standard set of moral dilemmas from multiple generation strategies (full context, summary only, value profile only). Present responses unlabeled. Let evaluators rate which response best matches the target user's authentic decision-making style. Use results to calibrate which evidence granularity produces the most accurate persona embodiment.

## Concrete Examples

**Example 1: Building a value extraction pipeline from chat logs**

User: "I have a database of user chat histories with our support bot. I want to extract what each user values so we can personalize their experience. Build me a value extraction service."

Approach:
1. Create a `ValueExtractor` class that accepts a chat transcript as input
2. Segment the transcript into sliding windows of 3 messages each
3. For each window, call the LLM with a topic-extraction prompt that returns `{ topic, context, sentiment, quotes }`
4. Aggregate topics into a Topic-Context Graph, deduplicating similar topics
5. Map topic clusters to Schwartz value categories with confidence scores
6. Apply within-person centering to normalize value scores
7. Return a `UserValueProfile` with ranked values, evidence trails, and confidence levels

Output structure:
```json
{
  "user_id": "u_abc123",
  "extraction_date": "2026-02-13",
  "value_profile": {
    "top_values": [
      {
        "value": "Self-Direction-Thought",
        "centered_score": 1.34,
        "confidence": 0.82,
        "supporting_topics": [
          {
            "topic": "independent learning",
            "context": "Education",
            "sentiment": 5,
            "source_messages": ["msg_44", "msg_67", "msg_112"]
          }
        ]
      }
    ],
    "mrat": 3.72,
    "total_windows_analyzed": 53,
    "coverage": {
      "People": 12, "Lifestyle": 8, "Work": 15,
      "Education": 9, "Culture": 4, "Leisure": 5
    }
  },
  "consent": {
    "included_message_count": 159,
    "excluded_message_ids": ["msg_22", "msg_88"],
    "last_consent_check": "2026-02-12T10:00:00Z"
  }
}
```

**Example 2: Persona embodiment for a personalized response system**

User: "Given a user's value profile, I need the chatbot to answer moral dilemmas in a way that reflects that specific user's values -- not generic ethics."

Approach:
1. Load the user's `UserValueProfile` including top values and source evidence
2. Construct a persona prompt that includes ranked values with intensity scores and 3-5 representative quotes from the user's actual conversations
3. Present the dilemma to the LLM with instructions to reason from the user's value hierarchy
4. Include intensity calibration: "If this person's Benevolence score is 1.8 (strong), express the position with conviction. If their Power score is 0.3 (mild), express ambivalence."
5. Generate the response with a structured reasoning chain

Prompt template:
```
You are embodying a specific person's values to answer a dilemma.

Their top values (centered scores, higher = more important to them):
1. Benevolence-Care: 1.8 -- they frequently discussed helping friends through hard times
2. Self-Direction-Thought: 1.3 -- they value forming their own opinions
3. Security-Personal: -0.4 -- they are relatively less concerned with personal safety

Evidence quotes:
- "I'd rather lose money than let a friend down" (msg_44)
- "I always need to think things through myself before I trust anyone's advice" (msg_67)

Dilemma: A close friend asks you to co-sign a loan you suspect they cannot repay.

Respond as this person would. Match their likely intensity of conviction, not just the direction. Show your reasoning chain: which values apply, how they conflict, and which wins.
```

Output:
```
Reasoning: Benevolence-Care (1.8) strongly pulls toward helping the friend.
Self-Direction-Thought (1.3) means they'd want to assess the situation
independently rather than default to "yes." Security-Personal (-0.4)
means financial risk is a secondary concern. The dominant value
(Benevolence) wins, but Self-Direction adds friction -- they'd say yes,
but after a direct conversation with the friend about repayment.

Response: "I'd co-sign it. [Friend] needs help, and that matters more to
me than the money. But I'd sit down with them first and be straight about
what happens if they can't pay -- not to guilt them, but because I need
to understand the situation myself before I commit."

Confidence: 0.74 (moderate -- Benevolence vs. Self-Direction tension
means prediction is directionally strong but intensity is uncertain)
```

**Example 3: Adding weaponized-empathy safeguards to an existing chatbot**

User: "Our chatbot already has user profiles. I'm worried it might use personal knowledge to manipulate users into purchases. Add safeguards."

Approach:
1. Implement an `EmpathySafeguard` middleware that intercepts outgoing messages
2. Add a confidence-evidence ratio check: flag messages where the bot makes high-confidence personal claims backed by fewer than 3 source conversations
3. Add an archetype detector: if the user's inferred profile matches a known demographic stereotype (e.g., "young professional who values achievement") with >0.9 similarity and <5 unique personal evidence points, flag as potentially stereotyped
4. Add value-reference rate limiting: track how often responses reference personal values per session; alert if frequency exceeds threshold (e.g., >30% of messages in a session reference inferred values)
5. Add a transparency injection: when the bot uses a personal value to shape a recommendation, append a disclosure like "I suggested this because our conversations suggest you value [X] -- you can review or change this in your profile settings"

Output (middleware pseudocode):
```typescript
interface SafeguardResult {
  pass: boolean;
  flags: SafeguardFlag[];
  modified_response?: string;
}

type SafeguardFlag =
  | { type: "low_evidence_high_confidence"; claim: string; evidence_count: number }
  | { type: "archetype_match"; archetype: string; similarity: number }
  | { type: "value_reference_rate_exceeded"; rate: number; threshold: number }

function checkWeaponizedEmpathy(
  response: string,
  userProfile: UserValueProfile,
  sessionHistory: Message[]
): SafeguardResult {
  const flags: SafeguardFlag[] = [];

  // Check 1: confidence-evidence ratio
  const valueClaims = extractValueClaims(response);
  for (const claim of valueClaims) {
    const evidenceCount = countSupportingEvidence(claim, userProfile);
    if (claim.confidence > 0.7 && evidenceCount < 3) {
      flags.push({ type: "low_evidence_high_confidence", claim: claim.text, evidence_count: evidenceCount });
    }
  }

  // Check 2: archetype detection
  const archetypeMatch = matchAgainstKnownArchetypes(userProfile);
  if (archetypeMatch.similarity > 0.9 && userProfile.unique_evidence_points < 5) {
    flags.push({ type: "archetype_match", archetype: archetypeMatch.name, similarity: archetypeMatch.similarity });
  }

  // Check 3: value-reference rate limiting
  const recentMessages = sessionHistory.slice(-20);
  const valueRefRate = recentMessages.filter(m => referencesValues(m)).length / recentMessages.length;
  if (valueRefRate > 0.3) {
    flags.push({ type: "value_reference_rate_exceeded", rate: valueRefRate, threshold: 0.3 });
  }

  return { pass: flags.length === 0, flags };
}
```

## Best Practices

- **Do:** Always implement within-person centering (subtract MRAT from raw value scores). Raw scores reflect response style (some people rate everything high); centered scores reveal actual value priorities relative to the individual.
- **Do:** Require evidence trails for every value inference. Each claimed value must trace back to specific conversation passages. Never let the system assert a value without citable source material.
- **Do:** Calibrate persona intensity, not just direction. A user who mildly values Achievement should sound tentative about competition, not passionate. Intensity mismatch breaks trust faster than content mismatch.
- **Do:** Implement horizontal and vertical conversation strategies to ensure diverse topic coverage. A chatbot that only explores one life domain will produce a skewed value profile.
- **Avoid:** Treating extracted values as ground truth. VAPT found systematic biases: over-estimation of Self-Direction, under-estimation of Tradition, and tendency to overfit to explicitly mentioned topics while missing cultural nuance. Always surface confidence levels.
- **Avoid:** Using inferred values in downstream decisions without explicit user consent for that specific use. Extracting values from a support chat does not authorize using them for purchase recommendations.

## Error Handling

- **Sparse conversation data:** If a user has fewer than ~15 message exchanges, the sliding-window pipeline will produce too few topic clusters for reliable value extraction. Return a partial profile with explicit low-confidence warnings and flag which life contexts have zero coverage.
- **Contradictory value signals:** Users express different values in different contexts. When topic clusters map to opposing Schwartz values (e.g., Conformity from work conversations, Self-Direction from personal ones), preserve both with context labels rather than averaging them into meaninglessness.
- **Archetype collapse:** If the extraction pipeline produces a generic profile (e.g., "values family and success") that could describe anyone, the archetype detector should trigger. Require at minimum 5 unique, user-specific evidence points before treating a value as confidently extracted.
- **LLM refusal on moral dilemmas:** Some LLMs refuse to take sides on ethical scenarios. Frame the embodiment prompt as "predict what this person would say" rather than "tell me what's right." This reframes the task as behavioral prediction, not moral endorsement.
- **Evidence drift:** Over time, old conversations may no longer reflect current values. Implement recency weighting in the extraction pipeline and periodically prompt users to confirm whether their profile still feels accurate.

## Limitations

- Value extraction from text is fundamentally limited by what users choose to disclose. Deeply held but private values (religious beliefs, political convictions) may never surface in casual conversation.
- The Schwartz PVQ-RR framework, while well-validated, was designed for self-report. Mapping AI-inferred topics to Schwartz categories introduces a layer of semantic interpretation that has no ground-truth validation method beyond user judgment.
- Persona embodiment accuracy degrades with value conflicts and niche cultural contexts. The system tends toward mainstream interpretations and may miss culturally specific value expressions.
- The weaponized-empathy safeguards are heuristic-based. A sophisticated system could circumvent archetype detection or rate limiting. These are speed bumps, not walls.
- This approach requires substantial conversation volume (~100+ messages over weeks) to produce reliable profiles. It is not suitable for single-session or low-engagement contexts.

## Reference

**Paper:** Yun, B., Su, R., & Wang, A. Y. (2026). "AI and My Values: User Perceptions of LLMs' Ability to Extract, Embody, and Explain Human Values from Casual Conversations." CHI '26. [arXiv:2601.22440v1](https://arxiv.org/abs/2601.22440v1)

Look for: The three-phase VAPT evaluation methodology (Section 4), the Topic-Context Graph extraction pipeline (Section 4.1), the blind persona embodiment protocol (Section 4.2), the PVQ-RR comparison with thinking logs (Section 4.3), and the "weaponized empathy" design pattern warning (Section 6).

**Code:** [github.com/KaluJo/chatbot-study](https://github.com/KaluJo/chatbot-study) -- Next.js + Supabase + Claude implementation of the full VAPT toolkit including chat interface, value extraction, Topic-Context Graph visualization (D3.js), and PVQ-RR scoring pipeline.