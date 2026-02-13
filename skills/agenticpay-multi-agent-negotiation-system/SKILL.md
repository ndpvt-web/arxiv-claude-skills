---
name: "agenticpay-multi-agent-negotiation-system"
description: "Build multi-agent LLM negotiation systems where buyer and seller agents reach deals through natural language. Use when asked to 'build a negotiation system', 'create buyer-seller agents', 'implement price bargaining with LLMs', 'simulate a marketplace with AI agents', 'design an agentic commerce pipeline', or 'build a multi-round trading framework'."
---

# AgenticPay: Multi-Agent LLM Negotiation System

This skill enables Claude to build structured multi-agent negotiation systems where LLM-powered buyer and seller agents conduct multi-round natural language bargaining under private constraints. Based on the AgenticPay framework, the core technique gives each agent a confidential reservation price (buyer's maximum willingness-to-pay, seller's minimum acceptable price), a structured action extraction protocol that parses natural language into formal offers, and a scoring function that rewards feasible deals, balanced surplus splitting, and time efficiency. This goes beyond simple chatbot dialogue -- it creates principled economic interactions with measurable outcomes.

## When to Use

- When the user asks to build an automated negotiation or bargaining system between AI agents
- When implementing a marketplace simulation where multiple buyers and sellers transact via language
- When designing a procurement or pricing agent that must negotiate on behalf of a user
- When building a benchmark to evaluate LLM strategic reasoning in adversarial or cooperative settings
- When creating a game-theoretic simulation with private information and natural language communication
- When the user needs agents that can autonomously close deals within budget constraints

## Key Technique

AgenticPay structures negotiation as a finite-horizon, turn-based protocol. Each agent holds a **private reservation price** injected into its system prompt but never revealed in dialogue: the buyer knows its `p_max` (maximum it will pay), the seller knows its `p_min` (minimum it will accept). The **bargaining zone** `Z = p_max - p_min` defines the space of mutually beneficial deals. Agents alternate natural language messages, each containing exactly one structured price proposal extracted via a parser.

The critical innovation is **structured action extraction from free-form language**. Each agent's message must embed a tagged price (e.g., `### BUYER_PRICE($X) ###` or `### SELLER_PRICE($X) ###`), and a deal finalizes when both parties propose the same price and include a `MAKE_DEAL` signal. This keeps negotiation in natural language (enabling persuasion, anchoring, and reasoning) while making outcomes machine-parseable and scorable.

Outcomes are scored with a composite utility: `GlobalScore = d * (D + Q*W + E)` where `D=30` is a deal bonus, `Q = 4 * r_b * r_s` is a symmetric quality term (buyer utility `r_b = (p_max - p)/Z`, seller utility `r_s = (p - p_min)/Z`), `W=55` weights surplus balance, `E=15` rewards efficiency, and `d = 0.99^(t-1)` discounts for rounds used. The quality term `Q` peaks at 1.0 when surplus is split equally, incentivizing fair outcomes. Failed negotiations score `-F` (penalty of 15).

## Step-by-Step Workflow

1. **Define the market configuration.** Specify how many buyers, sellers, and products are involved. Start with 1-buyer-1-seller-1-product (1B1P1S) for simplicity, then scale to multi-party markets (MBMPMS). Assign each scenario a product category and context (e.g., "used car negotiation", "SaaS license procurement").

2. **Set private constraints for each agent.** Assign each buyer a `buyer_max_price` and each seller a `seller_min_price`. Ensure `p_max > p_min` so a bargaining zone exists. Set an `initial_seller_price` (the listed/asking price) as the negotiation starting point.

3. **Write system prompts with strict formatting rules.** The buyer prompt must include: the product description, the buyer's private `p_max` (marked confidential), the required price format `### BUYER_PRICE($X) ###`, the `MAKE_DEAL` keyword for acceptance, a word limit (~150 words per turn), and instructions to never reveal the reservation price. Mirror this for the seller with `### SELLER_PRICE($X) ###` and `seller_min_price`.

