---
name: "audiorouter-data-audio-understanding"
description: "Build audio understanding systems that route between internal LLM reasoning and external audio tools using a lightweight RL-trained policy. Use when: 'build an audio analysis pipeline with tool routing', 'create a system that decides when to call ASR vs reason internally', 'implement dual-path audio reasoning', 'design a data-efficient audio understanding agent', 'add smart tool dispatch for audio tasks', 'route audio queries to specialized tools with RL'."
---

# AudioRouter: Data-Efficient Audio Understanding via RL-Based Dual Reasoning

This skill enables Claude to help users build audio understanding systems that intelligently route between internal language model reasoning and external specialized audio tools. Based on the AudioRouter framework, the core idea is: instead of fine-tuning a large model to internalize all audio perception capabilities (expensive, data-hungry), train a lightweight routing policy via reinforcement learning that learns *when* to call external tools (ASR, sound event detection, music analysis) and *when* the base model's internal reasoning suffices. This achieves up to 600x less training data than conventional approaches while matching or exceeding their accuracy.

## When to Use

- When the user wants to build an audio processing pipeline that selectively invokes expensive tools (Whisper, sound classifiers, music IR) only when needed
- When designing an agent system that must decide between calling an external API (ASR, audio tagging) vs. answering from context/reasoning alone
- When the user asks for a data-efficient way to add audio understanding capabilities to an LLM-based application
- When building a multi-tool orchestration system where routing decisions should be learned rather than hardcoded with if/else rules
- When the user wants to reduce inference cost by avoiding unnecessary tool calls for audio queries the model can handle internally
- When implementing any dual-path reasoning system (not just audio) where a frozen base model is paired with specialized external tools

## Key Technique

**The Routing Problem.** Traditional Large Audio Language Models (LALMs) try to internalize all perceptual abilities through massive supervised fine-tuning — feeding the model thousands of labeled audio examples until it learns to transcribe speech, detect sound events, and analyze music internally. This is data-hungry (often requiring hundreds of thousands of examples) and fragile for fine-grained perception. AudioRouter reframes this: instead of teaching the model to *be* every tool, teach it to *use* every tool. The routing decision — "should I call Whisper for this, or can I reason about it directly?" — becomes the learning target.

**GRPO-Trained Routing Policy.** AudioRouter uses Group Relative Policy Optimization (GRPO) to train the routing policy. GRPO compares routing decisions within batches: for a group of audio queries, it tries multiple routing choices (internal vs. external tool X), observes which choices led to correct answers, and adjusts the policy to prefer better routes. The reward signal combines task accuracy (did the final answer improve?) with efficiency (did we avoid unnecessary tool calls?). Critically, the underlying reasoning model stays frozen — only the lightweight router is trained, which is why data requirements drop by orders of magnitude.

**Dual Reasoning Paths.** At inference time, the system operates two paths. The *internal path* passes the audio query directly to the frozen base LLM for reasoning. The *external path* invokes one or more specialized tools — Whisper for ASR, librosa for spectral features, AutoChord for music analysis, or audio event classifiers — then feeds their structured output back to the LLM for final reasoning. The router's job is to pick the path (or combination of paths) that maximizes answer quality for each specific query. Simple questions ("Is someone speaking?") may not need ASR; complex ones ("What are the exact lyrics in this noisy recording?") clearly do.

## Step-by-Step Workflow

1. **Define the audio tool inventory.** List every external audio tool available in your system — ASR engines (Whisper, wav2vec2), sound event detectors (PANNs, BEATs), music analyzers (AutoChord, librosa), audio taggers (Qwen2-Audio). For each tool, document its input format, output schema, latency, and what it's good at.

2. **Design the routing interface.** Create a structured routing decision format. The router should output a JSON object like `{"route": "whisper_asr", "confidence": 0.92}` or `{"route": "internal", "confidence": 0.85}`. Define the action space — one action per tool plus one "internal reasoning" action.

3. **Build the dual-path execution engine.** Implement two code paths: (a) an internal path that formats the audio query + any available metadata as a prompt to the base LLM, and (b) an external path for each tool that calls the tool API, parses the structured output, and injects it into the LLM prompt as additional context for final reasoning.

4. **Collect a small seed dataset.** Gather 500–2000 audio query examples spanning your use cases (speech transcription, sound classification, music QA, audio captioning). Label each with the correct answer — but you do NOT need to label which tool should be used. The RL policy learns that from reward signals.

5. **Implement the GRPO training loop.** For each batch of queries: (a) sample K routing decisions per query from the current policy, (b) execute each routed path to get answers, (c) score answers against ground truth, (d) compute group-relative advantages (how much better was each routing choice vs. the batch average), (e) update the routing policy weights via policy gradient.

6. **Design the reward function.** Use a composite reward: `R = accuracy_score * 1.0 - tool_cost_penalty * 0.1`. The accuracy score is 1.0 for correct answers, 0.0 for incorrect. The tool cost penalty is 0.0 for internal reasoning, and a small value (0.05–0.2) per external tool call. This teaches the router to prefer internal reasoning when it's sufficient.

