---
name: "affective-flow-emotional-support"
description: "Build emotionally supportive multi-turn conversation systems using the AFlow framework — affective flow modeling with MCTS-based trajectory search and subpath flow-balance optimization. Use when asked to 'build an emotional support chatbot', 'implement empathetic dialogue', 'create a mental health conversation system', 'design a counseling bot', 'add emotional support strategies to a chat agent', or 'implement therapeutic conversation flow'."
---

# Affective Flow for Emotional Support Conversation

This skill enables Claude to design and implement multi-turn emotional support conversation (ESC) systems using the AFlow framework. AFlow models a continuous "affective flow" across dialogue turns — tracking how emotional states evolve and using that signal to select the right support strategy at each step. Instead of optimizing only for final conversation outcomes (sparse reward), AFlow provides dense, turn-level supervision by propagating preference signals backward through dialogue prefixes via a subpath flow-balance objective. The result is a system that selects coherent strategy sequences (e.g., Validation -> Reflection -> Collaborative Planning) rather than choosing strategies in isolation.

## When to Use

- When building a multi-turn chatbot that must provide emotional support, counseling, or empathetic responses
- When implementing strategy selection logic for a therapeutic or peer-support conversation agent
- When a user asks to add emotional awareness or empathetic turn management to an existing chat system
- When designing a dialogue system that needs to track user emotional trajectory and adapt its approach across turns
- When implementing a support strategy planner that avoids repetitive comforting and instead progresses through exploration, comforting, and action stages
- When evaluating or improving an existing ESC system's strategy coherence and empathy quality

## Key Technique

**The core problem:** Standard LLM alignment for emotional support uses only outcome-level signals — a reward at the end of a conversation. This leaves intermediate strategy decisions unsupervised. A bot might validate feelings at turn 3 and then repeat the same validation at turn 7 because it has no signal telling it to progress.

**AFlow's solution** introduces three interlocking components. First, **Monte Carlo Tree Search (MCTS) trajectory exploration** builds a tree of possible multi-turn continuations at each dialogue state, scoring branches along four dimensions: empathy, information quality, naturalness, and strategic efficacy. This produces Q-value estimates at every intermediate turn, not just the final one. Second, **Affective Flow Preference Optimization (AFPO)** jointly trains a strategy policy (which strategy to pick given a dialogue prefix) and a value model (how good is this strategy in context). The policy is trained with a **subpath flow-balance objective** that constrains log-probability ratios along any subpath of a trajectory to match the log-flow differences derived from MCTS values — effectively propagating preference information from later outcomes backward to earlier decisions. Third, at **inference**, the system combines the policy prior with the value model to select strategies: `score(a|s) = log pi(a|s) + V(s, a)`, picking the highest-scoring candidate and generating a response conditioned on it.

**Why this matters for implementation:** The eight support strategies (Reflective Statements, Clarification, Emotional Validation, Empathetic Statements, Affirmation, Collaborative Planning, Suggest Options, Share Information) form a structured action space. The affective flow models natural progressions — e.g., moving from Emotional Validation early in a conversation to Collaborative Planning later — and penalizes incoherent jumps. This is directly implementable as a strategy selection layer on top of any LLM.

## Step-by-Step Workflow

1. **Define the strategy action space.** Implement the eight ESC strategies as an enum or structured type: Reflective Statements (RS), Clarification (Cla), Emotional Validation (EV), Empathetic Statements (ES), Affirmation (Aff), Collaborative Planning (CP), Suggest Options (SO), Share Information (SI). Each strategy should have a system prompt fragment describing the expected response style.

