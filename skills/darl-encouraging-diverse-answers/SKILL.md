---
name: "darl-encouraging-diverse-answers"
description: "Generate diverse, high-quality answer variants for open-ended tasks using DARL's bounded-diversity framework. Use when: 'generate multiple solutions', 'diverse answers to this prompt', 'brainstorm varied approaches', 'explore alternative implementations', 'rewrite this N different ways', 'what are different ways to solve this'."
---

# DARL: Encouraging Diverse Answers within Bounded Deviation

This skill applies the DARL (Diverse Answers via Reinforcement Learning) framework to generate multiple high-quality, meaningfully distinct solutions to a problem while keeping each variant within a controlled deviation range from a reference answer. Instead of producing near-duplicate paraphrases or wildly divergent outputs, DARL's core insight is to reward diversity only when it stays within a confidence-scaled boundary of the reference quality -- producing answers that are genuinely different in approach but equivalently correct.

## When to Use

- When a user asks for multiple implementations of the same feature (e.g., "show me 3 different ways to implement a cache")
- When brainstorming architectural approaches where several valid designs exist
- When generating diverse test cases or edge cases for a function
- When rewriting prose, documentation, or commit messages in meaningfully different styles
- When exploring alternative algorithms for a problem (e.g., "what are different ways to sort this data given these constraints?")
- When a user needs varied prompt templates, API response formats, or data schemas for the same requirement
- When generating multiple candidate refactors and the user wants to compare tradeoffs

## Key Technique

**The Overfitting Problem.** Standard approaches to generating multiple answers tend to cluster around a single "best" reference. If you ask for five implementations, you often get the same logic with superficial variable renaming. RLPR-style methods that train against reference answers exacerbate this by rewarding proximity to one canonical solution. DARL addresses this by adding a bounded diversity incentive.

**Bounded Diversity Reward.** DARL modifies the reward signal with a two-part formula: `r_total = alpha * r_reference + beta * delta_r * I[delta_r <= r_reference / gamma]`. Here, `r_reference` measures alignment with the reference answer, `delta_r` measures how much a candidate deviates from the reference, and the indicator function `I[...]` gates the diversity bonus so it only applies when deviation is within `r_reference / gamma`. The key hyperparameters are `beta` (diversity weight, default 0.01) and `gamma` (exploration bound, default 8-10). This means: small, controlled deviations from the reference are rewarded; large deviations that risk correctness are not.

**Why This Works for Code Generation.** When applied to generating diverse code solutions, the reference answer acts as a quality anchor. Variants that change algorithm choice, data structure, or control flow are rewarded (bounded deviation), while variants that introduce bugs or miss requirements are penalized (exceeding the bound). The gamma parameter naturally scales -- high-confidence problems (simple utility functions) allow wider exploration, while low-confidence problems (complex algorithms) constrain diversity to safer variants.

## Step-by-Step Workflow

1. **Establish the reference solution.** Solve the problem once with your best approach. This becomes `y*`, the reference against which diversity is measured. Ensure it is correct and complete before proceeding.

2. **Identify diversity axes.** Analyze the reference to determine which dimensions can vary meaningfully: algorithm choice, data structure, API style (callback vs. promise vs. async/await), error handling strategy, naming conventions, architectural pattern, or level of abstraction.

3. **Set the deviation bound.** For well-defined problems (math, algorithms with known solutions), use a tight bound (gamma ~ 10): variants should differ in implementation but converge on the same result. For open-ended tasks (writing, architecture, UI), use a loose bound (gamma ~ 6-8): variants can differ more substantially in structure and style.

4. **Generate candidate variants.** For each variant, change exactly one or two diversity axes from the reference. Produce N candidates (typically 3-5), each exploring a different axis. Do not try to maximize difference -- aim for meaningful, purposeful divergence.

5. **Apply the bounded filter.** For each candidate, assess: does this variant still satisfy all the original requirements? If it introduces a correctness regression (fails a test case the reference passes, omits a requirement, introduces a vulnerability), it exceeds the deviation bound and should be discarded or revised.

6. **Score and rank candidates.** Rank remaining variants by the combined metric: quality (does it work as well as the reference?) plus diversity bonus (how different is it from the reference and from other candidates?). Prefer a set of variants that are mutually diverse, not just diverse from the reference.

