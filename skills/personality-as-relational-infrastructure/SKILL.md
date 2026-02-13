---
name: "personality-as-relational-infrastructure"
description: "Design LLM messaging systems that infuse Big Five personality traits for sustained user engagement. Uses aggregate-exposure personality alignment rather than per-message optimization. Trigger phrases: 'personality-aligned messages', 'BFPT messaging', 'adaptive notification system', 'personality-infused prompts', 'behavior change messaging', 'JITAI system design'"
---

# Personality as Relational Infrastructure

This skill enables Claude to design and implement LLM-powered messaging systems that embed Big Five Personality Traits (BFPT) — Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism — into system-initiated messages for behavior change, notifications, coaching, and sustained user engagement. The core insight, drawn from Hofer et al. (2026), is that personality-based personalization works through **aggregate exposure** (cumulative tone consistency across many messages) rather than per-message optimization. This means the system architecture should prioritize consistent personality alignment across an entire message stream, not perfecting each individual message in isolation.

## When to Use

- When the user is building a notification, coaching, or nudge system that sends repeated messages to users over time (e.g., fitness apps, habit trackers, learning platforms)
- When the user asks to personalize LLM-generated messages using personality profiles or Big Five traits
- When designing JITAI (Just-In-Time Adaptive Intervention) systems that deliver context-aware prompts at opportune moments
- When the user wants to implement prompt strategies (baseline, few-shot, fine-tuned, or RAG) for personality-aligned text generation
- When building A/B testing or evaluation frameworks for personalized messaging systems
- When the user asks how to make AI-generated messages feel more natural, appropriate, or less annoying over repeated exposure
- When implementing Communication Accommodation Theory principles in chatbot or messaging UX

## Key Technique

**Aggregate Exposure over Per-Message Optimization.** The paper tested four LLM generation strategies — baseline prompting, few-shot prompting, fine-tuned models, and retrieval-augmented generation (RAG) — each with and without Big Five Personality Trait infusion. The surprising finding: infusing personality traits into a single message produced no measurable improvement on that specific message's perceived quality (no trial-level effect). However, participants who received a *higher proportion* of personality-informed messages across their entire experience rated the overall message stream as more personalized, more appropriate, and reported less negative affect. This is a person-level exposure effect.

**Communication Accommodation Theory (CAT)** explains why. CAT describes how communicators adjust their style to converge with or diverge from their audience. When an LLM consistently mirrors a user's personality traits across messages — matching the energy level of an extravert, the precision preferences of a conscientious person, or the warmth expectations of an agreeable person — users perceive the system as more attuned to them. This convergence effect is cumulative: it builds through pattern recognition across interactions, not through any single message being noticeably "better."

**Practical implication for system design:** Instead of investing engineering effort into making each message maximally personality-optimized (diminishing returns), invest in ensuring personality alignment is *consistently present* across the message stream. A messaging system that infuses personality traits into 80-100% of its outputs will outperform one that perfectly optimizes 20% of messages. The architecture should treat personality as infrastructure — a persistent system-prompt-level concern — not as a per-request feature.

## Step-by-Step Workflow

1. **Define the user personality model.** Collect or infer Big Five trait scores for each user. Represent each trait on a continuous scale (e.g., 1-5 or 0-100). Store as a structured profile: `{ openness: 4.2, conscientiousness: 3.8, extraversion: 2.1, agreeableness: 4.5, neuroticism: 1.9 }`. If direct assessment is unavailable, infer from user behavior patterns or allow self-reporting via a brief questionnaire (e.g., TIPI-10).

2. **Translate trait scores into communication style directives.** Map each trait dimension to concrete language properties. For example:
   - High Openness → use creative metaphors, varied vocabulary, exploratory framing
   - High Conscientiousness → use specific numbers, structured lists, clear action items
   - High Extraversion → use enthusiastic tone, social references, exclamation energy
   - High Agreeableness → use warm language, collaborative framing, empathetic acknowledgments
   - High Neuroticism → use reassuring tone, minimize pressure language, emphasize safety and incremental progress

3. **Embed personality directives at the system-prompt level.** Place personality alignment instructions in the system prompt (not the user message), ensuring every generation from the model inherits the personality frame. This is the critical architectural decision: personality is infrastructure, not a per-message addon.

4. **Choose a generation strategy based on available resources:**
   - **Baseline prompting**: Include personality directives directly in the system prompt. Lowest cost, easiest to implement. Sufficient for most applications.
   - **Few-shot prompting**: Provide 3-5 example messages that exemplify the target personality style in the prompt. Better style consistency.
   - **Fine-tuned model**: Train on a corpus of messages labeled by personality style. Best consistency but highest upfront cost.
   - **RAG**: Retrieve personality-matched example messages from a vector store at generation time. Good balance of quality and flexibility.

5. **Design the JITAI message schema.** Each message needs: (a) situational context (time of day, user activity state, recent behavior), (b) intervention goal (encourage, remind, celebrate, redirect), (c) personality-aligned tone directives from step 2. Structure this as a generation request template.

