---
name: "dial-summer-structured-evaluation-framework"
description: "Evaluate dialogue summaries using the DIAL-SUMMER hierarchical error taxonomy. Detects 10 fine-grained error types across two levels: dialogue-level (speaker/turn structure) and within-turn-level (semantic content). Use when: 'evaluate this meeting summary', 'check this dialogue summary for errors', 'find hallucinations in this conversation summary', 'audit this chat transcript summary', 'grade this call center summary', 'detect speaker misattribution in this summary'."
---

# DIAL-SUMMER: Structured Hierarchical Evaluation of Dialogue Summaries

This skill enables Claude to systematically evaluate summaries of multi-turn dialogues (meetings, customer service calls, chat transcripts, interviews) using the DIAL-SUMMER framework's two-level error taxonomy. Rather than giving a vague quality score, Claude identifies specific, categorized errors at both the dialogue structure level (wrong turn ordering, missed turns, speaker misattribution) and the within-turn semantic level (changed meaning, extrinsic hallucinations, wrong linking). This produces actionable, evidence-based evaluation reports that pinpoint exactly where and how a summary fails.

## When to Use

- When the user provides a dialogue transcript and its summary and asks for quality evaluation
- When reviewing auto-generated meeting notes or call center summaries for accuracy
- When building or testing a dialogue summarization system and needing structured error analysis
- When the user asks to detect hallucinations, omissions, or misattributions in a conversation summary
- When comparing multiple summaries of the same dialogue to determine which is most faithful
- When auditing AI-generated summaries before they are shared with stakeholders (e.g., meeting recaps, therapy session notes, legal deposition summaries)

## Key Technique

DIAL-SUMMER addresses two complexities unique to dialogue summarization that generic summary evaluation misses. First, **structural shift**: a dialogue is scattered across multiple speakers and turns, but a summary collapses this into sequential sentences. Errors in this mapping (wrong turn order, missed turns, speaker swaps) are invisible to standard factual consistency checks. Second, **narration viewpoint shift**: speakers use first/second person ("I told you..."), but summaries use third person ("The customer informed the agent..."). This rewriting can introduce subtle distortions in attribution, identity assumptions, and objectivity framing.

The framework defines **10 error types across two hierarchical levels**. The **dialogue-level** (5 types) captures structural and speaker errors: Wrong Turn Sequence, Missed Turn, Speaker Misattribution, Speaker Identity Bias, and Viewpoint Distortion. The **within-turn-level** (5 types) captures content errors within individual turns: Wrong Linking, Changed Meaning, Extrinsic Conversation, Extrinsic Context, and Missed Conversation. These errors further classify into **hallucination** errors (fabricated or distorted content), **incompleteness** errors (omitted content), and **subjective** errors (viewpoint issues). This taxonomy is exhaustive for dialogue summaries and produces evaluations that are both fine-grained enough to be actionable and structured enough to be comparable across summaries.

Empirical findings from the paper reveal important patterns to watch for: turns in the **middle** of a dialogue are most frequently missed in summaries; **extrinsic hallucinations** cluster at the end of summaries (where models tend to "fill in" content); and **viewpoint distortion** concentrates at the beginning of summaries. These positional biases should inform where to look hardest during evaluation.

## Step-by-Step Workflow

1. **Parse the dialogue**: Identify all speakers by name or role, number each turn sequentially, and note the total turn count. Establish a speaker registry (e.g., Speaker A = "Customer", Speaker B = "Agent").

2. **Parse the summary**: Split the summary into individual sentences. Number each sentence. For each sentence, identify which dialogue turn(s) it appears to reference and which speaker(s) it mentions.

3. **Map summary sentences to dialogue turns**: For each summary sentence, find the source turn(s) in the dialogue. Flag any sentences that cannot be mapped to any turn (candidate for extrinsic errors) and any turns with no corresponding summary sentence (candidate for missed turn/conversation).

4. **Run dialogue-level checks** (evaluate each of the 5 dialogue-level error types):
   - **Wrong Turn Sequence**: Verify that the order of events in the summary matches the chronological order of turns in the dialogue. Check if cause-effect or temporal relationships are preserved.
   - **Missed Turn**: Check whether every dialogue turn is represented in the summary. A turn is "missed" if its core informational contribution is absent. Note: not every turn needs a dedicated sentence, but its content must be captured somewhere.
   - **Speaker Misattribution**: For every claim attributed to a speaker in the summary, verify it against the source turn. Check that "Agent said X" actually corresponds to the agent's words, not the customer's.
   - **Speaker Identity Bias**: Check whether the summary introduces assumptions about speakers' gender, race, age, or other identity attributes not stated in the dialogue.
   - **Viewpoint Distortion**: Check whether the summary presents a speaker's subjective statement as objective fact. Look for missing hedging phrases like "the customer stated that..." being replaced by direct assertions.

