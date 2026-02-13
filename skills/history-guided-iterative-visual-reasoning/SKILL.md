---
name: "history-guided-iterative-visual-reasoning"
description: |
  Apply the H-GIVR (History-Guided Iterative Visual Reasoning) framework for self-correcting
  multimodal reasoning. Uses iterative re-observation of images with accumulated answer history
  to dynamically correct errors and converge on accurate answers. Trigger phrases:
  - "analyze this image iteratively"
  - "use self-correction to answer this visual question"
  - "reason about this image step by step with verification"
  - "iteratively verify your answer about this image"
  - "use history-guided reasoning on this screenshot"
  - "double-check your visual analysis"
---

# History-Guided Iterative Visual Reasoning with Self-Correction (H-GIVR)

This skill enables Claude to apply the H-GIVR framework when answering questions about images, screenshots, diagrams, or any visual content. Instead of producing a single answer and moving on, Claude iteratively re-examines the image, accumulates a history of candidate answers, uses that history to guide subsequent reasoning, and stops only when two iterations converge on the same answer. This catches visual misinterpretations, OCR-like errors, and reasoning mistakes that a single pass would miss.

## When to Use

- When the user asks Claude to answer a question about an image and accuracy is critical (e.g., reading data from a chart, interpreting a scientific diagram, extracting values from a screenshot)
- When Claude's first answer about visual content seems uncertain or the image is complex/ambiguous
- When the user explicitly asks Claude to "double-check" or "verify" its visual reasoning
- When answering multiple-choice questions about images (science, math, visual puzzles)
- When extracting structured data from screenshots, photographs of documents, or handwritten content
- When debugging UI issues from screenshots where precise detail matters
- When the user asks to iteratively refine analysis of a visual artifact (architecture diagram, flowchart, error screenshot)

## Key Technique

**Standard self-consistency** methods generate multiple independent answers via repeated sampling and pick the majority vote. Each sample is isolated -- the model never learns from its own prior attempts. H-GIVR breaks this paradigm by feeding the history of all previous answers back into each subsequent iteration. This transforms independent sampling into a guided search where the model can actively correct mistakes it recognizes in its prior outputs.

The framework has two core mechanisms. **Consistency-Iterative Reasoning** appends the growing answer history to each new prompt, instructing the model to critically evaluate all prior answers before producing a new one. This lets the model notice patterns in its own errors (e.g., "I keep alternating between B and C -- let me re-examine why"). **Visual Re-observation** periodically re-describes the image from scratch at even-numbered iterations, combating the tendency to anchor on an initial (possibly flawed) visual interpretation. The fresh description can capture details missed on the first pass.

Convergence is simple and efficient: the moment any two iterations produce the same answer, that answer is returned as final. This typically happens within 2-4 iterations, far below the safety cap of 10. The result is significantly higher accuracy than single-pass or independent-sampling approaches, at modest additional cost.

## Step-by-Step Workflow

1. **Receive the visual question.** Parse the user's question and identify the image(s) to reason about. Note whether it is multiple-choice (closed-ended) or free-form (open-ended).

2. **Generate an initial visual description.** Examine the image carefully and produce a detailed textual description of its contents -- layout, text, colors, data values, relationships, labels, anything potentially relevant to the question. Store this as the current `FeatureSet`.

3. **Produce the first candidate answer.** Using the question, image, and `FeatureSet`, reason through the problem and produce answer `a₁`. Initialize `AnswerList = [a₁]`.

4. **Check for convergence.** If `a₁` matches no prior answer (trivially true on iteration 1), continue. If the new answer matches any previous answer in `AnswerList`, stop and return that answer as the final output.

5. **Construct the history-augmented prompt.** For the next iteration, build the prompt as: the original question + the current `FeatureSet` + the full `AnswerList`. Explicitly instruct yourself: "Consider all previous answers carefully. If you believe they are all wrong, provide a corrected answer with reasoning. If one is correct, confirm it."

6. **Re-observe the image on even iterations.** On iterations 2, 4, 6, etc., look at the image again with fresh eyes and produce an updated visual description. Replace or augment the `FeatureSet`. This combats anchoring bias from the initial observation.

7. **Generate the next candidate answer.** Using the augmented prompt (with history and possibly refreshed features), reason through the problem again. Append the new answer to `AnswerList`.

8. **Repeat convergence check.** If the new answer matches any previous answer in `AnswerList`, that answer is the final output. Otherwise, return to step 5.

9. **Enforce a maximum iteration cap.** If after 5 iterations (for practical use -- the paper uses 10) no convergence is reached, select the most recent answer or the answer appearing most frequently in `AnswerList`.

10. **Present the final answer with confidence annotation.** Report the converged answer, how many iterations it took to converge, and flag if the cap was hit without convergence (indicating lower confidence).

## Concrete Examples

**Example 1: Reading a bar chart from a screenshot**

```
User: [uploads bar chart screenshot] What was the revenue in Q3?

Approach:
1. VISUAL DESCRIPTION (iteration 1): "Bar chart with 4 bars labeled Q1-Q4.
   Y-axis shows revenue in millions from 0-50. Q1 ~$12M, Q2 ~$28M,
   Q3 bar reaches approximately the $35M line, Q4 ~$42M."
2. ANSWER a₁: "$35 million"
3. HISTORY-AUGMENTED (iteration 2, even → re-observe image):
   Fresh observation: "Looking again, the Q3 bar top edge sits between
   the $35M and $40M gridlines, closer to $37M. The gridlines are
   spaced at $5M intervals."
   Previous answers: [$35M]
   Answer a₂: "$37 million"
4. ITERATION 3: Previous answers: [$35M, $37M]. Re-examining with both
   candidates, the bar clearly passes the $35M line by about 40% of
   the interval to $40M. That's ~$37M.
   Answer a₃: "$37 million"  ← MATCHES a₂, CONVERGE.

Output: "The Q3 revenue was approximately $37 million.
(Converged after 3 iterations; initial estimate of $35M was corrected
on re-observation.)"
```

