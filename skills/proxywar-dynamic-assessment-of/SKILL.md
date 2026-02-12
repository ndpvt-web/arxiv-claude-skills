---
name: "proxywar-dynamic-assessment-of"
description: "Build competitive game-arena evaluation frameworks for LLM-generated code using ProxyWar's multi-layer pipeline: agent generation, hierarchical testing, iterative repair, and TrueSkill tournaments. Triggers: 'evaluate code generation in game arenas', 'build a competitive agent tournament', 'assess LLM agents with ProxyWar', 'iterative code repair pipeline', 'TrueSkill ranking for generated agents', 'dynamic assessment of code quality beyond benchmarks'"
---

This skill enables Claude to build **competition-based evaluation systems** for LLM-generated code using the ProxyWar framework. Instead of relying on static pass/fail benchmarks, ProxyWar embeds generated agents in competitive game environments and measures operational performance through multi-agent tournaments with TrueSkill ratings. Claude can apply this technique to design game arenas, generate agent code with iterative repair loops, run round-robin tournaments, and surface the real behavioral gaps that static metrics miss entirely.

## When to Use

- When the user wants to evaluate LLM code generation quality beyond pass@1 or unit-test pass rates
- When building a competitive game environment where multiple AI agents face off (board games, card games, puzzles)
- When the user asks to set up an automated pipeline that generates agent code, tests it hierarchically, and repairs failures
- When comparing multiple LLMs or prompting strategies by their actual competitive performance rather than benchmark scores
- When designing a tournament system with proper statistical ranking (TrueSkill/ELO) for generated agents
- When the user needs to expose operational weaknesses (timeouts, runtime errors, poor strategy) that functional tests alone cannot catch
- When building agent-vs-agent evaluation for reinforcement learning, game AI, or algorithm discovery research

## Key Technique

ProxyWar's core insight is that **static benchmarks weakly predict real-world code quality** (Spearman correlation rho=0.23 between pass@1 and tournament ranking). Two models can both achieve 100% test passage yet differ by 15 percentage points in competitive win rate. The framework replaces single-shot correctness checks with a five-layer evaluation pipeline: (1) code generation from standardized game specs, (2) hierarchical testing across structure/function/logic/robustness layers, (3) iterative repair with concrete error feedback for up to 3 rounds, (4) tournament orchestration with execution isolation, and (5) TrueSkill Bayesian ranking that models each agent as a skill distribution N(mu, sigma^2).

The game environment abstraction `G = <S, A, T, R, O, s0>` unifies diverse problem types under a single interface. Agents implement one method -- `select_action(observation, action_mask) -> action` -- making the framework extensible to any domain expressible as state-observation-action cycles. Critically, each agent runs in an isolated process with strict resource limits (memory caps, 45-second per-move timeout, no network access), so the evaluation captures real operational characteristics: decision latency, memory efficiency, error handling under adversarial conditions, and strategic depth.

The iterative repair loop `(Model, Code, Errors) -> Code'` is what distinguishes this from one-shot generation benchmarks. When code fails any test layer, the full error trace is fed back to the LLM for regeneration. This measures a model's debugging capability alongside its generation capability -- some models achieve perfect repair rates while others cannot recover from even simple syntax errors.

## Step-by-Step Workflow

1. **Define the game environment formally.** Specify the state space S, action space A, transition function T, reward function R, observation function O (perfect or imperfect information), and initial state s0. Implement a `GameEnvironment` class that enforces rules, manages state transitions, and produces consistent observations for all agents.

2. **Design the agent interface.** Create a `BaseAgent` abstract class requiring a single method:
   ```python
   @abstractmethod
   def select_action(self, observation: Any, action_mask: List[bool]) -> Optional[int]:
       pass
   ```
   All generated agents must inherit from this class. The `action_mask` ensures agents can only choose legal moves.

