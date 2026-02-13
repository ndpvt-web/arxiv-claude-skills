---
name: "chain-simulation-dual-mode-reasoning"
description: "Dual-mode reasoning framework that dynamically routes problems to specialized strategies: computational flow for math, symbolic JSON state tracking for spatial/entity reasoning, and hybrid fact-extraction for multi-hop inference. Use when asked to 'solve this step by step', 'reason through this problem', 'track state changes', 'figure out the answer to this logic puzzle', 'solve this math word problem', or 'chain these facts together'."
---

# Chain of Simulation (CoS): Dual-Mode Reasoning Framework

This skill enables Claude to apply the Chain of Simulation framework -- a dynamic problem-routing system that classifies reasoning tasks and dispatches them to one of three specialized modes: **computational flow** (math with self-consistency), **symbolic state tracking** (spatial/entity reasoning via JSON), or **hybrid fact-extraction** (multi-hop logical inference). Instead of applying a single generic chain-of-thought strategy to every problem, CoS matches the reasoning strategy to the problem type, achieving higher accuracy at lower computational cost.

## When to Use

- When the user poses a **math word problem** requiring multi-step arithmetic, algebra, or cost/quantity calculations (e.g., GSM8K-style problems)
- When the user describes a **spatial or entity-tracking scenario** where objects move between locations, people change positions, or state evolves over a sequence of events
- When the user asks a **multi-hop factual question** that requires extracting facts from multiple sources, chaining them logically, and arriving at a yes/no or short-answer conclusion
- When the user asks Claude to **debug a stateful system** by tracing object/variable states through a sequence of operations
- When a problem combines multiple reasoning types (e.g., math + spatial context) and needs hybrid treatment
- When the user wants **structured, verifiable reasoning** rather than a single-pass chain-of-thought answer

## Key Technique

CoS works by first classifying the input problem along four dimensions -- mathematical content, spatial content, multi-hop logical structure, and entity-tracking density -- then routing to the mode best suited for that problem type. This matters because applying the wrong mode catastrophically fails: computational mode scores 81.2% on math problems but 0% on spatial tasks. The routing is deterministic and keyword-driven, not probabilistic.

The three modes each impose a different reasoning structure. **Computational flow** generates step-by-step arithmetic, extracts a final numeric answer, and optionally samples multiple reasoning paths (self-consistency with k=5) to pick the median/majority answer. **Symbolic state tracking** initializes a JSON object representing world state (`{"locations": {}, "objects": {}}`) and iteratively updates it event-by-event, producing a machine-parseable final state from which the answer is read. **Hybrid fact-extraction** decomposes the problem into fact extraction, relationship identification, logical chaining, and conclusion -- ideal for yes/no questions requiring external knowledge synthesis.

The efficiency gain comes from targeted application: instead of running expensive self-consistency (k=5+ samples) on every problem, CoS only applies multi-sample generation to math problems where it helps, and uses deterministic single-pass reasoning (temperature=0) for state tracking. This achieves comparable accuracy to blanket self-consistency at 54% lower cost.

## Step-by-Step Workflow

1. **Classify the problem** by scanning for four indicator types:
   - *Mathematical*: presence of numbers, arithmetic operators, or keywords like "calculate", "total", "cost", "how many", "price", "sum"
   - *Spatial*: keywords like "where", "moved", "location", "travelled", "left", "right", "went to"
   - *Multi-hop*: logical connectors ("if", "therefore", "because", "since") appearing 2+ times, or questions requiring chained inference
   - *Entity-tracking*: 3+ named entities combined with movement or state-change verbs

2. **Route to the appropriate mode** using this priority logic:
   - Mathematical + NOT spatial --> **Computational mode**
   - Mathematical + spatial --> **Hybrid mode**
   - Spatial OR entity-tracking --> **Symbolic mode**
   - Multi-hop logical reasoning --> **Hybrid mode**
   - Ambiguous or default --> **Symbolic mode**

3. **For Computational mode**: Structure the solution as explicit step-by-step calculations. Show every intermediate value. Extract the final numeric answer on a clearly marked line. If confidence is low, generate the solution via multiple reasoning paths and take the median of numeric answers or majority vote of categorical answers.

