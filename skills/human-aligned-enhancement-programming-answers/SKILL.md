---
name: "human-aligned-enhancement-programming-answers"
description: >
  Enhance programming answers by classifying user feedback comments as actionable
  or non-actionable, then surgically incorporating valid concerns while preserving
  original intent. Based on the AUTOCOMBAT technique from Bappon et al. (2026).
  Trigger phrases: "improve this answer based on the comments", "incorporate feedback
  into this code answer", "fix this Stack Overflow answer using the comments",
  "enhance this programming answer with user feedback", "refine this technical answer
  based on reviewer comments", "update this code based on the critique"
---

# Human-Aligned Enhancement of Programming Answers

This skill enables Claude to take a programming answer (from Stack Overflow, code reviews, documentation, or internal knowledge bases) along with associated user feedback comments, classify which comments contain actionable improvement concerns versus general chatter, and produce a surgically refined answer that addresses valid feedback while preserving the original author's intent and structure. The technique is derived from the AUTOCOMBAT system, which demonstrated near-human quality improvements across 790 Stack Overflow answers and was endorsed by 84.5% of 58 practicing developers.

## When to Use

- When the user provides a programming answer alongside comments/feedback and asks to improve it
- When refining Stack Overflow answers, GitHub discussion replies, or code review responses based on accumulated feedback
- When the user has a technical answer with critique threads and wants a consolidated, corrected version
- When updating documentation or FAQ answers that have received corrections or suggestions in comments
- When a code snippet has been flagged with issues in review comments and needs targeted fixes
- When the user wants to triage a batch of comments to identify which ones contain real actionable concerns versus noise

## Key Technique

AUTOCOMBAT separates the enhancement task into two distinct phases: **concern extraction** and **targeted refinement**. In the first phase, each comment in the thread is classified into one of three categories: **Improvement-related Addressed (IA)** — comments that point to genuine errors, inefficiencies, or missing explanations; **Improvement-related Not Addressed (INA)** — valid improvement suggestions that the original author never acted on; and **General Comments (GC)** — conversational remarks, thanks, jokes, or meta-discussion with no actionable content. This classification prevents the model from hallucinating improvements or over-editing based on noise.

In the second phase, the system constructs a refinement prompt containing the original answer, the full comment thread (preserving order, mixing actionable and general), and optionally the original question for context. The model is constrained by strict policies: edit only if comments contain actionable concerns, do not inject your own corrections or ideas, use question context only when needed to resolve ambiguity, and keep edits minimal and targeted. The output is structured as a JSON object containing extracted concerns, a boolean indicating whether question context was used, a change log mapping each concern to specific edits, and the improved answer. This structured output ensures traceability — every change can be traced back to a specific piece of user feedback.

The critical insight is **selective question context grounding**: the original question is available but used only when comment feedback is ambiguous or requires understanding the asker's constraints. Empirically, this context was needed in roughly 16-32% of cases. Over-relying on question context causes over-generalization; ignoring it entirely misses constraint-dependent fixes (e.g., a comment saying "this doesn't work" might only make sense when you see the original question specifies Python 2 compatibility).

## Step-by-Step Workflow

1. **Collect inputs.** Gather three components: the original programming answer (code + explanation), the comment thread in chronological order, and optionally the original question or problem statement that prompted the answer.

2. **Classify each comment.** Read every comment and label it as one of:
   - **Actionable (IA/INA):** Points out a bug, logic error, security issue, missing edge case, outdated API, incomplete explanation, or performance problem.
   - **General (GC):** Thanks, agreement, tangential discussion, jokes, meta-comments about voting, or requests already addressed in the answer.

3. **Extract actionable concerns as a numbered list.** For each actionable comment, distill the specific concern into a concrete, testable statement (e.g., "The `sort()` call mutates the original list instead of returning a new one" rather than "comment #3 has a concern").

4. **Assess whether question context is needed.** If any concern is ambiguous without knowing the original constraints (language version, framework, specific use case), flag that the question context should be consulted. Otherwise, rely solely on the answer and comments.

5. **Apply minimal, targeted edits to the answer.** For each extracted concern:
   - Fix code errors (logic bugs, incorrect API usage, syntax issues).
   - Add missing explanations or caveats where comments flagged confusion.
   - Update outdated references (deprecated methods, version-specific behavior).
   - Preserve the original structure, variable naming conventions, and explanation style.

6. **Do NOT inject unsolicited improvements.** If no comment flags a particular issue, leave that part of the answer untouched — even if you notice something that could be improved. The constraint is feedback-driven enhancement only.