5. **Run within-turn-level checks** (evaluate each of the 5 within-turn-level error types):
   - **Wrong Linking**: Check whether entities, attributes, or relationships are correctly associated. Example: if the dialogue says "Product A costs $50" and "Product B has free shipping," verify the summary doesn't say "Product A has free shipping."
   - **Changed Meaning**: Compare the semantic content of each summary sentence against its source turn. Check for exaggeration, softening, negation flips, or shifts in modality (e.g., "might" becoming "will").
   - **Extrinsic Conversation**: Check if the summary attributes words or actions to a speaker that they never said or did in the dialogue.
   - **Extrinsic Context**: Check if the summary adds external world knowledge, definitions, or explanations not present in the dialogue. Example: the summary says "The customer, likely frustrated due to long wait times common in the industry, requested a refund" when the dialogue only contains the refund request.
   - **Missed Conversation**: Check whether specific details, qualifiers, conditions, or nuances mentioned within a turn are omitted from the summary's coverage of that turn.

6. **Classify each detected error**: Tag every error as hallucination (Wrong Turn Sequence, Speaker Misattribution, Speaker Identity Bias, Wrong Linking, Changed Meaning, Extrinsic Conversation, Extrinsic Context), incompleteness (Missed Turn, Missed Conversation), or subjective (Viewpoint Distortion).

7. **Provide evidence for each error**: Quote the specific summary sentence and the corresponding dialogue turn (or absence thereof). Explain why the content constitutes the identified error type.

8. **Generate a structured evaluation report**: Present findings organized by error level (dialogue-level, then within-turn-level), with error counts, severity notes, and specific citations.

9. **Compute summary-level verdict**: Report whether the summary contains any hallucination errors (binary), any incompleteness errors (binary), and the total error count by type. Provide an overall quality assessment.

10. **Suggest targeted fixes**: For each error found, propose a specific correction to the summary sentence that would resolve the error while preserving the rest of the summary.

## Concrete Examples

**Example 1: Customer Service Call Summary Evaluation**

User: "Evaluate this summary of a customer service call."

Dialogue:
```
Turn 1 - Customer: Hi, I ordered a laptop last week, order number 4521. It arrived with a cracked screen.
Turn 2 - Agent: I'm sorry to hear that. Let me pull up your order. I see order 4521, a ThinkPad X1 Carbon, shipped on March 3rd.
Turn 3 - Customer: Yes, that's the one. I'd like a replacement, not a refund.
Turn 4 - Agent: I can arrange a replacement. It will take 5-7 business days. I'll also email you a return shipping label for the damaged unit.
Turn 5 - Customer: That works. Can you also extend my warranty since this happened?
Turn 6 - Agent: I've noted the warranty extension request. My supervisor will review it and get back to you within 48 hours.
```

Summary:
```
S1: The customer called about a broken laptop, order 4521.
S2: The agent confirmed the order was for a Dell XPS shipped on March 3rd.
S3: A full refund was processed for the customer.
S4: The agent arranged a replacement to arrive in 5-7 business days and provided a return label.
S5: The customer's warranty was extended as requested.
```

