---
name: "scratcheval-multimodal-evaluation-framework"
description: "Evaluate, debug, and repair block-based Scratch programs using a three-layer executable protocol (VM execution, block-level edit distance, explanation rubrics). Use when: 'debug this Scratch project', 'analyze Scratch program logic', 'find bugs in my Scratch code', 'evaluate Scratch repair quality', 'convert Scratch blocks to analyzable AST', 'fix concurrency issues in Scratch sprites'."
---

# ScratchEval: Multimodal Evaluation Framework for Block-Based Programming

This skill enables Claude to systematically analyze, debug, and repair Scratch (and similar block-based) programs by applying ScratchEval's three-layer executable evaluation protocol. Instead of treating Scratch as a toy language, this approach recognizes its unique challenges — deeply nested non-linear control flow, event-driven concurrency across sprites, and tight coupling between code logic and multimedia assets — and provides structured methods to reason about correctness at the VM execution level, measure repair quality via block-level edit distance, and validate explanations against generated patches.

## When to Use

- When a user shares a Scratch project (.sb3 file) and asks to find or fix bugs in it
- When analyzing block-based program logic involving event broadcasts, sprite concurrency, or state management across clones
- When a user needs to understand why a Scratch program behaves incorrectly (e.g., sprites not responding to events, race conditions between scripts)
- When evaluating whether a proposed repair to a Scratch program is minimal and semantically correct rather than an invasive rewrite
- When converting Scratch block structures to JSON AST for programmatic analysis or LLM consumption
- When building or extending automated test suites for Scratch projects that validate runtime behavior
- When debugging educational Scratch projects where students have introduced subtle logic errors in event ordering or variable scoping

## Key Technique

**The core insight of ScratchEval is that block-based programs cannot be evaluated like text-based code.** Scratch programs are structured as JSON ASTs where blocks snap together in trees, scripts are triggered by hat blocks (events), multiple sprites run concurrently via broadcast-driven activation, and visual/audio assets are integral to program semantics. LLMs often generate repairs that are syntactically valid JSON but semantically wrong — they restructure entire script trees when only a single block swap was needed. ScratchEval counters this with constrained, measurable evaluation.

**The three-layer protocol works as follows.** Layer 1 (Functional Correctness) runs the repaired program in the Scratch VM and compares execution traces — sprite positions, variable states, broadcast events, and visual outputs — against reference traces from the correct program. Layer 2 (Repair Quality) computes block-level edit distance between the buggy program and the repair, enforcing that fixes respect block-type compatibility constraints and remain minimal (fewest block insertions, deletions, or moves). Layer 3 (Explanation Quality) uses structured rubrics to assess whether the model's reasoning about the bug correctly identifies the root cause, maps to the actual patch applied, and addresses the right semantic category (event ordering, concurrency, state management).

**The human-in-the-loop benchmark construction pipeline** combines automated mining of Scratch projects (filtering by structural complexity, sprite count, and script depth) with expert annotation of trigger-outcome semantics — the specific event sequences that must occur for a program to behave correctly. Bugs are categorized into event ordering errors (wrong hat block or missing broadcast), concurrency errors (race conditions between sprite scripts), and state management errors (incorrect variable initialization or scope). Each bug is paired with a minimal correct fix and block-level edit constraints defining the acceptable repair boundary.

## Step-by-Step Workflow

1. **Extract and parse the Scratch project**: Unzip the .sb3 file (it is a ZIP archive) and parse `project.json` into a structured JSON AST. Identify all sprites, their scripts (block trees rooted at hat blocks), variables, lists, and referenced assets (costumes, sounds).

2. **Build a script dependency graph**: For each sprite, map hat blocks to their trigger events (green flag, broadcast received, clone start, key press, backdrop change). Trace broadcast-send → broadcast-receive edges across sprites to construct the inter-sprite event flow graph. Flag any orphaned scripts (no trigger) or dead broadcasts (sent but never received).

3. **Identify the bug category**: Classify the reported or suspected issue into one of three primary categories:
   - **Event ordering**: A broadcast is sent before the receiving script is ready, or hat blocks fire in the wrong sequence.
   - **Concurrency**: Two sprites modify the same variable or visual state without synchronization, causing race conditions.
   - **State management**: A variable is not initialized before use, has wrong scope (local vs. global), or is overwritten by a clone.

4. **Trace execution behavior**: Simulate or describe the program's execution step by step, tracking sprite positions, variable values, costume changes, and broadcast events at each tick. Identify the exact divergence point where actual behavior departs from intended behavior.

