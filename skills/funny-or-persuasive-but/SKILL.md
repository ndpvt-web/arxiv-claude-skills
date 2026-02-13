---
name: "funny-or-persuasive-but"
description: "Fine-grained multi-concept text control that avoids the compositionality trap where LLMs degrade when asked to be e.g. funny AND persuasive simultaneously. Use when: 'write a funny persuasive email', 'make this formal but warm', 'generate humorous and convincing copy', 'control tone on two axes at once', 'blend humor with authority in this text', 'write polite but assertive feedback'."
---

# Fine-Grained Multi-Concept Text Control

This skill implements the evaluation-driven decomposition strategy from Labroo et al. (EACL 2026), which demonstrated that LLMs systematically fail at simultaneous multi-attribute text generation. When asked to produce text that is both humorous and persuasive (or formal and warm, or polite and assertive), models degrade on at least one dimension -- even though these concepts are linguistically independent. This skill teaches Claude to detect multi-concept requests, decompose them into sequential single-concept passes, and self-evaluate each dimension on a 1-5 intensity scale to ensure both attributes hit their targets.

## When to Use

- When the user asks for text combining two stylistic attributes (e.g., "write a funny but persuasive product pitch")
- When generating emails, reviews, copy, or arguments that must balance competing tonal qualities (e.g., "formal yet approachable")
- When the user specifies intensity levels for multiple concepts (e.g., "very humorous, slightly persuasive")
- When refining existing text to add a second attribute without losing the first (e.g., "make this funnier without losing the authority")
- When building prompt templates or LLM pipelines that require reliable multi-attribute output
- When the user reports that generated text "lost its humor" or "stopped being convincing" after adding a second style constraint

## Key Technique

**The compositionality trap.** Labroo et al. tested concept pairs -- humor vs. persuasiveness, formality vs. politeness, sentiment polarity -- across Llama, Gemma, and Qwen models on tasks like review generation, email composition, and argument construction. They measured output quality using fine-tuned classifiers and human annotators on a 1-5 Likert scale. The core finding: naive dual-concept prompts (e.g., "Write a humorous AND persuasive review") consistently score lower on one or both dimensions compared to single-concept prompts. The model cannot serve two masters in a single generation pass.

**Sequential decomposition as mitigation.** The paper identifies that explicit decomposition -- handling each concept in a distinct phase rather than a simultaneous instruction -- yields measurably better results. Rather than asking for "funny and persuasive" in one shot, you first generate with one concept dominant, then revise to layer in the second. This exploits the model's strong single-concept control while sidestepping the compositional failure mode.

**Self-evaluation closes the loop.** Each concept is scored on a 1-5 intensity scale after generation. If either dimension falls below the target, a targeted revision pass adjusts only the deficient attribute. This mirrors the paper's evaluation pipeline (automated classifier + human annotation) but made practical for a generation workflow.

## Step-by-Step Workflow

1. **Identify the concept pair.** Parse the user's request to extract exactly which stylistic attributes are being combined. Map informal language to canonical concepts: humor, persuasiveness, formality, politeness, sentiment, warmth, assertiveness, authority.

2. **Assign target intensities.** For each concept, determine the desired intensity on a 1-5 scale. If the user says "very funny, slightly persuasive," map to humor=5, persuasiveness=2. If no intensity is specified, default both to 3 (moderate).

3. **Choose a primary concept.** Select whichever concept is harder to retrofit after the fact. Humor and creativity are harder to add to existing text than persuasiveness or formality. Prioritize the more generative/creative attribute as primary.

4. **Generate the primary-concept draft.** Write the text focusing exclusively on the primary concept at its target intensity. Do not mention or attempt the secondary concept. This ensures the harder attribute is strongly established.

5. **Score the primary concept.** Rate the draft on the 1-5 scale for the primary concept. If it falls below target, revise before proceeding. Do not move to step 6 with a weak primary attribute.

