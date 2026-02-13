---
name: "ctrlcot-dual-granularity-chain-of-thought-compress"
description: "Compress chain-of-thought reasoning using CtrlCoT's dual-granularity framework: hierarchical semantic abstraction combined with logic-preserving token pruning. Use when asked to 'compress reasoning', 'shorten chain of thought', 'optimize CoT tokens', 'reduce reasoning verbosity', 'budget-controlled reasoning', or 'efficient step-by-step thinking'."
---

# CtrlCoT: Dual-Granularity Chain-of-Thought Compression

This skill enables Claude to produce compressed chain-of-thought reasoning that uses significantly fewer tokens while preserving correctness. Based on the CtrlCoT framework, it combines two complementary strategies: **semantic abstraction** (rewriting reasoning at coarser granularity levels) and **logic-preserving pruning** (removing redundant tokens while retaining numbers, operators, and logical connectives). The result is reasoning traces that are 30-60% shorter than naive CoT while maintaining or improving accuracy, particularly on mathematical and logical tasks.

## When to Use

- When a user asks you to solve a complex problem but wants concise reasoning, not a verbose walkthrough
- When token budget is constrained and you need to reason through multi-step math, logic, or code problems efficiently
- When a user says "think step by step but keep it short" or "show your work briefly"
- When generating reasoning traces for downstream consumption (e.g., feeding into another prompt, logging, or auditing) where brevity matters
- When solving batches of similar problems where verbose CoT would be wasteful
- When a user explicitly asks to compress, shorten, or optimize chain-of-thought output
- When debugging or explaining code logic where the full derivation would obscure the key insight

## Key Technique

**Dual-granularity compression** addresses a fundamental tension: semantic-level compression (rewriting steps at a higher abstraction) is conservative and often leaves redundancy, while token-level pruning (deleting individual tokens) is aggressive and can destroy critical reasoning cues like numbers, variable names, and operators. CtrlCoT harmonizes both through three interlocking mechanisms.

**Hierarchical Reasoning Abstraction** generates reasoning at four semantic tiers -- Detailed, Standard, Concise, and Ultra-Concise -- each produced by controlling the verbosity of the explanation. The key insight is that a correct proof at the "Concise" level is not just a shorter version of "Detailed" -- it restructures the argument, merging steps and eliding obvious sub-derivations. Answer-consistency filtering ensures that only traces producing the correct final answer survive at each tier.

**Logic-Preserving Pruning** then operates within a chosen semantic tier, removing filler tokens (transition phrases, restatements, hedging language) while explicitly protecting logic-critical tokens: numerical values, mathematical operators, variable bindings, conditional keywords (`if`, `then`, `else`), and intermediate results. This is not generic summarization -- it is domain-aware compression that understands which tokens carry reasoning load. The pruning ratio is controllable via a parameter (0.3 to 1.0), where 0.3 means "keep 30% of tokens" and 1.0 means "keep everything." Empirically, 0.3-0.4 is the sweet spot for math reasoning.

**Distribution Alignment** resolves the final problem: pruned traces read like telegrams ("calculate... divide... get 7") which diverge from fluent reasoning style. The compressed trace is re-rendered into natural language that flows coherently at the target length, avoiding fragmentation while respecting the token budget.

## Step-by-Step Workflow

1. **Classify the reasoning task.** Determine whether the problem is mathematical, logical, code-based, or analytical. This determines which tokens are "logic-critical" (numbers and operators for math; variable names and control flow for code; premises and conclusions for logic).

2. **Generate a full-detail reasoning trace first.** Solve the problem with complete step-by-step reasoning internally. This is your "Detailed" tier -- the ground-truth derivation you will compress from.

3. **Select the target semantic granularity.** Choose one of four tiers based on the user's needs:
   - **Detailed**: Full derivation, every sub-step shown (~100% tokens)
   - **Standard**: Key steps with brief justifications (~60-70% tokens)
   - **Concise**: Only pivotal reasoning transitions (~35-50% tokens)
   - **Ultra-Concise**: Minimal representation, just the logical spine (~20-30% tokens)

4. **Rewrite at the chosen granularity.** Restructure the reasoning at the target level. This is not truncation -- merge sequential steps that share a single logical move, drop sub-derivations the reader can infer, and collapse intermediate arithmetic into single expressions.

5. **Identify and protect logic-critical tokens.** Before any further compression, tag tokens that must survive: all numerical values, operators (+, -, *, /, =, <, >), variable names, function names, conditional keywords, and final/intermediate results. These are non-negotiable.

6. **Prune filler tokens.** Remove transition phrases ("Let's think about this," "Now we need to," "This means that"), restatements of the problem, hedging language ("probably," "I think"), and redundant renamings. Preserve sentence structure around protected tokens.

7. **Re-render into fluent language.** Smooth the pruned trace so it reads naturally. Replace telegram-style fragments with grammatical sentences. Ensure each step logically connects to the next without requiring the reader to mentally reconstruct missing words.

8. **Verify answer consistency.** Re-derive the final answer from only the compressed trace. If the compressed reasoning no longer supports the correct answer, back off to a less aggressive compression tier and repeat from step 4.

9. **Present the compressed reasoning.** Output the final compressed CoT, optionally prefixed with the compression tier used and approximate token savings if the user requested that information.

## Concrete Examples

