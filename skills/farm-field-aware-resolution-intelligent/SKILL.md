---
name: "farm-field-aware-resolution-intelligent"
description: "Build intelligent trigger-action automation systems using FARM's two-stage architecture: contrastive retrieval + multi-agent LLM selection with field-level configuration. Use when asked to 'create an IFTTT-style automation', 'build a trigger-action pipeline', 'connect event triggers to actions with field bindings', 'design a multi-agent workflow resolver', 'generate executable automation rules from natural language', or 'wire up webhook triggers to API actions'."
version: 1
---

# FARM: Field-Aware Resolution for Intelligent Trigger-Action Automation

This skill enables Claude to design and implement trigger-action automation systems that go beyond simple service-level matching. Using the FARM architecture from Badalov & Yoon (2026), Claude can build two-stage pipelines that (1) retrieve candidate trigger/action functions from large catalogs via contrastive embeddings, then (2) select and fully configure executable applets using a multi-agent LLM pipeline with schema-aware field bindings. The key advance over naive approaches: FARM produces complete, executable configurations -- not just "use service X with service Y," but the exact field-to-ingredient wiring that makes the automation actually run.

## When to Use

- When the user asks to build an event-driven automation platform (IFTTT/Zapier-style) that must resolve natural language descriptions into executable rules
- When designing a system that matches trigger outputs (ingredients) to action inputs (fields) across heterogeneous APIs or services
- When building a retrieval pipeline over a large catalog of API functions where both the trigger and action must be jointly correct
- When implementing a multi-agent coordination pattern where agents share state and use agreement-based selection to avoid hallucination
- When the user needs to generate complete webhook/event configurations from user intent, not just service recommendations
- When wiring up IoT, smart home, or developer-tool automations that require schema-level field binding

## Key Technique

**Stage 1 -- Contrastive Dual-Encoder Retrieval.** FARM represents each trigger and action function using schema-enriched descriptions: trigger schemas list "Provides: [ingredient names, types, examples]" and action schemas list "Requires: [field names, requirement flags]." A dual encoder (based on EmbeddingGemma, 307M params) is trained with InfoNCE contrastive loss (temperature 0.05) to embed user queries and function descriptions into a shared space. The critical implementation detail: freeze the lower 82% of transformer layers (layers 0-11 of 24) to preserve pretrained semantics while training only upper layers for trigger-action discrimination. This reduces the 2.2M-pair search space to a manageable top-k set per side.

