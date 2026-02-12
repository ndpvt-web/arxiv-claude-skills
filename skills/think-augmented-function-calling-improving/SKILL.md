---
name: "think-augmented-function-calling-improving"
description: |
  Improve LLM function/tool calling accuracy by injecting explicit "think" reasoning parameters into function schemas before the model generates arguments. Based on the TAFC framework (arXiv:2601.18282). Use this skill when:
  - "Add reasoning to my tool definitions"
  - "My agent is passing wrong parameters to functions"
  - "Improve function calling accuracy for complex APIs"
  - "Debug why my LLM agent picks incorrect argument values"
  - "Make my tool calls more reliable with chain-of-thought"
  - "Augment my OpenAI function schemas with reasoning"
---

# Think-Augmented Function Calling (TAFC)

This skill enables Claude to apply the Think-Augmented Function Calling framework to improve how LLMs generate function call parameters. The core technique injects an optional `think` string parameter into every tool/function schema so the model must articulate its reasoning *before* committing to argument values. For complex parameters with interdependencies, TAFC triggers per-parameter reasoning based on a complexity score. This requires zero architectural changes to LLMs and preserves full OpenAI-compatible API compatibility -- you only modify the JSON schemas you pass to the model.

## When to Use

- When building an AI agent whose tool calls frequently have incorrect or hallucinated parameter values
- When your function has interdependent parameters (e.g., `start_date` must precede `end_date`, `currency` must match `country`)
- When debugging why an LLM agent chose a specific argument value and you need interpretable traces
- When wrapping third-party APIs with complex validation rules and the LLM struggles with constraint satisfaction
- When designing a function-calling pipeline and you want to maximize first-call accuracy without fine-tuning
- When you have multi-parameter functions where the model tends to ignore descriptions or constraints

## Key Technique

**Function-level think augmentation.** TAFC adds a single optional `think` parameter to every function schema. This parameter has no effect on execution -- the runtime strips it before calling the real function. But it forces the model into a causal ordering: generate reasoning first, then generate the actual arguments conditioned on that reasoning. The probabilistic relationship is `P(params, think | context) = P(think | context) * P(params | context, think)`. By making the model write out its logic, parameter accuracy improves because the model resolves ambiguities and dependencies in the reasoning step rather than guessing during argument generation.

**Parameter-level granular reasoning.** Not every parameter needs deep reasoning. TAFC scores each parameter's complexity using three normalized factors: (1) `dep` -- how many other parameters it depends on or affects, (2) `type` -- how complex the data type is (a nested object scores higher than a string), and (3) `constraint` -- how strict the validation rules are (enums, regex patterns, ranges). The composite score `psi = sigmoid(a1*dep + a2*type + a3*constraint)` determines whether a parameter gets its own `think_<param>` reasoning field. Parameters exceeding a threshold (typically 0.6) get individual reasoning slots. This avoids wasting tokens on trivial parameters while ensuring critical ones receive dedicated attention.

**Dynamic description optimization.** TAFC iteratively refines the `think` parameter's description using execution feedback. After observing which calls succeed and fail, a meta-pass rewrites the think parameter's description to better guide reasoning. This is a discrete optimization loop: score current descriptions by downstream accuracy, then prompt an LLM to improve descriptions that correlate with failures. In practice, 3-5 iterations are enough to converge.

## Step-by-Step Workflow

1. **Inventory your function schemas.** Collect all tool/function definitions your agent uses. For each function, list every parameter with its type, description, constraints, and dependencies on other parameters.

2. **Add the function-level `think` parameter.** Insert an optional string parameter named `think` into each function's `properties` with a description that instructs the model to reason about parameter selection. Place it first in the property order so the model generates it before other arguments.

3. **Score each parameter's complexity.** For every parameter, compute three factors on a 0-1 scale:
   - `dep`: 0 if independent, 0.5 if it shares a soft relationship with another param, 1.0 if another param's valid values depend on it
   - `type`: 0 for primitives (string, int, bool), 0.5 for enums or arrays, 1.0 for nested objects or union types
   - `constraint`: 0 for unconstrained, 0.5 for simple validation (min/max, enum), 1.0 for regex patterns, cross-field rules, or format requirements
   - Composite: `psi = sigmoid(0.4*dep + 0.3*type + 0.3*constraint)`

4. **Add parameter-level `think_<name>` fields for complex params.** For any parameter where `psi > 0.6`, add a companion `think_<name>` string property. Its description should name the specific constraints and dependencies the model must reason about.

5. **Update the `required` array.** Keep all `think` and `think_<name>` fields optional. Never require them -- this preserves backward compatibility and lets simpler calls skip reasoning.

6. **Modify your function execution layer.** Before dispatching the actual function call, strip all `think*` keys from the arguments dict. The reasoning parameters are for the model only; the function never sees them.

