---
name: "reflect-transparent-principle-guided-reasoning"
description: >
  Apply the REFLECT constitutional alignment framework to enforce user-defined principles on LLM outputs
  through a multi-stage pipeline: constitution-conditioned generation, self-evaluation with Likert scoring,
  self-critique, and principled revision. Use this skill when asked to:
  "align output to these principles", "apply constitutional review to this response",
  "reflect on whether this follows our guidelines", "enforce these coding standards on generated code",
  "review this output against our policy", "run a principled critique and revision pass".
---

# REFLECT: Transparent Principle-Guided Reasoning for Constitutional Alignment

REFLECT is an inference-time framework that aligns LLM outputs to arbitrary sets of natural-language principles without any fine-tuning or training data. It works by (1) conditioning the initial generation on a constitution of principles, then running a post-generation loop of (2) self-evaluation where each principle is scored on a 1-5 Likert scale, (3a) self-critique that reasons over any flagged violations, and (3b) revision that produces a corrected output. This makes alignment transparent — every decision is traceable to a specific principle and score — and plug-and-play with respect to any constitution you define.

## When to Use

- When the user provides a set of coding standards, style guidelines, or security policies and asks you to generate code that conforms to all of them
- When reviewing generated text, code, or documentation against an explicit checklist of requirements or values
- When the user asks to "align", "review against principles", or "constitutionally audit" any output
- When building a prompt pipeline that needs transparent, auditable reasoning about why output does or doesn't conform to guidelines
- When generating content that must satisfy multiple potentially competing constraints (e.g., "be concise" AND "be thorough")
- When the user wants to reduce tail-risk violations — rare but serious failures to follow critical rules like security or safety policies
- When creating training data by capturing the (prompt, base response, revised response) triples for downstream fine-tuning

## Key Technique

REFLECT separates alignment into four distinct, transparent stages. First, the full constitution (a numbered list of natural-language principles) is injected into the system prompt alongside the user query, conditioning the initial generation. This alone improves conformance but is insufficient for complex or competing principles.

The critical innovation is the post-generation evaluation loop. After the base response is generated, REFLECT prompts the model to score its own output against each principle individually on a 1-5 Likert scale. If any principle scores below a configurable threshold (default: 3), the model enters a critique-and-revision phase where it performs chain-of-thought reasoning over the flagged principles, articulates specifically what went wrong, and produces a revised response. This loop can repeat, but one cycle is usually sufficient.

What distinguishes REFLECT from prior Constitutional AI work is that it evaluates all principles simultaneously at inference time, produces transparent reasoning traces for every decision, requires zero training data or parameter updates, and naturally generates (prompt, base, revision) triples that can be harvested as supervised fine-tuning data to reduce the inference overhead for long-term deployment.

## Step-by-Step Workflow

1. **Define the constitution.** Collect the user's principles into a numbered list. Each principle should be a single, testable natural-language statement (e.g., "All SQL queries must use parameterized statements to prevent injection"). Aim for 5-15 principles; more is fine but increases evaluation cost.

2. **Format the constitution prompt.** Prepend the constitution to the system context using this structure:
   ```
   You must follow these principles in your response:
   1. [Principle 1]
   2. [Principle 2]
   ...
   N. [Principle N]

   Generate your response to the user's request while adhering to all principles above.
   ```

3. **Generate the base response.** Produce the initial output with the constitution in context. This is the constitution-conditioned base response.

4. **Self-evaluate against each principle.** Score the base response on each principle using a 1-5 Likert scale. Output the evaluation in a structured format:
   ```
   Principle 1 ("[principle text]"): [score] — [one-line justification]
   Principle 2 ("[principle text]"): [score] — [one-line justification]
   ...
   ```

