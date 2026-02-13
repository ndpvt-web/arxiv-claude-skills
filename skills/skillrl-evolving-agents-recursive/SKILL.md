---
name: "skillrl-evolving-agents-recursive"
description: "Build self-improving agent systems that distill raw execution traces into a hierarchical skill library (SkillBank) and recursively evolve those skills based on failure analysis. Use when asked to: 'build an agent that learns from mistakes', 'create a skill library from past runs', 'improve agent performance over iterations', 'extract reusable patterns from logs', 'reduce prompt bloat from trajectory history', 'make my agent pipeline self-improving'."
---

# SkillRL: Recursive Skill-Augmented Agent Evolution

This skill enables Claude to design and implement **self-improving agent systems** based on the SkillRL framework. Instead of stuffing raw execution histories into prompts (which bloats token counts and adds noise), you distill successful and failed trajectories into compact, structured skills organized in a two-tier SkillBank. Skills are retrieved adaptively at inference time and recursively evolved by analyzing validation failures -- creating a feedback loop where the agent's knowledge base co-evolves with its policy.

## When to Use

- When building an LLM agent pipeline that runs repeatedly and should get better over time (e.g., web scraping agents, search-augmented QA, task automation)
- When an agent's context window is being consumed by verbose execution history or memory logs
- When the user wants to extract reusable heuristics or "playbooks" from successful and failed agent runs
- When designing a multi-step agent that fails on certain task categories and needs targeted improvement
- When implementing retrieval-augmented agent prompts and wanting structured skill injection rather than raw few-shot examples
- When the user asks to "compress agent memory" or "learn from agent failures"

## Key Technique

**The core insight** is that raw trajectories are a poor form of agent memory. A 50-step execution trace contains maybe 3-4 genuinely reusable strategic decisions buried in noise. SkillRL replaces trajectory storage with a **distillation step**: a teacher model (or the same LLM) analyzes completed episodes and extracts structured skills -- each containing a title, an actionable principle, and applicability conditions. Successful trajectories yield strategic patterns; failed trajectories yield counterfactual lessons ("what went wrong, why, and what should have been done instead"). This achieves 10-20x token compression compared to storing raw traces.

**The SkillBank** organizes these distilled skills into two tiers: **General Skills** (universal strategies like "explore systematically before revisiting locations" or "validate intermediate results before proceeding") that apply across all tasks, and **Task-Specific Skills** (domain heuristics like "for price-comparison queries, extract all prices before ranking" or "when login fails, check for CAPTCHA before retrying credentials") scoped to task categories. At inference, general skills are always injected into the prompt while task-specific skills are retrieved via semantic similarity (top-K with a similarity threshold).

**Recursive evolution** is the mechanism that makes this a living system rather than a static knowledge base. After each evaluation cycle, the framework identifies underperforming task categories (accuracy below a threshold), samples failures from those categories with diversity-aware stratification, and feeds them to the teacher model to generate new skills or refine existing ones. This means the skill library grows precisely where the agent is weakest, with a cap of ~3 new skills per evolution cycle to prevent unbounded growth.

## Step-by-Step Workflow

1. **Define the skill schema.** Create a JSON structure for skills with four fields: `skill_id` (string, e.g., `"gen_001"` for general, `"search_003"` for task-specific), `title` (3-5 word name), `principle` (1-2 sentence actionable rule), and `when_to_apply` (conditions under which this skill is relevant). Store the SkillBank as a JSON file or database.

2. **Collect execution trajectories.** Run the agent on a batch of tasks, logging each step: the observation, the agent's reasoning, the action taken, and the outcome. Tag each trajectory with its task category and final result (success/failure with reward signal).

3. **Distill successful trajectories into skills.** For each successful episode, prompt the teacher model: "Analyze this successful trajectory. Identify 1-3 critical strategic decisions that led to success. For each, output a structured skill with title, principle, and when_to_apply. Focus on patterns that would generalize to similar tasks, not task-specific details." Parse the JSON output and add to SkillBank.

4. **Distill failed trajectories into counterfactual lessons.** For each failed episode, prompt: "Analyze this failed trajectory. Identify the failure point, the flawed reasoning, what should have been done instead, and a preventive principle. Output 1-2 structured skills encoding the corrective lesson." This converts verbose failures into compact, actionable knowledge.

