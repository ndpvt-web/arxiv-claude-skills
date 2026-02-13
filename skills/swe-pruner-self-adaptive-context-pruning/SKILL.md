---
name: "swe-pruner-self-adaptive-context-pruning"
description: |
  Apply SWE-Pruner's goal-conditioned context pruning to reduce token usage when working with large codebases.
  Teaches Claude to selectively skim code like a human programmer: formulate an explicit focus goal,
  score lines by relevance, and drop low-value content while preserving syntactic structure.
  Trigger phrases:
  - "prune the context" / "reduce context size"
  - "too much code, focus on what matters"
  - "compress this code for the task"
  - "skim this file for [specific concern]"
  - "only show me the relevant parts"
  - "reduce token usage on this codebase"
---

# SWE-Pruner: Self-Adaptive Context Pruning for Coding Agents

This skill enables Claude to apply goal-conditioned context pruning when working with large code files or long agent interaction histories. Instead of feeding entire files into context, Claude formulates an explicit task goal (e.g., "focus on error handling in the HTTP client"), then selectively retains only the lines relevant to that goal. This mirrors how experienced programmers skim source code — they don't read every line; they scan for the parts that matter given their current objective. The technique achieves 23-54% token reduction on real software engineering tasks while maintaining or even improving task success rates.

## When to Use

- When reading a large file (500+ lines) but only needing to understand a specific aspect of it (e.g., authentication logic, error paths, a particular API)
- When agent context is growing long from accumulated tool outputs and you need to summarize or prune previous results before continuing
- When the user asks you to focus on a specific concern within a broad codebase (e.g., "find the bug in the payment flow" across many files)
- When multiple files have been read and you need to retain only the task-relevant portions for reasoning
- When debugging and you want to isolate the relevant code paths from surrounding boilerplate
- When reviewing a PR diff that is large and the user only cares about specific functional areas

## Key Technique

**Goal-conditioned pruning** is the core insight. Traditional context compression (like LLMLingua) uses fixed metrics such as perplexity to decide what to remove — this is task-agnostic and frequently destroys syntactic structure or discards critical implementation details. SWE-Pruner instead requires an explicit **focus goal** that describes what the agent currently needs from the code. This goal acts as a query that makes pruning adaptive to the task at hand.

The pruning operates at **line granularity**. Each line of code is scored for relevance against the focus goal. Lines scoring above a threshold are retained; lines below it are dropped. To avoid fragmentation, single-line gaps between retained blocks are kept (preventing isolated orphan lines that break readability). The result preserves logical structure — function signatures, control flow, and the specific implementation details relevant to the goal — while removing unrelated code like imports for unused modules, unrelated helper functions, or boilerplate.

In the original paper, a 0.6B-parameter neural skimmer performs this scoring. When Claude applies this technique natively (without the external model), it uses its own judgment to perform the same goal-conditioned line selection — formulating the goal explicitly, then deciding per-block what is relevant. The key discipline is: **always state the goal before pruning, and prune against that goal, not general "importance."**

## Step-by-Step Workflow

1. **Identify the pruning trigger.** Recognize when context is large relative to the task scope. If you've read a 1000-line file but only need to understand its caching logic, that's a pruning opportunity. If tool outputs from previous turns are accumulating, consider pruning older results.

2. **Formulate an explicit focus goal.** Write a concise, specific statement of what you need from this code. Good goals are narrow and actionable:
   - "Identify how database connections are pooled and recycled"
   - "Find the validation logic for user input on the /signup endpoint"
   - "Trace the error propagation path from the HTTP handler to the logging layer"

   Bad goals are vague: "understand the code" or "find bugs."

3. **Segment the code into logical blocks.** Rather than scoring individual lines in isolation, group code into blocks: function/method bodies, class definitions, import sections, configuration blocks. This prevents splitting a function in half.