4. **For Symbolic mode**: Initialize a JSON state object representing all known entities, locations, and attributes. Process each event/sentence sequentially, updating the JSON state after each step. Show the state transitions explicitly. Extract the answer by querying the final JSON state.

5. **For Hybrid mode**: First extract all relevant facts as a numbered list. Then identify relationships between facts. Then chain the facts through logical steps. Finally, state the conclusion with a clear final answer.

6. **Extract the answer** in a normalized format: strip whitespace, lowercase text answers, apply domain-specific aliases (e.g., "bath" = "bathroom"), and for numeric answers compare with tolerance (epsilon = 1e-9).

7. **Validate the answer** by checking internal consistency: does the final answer match intermediate results? For computational mode, verify the last calculation produces the claimed number. For symbolic mode, confirm the final JSON state is well-formed and the queried field exists. For hybrid mode, confirm the conclusion follows from the extracted facts.

8. **Present the result** with the reasoning trace visible, the mode used identified, and the final answer clearly marked.

## Concrete Examples

**Example 1: Math Word Problem (Computational Mode)**

User: "A store sells notebooks for $4 each and pens for $1.50 each. Sarah buys 3 notebooks and 7 pens. She pays with a $50 bill. How much change does she receive?"

Classification: Contains numbers, arithmetic keywords ("buys", "pays", "how much"), no spatial indicators.
Route: **Computational mode**

Approach:
1. Identify quantities: notebooks = $4 each, pens = $1.50 each, 3 notebooks, 7 pens, payment = $50
2. Calculate notebook cost: 3 x $4 = $12
3. Calculate pen cost: 7 x $1.50 = $10.50
4. Calculate total cost: $12 + $10.50 = $22.50
5. Calculate change: $50 - $22.50 = $27.50
6. Verify: $12 + $10.50 = $22.50; $50 - $22.50 = $27.50

FINAL_ANSWER: $27.50

---

**Example 2: Entity/Spatial Tracking (Symbolic Mode)**

User: "John put the apple in the kitchen. Mary moved the apple to the garden. John went to the bedroom. Mary moved the apple to the bedroom. Where is the apple?"

Classification: Named entities (John, Mary), spatial keywords ("put", "moved", "went to", "kitchen", "garden", "bedroom"), object tracking required.
Route: **Symbolic mode**

Approach:
1. Initialize state:
```json
{"locations": {"John": "unknown", "Mary": "unknown"}, "objects": {"apple": "unknown"}}
```
2. Event: "John put the apple in the kitchen"
```json
{"locations": {"John": "kitchen", "Mary": "unknown"}, "objects": {"apple": "kitchen"}}
```
3. Event: "Mary moved the apple to the garden"
```json
{"locations": {"John": "kitchen", "Mary": "garden"}, "objects": {"apple": "garden"}}
```
4. Event: "John went to the bedroom"
```json
{"locations": {"John": "bedroom", "Mary": "garden"}, "objects": {"apple": "garden"}}
```
5. Event: "Mary moved the apple to the bedroom"
```json
{"locations": {"John": "bedroom", "Mary": "bedroom"}, "objects": {"apple": "bedroom"}}
```
6. Query final state for apple's location.

FINAL_ANSWER: bedroom

---

**Example 3: Multi-Hop Inference (Hybrid Mode)**

User: "Was the first president of the United States born in the same century as the invention of the steam engine?"

Classification: No arithmetic, no spatial movement, requires chaining multiple facts ("first president" -> birth year, "steam engine" -> invention date, then comparison). Multi-hop indicators present.
Route: **Hybrid mode**

Approach:
1. Extract facts:
   - Fact 1: The first president of the United States was George Washington.
   - Fact 2: George Washington was born on February 22, 1732.
   - Fact 3: The first practical steam engine was developed by Thomas Newcomen around 1712.
   - Fact 4: The 18th century spans 1701-1800.
2. Identify relationships:
   - Washington's birth (1732) falls in the 18th century.
   - Newcomen's steam engine (1712) falls in the 18th century.
3. Apply reasoning:
   - Both events occurred in the 18th century, so they occurred in the same century.
4. Conclude:

FINAL_ANSWER: Yes

---

**Example 4: Hybrid Problem (Math + Spatial)**

User: "A delivery driver starts at the warehouse, drives 15 km to Store A, then 8 km to Store B, then 12 km back to the warehouse. Gas costs $1.20 per km. How much did the driver spend on gas?"

Classification: Mathematical (numbers, "costs", "how much") AND spatial ("drives", "to Store A", "back to warehouse").
Route: **Hybrid mode** (math + spatial)

Approach:
1. Extract facts: warehouse -> Store A = 15 km, Store A -> Store B = 8 km, Store B -> warehouse = 12 km, gas rate = $1.20/km
2. Track spatial route to compute total distance: 15 + 8 + 12 = 35 km
3. Apply arithmetic: 35 km x $1.20/km = $42.00
4. Verify: route makes a complete loop (warehouse -> A -> B -> warehouse), 35 km total, $42.00

FINAL_ANSWER: $42.00

## Best Practices

- **Do:** Always classify before solving. Spend a moment identifying which mode fits before diving into the solution. Misrouting is catastrophic.
- **Do:** Show the reasoning trace for the selected mode explicitly. For computational mode, show every arithmetic step. For symbolic mode, show every JSON state transition. For hybrid mode, show the numbered fact list.
- **Do:** Use JSON state tracking (not prose descriptions) for problems involving 3+ entities or 3+ state changes. JSON prevents the "lost in the middle" problem where intermediate states get confused.
- **Do:** For math problems where you're uncertain, solve the problem via two independent reasoning paths and compare answers before committing.
- **Avoid:** Applying computational mode to spatial/entity-tracking problems. The paper shows this produces 0% accuracy -- arithmetic reasoning cannot substitute for state tracking.
- **Avoid:** Over-complicating the mode selection. The routing is intentionally simple: check for math indicators first, then spatial, then multi-hop. Don't deliberate -- classify quickly and commit.
- **Avoid:** Generating verbose, unstructured reasoning when structured modes are available. The power of CoS is in the imposed structure, not in more words.

## Error Handling

- **Ambiguous classification**: When a problem has indicators for multiple modes, prefer hybrid mode, which can handle mixed problem types. If still unclear, default to symbolic mode (the paper's fallback).
- **Malformed JSON in symbolic mode**: If a state update produces invalid JSON, attempt repair by: (1) fixing common syntax errors like trailing commas or missing brackets, (2) re-generating the state update from the last valid state. Do not abandon JSON tracking and fall back to prose.
- **Contradictory facts in hybrid mode**: When extracted facts conflict, flag the contradiction explicitly. Present both interpretations and note the ambiguity rather than silently choosing one.
- **Arithmetic verification failure**: If the final answer doesn't match a re-computation, redo the calculation from scratch rather than patching. Accumulated rounding or carry errors are easier to fix by restarting than debugging.

## Limitations

- **Mode selection is keyword-heuristic**: The routing relies on surface-level keyword detection, not deep semantic understanding. Problems with unusual phrasing may be misrouted (e.g., a math problem that uses no standard math keywords).
- **Symbolic mode scales poorly with state size**: For problems with 20+ entities or deeply nested state, the JSON representation becomes unwieldy and update accuracy degrades.
- **Hybrid mode assumes factual knowledge**: Multi-hop reasoning depends on correctly recalled facts. If the underlying knowledge is wrong, the chaining amplifies the error.
- **Not designed for creative, open-ended, or subjective tasks**: CoS is a reasoning framework for problems with verifiable correct answers. It does not help with writing, brainstorming, or opinion-based tasks.
- **Self-consistency sampling is only beneficial for computational mode**: Applying multi-sample generation to symbolic or hybrid modes adds cost without improving accuracy, since those modes use deterministic (temperature=0) generation.

## Reference

**Paper**: [Chain of Simulation: A Dual-Mode Reasoning Framework for Large Language Models with Dynamic Problem Routing](https://arxiv.org/abs/2602.02842v1) by Saeid Sheikhi (2026). Look for Algorithms 1-4 which detail the classification vector computation, mode selection priority logic, computational flow with self-consistency, and JSON state tracking with repair mechanisms.