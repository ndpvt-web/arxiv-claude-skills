---
name: "chain-mindset-reasoning-adaptive"
description: "Solve complex problems by switching between four cognitive mindsets (Spatial, Convergent, Divergent, Algorithmic) at each reasoning step, adapting the thinking mode to what the current sub-problem actually needs. Use when asked to: 'reason through this step by step', 'solve this hard problem', 'think carefully about this', 'use chain of mindset', 'adaptive reasoning', 'multi-step problem solving'."
---

# Chain of Mindset: Adaptive Cognitive Mode Reasoning

Chain of Mindset (CoM) enables Claude to solve complex problems by dynamically switching between four distinct cognitive modes at each reasoning step, rather than applying one uniform thinking style throughout. Inspired by how humans naturally shift between visualization, focused analysis, brainstorming, and computation when tackling hard problems, CoM decomposes reasoning into **Spatial** (visualize structure), **Convergent** (deep focused logic), **Divergent** (parallel exploration of alternatives), and **Algorithmic** (precise code-based computation) mindsets. A meta-cognitive selection step before each reasoning phase picks the mode best suited to the current sub-problem, and a context-gating discipline filters what information flows between steps to prevent context pollution.

## When to Use

- When the user asks you to solve a multi-step problem that mixes different kinds of reasoning (e.g., a math problem requiring both geometric intuition and algebraic computation)
- When a coding task involves architectural design decisions alongside precise implementation details
- When debugging requires both broad hypothesis generation and narrow logical verification
- When the user asks to "think through this carefully" or "reason step by step" on a non-trivial problem
- When a scientific or technical question needs both creative exploration of hypotheses and rigorous convergent analysis
- When a problem involves spatial/structural reasoning (data flow, system architecture, graph traversal) combined with algorithmic precision
- When you find yourself stuck in a single reasoning mode and the problem isn't yielding to it

## Key Technique

**The core insight**: Different stages within a single problem demand fundamentally different cognitive modes. A geometry proof might start with spatial visualization, shift to divergent exploration of proof strategies, then converge on one via focused logic, and finally verify with algorithmic computation. Existing chain-of-thought methods apply one fixed mode throughout, which is like trying to hammer every stage of carpentry with the same tool.

**The four mindsets** serve distinct functions. The **Spatial** mindset transforms abstract relationships into structural representations -- ASCII diagrams, dependency graphs, state machines, data flow sketches. The **Convergent** mindset performs deep, focused logical analysis grounded in established facts, following one thread to its conclusion without branching. The **Divergent** mindset generates 2-5 parallel solution branches when uncertainty is high, exploring each independently before selecting the most promising path. The **Algorithmic** mindset handles precise computation via code execution -- calculations, simulations, data transformations -- using a generate-execute-repair loop.

**The meta-cognitive step** is what makes this adaptive rather than scripted. Before each reasoning phase, evaluate the current state: What does the problem need *right now*? If you're stuck on understanding the structure, switch to Spatial. If you have too many possibilities, switch to Convergent to narrow down. If you're stuck in a rut, switch to Divergent. If you need exact numbers, switch to Algorithmic. The **context gate** discipline means each mindset receives only the relevant subset of prior reasoning, and its output is distilled to its essential insight before being carried forward.

## Step-by-Step Workflow

1. **Read the full problem and classify its domains.** Identify which cognitive demands are present: structural/spatial understanding, logical deduction, creative exploration, or precise computation. Most hard problems involve at least two.

2. **Select the initial mindset based on the problem's entry point.** If the problem presents a structure to understand (architecture, graph, geometry), start with Spatial. If it presents a clear logical chain to follow, start with Convergent. If the approach is unclear, start with Divergent. If it's primarily computational, start with Algorithmic.

3. **Execute the chosen mindset with filtered context.** Work within that mode using only the information relevant to the current sub-task. For Spatial: produce a concrete structural representation (ASCII diagram, bullet-point graph, state table). For Convergent: follow a single logical thread step by step to a conclusion. For Divergent: generate 2-5 distinct approaches, briefly analyze each, then select the most promising. For Algorithmic: write and mentally execute code or formal computation.

4. **Distill the output into a concise insight.** After completing a mindset phase, extract the key finding in 1-3 sentences. This is what carries forward -- not the full reasoning trace. This prevents context bloat and keeps subsequent steps focused.