6. **Ensure high personality-infusion coverage.** Based on the paper's findings, target 80-100% of messages in the stream to carry personality alignment. Do not selectively apply personality only to "important" messages — the effect is cumulative and depends on proportion of exposure.

7. **Implement message variation to prevent fatigue.** Within the personality-consistent frame, vary message structure, length, and specific content. Personality alignment should constrain *tone and style*, not make messages repetitive.

8. **Build evaluation around aggregate metrics, not per-message scores.** Measure perceived personalization, appropriateness, and affective response at the session/week level, not per individual message. Use within-between decomposition: separate whether a specific message scores well (trial-level) from whether the user's overall experience improves (person-level).

9. **Implement A/B testing with personality-infusion proportion as the variable.** Compare groups receiving 0%, 50%, and 100% personality-infused messages. Measure person-level outcomes: overall perceived personalization, message appropriateness, and user affect over the exposure period.

10. **Iterate on trait-to-style mappings using CAT principles.** Monitor for over-accommodation (style feels patronizing) or under-accommodation (style feels generic). Adjust the intensity of personality expression based on user feedback signals.

## Concrete Examples

**Example 1: Fitness App Notification System**

User: "I'm building a fitness app that sends daily motivation messages. I want to personalize them based on user personality. How should I architect this?"

Approach:
1. Define a personality profile schema stored per user
2. Create a system prompt template with personality slot injection
3. Generate messages with personality as persistent infrastructure

Output — System prompt template:
```
You are a fitness coaching assistant. You send one motivational message
per day to help the user stay active.

USER PERSONALITY PROFILE:
- Openness: {openness}/5 — {"Use creative analogies and novel framing" if >= 3.5 else "Use straightforward, familiar language"}
- Conscientiousness: {conscientiousness}/5 — {"Include specific metrics, times, and action steps" if >= 3.5 else "Keep it flexible and low-pressure"}
- Extraversion: {extraversion}/5 — {"Use energetic, enthusiastic tone with social references" if >= 3.5 else "Use calm, reflective tone focused on personal growth"}
- Agreeableness: {agreeableness}/5 — {"Use warm, supportive language and collaborative framing" if >= 3.5 else "Use direct, results-focused language"}
- Neuroticism: {neuroticism}/5 — {"Emphasize safety, self-compassion, and small wins" if >= 3.5 else "Use confident, challenge-oriented framing"}

CONTEXT: {time_of_day}, user's last activity was {days_since_active} days ago,
current streak: {streak_days} days.

Generate a short motivational message (1-3 sentences). Match the personality
style consistently. Do not mention personality traits directly.
```

Sample output for a user with high Conscientiousness (4.5) and low Extraversion (1.8):
```
Your 3pm window is open — a 20-minute walk would put you at 7,200 steps
for the day. You've hit your target 4 of the last 5 days. Quiet consistency
is building something real.
```

Sample output for a user with high Extraversion (4.7) and high Agreeableness (4.3):
```
Hey! Your walking crew hit the park yesterday — maybe today's your day to
join them? Even a quick 15 minutes together would be awesome. You've got this!
```

**Example 2: Learning Platform Nudge System**

User: "We have a language learning app. Users drop off after week 2. Can we use personality-based messaging to improve retention?"

Approach:
1. Collect Big Five scores during onboarding (10-item TIPI questionnaire)
2. Build a message generation pipeline with personality as system-level config
3. Ensure all automated messages carry personality alignment (not just special ones)
4. Track retention at the cohort level grouped by personality-infusion proportion

Output — Implementation architecture:
```python
# Personality-infused message generation pipeline
from dataclasses import dataclass

@dataclass
class PersonalityProfile:
    openness: float        # 1.0 - 5.0
    conscientiousness: float
    extraversion: float
    agreeableness: float
    neuroticism: float

def build_style_directives(profile: PersonalityProfile) -> str:
    """Translate trait scores into concrete language style instructions."""
    directives = []

    if profile.openness >= 3.5:
        directives.append("Use creative metaphors and playful language.")
    else:
        directives.append("Use clear, literal explanations.")

    if profile.conscientiousness >= 3.5:
        directives.append("Include progress stats and specific next steps.")
    else:
        directives.append("Keep it casual; avoid rigid structure.")

    if profile.extraversion >= 3.5:
        directives.append("Reference community, leaderboards, social features.")
    else:
        directives.append("Focus on personal mastery and quiet achievement.")

    if profile.agreeableness >= 3.5:
        directives.append("Use encouraging, empathetic tone.")
    else:
        directives.append("Use matter-of-fact, efficient tone.")

    if profile.neuroticism >= 3.5:
        directives.append(
            "Normalize setbacks. Emphasize that any progress counts."
        )
    else:
        directives.append("Use confident, forward-looking framing.")

    return " ".join(directives)

def generate_nudge(profile: PersonalityProfile, context: dict) -> str:
    """Generate a personality-aligned nudge message.

    Key design decision: personality directives live in the system prompt,
    ensuring EVERY message carries the alignment. This is infrastructure,
    not a per-message feature.
    """
    style = build_style_directives(profile)

    system_prompt = (
        f"You write short retention nudges for a language learning app. "
        f"Style: {style} "
        f"Keep messages to 1-2 sentences. Never mention personality."
    )

    user_prompt = (
        f"User context: learning {context['language']}, "
        f"current streak: {context['streak']} days, "
        f"last session: {context['days_inactive']} days ago, "
        f"skill level: {context['level']}. "
        f"Generate one nudge message."
    )

    return call_llm(system_prompt=system_prompt, user_prompt=user_prompt)
```

