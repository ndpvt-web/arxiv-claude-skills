---
name: "dancing-chains-strategic-persuasion"
description: >
  Apply Theory of Mind-based strategic persuasion to code reviews, PR rebuttals,
  RFC objections, and technical disagreements. Uses the ToM-Strategy-Response (TSR)
  pipeline from RebuttalAgent to model the reviewer's mental state, formulate a
  targeted persuasion strategy, and generate a grounded response.
  Trigger phrases:
  - "Help me respond to this code review"
  - "Write a rebuttal for this PR feedback"
  - "How should I push back on this review comment"
  - "Draft a response to this RFC objection"
  - "Help me persuade the reviewer"
  - "Respond to this technical critique"
---

# Strategic Persuasion via Theory of Mind for Technical Rebuttals

This skill enables Claude to craft persuasive, strategically grounded responses to code reviews, PR feedback, RFC objections, and technical critiques. It applies the ToM-Strategy-Response (TSR) pipeline from the RebuttalAgent paper (He et al., ICLR 2026), which treats rebuttal not as blunt counter-argument but as strategic communication under information asymmetry -- modeling what the reviewer thinks, deciding how to shift their position, and only then composing the response.

## When to Use

- When a user receives code review comments they disagree with and needs to craft a convincing response
- When a user needs to defend an architectural decision in a PR against pushback
- When an RFC or design document gets critical feedback that needs a structured reply
- When a user must respond to multiple review comments with varying levels of validity
- When a user wants to concede some points while firmly defending others in a review thread
- When a user is drafting a response to a conference/journal paper review
- When a user needs to navigate a technically charged disagreement with a colleague or maintainer

## Key Technique: ToM-Strategy-Response Pipeline

The core insight from "Dancing in Chains" is that effective rebuttal requires **perspective-taking before argument construction**. Most people jump directly to defending their code. The TSR pipeline inserts two critical reasoning stages before generating any text.

**Stage 1 -- Theory of Mind (T):** Build a hierarchical profile of the reviewer at two levels. At the *macro level*, infer their overall stance (supportive, skeptical, hostile), dominant concern (correctness, performance, maintainability, style), and expertise level (domain expert, generalist, junior). At the *micro level*, classify each specific comment by its concern category: significance of the issue raised, methodology/design critique, rigor of evidence (tests, benchmarks), or presentation/clarity complaints. This profile determines *who you are talking to* and *what they actually care about*.

**Stage 2 -- Strategy (S):** Translate the reviewer profile into an actionable persuasion plan. This is where strategic trade-offs happen: when to concede a point to build goodwill, when to stand firm with evidence, when to reframe the narrative, and when to propose a compromise. The strategy is a short explicit plan (2-4 sentences) that guides tone, structure, and argumentation before any response text is written.

**Stage 3 -- Response (R):** Generate the actual reply conditioned on both the reviewer profile and the strategy. This ensures the response is coherent, targeted, and persuasive rather than reactive. Each response addresses the reviewer's underlying concern (not just their surface objection), uses evidence aligned to their expertise level, and maintains the strategic tone decided in Stage 2.

## Step-by-Step Workflow

1. **Collect all inputs.** Gather the review comment(s), the relevant code/design under review, and any surrounding context (PR description, commit messages, related discussions). If the user provides only the review comment, ask for the code or design being reviewed.

2. **Macro-level reviewer profiling.** Analyze the full set of review comments to infer:
   - **Overall Stance**: Approve-leaning, neutral, reject-leaning, or actively hostile
   - **Overall Attitude**: Constructive, terse, dismissive, or thorough
   - **Dominant Concern**: What class of issue drives most comments (correctness, design, performance, style, testing)
   - **Reviewer Expertise**: Based on vocabulary, specificity of critique, and what they do/don't flag

3. **Micro-level comment classification.** For each individual comment, tag it with:
   - **Concern type**: Significance (does this matter?), Methodology (is the approach sound?), Rigor (where are the tests/benchmarks?), Presentation (clarity/naming/formatting)
   - **Validity assessment**: Is the reviewer correct, partially correct, or mistaken?
   - **Emotional charge**: Neutral observation, mild concern, or strong objection

4. **Formulate explicit strategy.** Based on the profile, write a 2-4 sentence strategy plan before drafting any response. Decide for each comment whether to:
   - **Concede** -- acknowledge the point and describe a fix (builds goodwill)
   - **Stand firm** -- provide concrete evidence (code, benchmarks, specs) that the current approach is correct
   - **Reframe** -- redirect from the surface objection to the actual trade-off being made
   - **Compromise** -- offer a middle-ground change that addresses the concern without abandoning the approach
   - **Defer** -- agree it's worth investigating but scope it as a follow-up