7. **Produce a structured output.** Return:
   - A list of extracted actionable concerns.
   - Whether question context was used (and why, if so).
   - A change log mapping each concern to the specific edit made.
   - The improved answer with changes integrated naturally.

8. **Verify intent preservation.** Re-read the improved answer against the original question/context. Confirm the answer still addresses the same problem, uses the same approach (unless the feedback explicitly called for a different approach), and does not introduce scope creep.

9. **Flag unresolvable feedback.** If a comment raises a valid concern but resolving it would require information not present in the answer, comments, or question (e.g., "this doesn't work on my machine"), note it as unresolvable rather than guessing.

10. **Present the result clearly.** Show the improved answer, the change log, and any unresolvable items so the user can make a final judgment.

## Concrete Examples

**Example 1: Fixing a Python logic error flagged by comments**

```
User: Improve this answer based on the comments.

Original Question: How do I check if all elements in a list are odd?

Original Answer:
def all_odd(lst):
    for num in lst:
        if num % 2 == 0:
            return True
    return False
# Returns True if all elements are odd.

Comments:
1. "This is backwards — it returns True when it finds an EVEN number." — user1234
2. "Thanks, this helped me get started!" — user5678
3. "Also doesn't handle empty lists explicitly, might want to note that." — user9012

Approach:
1. Classify comments: #1 is actionable (logic error), #2 is general, #3 is actionable (missing edge case note).
2. Extract concerns: (a) Logic is inverted — returns True on even, should return False. (b) No mention of empty list behavior.
3. Question context not needed — the comments and answer are self-contained.
4. Apply targeted fixes.

Output:

Concerns extracted:
1. Logic inversion: function returns True when an even number is found, but should return False (flagged by user1234)
2. Empty list behavior undocumented (flagged by user9012)

Change log:
- Concern 1 → Flipped return values: `return True` → `return False`, `return False` → `return True`
- Concern 2 → Added note about empty list behavior after the code block

Question context used: No

Improved answer:
def all_odd(lst):
    for num in lst:
        if num % 2 == 0:
            return False
    return True
# Returns True if all elements are odd.
# Note: Returns True for an empty list (vacuous truth), which is consistent
# with Python's built-in `all()` behavior.
```

**Example 2: Updating a JavaScript answer with outdated API usage**

```
User: Incorporate the feedback into this code answer.

Original Question: How to fetch JSON data from an API in JavaScript?

Original Answer:
var xhr = new XMLHttpRequest();
xhr.open('GET', 'https://api.example.com/data', true);
xhr.onreadystatechange = function() {
    if (xhr.readyState === 4 && xhr.status === 200) {
        var data = JSON.parse(xhr.responseText);
        console.log(data);
    }
};
xhr.send();

Comments:
1. "Why not use fetch()? XMLHttpRequest is pretty outdated for modern browsers." — devUser42
2. "You should also handle errors, what if the request fails?" — codeReview99
3. "Upvoted, works great" — newbie101
4. "The var keyword should be let or const in modern JS" — lintBot

Approach:
1. Classify: #1 actionable (outdated API), #2 actionable (missing error handling),
   #3 general, #4 actionable (outdated syntax).
2. Extract three concerns. Question context not needed.
3. Modernize the code per feedback while keeping the same goal (fetch JSON, log it).

Output:

Concerns extracted:
1. XMLHttpRequest is outdated; fetch() is the modern standard (devUser42)
2. No error handling for failed requests (codeReview99)
3. Uses `var` instead of `const`/`let` (lintBot)

Change log:
- Concern 1 → Replaced XMLHttpRequest with fetch() API
- Concern 2 → Added .catch() handler and response.ok check
- Concern 3 → Replaced var with const (resolved implicitly by rewrite)

Question context used: No

Improved answer:
fetch('https://api.example.com/data')
    .then(response => {
        if (!response.ok) {
            throw new Error(`HTTP error: ${response.status}`);
        }
        return response.json();
    })
    .then(data => {
        console.log(data);
    })
    .catch(error => {
        console.error('Failed to fetch data:', error);
    });
```

**Example 3: Using question context to resolve ambiguous feedback**