5. **Classify skills into general vs. task-specific tiers.** Review distilled skills: those applicable across task categories go into the general tier (aim for 8-12 total general skills); those specific to a domain or task type go into the task-specific tier, keyed by category. Deduplicate by merging skills with overlapping principles.

6. **Implement adaptive retrieval.** At inference, always inject all general skills into the agent prompt. For task-specific skills, embed the current task description and compute cosine similarity against skill `when_to_apply` embeddings. Retrieve the top-K (K=6) skills exceeding a similarity threshold (delta=0.4). Append retrieved skills to the system prompt before the agent acts.

7. **Run the agent with skill-augmented prompts.** Structure the agent prompt as: system instructions + general skills + retrieved task-specific skills + task description + action history + available actions. Enforce chain-of-thought reasoning so the agent explicitly references applicable skills in its reasoning.

8. **Evaluate and identify weak categories.** After a validation run, compute per-category accuracy. Flag any category where accuracy falls below your threshold (e.g., 60%). These are targets for skill evolution.

9. **Evolve the SkillBank via failure analysis.** For each underperforming category, sample up to 10 failed trajectories (prioritized by severity, selected round-robin across subcategories for diversity). Prompt the teacher: "Given these failures and the current skill library, identify gaps not addressed by existing skills. Propose up to 3 new skills or refinements to existing skills." Merge results into SkillBank.

10. **Iterate.** Repeat steps 7-9 across evaluation cycles. The SkillBank grows where the agent is weakest, while stable categories remain untouched. Cap total skills per category to prevent prompt bloat (recommended: 15-20 task-specific skills per category).

## Concrete Examples

**Example 1: Search-augmented QA agent that keeps failing on multi-hop questions**

User: "My search agent answers single-hop factual questions well but fails on multi-hop reasoning. Help me build a skill library to improve it."

Approach:
1. Collect 50 failed multi-hop trajectories from logs, noting where the agent stopped searching or combined information incorrectly.
2. Distill failures into skills:

```json
{
  "skill_id": "multihop_001",
  "title": "Decompose Before Searching",
  "principle": "When a question contains multiple entities or implicit comparisons, decompose it into sub-questions and search for each independently before synthesizing.",
  "when_to_apply": "The query mentions two or more entities, asks for a comparison, or requires information that cannot be found in a single search result."
}
```

```json
{
  "skill_id": "multihop_002",
  "title": "Verify Intermediate Answers",
  "principle": "After obtaining an intermediate answer from search, verify it against at least one additional source before using it as input for the next reasoning step.",
  "when_to_apply": "An intermediate answer will be used as a premise for further reasoning, especially when the source is a single search snippet."
}
```

3. Add these to the task-specific tier under category `"multi_hop_qa"`.
4. On next evaluation, inject these skills into prompts for multi-hop questions via retrieval.

Output: Agent accuracy on multi-hop questions improves because it now decomposes queries and cross-validates intermediate results, rather than attempting single-pass answers.

---

**Example 2: Web automation agent with a growing execution log**

User: "My web scraping agent stores full browser interaction logs as memory. It's hitting context limits. Help me compress this into a skill library."

Approach:
1. Parse the raw logs (e.g., 200 episodes, ~2000 tokens each = 400K tokens total).
2. Group by outcome: 150 successes, 50 failures. Group by task type (login flows, data extraction, form submission).
3. Distill general skills from successes:

```json
{
  "skill_id": "gen_001",
  "title": "Wait for Dynamic Content",
  "principle": "After triggering any navigation or AJAX call, wait for the target element to appear in the DOM before interacting, rather than using fixed delays.",
  "when_to_apply": "Any page interaction that triggers asynchronous content loading."
}
```

4. Distill failure lessons:

```json
{
  "skill_id": "login_001",
  "title": "Handle Session Expiry Gracefully",
  "principle": "If a previously authenticated action returns a 401 or redirects to login, re-authenticate before retrying the action rather than reporting failure.",
  "when_to_apply": "An authenticated request unexpectedly fails with auth-related errors mid-workflow."
}
```

5. Replace the 400K-token log archive with a SkillBank of ~30 skills (~3K tokens total). That is a 130x compression.

Output: The agent now operates with a compact skill-augmented prompt instead of verbose history, staying well within context limits while retaining the strategic knowledge from 200 episodes.

---