5. **Structure the response for the reviewer's expertise level.** For expert reviewers: be precise and technical, skip obvious context. For generalist reviewers: provide more rationale for domain-specific decisions. For junior reviewers: be encouraging and educational, not dismissive.

6. **Draft each comment response.** For each review point, write a response that:
   - Opens by acknowledging the reviewer's concern (never start with "No" or "Actually")
   - Executes the chosen strategy with specific evidence (code snippets, benchmark numbers, spec references)
   - Closes with a clear action item or explicit stance

7. **Calibrate tone across the full response.** Read the complete set of replies together. Ensure the cumulative tone matches the strategy: if you conceded 3 minor points, your firm stance on the major point should feel earned, not combative.

8. **Add structural signals.** For multi-comment responses, use clear headers or numbered references to match the reviewer's original structure, making it easy for them to track your reply against their comments.

9. **Self-check for strategy drift.** Reread each response against the explicit strategy from Step 4. Flag any response that contradicts the planned approach (e.g., strategy says "concede" but the draft argues back).

10. **Present the output with the reasoning visible.** Show the user the reviewer profile, the strategy rationale, and the drafted responses -- so they can adjust the strategy before sending.

## Concrete Examples

**Example 1: Defending an architectural choice in a PR review**

```
User: Help me respond to this review comment on my PR:

Reviewer says: "I don't understand why you're using an event-driven
architecture here. This adds unnecessary complexity. A simple
synchronous call chain would be much easier to maintain and test.
Please simplify."

My PR adds an event bus for decoupling the notification system from
the order processing pipeline in our e-commerce service.

---

Approach:

[ToM Analysis]
- Macro: Reviewer is skeptical but constructive ("please simplify"
  not "this is wrong"). Dominant concern is maintainability/complexity.
  Expertise appears generalist -- they haven't engaged with the specific
  coupling problem.
- Micro: Methodology critique, partially valid (event-driven IS more
  complex), moderate emotional charge.

[Strategy]
Concede the complexity concern to build goodwill, then reframe: the
complexity is the *cost* of solving a real coupling problem that causes
bugs today. Provide concrete evidence of the problem. Offer a compromise
on scope.

[Response]
"You're right that event-driven adds indirection compared to direct
calls -- that's a real trade-off. The reason I went this direction is
that we currently have 4 places where order processing directly imports
and calls notification logic (OrderService.java:142, :203, :287,
RefundHandler.java:56). This coupling caused the outage last sprint
when the email provider timeout cascaded into order failures (see
incident #1247).

The event bus decouples these so notification failures can't block
orders. I've kept the event surface small -- just 2 event types
(OrderCompleted, OrderRefunded) rather than a general-purpose bus.

That said, I'm open to scoping this down: I could start with only
the OrderCompleted event (which covers the incident case) and handle
RefundHandler in a follow-up. Would that address your complexity
concern while still fixing the coupling issue?"
```

**Example 2: Responding to multiple review comments with mixed validity**

```
User: I got these review comments on my Python PR. Help me respond:

1. "This function is 45 lines long, please break it up"
2. "You're not handling the case where the API returns a 429"
3. "Why are you using dataclasses instead of Pydantic here?"

The PR adds a data ingestion pipeline that fetches from a rate-limited
external API and transforms records into our internal format.

---

Approach:

[ToM Analysis]
- Macro: Reviewer is thorough and constructive. Three distinct concerns
  spanning style, correctness, and design. Likely experienced -- the 429
  catch is specific and practical.
- Micro:
  1. Presentation concern, partially valid
  2. Rigor concern, fully valid (genuine bug)
  3. Methodology concern, debatable (both are reasonable)

[Strategy]
Concede #2 immediately and enthusiastically -- it's a real bug and
acknowledging it builds credibility. For #1, offer a targeted
compromise. For #3, stand firm with a brief rationale but keep the
door open.

[Response]

> 1. "This function is 45 lines long, please break it up"

Good call. I've extracted the field-mapping logic into
`_map_external_to_internal()` and the validation into
`_validate_record()`. The main function is now 18 lines and reads
as a clear pipeline: fetch -> map -> validate -> persist.

> 2. "You're not handling the case where the API returns a 429"

You're right, and that's a real gap given this API's rate limits.
I've added exponential backoff with jitter (3 retries, base 1s)
in commit abc123. Also added a test that mocks 429 responses to
verify the retry behavior. Thanks for catching this.

> 3. "Why are you using dataclasses instead of Pydantic here?"

I went with dataclasses because these objects are internal transfer
types that never touch serialization boundaries -- they're created
in `ingest()` and consumed in `persist()` within the same process.
Pydantic's validation overhead (~4x slower construction in our
benchmarks) doesn't buy us anything here since the data is already
validated in `_validate_record()`. Happy to reconsider if we later
need to serialize these, but for now the simpler tool fits.
```

**Example 3: Responding to a hostile/dismissive review**

