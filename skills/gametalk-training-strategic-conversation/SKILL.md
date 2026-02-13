---
name: "gametalk-training-strategic-conversation"
description: "Build multi-agent strategic conversation systems where LLMs negotiate, coordinate, and optimize long-term objectives through dialogue. Uses conversation-level reward signals with DPO/GRPO/STaR fine-tuning. Triggers: 'build a negotiation agent', 'train LLM for strategic dialogue', 'multi-agent conversation game', 'implement game-theoretic fine-tuning', 'dialogue-based coordination system', 'LLM bargaining framework'"
---

# GameTalk: Training LLMs for Strategic Conversation

This skill enables Claude to build systems where LLMs conduct strategic multi-turn conversations -- negotiating, coordinating, and optimizing outcomes across full dialogues rather than individual turns. Based on the GameTalk framework, the core insight is that fine-tuning methods (DPO, GRPO, STaR) can be adapted to use **conversation-level reward signals** that evaluate the entire interaction, producing agents that plan ahead, model opponents, and achieve better strategic outcomes than turn-level optimization.

## When to Use

- When the user wants to build a negotiation or bargaining agent that optimizes deal outcomes across a full dialogue
- When implementing multi-agent systems where LLMs must coordinate through conversation (e.g., resource allocation, task delegation)
- When creating game-playing agents that communicate strategically (Prisoner's Dilemma, ultimatum games, trading simulations)
- When fine-tuning an LLM to reason about opponent behavior and adapt strategy mid-conversation
- When building a training pipeline that generates strategic dialogue data and ranks conversations by outcome quality
- When the user asks to implement DPO/GRPO with rewards that depend on multi-turn interaction history rather than single responses

## Key Technique

**Conversation-level optimization** is the central idea. Standard RLHF/DPO treats each response independently -- a reward model scores individual outputs. GameTalk instead assigns rewards based on how the full conversation resolves: did the agent reach an agreement? How favorable were the terms? Did it cooperate when cooperation was optimal? This means the training signal propagates backward through every turn, teaching the model that an early concession might yield a better final deal, or that a tough opening stance pays off three turns later.

The framework adapts three fine-tuning methods. **DPO (Direct Preference Optimization)** pairs winning conversations against losing ones -- same game setup, different dialogue trajectories, and the model learns to prefer the trajectory that led to a better outcome. **GRPO (Group Relative Policy Optimization)** samples multiple conversation rollouts per game scenario and uses relative rankings as the reward signal. **STaR (Self-Taught Reasoner)** bootstraps by having the model generate chain-of-thought reasoning about strategic choices, then filtering for reasoning traces that led to successful outcomes. DPO consistently yields the strongest gains because it directly optimizes the contrast between good and bad conversation strategies.

**Reward shaping** is critical. Raw game payoffs are sparse (you only know the outcome at the end), so the framework adds intermediate signals: partial agreement progress, information gained about the opponent's preferences, and negotiation efficiency (reaching good deals in fewer turns). These shaped rewards make training tractable and produce agents that exhibit coherent multi-turn strategy rather than greedy turn-by-turn behavior.

## Step-by-Step Workflow

1. **Define the strategic interaction as a structured game.** Specify the players, their possible actions/agreements, the payoff function mapping outcomes to numerical rewards, and any constraints (e.g., both parties must agree for a deal to execute). Encode this as a JSON game specification with fields for `players`, `actions`, `payoffs`, and `termination_conditions`.

2. **Design the conversation protocol.** Set the turn structure: how many rounds of dialogue are allowed, what format messages take (natural language, structured proposals, or hybrid), and how the game terminates (explicit agreement, timeout, or unilateral action). Implement this as a `ConversationManager` class that tracks state and enforces rules.

3. **Build the self-play data generation pipeline.** Run pairs of LLM agents against each other across many game instances. For each game, generate 4-8 complete conversation rollouts with different sampling temperatures (0.7-1.0) to produce diverse strategies. Store each rollout as `(game_config, conversation_turns[], final_outcome, per_turn_rewards[])`.

4. **Compute conversation-level rewards with shaping.** For each completed dialogue, calculate the primary reward from the game payoff. Then add shaped intermediate rewards: +0.1 for each turn where the agent extracted useful information about the opponent, +0.2 for making a proposal that moves closer to the Nash equilibrium, -0.1 for contradicting a previous commitment. Normalize rewards across all rollouts for a given game instance.

5. **Construct training pairs for DPO (recommended) or ranking data for GRPO.** For DPO: pair the highest-reward conversation against the lowest-reward conversation for the same game setup. The "chosen" trajectory is the full sequence of the agent's messages in the winning conversation; the "rejected" trajectory is the losing one. Crucially, include the full conversation context (both players' messages) so the model learns conditional strategy. For GRPO: rank all rollouts per game instance by reward and use group-relative scores.