5. **Re-evaluate: select the next mindset based on what you now know.** Ask: "Given what I just learned, what does the problem need next?" This is the critical adaptive step. Do not default to the same mode. Explicitly justify why the next mode is appropriate.

6. **Repeat the mindset-execute-distill-select cycle** until the problem reaches a solution or a termination condition (answer found, all branches exhausted, computation verified).

7. **Cross-verify the solution using a different mindset than the one that produced it.** If you arrived at the answer via Convergent logic, verify with Algorithmic computation. If via Divergent exploration, verify with Convergent focused analysis. This catches mode-specific blind spots.

8. **Synthesize the final answer** by integrating insights from all phases, presenting the solution with clear reasoning and appropriate confidence level.

## Concrete Examples

**Example 1: Debugging a Race Condition**

User: "Our API sometimes returns stale data. The endpoint reads from a cache that's updated by a background worker. It happens under load but we can't reproduce it consistently."

Approach:
1. **Spatial** -- Map the data flow: Request -> Cache Read -> Response, alongside Worker -> DB Read -> Cache Write. Draw the timing diagram to identify where reads and writes can interleave.
   - *Insight*: There's a window where the cache read happens between the worker's DB read and cache write, or where two worker cycles overlap.

2. **Divergent** -- Generate hypotheses: (a) Cache read-write is not atomic, (b) Worker has no lock and two instances run concurrently, (c) Cache TTL causes reads of expired-but-not-yet-refreshed entries, (d) Network delay causes worker writes to arrive out of order.
   - *Insight*: Hypotheses (a) and (b) are most consistent with "under load" trigger.

3. **Convergent** -- Focus on hypothesis (b): If the worker is triggered by a cron/interval and the previous run hasn't finished, two workers read DB at different times, then write to cache in unpredictable order. The slower worker overwrites the faster one's newer data.
   - *Insight*: The root cause is concurrent worker execution without mutual exclusion.

4. **Algorithmic** -- Produce the fix: Add a distributed lock (e.g., Redis SETNX) around the worker's read-write cycle, or use a single-flight pattern.

Output:
```
Root cause: Your background worker lacks mutual exclusion. Under load, overlapping
worker executions read the DB at different times and write to cache in
unpredictable order, causing stale overwrites.

Fix: Wrap the worker's cache-update cycle in a distributed lock:

  async function refreshCache() {
    const acquired = await redis.set('cache-lock', '1', 'NX', 'EX', 30);
    if (!acquired) return; // another worker is already running
    try {
      const data = await db.query('SELECT ...');
      await redis.set('cache-key', JSON.stringify(data));
    } finally {
      await redis.del('cache-lock');
    }
  }

Alternatively, use a single-flight/coalescing pattern if you want waiting
workers to receive the result of the in-progress refresh rather than skipping.
```

**Example 2: Designing a Rate Limiter**

User: "Implement a rate limiter that allows 100 requests per minute per user, with smooth distribution (not bursty)."

Approach:
1. **Divergent** -- Explore algorithms: (a) Fixed window counter, (b) Sliding window log, (c) Sliding window counter, (d) Token bucket, (e) Leaky bucket. Evaluate each against "smooth distribution" requirement.
   - *Insight*: Token bucket allows bursts. Fixed window has boundary burst problem. Sliding window log is memory-heavy. Leaky bucket enforces smooth output -- best fit.

2. **Convergent** -- Analyze leaky bucket: Requests enter a queue processed at a fixed rate (100/60s = 1 request per 0.6s). Excess requests are rejected. This guarantees even distribution.
   - *Insight*: Leaky bucket with queue size = 100 and drain rate = 1/600ms meets the requirement exactly.

3. **Algorithmic** -- Implement the solution:

Output:
```python
import time
from collections import defaultdict
import threading

class LeakyBucketRateLimiter:
    def __init__(self, capacity: int = 100, rate_per_second: float = 100 / 60):
        self.capacity = capacity
        self.rate = rate_per_second  # requests drained per second
        self._buckets: dict[str, dict] = defaultdict(
            lambda: {"water": 0.0, "last_check": time.monotonic()}
        )
        self._lock = threading.Lock()

    def allow(self, user_id: str) -> bool:
        with self._lock:
            bucket = self._buckets[user_id]
            now = time.monotonic()
            elapsed = now - bucket["last_check"]
            bucket["last_check"] = now

            # Drain water based on elapsed time
            bucket["water"] = max(0.0, bucket["water"] - elapsed * self.rate)

            if bucket["water"] < self.capacity:
                bucket["water"] += 1.0
                return True
            return False
```