**Example 1: Mathematical problem with budget constraint**

User: "Solve this concisely: If f(x) = 2x^2 - 3x + 1, find f(5) - f(2)."

Full internal reasoning (Detailed tier, ~80 tokens):
```
f(5) = 2(25) - 3(5) + 1 = 50 - 15 + 1 = 36
f(2) = 2(4) - 3(2) + 1 = 8 - 6 + 1 = 3
f(5) - f(2) = 36 - 3 = 33
```

Compressed output (Concise tier, ~35 tokens):
```
f(5) = 2(25) - 15 + 1 = 36
f(2) = 2(4) - 6 + 1 = 3
Difference: 36 - 3 = **33**
```

What was removed: Redundant expansions (writing "3(5)" then "15" separately), transition words. What was preserved: Every number, every operator, every intermediate result.

---

**Example 2: Code debugging with compressed reasoning**

User: "Why does this function return None? Keep the explanation brief."
```python
def find_max(lst):
    if len(lst) == 0:
        return None
    max_val = lst[0]
    for item in lst:
        if item > max_val:
            max_val = item
```

Full reasoning (Detailed, ~120 tokens):
```
The function defines max_val and iterates through the list correctly.
The comparison logic is right. However, after the loop completes,
there is no return statement for max_val. Python functions without
an explicit return statement return None by default. The fix is to
add "return max_val" after the for loop.
```

Compressed output (Concise, ~40 tokens):
```
The loop logic is correct, but the function never returns `max_val` --
it falls off the end after the `for` block. Add `return max_val` after
the loop.
```

What was removed: Restatement of what the code does ("defines max_val and iterates"), explanation of Python's default None behavior (the user likely knows this). What was preserved: The root cause ("never returns max_val"), the location ("after the for block"), the fix.

---

**Example 3: Multi-step logic problem at Ultra-Concise tier**

User: "Compress your reasoning to the minimum. All widgets cost $5. Alice buys 3, Bob buys 7, Carol buys twice what Alice bought. Total spent by all three?"

Compressed output (Ultra-Concise, ~25 tokens):
```
Alice: 3*5=15, Bob: 7*5=35, Carol: 2*3*5=30.
Total: 15+35+30 = **$80**
```

Every number and operator preserved. Zero filler words. The reasoning spine is fully traceable.

## Best Practices

**Do:**
- Always generate the full reasoning internally before compressing -- compression is a post-processing step on correct reasoning, not a shortcut to skip thinking
- Protect numerical values, operators, and variable names unconditionally -- these are the load-bearing tokens of any derivation
- Match compression tier to the audience: use Standard for explanations, Concise for experienced users, Ultra-Concise for batch/automated contexts
- Verify answer consistency after compression -- if the compressed trace can't independently justify the answer, it's over-compressed

**Avoid:**
- Never compress by removing intermediate results -- "2+3=5, 5*4=20" should not become "2+3... *4=20" even if it saves tokens
- Never merge steps that cross logical boundaries (e.g., collapsing setup and solution into one sentence) -- each distinct logical move deserves its own clause
- Avoid removing conditional branches or edge-case handling from code reasoning -- these are logic-critical even though they look like filler
- Don't apply Ultra-Concise to problems where the user is learning -- concise reasoning aids experts but confuses novices

## Error Handling

**Compression destroys correctness:** If the compressed trace produces a different answer than the full trace, immediately back off one granularity tier. The hierarchy is your safety net: Concise fails? Use Standard. Standard fails? Use Detailed. Never ship a compressed trace that yields the wrong answer.

**Logic-critical token accidentally pruned:** If a number, operator, or variable name was dropped and the reasoning gap is noticed during verification, restore it and re-prune only the surrounding filler. The protected-token list is inviolable.

**Fluency degradation:** If the compressed trace reads as fragmented bullet points rather than coherent reasoning, spend a few tokens on connective tissue ("so," "thus," "giving"). A 5% token increase for readability is worth it -- the goal is compression, not minimalism at all costs.

**Ambiguous compression request:** If the user says "be brief" without specifying a tier, default to **Concise** (35-50% tokens). This balances informativeness with economy. Only use Ultra-Concise when explicitly requested or in automated pipelines.

## Limitations

- **Novel or creative reasoning** (open-ended brainstorming, exploratory analysis) compresses poorly because there is no single logical spine to preserve -- the "filler" often contains the value
- **Pedagogical contexts** where the user needs to learn the method, not just see the answer, should use Detailed or Standard tier; over-compression defeats the purpose
- **Ambiguous problems** with multiple valid solution paths lose important alternative-exploration when compressed -- the compressed trace commits to one path and discards others
- **Very short reasoning chains** (1-3 steps) gain negligible benefit from compression; the overhead of tier selection exceeds the savings
- **Domain-specific jargon** may be misclassified as filler and pruned if the task domain is unfamiliar -- when in doubt, protect any token that looks like a technical term

## Reference

**Paper:** [CtrlCoT: Dual-Granularity Chain-of-Thought Compression for Controllable Reasoning](https://arxiv.org/abs/2601.20467v1) (Fan et al., 2026). Focus on Section 3 for the three-component framework (HRA, LPD, DAG) and Table 2 for compression-ratio-vs-accuracy tradeoffs across MATH-500 and GSM8K benchmarks.