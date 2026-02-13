---
name: "from-gameplay-traces-game"
description: "Reverse-engineer game mechanics from gameplay traces using a two-stage causal induction pipeline: first infer a Structural Causal Model (SCM) from observations, then translate it into executable game rules (VGDL or equivalent). Trigger phrases: 'infer game mechanics from traces', 'reverse-engineer game rules', 'build causal model from gameplay', 'extract game logic from observations', 'generate VGDL from gameplay', 'causal induction for games'"
---

# From Gameplay Traces to Game Mechanics: Causal Induction Pipeline

This skill enables Claude to reverse-engineer the rules and mechanics of a game from observational gameplay traces using a two-stage causal induction approach. Rather than generating game code directly from raw observations (which tends to produce inconsistent rules), Claude first infers an intermediate **Structural Causal Model (SCM)** that captures entity relationships and cause-effect dynamics, then translates that SCM into executable game descriptions in VGDL or equivalent rule systems. This technique, drawn from Jiwatode et al. (2026), achieves up to 81% preference win rates over direct generation in blind evaluations.

## When to Use

- When the user provides gameplay logs, frame sequences, or state-action traces and asks to reconstruct the game's rules
- When building an interpretable model of a game environment from observation data for reinforcement learning
- When the user wants to procedurally generate new games that are mechanically consistent with observed gameplay patterns
- When reverse-engineering interaction rules from a grid-based or arcade-style game for modding, cloning, or analysis
- When the user needs to convert informal gameplay descriptions into a formal executable game specification (VGDL, PuzzleScript, etc.)
- When debugging or validating game logic by comparing inferred rules against observed behavior

## Key Technique

**Why not generate rules directly?** Direct code generation from gameplay traces asks the LLM to simultaneously identify entities, infer their properties, deduce all interaction rules, and synthesize valid code. This conflates causal reasoning with code synthesis, producing logically inconsistent rules -- for example, generating a collision handler for entities that never interact, or missing a termination condition that the traces clearly demonstrate.

**The two-stage SCM approach** decouples these concerns. Stage 1 (Causal Induction) analyzes gameplay traces to produce a Structural Causal Model: a directed graph where nodes are game entities and their properties (player position, enemy count, score) and edges are causal relationships (collision causes score increment, timeout causes game loss). Stage 2 (Code Synthesis) translates this SCM into executable game rules -- mapping SCM variables to sprite definitions, causal edges to interaction rules, and terminal conditions to win/loss logic. Because the SCM makes causal assumptions explicit and inspectable before code generation, the resulting rules are more faithful to the observed gameplay and contain fewer contradictions.

**Context regimes** control how much information the LLM receives: from raw ASCII frame traces alone (hardest), to traces plus entity descriptions, to traces plus partial rule specifications. More context reduces hallucinated mechanics, but the SCM-based approach shows the largest gains in low-context regimes where the model must reason most about causality.

## Step-by-Step Workflow

1. **Parse gameplay traces into structured observations.** Convert raw input (log files, ASCII frames, state dumps, replay data) into a sequence of timestamped game states. Each state should capture: entity positions on a grid or coordinate system, entity types and properties, the player action taken, and any observable outcomes (score changes, entity spawns/deaths, level transitions).

2. **Narrate the observations in natural language.** Transform the structured states into a textual narrative the LLM can reason about. For each transition, describe what happened: "The player moved RIGHT. The player collided with a diamond at (3,5). The diamond disappeared. Score increased by 1." This narration step bridges raw data and causal reasoning.

3. **Identify all unique entities and their observable properties.** From the narrated traces, enumerate every distinct entity type (player, walls, enemies, collectibles, projectiles, portals) and their behaviors (movement patterns, spawn conditions, persistence). Output a flat entity catalog with type, movement class, and observed property ranges.

4. **Infer the Structural Causal Model (SCM).** Prompt the LLM to construct a causal graph from the narrated observations. For each pair of entities or events that co-occur in the traces, determine: Is there a causal relationship? What is the direction? What is the mechanism? Output the SCM as a list of causal edges in structured format:
   ```
   { "cause": "player collides with enemy", "effect": "player loses life", "type": "deterministic" }
   { "cause": "player collides with key", "effect": "door opens", "type": "deterministic" }
   { "cause": "timer reaches zero", "effect": "game over (loss)", "type": "deterministic" }
   ```

5. **Validate the SCM against the traces.** Walk through each observation and verify that every observed outcome is explained by at least one SCM edge, and no SCM edge predicts an outcome contradicted by the traces. Flag unexplained events (missing edges) and unsupported edges (hallucinated mechanics).

6. **Map SCM nodes to SpriteSet definitions.** For each entity in the SCM, generate a sprite definition with the appropriate physics class: `MovingAvatar` for player-controlled entities, `RandomNPC`/`Chaser` for enemies, `Immovable` for walls and collectibles, `Missile`/`SpawnPoint` for projectiles and spawners.