**Example 3: Implementing the recursive evolution loop in code**

User: "Show me how to implement the skill evolution step."

Approach:
1. After each evaluation batch, compute per-category accuracy.
2. For underperforming categories, run the evolution prompt:

```python
def evolve_skillbank(skillbank, failed_trajectories, category, llm):
    # Sample failures: up to 10, prioritized by severity
    sorted_failures = sorted(failed_trajectories, key=lambda t: t["reward"])
    sampled = sorted_failures[:10]

    current_skills = [s for s in skillbank if s["category"] == category]

    prompt = f"""Analyze these {len(sampled)} failed trajectories for category '{category}'.
Current skills for this category:
{json.dumps(current_skills, indent=2)}

For each failure, identify:
1. The failure pattern
2. Whether any existing skill should have prevented it (and why it didn't)
3. Whether a NEW skill is needed

Output up to 3 new skills as JSON objects with fields:
skill_id, title, principle, when_to_apply

Only propose skills that address gaps NOT covered by existing skills."""

    new_skills = llm.generate(prompt, format="json")
    skillbank.extend(new_skills)
    return deduplicate(skillbank)
```

3. Run this after each evaluation epoch. The SkillBank grows by at most 3 skills per category per cycle.

Output: A living skill library that automatically patches its own blind spots after every evaluation round.

## Best Practices

- **Do:** Keep skill principles to 1-2 sentences. Verbose skills defeat the purpose of compression. If a principle needs a paragraph, split it into multiple skills.
- **Do:** Use the two-tier structure strictly. General skills should be environment-level truths (5-12 total); task-specific skills should encode category-level heuristics. Misclassifying task-specific knowledge as general dilutes prompt quality.
- **Do:** Cap skill growth. Set a maximum per category (15-20) and prune low-utility skills (those never retrieved or never referenced in agent reasoning) during evolution cycles.
- **Do:** Include `when_to_apply` conditions on every skill. Skills without applicability conditions get retrieved indiscriminately, adding noise.
- **Avoid:** Storing raw trajectories alongside the SkillBank. The whole point is compression -- if you keep both, you get the worst of both worlds (bloated context with no structure).
- **Avoid:** Evolving skills every single run. Batch evaluations and only trigger evolution when a category's accuracy drops below threshold. Continuous evolution produces redundant, contradictory skills.

## Error Handling

- **Skill conflicts:** Two skills may give contradictory advice (e.g., "always retry" vs. "fail fast on auth errors"). During evolution, explicitly prompt the teacher to check for conflicts with existing skills. Resolve by adding scope constraints to `when_to_apply`.
- **Retrieval misses:** If the similarity threshold is too high, relevant skills won't be retrieved. Monitor retrieval hit rates; lower delta from 0.4 to 0.3 if agents consistently ignore available skills. If too low, agents get irrelevant skills -- raise delta or reduce K.
- **Skill drift:** After many evolution cycles, early skills may become stale or redundant. Implement periodic pruning: remove skills not retrieved in the last N evaluation cycles, or skills whose `when_to_apply` conditions are fully subsumed by newer skills.
- **Cold-start problem:** A new task category has no skills. Bootstrap by running 10-20 episodes without skills, then distill immediately. Alternatively, transfer general skills from similar categories as a starting point.

## Limitations

- Requires a corpus of execution trajectories to bootstrap -- not useful for one-shot tasks with no iteration.
- The distillation step depends on the teacher model's ability to identify strategic patterns. If the teacher model is weak at meta-reasoning, skill quality degrades.
- Semantic retrieval of task-specific skills assumes task descriptions are sufficiently informative for embedding similarity. Tasks with vague or ambiguous descriptions may get poor skill matches.
- The framework assumes task categories are known or discoverable. Fully open-ended tasks without natural categorization make the two-tier structure less effective.
- Recursive evolution can overfit to validation failures if the failure sample is not diverse. The diversity-aware stratified sampling is critical -- skipping it leads to skills that address symptoms rather than root causes.

## Reference

[SkillRL: Evolving Agents via Recursive Skill-Augmented Reinforcement Learning](https://arxiv.org/abs/2602.08234v1) -- Focus on Section 3 (SkillBank construction and adaptive retrieval), Section 4 (recursive evolution mechanism), and Appendix A (prompt templates for distillation) and Appendix C (concrete skill examples).