7. **Annotate tradeoffs.** For each variant, document what it gains and what it costs relative to the reference. Be explicit: "This variant uses O(1) lookup but O(n) memory" or "This approach is more readable but less performant for large inputs."

8. **Present as a structured comparison.** Deliver the variants in a format that enables direct comparison: side-by-side code blocks, a tradeoff table, or a decision matrix. Never present diverse answers as an undifferentiated list.

## Concrete Examples

**Example 1: Diverse implementations of a rate limiter**

User: "Show me 3 different ways to implement a rate limiter in Python."

Approach:
1. Reference solution: Token bucket algorithm using `time.monotonic()`
2. Diversity axes identified: algorithm choice (token bucket vs. sliding window vs. fixed window), storage backend (in-memory dict vs. deque vs. counter)
3. Bound: tight (gamma=10) -- all must correctly limit rates

Output:

```python
# Variant A: Token Bucket (reference)
# Tradeoff: Smooth rate limiting, allows bursts up to bucket capacity
import time

class TokenBucketLimiter:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate
        self.capacity = capacity
        self.tokens = capacity
        self.last_refill = time.monotonic()

    def allow(self) -> bool:
        now = time.monotonic()
        self.tokens = min(self.capacity, self.tokens + (now - self.last_refill) * self.rate)
        self.last_refill = now
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False
```

```python
# Variant B: Sliding Window Log
# Tradeoff: Exact counting (no bursts), but O(n) memory per window
from collections import deque
import time

class SlidingWindowLimiter:
    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window = window_seconds
        self.timestamps = deque()

    def allow(self) -> bool:
        now = time.monotonic()
        while self.timestamps and self.timestamps[0] <= now - self.window:
            self.timestamps.popleft()
        if len(self.timestamps) < self.max_requests:
            self.timestamps.append(now)
            return True
        return False
```

```python
# Variant C: Fixed Window Counter
# Tradeoff: O(1) memory and time, but allows 2x burst at window boundaries
import time

class FixedWindowLimiter:
    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window = window_seconds
        self.count = 0
        self.window_start = time.monotonic()

    def allow(self) -> bool:
        now = time.monotonic()
        if now - self.window_start >= self.window:
            self.count = 0
            self.window_start = now
        if self.count < self.max_requests:
            self.count += 1
            return True
        return False
```

| Variant | Burst Behavior | Memory | Precision |
|---------|---------------|--------|-----------|
| A: Token Bucket | Controlled bursts | O(1) | Approximate |
| B: Sliding Window | No bursts | O(n) | Exact |
| C: Fixed Window | 2x burst at edges | O(1) | Approximate |

---

**Example 2: Diverse approaches to error handling in an API client**

User: "What are different ways to handle errors in this HTTP client?"

Approach:
1. Reference: try/except with custom exception hierarchy
2. Diversity axes: error representation (exceptions vs. Result type vs. error codes), retry strategy (none vs. exponential backoff vs. circuit breaker)
3. Bound: loose (gamma=8) -- these are genuinely different philosophies

Output:

```python
# Variant A: Exception hierarchy (reference)
# Philosophy: Fail fast, let callers decide recovery
class ApiError(Exception):
    def __init__(self, status: int, message: str):
        self.status = status
        self.message = message

def fetch(url: str) -> dict:
    resp = requests.get(url)
    if not resp.ok:
        raise ApiError(resp.status_code, resp.text)
    return resp.json()
```

```python
# Variant B: Result monad pattern
# Philosophy: Make errors explicit in return types, no hidden control flow
from dataclasses import dataclass
from typing import TypeVar, Generic

T = TypeVar("T")

@dataclass
class Ok(Generic[T]):
    value: T

@dataclass
class Err:
    status: int
    message: str

Result = Ok[T] | Err

def fetch(url: str) -> Result[dict]:
    resp = requests.get(url)
    if not resp.ok:
        return Err(resp.status_code, resp.text)
    return Ok(resp.json())
```