5. **Propose a minimal repair**: Generate a fix that modifies the fewest possible blocks. Respect block-type compatibility — a `motion` block cannot replace a `control` block. Express the repair as specific block operations: insert block X after block Y, delete block Z, move block W from script A to script B, or change a parameter value in block V.

6. **Compute block-level edit distance**: Count the number of block-level operations (insert, delete, move, parameter-change) in your proposed repair. Compare against the complexity of the original bug. A good repair has edit distance proportional to the bug's scope — a single misplaced broadcast should require 1-2 block operations, not a script rewrite.

7. **Validate functional correctness**: Describe the expected execution trace after the repair. Verify that all trigger-outcome pairs now produce correct results: the right sprites move to the right positions, variables reach expected values, and broadcast chains complete in the correct order.

8. **Explain the repair with structured reasoning**: Provide an explanation that explicitly links (a) the identified root cause, (b) the bug category, (c) the specific blocks modified, and (d) why the fix restores correct behavior. The explanation should be verifiable against the patch — every claim about the bug must correspond to a block operation in the repair.

9. **Check for side effects**: Verify that the repair does not break other scripts or sprites. Trace through the event dependency graph to confirm no downstream broadcasts, variable reads, or clone behaviors are affected by the changed blocks.

10. **Output the repaired project.json**: If working with actual files, produce the modified JSON AST with only the changed blocks updated. Preserve all asset references, sprite ordering, and metadata.

## Concrete Examples

**Example 1: Event Ordering Bug — Broadcast Race Condition**

User: "My Scratch project has two sprites. Sprite1 sends a broadcast 'start-game' when the green flag is clicked, and Sprite2 is supposed to move to position (100, 100) when it receives 'start-game'. But Sprite2 doesn't move. What's wrong?"

Approach:
1. Parse the script structure: Sprite1 has `when green flag clicked → broadcast 'start-game'`. Sprite2 has `when I receive 'start-game' → glide to x:100 y:100`.
2. Build event graph: green flag → Sprite1 broadcasts 'start-game' → Sprite2 receives 'start-game'.
3. Check broadcast name matching: Verify the broadcast name is identical in both send and receive blocks (case-sensitive, whitespace-sensitive).
4. Identify divergence: The broadcast in Sprite1 is `'start-game '` (trailing space) while Sprite2 listens for `'start-game'` (no space). The names don't match.
5. Minimal repair: Change the broadcast message parameter in Sprite1's broadcast block from `'start-game '` to `'start-game'`. Edit distance: 1 parameter change.

Output:
```
Bug Category: Event Ordering (broadcast name mismatch)
Root Cause: Sprite1 broadcasts 'start-game ' (trailing space) but Sprite2
            listens for 'start-game' — these are distinct message names.
Repair: In Sprite1's script, change the broadcast block's MESSAGE field
        from "start-game " to "start-game".
Edit Distance: 1 (single parameter change)
Verification: After fix, green flag → Sprite1 sends 'start-game' →
              Sprite2 receives it → glides to (100, 100). ✓
```

**Example 2: State Management Bug — Variable Not Reset Between Clones**

User: "I have a Scratch game where a sprite clones itself to create enemies. Each clone should start with health=3, but after the first clone is destroyed, new clones start with health=0."

Approach:
1. Parse clone lifecycle: Main sprite has `when green flag clicked → forever [create clone of myself]`. Clone has `when I start as a clone → [move, interact, check health]`.
2. Check variable scope: Variable `health` is a "for this sprite only" variable (local).
3. Trace execution: Clone 1 starts, health is set somewhere to 3, takes damage, health reaches 0, clone deletes itself. Clone 2 starts — but `health` is never explicitly set to 3 in `when I start as a clone`.
4. Identify bug: The `set health to 3` block is in the green flag script (runs once for parent) but not in the clone-start script. Clones inherit the parent's current variable state, which is 0 after the first clone was destroyed.
5. Minimal repair: Insert `set health to 3` as the first block inside the `when I start as a clone` hat block.

Output:
```
Bug Category: State Management (variable not initialized in clone lifecycle)
Root Cause: "health" is set to 3 only in the green-flag script. Clones
            inherit the parent sprite's current variable value, which
            becomes 0 after any clone is destroyed and the parent's
            variable is left at 0.
Repair: Insert [set health to 3] as the first block under
        [when I start as a clone] in the clone sprite's scripts.
Edit Distance: 1 (single block insertion)
Verification: Each new clone now initializes health=3 independently of
              parent state. Clone destruction no longer affects subsequent
              clones. ✓
```

**Example 3: Concurrency Bug — Two Scripts Racing on Shared Variable**

