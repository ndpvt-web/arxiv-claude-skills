---
name: "curiosity-driven-knowledge-retrieval"
description: |
  Implements a curiosity-driven knowledge retrieval framework for autonomous agents.
  Formalizes agent uncertainty as a curiosity score, triggers external knowledge
  retrieval when uncertainty exceeds a threshold, and organizes retrieved knowledge
  into structured AppCards for selective integration into reasoning.
  Trigger phrases: "build an agent with curiosity-driven retrieval",
  "add uncertainty-aware knowledge lookup", "implement AppCard knowledge system",
  "create a curiosity-scored agent pipeline", "build adaptive knowledge retrieval",
  "implement uncertainty-triggered documentation lookup"
---

# Curiosity-Driven Knowledge Retrieval

This skill teaches Claude to build agent systems that detect their own knowledge gaps during task execution and autonomously retrieve relevant external knowledge to fill them. The core technique, from "Curiosity Driven Knowledge Retrieval for Mobile Agents" (Li et al., 2026), formalizes uncertainty as a **curiosity score** using Jensen-Shannon divergence between predicted and observed states. When cumulative uncertainty for a domain exceeds a threshold, the system retrieves documentation, code, and historical trajectories, then organizes them into **AppCards** — structured knowledge cards encoding functional semantics, parameter conventions, interface mappings, and interaction patterns. The agent selectively injects relevant AppCards into its reasoning context, reducing ambiguity and shortening exploration.

## When to Use

- When building an autonomous agent that must operate across multiple tools, APIs, or applications it may not fully understand
- When a user asks to create a system that detects when an agent is "confused" or uncertain and fetches help
- When implementing retrieval-augmented generation (RAG) where retrieval should be triggered adaptively rather than on every query
- When designing a multi-step task executor that needs to recover from knowledge blind spots mid-execution
- When creating a documentation lookup system that activates only when the agent's predictions diverge from observed behavior
- When building a mobile/UI automation agent that must generalize to unseen applications

## Key Technique

**Curiosity Score as Retrieval Gate.** Traditional RAG retrieves on every query. This framework instead monitors the agent's epistemic uncertainty continuously. It models the agent's predicted next state as a prior distribution P and the actual observed state as a posterior Q. The divergence between them — computed as a tail-adjusted Jensen-Shannon divergence `JS*(P||Q) = JS(P||Q) + λ · ½(P(<OTHER>) + Q(<OTHER>))` — quantifies information gain per step. The tail adjustment captures uncertainty from low-probability tokens that standard top-K sampling misses. Cumulative uncertainty per domain accumulates as `U(domain) = Σ γ_t · JS*(P_t || Q_t)`, where γ_t is a decay-weighted coefficient. When `U(domain) > τ`, retrieval fires.

**AppCards as Structured Knowledge.** Retrieved content is not injected as raw text. Instead, it is consolidated into AppCards: compact, standardized knowledge cards with four fields — (1) functional semantics (what the tool/API does), (2) input/output constraints (parameter types, return values, boundary conditions), (3) interface-functionality mapping (UI elements or endpoints mapped to their triggered functions), and (4) interaction patterns (common workflows and exception handling). Each AppCard contains 5-10 entries covering major modules or workflows, not micro-actions. This structure prevents prompt pollution and allows the agent to selectively integrate only the relevant fragments.

**Selective Integration.** During execution, the agent identifies which domains are needed for the current task, retrieves their AppCards, and injects them into its system prompt. This is not a blanket context dump — only AppCards for domains with high cumulative uncertainty are included. The result: the agent compensates for knowledge gaps without overloading its context window.

## Step-by-Step Workflow

1. **Define the domain registry.** Enumerate every tool, API, or application the agent may interact with. For each, assign a domain key (e.g., `"stripe-api"`, `"gallery-app"`, `"kubernetes-cli"`). Initialize cumulative uncertainty `U[domain] = 0` for each.

2. **Implement the state prediction module.** Before the agent takes an action, have it predict the expected outcome (next state, API response shape, UI state). Capture this as a probability distribution P over possible outcomes. Use the LLM's own token probabilities or a lightweight classifier.

3. **Observe the actual outcome.** After the action executes, capture the real result as distribution Q. For discrete outcomes, build Q from the observed state. For text-based outcomes, use token-level probability distributions from a forward pass.

4. **Compute the curiosity score.** Calculate the tail-adjusted JS divergence between P and Q. Aggregate low-probability tokens into an `<OTHER>` bucket and add the tail penalty weighted by λ (start with λ=0.1). Accumulate into `U[domain]` with temporal decay (γ=0.9 per step).

5. **Check the retrieval threshold.** If `U[domain] > τ` (start with τ=1.5, tune per domain), trigger knowledge retrieval for that domain. Reset `U[domain]` after retrieval to avoid redundant fetches.

