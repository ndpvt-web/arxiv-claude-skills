---
name: "towards-adaptive-scalable-robust"
description: "Implement RAPS (Reputation-Aware Publish-Subscribe) multi-agent coordination using intent-based pub/sub messaging, reactive subscription refinement, and Bayesian reputation scoring. Use when: 'coordinate multiple LLM agents', 'build a robust multi-agent system', 'pub/sub agent architecture', 'agents that detect bad peers', 'scalable agent orchestration', 'fault-tolerant agent swarm'."
---

# RAPS: Reputation-Aware Publish-Subscribe Agent Coordination

This skill enables Claude to design and implement multi-agent systems using the RAPS paradigm from Li et al. (2026). Instead of hardcoding agent topologies (chains, stars, trees), RAPS lets agents exchange messages based on **declared intents** (subscriptions) matched by a **semantic broker**, with each agent running a **Bayesian watchdog** to detect and isolate unreliable peers. The result is a coordination layer that scales dynamically, adapts at inference time, and degrades gracefully under adversarial conditions.

## When to Use

- When the user asks to build a multi-agent system where the number of agents or their roles may change at runtime
- When coordinating agents that must tolerate unreliable or adversarial participants (e.g., mixed-model pipelines, untrusted tool outputs)
- When replacing a rigid agent topology (chain-of-agents, star hub) with flexible intent-based routing
- When the user wants agents to self-organize around a task without a centralized planner
- When building a code review, research, or QA pipeline where agents should dynamically specialize based on incoming work
- When scaling an agent swarm from 3 to 20+ agents without rewriting the coordination logic

## Key Technique

**Publish-Subscribe Decoupling.** RAPS borrows from distributed pub/sub networking. Each agent declares a *subscription* (a semantic intent, implemented as a system prompt like "Python debugging expert" or "security auditor"). When an agent produces a message (a *publication*), a *broker* performs semantic matching -- via embedding cosine similarity or an LLM selector -- to route that message only to agents whose subscriptions align. This replaces hardwired topologies with content-centric routing: agents don't need to know who else exists, only what they care about.

**Reactive Subscription.** Agents aren't locked into their initial role. After each round, an LLM-driven prompt rewriter refines each agent's subscription based on the messages it received: `S_i(t) = rewrite(S_i(t-1), received_messages)`. A "Python Expert" agent that keeps receiving database schema questions might refine its subscription to "Python + SQL optimization expert." This enables emergent specialization without central reconfiguration.

**Bayesian Reputation.** Each agent maintains a Beta-Bernoulli belief `Beta(x, y)` about every peer, starting from an uninformative prior `Beta(1, 1)`. After each interaction, a watchdog function (an LLM auditor) produces binary evidence: did the peer's message contain factual errors or tool misuse? Positive evidence increments `x`, negative increments `y`, both discounted by a decay factor `lambda` to emphasize recent behavior. Agents also cross-check peer reports via a deviation test, downweighting witnesses whose claims diverge significantly from local observations. The broker then factors reputation into routing, starving low-reputation agents of message flow. In experiments, this kept accuracy at 83% even with 3 out of 5 agents being adversarial.

## Step-by-Step Workflow

1. **Define agent subscriptions as semantic intents.** Write each agent's initial system prompt to declare what it can do and what input it wants. Example: `"I analyze Python code for performance bottlenecks. Send me profiling data, slow functions, or optimization questions."` These are the standing subscriptions.

2. **Implement the publication function.** Each agent's publisher takes its subscription, message history, and newly received messages, then produces an output via LLM call: `publication = llm(system=subscription, messages=history + inbox)`. The publication contains reasoning, intermediate results, or tool outputs.

3. **Build the semantic broker.** Implement a matching function that routes each publication to relevant subscribers. Two options:
   - **Embedding-based** (fast): Embed the publication and all subscriptions with a model like `text-embedding-3-small`, route to top-k by cosine similarity.
   - **LLM-based** (precise): Ask an LLM to select which subscriptions match the publication's content.

4. **Add reactive subscription refinement.** After each round, pass each agent's current subscription and received messages to an LLM rewriter: `new_subscription = llm("Given your current role and these messages, refine your subscription to better serve the team.")`. Replace the agent's system prompt with the result.

5. **Initialize Bayesian reputation state.** For each agent pair `(i, j)`, initialize `first_hand = Beta(1, 1)`, `trust = Beta(1, 1)`, `reputation = Beta(1, 1)`. Store these as simple `(x, y)` tuples.