4. **Score each block against the focus goal.** For each block, ask: "Does this block contain information needed to achieve my focus goal?" Assign a mental relevance score:
   - **High relevance**: Directly implements or affects the goal (e.g., the actual caching function when the goal is about caching). Always retain.
   - **Medium relevance**: Provides necessary context (e.g., the class constructor that initializes the cache). Retain.
   - **Low relevance**: Unrelated functionality in the same file (e.g., a logging utility in a file about caching). Prune, replacing with a brief `# ... [N lines: unrelated helper functions] ...` marker.

5. **Preserve structural anchors.** Always retain:
   - File-level docstrings or module descriptions (1-2 lines)
   - Class/function signatures for pruned blocks (so the reader knows what was removed)
   - Import statements that are referenced by retained code
   - Lines immediately adjacent to retained blocks (avoid orphaned snippets)

6. **Assemble the pruned context.** Replace pruned sections with concise markers indicating what was removed and how many lines. This gives the reader (or future agent turns) enough information to request the full content if needed.

7. **Validate coherence.** After pruning, scan the result: Can you still follow the control flow relevant to the goal? Are there dangling references to pruned code? If so, restore the minimum needed to resolve them.

8. **Iterate if context is still too large.** If the pruned result is still large, tighten the focus goal (e.g., narrow from "error handling" to "error handling in the retry logic") and re-prune.

9. **Apply to multi-turn context.** For accumulated agent history, apply the same logic to previous tool outputs: retain results relevant to the current sub-task, replace others with one-line summaries of what was found.

## Concrete Examples

**Example 1: Pruning a large file to debug a specific function**

```
User: "There's a bug in the payment processing. The charge succeeds but
       the receipt email never sends. Look at services/payment.py (800 lines)."

Focus Goal: "Trace the code path from successful charge to receipt email dispatch"

Before pruning (800 lines):
  - Lines 1-45: imports and config
  - Lines 46-120: PaymentService class init, DB setup
  - Lines 121-250: validate_card(), check_fraud() — card validation logic
  - Lines 251-340: process_charge() — the core charge function
  - Lines 341-410: send_receipt_email() — email dispatch
  - Lines 411-500: refund(), partial_refund() — refund logic
  - Lines 501-650: reporting helpers, CSV export
  - Lines 651-800: admin endpoints, rate limiting

After pruning (~180 lines retained):
  - Lines 1-12: imports relevant to charge + email (smtp, stripe, models)
  - Lines 46-55: class init (DB and email client setup)
  - # ... [75 lines: card validation — not related to post-charge flow] ...
  - Lines 251-340: process_charge() — FULL (this is where charge completes)
  - Lines 341-410: send_receipt_email() — FULL (this is the email dispatch)
  - # ... [90 lines: refund logic] ...
  - # ... [150 lines: reporting helpers] ...
  - # ... [150 lines: admin endpoints] ...

Result: 78% reduction. The agent can now see that process_charge() catches
exceptions on line 318 and returns early before reaching the
send_receipt_email() call on line 335.
```

**Example 2: Pruning accumulated agent context in a multi-turn session**

```
User: "Find and fix the memory leak in the worker pool."

Turn 1: Agent reads worker_pool.py (400 lines), thread_manager.py (300 lines),
        config.py (200 lines). Total accumulated: ~900 lines of code in context.

Turn 2: Agent has identified the leak is in worker_pool.py's cleanup() method.

Focus Goal for context pruning: "Retain worker lifecycle management and
resource cleanup; prune thread_manager internals and config details."

Pruned context (~250 lines):
  - worker_pool.py: Full retention of __init__, spawn_worker(), cleanup(),
    and __del__. Pruned: logging setup, metrics collection, health checks.
  - thread_manager.py: Only retain the public interface (start/stop/join
    signatures + docstrings). Prune implementation details.
    # ... [260 lines: thread_manager internals — not needed for cleanup bug] ...
  - config.py: Only retain POOL_SIZE and CLEANUP_INTERVAL constants.
    # ... [180 lines: unrelated config] ...

Result: 72% reduction. Agent can now focus reasoning on the cleanup() method
without context pressure from irrelevant code.
```