**Example 3: Evaluating a Personality-Infused Messaging System**

User: "How should I measure whether personality-based messages are working?"

Approach:
1. Do NOT evaluate individual messages in isolation — the effect is aggregate
2. Set up within-between measurement framework
3. Compare cohorts by personality-infusion proportion

Output — Evaluation framework:
```
WRONG approach (per-message evaluation):
  "Rate this message on a scale of 1-5 for personalization."
  → This will show NO significant effect (confirmed by the paper).

RIGHT approach (aggregate exposure evaluation):
  After N messages over T days, measure:

  1. Perceived Personalization (person-level):
     "Overall, how well did the messages match your communication style?"
     Scale: 1 (not at all) to 7 (extremely well)

  2. Perceived Appropriateness (person-level):
     "Overall, how appropriate were the messages you received?"
     Scale: 1 (not at all) to 7 (extremely)

  3. Affective Response (person-level):
     "How did the messages make you feel overall?"
     Scale: 1 (very negative) to 7 (very positive)

  Comparison groups:
  - Control: 0% personality-infused messages
  - Partial: 50% personality-infused messages
  - Full: 100% personality-infused messages

  Analysis: Compare person-level means across groups.
  Expected: Full > Partial > Control on all three metrics.
```

## Best Practices

- **Do:** Place personality alignment in the system prompt so it persists across all messages by default. Treat it as infrastructure, not a feature toggle.
- **Do:** Map personality traits to concrete language behaviors (word choice, sentence structure, framing), not to vague "be more X" instructions.
- **Do:** Maintain high personality-infusion coverage (80%+ of messages). The effect depends on cumulative proportion, not individual message quality.
- **Do:** Evaluate at the session/week level, not per-message. Use within-between decomposition to separate trial-level noise from person-level signal.
- **Avoid:** Over-accommodating — exaggerating personality style to the point of caricature (e.g., excessive exclamation marks for extraverts). Subtle, consistent alignment beats heavy-handed single-message personalization.
- **Avoid:** Selectively applying personality only to "high-stakes" messages while sending generic messages the rest of the time. This breaks the aggregate exposure pattern that drives the effect.

## Error Handling

- **No personality data available:** Fall back to a neutral, moderately warm baseline style (mid-range on all five traits). Offer an onboarding questionnaire (TIPI-10 takes under 2 minutes) or infer traits from behavioral signals over time.
- **Conflicting trait combinations:** Some profiles create tension (e.g., high Neuroticism + high Extraversion). Prioritize the trait most relevant to the message context: Neuroticism when addressing setbacks, Extraversion when suggesting social activities.
- **User reports messages feel "off":** This likely indicates over-accommodation. Reduce the intensity of personality expression by moving trait thresholds closer to the midpoint (e.g., only apply strong trait-specific language at scores above 4.0 instead of 3.5).
- **Message fatigue despite personality alignment:** Personality alignment constrains tone, not content. Increase content variation (different topics, framings, information types) while maintaining the personality-consistent style layer.
- **Evaluation shows no effect:** Verify that you are measuring at the person-level (aggregate), not the trial-level (per-message). The paper found zero trial-level effects but significant person-level effects. If you are already measuring correctly, check personality-infusion coverage — the effect requires consistent exposure.

## Limitations

- The paper's evidence comes from a **retrospective evaluation**, not a longitudinal in-situ study. Real-world deployment may produce different effect sizes.
- The study used **physical activity** as the application domain. Generalization to other domains (finance, education, mental health) is plausible but unvalidated.
- Big Five traits are **broad dimensions**. Finer-grained personality models (e.g., HEXACO, or domain-specific preference profiles) might yield stronger effects but lack the same empirical backing from this study.
- The approach assumes users have **relatively stable personality profiles**. For contexts where user state changes rapidly (crisis support, acute illness), state-based personalization may be more appropriate than trait-based.
- The study tested with **90 participants**. Effect sizes were meaningful but the sample limits generalizability, particularly for underrepresented personality profiles.

## Reference

Hofer, D. P., Haag, D., Islambouli, R., & Smeddinck, J. D. (2026). Personality as Relational Infrastructure: User Perceptions of Personality-Trait-Infused LLM Messaging. *arXiv:2602.06596v1*. https://arxiv.org/abs/2602.06596v1

Key takeaway: Personality-based personalization works through **aggregate exposure proportion**, not per-message optimization. Look for the within-between decomposition results (Section on ordinal multilevel models) and the Communication Accommodation Theory analysis.