6. **Layer in the secondary concept via targeted revision.** Revise the draft to introduce the secondary concept at its target intensity. The revision instruction must be explicit: "Revise the following text to add [concept] at intensity [N]/5 while preserving the existing [primary concept]." Constrain edits to additive phrasing changes, not structural rewrites.

7. **Score both concepts on the revision.** Rate the revised text on both dimensions independently. Check that neither has dropped below target. If the primary concept degraded during revision, the edit was too aggressive.

8. **Iterative correction if needed.** If one concept is below target, perform a focused micro-edit on only the deficient dimension. Limit to 2 correction passes maximum -- if it still fails, the concept pair may have genuine tension at those intensities (see Limitations).

9. **Verify naturalness.** Read the final text for coherence. Multi-concept text that sounds forced or awkward is worse than slightly missing one target. Flag any jarring juxtapositions to the user.

10. **Present the output with a concept scorecard.** Show the final text alongside a brief 1-5 rating for each attribute so the user can see where each dimension landed and request adjustments.

## Concrete Examples

**Example 1: Humorous and persuasive product review**

```
User: Write a funny but convincing review for a standing desk.

Step 1 - Identify concepts: humor (target: 4/5), persuasiveness (target: 4/5)
Step 2 - Primary concept: humor (harder to retrofit)

Primary draft (humor-focused):
"I bought this standing desk because my doctor said sitting is the new
smoking, and I figured if I'm going to die, I'd rather do it on my feet
like a man. Three months in, my legs have calves now. Actual calves.
I didn't know that was possible after 30."

Score: humor=4/5, persuasiveness=1/5. Primary concept is strong.

Revision pass (adding persuasiveness):
"I bought this standing desk because my doctor said sitting is the new
smoking, and I figured if I'm going to die, I'd rather do it on my feet
like a man. Three months in, my legs have calves now. Actual calves.
I didn't know that was possible after 30. But seriously -- the
pneumatic lift is dead silent, the surface fits two monitors and a
coffee graveyard, and the wobble factor is zero. At $349, it costs less
than the back surgery I was headed toward. My productivity is up because
I pace while thinking, which apparently is what smart people do."

Final score: humor=4/5, persuasiveness=4/5.
Both targets met. Text reads naturally.
```

**Example 2: Formal yet warm rejection email**

```
User: Write a formal but warm email declining a job candidate.

Step 1 - Identify concepts: formality (target: 4/5), warmth (target: 4/5)
Step 2 - Primary concept: warmth (harder to add to formal text)

Primary draft (warmth-focused):
"Hi Jordan, I genuinely enjoyed our conversation last Thursday --
your take on distributed systems was one of the most thoughtful I've
heard this quarter. I want to be straightforward with you: we've
decided to move forward with another candidate whose background
more closely matches the immediate needs of this role. That said,
your skills are strong, and I'd be disappointed if we didn't cross
paths again. I'll reach out personally if something opens up that
fits your profile."

Score: warmth=4/5, formality=2/5. Needs formality lift.

Revision pass (adding formality):
"Dear Jordan, Thank you for taking the time to speak with us last
Thursday. Your perspective on distributed systems architecture was
genuinely one of the most thoughtful I have encountered this quarter.
After careful consideration, we have decided to move forward with
a candidate whose background more closely aligns with the immediate
requirements of the role. I want to be straightforward: this was a
difficult decision. Your technical depth and communication skills
left a strong impression on the team. I would welcome the
opportunity to reconnect should a position that suits your profile
become available, and I will reach out personally if it does."

Final score: warmth=4/5, formality=4/5.
Warmth preserved through specific compliments and personal commitment.
Formality achieved through structure, salutation, and register.
```

**Example 3: Polite but assertive Slack message**

