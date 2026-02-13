---
name: "contextual-drag-errors-context"
description: >
  Mitigate contextual drag — the phenomenon where failed attempts in conversation context
  bias LLM reasoning toward structurally similar errors. Apply context-denoising and
  fallback-reasoning strategies when iterating on broken code, debugging multi-step failures,
  or refining solutions that keep failing in similar ways.
  Trigger phrases:
  - "I keep getting the same kind of error"
  - "My fix attempt made it worse"
  - "The refactored code has the same bug pattern"
  - "Each iteration introduces a similar failure"
  - "Self-correction loop isn't converging"
  - "Why does my retry keep failing the same way"
---

# Contextual Drag Mitigation for Iterative Code Reasoning

This skill equips Claude to recognize and counteract **contextual drag** — a failure mode where
prior incorrect attempts in the conversation context bias subsequent reasoning toward
structurally similar errors. Based on research showing 10-20% accuracy drops when LLMs condition
on their own failed outputs (Cheng et al., 2026), this skill applies context-denoising and
fallback-reasoning techniques to break out of repetitive error cycles during debugging,
refactoring, and iterative code generation.

## When to Use

- When a user reports that successive fix attempts produce the same category of bug (e.g., repeated off-by-one errors, recurring null reference patterns, repeated misuse of an API).
- When an iterative debugging session has gone through 2+ rounds without converging on a working solution.
- When refactoring code that was already attempted and reverted, and the new attempt risks inheriting structural flaws from the old one.
- When a self-correction loop (e.g., run tests, read errors, fix, repeat) keeps producing failing code with similar test output.
- When a user pastes multiple failed attempts and asks "why do all my approaches fail the same way?"
- When reviewing a conversation history heavy with prior incorrect code and preparing a fresh solution.

## Key Technique

**Contextual drag** occurs because attention-based architectures lack mechanisms to truly reset
their reasoning state. When an incorrect solution appears in context — even explicitly labeled as
wrong — the model's next generation is measurably biased toward the structural shape of that
error. Tree edit distance analysis confirms that conditioned responses remain structurally closer
to erroneous drafts than clean-slate responses would be. This is not a matter of the model
"copying" the error verbatim; it inherits the *shape* of the mistake (e.g., the same
decomposition strategy, the same ordering of operations, the same architectural choice that
caused the failure).

This has direct consequences for coding workflows. When Claude sees three failed attempts at
fixing a race condition — all using the same locking strategy — the fourth attempt will
gravitationally pull toward that same strategy even if the correct fix requires a fundamentally
different approach (e.g., switching from locks to message passing). Simply telling the model
"the previous approach was wrong" does not eliminate the bias; the structural pattern persists
in the attention computation.

Two mitigation strategies show partial effectiveness: **context denoising** (filtering or
rewriting prior attempts to remove structural error patterns before generating the next attempt)
and **fallback-behavior reasoning** (deliberately abandoning the prior approach and reasoning
from scratch using only the original problem specification). Neither fully eliminates the drag,
but combining them substantially reduces it. The key insight for practitioners: when stuck in a
loop, *remove the failing context* rather than annotating it.

## Step-by-Step Workflow

1. **Detect the drag signal.** Count how many prior attempts exist in the conversation context for the same problem. If there are 2+ failed attempts, contextual drag is likely active. Look for structural similarity across failures — same function decomposition, same algorithm family, same error category.

2. **Classify the error pattern.** Identify what is structurally shared across the failed attempts. Is it the same algorithmic approach? The same data structure choice? The same API usage pattern? The same control flow shape? Name the shared structural element explicitly.

3. **Isolate the original specification.** Extract the *original problem statement* — what the code should do, what inputs it takes, what outputs it produces, what constraints it must satisfy — stripping away all implementation details from prior attempts.

4. **Apply context denoising (Filter strategy).** Mentally discard all prior implementation attempts. Do not reference, build upon, or "fix" any previous code. Treat the problem as if encountered for the first time. If the user's failed code contains useful *domain knowledge* (e.g., correct API endpoints, valid schema definitions), extract only those factual elements and discard the surrounding logic.