User: "In my Scratch project, Sprite1 and Sprite2 both modify a global variable 'score'. Sometimes the score doesn't update correctly when both sprites detect a collision at the same time."

Approach:
1. Parse scripts: Sprite1 has `when green flag clicked → forever [if touching Sprite2 then change score by 1]`. Sprite2 has `when green flag clicked → forever [if touching Sprite1 then change score by 1]`.
2. Identify concurrency pattern: Both sprites run `forever` loops checking the same collision from both sides. A single collision triggers both scripts, incrementing score by 2 instead of 1.
3. Trace execution: Frame N — sprites touch. Sprite1's loop detects collision, increments score. Sprite2's loop also detects collision (same frame), increments score again. Score increases by 2 per collision event.
4. Minimal repair: Remove the collision check from one sprite. Keep collision detection only in Sprite1. Delete the `if touching` block and its contents from Sprite2's forever loop, or replace the entire Sprite2 collision script with a different responsibility.

Output:
```
Bug Category: Concurrency (duplicate collision detection on shared state)
Root Cause: Both sprites independently detect the same collision event
            and both modify the global "score" variable. Scratch's
            concurrent execution model runs both forever-loops each frame,
            so a single collision triggers two increments.
Repair: Delete the [if touching Sprite1] conditional and its nested
        [change score by 1] block from Sprite2's forever loop.
        Collision scoring should be owned by exactly one sprite.
Edit Distance: 2 (delete conditional block + delete nested change block)
Verification: Single collision now triggers exactly one score increment
              from Sprite1's script only. ✓
```

## Best Practices

- **Do** always express repairs as specific block operations (insert, delete, move, parameter-change) rather than describing the fix in prose alone. Block-level precision prevents ambiguity.
- **Do** trace the full event dependency graph before proposing fixes. A bug that appears to be in one sprite may actually be caused by a missing or misordered broadcast from another sprite.
- **Do** check broadcast name matching exactly — Scratch broadcast names are case-sensitive strings, and trailing whitespace is a common source of silent failures.
- **Do** consider clone lifecycle carefully. Clones inherit parent variable state at creation time, and "for this sprite only" variables behave differently for clones vs. the parent.
- **Avoid** proposing repairs that restructure entire scripts when a single block change suffices. Invasive edits are the primary failure mode for LLMs working with Scratch — always minimize edit distance.
- **Avoid** treating Scratch as sequential code. Multiple scripts within a sprite and across sprites execute concurrently each frame. Never assume execution order between parallel scripts unless explicit synchronization (broadcast-wait) is used.

## Error Handling

- **Malformed project.json**: If the JSON AST is corrupted or uses an unsupported Scratch version (pre-3.0), flag this immediately. Scratch 2.0 and 1.x use entirely different project formats.
- **Missing asset references**: If costumes or sounds referenced in blocks are absent from the project archive, note these as potential runtime errors but continue analyzing code logic.
- **Ambiguous bug reports**: When the user describes unexpected behavior without specifying which sprites or scripts are involved, ask them to identify the specific sprites and the expected vs. actual behavior before attempting diagnosis.
- **Multiple interacting bugs**: If the program has several bugs, isolate and address them one at a time, starting with event ordering issues (which often cascade into apparent concurrency or state bugs that resolve once events fire correctly).
- **Unfamiliar custom blocks (procedures)**: Scratch allows user-defined blocks. Inline the procedure body at call sites before analyzing control flow to avoid missing interactions.

## Limitations

- This approach is most effective for Scratch 3.0 projects. Scratch 2.0 and 1.x use different block representations and VM semantics.
- Without actually running the Scratch VM, execution trace analysis is simulated through reasoning. Subtle timing-dependent bugs (exact frame ordering of concurrent scripts) may require actual VM execution to confirm.
- Multimedia-dependent bugs (e.g., wrong costume triggering incorrect collision hitbox) require visual inspection of assets, which text-based analysis alone cannot fully resolve.
- The three-layer protocol assumes a reference correct program exists for comparison. For novel programs where no reference is available, Layer 1 (functional correctness) relies on the user's description of intended behavior, which may itself be incomplete.
- Scratch's extension blocks (pen, music, video sensing, micro:bit) have specialized semantics not covered by the core block analysis and may require domain-specific knowledge.

## Reference

**Paper**: [ScratchEval: A Multimodal Evaluation Framework for LLMs in Block-Based Programming](https://arxiv.org/abs/2602.00757v1) — Si et al., 2026. Key sections: the three-layer executable protocol (Section 3), block-level edit distance constraints (Section 4), and the bug taxonomy covering event ordering, concurrency, and state management patterns (Section 5).