**Example 3: Understanding a Complex Algorithm**

User: "Explain how this recursive function works and whether it's correct for finding all valid parentheses combinations."

Approach:
1. **Spatial** -- Trace the recursion tree for n=2: draw the branching structure showing how open/close counts evolve at each node, marking valid leaves.
   - *Insight*: The tree branches on "add open" or "add close" at each level, pruning when close > open or open > n.

2. **Convergent** -- Verify the pruning conditions are both necessary and sufficient: (a) open <= n prevents excess opens, (b) close <= open ensures validity at every prefix. Together they generate exactly the Catalan(n) valid strings.
   - *Insight*: The invariant `close <= open <= n` at every recursion point is both necessary and sufficient for generating all and only valid combinations.

3. **Algorithmic** -- Verify by counting: for n=3, the function should produce exactly C(3) = 5 results. Enumerate them mentally: ((())), (()()), (())(), ()(()), ()()().
   - *Insight*: Confirmed correct -- 5 results matching the 3rd Catalan number.

## Best Practices

**Do:**
- Explicitly name the mindset you're using at each step. This forces conscious selection rather than drifting into a default mode.
- Keep the distilled insight from each phase to 1-3 sentences. Verbosity between phases is the primary source of reasoning degradation.
- Switch mindsets when you feel stuck or when the nature of the sub-problem clearly changes (e.g., moving from "what approach?" to "execute the computation").
- Use Divergent mode early when the problem space is ambiguous -- it prevents premature commitment to a suboptimal path.

**Avoid:**
- Running more than one Divergent phase in sequence. If the first exploration didn't narrow things down, switch to Convergent to analyze what you have rather than generating more branches.
- Using Algorithmic mode for conceptual reasoning. Code execution is for precise computation, not for thinking through design tradeoffs.
- Carrying the full reasoning trace from one phase into the next. Apply the context gate: pass forward only the distilled insight, not the working notes.
- Defaulting to Convergent for every step. This is the most common failure mode -- it feels productive but misses structural insights (Spatial) and alternative approaches (Divergent).

## Error Handling

- **Mindset selection stalls**: If you can't decide which mindset to use next, default to Divergent to generate options, then immediately follow with Convergent to pick one.
- **Divergent produces no clear winner**: When all branches seem equally plausible, apply Algorithmic to test concrete cases for each, or apply Convergent to find a distinguishing criterion.
- **Spatial representation is too complex**: Simplify by focusing on the subgraph/substructure relevant to the current step rather than the entire system.
- **Algorithmic computation gives unexpected results**: Switch to Convergent to re-examine assumptions, then re-run with corrected logic. Do not iterate on code more than twice without re-evaluating the approach.
- **Context overload between phases**: If reasoning is getting muddled, pause and re-distill all accumulated insights into a clean 3-5 bullet summary before proceeding.

## Limitations

- **Simple problems don't benefit.** If a problem has a straightforward single-mode solution (pure computation, pure logic), the overhead of mode-switching adds complexity without value. Use CoM only when the problem genuinely spans multiple cognitive demands.
- **The meta-cognitive step requires judgment.** Mindset selection is itself a reasoning task and can be wrong. If a chosen mode isn't producing progress after reasonable effort, switch rather than persisting.
- **Divergent mode is expensive.** Generating 2-5 parallel branches multiplies reasoning effort. Use it when genuine uncertainty exists, not as a default.
- **Not a substitute for domain knowledge.** CoM improves reasoning *structure* but cannot compensate for missing domain expertise. It works best when Claude already has the relevant knowledge and the challenge is organizing the reasoning.
- **Spatial mode is limited to text.** Without image generation, spatial representations are ASCII diagrams and structural descriptions -- effective for architecture and data flow, less so for complex geometric reasoning.

## Reference

[Chain of Mindset: Reasoning with Adaptive Cognitive Modes](https://arxiv.org/abs/2602.10063v1) -- Jiang et al., 2026. See Section 3 for the formal mindset definitions and Meta-Agent selection policy, and Section 4 for ablation studies showing the Context Gate contributes 8.24% accuracy and Divergent mode is critical for mathematical reasoning (16.66% drop on AIME when removed).