5. **Generate a structurally divergent approach.** Deliberately choose a different algorithmic strategy, data structure, or architectural pattern from what was used in the failed attempts. If prior attempts used recursion, consider iteration. If they used mutation, consider immutable transforms. If they used callbacks, consider async/await. The goal is maximum structural distance from the error pattern identified in step 2.

6. **Build incrementally with verification checkpoints.** Write the new solution in small, independently testable units. After each unit, verify correctness before proceeding. This prevents a single structural error from propagating through the entire solution.

7. **Compare structural divergence before delivering.** Before presenting the solution, verify that it does not share the structural pattern identified in step 2. If it does, return to step 5 and choose a more divergent approach.

8. **Communicate the reset to the user.** Explicitly tell the user that you are starting from scratch rather than iterating on prior attempts, and explain why. This sets expectations and prevents them from asking you to "just fix" the previous code (which would reintroduce the drag).

9. **If the fresh approach also fails**, do NOT iterate on it within the same context. Instead, summarize only the *factual learnings* (e.g., "endpoint X requires header Y", "the input can contain nulls") and use those as constraints for yet another structurally independent attempt.

## Concrete Examples

**Example 1: Breaking out of a repeated async error pattern**

```
User: I've tried fixing this three times. My Express middleware keeps throwing
"Cannot set headers after they are sent to the client." Each fix attempt
just moves the error to a different branch.

Prior attempts in context: Three versions of the middleware, all using
early-return guards with res.send() scattered across conditional branches.

Approach:
1. DETECT: Three failed attempts, all producing the same error class.
2. CLASSIFY: Shared structure — multiple res.send() calls across branching
   conditionals, relying on early returns that sometimes don't fire.
3. ISOLATE: The middleware should validate auth tokens, attach user info
   to req, and call next() on success or respond with 401/403 on failure.
4. DENOISE: Discard all three implementations entirely.
5. DIVERGE: Instead of scattered res.send() calls, use a single-exit
   pattern — accumulate the result in a variable and send exactly once
   at the end.

Output:
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return void res.status(401).json({ error: 'Token required' });
  }

  let decoded;
  try {
    decoded = jwt.verify(token, SECRET);
  } catch (err) {
    return void res.status(401).json({ error: 'Invalid token' });
  }

  req.user = decoded;
  next();
}

Note to user: "I'm starting fresh rather than patching the previous
attempts. The prior versions all shared a structural pattern — multiple
response paths with conditional early returns — that made the
headers-already-sent error likely. This version uses explicit early
returns with void to guarantee single-response semantics."
```

**Example 2: Escaping a recursive algorithm trap**

```
User: My tree-flattening function keeps hitting stack overflow for deep
trees. I've rewritten it twice but both rewrites are still recursive.
Can you fix the latest version?

Prior attempts in context: Two recursive implementations with different
base cases but identical recursive structure.

Approach:
1. DETECT: Two failed attempts, both hitting the same failure (stack overflow).
2. CLASSIFY: Shared structure — recursive traversal. The error is inherent
   to the approach, not the implementation details.
3. ISOLATE: Need a function that takes a nested tree and returns a flat
   array of all node values in depth-first order.
4. DENOISE: Do not "fix" the recursive version. The recursion IS the bug
   for deep trees.
5. DIVERGE: Use an explicit stack (iterative DFS) — structurally opposite
   to recursion.

Output:
function flattenTree(root) {
  const result = [];
  const stack = [root];

  while (stack.length > 0) {
    const node = stack.pop();
    if (!node) continue;
    result.push(node.value);
    // Push right first so left is processed first (DFS order)
    for (let i = node.children.length - 1; i >= 0; i--) {
      stack.push(node.children[i]);
    }
  }

  return result;
}

Note to user: "Rather than debugging the recursive version, I've replaced
it with iterative DFS using an explicit stack. The prior attempts both
shared the recursive structure that causes stack overflow on deep trees —
patching the recursion would likely produce another variant of the same
failure."
```

**Example 3: Schema migration stuck in a validation loop**