7. **Map SCM edges to InteractionSet rules.** Translate each causal edge into a collision/interaction rule. Example: `{ cause: "player collides with enemy", effect: "player loses life" }` becomes `player enemy > killSprite scoreChange=-1`. Ensure every interaction is between sprites that can actually collide given their movement models.

8. **Derive TerminationSet from SCM terminal nodes.** Identify which SCM edges lead to game-ending states. Map these to termination conditions: `SpriteCounter stype=goal limit=0 win=True` (all goals collected), `Timeout limit=1000 win=False` (time runs out), `SpriteCounter stype=player limit=0 win=False` (player dies).

9. **Assemble and validate the complete game description.** Combine SpriteSet, InteractionSet, TerminationSet, and LevelMapping into a complete VGDL file (or equivalent target format). Run syntactic validation to ensure all referenced sprite types are defined, all interactions reference valid sprite pairs, and termination conditions reference existing sprites.

10. **Test the generated rules against the original traces.** Simulate the generated game using the same action sequence from the traces and compare the resulting state transitions with the observed ones. Report fidelity metrics: what percentage of state transitions match, which mechanics diverge, and which rules need refinement.

## Concrete Examples

**Example 1: Reverse-engineering a Sokoban-like puzzle game**

User: "I have these gameplay traces from a grid puzzle game. The player pushes boxes onto targets. Can you figure out the game rules?"

```
Trace (abbreviated):
Frame 0: Player at (1,1), Box at (2,1), Box at (3,3), Target at (4,1), Target at (4,3), Walls bordering grid
Frame 1: Action=RIGHT, Player->(2,1), Box->(3,1)  [player pushed box]
Frame 2: Action=RIGHT, Player->(3,1), Box->(4,1)  [box reached target, box stays]
Frame 3: Action=DOWN, Player->(3,2)                [no box interaction]
...
Frame 8: All targets covered -> WIN
```

Approach:
1. Parse traces into state transitions with entity tracking
2. Narrate: "Player moved RIGHT into Box. Box moved RIGHT one cell. Box reached Target and remained."
3. Entity catalog: Player (avatar), Box (pushable), Target (immovable goal), Wall (immovable barrier)
4. Infer SCM:
   ```json
   [
     {"cause": "player moves into box", "effect": "box moves in same direction", "type": "deterministic"},
     {"cause": "box moves into wall", "effect": "box stays, player stays", "type": "deterministic"},
     {"cause": "box moves into box", "effect": "both stay, player stays", "type": "deterministic"},
     {"cause": "all targets covered by boxes", "effect": "game won", "type": "deterministic"}
   ]
   ```
5. Validate: all 8 frames consistent with SCM edges

Output (VGDL):
```vgdl
SpriteSet
  wall   > Immovable
  target > Immovable
  box    > Passive
  player > MovingAvatar

InteractionSet
  box player    > bounceForward
  wall player   > stepBack
  wall box      > undoAll
  box box       > undoAll

TerminationSet
  SpriteCounter stype=box_on_target limit=0 win=True

LevelMapping
  w > wall
  . > target
  b > box
  A > player
```

**Example 2: Inferring mechanics from a shooter game log**

User: "Here's a log from a space shooter. Enemies spawn from the top, player shoots missiles. Help me extract the game rules as a formal specification."

```
Log (abbreviated):
t=0:  Player at (5,9), score=0, lives=3
t=1:  Player fires missile from (5,9), missile spawns at (5,8)
t=2:  Missile at (5,7), Enemy spawns at (2,0)
t=5:  Missile hits Enemy at (5,3) -> enemy destroyed, score +10
t=8:  Enemy at (3,9) reaches Player -> Player hit, lives=2
t=50: Score >= 100 -> WIN
```

Approach:
1. Parse log into state-action-outcome triples
2. Narrate transitions and identify recurring patterns
3. Entity catalog: Player (avatar, horizontal movement), Missile (projectile, moves up), Enemy (NPC, moves down), ScoreCounter, LivesCounter
4. Infer SCM:
   ```json
   [
     {"cause": "player fires", "effect": "missile spawns at player position", "type": "deterministic"},
     {"cause": "missile collides with enemy", "effect": "enemy destroyed, score +10", "type": "deterministic"},
     {"cause": "enemy collides with player", "effect": "lives -1, enemy destroyed", "type": "deterministic"},
     {"cause": "score >= 100", "effect": "game won", "type": "deterministic"},
     {"cause": "lives == 0", "effect": "game lost", "type": "deterministic"},
     {"cause": "periodic timer", "effect": "enemy spawns at random top position", "type": "stochastic"}
   ]
   ```

Output (VGDL):
```vgdl
SpriteSet
  player  > MovingAvatar cooldown=3
  missile > Missile orientation=UP speed=2
  enemy   > RandomNPC orientation=DOWN speed=1
  spawner > SpawnPoint stype=enemy prob=0.1

InteractionSet
  missile enemy  > killBoth scoreChange=10
  enemy player   > killSprite scoreChange=-1
  missile wall   > killSprite

TerminationSet
  MultiSpriteCounter stype1=score limit=100 win=True
  SpriteCounter stype=player limit=0 win=False
```

