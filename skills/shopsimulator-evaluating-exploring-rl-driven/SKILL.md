---
name: "shopsimulator-evaluating-exploring-rl-driven"
description: "Build and evaluate LLM-based shopping assistant agents using structured multi-turn dialogue, personalized product search, and strict reward-driven RL training. Use when: 'build a shopping agent', 'evaluate product recommendation agent', 'create e-commerce chatbot with RL', 'multi-turn shopping dialogue system', 'personalized product search agent', 'train an RL agent for e-commerce'."
---

# ShopSimulator: RL-Driven LLM Shopping Agent Framework

This skill enables Claude to design, implement, and evaluate LLM-based shopping assistant agents that handle multi-turn dialogues, personalized product search, and fine-grained product selection. The core technique combines structured action-space design (Search/Click/BuyNow/AskShopper), multiplicative strict reward functions that penalize bottleneck constraint failures, and a two-phase SFT+GRPO training pipeline. This approach addresses the fundamental challenge that even frontier LLMs achieve under 40% full-success rate on realistic shopping tasks due to failures in deep search, personalization balance, and purchase confirmation discipline.

## When to Use

- When the user asks to build a conversational shopping assistant or product recommendation agent
- When designing an e-commerce chatbot that must handle multi-turn clarification dialogues
- When implementing an RL training pipeline for an LLM agent operating in a tool-use environment
- When creating an evaluation harness for shopping/retrieval agents with strict multi-dimensional success criteria
- When the user needs to integrate user profile personalization into an agent's search and selection logic
- When building a simulated environment for training agents that interact with product catalogs
- When debugging why an LLM agent fails at fine-grained product attribute matching or option selection

## Key Technique

**Structured Action Space with Strict Reward.** The ShopSimulator framework defines four discrete action types for the agent: `Search[keywords]` to query a product catalog, `Click[element]` to view product details or select options, `BuyNow[]` to execute a purchase, and `AskShopper[question]` to request missing information from the user. Each agent turn produces exactly one action, formatted as a structured `Thought / Action_type / Action_content` triple. This constrained output format prevents agents from hallucinating UI elements or skipping confirmation steps.

**Multiplicative Bottleneck Reward.** The critical design insight is the strict reward function: `R_strict = r_cat * (|U_att intersect Y_att| / |U_att|) * (|U_opt intersect Y_opt| / |U_opt|) * 1[Y_price <= U_price]`. Unlike additive (loose) rewards that give partial credit, this multiplicative formulation means a zero on any single dimension (wrong category, missing attribute, wrong option, over budget) zeros the entire reward. This forces the RL optimizer to address its weakest capability rather than coasting on easy dimensions. Empirically, strict reward consistently outperforms loose reward across all scenario types.

**Two-Phase SFT + GRPO Training.** Phase one collects successful trajectories from a strong teacher model (e.g., GPT-4-class) and fine-tunes a smaller model (e.g., Qwen3-8B) via supervised learning to learn the workflow structure. Phase two applies Group Relative Policy Optimization (GRPO) with the strict reward signal, using 8 trajectory rollouts per batch, a 32K context window, and no KL penalty to encourage exploration. SFT provides the behavioral prior that makes RL tractable; RL then refines attribute matching and personalization handling where SFT plateaus.

## Step-by-Step Workflow

1. **Define the product catalog schema.** Structure each product as `{id, category, subcategory, title, attributes: {k: v}, options: [{size, color, ...}], price}`. Ensure each subcategory contains 100+ similar items to force fine-grained discrimination. Index the catalog for keyword search with BM25 or similar.

2. **Design user task specifications.** Each task defines a target product through constraints: required category, required attributes (e.g., "cushioning", "leather"), required options (e.g., "size 42, black"), and a price ceiling. For personalized tasks, attach a user profile with demographics, brand affinities, attribute preferences, and behavioral history.

3. **Implement the agent action space.** Constrain the agent to exactly four action types per turn:
   - `Search[keywords]` — query the catalog; keywords must include core attributes
   - `Click[element]` — select from currently visible UI elements only
   - `BuyNow[]` — finalize purchase (only after confirming all specifications)
   - `AskShopper[question]` — request clarification from the user
   Enforce that the agent outputs a structured `Thought: ... / Action_type: ... / Action_content: ...` format.