4. **Implement the conversation loop.** Each round: (a) buyer generates a response conditioned on the full dialogue history plus its private state, (b) append the buyer message with `(role='buyer', content=..., round=t)` metadata, (c) seller generates a counter-response given the same history, (d) append with seller metadata. Cap at `max_rounds=20`.

5. **Build the action parser.** Extract the tagged price from each message using regex (e.g., `### BUYER_PRICE\(\$?([\d,.]+)\) ###`). Detect `MAKE_DEAL` signals. A deal is reached when both parties propose the same price in the same round and both signal `MAKE_DEAL`. Validate that the agreed price `p` satisfies `p_min <= p <= p_max`.

6. **Implement the scoring function.** Compute buyer utility `r_b = (p_max - p) / Z`, seller utility `r_s = (p - p_min) / Z`, quality `Q = 4 * r_b * r_s`, and the full `GlobalScore = 0.99^(t-1) * (30 + Q*55 + 15)`. For failed negotiations (timeout or overflow), return `-15`.

7. **Add memory management.** Each agent maintains an independent conversation history as a list of `(role, content, round)` tuples. For multi-seller scenarios, the buyer must track separate conversation threads per seller. Inject relevant history into each LLM call's message list.

8. **Handle multi-party markets.** For many-to-many settings, implement a market coordinator that routes buyer-seller pairs, manages parallel negotiation threads, and resolves allocation (a buyer can only purchase one unit; a seller can only sell to one buyer per product). Use round-robin or simultaneous matching.

9. **Add environment metadata.** Include product attributes (condition, features, market comparisons), user profile text (e.g., "prefers aggressive bargaining style"), and scenario context in the system prompt to make negotiations realistic.

10. **Evaluate and log results.** Track deal rate (% of negotiations reaching agreement), timeout rate (exceeding max rounds), overflow rate (agreed price outside bargaining zone), average rounds to completion, and mean GlobalScore. Log full conversation transcripts for analysis.

## Concrete Examples

**Example 1: Bilateral Used Car Negotiation**

User: "Build a system where a buyer agent and seller agent negotiate over a used car price."

Approach:
1. Set product context: "2019 Honda Civic, 45K miles, good condition"
2. Private constraints: `seller_min_price=12000`, `buyer_max_price=16000`, `initial_seller_price=18000`
3. Configure `max_rounds=15`

Implementation skeleton:

```python
import re
from openai import OpenAI

client = OpenAI()

BUYER_SYSTEM = """You are a buyer negotiating for a 2019 Honda Civic (45K miles, good condition).
Your MAXIMUM budget is $16,000. NEVER reveal this number.
Each turn, make exactly one price offer using: ### BUYER_PRICE($X) ###
When you accept a price, include MAKE_DEAL in your response.
Keep responses under 150 words."""

SELLER_SYSTEM = """You are a seller listing a 2019 Honda Civic (45K miles, good condition) at $18,000.
Your MINIMUM acceptable price is $12,000. NEVER reveal this number.
Each turn, make exactly one price offer using: ### SELLER_PRICE($X) ###
When you accept a price, include MAKE_DEAL in your response.
Keep responses under 150 words."""

def extract_price(text, role):
    tag = "BUYER_PRICE" if role == "buyer" else "SELLER_PRICE"
    match = re.search(rf"### {tag}\(\$?([\d,]+)\) ###", text)
    return int(match.group(1).replace(",", "")) if match else None

def check_deal(buyer_msg, seller_msg, buyer_price, seller_price):
    if buyer_price and seller_price and buyer_price == seller_price:
        if "MAKE_DEAL" in buyer_msg and "MAKE_DEAL" in seller_msg:
            return buyer_price
    return None

def run_negotiation(max_rounds=15, p_min=12000, p_max=16000):
    history = []
    for round_num in range(1, max_rounds + 1):
        # Buyer turn
        buyer_messages = [{"role": "system", "content": BUYER_SYSTEM}]
        buyer_messages += [{"role": m["role"], "content": m["content"]} for m in history]
        buyer_resp = client.chat.completions.create(
            model="gpt-4o", messages=buyer_messages, max_tokens=200
        ).choices[0].message.content
        history.append({"role": "user", "content": f"[Buyer, Round {round_num}]: {buyer_resp}"})

        # Seller turn
        seller_messages = [{"role": "system", "content": SELLER_SYSTEM}]
        seller_messages += [{"role": m["role"], "content": m["content"]} for m in history]
        seller_resp = client.chat.completions.create(
            model="gpt-4o", messages=seller_messages, max_tokens=200
        ).choices[0].message.content
        history.append({"role": "assistant", "content": f"[Seller, Round {round_num}]: {seller_resp}"})

        # Check for deal
        bp = extract_price(buyer_resp, "buyer")
        sp = extract_price(seller_resp, "seller")
        deal_price = check_deal(buyer_resp, seller_resp, bp, sp)
        if deal_price:
            Z = p_max - p_min
            r_b = (p_max - deal_price) / Z
            r_s = (deal_price - p_min) / Z
            Q = 4 * r_b * r_s
            score = (0.99 ** (round_num - 1)) * (30 + Q * 55 + 15)
            return {"deal": True, "price": deal_price, "rounds": round_num,
                    "buyer_utility": r_b, "seller_utility": r_s, "score": score}
    return {"deal": False, "rounds": max_rounds, "score": -15}
```