6. **Format training data as conversation-completion pairs.** Structure each example as: system prompt (game rules + role assignment) -> alternating user/assistant turns (opponent messages as user, agent messages as assistant) -> final outcome annotation. This preserves the multi-turn structure that standard chat fine-tuning expects while carrying conversation-level reward information.

7. **Fine-tune with conversation-aware loss.** Run DPO fine-tuning (or GRPO/STaR) using the paired data. Use LoRA (rank 16-64) on attention layers for parameter efficiency. Key hyperparameters: DPO beta=0.1-0.3 (lower beta makes the model diverge more from the reference, allowing stronger strategic adaptation), learning rate 1e-6 to 5e-6, batch size of 4-8 conversation pairs.

8. **Evaluate against diverse opponents.** Test the fine-tuned model against: (a) the base untrained model, (b) a copy of itself (self-play), (c) hand-crafted strategy bots (always-cooperate, tit-for-tat, random). Measure average payoff, agreement rate, negotiation efficiency (turns to agreement), and exploitability (how much a best-response opponent can extract).

9. **Iterate with curriculum difficulty.** Start training on simpler games (e.g., binary cooperation choices) and progressively introduce harder scenarios (multi-issue negotiation, incomplete information, more than two players). Each stage uses the previous stage's model as the base for data generation and fine-tuning.

10. **Deploy with opponent modeling at inference.** At serving time, wrap the fine-tuned model with a lightweight prompt that tracks the current conversation state, inferred opponent type (cooperative, aggressive, random), and remaining negotiation budget. Update opponent-type estimates after each opponent turn using Bayesian updating over a small set of strategy archetypes.

## Concrete Examples

**Example 1: Resource Negotiation Agent**

User: "Build a system where two LLM agents negotiate how to split a pool of 100 resources, where each agent has different private valuations for the resources."

Approach:
1. Define game: 3 resource types (A, B, C), 100 units total. Agent 1 values A=3, B=2, C=1. Agent 2 values A=1, B=2, C=3. Payoff = sum of (quantity * value) for allocated resources.
2. Conversation protocol: 5 rounds max. Each turn, agent proposes a split or accepts/rejects the opponent's proposal. Natural language with embedded structured proposals.
3. Generate 500 game instances with randomized valuations. Run 6 rollouts per instance using Llama-3-8B at temperature 0.8.
4. Compute rewards: primary = agent payoff / maximum possible payoff. Shaped rewards: +0.15 for proposals closer to Pareto-optimal frontier, -0.1 for ultimatums that reduce agreement probability.
5. DPO pairs: for each game instance, pair the rollout with highest normalized payoff against the lowest.