```
User: Help me write a polite but firm message to a coworker who
keeps missing deadlines. Make it assertive enough to actually work.

Step 1 - Identify concepts: politeness (target: 3/5), assertiveness (target: 5/5)
Step 2 - Primary concept: assertiveness (user emphasized "actually work")

Primary draft (assertiveness-focused):
"The Q3 reports were due Friday. This is the third time this quarter
the deadline has been missed. I need the completed reports by end of
day Wednesday. If there's a recurring blocker, we need to address it
now -- not after the next missed deadline."

Score: assertiveness=5/5, politeness=1/5. Too blunt.

Revision pass (adding politeness):
"Hey Sam -- I noticed the Q3 reports didn't land by Friday's deadline,
and I want to flag this because it's the third time this quarter.
I know things get hectic, but I need the completed reports by end of
day Wednesday so the team can stay on track. If something is
consistently blocking you, I'd rather we figure that out together now
than keep running into the same wall. Can we grab 15 minutes tomorrow
to sort it out?"

Final score: politeness=3/5, assertiveness=4/5.
The assertiveness dipped slightly from 5 to 4 -- acceptable trade-off.
The deadline is still concrete, the pattern is named, but the tone
invites collaboration rather than issuing an ultimatum.
```

## Best Practices

- **Do:** Always generate the harder-to-retrofit concept first. Humor, creativity, and emotional warmth are easier to dilute than to inject after the fact.
- **Do:** Use the 1-5 scorecard explicitly. Writing "humor=3, persuasiveness=4" forces honest assessment rather than vague satisfaction.
- **Do:** Keep revision passes surgical. Change phrasing and add sentences; avoid rewriting the entire structure, which destroys the primary concept.
- **Do:** Tell the user when concept targets conflict at the requested intensities. Humor=5 and formality=5 is genuinely difficult; humor=3 and formality=4 is achievable.
- **Avoid:** Jamming both concepts into a single generation prompt. This is the exact failure mode the paper documents. Never write "be very funny and very persuasive" in one instruction.
- **Avoid:** More than 2 correction passes. If the text still misses after two rounds, the concept pair likely has inherent tension at those intensity levels, and the user should be told.

## Error Handling

**Primary concept degrades during secondary pass.** This is the most common failure. Roll back to the primary draft and attempt a lighter touch -- add the secondary concept at a lower intensity (e.g., 2/5 instead of 4/5), then incrementally increase.

**Both concepts score low after revision.** The text has likely been over-edited into incoherence. Discard the revision and restart from step 4 with a fresh primary draft. Do not attempt to patch a broken revision.

**User requests contradictory concepts at high intensity.** Some pairs genuinely conflict at extreme levels (e.g., humor=5 + formality=5, or assertiveness=5 + politeness=5). Acknowledge the tension honestly, propose a feasible intensity split (e.g., humor=4, formality=3), and let the user decide which attribute to prioritize.

**The text sounds mechanical or forced.** Multi-concept text that reads like a checklist is worse than missing one target by a point. Prioritize naturalness. If blending feels artificial, lower one attribute by one level and check if the text flows better.

## Limitations

- **Inherent concept tension.** Some concept pairs have genuine semantic overlap or opposition at high intensities. The paper shows this is not just a prompting failure but a structural property of how LLMs encode stylistic attributes. No decomposition strategy fully resolves humor=5 + formality=5.
- **Diminishing returns beyond two concepts.** This workflow is designed for dual-concept control. Three or more simultaneous attributes (e.g., humorous, persuasive, and formal) compound the compositionality problem exponentially. For three+ concepts, apply the workflow iteratively: establish two, then layer the third.
- **Domain sensitivity.** The paper tested on reviews, emails, stories, and arguments. Highly technical or constrained domains (legal briefs, medical reports) may have narrower tolerance for stylistic variation, making even moderate dual-concept control difficult.
- **Scoring subjectivity.** The 1-5 self-evaluation is inherently approximate. It works well for relative comparison between drafts but should not be treated as a calibrated measurement.

## Reference

Labroo, A., Sheth, I., Raina, V., Ahmed, A., & Fritz, M. (2026). *Funny or Persuasive, but Not Both: Evaluating Fine-Grained Multi-Concept Control in LLMs.* EACL 2026. [arXiv:2601.18483](https://arxiv.org/abs/2601.18483v1). Key takeaway: naive multi-attribute prompts systematically degrade output; sequential single-concept generation with self-evaluation scoring preserves both attributes.