3. **Build the hierarchical test suite.** Write four layers of tests for each game:
   - **Structure tests**: Verify syntax correctness, import validity, and interface compliance (class inherits BaseAgent, method signatures match).
   - **Function tests**: Check fundamental subtask solving (can the agent make a single valid move? handle edge states?).
   - **Logic tests**: Validate complex game-specific reasoning (does it block an opponent's winning move? handle endgame correctly?).
   - **Robustness tests**: Stress-test with adversarial inputs, boundary conditions, and timeout scenarios.

4. **Craft the generation prompt.** Structure the prompt with four components: (a) task framing emphasizing competitive performance, (b) detailed game rules with observation/action format examples, (c) code structure constraints (BaseAgent inheritance, allowed imports, method signatures), and (d) strategic guidance encouraging algorithmic creativity beyond naive baselines.

5. **Implement the iterative repair loop.** When generated code fails any test layer, capture the full error set (exception type, traceback, failing test name). Feed these errors back to the LLM alongside the original prompt and failed code. Allow up to 3 repair iterations. Track repair rate as a separate metric.

6. **Set up execution isolation.** Run each agent in a separate subprocess with: memory limits, per-move timeout (e.g., 45 seconds), no file system or network access, and exception/illegal-move detection. Agents that violate constraints forfeit the match with detailed logging.

7. **Orchestrate the tournament.** For two-player games, run full round-robin with role swapping (both as player 1 and player 2) to eliminate first-move bias. For multiplayer games, use Swiss-system or randomized scheduling. For single-player puzzles, use standardized challenge sets with fixed random seeds.

8. **Apply TrueSkill rating.** Model each agent's skill as a Gaussian distribution N(mu_i, sigma_i^2). After each match, update both mu and sigma via Bayesian inference. Rank agents by conservative skill estimate `mu - 3*sigma` (99.7% confidence lower bound). Run enough matches for sigma to converge.

9. **Collect multi-dimensional metrics.** Beyond win/loss, record: decision latency per move, runtime error incidence, timeout frequency, participation rate (fraction of games where the agent was eligible to compete), and per-game-type breakdowns. Correlate these with traditional pass@1 scores to quantify the gap.

10. **Analyze and report.** Generate a ranking table with TrueSkill ratings, win percentages, and operational metrics per agent/model. Highlight cases where benchmark-leading models underperform competitively. Identify per-game-type strengths (some models excel at perfect-information games but fail at imperfect-information ones).

## Concrete Examples

**Example 1: Building a Connect Four Arena**

User: "I want to compare GPT-4, Claude, and DeepSeek at generating Connect Four agents. Set up an evaluation framework."

Approach:
1. Define Connect Four as `G = <S, A, T, R, O, s0>` with a 6x7 grid state, 7-column action space, gravity-based transitions, and win/loss/draw rewards.
2. Implement the game engine with legal move masking (full columns are masked out).
3. Write the four-layer test suite:
   - Structure: agent inherits BaseAgent, select_action has correct signature
   - Function: agent returns a valid column index when given a non-full board
   - Logic: agent blocks opponent three-in-a-row, takes winning moves when available
   - Robustness: agent handles near-full boards, simultaneous threats, timeout scenarios
4. Generate agents from each model using a prompt that includes board representation format, action encoding, and strategic hints about threat detection.
5. Run repair loop for any failures. Feed back exact test names and tracebacks.
6. Deploy passing agents into round-robin tournament (each pair plays 10 matches as each color = 20 games per pair).
7. Compute TrueSkill ratings and report.

Output:
```
Connect Four Tournament Results (20 games per pair, role-swapped)
================================================================
Rank  Agent            TrueSkill   Win%    Avg Move Time   Errors
1     deepseek-r1      28.4        62.3%   0.8s            0
2     claude-3.5       25.1        55.0%   0.3s            0
3     gpt-4.1          19.7        41.2%   2.1s            3

Notes:
- GPT-4.1 generated minimax with alpha-beta pruning (theoretically superior)
  but runs 7x slower due to deep search, causing 3 timeout forfeits.
- DeepSeek-R1 used simple heuristic evaluation but made faster, more
  consistent decisions.
- Pass@1 was identical (100%) for all three models -- tournament reveals
  the real performance gap.
```

**Example 2: Iterative Repair Pipeline for Reversi**

User: "My LLM-generated Reversi agent keeps failing tests. Build a repair pipeline."

Approach:
1. Run the agent through hierarchical tests. Identify failure layer.
2. Capture errors:
   ```
   FAIL: test_logic_captures_flipped_pieces
   AssertionError: Agent placed at (3,4) but did not account for diagonal
   flips. Expected board state [...] but got [...]
   ```
3. Construct repair prompt:
   ```
   Your Reversi agent failed the following test:
   - test_logic_captures_flipped_pieces: placing at (3,4) missed diagonal
     flip direction. The agent must check all 8 directions (N,S,E,W,NE,NW,
     SE,SW) for opponent pieces bounded by a friendly piece.

   Original code: [attached]
   Error traceback: [attached]

   Fix the directional scanning logic in select_action.
   ```
4. Regenerate. Re-run tests. If still failing, repeat with updated errors (max 3 iterations).
5. Track: iteration count to pass, which test layers failed, repair success rate.

Output:
```
Repair Pipeline Results
=======================
Iteration 1: FAIL (logic layer - missed diagonal flips)
Iteration 2: FAIL (robustness layer - crashed on full board edge case)
Iteration 3: PASS (all 4 layers)

Repair rate: 1/1 (succeeded within budget)
Iterations needed: 3
Failure pattern: directional scanning -> edge case handling
```

**Example 3: Multi-Game Model Comparison**

User: "Compare 5 code-generation models across multiple game types to find which is best overall."

Approach:
1. Select games spanning complexity and information types:
   - Tic-Tac-Toe (simple, perfect info, solved)
   - Connect Four (medium, perfect info)
   - Texas Hold'em (complex, imperfect info)
2. Generate agents from each model for each game. Run repair loops.
3. Run per-game tournaments. Compute TrueSkill per game.
4. Aggregate: compute cross-game ranking, identify per-game-type specialists.
5. Measure correlation between perfect-info and imperfect-info performance.

Output:
```
Cross-Game Ranking (5 models x 3 games)
========================================
              Tic-Tac-Toe  Connect4  Hold'em  Aggregate
Model A       1st          3rd       2nd      2nd
Model B       2nd          1st       4th      3rd
Model C       3rd          2nd       1st      1st
Model D       4th          4th       3rd      4th
Model E       5th          5th       5th      5th

Cross-game correlation (perfect vs imperfect info): rho=0.28
Conclusion: No single model dominates all game types. Model C's
imperfect-information strength makes it the best generalist despite
not leading any perfect-information game.
```

## Best Practices

- **Do:** Swap roles in two-player games (each agent plays as both player 1 and player 2) to eliminate positional advantage bias from your rankings.
- **Do:** Use `action_mask` in the agent interface to guarantee legal moves -- never rely on the agent to self-enforce game rules.
- **Do:** Set strict, uniform resource limits (memory, time per move) across all agents. A brilliant algorithm that times out is worse than a simple one that finishes.
- **Do:** Run enough tournament rounds for TrueSkill sigma to converge (typically 20+ games per agent pair). Premature rankings are unreliable.
- **Avoid:** Treating pass@1 as a proxy for competitive quality. The paper shows rho=0.23 correlation -- passing tests says almost nothing about winning games.
- **Avoid:** Giving agents access to the file system, network, or shared memory during tournament matches. Isolation is essential for fair and safe evaluation.
- **Avoid:** Comparing agents across games with incompatible complexity. A model's Tic-Tac-Toe rating tells you little about its Reversi ability (performance variance scales with game complexity).

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Agent throws exception mid-game | Unhandled game state | Forfeit the match, log the exception, and include it in repair feedback for the next iteration |
| Agent exceeds move timeout | Overly complex algorithm (e.g., deep minimax without pruning) | Enforce timeout strictly, record as a loss. In repair prompt, explicitly state the time constraint |
| Agent returns illegal action despite action_mask | Ignoring the mask parameter | Structure test catches this. Repair prompt should emphasize mask compliance with a concrete example |
| TrueSkill ratings not converging | Too few matches | Increase tournament rounds. Monitor sigma values -- stop when max sigma < 1.0 |
| All agents tie repeatedly | Game too simple or agents too similar | Add logic-layer tests that demand strategic depth, or choose a more complex game environment |
| Repair loop exhausts iterations | Model cannot debug its own output | Log the persistent failure pattern. Flag the model as unable to self-repair for this game type. Consider a different prompting strategy |

## Limitations

- **Game-domain specificity**: The framework evaluates code that interacts through a state-observation-action loop. It does not directly assess code generation for APIs, data pipelines, web services, or other non-game software without adapting the environment abstraction.
- **Computational cost**: Full round-robin tournaments scale as O(n^2) in agent count. For large model comparisons (18+ models x 10 games x multiple rounds), this requires significant compute. Swiss-system scheduling helps but reduces ranking precision.
- **Single-method interface constraint**: Agents are evaluated through `select_action` only. This misses code quality attributes like modularity, readability, and maintainability. ProxyWar measures what the code *does*, not how it's structured.
- **Imperfect information games require more matches**: Poker-style games have high variance per match. Rankings need substantially more games to stabilize compared to deterministic games like Connect Four.
- **Repair budget ceiling**: Capping repair at 3 iterations is practical but means some fixable agents get discarded. Increasing the budget improves coverage but increases cost linearly.

## Reference

**Paper**: [ProxyWar: Dynamic Assessment of LLM Code Generation in Game Arenas](https://arxiv.org/abs/2602.04296v1) (ICSE 2026) -- Peng, Wang, Wu. Look for Section 3 (framework architecture), Section 4 (game environment definitions and hierarchical test design), and Section 5 (tournament results showing the weak correlation between static benchmarks and competitive performance).