**Stage 2 -- Multi-Agent Selection and Configuration.** Four LLM-based agents coordinate through a shared state object: (1) the **Intent Analyzer** decomposes the user query into separate trigger and action intents using chain-of-thought; (2) the **Trigger Selector** ranks all k x k candidate pairs using a weighted formula (70% schema coverage + 30% retrieval similarity) and applies agreement-based selection -- the LLM can only override the retrieval ranking if its preferred candidate has at least 95% of the top retrieval score; (3) the **Action Selector** performs cross-schema scoring measuring how many required action fields can be satisfied by available trigger ingredients, generates bindings (ingredient-sourced or static), and uses a lower agreement threshold (0.80) to allow more LLM reasoning; (4) the **Verifier** scores binding quality, completeness, and executability, triggering a quality-gated fallback (retry with RAG's top choice, then advance to next candidate pair) if the score falls below 0.5.

**Why this matters for implementation.** Most automation builders stop at "pick the right service." FARM solves the harder problem: generating the `{field: source, value}` binding tuples that make an applet actually executable. The agreement-based selection pattern -- where the LLM must earn the right to override retrieval -- is a reusable technique for any RAG system where you want LLM flexibility without hallucination risk.

## Step-by-Step Workflow

1. **Define your function catalog schema.** For each trigger function, record: function ID, service name, description, and a "Provides" list of ingredients with name, type, and example value. For each action function, record: function ID, service name, description, and a "Requires" list of fields with name, type, required flag, and example.

2. **Build schema-enriched text representations.** Concatenate each function's description with its schema into a single string. For triggers: `"{description}. Provides: {ingredient_name} ({type}): {example}, ..."`. For actions: `"{description}. Requires: {field_name} ({type}, {'required'|'optional'}): {example}, ..."`.

3. **Train or configure contrastive dual encoders.** Use a sentence-transformer model. Freeze the lower ~80% of layers. Train with InfoNCE loss over (query, positive_function) pairs with in-batch negatives. Use temperature 0.05, batch size 16, learning rate 2e-5, for 3 epochs. At inference, encode the user query and retrieve top-k triggers and top-k actions by cosine similarity.

4. **Implement the Intent Analyzer agent.** Prompt an LLM to decompose the user's natural language request into two parts: the trigger event ("what condition or event starts this?") and the action intent ("what should happen?"). Output optimized sub-queries for each.

5. **Implement the Trigger Selector agent.** For all k x k candidate pairs, compute: `score = 0.7 * coverage(trigger, action) + 0.3 * mean(sim(query, trigger), sim(query, action))`. Coverage measures what fraction of the action's required fields can be satisfied by the trigger's ingredients (using exact match, substring match, and semantic similarity). Have the LLM select its preferred trigger, but only accept the override if `score(LLM_choice) / score(top_retrieval) >= 0.95`.

6. **Implement the Action Selector agent.** Re-rank actions using the selected trigger's full ingredient schema as context. Apply cross-schema coverage scoring. Use an agreement threshold of 0.80. For each action field, generate a binding tuple: `(field_name, "ingredient" | "static", value)`. Ingredient bindings map from trigger outputs; static bindings use user-specified or default values.

7. **Implement the Verifier agent.** Score the complete configuration on three axes: binding quality (are mappings semantically appropriate?), completeness (are all required fields bound?), executability (would this run without errors?). Produce a score in [0, 1] and a natural language critique.

8. **Implement quality-gated fallback.** If verifier score < 0.5: (a) if the LLM overrode the retrieval ranking, retry with the retrieval top choice; (b) if still failing, advance to the next candidate pair from the priority queue; (c) attempt up to k-squared pairs before returning a failure with explanation.

9. **Emit the executable configuration.** Output a complete JSON specification: trigger function ID, action function ID, the binding map, and any static parameter values. Validate that all required fields are bound and types are compatible.

10. **Expose the pipeline as an API or CLI.** Accept natural language automation descriptions, return fully configured applets. Log the shared state at each agent step for debugging and audit.

## Concrete Examples

**Example 1: Smart Home Automation**

```
User: "Turn my Hue lights green when Tesla stock goes up more than 5%"

Step 1 - Intent Analysis:
  Trigger intent: "stock price rises by percentage"
  Action intent: "change smart light color"
  Optimized queries: q_T="stock price percentage increase alert"
                     q_A="change hue light color mode"

Step 2 - Retrieval (top-3 each):
  Triggers: [stock.price_rises_by_pct, stock.daily_price_change, crypto.price_alert]
  Actions:  [hue.turn_on_change_mode, hue.blink_lights, lifx.set_color]

Step 3 - Trigger Selection:
  coverage(stock.price_rises_by_pct, hue.turn_on_change_mode) = 0.5
  Ingredients available: [StockName, Price, PercentageChange, CheckTime]
  Fields required: [color (required), light (required)]
  Agreement check: LLM agrees with retrieval top pick. Selected.

Step 4 - Action Selection + Binding:
  cross_schema_score(hue.turn_on_change_mode) = 0.82
  Bindings generated:
    { "color": { "source": "static", "value": "green" },
      "light": { "source": "static", "value": "all lights" } }
  Note: No ingredient maps directly to color/light, so both are static.

Step 5 - Verification:
  Score: 0.88 (all required fields bound, semantically coherent)

Output:
{
  "trigger": { "function": "stock.price_rises_by_pct",
               "params": { "stock": "TSLA", "threshold": 5 } },
  "action":  { "function": "hue.turn_on_change_mode",
               "params": { "color": "green", "light": "all lights" } },
  "bindings": [],
  "static_values": { "color": "green", "light": "all lights" }
}
```

**Example 2: Developer Workflow with Ingredient Bindings**

```
User: "When a new GitHub issue is opened with label 'bug', send a Slack message
       to #engineering with the issue title and link"

Step 1 - Intent Analysis:
  Trigger intent: "new GitHub issue opened with specific label"
  Action intent: "send Slack channel message with issue details"

Step 2 - Retrieval:
  Triggers: [github.new_issue_labeled, github.new_issue, github.new_pr]
  Actions:  [slack.post_to_channel, slack.send_dm, teams.post_message]

Step 3 - Trigger Selection:
  github.new_issue_labeled provides: [IssueTitle, IssueBody, IssueURL,
                                       IssueNumber, Label, RepoName]
  Required by slack.post_to_channel: [channel (req), message (req)]
  Coverage: IssueTitle/IssueURL can compose "message" -> coverage = 1.0

Step 4 - Action Selection + Binding:
  Bindings:
    { "channel": { "source": "static", "value": "#engineering" },
      "message": { "source": "ingredient",
                   "value": "Bug: {{IssueTitle}} - {{IssueURL}}" } }

Step 5 - Verification:
  Score: 0.95 (ingredient binding is semantically appropriate, all fields bound)

Output:
{
  "trigger": { "function": "github.new_issue_labeled",
               "params": { "label": "bug" } },
  "action":  { "function": "slack.post_to_channel",
               "params": {} },
  "bindings": [
    { "field": "message", "source": "ingredient",
      "template": "Bug: {{IssueTitle}} - {{IssueURL}}" }
  ],
  "static_values": { "channel": "#engineering" }
}
```

**Example 3: Multi-Agent Fallback in Action**

```
User: "Email me a summary when my Fitbit shows I hit 10,000 steps"

Step 1 - Intent Analysis:
  Trigger intent: "Fitbit step count reaches goal"
  Action intent: "send email with fitness summary"

Step 2 - First Attempt:
  Selected: fitbit.daily_step_goal -> gmail.send_email
  Ingredients: [StepCount, Date, CaloriesBurned, Distance]
  Fields required: [to (req), subject (req), body (req)]
  Bindings: to=static(user@email), subject=static("Step Goal Hit!"),
            body=ingredient("{{StepCount}} steps on {{Date}}")
  Verifier score: 0.78 -> PASS

  If verifier had scored 0.4 (e.g., wrong trigger selected):
  -> Fallback 1: Retry with RAG top choice instead of LLM override
  -> Fallback 2: Advance to next pair (fitbit.daily_activity_summary, gmail.send_email)
  -> Continue until score >= 0.5 or k^2 pairs exhausted
```

## Best Practices

- **Do:** Enrich function descriptions with full schema (ingredients/fields with types and examples) before embedding. Plain descriptions lose the structural information that makes field binding possible.
- **Do:** Use the 70/30 weighting (schema coverage vs. retrieval similarity) when scoring candidate pairs. Coverage of required fields is more important than semantic similarity to the query.
- **Do:** Implement agreement thresholds (0.95 for triggers, 0.80 for actions) to let the LLM reason freely while preventing it from overriding high-confidence retrieval results with hallucinated choices.
- **Do:** Always run the verification step. Skipping it means shipping broken automations that fail at runtime due to unbound required fields or type mismatches.
- **Avoid:** Treating this as a service-level matching problem. "Use Gmail" is not a solution -- "Use gmail.send_email with body={{IssueTitle}}" is. Function-level resolution with field bindings is the whole point.
- **Avoid:** Fine-tuning all encoder layers. Freezing the lower 80% is critical to prevent catastrophic forgetting while still adapting to the trigger-action domain.

## Error Handling

- **Ambiguous intent decomposition:** If the Intent Analyzer cannot clearly separate trigger from action (e.g., "sync my calendar"), prompt the user for clarification: "What event should start this? What should happen when it fires?"
- **Low retrieval recall:** If no candidate in top-k looks relevant (all cosine similarities < 0.3), expand k or report that the requested automation may not be supported by the current function catalog.
- **All candidate pairs fail verification:** After exhausting k-squared pairs, return the highest-scoring attempt with the verifier's critique, so the user can manually adjust the configuration.
- **Type mismatches in bindings:** When an ingredient type (e.g., integer) doesn't match a field type (e.g., string), insert an explicit type coercion in the binding or flag it for user review.
- **Missing required fields with no matching ingredients:** Generate static placeholder bindings and flag them prominently in the output so the user knows manual input is needed.

## Limitations

- FARM assumes a pre-existing catalog of trigger and action function schemas. If your services don't expose structured schemas (ingredient names, types, required flags), you'll need to build that catalog first.
- The contrastive encoder training requires paired (query, correct_function) data. Cold-start scenarios with no usage history need synthetic pair generation or manual annotation.
- Agreement-based selection thresholds (0.95, 0.80) were tuned on IFTTT data. Different domains (enterprise APIs, IoT protocols) may need threshold recalibration.
- The system handles single trigger -> single action applets. Multi-trigger or branching/conditional workflows (if-then-else chains) are outside scope.
- Field binding quality depends on how descriptive ingredient and field names are. Opaque names like "field_1" or "param_a" degrade both retrieval and binding accuracy.

## Reference

**Paper:** Badalov & Yoon, "FARM: Field-Aware Resolution Model for Intelligent Trigger-Action Automation," arXiv:2601.15687v1 (2026). Look for: the schema-enriched representation format, the InfoNCE training configuration, the agreement-based override formula, and the quality-gated fallback algorithm.