```
User: Got this review on my RFC for migrating from REST to gRPC:

"This proposal is half-baked. You haven't considered backward
compatibility at all. The migration cost alone makes this DOA."

---

Approach:

[ToM Analysis]
- Macro: Reviewer is reject-leaning and dismissive. Attitude is terse
  and emotionally charged ("half-baked", "DOA"). Dominant concern is
  migration cost/compatibility. Likely has operational experience --
  these are practical concerns, not theoretical.
- Micro: Significance + Methodology critique. The concerns are
  legitimate even if the tone is harsh.

[Strategy]
Do NOT match the hostile tone or become defensive. Acknowledge the
valid underlying concerns with specific evidence that they were
considered. The goal is to demonstrate thoroughness, which directly
counters "half-baked." Provide concrete migration plan details to
shift the conversation from vibes to specifics.

[Response]
"Backward compatibility and migration cost are central concerns --
let me clarify how the RFC addresses them, since I may not have
made this prominent enough in the document.

Section 4.2 describes the dual-protocol phase: existing REST
endpoints remain live behind the API gateway while gRPC endpoints
come online service-by-service. No client breaks during migration.
I've estimated the timeline in Appendix B based on our 23 endpoints
-- the proposal is to migrate the 8 highest-traffic endpoints first
(which handle 91% of internal RPC volume) and leave the long-tail
REST endpoints as-is until Q3.

On cost: the RFC includes a comparison table (Table 3) showing
current REST serialization overhead vs. projected gRPC performance
at our p99 latency targets. The break-even point is ~4 months
post-migration for the top-8 endpoints.

I'll update the RFC to move the compatibility and cost analysis
into the main body rather than the appendix so these are more
visible. Would you be open to reviewing a revised version that
foregrounds these sections?"
```

## Best Practices

- **Do:** Always write the strategy explicitly before drafting any response text. Skipping straight to writing is the most common failure mode.
- **Do:** Concede minor points readily and early. Concessions on small issues increase your credibility when you stand firm on important ones.
- **Do:** Provide specific evidence (line numbers, benchmark data, incident links, spec references) for every point where you stand firm. Unsupported assertions trigger reviewer resistance.
- **Do:** Match response depth to comment depth. A one-line nitpick deserves a one-line fix acknowledgment, not a three-paragraph defense.
- **Avoid:** Starting any response with disagreement words ("No", "Actually", "But"). Open by acknowledging the reviewer's perspective, then pivot.
- **Avoid:** Using the ToM analysis to manipulate rather than communicate. The goal is mutual understanding, not winning through psychological tactics. If the reviewer is right, say so.
- **Avoid:** Applying this framework to trivial comments (typo fixes, formatting). Reserve strategic thinking for substantive disagreements.

## Error Handling

- **Insufficient context:** If the user provides only the review comment without the code/design being reviewed, ask for it. The TSR pipeline cannot generate a good strategy without knowing what is being defended.
- **Ambiguous reviewer intent:** If a comment could be read as either a blocking objection or a suggestion, flag this ambiguity to the user and draft two strategy variants (one for each reading). Let the user pick based on their knowledge of the reviewer.
- **Emotionally charged user:** If the user is visibly frustrated, generate the ToM analysis first and show it to them before drafting. Seeing the reviewer's perspective modeled objectively often de-escalates the user's emotional state.
- **Strategy-response mismatch:** If during Step 9 a response contradicts its strategy, regenerate that specific response rather than patching it. Patched responses read as incoherent.

## Limitations

- This technique works best for substantive technical disagreements. For simple factual corrections ("this variable is misspelled"), just fix it -- no strategy needed.
- The ToM profile is an inference, not ground truth. The reviewer's actual mental state may differ from what their written comments suggest. Present the profile as a working hypothesis to the user, not a certainty.
- This framework assumes good-faith review. In cases of genuinely bad-faith gatekeeping or personal antagonism, strategic persuasion has limits; the user may need to escalate through process channels rather than craft a better argument.
- Written reviews lose tone of voice, facial expressions, and other cues. The macro-level stance inference is less reliable for very short or terse reviews (1-2 sentences). When confidence is low, default to a neutral-constructive strategy.
- This skill helps draft responses but the user should always review and personalize before sending. The user knows the reviewer, the team dynamics, and the organizational context in ways Claude cannot infer from text alone.

## Reference

He, Z., Lyu, Z., & Fung, Y.R. (2026). *Dancing in Chains: Strategic Persuasion in Academic Rebuttal via Theory of Mind.* ICLR 2026. [arXiv:2601.15715v2](https://arxiv.org/abs/2601.15715v2) -- See Section 3 for the full TSR pipeline specification, Section 4 for the critique-and-refine data synthesis method, and Appendix D for complete worked examples of the ToM profiling and strategy formulation stages.