Output structure:
```python
# game_config.json
{
  "players": 2,
  "resources": {"A": 100, "B": 100, "C": 100},
  "valuations": {
    "player_1": {"A": 3, "B": 2, "C": 1},
    "player_2": {"A": 1, "B": 2, "C": 3}
  },
  "max_rounds": 5,
  "no_deal_payoff": 0
}

# conversation_manager.py
class NegotiationManager:
    def __init__(self, game_config):
        self.config = game_config
        self.history = []
        self.proposals = []

    def step(self, agent_id, message, proposal=None):
        self.history.append({"agent": agent_id, "message": message})
        if proposal and proposal.get("accept"):
            return self.resolve(self.proposals[-1])
        if proposal:
            self.proposals.append(proposal)
        return None  # game continues

    def compute_reward(self, agent_id, outcome):
        if outcome is None:  # no deal
            return self.config["no_deal_payoff"]
        vals = self.config["valuations"][f"player_{agent_id}"]
        return sum(outcome[agent_id].get(r, 0) * v for r, v in vals.items())

# dpo_data_generator.py
def generate_dpo_pairs(game_configs, model, n_rollouts=6):
    pairs = []
    for config in game_configs:
        rollouts = []
        for _ in range(n_rollouts):
            conv = run_self_play(model, config, temperature=0.8)
            reward = compute_shaped_reward(conv, config)
            rollouts.append((conv, reward))
        rollouts.sort(key=lambda x: x[1])
        pairs.append({
            "chosen": rollouts[-1][0],  # best outcome
            "rejected": rollouts[0][0],  # worst outcome
            "game_config": config
        })
    return pairs
```

**Example 2: Cooperative Task Delegation**

User: "I want to train an LLM to coordinate task assignment through dialogue in a team of 3 agents, where each agent has different capabilities and tasks have varying requirements."