6. **Implement the watchdog auditor.** After agent `i` receives a message from agent `j`, run an LLM evaluation: `"Does this message contain factual errors, hallucinations, or tool misuse? Answer 0 (reliable) or 1 (unreliable)."` Update first-hand rating: `x_ij = lambda * x_ij + (1 - score)`, `y_ij = lambda * y_ij + score`.

7. **Implement second-hand witness checks.** Query a small subset of peers for their first-hand ratings of agent `j`. For each witness `k`, compute a deviation test: `|E[F_kj] - E[P_ij]| / sqrt(Var(F_kj) + Var(P_ij))`. If deviation exceeds threshold `delta`, mark the witness as untrustworthy (update trust rating). Otherwise, merge the witness report into the overall reputation with a discount factor `omega`.

8. **Feed reputation into brokerage.** Modify the broker to filter or downrank agents whose reputation `E[P_ij] = x / (x + y)` falls below a threshold `tau`. This isolates unreliable agents from the message flow without removing them entirely.

9. **Define termination criteria.** The system halts when either: (a) a configurable majority of agents emit a "DONE" signal in their publications, or (b) a maximum round count is reached. Aggregate final publications into the answer.

10. **Tune hyperparameters.** Start with `lambda=0.9` (reputation decay), `delta=1.5` (deviation tolerance), `omega=0.3` (witness discount), `tau=0.4` (reputation threshold). Adjust `lambda` lower for faster adaptation, `delta` higher for more permissive witness acceptance.

## Concrete Examples

**Example 1: Fault-tolerant code review pipeline**

```
User: Build a multi-agent code review system where 5 agents review PRs,
but some agents might hallucinate or give bad advice.

Approach:
1. Define 5 agent subscriptions:
   - "Security vulnerability scanner: send me code diffs with auth, crypto, or input handling"
   - "Performance reviewer: send me code with loops, queries, or data structures"
   - "Style and readability checker: send me any code diff"
   - "Test coverage analyst: send me code changes and corresponding test files"
   - "Architecture reviewer: send me changes affecting module boundaries or APIs"

2. When a PR arrives, the broker embeds the diff and routes chunks
   to matching reviewers by subscription similarity.

3. Each reviewer publishes findings. The watchdog on each agent
   evaluates peer findings: "Does this review comment contain
   a factual error about the code?" Agents that repeatedly
   hallucinate issues get their reputation decayed.

4. After round 1, reactive subscription kicks in. The security
   scanner, having seen the diff involves database queries,
   refines to: "Security + SQL injection specialist."

5. The final aggregation collects all findings from agents
   with reputation > 0.4, producing a consolidated review.

Output:
{
  "reviews": [
    {"agent": "security", "reputation": 0.91, "findings": ["SQL injection in line 42"]},
    {"agent": "performance", "reputation": 0.87, "findings": ["N+1 query in user_loader"]},
    {"agent": "style", "reputation": 0.34, "findings": [...], "status": "isolated"}
  ],
  "consensus_findings": ["SQL injection in line 42", "N+1 query in user_loader"]
}
```

**Example 2: Scalable research synthesis**

```
User: I need 10 agents to research a topic and synthesize findings,
but I want the system to work even if I scale to 20 agents later.

Approach:
1. Create agents with broad initial subscriptions:
   - "Literature searcher: I find and summarize papers on [topic]"
   - "Fact checker: send me claims with citations to verify"
   - "Synthesizer: send me verified findings to combine"
   (Repeat with domain-specific variants as needed)

2. The broker uses embedding similarity to route. No topology
   to reconfigure when adding agents -- new agents just register
   subscriptions and the broker includes them in matching.

3. Reactive subscription narrows agents over rounds. A generic
   "literature searcher" receiving messages about neural architecture
   search refines to "NAS literature specialist."

4. Bayesian reputation ensures that an agent producing unverifiable
   claims gets progressively isolated. The fact-checker's watchdog
   flags unsupported assertions, decaying that agent's reputation.

5. Termination: when synthesizer agents emit DONE or after 5 rounds.

Output:
Round 1: 10 agents active, 45 messages routed
Round 2: 10 agents, 3 subscriptions refined, 38 messages routed
Round 3: 1 agent isolated (reputation 0.29), 9 active, 31 messages
Round 4: Synthesizers emit DONE. Final report assembled from 9 agents.
```

**Example 3: Implementing the core data structures in Python**