7. **Freeze the base model, train only the router.** The router should be a lightweight module — a small MLP or a LoRA adapter on the LLM's first few layers. Total trainable parameters should be under 1% of the base model. Train for 3–10 epochs on your seed dataset with GRPO.

8. **Evaluate routing quality.** Beyond task accuracy, measure: (a) tool call rate — what fraction of queries trigger external tools, (b) routing precision — when the router calls a tool, does it actually improve the answer, (c) routing recall — when a tool would help, does the router call it. Target >80% routing precision and >70% routing recall.

9. **Deploy with fallback logic.** In production, add a confidence threshold: if the router's confidence is below 0.6, try both paths and pick the better answer. Log all routing decisions for later analysis and policy refinement.

10. **Iterate with online learning.** Periodically collect new queries where the system got wrong answers, add them to the training set, and re-run GRPO for 1–2 epochs. The data efficiency of the approach means even 50–100 new examples can meaningfully improve routing.

## Concrete Examples

**Example 1: Building an Audio QA System with Smart Tool Routing**

User: "I have an audio QA application that uses Whisper for transcription and a sound event detector. Right now it calls both tools on every query, which is slow and expensive. Help me build a router that only calls tools when needed."

Approach:
1. Define the tool inventory: `whisper_asr` (latency: 2s, good at speech), `sound_detector` (latency: 1s, good at environmental sounds), `internal` (latency: 0.1s, good at reasoning from context)
2. Build the routing policy as a small classifier on top of audio embeddings
3. Train with GRPO on existing QA pairs

Output (router architecture):
```python
import torch
import torch.nn as nn

class AudioRouter(nn.Module):
    """Lightweight routing policy for audio tool selection."""

    def __init__(self, audio_embed_dim=768, num_tools=3):
        super().__init__()
        # Small MLP router — under 100K parameters
        self.router = nn.Sequential(
            nn.Linear(audio_embed_dim, 256),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(256, 64),
            nn.ReLU(),
            nn.Linear(64, num_tools),  # [internal, whisper, sound_detector]
        )
        self.tool_names = ["internal", "whisper_asr", "sound_detector"]

    def forward(self, audio_embedding):
        logits = self.router(audio_embedding)
        probs = torch.softmax(logits, dim=-1)
        return probs  # shape: (batch, num_tools)

    def route(self, audio_embedding, threshold=0.6):
        probs = self.forward(audio_embedding)
        best_tool_idx = probs.argmax(dim=-1).item()
        confidence = probs[0, best_tool_idx].item()
        return {
            "route": self.tool_names[best_tool_idx],
            "confidence": confidence,
            "all_probs": {
                name: probs[0, i].item()
                for i, name in enumerate(self.tool_names)
            },
            "use_fallback": confidence < threshold,
        }
```

Output (GRPO training loop):
```python
def grpo_train_step(router, queries, ground_truth, tools, base_llm, K=4):
    """One GRPO training step over a batch of audio queries."""
    all_rewards = []
    all_log_probs = []

    for query, answer in zip(queries, ground_truth):
        audio_emb = extract_embedding(query["audio"])
        probs = router(audio_emb)

        # Sample K routing decisions for this query
        dist = torch.distributions.Categorical(probs)
        sampled_routes = [dist.sample() for _ in range(K)]
        log_probs = [dist.log_prob(r) for r in sampled_routes]

        # Execute each route and score
        rewards = []
        for route_idx in sampled_routes:
            tool_name = router.tool_names[route_idx.item()]
            if tool_name == "internal":
                result = base_llm.reason(query)
                cost_penalty = 0.0
            else:
                tool_output = tools[tool_name].run(query["audio"])
                result = base_llm.reason_with_context(query, tool_output)
                cost_penalty = 0.1
            accuracy = score_answer(result, answer)
            rewards.append(accuracy - cost_penalty)

        # Group-relative advantage: how much better than batch average
        mean_reward = sum(rewards) / len(rewards)
        advantages = [r - mean_reward for r in rewards]

        all_rewards.extend(advantages)
        all_log_probs.extend(log_probs)

    # Policy gradient update
    loss = -sum(lp * adv for lp, adv in zip(all_log_probs, all_rewards))
    loss = loss / len(all_log_probs)
    loss.backward()
    return loss.item()
```

**Example 2: Adding Routing to an Existing Audio Agent**

User: "I have a LangChain agent that handles audio queries. It always calls all tools sequentially. Help me add an AudioRouter-style policy so it learns which tools to skip."

Approach:
1. Wrap existing tools as callable functions with standardized I/O
2. Add a routing layer before the tool dispatch
3. Collect traces of past queries to bootstrap GRPO training