Approach:
1. Define game: 3 agents, 6 tasks. Each agent has skill vector [0-1] across 3 dimensions. Each task has requirement vector. Payoff = sum of task completion quality (skill-requirement match) across all assigned tasks.
2. Conversation: round-robin dialogue, 4 full rounds. Agents propose task claims, discuss tradeoffs, finalize assignments.
3. Generate data via self-play with 3 copies of base model. Reward depends on team-level outcome (sum of all agents' payoffs), encouraging cooperation.
4. Key DPO insight: "chosen" conversations are ones where agents truthfully revealed capabilities and reached efficient allocations; "rejected" ones are where agents hoarded easy tasks or miscommunicated.

Output structure:
```python
# Multi-agent conversation format for training
training_example = {
    "system": "You are Agent 1 in a team of 3. Your skills: coding=0.9, "
              "design=0.3, writing=0.6. Negotiate task assignments to "
              "maximize team performance. Be honest about your strengths.",
    "conversations": [
        {"role": "user", "content": "[Agent 2]: I'm strong at design (0.8) "
         "and decent at writing (0.5). I'd like to claim the UI task."},
        {"role": "assistant", "content": "That makes sense -- your design "
         "strength is much higher than mine. I can take the backend API task "
         "since my coding skill is 0.9. What about the documentation task? "
         "My writing is 0.6, which is moderate."},
        {"role": "user", "content": "[Agent 3]: My writing is 0.85, I should "
         "take documentation. Can someone help with the testing task though?"},
        {"role": "assistant", "content": "I can handle testing alongside the "
         "API work since they're related. Let me propose: I take API + testing, "
         "Agent 2 takes UI + styling, Agent 3 takes docs + content review."}
    ],
    "reward": 0.87  # team efficiency score
}
```

**Example 3: Iterated Prisoner's Dilemma with Communication**

User: "Implement the GameTalk training loop for an iterated Prisoner's Dilemma where agents can talk before choosing cooperate/defect each round."

Approach:
1. Game: 10-round IPD. Before each round, agents exchange 2 messages. Then simultaneously choose C or D. Payoffs: CC=(3,3), CD=(0,5), DC=(5,0), DD=(1,1).
2. Conversation-level reward: total payoff across all 10 rounds, normalized. Shaped reward: +0.05 per round of mutual cooperation maintained, -0.2 for breaking a verbal commitment.
3. Generate rollouts, construct DPO pairs where "chosen" = conversations leading to sustained cooperation, "rejected" = conversations where defection spirals occurred.

```python
# Reward shaping for IPD
def compute_shaped_reward(conversation, actions_per_round):
    base_reward = sum(PAYOFF[a1][a2] for a1, a2 in actions_per_round)

    # Bonus for sustained cooperation
    coop_streak = 0
    for a1, a2 in actions_per_round:
        if a1 == "C" and a2 == "C":
            coop_streak += 1
            base_reward += 0.05 * coop_streak  # increasing bonus
        else:
            coop_streak = 0

    # Penalty for breaking verbal commitments
    for round_idx, (a1, a2) in enumerate(actions_per_round):
        pre_round_msgs = get_messages_before_round(conversation, round_idx)
        if promised_cooperation(pre_round_msgs, agent=0) and a1 == "D":
            base_reward -= 0.2

    return base_reward / MAX_POSSIBLE_REWARD
```

## Best Practices

- **Do:** Use DPO over GRPO/STaR as the default -- it consistently produces the strongest strategic gains in the GameTalk evaluation, and it is simpler to implement since it only needs pairwise comparisons rather than group rankings.
- **Do:** Include the full conversation context (both players' turns) in training examples. The model must learn conditional strategy -- what to say given what the opponent has said -- not just good monologues.
- **Do:** Apply reward shaping with intermediate signals. Pure outcome-based rewards are too sparse for conversations longer than 3-4 turns. Shape rewards that credit information gathering, commitment consistency, and proposal quality.
- **Do:** Generate diverse rollouts by varying temperature (0.7-1.0) and using multiple game configurations. DPO needs genuine strategic contrast between chosen/rejected pairs, not just surface variation.
- **Avoid:** Training on single-turn rewards or treating each message independently. This produces locally greedy agents that make short-sighted concessions or aggressive moves that backfire later.
- **Avoid:** Using identical agents for both sides in all training data. Include asymmetric game configurations and varied opponent strategies to prevent the model from overfitting to self-play equilibria that collapse against different opponents.

## Error Handling

- **Degenerate conversations (agents repeat themselves or refuse to engage):** Filter rollouts by minimum unique proposal count. Require at least 2 distinct proposals per conversation before including in training data. Increase temperature if >30% of rollouts are degenerate.
- **Reward hacking (agents find loopholes in the game protocol):** Validate that final agreements are actually feasible under game constraints. Add a feasibility checker in the `ConversationManager` that rejects invalid proposals before reward computation.
- **Mode collapse after fine-tuning (agent uses one rigid strategy):** Monitor strategy diversity across evaluation games. If the agent plays identically regardless of opponent behavior, reduce DPO beta (making the model stay closer to the reference policy) and increase rollout diversity in the next data generation round.
- **Opponent modeling failure (agent doesn't adapt to different opponents):** Ensure training data includes games against varied opponent types. At inference, add explicit opponent-type tracking in the system prompt and update it after each opponent turn.

## Limitations

- Conversation-level rewards require running complete dialogues to evaluate, making data generation expensive -- each training example costs 5-20x more inference than single-turn RLHF.
- The framework assumes a well-defined payoff function. It does not apply to open-ended negotiations where "success" is subjective or hard to quantify.
- Models fine-tuned for specific game types may not transfer well to structurally different games. A model trained on bilateral negotiation may not perform well in coalition-formation settings with 4+ agents.
- DPO requires meaningful contrast between chosen/rejected pairs. If the base model already performs well (or terribly) at a game, rollout diversity may be insufficient to construct useful training pairs.
- Strategic conversation fine-tuning can produce models that are more manipulative or deceptive. Consider alignment guardrails if deploying in user-facing contexts.

## Reference

**Paper:** [GameTalk: Training LLMs for Strategic Conversation](https://arxiv.org/abs/2601.16276v1) (Conchello Vendrell, Ruiz Luyten, van der Schaar, 2026). Key sections: Section 3 for the conversation-level DPO/GRPO/STaR adaptation, Section 4 for the game suite and reward shaping formulations, Section 5 for ablations showing DPO's dominance and the importance of reward shaping.