4. **Build the simulated shopper (for multi-turn).** Implement an LLM-driven user simulator that holds the ground-truth purchase intent but reveals details gradually. The simulator starts with a vague request ("I need running shoes"), responds to agent questions with progressively specific answers, and refuses to approve purchases when critical information hasn't been confirmed.

5. **Implement the observation function.** After each agent action, return the environment state: search results (product titles + snippets), product detail pages (full attributes, options, price), or shopper responses. Cap episodes at 30 turns (single-turn) or 40 turns (multi-turn).

6. **Implement the three-tier evaluation metrics.**
   - `R_loose`: additive — category match + averaged attribute/option/price satisfaction
   - `R_strict`: multiplicative — product of per-dimension match rates (zeros on any failure)
   - `R_succ`: binary — 1 only if the purchased product matches the target across ALL dimensions

7. **Collect SFT training data.** Run a strong teacher model (GPT-4-class or best available) across the training tasks. Filter for successful trajectories only. Format as `(observation, thought, action)` tuples. Target ~5-6K successful trajectories for a robust SFT dataset.

8. **Fine-tune with SFT.** Train a base model on the collected trajectories using standard causal LM loss. Use batch size 32, learning rate 1e-5, for 3-4 epochs. This teaches the agent the workflow: when to search, when to ask, when to click, when to buy.

9. **Apply GRPO reinforcement learning.** Starting from the SFT checkpoint, run GRPO with the strict reward signal. Use 8 rollouts per sample, learning rate 1e-6, 32K context limit, and omit the KL penalty to allow exploration beyond the SFT prior. Train for ~200 steps. The strict reward drives the agent to fix its weakest dimension.

10. **Run error analysis on failures.** Categorize failures by action type: Search errors (ignored attributes, repeated queries), Click errors (violated constraints, hallucinated buttons), BuyNow errors (no confirmation, purchase after rejection), AskShopper errors (failed to request missing info). For personalization failures, classify as omission (55%), over-interpretation (36%), or temporal confusion (9%).

## Concrete Examples

**Example 1: Building a Shopping Agent Action Loop**

User: "Build me a multi-turn shopping assistant agent that can search products, ask clarifying questions, and make purchases."

Approach:
1. Define the structured output format the LLM must follow
2. Implement the action parser and environment step function
3. Wire up the search backend, product detail viewer, and purchase executor

Output:
```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional
import json

class ActionType(Enum):
    SEARCH = "Search"
    CLICK = "Click"
    BUY_NOW = "BuyNow"
    ASK_SHOPPER = "AskShopper"

@dataclass
class AgentAction:
    thought: str
    action_type: ActionType
    action_content: str

AGENT_SYSTEM_PROMPT = """You are a shopping assistant. On each turn you receive
an observation (search results, product details, or shopper message).

You MUST respond in this exact format:
Thought: <your reasoning about what to do next>
Action_type: <Search|Click|BuyNow|AskShopper>
Action_content: <keywords|element_text|""|question>

Rules:
- Search keywords must include core attributes (brand, category, key specs)
- Click values must come from currently displayed elements only
- Never execute BuyNow without first confirming all specifications match
- Use AskShopper when the user's requirements are ambiguous or incomplete
- Only one action per turn"""

def parse_agent_response(response: str) -> AgentAction:
    lines = response.strip().split("\n")
    thought, action_type, action_content = "", "", ""
    for line in lines:
        if line.startswith("Thought:"):
            thought = line[len("Thought:"):].strip()
        elif line.startswith("Action_type:"):
            action_type = line[len("Action_type:"):].strip()
        elif line.startswith("Action_content:"):
            action_content = line[len("Action_content:"):].strip()
    return AgentAction(
        thought=thought,
        action_type=ActionType(action_type),
        action_content=action_content,
    )

def run_episode(agent_llm, environment, user_goal, max_turns=40):
    """Execute one shopping episode with structured action loop."""
    history = []
    obs = environment.reset(user_goal)

    for turn in range(max_turns):
        prompt = format_observation(obs, history)
        response = agent_llm.generate(AGENT_SYSTEM_PROMPT, prompt)
        action = parse_agent_response(response)
        history.append({"observation": obs, "action": action})

        if action.action_type == ActionType.BUY_NOW:
            reward = environment.evaluate_purchase()
            return history, reward

        obs = environment.step(action)

    return history, 0.0  # Failed: exceeded turn limit
```