```python
from dataclasses import dataclass, field
import math

@dataclass
class BetaBelief:
    x: float = 1.0  # positive evidence (uninformative prior)
    y: float = 1.0  # negative evidence

    @property
    def mean(self) -> float:
        return self.x / (self.x + self.y)

    @property
    def variance(self) -> float:
        s = self.x + self.y
        return (self.x * self.y) / (s * s * (s + 1))

    def update(self, success: bool, decay: float = 0.9):
        self.x = decay * self.x + (1.0 if success else 0.0)
        self.y = decay * self.y + (0.0 if success else 1.0)

    def merge_witness(self, witness: "BetaBelief", omega: float = 0.3):
        self.x += omega * (witness.x - 1)  # subtract prior
        self.y += omega * (witness.y - 1)

@dataclass
class AgentState:
    name: str
    subscription: str  # current intent / system prompt
    reputation_of: dict[str, BetaBelief] = field(default_factory=dict)
    trust_of: dict[str, BetaBelief] = field(default_factory=dict)
    history: list[str] = field(default_factory=list)

def deviation_test(witness_belief: BetaBelief, local_belief: BetaBelief, delta: float = 1.5) -> bool:
    """Returns True if witness report is incompatible with local belief."""
    diff = abs(witness_belief.mean - local_belief.mean)
    combined_std = math.sqrt(witness_belief.variance + local_belief.variance)
    if combined_std == 0:
        return False
    return (diff / combined_std) >= delta
```

## Best Practices

- **Do:** Start with embedding-based brokerage for speed, then switch to LLM-based brokerage only for tasks where semantic nuance matters (e.g., distinguishing "security review" from "performance review" in ambiguous diffs).
- **Do:** Write subscriptions as specific capability declarations, not vague roles. "I detect SQL injection and XSS in Python web code" routes better than "security agent."
- **Do:** Set the reputation decay `lambda` lower (0.7-0.8) when agents interact many times per session, so recent behavior dominates quickly.
- **Do:** Log all reputation updates and broker routing decisions for debugging. The pub/sub decoupling makes the system easy to instrument.
- **Avoid:** Skipping the uninformative prior `Beta(1,1)`. Starting from `Beta(0,0)` causes division-by-zero; starting from a strong prior like `Beta(10,1)` makes reputation too slow to react.
- **Avoid:** Running reactive subscription every round in short tasks. For tasks under 3 rounds, keep subscriptions static -- the refinement overhead isn't worth it.
- **Avoid:** Setting the reputation threshold `tau` too high (>0.7). Agents need several rounds to build reputation from the `Beta(1,1)` prior; aggressive thresholds isolate agents before they can prove themselves.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| All agents isolated | No messages routed after round N | Lower `tau` or reset reputations to prior `Beta(1,1)` |
| Subscription drift | Agents refine into overly narrow intents, missing relevant messages | Cap subscription refinement to N rounds or add a "breadth preservation" instruction to the rewriter |
| Broker routes everything to one agent | One subscription is too broad | Make subscriptions more specific; add a max-fanin limit to the broker |
| Watchdog too aggressive | Good agents flagged as unreliable for stylistic differences | Tune the watchdog prompt to focus on factual correctness only, not style |
| Slow convergence | Agents take many rounds to specialize | Increase information density in initial subscriptions; lower `lambda` for faster reputation signal |
| Witness collusion | Multiple adversarial agents vouch for each other | Increase `delta` threshold and reduce `omega` to limit second-hand influence |

## Limitations

- **LLM cost scales with agent count.** Each round requires O(N) LLM calls for publishing, plus O(N) for watchdog evaluations, plus broker calls. With 20 agents over 5 rounds, this is 200+ LLM calls minimum.
- **Reputation needs multiple rounds to be meaningful.** For one-shot tasks, the Bayesian reputation layer adds overhead with no benefit. Use it only when agents interact across 3+ rounds.
- **Broker quality bottleneck.** The broker's semantic matching determines everything. If embeddings can't distinguish "security review" from "code review," routing degrades. LLM-based brokerage helps but adds latency.
- **Not suited for strict sequential pipelines.** If your task inherently requires agent A to finish before agent B starts, a simple chain is better than pub/sub overhead.
- **Reactive subscription can overfit.** An agent that receives skewed messages in early rounds may refine its subscription away from its intended role. Monitor subscription drift.

## Reference

Li, R., Zhang, Z., Bo, X., Dai, Q., & Li, C. (2026). *Towards Adaptive, Scalable, and Robust Coordination of LLM Agents: A Dynamic Ad-Hoc Networking Perspective.* arXiv:2602.08009v1. [https://arxiv.org/abs/2602.08009v1](https://arxiv.org/abs/2602.08009v1)

Key sections: Section 3 for the pub/sub formalization (Equations 1-3), Section 4.1 for reactive subscription, Section 4.2 for the full Bayesian reputation model (Equations 4-10), and Section 5 for benchmark comparisons showing +4.8% average gain over the best prior method.