---
name: "openguandan-large-scale-imperfect-information"
description: "Build AI agents for the OpenGuanDan imperfect-information card game benchmark. Covers WebSocket client implementation, game state parsing, action selection, and agent strategy design for the GuanDan four-player cooperative-competitive card game. Trigger phrases: 'build a GuanDan agent', 'OpenGuanDan bot', 'imperfect information game AI', 'card game agent', 'GuanDan WebSocket client', 'multi-agent card game'."
---

# OpenGuanDan: Building AI Agents for Large-Scale Imperfect Information Games

This skill enables Claude to build fully functional AI agents that connect to the OpenGuanDan game server and play GuanDan -- a four-player, two-team Chinese card game with imperfect information, variable action spaces, and mixed cooperative-competitive objectives. The OpenGuanDan benchmark (Li et al., 2026) provides a WebSocket-based game server with a per-player API, letting you implement rule-based agents, reinforcement learning agents, or LLM-powered agents in any language that supports WebSocket connections.

## When to Use

- When the user asks to build a bot or AI agent for the GuanDan card game
- When the user wants to connect a Python/JS/Java client to the OpenGuanDan game server via WebSocket
- When the user needs to implement decision-making logic for a four-player imperfect-information card game
- When the user is designing a multi-agent system where agents must cooperate within teams and compete against opponents
- When the user wants to evaluate or benchmark different game-playing strategies (rule-based, RL, LLM) against each other
- When the user asks about handling large action spaces with variable-length legal move lists in card games
- When the user wants to integrate an LLM as a game-playing agent through a structured API

## Key Technique

**The OpenGuanDan Architecture.** OpenGuanDan is a server-client system where a Java-based game server (JDK 17+) manages all game logic, card dealing, rule enforcement, and scoring. Each of the four player seats connects independently via WebSocket (`ws://127.0.0.1:8181`). The server sends JSON messages describing the current game state and a list of legal actions; the client responds by selecting an action from that list. This "action-list constraint" is critical: agents never construct moves from scratch but always choose from the server-provided `actionList`, which guarantees legal play and simplifies agent development.

**Game Structure and Challenges.** GuanDan uses two standard 54-card decks (108 cards total) dealt to four players in fixed team pairs (seats 0+2 vs. seats 1+3). A full match spans multiple rounds with a tribute/return-tribute phase between rounds (where losing players give cards to winners). The action space includes singles, pairs, triples, straights, bombs, and other combinations -- often yielding hundreds of legal moves per turn. The game is imperfect-information because players cannot see opponents' hands, and it is long-horizon because a match may last dozens of rounds with strategic carry-over between rounds (rank progression from 2 to A). Agents must balance cooperation with their teammate against competition with opponents.

**Agent Design Patterns.** The benchmark supports three agent architectures: (1) **Rule-based agents** that use hand-crafted heuristics (e.g., play smallest valid combo, save bombs for critical moments), (2) **Learning-based agents** using DQN or PPO trained on self-play within the simulator, and (3) **LLM agents** that receive the game state as a structured prompt and return an action index. The independent per-player API means you can mix agent types in the same game -- for example, pit an LLM agent and a rule-based agent as teammates against two RL agents.

## Step-by-Step Workflow

### 1. Set up the OpenGuanDan game server

Download the server from `https://github.com/GameAI-NJUPT/OpenGuanDan`. Run with Java 17+:

```bash
java -jar guandan-java-1.0.0.jar
# Or for multi-threaded: java -Dguandan.cluster.mode=true -Dguandan.cluster.workers=4 -jar guandan-java-1.0.0.jar
```

The server listens on port 8181 (WebSocket) and 3000 (HTTP dashboard).

### 2. Implement the WebSocket client skeleton

Create a client that connects to `ws://127.0.0.1:8181` and handles JSON messages. Every message has a `type` field. The client must handle room management (`CREATE_ROOM`, `JOIN_ROOM`) and three game phases: play, tribute, and return-tribute.

### 3. Implement room creation and joining logic

Send a `CREATE_ROOM` message with a `userId` and desired `round` count. Then connect three more clients that send `JOIN_ROOM` with the returned `roomId`. The game auto-starts when four players are connected.