5. **Check the threshold.** If all scores are >= 3 (or the user's custom threshold), the base response passes — output it as final. If any score is below the threshold, proceed to critique.

6. **Critique flagged principles.** For each principle that scored below threshold, perform chain-of-thought reasoning:
   - State which principle was violated
   - Quote the specific part of the base response that violates it
   - Explain why it violates the principle
   - Propose a concrete fix

7. **Revise the response.** Produce a new response that addresses every critique point while preserving the parts that already scored well. Do not regress on passing principles.

8. **Re-evaluate (optional).** Score the revised response against all principles. If violations remain and the user configured multiple cycles, repeat steps 6-7. Otherwise, output the revised response as final.

9. **Present the result.** Show the user the final response. If they requested transparency, include the evaluation scores and any critique reasoning as a collapsible or secondary section.

10. **Capture training data (optional).** If the user is building a fine-tuning dataset, output the triple: `(original_prompt, base_response, revised_response)` in a structured format for SFT harvesting.

## Concrete Examples

**Example 1: Enforcing secure coding principles on generated Python code**

User: "Generate a Flask endpoint that accepts a username and returns their profile from the database. Follow these principles: (1) All database queries must use parameterized statements, (2) All user input must be validated and sanitized, (3) Error messages must not leak internal details, (4) Responses must include appropriate HTTP status codes."

Approach:
1. Inject the four principles as a constitution
2. Generate the base Flask endpoint
3. Self-evaluate:
   ```
   Principle 1 ("parameterized statements"): 5 — Uses db.execute with ? placeholders
   Principle 2 ("input validation"):         2 — Username is passed directly without length/format check
   Principle 3 ("no internal details"):       4 — Catches exceptions with generic message
   Principle 4 ("HTTP status codes"):         5 — Returns 200, 404, 400 appropriately
   ```
4. Principle 2 scores below threshold (3). Critique: "The username parameter is taken from the request and passed to the query without validating format, length, or allowed characters. This could allow excessively long inputs or unexpected characters."
5. Revise: Add `if not re.match(r'^[a-zA-Z0-9_]{1,30}$', username): abort(400)`
6. Re-evaluate: All principles now score >= 4

Output:
```python
@app.route("/profile/<username>")
def get_profile(username):
    if not re.match(r'^[a-zA-Z0-9_]{1,30}$', username):
        return jsonify({"error": "Invalid username format"}), 400
    try:
        row = db.execute("SELECT * FROM users WHERE username = ?", (username,)).fetchone()
        if row is None:
            return jsonify({"error": "User not found"}), 404
        return jsonify(dict(row)), 200
    except Exception:
        return jsonify({"error": "An internal error occurred"}), 500
```

**Example 2: Reviewing documentation against style guidelines**

User: "Review this API documentation against our style guide: (1) Use active voice, (2) Keep sentences under 25 words, (3) Include a code example for every endpoint, (4) Avoid jargon without definition."

Input documentation snippet:
> "The endpoint is called by the client to retrieve data. The response payload is serialized using protobuf marshalling and deserialized on the client side."

Approach:
1. Self-evaluate:
   ```
   Principle 1 ("active voice"):        2 — "is called by" and "is serialized" are passive
   Principle 2 ("under 25 words"):       4 — Both sentences are within limit
   Principle 3 ("code example"):         1 — No code example provided
   Principle 4 ("no undefined jargon"):  2 — "protobuf marshalling" and "deserialized" used without explanation
   ```
2. Critique: Passive voice in both sentences. Missing code example entirely. "Protobuf marshalling" and "deserialized" need definitions or simpler alternatives.
3. Revise:

Output:
> "Call `GET /data` to retrieve records. The server encodes the response using Protocol Buffers (protobuf), a compact binary format, and your client decodes it automatically.
>
> ```bash
> curl https://api.example.com/data -H "Accept: application/protobuf"
> ```"

**Example 3: Generating a constitution for a code review bot**

User: "Help me define principles for an automated code review agent that checks pull requests."

Approach:
1. Draft principles covering common code review concerns
2. Self-evaluate the principles themselves against meta-criteria (testable, non-overlapping, actionable)
3. Revise any that are vague or overlapping

Output:
```
Constitution for Code Review Agent:
1. Flag any function longer than 50 lines and suggest decomposition
2. Require error handling for all external API calls and I/O operations
3. Reject string concatenation in SQL queries; require parameterized statements
4. Warn when test coverage for changed lines falls below 80%
5. Flag TODO/FIXME comments that lack an associated issue tracker link
6. Require type annotations on all public function signatures (Python/TypeScript)
7. Reject secrets, API keys, or credentials committed in plaintext
8. Warn when cyclomatic complexity of any function exceeds 10
```

## Best Practices

- **Do:** Write principles that are specific and testable. "Use parameterized SQL queries" is enforceable; "write secure code" is not.
- **Do:** Include the full principle text in the evaluation output so the reasoning trace is self-contained and auditable.
- **Do:** Use the threshold mechanism to avoid unnecessary revision passes — if all scores are above threshold, emit the base response directly. This reduces latency by 3-5x on well-aligned outputs.
- **Do:** When principles conflict (e.g., "be concise" vs. "include all edge cases"), explicitly rank them or add a meta-principle about priority ordering.
- **Avoid:** Defining more than ~20 principles in a single constitution. Beyond that, evaluation quality degrades because the model struggles to hold all principles in context simultaneously.
- **Avoid:** Setting the threshold too high (e.g., 5). A threshold of 3 catches genuine violations without triggering unnecessary revisions on minor stylistic preferences. Use 4 only for safety-critical applications.
- **Avoid:** Running more than 2 revision cycles. Empirically, one cycle captures the vast majority of improvements; additional cycles show diminishing returns and risk introducing regressions.

## Error Handling

- **Self-evaluation returns all 5s despite visible violations:** The model is being uncritical. Rephrase principles to be more specific and testable, or add an explicit instruction: "Be strict in your evaluation. A score of 5 means flawless adherence with zero room for improvement."
- **Revision degrades previously passing principles:** After revision, always re-evaluate all principles, not just the flagged ones. If a regression is detected, instruct the model to preserve the specific passing elements while fixing the flagged ones.
- **Conflicting principles cause oscillation:** If revision fixes one principle but breaks another in a loop, add an explicit priority ordering to the constitution or merge the conflicting principles into a single balanced statement.
- **Token budget exceeded on large constitutions:** Reduce the number of principles by grouping related ones, or split the evaluation into batches (evaluate principles 1-10, then 11-20) and merge the flagged sets before critique.

## Limitations

- **Inference cost:** The full pipeline uses 3.7-5x more tokens than a single generation pass. For latency-sensitive applications, use the self-evaluation threshold aggressively to skip revision when the base response is already compliant.
- **Self-evaluation accuracy:** The model's ability to score its own output is imperfect. It tends to be more lenient on subtle violations and more strict on surface-level formatting. For high-stakes applications, consider an external evaluator model.
- **Novel or abstract principles:** REFLECT works best on concrete, testable principles. Abstract values like "be fair" or "respect autonomy" are harder to evaluate reliably than "do not include personally identifiable information."
- **Not a substitute for fine-tuning:** For production systems processing millions of requests, the inference overhead makes REFLECT impractical as a permanent solution. Use it to generate (prompt, base, revision) training triples, then fine-tune to internalize the principles.
- **Single-model limitation:** Both generation and evaluation use the same model, so blind spots in the model's reasoning will persist through the pipeline. Particularly nuanced ethical or cultural principles may require human review.

## Reference

Bell, H., Zhang, C., Haque, M. M., Potdar, D., Zaman, S. (2026). *Reflect: Transparent Principle-Guided Reasoning for Constitutional Alignment at Scale.* arXiv:2601.18730v1. [https://arxiv.org/abs/2601.18730v1](https://arxiv.org/abs/2601.18730v1)

Key sections to consult: Algorithm 1 (full pipeline pseudocode), Appendix G (prompt templates for each stage), Section 5 (evaluation results showing 34-50% violation reduction across models).