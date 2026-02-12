---
name: reasoning-while-asking-transforming
description: |
  Proactive Interactive Reasoning (PIR) — instead of guessing when requirements are ambiguous or premises are missing, Claude pauses reasoning to ask the user targeted clarifying questions, then resumes with the new information. Produces more accurate code, math solutions, and document edits while using fewer tokens.
  Trigger phrases: "help me solve this (but I haven't given all the details)", "write code for this feature", "edit this document to match my intent", "figure out what I mean", "build this but ask me questions first", "reason through this problem with me"
---

# Proactive Interactive Reasoning (PIR)

This skill implements the Proactive Interactive Reasoning paradigm from Chen et al. (2026). Instead of performing blind self-thinking — reasoning extensively over incomplete or ambiguous inputs — Claude learns to **interleave reasoning with targeted clarification questions**. When Claude detects that a problem has missing premises (unstated constraints, undefined variables, absent context) or ambiguous intent (multiple valid interpretations of what the user wants), it pauses its chain of thought, asks a specific question, incorporates the answer, and then continues reasoning. This produces dramatically better results: up to 32.7% higher accuracy on math, 22.9% higher pass rate on code generation, and 41 BLEU points improvement on document editing, while cutting reasoning token usage nearly in half.

## When to Use

- When the user asks you to solve a math problem but hasn't stated all constraints (e.g., "Find the maximum of x - y" without specifying the domain)
- When the user requests code for a feature but key requirements are implicit or missing (e.g., "Write me an API endpoint for user search" without specifying filters, pagination, auth, or response format)
- When the user asks to edit or rewrite a document but their intended tone, audience, or scope is unclear
- When a coding task has multiple valid architectural approaches and the user hasn't indicated a preference
- When you detect that proceeding with assumptions would likely produce something the user has to reject and redo
- When the user provides a partial specification and says something like "you figure out the rest" — this is precisely when PIR adds the most value
- When debugging a reported issue where the reproduction steps or environment details are incomplete

## Key Technique

**The core insight of PIR is uncertainty-triggered interaction.** Traditional chain-of-thought reasoning commits to a full solution path even when the model's internal confidence drops — it fills gaps with assumptions rather than questions. PIR flips this: at each reasoning step, the model monitors its own confidence (conceptually, its predictive entropy across possible next tokens). When confidence dips below a threshold — indicating a missing premise or ambiguous intent — the model stops reasoning and emits a clarifying question targeted at exactly the information gap it detected.

**Two types of uncertainty drive different questions.** *Premise-level uncertainty* means factual information needed for the solution is absent: a constraint, a data type, a boundary condition, an environment detail. The model asks for the missing fact. *Intent-level uncertainty* means the user's goal can be interpreted multiple ways: which output format, which edge-case behavior, which trade-off to prioritize. The model asks which interpretation the user prefers. Distinguishing these types lets the model ask the *right kind* of question rather than generic "can you clarify?" requests.

**The interaction follows a think-ask-respond loop.** The model reasons through what it can (`<think>` phase), identifies what it cannot resolve alone, asks a targeted question, receives the user's answer, then resumes reasoning with the new information integrated. This loop may repeat 1-3 times for complex tasks but averages fewer than 2 turns — the goal is *minimal, high-value* interaction, not an interrogation. The result is a solution that was correct on the first real attempt because it was built on verified rather than assumed premises.

## Step-by-Step Workflow

1. **Read the full problem statement and identify all explicit givens.** List every constraint, requirement, input, output, and context detail the user has actually stated. Do not infer anything yet.

2. **Perform an initial reasoning pass to surface what's missing.** Begin solving the problem mentally. At each reasoning step, ask yourself: "Am I making an assumption here, or do I know this from the user's input?" Track every assumption as a potential question.

3. **Classify each uncertainty as premise-level or intent-level.**
   - *Premise-level*: A fact you need but don't have (e.g., "What database are you using?", "What's the allowed range for n?", "Is the input guaranteed to be sorted?")
   - *Intent-level*: A choice the user should make (e.g., "Should this return an error or a default value on invalid input?", "Do you want this optimized for readability or performance?", "Should the edit preserve the original paragraph structure?")