```
User: Fix this Stack Overflow answer using the comments.

Original Question: How can I disable the close button (X) on a WinForms
application so users can only exit via the File menu?

Original Answer:
protected override void WndProc(ref Message m) {
    const int WM_CLOSE = 0x0010;
    if (m.Msg == WM_CLOSE) return; // block close
    base.WndProc(ref m);
}

Comments:
1. "This also blocks Alt+F4 and programmatic Close() calls, not just the X button." — winFormsGuru
2. "Nice approach, clean solution" — beginner88

Approach:
1. Classify: #1 actionable (over-broad blocking), #2 general.
2. The concern is that WM_CLOSE is too broad. But what exactly should be blocked?
3. Question context IS needed: the question says "only exit via the File menu,"
   meaning programmatic Close() from the File menu handler must still work.
4. Refine to intercept only user-initiated close (SC_CLOSE from system menu).

Output:

Concerns extracted:
1. Intercepting WM_CLOSE blocks all close sources including programmatic Close()
   and Alt+F4, not just the X button (winFormsGuru)

Question context used: Yes — needed to determine that File menu exit (programmatic
Close()) must remain functional; only the title bar X button should be disabled.

Change log:
- Concern 1 → Replaced WM_CLOSE interception with WM_SYSCOMMAND/SC_CLOSE check,
  which targets only the window chrome close button while allowing programmatic close.

Improved answer:
protected override void WndProc(ref Message m) {
    const int WM_SYSCOMMAND = 0x0112;
    const int SC_CLOSE = 0xF060;
    if (m.Msg == WM_SYSCOMMAND && (m.WParam.ToInt32() & 0xFFF0) == SC_CLOSE)
        return; // blocks only the X button and Alt+F4
    base.WndProc(ref m);
}
// Note: This still blocks Alt+F4. To allow Alt+F4, you would need additional
// logic to distinguish the source. Programmatic this.Close() calls (e.g., from
// a File > Exit menu handler) will work normally.
```

## Best Practices

- **Do:** Preserve the original answer's structure, tone, and variable naming conventions. The goal is surgical improvement, not a rewrite.
- **Do:** Process comments in chronological order — later comments may reference or supersede earlier ones.
- **Do:** Produce a traceable change log so every edit maps back to a specific comment. Never make silent changes.
- **Do:** Mark concerns as "unresolvable" when the feedback requires external information (environment-specific bugs, version conflicts not specified).
- **Avoid:** Injecting your own improvements beyond what the comments request. If no one flagged it, don't fix it — this is the core discipline of the technique.
- **Avoid:** Treating all comments equally. General comments (thanks, +1, tangential discussion) must be filtered out before refinement to prevent hallucinated edits.
- **Avoid:** Over-relying on question context. Use it only when a comment's concern is ambiguous without it. Empirically, question context is needed in roughly 20-30% of cases.
- **Avoid:** Merging contradictory feedback. If two comments suggest opposite approaches, flag the contradiction rather than picking a side.

## Error Handling

- **No actionable comments found:** If all comments are general (thanks, +1, jokes), report that no improvement-related feedback was identified and return the original answer unchanged. Do not manufacture edits.
- **Contradictory feedback:** If comments suggest mutually exclusive changes (e.g., "use async/await" vs. "keep it callback-based for Node 6 compatibility"), flag the contradiction, present both options, and let the user decide.
- **Feedback references missing context:** If a comment says "this doesn't work" without specifying what fails, mark it as unresolvable and note what additional information would be needed.
- **Ambiguous code language:** If the programming language is unclear from the answer alone, check the question context or tags before making syntax-level changes.
- **Stale feedback:** If comments reference APIs or behavior from an older version, verify current behavior before applying the suggestion — the comment itself may now be outdated.

## Limitations

- This technique is **feedback-driven only**. It will not catch bugs, security issues, or inefficiencies that no commenter has flagged. It is not a substitute for comprehensive code review.
- The approach assumes at least some comments contain actionable feedback. For answers with zero comments or only general praise, there is nothing to enhance.
- Quality depends heavily on comment quality. Incorrect feedback (a commenter who is wrong) will be incorporated unless the error is obvious. When in doubt, flag uncertain feedback for user review rather than blindly applying it.
- Works best for self-contained code answers (single functions, short scripts). For answers involving complex multi-file architectures or infrastructure, comment-driven refinement may be insufficient.
- The technique preserves the original approach. If the entire approach is flawed (not just a detail), comment-based patching may produce a polished but fundamentally wrong answer. In such cases, a full rewrite is more appropriate.

## Reference

[Human-Aligned Enhancement of Programming Answers with LLMs Guided by User Feedback](https://arxiv.org/abs/2601.17604v1) — Bappon, Mondal, Roy, Schneider (2026). Focus on Table 2 (refinement prompt policies), Section 4 (AUTOCOMBAT architecture), and Section 5.2 (intent preservation analysis) for the core actionable details behind this technique.