**Example 2: Implementing Strict vs. Loose Reward Functions**

User: "I need evaluation metrics for my product recommendation agent that penalize partial matches properly."

Approach:
1. Implement both additive (loose) and multiplicative (strict) reward functions
2. Add binary success metric for exact matches
3. Show how strict reward surfaces bottleneck failures

Output:
```python
def compute_rewards(target, purchased):
    """
    target: {category, attributes: set, options: set, max_price: float}
    purchased: {category, attributes: set, options: set, price: float}
    """
    cat_match = 1.0 if target["category"] == purchased["category"] else 0.0
    att_overlap = len(target["attributes"] & purchased["attributes"])
    att_total = len(target["attributes"]) if target["attributes"] else 1
    opt_overlap = len(target["options"] & purchased["options"])
    opt_total = len(target["options"]) if target["options"] else 1
    price_ok = 1.0 if purchased["price"] <= target["max_price"] else 0.0

    att_rate = att_overlap / att_total
    opt_rate = opt_overlap / opt_total

    # Loose: additive partial credit
    r_loose = cat_match * (att_overlap + opt_overlap + price_ok) / (att_total + opt_total + 1)

    # Strict: multiplicative — any zero dimension zeros everything
    r_strict = cat_match * att_rate * opt_rate * price_ok

    # Binary success: all dimensions must be perfect
    r_succ = 1.0 if (cat_match and att_rate == 1.0
                      and opt_rate == 1.0 and price_ok) else 0.0

    return {"loose": r_loose, "strict": r_strict, "success": r_succ}

# Example: agent gets category and price right but misses one option
target = {"category": "running_shoes", "attributes": {"cushioning", "breathable"},
          "options": {"size_42", "black"}, "max_price": 500}
purchased = {"category": "running_shoes", "attributes": {"cushioning", "breathable"},
             "options": {"size_42", "white"}, "price": 450}  # wrong color

result = compute_rewards(target, purchased)
# loose: 0.8  (partial credit — looks decent)
# strict: 0.5 (multiplicative penalty — surfaces the option mismatch)
# success: 0.0 (binary fail — wrong product)
```

**Example 3: Personalization-Aware Search with Error Detection**

User: "My shopping agent keeps ignoring user preferences or over-constraining searches based on profile data. How do I fix this?"

Approach:
1. Separate hard constraints (from explicit request) from soft preferences (from profile)
2. Implement a preference integration layer with priority levels
3. Add error detection for the three personalization failure modes

Output:
```python
def build_search_query(user_request: dict, user_profile: dict) -> str:
    """
    Merge explicit request constraints with profile preferences.
    Explicit request always takes priority over profile data.
    """
    # Hard constraints: directly stated by user in this session
    hard = {
        "category": user_request.get("category"),
        "attributes": set(user_request.get("attributes", [])),
        "price_max": user_request.get("price_max"),
    }

    # Soft preferences: from user profile (lower priority)
    soft = {
        "brand_affinity": user_profile.get("brand_preferences", {}),
        "preferred_attrs": set(user_profile.get("attribute_preferences", [])),
        "price_tier": user_profile.get("spending_tier"),
    }

    # Build query: start with hard constraints
    keywords = []
    if hard["category"]:
        keywords.append(hard["category"])
    keywords.extend(hard["attributes"])

    # Add soft preferences ONLY if they don't conflict with hard constraints
    for attr in soft["preferred_attrs"]:
        if attr not in hard["attributes"]:  # Don't duplicate
            keywords.append(attr)

    # Add top brand affinity only if no brand explicitly requested
    if "brand" not in user_request:
        top_brand = max(soft["brand_affinity"],
                        key=soft["brand_affinity"].get, default=None)
        if top_brand and soft["brand_affinity"][top_brand] == "high":
            keywords.append(top_brand)

    return " ".join(keywords)

def classify_personalization_error(agent_query: str, user_request: dict,
                                    user_profile: dict) -> str:
    """Detect which personalization failure mode the agent exhibits."""
    profile_attrs = set(user_profile.get("attribute_preferences", []))
    request_attrs = set(user_request.get("attributes", []))
    query_terms = set(agent_query.lower().split())

    profile_used = profile_attrs & query_terms
    request_used = request_attrs & query_terms

    if len(profile_used) == 0 and len(profile_attrs) > 0:
        return "omission"       # 55% of errors: ignored profile entirely
    if len(profile_used) > len(request_used):
        return "over_interpret"  # 36% of errors: profile overrides request
    if any(p in query_terms for p in user_profile.get("search_history", [])):
        return "temporal_confusion"  # 9%: mixed up past and current needs
    return "balanced"  # Correct behavior
```