6. **Retrieve from heterogeneous sources.** Query three source types in parallel: (a) web documentation and API guides (`D_web`), (b) source code and function definitions (`D_git`), (c) historical execution trajectories from past successful runs (`D_traj`). Use the domain key and current task context as the retrieval query.

7. **Build or update the AppCard.** Consolidate retrieved content into a structured AppCard with four sections: functional semantics, input/output constraints, interface-functionality mappings, and interaction patterns. Limit to 5-10 entries per card. If an AppCard already exists for this domain, merge new information rather than replacing.

8. **Inject the AppCard into agent context.** Prepend the relevant AppCard(s) to the agent's system prompt or tool description before the next reasoning step. Include only AppCards for domains actively involved in the current task and flagged by high uncertainty.

9. **Execute with augmented knowledge.** The agent proceeds with the AppCard-enriched context. Monitor whether uncertainty drops after integration — if `U[domain]` remains high after AppCard injection, escalate to a human or flag the task as requiring manual documentation.

10. **Log and iterate.** Store the task trajectory (actions, states, curiosity scores, retrieved AppCards) as a new entry in `D_traj` for future retrieval. Over time, the trajectory store becomes the most valuable retrieval source.

## Concrete Examples

**Example 1: API automation agent hitting an unknown endpoint**

```
User: Build an agent that automates Stripe payment workflows. It should
      handle subscriptions, invoices, and refunds.

Approach:
1. Define domains: ["stripe-subscriptions", "stripe-invoices", "stripe-refunds"]
2. Agent starts a subscription creation task. It predicts the API will
   return a Subscription object with `status: "active"`.
3. Actual response includes `status: "incomplete"` with a `pending_setup_intent`.
   JS divergence spikes — curiosity score for stripe-subscriptions jumps to 1.8.
4. Threshold exceeded (τ=1.5). Retrieve from Stripe docs, stripe-python
   source code, and past successful subscription trajectories.
5. Build AppCard:

   AppCard: stripe-subscriptions
   1. Create: POST /v1/subscriptions — requires customer, price; returns
      Subscription with status in [incomplete, active, past_due, canceled]
   2. PaymentIntent: incomplete status means payment_intent needs confirmation;
      use latest_invoice.payment_intent to retrieve and confirm
   3. Lifecycle: incomplete → active (on payment) or incomplete_expired (after 23h)
   4. Trials: Set trial_period_days or trial_end; status becomes "trialing"
   5. Webhooks: Listen for customer.subscription.updated, invoice.paid

6. Agent re-plans with AppCard context, correctly handles the
   pending_setup_intent, and completes the subscription flow.
```

**Example 2: Multi-tool task with cross-domain uncertainty**

```
User: Create an agent that reads data from a Google Sheet, processes it,
      and posts a summary to Slack.

Approach:
1. Domains: ["google-sheets-api", "data-processing", "slack-api"]
2. Agent reads the sheet successfully (low uncertainty for google-sheets-api).
3. Agent attempts to post to Slack using chat.postMessage with blocks.
   It predicts a 200 OK but receives {"error": "invalid_blocks"}.
   Curiosity score for slack-api: 2.1 (exceeds τ).
4. Retrieve Slack Block Kit documentation, slack-sdk source, and past
   Slack posting trajectories.
5. Build AppCard:

   AppCard: slack-api
   1. PostMessage: chat.postMessage — channel (required), text (fallback),
      blocks (array of Block objects, max 50)
   2. BlockKit: Each block needs "type" field; section blocks require
      "text" object with "type":"mrkdwn" and "text" string
   3. RichText: For formatted messages, use rich_text block type with
      rich_text_section elements
   4. Errors: invalid_blocks means malformed JSON in blocks array;
      validate with Block Kit Builder API first
   5. RateLimit: Tier 2 = 20 requests/min; use retry-after header

6. Agent rebuilds the Slack message with valid Block Kit JSON and posts
   successfully.
```

**Example 3: Implementing the curiosity scoring module in Python**