**Example 2: Multiple-choice science question with diagram**

```
User: [uploads diagram of a circuit with a battery, two resistors in
parallel, and an ammeter] Which answer is correct?
A) Current through R1 equals current through R2
B) Total current equals current through R1
C) Total current equals sum of currents through R1 and R2
D) Voltage across R1 is different from voltage across R2

Approach:
1. VISUAL DESCRIPTION: "Circuit diagram showing a battery connected to
   two resistors R1 and R2 in parallel. An ammeter is in the main branch
   before the parallel split."
2. ANSWER a₁: "C" (parallel circuit → currents add)
3. ITERATION 2 (re-observe): Confirmed parallel topology. Previous
   answers: [C]. For parallel resistors: same voltage across both,
   total current = I₁ + I₂. Answer a₂: "C" ← MATCHES a₁, CONVERGE.

Output: "The answer is C. Total current equals the sum of currents
through R1 and R2. (Converged after 2 iterations.)"
```

**Example 3: Debugging a UI screenshot with subtle error**

```
User: [uploads screenshot of a web form] The submit button isn't working.
What's wrong based on this screenshot?

Approach:
1. VISUAL DESCRIPTION: "Web form with fields: Name (filled), Email
   (filled with 'user@test'), Phone (empty), Submit button (gray).
   There's a small red asterisk next to Phone. Console visible at
   bottom shows no errors."
2. ANSWER a₁: "The Phone field is required (red asterisk) but empty,
   so the submit button is likely disabled."
3. ITERATION 2 (re-observe): "Looking more carefully at the email field,
   the value 'user@test' lacks a TLD. Also confirming: Phone field
   has a red asterisk and is empty. The submit button appears to have
   a 'disabled' style (gray, no hover state)."
   Previous answers: [Phone field required but empty]
   Answer a₂: "Two issues: (1) Phone field is required but empty,
   (2) Email 'user@test' may fail validation -- missing TLD."
4. ITERATION 3: Previous answers: [Phone required, Phone + Email invalid].
   The email validation issue is a genuine finding from re-observation.
   Answer a₃: "Phone field required but empty, and email may be
   invalid." ← Consistent with a₂ in substance, CONVERGE.

Output: "Two issues found:
1. The Phone field is marked required (red asterisk) but is empty
2. The email 'user@test' appears to be missing a top-level domain
Either or both could be preventing submission.
(Converged after 3 iterations; the email issue was caught on
re-observation in iteration 2.)"
```

## Best Practices

- **Do:** Write out the visual description explicitly on each observation pass. The act of describing forces attention to details that quick scanning misses.
- **Do:** Present the full answer history to yourself before each new attempt. Explicitly compare candidates and reason about why they differ.
- **Do:** On re-observation passes, deliberately look for things you might have missed -- small text, axis labels, subtle color differences, partially occluded elements.
- **Do:** Stop as soon as two answers match. The convergence signal is reliable; extra iterations add cost without meaningful accuracy gains.
- **Avoid:** Anchoring on the first answer. The whole point of re-observation is to challenge initial interpretations. If you always confirm your first answer, the framework adds no value.
- **Avoid:** Changing answers without justification. Each new candidate should include explicit reasoning about what changed and why, referencing the history.

## Error Handling

- **No convergence within cap:** If 5 iterations pass without any two answers matching, report the most frequent answer with a clear warning that confidence is low. Suggest the user provide additional context or a higher-resolution image.
- **Contradictory visual descriptions:** If re-observation produces descriptions that conflict with earlier ones (e.g., "3 bars" vs "4 bars"), flag this explicitly. The image may be ambiguous, low-resolution, or partially occluded. Ask the user to confirm or provide a better image.
- **Oscillating answers:** If the answer list alternates (e.g., [A, B, A, B, ...]), the model is stuck between two interpretations. After detecting a cycle of length 2+, stop and present both candidates with the reasoning for each, letting the user decide.
- **Invalid or unparseable visual content:** If the image cannot be meaningfully described (blank, corrupted, or irrelevant), say so on the first iteration rather than iterating on garbage.

## Limitations

- **Computational cost scales with difficulty.** Easy questions converge in 2 iterations; genuinely ambiguous images may hit the cap. The framework is most valuable for medium-difficulty visual reasoning, not trivial or impossible cases.
- **Not a substitute for better images.** If the source image is too low-resolution or blurry to read, iterating won't manufacture information that isn't there. Garbage in, garbage out.
- **Single-model bias persists.** While history-guided iteration reduces random errors, systematic biases (e.g., consistently misreading a specific font, misidentifying a specific object class) may not self-correct because the same model makes the same mistake each time.
- **Open-ended questions converge slower.** Free-form answers are less likely to be identical strings across iterations. For open-ended visual questions, consider extracting a key fact or number as the convergence target rather than comparing full sentences.
- **Overkill for simple queries.** If the user asks "what color is this button?" and the answer is obviously blue, a single pass suffices. Reserve iterative reasoning for genuinely complex or high-stakes visual analysis.

## Reference

**Paper:** [History-Guided Iterative Visual Reasoning with Self-Correction](https://arxiv.org/abs/2602.04413v1) (Yang et al., 2026)
**Key insight:** Feeding accumulated answer history back into each iteration transforms independent sampling into guided self-correction, achieving 107% accuracy improvement over baselines with an average of only 2.57 iterations per question.