Output:
```
{'deal': True, 'price': 14000, 'rounds': 6, 'buyer_utility': 0.5, 'seller_utility': 0.5, 'score': 95.3}
```

**Example 2: Multi-Seller Comparison Shopping**

User: "Create a system where one buyer negotiates with three competing laptop sellers simultaneously."

Approach:
1. Define three sellers with different floor prices and product specs
2. Buyer maintains parallel negotiation threads
3. Buyer commits to one deal after exploring all options

```python
sellers = [
    {"id": "seller_A", "product": "Dell XPS 15, 16GB RAM", "min_price": 900, "list_price": 1400},
    {"id": "seller_B", "product": "MacBook Air M3, 16GB RAM", "min_price": 1000, "list_price": 1300},
    {"id": "seller_C", "product": "ThinkPad X1, 32GB RAM", "min_price": 850, "list_price": 1250},
]
buyer_config = {"max_price": 1100, "requirement": "lightweight laptop for development"}

def run_multi_seller_negotiation(buyer_config, sellers, max_rounds=10):
    threads = {s["id"]: [] for s in sellers}
    best_deal = None

    for round_num in range(1, max_rounds + 1):
        for seller in sellers:
            sid = seller["id"]
            # Buyer prompt includes awareness of other ongoing negotiations
            buyer_system = f"""You are buying a laptop (budget: ${buyer_config['max_price']} max, CONFIDENTIAL).
You are negotiating with {len(sellers)} sellers simultaneously.
Current negotiation: {seller['product']} listed at ${seller['list_price']}.
Use ### BUYER_PRICE($X) ### format. Say MAKE_DEAL to accept."""

            seller_system = f"""You sell: {seller['product']} at ${seller['list_price']}.
Minimum acceptable: ${seller['min_price']} (CONFIDENTIAL).
Use ### SELLER_PRICE($X) ### format. Say MAKE_DEAL to accept."""

            # Run one round per seller, check deals, track best offer
            # ... (same turn logic as Example 1, per thread)

    # Buyer commits to best deal across all threads
    return best_deal
```

**Example 3: Procurement Agent with User Profile**

User: "Build a negotiation agent that buys SaaS licenses on behalf of a company, using a firm but professional style."

Approach:
1. Inject a user profile into the buyer's system prompt
2. Add product-specific context (seats, contract length, support tier)
3. Track multiple negotiation metrics