```python
import numpy as np
from collections import Counter

def compute_curiosity_score(predicted_probs: dict, observed_probs: dict,
                            lambda_tail: float = 0.1, top_k: int = 50) -> float:
    """
    Compute tail-adjusted Jensen-Shannon divergence between predicted
    and observed outcome distributions.

    Args:
        predicted_probs: {outcome: probability} from agent's prediction
        observed_probs: {outcome: probability} from actual observation
        lambda_tail: weight for the tail penalty term
        top_k: number of top outcomes to keep; rest go to <OTHER>
    """
    def bucket_distribution(probs, k):
        sorted_items = sorted(probs.items(), key=lambda x: -x[1])[:k]
        top = dict(sorted_items)
        other_mass = max(0, 1.0 - sum(top.values()))
        top["<OTHER>"] = other_mass
        return top

    P = bucket_distribution(predicted_probs, top_k)
    Q = bucket_distribution(observed_probs, top_k)

    # Union of all keys
    all_keys = set(P.keys()) | set(Q.keys())
    p = np.array([P.get(k, 1e-10) for k in all_keys])
    q = np.array([Q.get(k, 1e-10) for k in all_keys])

    # Normalize
    p = p / p.sum()
    q = q / q.sum()

    # Jensen-Shannon divergence
    m = 0.5 * (p + q)
    js = 0.5 * np.sum(p * np.log2(p / m)) + 0.5 * np.sum(q * np.log2(q / m))

    # Tail penalty
    tail_penalty = lambda_tail * 0.5 * (P.get("<OTHER>", 0) + Q.get("<OTHER>", 0))

    return js + tail_penalty


class CuriosityTracker:
    """Track cumulative uncertainty per domain and trigger retrieval."""

    def __init__(self, threshold: float = 1.5, decay: float = 0.9):
        self.threshold = threshold
        self.decay = decay
        self.uncertainty = {}  # domain -> cumulative score

    def update(self, domain: str, predicted: dict, observed: dict) -> bool:
        """Update uncertainty for a domain. Returns True if retrieval needed."""
        score = compute_curiosity_score(predicted, observed)
        current = self.uncertainty.get(domain, 0.0)
        self.uncertainty[domain] = self.decay * current + score
        return self.uncertainty[domain] > self.threshold

    def reset(self, domain: str):
        """Reset after successful retrieval."""
        self.uncertainty[domain] = 0.0
```

## Best Practices

- **Do** start with a high threshold (τ=2.0) and lower it gradually. Too-frequent retrieval wastes tokens and latency; too-rare retrieval defeats the purpose. Tune per domain based on task complexity.
- **Do** keep AppCards concise (5-10 entries per domain). Each entry should cover a workflow or module, not individual buttons or parameters. Think "cheat sheet," not "full documentation."
- **Do** store successful trajectories after every completed task. The trajectory store (`D_traj`) becomes the highest-value retrieval source over time because it captures proven action sequences.
- **Do** merge new retrieved information into existing AppCards rather than replacing them. Knowledge accumulates; previous entries remain valid unless explicitly contradicted.
- **Avoid** retrieving for every action. The entire point of the curiosity score is to gate retrieval. If you find retrieval triggering on more than 20% of steps, raise the threshold.
- **Avoid** injecting all AppCards into context simultaneously. Only include AppCards for domains with high uncertainty that are actively relevant to the current step. Context window pollution degrades performance.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| Retrieval triggers constantly | Threshold τ too low or decay γ too high | Raise τ to 2.0+; lower γ to 0.8 |
| Agent ignores AppCard content | AppCard injected too far from relevant reasoning step | Move AppCard injection to immediately before the action requiring that domain |
| Curiosity score stays high after retrieval | Retrieved docs are irrelevant or outdated | Check retrieval query quality; add domain-specific filters; fall back to `D_traj` |
| AppCards grow too large | No pruning of stale entries | Cap at 10 entries per card; replace lowest-utility entries based on access frequency |
| Observed distribution is degenerate (single outcome) | Deterministic API responses | Use output schema matching instead of token distributions; compare structural similarity |

## Limitations

- **Requires probability access.** The curiosity score relies on comparing predicted vs. observed distributions. If the underlying LLM does not expose token probabilities (e.g., some API providers), you must approximate with heuristic confidence scores or structural diff comparisons.
- **Cold start problem.** With no trajectory history (`D_traj` empty), the system relies entirely on web documentation and code — which may be incomplete or hard to parse. Seed the trajectory store with manually curated examples for critical domains.
- **Weaker models may degrade.** The paper shows that AppCard injection can hurt performance on weaker backbones. If the base model struggles to integrate structured knowledge, the extra context adds confusion rather than clarity. Test with your specific model before deploying.
- **Not suited for single-shot tasks.** The curiosity score accumulates over multiple steps. For one-off queries with no iterative execution, standard RAG is simpler and sufficient.
- **Domain registry must be maintained.** Adding new tools or APIs requires registering them as domains and optionally seeding their AppCards. This is a manual maintenance cost.

## Reference

**Paper:** "Curiosity Driven Knowledge Retrieval for Mobile Agents" — Li, Tan, Ali, Schmidt, Ma (2026). arXiv:2601.19306v1. [https://arxiv.org/abs/2601.19306v1](https://arxiv.org/abs/2601.19306v1)

**Key insight:** Formalizing agent uncertainty as a continuous curiosity signal via tail-adjusted JS divergence, then using it to gate retrieval rather than retrieving on every step, yields a 6 percentage point average improvement and state-of-the-art 88.8% success on AndroidWorld. The structured AppCard format is what makes the retrieved knowledge actually usable — raw document injection performs significantly worse.

**Task trajectories and AppCard examples:** [https://lisalsj.github.io/Droidrun-appcard/](https://lisalsj.github.io/Droidrun-appcard/)