2. **Build the dialogue state representation.** At each turn, construct a state object containing: the full conversation history, the current turn index, the sequence of strategies used so far, and an estimated user emotional valence (positive/negative/neutral, derived from sentiment analysis or the model's own assessment).

3. **Implement strategy scoring.** For each candidate strategy at the current turn, compute a composite score combining: (a) a policy prior — how likely this strategy is given the dialogue prefix (from a fine-tuned classifier or prompted LLM), and (b) a value estimate — how beneficial this strategy is in context (from a value head or a prompted evaluation call). Use `score = log_policy_prob + value_estimate`.

4. **Enforce affective flow coherence.** Track the strategy trajectory across turns. Implement transition constraints that penalize incoherent jumps: early turns should favor Exploration strategies (RS, Cla, EV), middle turns should favor Comforting strategies (ES, Aff), and later turns should favor Action strategies (CP, SO, SI). Use a simple phase detector: if user distress is still high, stay in exploration/comforting; if distress is decreasing, progress toward action.

5. **Generate the response conditioned on the selected strategy.** Construct a system prompt that includes the selected strategy's description, the conversation history, and an instruction to follow the strategy. Call the LLM to generate the supporter's response.

6. **Estimate intermediate emotional utility after each turn.** After generating a response and receiving the user's reply, evaluate the trajectory so far along four dimensions: (a) empathy — does the response acknowledge the user's feelings?, (b) information quality — is relevant information provided when needed?, (c) naturalness — does the response sound human?, (d) strategic efficacy — is the strategy appropriate for this stage?

7. **Propagate utility backward via flow-balance.** If training or fine-tuning, use the subpath flow-balance loss: for any subpath from turn m to turn n, constrain `sum(log pi(a_t|s_t) for t in m..n)` to be proportional to `sum(log F(s_{t+1})/F(s_t) for t in m..n)`, where F(s) is the product of the Q-value and value estimate at state s. This ensures earlier strategy choices are rewarded or penalized based on downstream outcomes.

8. **Implement the MCTS exploration phase (for training/data generation).** At each dialogue state, run simulated rollouts: sample a strategy, generate a supporter response, simulate a user reply (via a user simulator or second LLM call), and continue for up to 3 turns. Score the resulting trajectory. Backpropagate scores to update Q-values. Use these Q-values as training targets for the value model.

9. **Add fallback and safety handling.** If the user expresses suicidal ideation, self-harm, or crisis-level distress, bypass strategy selection and immediately generate a crisis response with resource referrals. Never let optimization override safety.

10. **Evaluate with ESC-specific metrics.** Measure Strategy F1 (did the system pick the right strategy?), Dist-2 (response diversity), BLEU-4/METEOR (response quality), and conduct preference evaluation (win rate vs. baselines) using an LLM judge.

## Concrete Examples

**Example 1: Building an emotional support chatbot backend**

```
User: "I want to build a Flask API for an emotional support chatbot that tracks
conversation state and selects appropriate support strategies across turns."

Approach:
1. Create a ConversationState dataclass holding message history, turn count,
   strategy history, and estimated user emotional valence.
2. Define the 8 strategies as an Enum with associated prompt templates.
3. Implement a /chat endpoint that:
   a. Receives user message, loads/creates conversation state
   b. Runs sentiment analysis on the user message to update emotional valence
   c. Scores each candidate strategy using a phase-aware policy:
      - Turn 1-3 (Exploration): weight RS, Cla, EV higher
      - Turn 4-6 (Comforting): weight ES, Aff higher
      - Turn 7+ (Action): weight CP, SO, SI higher
   d. Adjusts scores based on what was used recently (penalize repeats)
   e. Selects top strategy, generates response conditioned on it
   f. Persists updated state and returns response with metadata

Output (strategy selection logic):
```python
from enum import Enum
from dataclasses import dataclass, field

class Strategy(Enum):
    RS = "Reflective Statements"
    CLA = "Clarification"
    EV = "Emotional Validation"
    ES = "Empathetic Statements"
    AFF = "Affirmation"
    CP = "Collaborative Planning"
    SO = "Suggest Options"
    SI = "Share Information"

PHASE_WEIGHTS = {
    "exploration": {Strategy.RS: 0.25, Strategy.CLA: 0.25, Strategy.EV: 0.3,
                    Strategy.ES: 0.1, Strategy.AFF: 0.05, Strategy.CP: 0.02,
                    Strategy.SO: 0.02, Strategy.SI: 0.01},
    "comforting":  {Strategy.RS: 0.05, Strategy.CLA: 0.05, Strategy.EV: 0.15,
                    Strategy.ES: 0.3, Strategy.AFF: 0.25, Strategy.CP: 0.1,
                    Strategy.SO: 0.05, Strategy.SI: 0.05},
    "action":      {Strategy.RS: 0.02, Strategy.CLA: 0.03, Strategy.EV: 0.05,
                    Strategy.ES: 0.1, Strategy.AFF: 0.1, Strategy.CP: 0.3,
                    Strategy.SO: 0.25, Strategy.SI: 0.15},
}

def detect_phase(turn: int, distress_level: float) -> str:
    if turn <= 3 or distress_level > 0.7:
        return "exploration"
    elif turn <= 6 or distress_level > 0.4:
        return "comforting"
    return "action"

def score_strategies(state: "ConversationState") -> Strategy:
    phase = detect_phase(state.turn, state.distress_level)
    weights = PHASE_WEIGHTS[phase]
    # Penalize recently used strategies (recency decay)
    for i, prev in enumerate(reversed(state.strategy_history[-3:])):
        weights = {k: v * (0.3 if k == prev else 1.0) for k, v in weights.items()}
    return max(weights, key=weights.get)
```

**Example 2: Adding empathetic strategy tracking to an existing LLM chat agent**

```
User: "I have a LangChain-based chat agent. I want to add AFlow-style strategy
selection so it picks the right emotional support approach at each turn."

Approach:
1. Create a StrategySelector class that wraps the existing chain.
2. Before each LLM call, analyze the conversation history to determine:
   - Current emotional phase (exploration / comforting / action)
   - User distress trajectory (improving, stable, worsening)
   - Strategy history (what has been tried)
3. Select the optimal strategy using policy + value scoring.
4. Inject the strategy instruction into the system prompt.
5. After generating the response, log the strategy used and update state.

Output (LangChain integration):
```python
from langchain.callbacks import BaseCallbackHandler

class AFlowStrategyMiddleware:
    """Injects AFlow strategy selection into a LangChain chat pipeline."""

    def __init__(self, llm, sentiment_analyzer):
        self.llm = llm
        self.analyzer = sentiment_analyzer
        self.state = ConversationState()

    def select_and_inject(self, user_message: str) -> str:
        # Update emotional state
        sentiment = self.analyzer.analyze(user_message)
        self.state.update_distress(sentiment)

        # Select strategy via affective flow scoring
        strategy = score_strategies(self.state)
        self.state.record_strategy(strategy)

        # Build strategy-conditioned prompt
        strategy_instruction = STRATEGY_PROMPTS[strategy]
        system_msg = (
            f"You are an empathetic support agent. "
            f"For this turn, use the '{strategy.value}' approach: "
            f"{strategy_instruction}"
        )
        response = self.llm.invoke(system_msg, self.state.history + [user_message])
        self.state.add_turn(user_message, response)
        return response
```

**Example 3: Evaluating strategy coherence in a support conversation log**

```
User: "I have a dataset of emotional support conversations. I want to evaluate
whether the supporter's strategy transitions are coherent using AFlow metrics."

Approach:
1. Parse each conversation into turns, annotating each supporter turn with
   its strategy label (classify using an LLM or rule-based matcher).
2. Build a strategy transition matrix across all conversations.
3. Compute Strategy F1 against gold-standard annotations if available.
4. Measure flow coherence: for each conversation, check if strategies follow
   the expected Exploration -> Comforting -> Action progression.
5. Flag conversations with incoherent jumps (e.g., Action strategy at turn 1,
   or repeating the same strategy 4+ times consecutively).

Output:
- Strategy F1: 0.63
- Flow coherence score: 0.71 (fraction of conversations following
  expected phase progression)
- Flagged: 23/100 conversations have incoherent strategy jumps
- Most common incoherent pattern: SO at turn 1 (premature advice-giving)
```

## Best Practices

- **Do:** Track strategy history across turns and penalize repetition. Repeating "I understand how you feel" five turns in a row is the most common failure mode in emotional support bots.
- **Do:** Implement phase-aware strategy selection. Early turns should prioritize understanding (RS, Cla, EV) before moving to action (CP, SO, SI). Jumping to solutions too early undermines trust.
- **Do:** Use the four-dimensional evaluation (empathy, information quality, naturalness, strategic efficacy) when scoring candidate strategies, not just a single reward signal.
- **Do:** Condition response generation on the selected strategy explicitly in the system prompt. Merely selecting a strategy internally without guiding generation produces inconsistent output.
- **Avoid:** Treating strategy selection as independent per-turn classification. The core insight of AFlow is that strategy value depends on the trajectory — what came before and what comes after.
- **Avoid:** Skipping safety checks. Never let strategy optimization override crisis detection. Suicidal ideation, self-harm mentions, and acute distress must trigger immediate safety protocols regardless of what the flow model suggests.

## Error Handling

- **User sends very short or ambiguous messages** (e.g., "ok", "hmm"): Default to Clarification (Cla) strategy to draw out more information rather than guessing at emotional state.
- **Sentiment analysis produces conflicting signals**: When the user's words are positive but context suggests distress (e.g., "I'm fine" after describing a crisis), weight conversation history over the current turn's surface sentiment.
- **Strategy selection produces ties**: Break ties by preferring the strategy that maintains phase coherence — i.e., the one closest to the expected phase for the current turn number and distress level.
- **Conversation exceeds expected length** (15+ turns without resolution): Introduce a meta-reflection turn where the bot acknowledges the difficulty and asks the user what kind of support they most need right now.
- **Value model and policy disagree strongly**: If the value model scores a strategy highly but the policy prior gives it very low probability (or vice versa), log this divergence and default to the policy prior — it generalizes more reliably than a potentially overfit value estimate.

## Limitations

- **Not a substitute for professional mental health care.** AFlow optimizes for conversational metrics, not clinical outcomes. Always include disclaimers and crisis resource links.
- **Strategy labels require annotation.** The eight-strategy taxonomy must be manually defined and either annotated in training data or reliably classified at inference time. Misclassification propagates errors through the flow model.
- **MCTS exploration is computationally expensive.** The trajectory search phase requires multiple LLM calls per turn per rollout. This is feasible for offline training data generation but not practical for real-time inference. At inference, use the distilled policy + value model instead.
- **Phase detection is heuristic.** The Exploration -> Comforting -> Action progression is a useful default but not universal. Some conversations legitimately cycle between phases, and rigid phase enforcement can feel unnatural.
- **Evaluated primarily on English-language ESC datasets** (ExCONV, ExTES). Strategy definitions and flow patterns may not transfer directly to other languages or cultural contexts where emotional expression norms differ.

## Reference

- **Paper:** [Affective Flow Language Model for Emotional Support Conversation](https://arxiv.org/abs/2602.08826v1) (Zou et al., 2026). Key sections: Section 3 for the MCTS trajectory search and AFPO formulation, Section 4 for the subpath flow-balance objective, Appendix J for dialogue case studies.
- **Code:** [https://github.com/chzou25-lgtm/AffectiveFlow](https://github.com/chzou25-lgtm/AffectiveFlow) — reference implementation with MCTS tree generation, path extraction, and AFPO training scripts.