```python
USER_PROFILE = "Corporate procurement officer. Prefers data-driven arguments. " \
               "References competitor pricing. Firm but professional tone."

BUYER_SYSTEM = f"""You are a procurement agent negotiating a SaaS license.
Product: ProjectFlow Pro, 50 seats, annual license.
Your maximum budget: $15,000/year. NEVER reveal this.
Negotiation style: {USER_PROFILE}
Reference points: Competitor A charges $12,000 for similar features.
Format: ### BUYER_PRICE($X) ###. Say MAKE_DEAL to accept."""

# Seller has min_price=10000, list_price=20000
# Bargaining zone: $10,000 - $15,000
```

## Best Practices

- **Do:** Always validate that `p_max > p_min` before starting a negotiation -- a zero or negative bargaining zone makes deals impossible and wastes compute.
- **Do:** Use the symmetric quality term `Q = 4 * r_b * r_s` to evaluate deal fairness. A score near 1.0 means balanced surplus; scores near 0 indicate one side captured almost all value.
- **Do:** Keep agent responses short (under 150 words per turn). Longer responses increase token cost without improving negotiation outcomes -- LLMs tend to over-explain and leak strategic information in verbose mode.
- **Do:** Log full conversation transcripts with round metadata. Post-hoc analysis of failed negotiations reveals systematic prompt weaknesses (e.g., agents anchoring too aggressively).
- **Avoid:** Letting agents access each other's system prompts or reservation prices. The entire framework depends on private information remaining private. Never share agent configs between roles.
- **Avoid:** Setting `max_rounds` too high (>20). Research shows most productive negotiations conclude within 8-12 rounds. Beyond that, agents tend to loop or deadlock, wasting tokens with no convergence.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| No price tag in agent response | LLM ignored formatting instructions | Add a retry with a reinforced prompt: "You MUST include `### BUYER_PRICE($X) ###` in your response." Cap retries at 3. |
| Agreed price outside bargaining zone | Agent hallucinated acceptance or parser error | Validate `p_min <= agreed_price <= p_max` before recording. Flag as overflow and score as failure (`-15`). |
| Both agents stall at same prices | Anchoring deadlock -- neither will concede | Inject a mediator message: "Neither party has moved in 3 rounds. Consider adjusting your offer to reach agreement." |
| Agent reveals its reservation price | Prompt leaking under adversarial pressure | Strengthen the confidentiality instruction. Add a post-generation filter that detects and redacts the private price before sending to the counterpart. |
| Token limit exceeded in long negotiations | Conversation history grows with each round | Implement a sliding window: keep the first 2 rounds and the last 5 rounds, summarize middle rounds as "Buyer offered $X, Seller countered at $Y." |

## Limitations

- **Single-issue negotiation only.** The framework prices a single deal dimension. Real procurement involves bundling, payment terms, delivery schedules, and SLAs that require multi-attribute utility functions not covered here.
- **No learning across sessions.** Each negotiation starts from scratch. Agents do not adapt strategies based on past negotiations with the same counterpart or market trends.
- **LLM strategic reasoning gaps.** Current LLMs (even frontier models) struggle with long-horizon planning beyond ~10 rounds. They tend to make premature concessions or anchor too rigidly, depending on prompt phrasing.
- **Price-only outcomes.** The `MAKE_DEAL` protocol assumes a single numeric agreement. Negotiations requiring contingent contracts ("I'll pay $X if you include Y") need a richer action space.
- **No mechanism design guarantees.** Unlike auction theory, the free-form language protocol provides no incentive-compatibility or individual-rationality guarantees -- agents can bluff, mislead, or behave irrationally.

## Reference

**Paper:** [AgenticPay: A Multi-Agent LLM Negotiation System for Buyer-Seller Transactions](https://arxiv.org/abs/2602.06008v1) (Liu, Gu, Song, 2026). Look for: the GlobalScore formula (Algorithm 1), the eight market configuration types (Table 1), system prompt templates (Tables 14-15), and the structured action extraction protocol that bridges natural language and formal negotiation outcomes.

**Code:** [github.com/SafeRL-Lab/AgenticPay](https://github.com/SafeRL-Lab/AgenticPay) -- reference implementation with Gymnasium-style environment registration, configurable agent backends (OpenAI, vLLM, SGLang), and 110+ pre-built negotiation tasks.