```
User: My database migration keeps failing validation. I've written three
versions of the migration script and they all fail with different
constraint violations.

Prior attempts in context: Three migration scripts, all trying to ALTER
TABLE in a single transaction with the same column ordering.

Approach:
1. DETECT: Three failures, all constraint violations during ALTER TABLE.
2. CLASSIFY: Shared structure — single-transaction bulk ALTER with
   dependent columns modified simultaneously. Each attempt reorders the
   ALTERs but keeps them in one transaction.
3. ISOLATE: Need to add a non-null foreign key column, backfill it, and
   drop the old column.
4. DENOISE: Discard all three scripts.
5. DIVERGE: Use a multi-step migration pattern — separate migrations for
   (a) add nullable column, (b) backfill data, (c) add NOT NULL
   constraint, (d) drop old column. Structurally different from the
   single-transaction approach.

Output:
-- Migration 1: Add column (nullable)
ALTER TABLE orders ADD COLUMN customer_id INT REFERENCES customers(id);

-- Migration 2: Backfill
UPDATE orders SET customer_id = legacy_customer_map.new_id
FROM legacy_customer_map
WHERE orders.old_customer_ref = legacy_customer_map.old_ref;

-- Migration 3: Add constraint
ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;

-- Migration 4: Clean up
ALTER TABLE orders DROP COLUMN old_customer_ref;
```

## Best Practices

**Do:**
- Explicitly name the shared structural pattern across failed attempts before generating a new solution. This forces conscious divergence.
- Extract factual constraints (API specs, schema definitions, type signatures) from failed attempts while discarding their implementation logic.
- Tell the user when you are deliberately ignoring their prior code. Transparency prevents confusion and repeat requests to "just fix the existing code."
- Prefer solutions with maximum structural distance from the failed pattern, even if they seem less elegant. Correctness first.

**Avoid:**
- Do not "fix" or "patch" code from a failed attempt when 2+ structurally similar failures exist. The fix will inherit the structural bias.
- Do not keep failed code in your reasoning as a "reference." Even negative references ("don't do X") anchor the structural pattern.
- Do not assume that labeling prior code as "incorrect" neutralizes its influence. Research shows explicit error labels do not eliminate contextual drag.
- Do not iterate more than twice on the same structural approach. If two attempts with the same shape fail, the shape is the problem.
- Do not use self-refinement prompts like "improve this code" when the code has fundamental structural issues. Refinement preserves structure by design.

## Error Handling

**Drag persists after one reset:** If the fresh approach fails in a *structurally different* way, that is progress — the drag has been broken. Debug the new error normally. If it fails in a *structurally similar* way, the problem specification itself may be pulling toward that structure. Re-examine constraints.

**User insists on patching existing code:** Explain the contextual drag phenomenon briefly. If they still want patches, comply but flag the risk. Suggest running the patched version alongside a clean-slate version for comparison.

**The structurally divergent approach is worse:** This can happen. If the divergent approach introduces its own problems, extract the factual learnings from both the original failures and the new failure, then attempt a third structural family. Do not merge the approaches — that reintroduces drag.

**Too many failed attempts in context (5+):** The context is heavily contaminated. Summarize the problem specification and all factual constraints in a compact block. Recommend the user start a new conversation with only that summary, eliminating the accumulated error context entirely.

## Limitations

- Contextual drag mitigation reduces but does not fully eliminate the bias. Even with explicit denoising, residual structural influence persists in attention-based architectures.
- This technique is most valuable for *reasoning-heavy* tasks (algorithm design, architectural decisions, complex debugging). For simple bugs with obvious fixes, standard iteration works fine.
- The "structurally divergent" approach requires that multiple valid solution structures exist. For problems with only one viable approach, divergence is not possible — instead, focus on isolating which *part* of the structure is wrong.
- Starting from scratch has a cost: useful partial progress from prior attempts is discarded. This is a deliberate tradeoff — the research shows the cost of inheriting structural errors typically exceeds the cost of re-deriving correct partial work.
- This skill addresses Claude's own reasoning biases. It cannot fix cases where the user's problem specification is itself incorrect or ambiguous.

## Reference

**Paper:** [Contextual Drag: How Errors in the Context Affect LLM Reasoning](https://arxiv.org/abs/2602.04288v1) (Cheng, Zhu, Zhao, Arora, 2026). Key findings: tree edit distance analysis proving structural inheritance of errors, the fallback-behavior and context-denoising mitigations, and evidence that verification signals alone cannot overcome the bias.