7. **Test with representative inputs.** Run your agent on 10-20 diverse queries. Inspect the `think` values to verify the model is reasoning about the right constraints. Look for cases where reasoning is correct but the parameter is still wrong (description problem) vs. reasoning is wrong (prompt problem).

8. **Iterate on think descriptions.** For functions with low accuracy, rewrite the `think` parameter description to be more specific. Replace generic "reason about parameters" with explicit instructions like "Consider the timezone of the user before selecting the date format."

9. **Optionally automate description refinement.** Build a feedback loop: collect (input, generated_args, success/failure) tuples, prompt an LLM to revise the think description to prevent observed failure modes, and re-evaluate. Three to five rounds typically suffice.

10. **Deploy and monitor.** Log the `think` values in production alongside the function call results. These traces are invaluable for debugging agent behavior without needing to replay the full conversation context.

## Concrete Examples

**Example 1: Augmenting a hotel booking API**

User: "My agent keeps booking hotels with check-out dates before check-in dates. How do I fix this with TAFC?"

Approach:
1. Examine the current function schema
2. Add function-level and parameter-level think fields
3. Strip think params in the execution layer

Original schema:
```json
{
  "name": "book_hotel",
  "parameters": {
    "type": "object",
    "properties": {
      "hotel_id": { "type": "string", "description": "Hotel identifier" },
      "check_in": { "type": "string", "format": "date", "description": "Check-in date (YYYY-MM-DD)" },
      "check_out": { "type": "string", "format": "date", "description": "Check-out date (YYYY-MM-DD)" },
      "guests": { "type": "integer", "minimum": 1, "maximum": 10 },
      "room_type": { "type": "string", "enum": ["standard", "deluxe", "suite"] }
    },
    "required": ["hotel_id", "check_in", "check_out", "guests"]
  }
}
```

TAFC-augmented schema:
```json
{
  "name": "book_hotel",
  "parameters": {
    "type": "object",
    "properties": {
      "think": {
        "type": "string",
        "description": "Before filling parameters, reason about: (1) which dates the user mentioned and their chronological order, (2) whether check_out is after check_in, (3) how many guests were specified or implied, (4) any room preference stated."
      },
      "hotel_id": { "type": "string", "description": "Hotel identifier" },
      "check_in": { "type": "string", "format": "date", "description": "Check-in date (YYYY-MM-DD)" },
      "think_check_out": {
        "type": "string",
        "description": "Verify: this date must be strictly after check_in. If the user said 'three nights from Jan 5', compute Jan 5 + 3 = Jan 8."
      },
      "check_out": { "type": "string", "format": "date", "description": "Check-out date (YYYY-MM-DD), must be after check_in" },
      "guests": { "type": "integer", "minimum": 1, "maximum": 10 },
      "room_type": { "type": "string", "enum": ["standard", "deluxe", "suite"] }
    },
    "required": ["hotel_id", "check_in", "check_out", "guests"]
  }
}
```

Execution layer (Python):
```python
def execute_tool_call(name: str, arguments: dict) -> Any:
    # Strip all TAFC reasoning parameters before dispatch
    clean_args = {k: v for k, v in arguments.items() if not k.startswith("think")}
    return registry[name](**clean_args)
```

**Example 2: Multi-API agent with currency/locale dependencies**

User: "Build me a price lookup tool where the currency must match the market region."

Approach:
1. Define the function with interdependent region and currency params
2. Score complexity -- currency depends on region (dep=1.0), has enum constraint (constraint=0.5), is a primitive (type=0): psi = sigmoid(0.4*1.0 + 0.3*0 + 0.3*0.5) = sigmoid(0.55) ~ 0.63 > 0.6, so it gets a think field
3. Add targeted reasoning

Output schema:
```json
{
  "name": "get_product_price",
  "parameters": {
    "type": "object",
    "properties": {
      "think": {
        "type": "string",
        "description": "Reason about: which market region the user is asking about, and which currency is valid for that region (US->USD, EU->EUR, JP->JPY, UK->GBP)."
      },
      "product_id": { "type": "string" },
      "region": { "type": "string", "enum": ["US", "EU", "JP", "UK"] },
      "think_currency": {
        "type": "string",
        "description": "The currency MUST match the region: US->USD, EU->EUR, JP->JPY, UK->GBP. State which region was selected and derive the currency."
      },
      "currency": { "type": "string", "enum": ["USD", "EUR", "JPY", "GBP"] }
    },
    "required": ["product_id", "region", "currency"]
  }
}
```

**Example 3: Applying TAFC to an existing OpenAI function-calling codebase**

User: "I have 15 tool definitions in my agent. How do I apply TAFC across all of them efficiently?"

Approach:
1. Write a schema transformer function
2. Apply complexity scoring programmatically
3. Inject think parameters automatically