```python
# Variant C: Retry with circuit breaker
# Philosophy: Resilience over simplicity, self-healing on transient failures
import time

class CircuitBreaker:
    def __init__(self, failure_threshold=3, reset_timeout=30):
        self.failures = 0
        self.threshold = failure_threshold
        self.reset_timeout = reset_timeout
        self.last_failure = 0.0
        self.open = False

    def fetch(self, url: str) -> dict:
        if self.open:
            if time.monotonic() - self.last_failure > self.reset_timeout:
                self.open = False
            else:
                raise ConnectionError("Circuit open, backing off")
        try:
            resp = requests.get(url, timeout=5)
            resp.raise_for_status()
            self.failures = 0
            return resp.json()
        except requests.RequestException as e:
            self.failures += 1
            self.last_failure = time.monotonic()
            if self.failures >= self.threshold:
                self.open = True
            raise
```

---

**Example 3: Diverse SQL query strategies**

User: "Give me different ways to find the top 5 customers by total spend."

Approach:
1. Reference: Simple GROUP BY with ORDER BY and LIMIT
2. Diversity axes: query technique (subquery vs. CTE vs. window function), readability vs. performance
3. Bound: tight (gamma=10) -- must return identical results

Output:

```sql
-- Variant A: Direct aggregation (reference)
-- Tradeoff: Simplest, works everywhere, may scan full table
SELECT customer_id, SUM(amount) AS total_spend
FROM orders
GROUP BY customer_id
ORDER BY total_spend DESC
LIMIT 5;

-- Variant B: CTE for readability and reuse
-- Tradeoff: Clearer intent, can reference totals multiple times
WITH customer_totals AS (
    SELECT customer_id, SUM(amount) AS total_spend
    FROM orders
    GROUP BY customer_id
)
SELECT * FROM customer_totals
ORDER BY total_spend DESC
LIMIT 5;

-- Variant C: Window function (useful when you also need per-order detail)
-- Tradeoff: Returns order-level rows with rank, heavier but more flexible
SELECT * FROM (
    SELECT customer_id, amount,
           DENSE_RANK() OVER (ORDER BY SUM(amount) OVER (PARTITION BY customer_id) DESC) AS rnk
    FROM orders
) ranked
WHERE rnk <= 5;
```

## Best Practices

- **Do:** Always produce the reference (best single answer) first. Diversity without a quality anchor produces noise, not variety.
- **Do:** Vary on meaningful axes (algorithm, architecture, paradigm) rather than superficial ones (variable names, comment style, whitespace).
- **Do:** Explicitly state the tradeoff for each variant. Diverse answers without annotated tradeoffs force the user to do the analysis themselves.
- **Do:** Scale the deviation bound to problem confidence. Tight bounds for correctness-critical code (crypto, financial calculations), loose bounds for subjective tasks (UI layouts, documentation style).
- **Avoid:** Generating more variants than the problem supports. If there are only two meaningfully different approaches, present two -- not five padded variants.
- **Avoid:** Treating diversity as an end in itself. A variant that is different but strictly worse on every dimension should be filtered out, not included for variety.

## Error Handling

- **Variant fails correctness check:** Discard and regenerate along a different diversity axis. Do not relax the quality bound to preserve a "creative" but broken solution.
- **Diversity axes exhausted:** If fewer than N meaningful variants exist, tell the user. "There are two fundamentally different approaches to this problem" is more useful than padding to five.
- **User's problem is ambiguous:** Generate diverse interpretations of the problem itself (variant A assumes X, variant B assumes Y), then solve each interpretation with one approach. Diversity in problem framing is valid diversity.
- **Performance-sensitive context:** If the user will benchmark variants, ensure all share the same test harness and input data. Annotate expected complexity (time/space) for each variant.

## Limitations

- This approach works best when multiple valid solutions genuinely exist. For problems with a single correct answer (e.g., "what is 2+2?"), forced diversity produces wrong answers.
- The bounded deviation concept requires a quality reference. If the first solution is itself flawed, all variants inherit that flaw. Validate the reference before generating variants.
- Diversity assessment is qualitative when applied by an LLM (unlike the paper's token-probability metric). Claude cannot compute exact deviation scores -- it approximates bounded diversity through structured reasoning about axes of variation.
- The technique adds overhead proportional to the number of variants requested. For time-sensitive tasks where one good answer suffices, skip diversity generation.

## Reference

**Paper:** [DARL: Encouraging Diverse Answers for General Reasoning without Verifiers](https://arxiv.org/abs/2601.14700v1) (Huang et al., 2026). Look for: the bounded diversity reward formula (Section 3), the gamma/beta hyperparameter ablation (Section 5.3), and the WritingBench results showing where diversity matters most (Table 3).