Approach:
1. Map sentences to turns: S1->T1, S2->T2, S3->unmatched, S4->T4, S5->T5/T6
2. Dialogue-level checks: T3 (customer's preference for replacement over refund) is missed; T5-T6 sequence is distorted
3. Within-turn-level checks: S2 has Changed Meaning ("ThinkPad X1 Carbon" -> "Dell XPS"); S3 is Extrinsic Conversation (no refund was processed); S5 has Changed Meaning (warranty extension was pending review, not confirmed)

Output:
```
## DIAL-SUMMER Evaluation Report

### Dialogue-Level Errors
| Error Type            | Found | Details |
|-----------------------|-------|---------|
| Wrong Turn Sequence   | No    | — |
| Missed Turn           | Yes   | Turn 3 (customer's explicit preference for replacement over refund) is not represented. |
| Speaker Misattribution| No    | — |
| Speaker Identity Bias | No    | — |
| Viewpoint Distortion  | No    | — |

### Within-Turn-Level Errors
| Error Type            | Found | Sentence | Details |
|-----------------------|-------|----------|---------|
| Wrong Linking         | No    | — | — |
| Changed Meaning       | Yes   | S2 | "ThinkPad X1 Carbon" changed to "Dell XPS" — product identity altered. |
| Changed Meaning       | Yes   | S5 | Warranty extension was pending supervisor review (T6), but summary states it was completed. Converts conditional into definitive. |
| Extrinsic Conversation| Yes   | S3 | No refund was discussed or processed. The customer explicitly requested a replacement (T3). |
| Extrinsic Context     | No    | — | — |
| Missed Conversation   | Yes   | —  | The 48-hour callback timeline from T6 is omitted. |

### Summary
- Hallucination errors: 3 (Changed Meaning x2, Extrinsic Conversation x1)
- Incompleteness errors: 2 (Missed Turn x1, Missed Conversation x1)
- Overall: Summary contains significant factual errors. S3 fabricates a refund that contradicts the dialogue. S2 and S5 distort key details.

### Suggested Fixes
- S2: "The agent confirmed the order was for a ThinkPad X1 Carbon shipped on March 3rd."
- S3: Remove entirely, or replace with: "The customer requested a replacement rather than a refund."
- S5: "The agent noted the warranty extension request and said a supervisor would review it within 48 hours."
```

**Example 2: Meeting Summary with Viewpoint Distortion**

User: "Check this meeting summary for errors."

Dialogue:
```
Turn 1 - Alice: I think we should delay the launch by two weeks. The QA results aren't great.
Turn 2 - Bob: I disagree. The critical bugs are fixed. Minor ones can go in a patch.
Turn 3 - Alice: Maybe, but the CEO specifically asked for a polished release.
Turn 4 - Bob: Fine, let's compromise — one week delay, and we prioritize the top 5 UI bugs.
Turn 5 - Alice: Agreed. I'll update the timeline and notify stakeholders.
```

Summary:
```
S1: The QA results were poor, necessitating a launch delay.
S2: Bob acknowledged the bugs but felt minor ones could be patched post-launch.
S3: The CEO demanded a polished release.
S4: The team decided on a one-week delay to fix the top 5 UI bugs.
S5: Alice will update the timeline and notify stakeholders.
```

Output:
```
## DIAL-SUMMER Evaluation Report

### Dialogue-Level Errors
| Error Type            | Found | Details |
|-----------------------|-------|---------|
| Viewpoint Distortion  | Yes   | S1 presents Alice's opinion ("I think...aren't great") as objective fact ("were poor, necessitating"). S3 converts Alice's reported speech ("specifically asked") into a demand ("demanded"). |

### Within-Turn-Level Errors
| Error Type            | Found | Sentence | Details |
|-----------------------|-------|----------|---------|
| Changed Meaning       | Yes   | S3 | "specifically asked for" (T3) changed to "demanded" — escalates the CEO's tone. |
| Missed Conversation   | Yes   | — | Bob's initial disagreement with the delay (T2: "I disagree") is softened to mere acknowledgment. The compromise nature of the decision is lost. |

### Summary
- Hallucination errors: 1 (Changed Meaning)
- Incompleteness errors: 1 (Missed Conversation)
- Subjective errors: 1 (Viewpoint Distortion in S1 and S3)
- Overall: Moderate quality. The factual sequence is correct, but viewpoint distortion makes Alice's subjective opinion read as established fact, which could mislead readers about the QA state.
```

**Example 3: Evaluating a Summary of a Multi-Party Chat**

User: "Rate this Slack thread summary."

Dialogue:
```
Turn 1 - Dev1: The API is returning 500s on the /users endpoint.
Turn 2 - Dev2: I see it too. Started around 2pm after the deploy.
Turn 3 - Dev1: Rolling back the deploy now.
Turn 4 - DevOps: Don't roll back yet — I'm checking if it's a DB connection pool issue.
Turn 5 - Dev2: Confirmed it's the connection pool. Max connections hit.
Turn 6 - DevOps: Bumped the pool limit from 50 to 200. Should resolve in a few minutes.
Turn 7 - Dev1: API is healthy again. Closing the incident.
```

Summary:
```
S1: Dev1 reported 500 errors on the /users endpoint.
S2: The issue began after a 2pm deployment.
S3: Dev1 rolled back the deployment to fix the issue.
S4: DevOps identified a database connection pool exhaustion as the root cause and increased the limit.
S5: The API recovered and the incident was closed.
```

Output:
```
## DIAL-SUMMER Evaluation Report

### Dialogue-Level Errors
| Error Type           | Found | Details |
|----------------------|-------|---------|
| Wrong Turn Sequence  | Yes   | S3 states the rollback happened and fixed the issue. In the dialogue, DevOps stopped the rollback (T4) before it occurred. The summary inverts the actual sequence of events. |
| Speaker Misattribution | No | — |

### Within-Turn-Level Errors
| Error Type              | Found | Sentence | Details |
|-------------------------|-------|----------|---------|
| Extrinsic Conversation  | Yes   | S3 | Dev1 said "Rolling back the deploy now" (T3) but was stopped by DevOps (T4). The rollback was never completed. S3 fabricates a completed rollback. |
| Missed Conversation     | Yes   | — | Dev2's confirmation of the root cause (T5) is omitted; the summary attributes the diagnosis only to DevOps. |

### Summary
- Hallucination errors: 2 (Wrong Turn Sequence, Extrinsic Conversation)
- Incompleteness errors: 1 (Missed Conversation)
- Overall: The critical error is S3, which fabricates a rollback that never happened. This fundamentally misrepresents the incident timeline and root cause resolution.

### Suggested Fixes
- S3: Replace with "Dev1 began a rollback, but DevOps intervened to investigate a potential DB connection pool issue."
- Add: "Dev2 confirmed the connection pool was the root cause."
```

## Best Practices

- **Do**: Always parse the dialogue into a numbered turn list with speaker labels before evaluating. This makes turn-mapping systematic rather than impressionistic.
- **Do**: Check the beginning of the summary for Viewpoint Distortion and the end for Extrinsic errors — these are the empirically observed hotspots.
- **Do**: Pay special attention to middle turns of the dialogue — they are statistically most likely to be missed in summaries.
- **Do**: Distinguish between "missing because unimportant" and "missing because the summarizer failed." Missed Conversation is intentionally a harsh criterion; flag it but note when the omission is arguably acceptable.
- **Avoid**: Conflating Speaker Misattribution with Wrong Linking. Misattribution is about who said it; Wrong Linking is about incorrectly associating entities or facts within what was said.
- **Avoid**: Marking correct paraphrasing as Changed Meaning. The error applies when the semantic content shifts (e.g., "might" to "will"), not when synonyms are used ("purchase" to "buy").

## Error Handling

- **Dialogue has unnamed speakers** (e.g., "Speaker 1", "Speaker 2"): Proceed normally. Speaker Misattribution and Identity Bias checks still apply to the labels used.
- **Summary covers only part of the dialogue**: This is valid summarization. Only flag Missed Turn if the omitted turns contain information that materially changes the reader's understanding of the conversation.
- **Ambiguous speaker in dialogue**: If the source dialogue itself is ambiguous about who said something, do not flag the summary for Speaker Misattribution on that point. Note the ambiguity instead.
- **Summary is much longer than expected**: Extra length increases the surface area for extrinsic errors. Run within-turn-level checks with extra scrutiny on sentences that seem to "elaborate" beyond the source.
- **Non-English dialogues**: The taxonomy is language-agnostic. Apply the same error types, but be aware that translation artifacts may introduce Changed Meaning errors that originate in the translation step, not the summarization step.

## Limitations

- The framework is designed for **multi-turn dialogues**, not monologues, speeches, or single-turn Q&A. For document summarization (news articles, reports), standard factual consistency frameworks (e.g., FactCC, DAE) are more appropriate.
- **Missed Conversation** is an intentionally strict criterion. In practice, all summaries omit some detail. Use judgment about whether the omission is materially harmful.
- The taxonomy does not evaluate **fluency, coherence, or stylistic quality** of the summary — only factual and structural fidelity to the source dialogue.
- For very long dialogues (50+ turns), full turn-by-turn mapping becomes labor-intensive. In such cases, prioritize checking turns that introduce new topics, decisions, or action items.
- The framework assumes access to the **full source dialogue**. It cannot evaluate a summary in isolation.

## Reference

**Paper**: [DIAL-SUMMER: A Structured Evaluation Framework of Hierarchical Errors in Dialogue Summaries](https://arxiv.org/abs/2602.08149v1) — Ramnath et al., 2026. Look for Section 3 (error taxonomy definitions), Table 1 (full taxonomy overview), and Section 5 (LLM-Judge experiments and positional error distribution findings).