4. **Prioritize questions by impact.** Rank uncertainties by how much the solution would change depending on the answer. Ask only high-impact questions — those where different answers lead to materially different code, solutions, or outputs. Discard questions where the answer wouldn't change your approach.

5. **Ask 1-3 targeted clarifying questions in a single turn.** Group related questions together. Each question must be specific and actionable — the user should be able to answer in one sentence. Never ask open-ended "tell me more" questions. Provide the 2-3 most likely answers as options when possible so the user can respond quickly.

6. **Integrate the user's responses into your reasoning state.** Update your mental model with the new information. Re-evaluate any earlier reasoning steps that were built on assumptions now replaced by facts.

7. **Resume reasoning from the point of uncertainty, not from scratch.** Continue your chain of thought using the clarified information. Do not restart the entire problem — the work you did before the question remains valid.

8. **Check for second-order uncertainties.** The user's answers may reveal new gaps. If so, ask one more focused question. But limit total interaction to 3 rounds maximum — beyond that, make reasonable assumptions and state them explicitly.

9. **Produce the final solution with an assumptions manifest.** Deliver the answer, code, or edit. If any assumptions remain (because you chose not to ask), list them explicitly at the end so the user can correct any that are wrong.

10. **Verify completeness against the original request.** Before delivering, re-read the user's original message and confirm every stated requirement is addressed in your output.

## Concrete Examples

**Example 1: Ambiguous code generation request**

```
User: Write me a function that processes user uploads.

Approach (PIR):
1. Read the request — "processes user uploads" is highly ambiguous.
2. Initial reasoning: Need to know file types, size limits, storage
   destination, validation rules, sync vs async, error handling strategy.
3. Classify: All are premise-level (facts about the system I don't have).
4. Prioritize: File types and storage destination change the implementation
   the most. Error handling is intent-level (return error vs retry?).

Questions asked:
> Before I write this, I need to clarify a few things:
> 1. What file types should be accepted? (e.g., images only, documents,
>    or any file type)
> 2. Where should uploads be stored? (local filesystem, S3, database blob)
> 3. Should the function validate files synchronously during the request
>    or queue them for async processing?

User responds: Images only (JPEG/PNG), S3, synchronous validation.

Resume reasoning with answers integrated:
- Validate MIME type and magic bytes for JPEG/PNG
- Resize/compress before upload
- Use boto3 for S3
- Return presigned URL after upload

Output: A complete, focused function with image validation, S3 upload,
and error handling — no wasted code for document parsing or database
storage that the user didn't need.
```

**Example 2: Math problem with missing premises**

```
User: Find the minimum value of a + b given that ab = 12.

Approach (PIR):
1. Read the problem — we have ab = 12, minimize a + b.
2. Initial reasoning: By AM-GM, a + b >= 2*sqrt(ab) = 4*sqrt(3),
   BUT this assumes a, b > 0. The problem doesn't state this.
   If a, b can be negative, then a + b is unbounded below
   (e.g., a = -1000, b = -12/1000).
3. Classify: Premise-level — the domain of a and b is unstated.

Question asked:
> Are a and b restricted to positive real numbers, or can they be any
> real numbers? (If they can be any reals, a + b has no minimum since
> one can be arbitrarily negative.)

User responds: Yes, both are positive reals.

Resume reasoning:
- By AM-GM: a + b >= 2*sqrt(12) = 4*sqrt(3)
- Equality when a = b = 2*sqrt(3)

Output: The minimum value is 4*sqrt(3), achieved at a = b = 2*sqrt(3).
```

**Example 3: Document editing with unclear intent**