```python
import math

def compute_complexity(param_name: str, param_schema: dict, all_params: dict) -> float:
    """Score a parameter's complexity on [0, 1]."""
    # Dependency: check if description references other param names
    dep = 0.0
    desc = param_schema.get("description", "").lower()
    for other in all_params:
        if other != param_name and other.lower() in desc:
            dep = 1.0
            break

    # Type complexity
    ptype = param_schema.get("type", "string")
    if ptype in ("object", "array") or "anyOf" in param_schema or "oneOf" in param_schema:
        type_score = 1.0
    elif "enum" in param_schema:
        type_score = 0.5
    else:
        type_score = 0.0

    # Constraint complexity
    constraint = 0.0
    if "pattern" in param_schema or "format" in param_schema:
        constraint = 1.0
    elif any(k in param_schema for k in ("minimum", "maximum", "minLength", "maxLength", "enum")):
        constraint = 0.5

    raw = 0.4 * dep + 0.3 * type_score + 0.3 * constraint
    return 1 / (1 + math.exp(-5 * (raw - 0.3)))  # sigmoid centered at 0.3, steepness 5


def augment_schema_with_tafc(tool_def: dict, threshold: float = 0.6) -> dict:
    """Add TAFC think parameters to a function schema."""
    import copy
    augmented = copy.deepcopy(tool_def)
    params = augmented["parameters"]["properties"]

    # Add function-level think
    param_names = list(params.keys())
    think_desc = (
        f"Before selecting parameter values, reason step-by-step about what the user "
        f"wants and how the parameters ({', '.join(param_names)}) relate to each other."
    )
    # Insert think first via ordered rebuild
    new_props = {"think": {"type": "string", "description": think_desc}}

    for name, schema in params.items():
        score = compute_complexity(name, schema, params)
        if score > threshold:
            new_props[f"think_{name}"] = {
                "type": "string",
                "description": f"Reason about constraints and dependencies for '{name}' before assigning its value."
            }
        new_props[name] = schema

    augmented["parameters"]["properties"] = new_props
    return augmented
```

Usage:
```python
for tool in my_tools:
    tool["function"] = augment_schema_with_tafc(tool["function"])
```

## Best Practices

- **Do:** Place `think` as the first property in the schema so the model generates reasoning before any parameter values. JSON property order influences generation order in most LLM APIs.
- **Do:** Write think descriptions that name the specific constraints and relationships to reason about, not generic instructions. "Verify the end_date is after start_date" beats "Think carefully."
- **Do:** Log think values in production -- they are free debugging traces that explain why your agent chose specific parameter values.
- **Do:** Strip think parameters server-side before execution so downstream functions never see them and existing code remains unchanged.
- **Avoid:** Requiring the think parameter. Keep it optional so the model can skip it for trivial calls, reducing latency and token cost.
- **Avoid:** Adding parameter-level think fields to simple params (booleans, unconstrained strings). The overhead is not worth it when `psi < 0.6`.
- **Avoid:** Overly long think descriptions. The model reads them on every call. Keep each under 50 words.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Model ignores `think` and still gets params wrong | Think description is too vague | Rewrite with explicit constraint names and cross-references |
| Model fills `think` but contradicts itself in actual params | Weak causal coupling | Move `think` earlier in property order; some APIs respect insertion order |
| Token usage spikes after adding TAFC | Too many parameter-level think fields | Raise the complexity threshold from 0.6 to 0.75 |
| Function execution fails with unexpected `think` key | Execution layer not stripping think params | Add the `{k:v for k,v in args.items() if not k.startswith("think")}` filter |
| Think reasoning is repetitive across calls | Static description | Use the iterative refinement loop to evolve descriptions based on failure patterns |

## Limitations

- **Token cost.** Every think parameter adds 20-100 tokens per call. For high-throughput, low-complexity tools (e.g., simple CRUD), the overhead may not justify the accuracy gain.
- **Model sensitivity.** The technique relies on the model respecting property order and descriptions. Models with weak instruction-following may ignore think fields entirely.
- **No guarantee of consistency.** The model may reason correctly in the think field but still produce an incorrect parameter value. TAFC improves odds, it does not enforce correctness.
- **Single-turn only.** TAFC augments individual function calls. It does not help with multi-turn planning where the agent must decide *which* function to call across a sequence.
- **Diminishing returns on simple functions.** Functions with 1-2 independent, unconstrained parameters see negligible improvement. The technique shines on 4+ parameter functions with interdependencies.

## Reference

[Think-Augmented Function Calling: Improving LLM Parameter Accuracy Through Embedded Reasoning](https://arxiv.org/abs/2601.18282v2) -- Wei et al., 2026. Look for: Section 3 (TAFC framework definition), Section 3.3 (complexity scoring formula), and Section 4 (ToolBench evaluation showing 2-5% pass rate improvements across model scales).