**Example 3: Causal model from informal gameplay description**

User: "I'm watching someone play a game: there's a character collecting coins in a maze. Ghosts chase the character. Eating a power pellet makes ghosts turn blue and edible for 10 seconds. Help me build a causal model and game rules."

Approach:
1. Treat the description as a narrated observation (skip raw trace parsing)
2. Entity catalog: Player (avatar), Coin (collectible), Ghost (chaser NPC), PowerPellet (collectible with state-change effect), BlueGhost (vulnerable state)
3. Infer SCM with state-dependent edges:
   ```json
   [
     {"cause": "player collides with coin", "effect": "coin collected, score +1", "type": "deterministic"},
     {"cause": "player collides with power pellet", "effect": "all ghosts become blue for 10s", "type": "deterministic"},
     {"cause": "player collides with blue ghost", "effect": "ghost destroyed, score +50", "type": "deterministic"},
     {"cause": "player collides with ghost (normal)", "effect": "player dies", "type": "deterministic"},
     {"cause": "all coins collected", "effect": "game won", "type": "deterministic"},
     {"cause": "player dies with 0 lives", "effect": "game lost", "type": "deterministic"}
   ]
   ```
4. Flag: the 10-second timer on blue ghost state requires a timed transformation mechanic; note this as a stateful interaction that simple collision rules cannot fully capture

Output: SCM graph + VGDL with `TransformTo` and `Timeout` mechanics annotated with implementation notes for the timer-based ghost state transition.

## Best Practices

- **Do:** Always build the SCM as an explicit intermediate artifact before generating code. Skipping this step and going directly from traces to rules produces more logical inconsistencies.
- **Do:** Validate the SCM against every trace before proceeding to code generation. An unvalidated SCM propagates hallucinated mechanics into the final rules.
- **Do:** Narrate raw observations into natural language before causal reasoning. LLMs reason about causality more reliably from textual descriptions than from raw coordinate data.
- **Do:** Use the richest context regime available. If entity descriptions or partial rules are known, include them -- this reduces the hypothesis space for causal inference.
- **Avoid:** Inferring mechanics from a single trace or very short observation window. Multiple traces showing the same mechanic under different conditions are needed to distinguish causal relationships from coincidences.
- **Avoid:** Generating interaction rules for entity pairs that never co-occur in the traces. Every InteractionSet rule should be backed by at least one observed event.

## Error Handling

| Problem | Symptom | Resolution |
|---------|---------|------------|
| Hallucinated mechanics | SCM contains edges with no trace evidence | Re-validate SCM against traces; remove unsupported edges |
| Missing mechanics | Trace events not explained by any SCM edge | Add observation narrations for unexplained transitions; re-prompt for the specific entity pair |
| Invalid VGDL syntax | Parser rejects generated output | Validate sprite names match across SpriteSet, InteractionSet, and TerminationSet; check that physics classes exist |
| Contradictory rules | Same entity pair has conflicting interactions | Review SCM for conditional/state-dependent edges; split into distinct game states |
| Stochastic mechanics misidentified as deterministic | Rule works for some traces but not others | Re-examine traces for variability; model as probabilistic SCM edge and document the distribution |
| Traces too sparse | Not enough observations to distinguish mechanics | Request additional gameplay traces or provide partial game descriptions as supplemental context |

## Limitations

- **Observation-only inference:** The method cannot discover mechanics that never manifest in the provided traces. Hidden mechanics (e.g., a secret room never visited) will be absent from the SCM.
- **Grid-based bias:** VGDL and this workflow are optimized for discrete grid-based games. Continuous-physics games (platformers with momentum, racing games) require adaptation of the sprite physics model.
- **State-dependent mechanics:** Complex stateful interactions (e.g., timed power-ups, inventory systems, multi-phase boss fights) are difficult to express as simple causal edges and may require manual annotation of state variables.
- **Scalability:** Games with many entity types and interaction rules produce large SCMs that are harder to validate. The method works best for arcade-style games with 4-10 distinct entity types.
- **No level layout inference:** This pipeline infers mechanics (rules), not level design. LevelMapping must be provided separately or derived from the trace grid data directly.

## Reference

Jiwatode, M., Dockhorn, A., & Rosenhahn, B. (2026). *From Gameplay Traces to Game Mechanics: Causal Induction with Large Language Models.* arXiv:2602.00190v1. [https://arxiv.org/abs/2602.00190v1](https://arxiv.org/abs/2602.00190v1)

Key takeaway: The two-stage SCM-then-VGDL pipeline outperforms direct generation because it forces explicit causal reasoning before code synthesis, reducing logical inconsistencies by making assumptions inspectable. Code: [https://github.com/jiwatode-mohit/SCM_4_GVGAI](https://github.com/jiwatode-mohit/SCM_4_GVGAI)