## Best Practices

- **Do:** Always enforce the structured `Thought / Action_type / Action_content` output format. Unstructured agent responses lead to hallucinated UI elements and skipped confirmation steps.
- **Do:** Use the strict (multiplicative) reward for RL training. It consistently outperforms additive rewards because it forces the optimizer to address its weakest constraint dimension rather than maximizing easy partial credit.
- **Do:** Separate hard constraints (from the current user request) from soft preferences (from the user profile). Hard constraints are non-negotiable; soft preferences guide ranking and tiebreaking.
- **Do:** Require explicit confirmation of all product specifications (size, color, model variant) before allowing the `BuyNow` action. Roughly 80% of purchase errors come from skipping this step.
- **Avoid:** Giving the agent access to UI elements that aren't currently displayed. The `Click` action must reference only elements present in the current observation — anything else is a hallucination.
- **Avoid:** Setting the KL penalty too high during GRPO training. Omitting or minimizing it allows the policy to explore beyond the SFT prior, which is necessary for learning personalization nuances that supervised data doesn't capture well.

## Error Handling

- **Agent exceeds turn limit:** The episode returns reward 0. If this happens frequently, the agent likely has a search loop (repeated similar queries). Add a deduplication check: if the last 3 search queries share >80% keyword overlap, force the agent to try a different strategy or ask the shopper for clarification.
- **Agent clicks a non-existent element:** Validate all `Click` actions against the current observation's clickable elements. Return an error observation: "Element not found. Available elements: [list]". Do not silently fail.
- **Agent purchases without viewing details:** Track whether the agent has visited the product detail page before allowing `BuyNow`. If not, return: "You must view product details before purchasing."
- **Personalization data missing:** If no user profile is available, the agent should operate in non-personalized mode. Never hallucinate preferences. The system prompt should explicitly state: "No user profile available — rely only on stated requirements."
- **Search returns zero results:** Return an observation stating no results were found and suggest broadening the query by removing the most specific attribute keyword.

## Limitations

- **Chinese e-commerce focus:** The original ShopSimulator dataset and catalog are Chinese Taobao products. Applying this to English or other-language e-commerce requires rebuilding the catalog and adjusting attribute taxonomies.
- **Multi-turn plateau:** Even with SFT+RL, multi-turn personalized scenarios plateau around 35% success rate. Long trajectories compound errors — each additional turn is another chance for the agent to lose track of constraints.
- **Simulated shopper fidelity:** The LLM-based user simulator may not capture real user behavior patterns (impatience, contradictory statements, ambiguous phrasing). Production deployment requires human evaluation.
- **Catalog scale dependency:** The strict reward assumes a one-to-one mapping between task specification and target product. In real catalogs with many valid matches, you'll need a relaxed version of `R_succ` that accepts any product satisfying all constraints.
- **Reward gaming:** Multiplicative strict reward can incentivize the agent to avoid purchasing entirely (getting 0 reward is the same as a wrong purchase). Use a small negative reward for timeout to break this symmetry.

## Reference

**Paper:** [ShopSimulator: Evaluating and Exploring RL-Driven LLM Agent for Shopping Assistants](https://arxiv.org/abs/2601.18225v1) — Wang et al., 2026. Focus on Section 4 (error analysis taxonomy), Section 5 (SFT+GRPO training pipeline), and the strict vs. loose reward comparison in Table 2 for the most actionable implementation guidance.