```json
{"type": "CREATE_ROOM", "data": {"userId": "agent-1", "round": 10}}
{"type": "JOIN_ROOM", "data": {"userId": "agent-2", "roomId": "<roomId>"}}
```

### 4. Parse the game state from server notifications

Listen for notification messages with `stage` field values: `beginning` (deal), `play` (turn notifications), `tribute`, `anti-tribute`, `back` (return tribute), `episodeOver`, and `gameResult`. Extract your hand cards, public information (other players' remaining card counts), current ranks, and the action history.

### 5. Handle action requests by selecting from the actionList

When the server sends an action request (`"act"` type with `"stage": "play"`), it includes:
- `handCards`: your current hand as two-character card codes (e.g., `"S2"` = Spade 2, `"HA"` = Heart Ace)
- `publicInfo`: array of remaining card counts per seat
- `actionList`: array of legal actions, each as `[Pattern, Rank, CardList]`
- `indexRange`: valid index bounds into the actionList

Respond with a `PLAY` message containing the selected action tuple.

### 6. Implement the tribute and return-tribute phases

Between rounds, handle `TRIBUTE` requests (select which card to give up from provided options) and `PAYTRIBUTE` requests (select which card to return). These use the same pattern: choose from the server's `actionList`. Match `tributePos` and `tribute` values exactly from the request.

### 7. Design the action selection strategy

This is where the agent intelligence lives. Choose one approach:

- **Rule-based**: Score each legal action using heuristics (prefer passing when partner leads, save high-value combos, play bombs strategically)
- **RL-based**: Encode the game state as a feature vector (hand composition, cards played, remaining counts, rank info) and train a policy network (DQN/PPO) via self-play
- **LLM-based**: Format the game state and action list as a structured prompt, ask the LLM to reason about strategy, and parse the returned action index

### 8. Encode game state features for learning agents

Represent the state as: (a) one-hot encoding of cards in hand (108 positions), (b) one-hot encoding of cards played by each player, (c) remaining card counts per opponent, (d) current team ranks and opponent ranks, (e) last action played and by whom. This gives a fixed-size observation vector suitable for neural network input.

### 9. Run evaluation tournaments

Pit agents against each other in round-robin tournaments. Track win rates per team configuration and compute Elo ratings. Use the multi-threaded cluster mode for parallel games:

```bash
java -Dguandan.cluster.mode=true -Dguandan.cluster.workers=8 -jar guandan-java-1.0.0.jar
```

### 10. Iterate on strategy using game logs

Parse `episodeOver` and `gameResult` messages to collect training data. For RL agents, use the reward signal from round outcomes. For rule-based agents, analyze losing patterns to refine heuristics.

## Concrete Examples

**Example 1: Python WebSocket Agent (Rule-Based)**

User: "Build a simple Python agent that connects to OpenGuanDan and plays using basic heuristics."

Approach:
1. Use the `websockets` library to connect to `ws://127.0.0.1:8181`
2. Implement message routing based on `type` and `stage` fields
3. For action selection, choose the smallest valid combo (lowest index in actionList that isn't a pass, unless strategically passing)

Output:
```python
import asyncio
import json
import websockets

class GuanDanAgent:
    def __init__(self, user_id, room_id=None):
        self.user_id = user_id
        self.room_id = room_id
        self.hand = []
        self.my_pos = -1

    async def connect(self, uri="ws://127.0.0.1:8181"):
        async with websockets.connect(uri) as ws:
            self.ws = ws
            if self.room_id is None:
                await self.create_room()
            else:
                await self.join_room()
            await self.listen()

    async def create_room(self):
        msg = {"type": "CREATE_ROOM", "data": {"userId": self.user_id, "round": 10}}
        await self.ws.send(json.dumps(msg))
        resp = json.loads(await self.ws.recv())
        self.room_id = resp["data"]["roomId"]
        print(f"Created room: {self.room_id}")

    async def join_room(self):
        msg = {"type": "JOIN_ROOM", "data": {"userId": self.user_id, "roomId": self.room_id}}
        await self.ws.send(json.dumps(msg))

    async def listen(self):
        async for raw in self.ws:
            msg = json.loads(raw)
            msg_type = msg.get("type", "")
            data = msg.get("data", {})

            if data.get("stage") == "beginning":
                self.hand = data["handCards"]
                self.my_pos = data["myPos"]
            elif msg_type == "act" and data.get("stage") == "play":
                await self.handle_play(data)
            elif msg_type == "act" and data.get("stage") == "tribute":
                await self.handle_tribute(data)
            elif msg_type == "act" and data.get("stage") == "back":
                await self.handle_return_tribute(data)
            elif data.get("stage") == "episodeOver":
                self.hand = []  # Reset for next round
            elif data.get("stage") == "gameResult":
                print(f"Game over. Winner: team {data['victory']}")

    async def handle_play(self, data):
        action_list = data["actionList"]
        self.hand = data["handCards"]
        # Heuristic: play the first non-pass action (smallest combo)
        # Pass is typically the last action in the list
        chosen = self.select_action(action_list)
        resp = {
            "type": "PLAY",
            "data": {
                "roomId": self.room_id,
                "seatNum": self.my_pos,
                "act": action_list[chosen]
            }
        }
        await self.ws.send(json.dumps(resp))

    def select_action(self, action_list):
        """Rule-based: prefer singles/pairs first, save bombs."""
        for i, action in enumerate(action_list):
            pattern = action[0]
            if pattern == "Pass":
                continue
            if pattern in ("Single", "Pair"):
                return i
        # If no simple plays, just pick the first legal action
        return 0

    async def handle_tribute(self, data):
        # Give the first available tribute option
        resp = {
            "type": "TRIBUTE",
            "data": {
                "roomId": self.room_id,
                "seatNum": self.my_pos,
                "act": data["actionList"][0]
            }
        }
        await self.ws.send(json.dumps(resp))

    async def handle_return_tribute(self, data):
        resp = {
            "type": "PAYTRIBUTE",
            "data": {
                "roomId": self.room_id,
                "seatNum": self.my_pos,
                "tributePos": data["tributePos"],
                "tribute": data["tribute"],
                "act": data["actionList"][0]
            }
        }
        await self.ws.send(json.dumps(resp))

# Launch four agents in one process
async def main():
    agent0 = GuanDanAgent("player-0")
    await agent0.connect()
    room = agent0.room_id
    tasks = [
        GuanDanAgent("player-1", room).connect(),
        GuanDanAgent("player-2", room).connect(),
        GuanDanAgent("player-3", room).connect(),
    ]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

**Example 2: LLM-Powered Agent**

User: "I want to use an LLM to decide which card to play in GuanDan."

Approach:
1. Reuse the WebSocket client skeleton from Example 1
2. In `select_action`, format the game state and action list into a structured prompt
3. Call the LLM API and parse the returned action index

Output:
```python
import openai

def select_action_llm(self, data):
    hand = data["handCards"]
    actions = data["actionList"]
    public = data.get("publicInfo", [])

    prompt = f"""You are playing GuanDan, a Chinese card game. You are seat {self.my_pos}.
Your team: seats {self.my_pos} and {(self.my_pos + 2) % 4}.

Your hand: {hand}
Opponents' remaining cards: {public}
Current rank: {data.get('curRank', '2')}

Legal actions (index: [Pattern, Rank, Cards]):
"""
    for i, act in enumerate(actions):
        prompt += f"  {i}: {act}\n"

    prompt += "\nChoose the best action index. Consider: save bombs for critical moments, "
    prompt += "support your teammate, play low cards early. Reply with ONLY the index number."

    response = openai.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=10,
    )
    try:
        idx = int(response.choices[0].message.content.strip())
        if 0 <= idx < len(actions):
            return idx
    except ValueError:
        pass
    return 0  # Fallback to first action
```

**Example 3: Evaluation Harness**

User: "Set up a tournament to compare my rule-based agent against a random agent over 100 games."

Approach:
1. Launch the server in cluster mode for parallelism
2. Run multiple game instances, pairing agents by team
3. Collect results and compute win rates

Output:
```python
import asyncio
import random
from collections import Counter

class RandomAgent(GuanDanAgent):
    def select_action(self, action_list):
        return random.randint(0, len(action_list) - 1)

class SmartAgent(GuanDanAgent):
    def select_action(self, action_list):
        # Your improved heuristic here
        for i, act in enumerate(action_list):
            if act[0] == "Bomb":
                continue  # Save bombs
            if act[0] != "Pass":
                return i
        return len(action_list) - 1  # Pass

async def run_tournament(n_games=100):
    results = Counter()
    for game_id in range(n_games):
        # Team 0 (seats 0,2): SmartAgent; Team 1 (seats 1,3): RandomAgent
        creator = SmartAgent(f"smart-{game_id}-0")
        await creator.connect()
        room = creator.room_id
        agents = [
            RandomAgent(f"random-{game_id}-1", room),
            SmartAgent(f"smart-{game_id}-2", room),
            RandomAgent(f"random-{game_id}-3", room),
        ]
        game_results = await asyncio.gather(
            creator.play_full_game(),
            *[a.play_full_game() for a in agents]
        )
        winner = game_results[0]  # 0 or 1
        results[winner] += 1
    print(f"Smart win rate: {results[0]/n_games:.1%}")
    print(f"Random win rate: {results[1]/n_games:.1%}")
```

## Best Practices

**Do:**
- Always select actions exclusively from the server-provided `actionList` -- never construct card combinations manually
- Handle all three game phases (play, tribute, return-tribute) even if your strategy focus is on play; the tribute phase affects hand composition significantly
- Track cards that have been played by all players to narrow down opponent hand estimates (card counting)
- Test agents against multiple opponent types (random, rule-based, RL) to avoid overfitting to one playstyle
- Use the `publicInfo` field (remaining card counts per seat) as a key signal -- knowing an opponent has 2 cards left changes strategy entirely

**Avoid:**
- Sending malformed action tuples -- the server rejects anything not in the provided actionList, which may cause a disconnect
- Ignoring the team structure -- seats 0+2 and 1+3 are fixed teams; agents that don't cooperate with their teammate perform poorly
- Training RL agents on too few games -- the 108-card deck and four-player dynamics require millions of episodes for convergence
- Hardcoding card values without accounting for the dynamic trump rank (the "current rank" card acts as a wild card in certain combinations)

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| WebSocket connection refused | Server not running or wrong port | Verify server is running; check port 8181 is available |
| Action rejected / disconnect | Sent an action not in `actionList` | Always index into the provided actionList; never fabricate actions |
| Room doesn't start | Fewer than 4 players joined | Ensure all 4 WebSocket clients send `JOIN_ROOM` with correct `roomId` |
| `tributePos`/`tribute` mismatch | Values don't match the server request | Copy `tributePos` and `tribute` exactly from the incoming request data |
| Timeout on action response | Agent took too long to respond | Add timeout handling; LLM agents need fast inference or a fallback |
| JSON parse error | Malformed message encoding | Ensure UTF-8 encoding and valid JSON serialization |

## Limitations

- **Closed game engine**: The server is a compiled Java binary -- you cannot modify game rules or add custom card types without forking the source
- **Local only**: The WebSocket server binds to localhost; multi-machine setups require tunneling or a proxy
- **No built-in replay**: The server does not persist game logs; you must implement your own logging in the client
- **RL training speed**: The Java server communication overhead makes pure RL training slower than a native Python simulator; consider using the server for evaluation only and a fast Python re-implementation of the rules for training
- **LLM latency**: Real-time play requires sub-second responses; large LLMs may need action caching or a distilled policy for practical play
- **Chinese documentation**: The primary README and some message field names are in Chinese; this skill provides the English equivalents for all key fields

## Reference

**Paper**: Li, C., Yang, S., Zhan, C., Ge, Z., & Hu, Y. (2026). *OpenGuanDan: A Large-Scale Imperfect Information Game Benchmark*. arXiv:2602.00676v1. [https://arxiv.org/abs/2602.00676v1](https://arxiv.org/abs/2602.00676v1)

Look for: Section 3 (benchmark design and API specification), Section 4 (agent implementations including DQN/PPO architectures), and Section 5 (evaluation methodology and Elo rating computation). The appendix contains the full GuanDan rule set and card encoding table.

**Code**: [https://github.com/GameAI-NJUPT/OpenGuanDan](https://github.com/GameAI-NJUPT/OpenGuanDan)