Output (integration pattern):
```python
from langchain.agents import Tool

class RoutedAudioAgent:
    def __init__(self, llm, tools: list[Tool], router: AudioRouter):
        self.llm = llm
        self.tools = {t.name: t for t in tools}
        self.router = router
        self.decision_log = []  # For later GRPO refinement

    def handle_query(self, audio_path: str, question: str):
        # Extract audio embedding for routing decision
        embedding = self.extract_embedding(audio_path)
        decision = self.router.route(embedding)

        self.decision_log.append({
            "audio": audio_path,
            "question": question,
            "decision": decision,
        })

        if decision["use_fallback"]:
            # Low confidence — run both paths, pick best
            internal = self.llm.invoke(question)
            tool_result = self.tools[decision["route"]].run(audio_path)
            augmented = self.llm.invoke(f"{question}\nTool output: {tool_result}")
            return self._pick_better(internal, augmented, question)

        if decision["route"] == "internal":
            return self.llm.invoke(question)

        # Route to specific tool, then reason
        tool_output = self.tools[decision["route"]].run(audio_path)
        return self.llm.invoke(
            f"{question}\nAnalysis from {decision['route']}: {tool_output}"
        )
```

**Example 3: Data-Efficient Training from Minimal Labels**

User: "I only have 200 labeled audio QA pairs. Is that enough to train a routing policy?"

Approach:
1. Yes — AudioRouter's key insight is that routing requires far less data than perception. With 200 examples and K=8 samples per query in GRPO, you get 1600 routing signal data points.
2. Start with a simple heuristic router as initialization, then refine with GRPO.

Output (bootstrapping from heuristics):
```python
def bootstrap_router_from_heuristics(audio_path, question):
    """Rule-based router to initialize before GRPO training."""
    q = question.lower()
    # Speech-related queries likely need ASR
    if any(kw in q for kw in ["say", "lyrics", "spoken", "transcript", "words"]):
        return "whisper_asr"
    # Sound identification queries need event detection
    if any(kw in q for kw in ["sound", "noise", "hear", "what is", "identify"]):
        return "sound_detector"
    # Music queries need music IR
    if any(kw in q for kw in ["chord", "key", "tempo", "genre", "music"]):
        return "music_analyzer"
    return "internal"

# Use heuristic labels to warm-start the router MLP,
# then switch to GRPO after 2 epochs of supervised pre-training.
# Even 200 examples with GRPO K=8 yields strong routing in 5 epochs.
```

## Best Practices

- **Do:** Keep the routing policy small (under 1M parameters). The whole point is that routing is a simpler problem than perception — a 3-layer MLP works well.
- **Do:** Log every routing decision in production. This gives you free training data for continuous GRPO refinement without additional labeling.
- **Do:** Include an "internal reasoning" option in the action space. Many audio questions ("Is this clip longer than 10 seconds?") don't need any tool at all.
- **Do:** Use the confidence threshold fallback (try both paths when uncertain). This costs more at inference but prevents routing errors from silently degrading quality.
- **Avoid:** Training the base LLM jointly with the router. Freezing the base model is essential — it prevents catastrophic forgetting and keeps the training data requirement low.
- **Avoid:** Using a uniform tool cost penalty. Different tools have different latencies and costs; weight the penalty proportionally (e.g., Whisper costs more than a simple audio tagger).
- **Avoid:** Hardcoding routing rules long-term. Heuristics are fine for bootstrapping, but learned routing consistently outperforms static rules as query diversity grows.

## Error Handling

- **Router always picks one tool:** If the router over-indexes on a single tool (>90% of queries route to it), the GRPO reward signal is likely too sparse. Increase K (samples per query) or add an entropy bonus to the policy loss to encourage exploration.
- **Tool failures at inference:** Wrap every external tool call in a try/except. On tool failure, fall back to internal reasoning and log the failure. Never let a tool timeout crash the pipeline.
- **Reward hacking:** If the router learns to always pick "internal" to avoid cost penalties, your cost penalty is too high. Reduce it or remove it temporarily until routing accuracy stabilizes.
- **Distribution shift:** If production queries differ significantly from training data, routing accuracy will degrade. Monitor routing precision weekly and retrain when it drops below 75%.

## Limitations

- The routing policy is only as good as the tool inventory. If no available tool handles a query type well, routing cannot help — you need to add a new tool.
- AudioRouter assumes tools return structured, parseable output. Tools with unreliable or highly variable output formats will confuse the downstream LLM reasoning step.
- The framework is designed for classification-style routing (pick one path). For queries that genuinely need output from multiple tools combined, you need a more complex multi-tool orchestration layer on top.
- GRPO training requires that you can programmatically score answer correctness. For open-ended audio captioning or subjective quality tasks, designing the reward function is non-trivial.
- This approach optimizes *which* tool to call, not *how* to call it. Tool parameters (e.g., Whisper language hint, VAD sensitivity) must still be configured separately.

## Reference

- **Paper:** [AudioRouter: Data Efficient Audio Understanding via RL based Dual Reasoning](https://arxiv.org/abs/2602.10439v1) — Focus on Section 3 (GRPO routing policy formulation) and Section 4 (ablation showing 600x data efficiency vs. supervised tool-use training).