**Example 3: Focused code review of a large PR**

```
User: "Review this PR for security issues. It touches 12 files,
       +500/-200 lines."

Focus Goal: "Identify user input handling, SQL queries, authentication
checks, and file system access in changed code."

Approach:
1. Read each changed file
2. For each file, retain only: functions that handle external input,
   database queries, auth decorators/middleware, file I/O operations
3. Prune: test boilerplate, CSS changes, logging adjustments,
   comment-only changes, import reordering
4. Mark pruned sections: # ... [PR: +30/-10 in test setup — no security surface] ...

Result: From 700 lines of diff, retain ~200 lines of security-relevant
changes for focused review.
```

## Best Practices

**Do:**
- Always write down the focus goal explicitly before pruning — this is the single most important step. Vague goals produce bad pruning.
- Preserve function/class signatures even when pruning their bodies, so the structural map of the file remains visible.
- Use descriptive pruning markers (`# ... [45 lines: CSV export utilities] ...`) rather than silent omission, so you or the user can request expansion later.
- Re-evaluate and tighten the focus goal if the first pass doesn't reduce enough. Narrower goals produce better pruning.

**Avoid:**
- Pruning code you haven't read yet. You must read the full content first to score relevance accurately — pruning is a second pass, not a shortcut for reading.
- Pruning based on "this looks like boilerplate" without checking against the goal. Boilerplate-looking code (e.g., `__init__` methods) often contains critical setup for the task.
- Removing all imports or all comments — imports tell you what dependencies matter, and comments near retained code often explain why the code works the way it does.
- Pruning in the middle of a control flow block (e.g., keeping the `if` but removing the `else`). Always prune at block boundaries.

## Error Handling

- **Over-pruning (lost critical context):** If after pruning you encounter a reference to something you removed (e.g., a variable defined in a pruned section), restore that specific section. Track these restoration events — if they happen frequently, your focus goal is too narrow.
- **Under-pruning (still too much context):** Tighten the focus goal. Instead of "understand the API," try "understand the POST /users endpoint's request validation."
- **Ambiguous relevance:** When unsure whether a block is relevant, keep it. False retention is cheaper than false pruning — you lose some token efficiency but don't lose correctness.
- **Pruning breaks syntax understanding:** If the pruned output makes it hard to understand the code structure, restore structural anchors (class definitions, function signatures) even if their bodies are pruned.

## Limitations

- **Small files (< 100 lines):** Pruning overhead exceeds benefit. Just read the whole file.
- **Highly interconnected code:** When every function calls every other function (tightly coupled code), pruning any part risks losing needed context. In these cases, refactoring is a better answer than pruning.
- **Exploratory tasks without a clear goal:** If the user says "look at this code and tell me what it does," there's no focus goal to prune against. Read fully first, then prune on follow-up questions.
- **Code with dense side effects:** If functions modify global state or have non-obvious side effects, pruning them away can hide important behavior. Be conservative with stateful code.
- **This is a reasoning technique, not a model call:** Without the actual 0.6B neural skimmer, Claude is performing goal-conditioned selection using its own judgment rather than a specialized scoring model. This works well but is not identical to the trained skimmer's decisions.

## Reference

**Paper:** [SWE-Pruner: Self-Adaptive Context Pruning for Coding Agents](https://arxiv.org/abs/2601.16746v2) — Wang et al., 2026. Look for Section 3 (method) for the goal formulation and line-scoring algorithm, and Section 4 (experiments) for compression ratios across SWE-Bench Verified and LongCodeQA.

**Code:** [github.com/Ayanami1314/swe-pruner](https://github.com/Ayanami1314/swe-pruner) — The `swe-pruner/hf/prune_wrapper.py` module contains the line-level scoring and threshold logic. The `examples/` directory shows integration with Claude Agent SDK and OpenHands.

**Model:** Available on HuggingFace at `ayanami-kitasan/code-pruner` (0.6B parameters). Runs as a FastAPI service on `localhost:8000/prune` for external integration.