```
User: Make this README better.

[pastes a 200-line README with installation, usage, API reference,
and contributing sections]

Approach (PIR):
1. Read the README — it exists and has structure, but "better" is
   purely intent-level ambiguity.
2. Initial reasoning: Could mean fix grammar, restructure, add
   missing sections, shorten it, make it more beginner-friendly,
   add badges/visuals, or all of the above.
3. Classify: Intent-level — I need to know what "better" means here.

Questions asked:
> "Better" can mean many things for a README. Which of these matter
> most to you?
> 1. **Clarity**: Simplify language, fix structure for scannability
> 2. **Completeness**: Add missing sections (changelog, FAQ,
>    troubleshooting)
> 3. **Brevity**: Cut length, remove redundancy, tighten prose
> 4. **Polish**: Add badges, table of contents, better code examples

User responds: Clarity and brevity — it's too long and hard to scan.

Resume reasoning:
- Focus on cutting redundant explanations
- Add headers for scannability
- Convert prose to bullet points where possible
- Preserve all technical accuracy

Output: A restructured README that's 40% shorter with clear section
headers, bulleted install steps, and a scannable API table — not a
rewrite that added a changelog the user didn't want.
```

## Best Practices

**Do:**
- Ask questions that differentiate between materially different solution paths. If the answer won't change your code, don't ask.
- Batch related questions into a single turn. Asking one question at a time across five turns is worse than asking three questions in one turn.
- Provide likely answer options with your questions (e.g., "Are you using PostgreSQL or MySQL?") — this reduces friction and shows you understand the problem space.
- State remaining assumptions explicitly when you deliver the solution, even after asking questions. Some gaps aren't worth asking about but are worth disclosing.

**Avoid:**
- Asking more than 3 questions per turn or more than 3 rounds of questions total. The paper shows diminishing returns beyond ~2 interaction rounds and user frustration increases.
- Asking obvious questions that you can resolve from context (e.g., asking "What language?" when the user pasted Python code).
- Using questions as a stalling tactic. If you have enough information to produce a good solution, produce it. PIR is for genuine uncertainty, not performative thoroughness.
- Asking generic questions like "Can you provide more details?" — every question must target a specific information gap and explain why you need it.
- Asking about things you should know as a programmer (standard library APIs, language semantics, common patterns). PIR targets *problem-specific* uncertainty, not general knowledge gaps.

## Error Handling

- **User refuses to answer questions**: Proceed with the most common/reasonable assumptions and list them explicitly. Say "I'll assume X — let me know if that's wrong and I'll adjust."
- **User's answers contradict each other**: Point out the contradiction specifically (e.g., "You said the input is always sorted, but also that duplicates should be handled — do you mean sorted with possible duplicates?"). Do not silently pick one interpretation.
- **User's answer reveals the problem is much larger than expected**: Acknowledge the expanded scope, break it into phases, and propose tackling the most critical part first rather than asking yet more questions.
- **You asked a question but the answer was already in the original prompt**: Apologize briefly and proceed. This indicates you missed context — re-read the original message more carefully before asking further questions.
- **User says "just do your best"**: This is implicit permission to use reasonable defaults. Proceed, but list your assumptions in the output so they can be corrected after the fact.

## Limitations

- PIR adds latency when questions require user response. For time-critical tasks where any reasonable solution is acceptable, skip the questioning phase and use explicit assumptions instead.
- Not useful when the problem is fully specified with no ambiguity (e.g., "Sort this array of integers in ascending order"). Asking unnecessary questions in clear-cut situations degrades the experience.
- The technique assumes the user *can* answer your questions. If the user is exploring and doesn't know what they want yet, switch to a prototyping approach — build something small, show it, iterate — rather than asking clarifying questions they can't answer.
- Less effective for pure refactoring tasks where the intent (improve structure without changing behavior) is inherently clear and the work is mechanical.
- Questions about deeply technical implementation details (e.g., "Should I use a B-tree or hash index?") often confuse non-technical users. Tailor question complexity to the user's apparent expertise level.

## Reference

**Paper**: [Reasoning While Asking: Transforming Reasoning LLMs from Passive Solvers to Proactive Inquirers](https://arxiv.org/abs/2601.22139v1) (Chen et al., 2026)
**Code**: [github.com/SUAT-AIRI/Proactive-Interactive-R1](https://github.com/SUAT-AIRI/Proactive-Interactive-R1)
**Key takeaway**: Section 3 describes the uncertainty-aware SFT procedure and composite reward function (correctness + interaction efficiency + helpfulness) that trains the think-ask-respond loop. Section 4 shows that PIR cuts reasoning tokens by ~50% while improving accuracy across math, code, and document editing — the gains come from *not reasoning about the wrong problem*.