---
name: "spotagent-grounding-visual-geo-localization"
description: "Build agentic geo-localization systems that combine vision-language model reasoning with tool-assisted verification using the SpotAgent ReAct framework. Triggers: 'geo-localize images', 'build a visual geo-localization agent', 'verify image location with external tools', 'ReAct loop for geo-localization', 'agentic visual grounding pipeline', 'tool-augmented location prediction'"
---

# SpotAgent: Agentic Visual Geo-Localization with Tool-Grounded Reasoning

This skill enables Claude to help users build agentic geo-localization systems that go beyond naive coordinate prediction. Based on the SpotAgent framework, the approach formalizes geo-localization as a Partially Observable Markov Decision Process (POMDP) where a vision-language model iteratively reasons about visual cues, invokes external tools (geocoding, web search, image zoom), and verifies hypotheses before committing to a location. The key insight is that confident-but-wrong predictions are replaced by a structured Thought-Action-Observe loop that grounds every claim in verifiable external evidence.

## When to Use

- When the user wants to build an agent that can determine geographic location from images using reasoning and external tool calls
- When implementing a ReAct-style reasoning loop that combines visual interpretation with geocoding APIs, web search, or map lookups
- When designing a multi-agent data synthesis pipeline to generate high-quality tool-calling trajectories for training
- When training a vision-language model with a 3-stage pipeline (SFT, agentic cold start, reinforcement learning) for any tool-augmented visual task
- When applying spatially-aware curriculum learning to filter training samples by difficulty for RL optimization
- When the user needs to reduce hallucinations in location or spatial reasoning by grounding predictions with external verification

## Key Technique

SpotAgent structures geo-localization as a **ReAct reasoning loop** with a budget of up to 6 tool invocations per image. The agent cycles through three phases: (1) **Thought** -- analyze visual evidence (architecture style, vegetation, road markings, text, terrain) and identify information gaps; (2) **Action** -- invoke one of three tools: `GeoCoding(address)` to convert place names to coordinates, `WebSearch(query)` to verify landmarks or decode text, or `ImageTool(bbox)` to zoom into discriminative image regions; (3) **Observe** -- consume tool output and integrate it into the reasoning chain. The loop terminates when the agent has enough evidence to commit to a coordinate prediction. Empirically, 59% of successful trajectories use only GeoCoding, while 41% combine zoom + web search, showing the agent learns selective tool deployment.

The training pipeline has three stages. **Stage 1 (SFT)** aligns the base model on ~200K image-coordinate pairs as visual QA. **Stage 2 (Agentic Cold Start)** is the critical innovation: a multi-agent framework where an Observer agent (VLM) generates hierarchical visual descriptions and a Tool-Call agent (LLM) orchestrates tool invocations. Trajectories are rejection-sampled -- only those with valid tool calls AND final predictions within 5km of ground truth are kept (~6K from ~20K raw). This stage activates tool-use awareness; without it, models show near-zero tool-call rates. **Stage 3 (RL with GRPO)** refines reasoning using a piecewise reward based on geodesic distance (1.0 at <1km, linearly decaying to 0 at 200km). A Spatially-Aware Dynamic Filtering strategy removes both trivially solved (Pass@K=K) and intractable (Pass@K=0) samples, training only on the "learnable" frontier.

This architecture generalizes beyond geo-localization to any task where a VLM must invoke external tools to verify visual hypotheses rather than hallucinate answers.

## Step-by-Step Workflow

1. **Define the tool interface** using a standardized schema (MCP-style). Implement three tools: `GeoCoding(address: str) -> {lat, lng}` wrapping a geocoding API (Google Maps, Nominatim), `WebSearch(query: str) -> str` wrapping a search API (Tavily, SerpAPI, Brave), and `ImageTool(bbox: [x1, y1, x2, y2]) -> Image` that crops and upscales a region of the input image for re-examination.

2. **Design the structured prompt template** with XML-style tags enforcing the reasoning format: `<think>` for chain-of-thought reasoning, `<tool_call>` for JSON-formatted tool invocations, `<tool_response>` for tool outputs, and `<answer>` for the final coordinate prediction. Require a thought step before every tool call.

3. **Build the ReAct execution loop** that parses model output, detects tool-call tags, routes to the appropriate tool handler, injects the response back into the conversation, and loops until the model emits an `<answer>` tag or hits the 6-call budget. Track the interaction history as the POMDP state.

4. **Curate the SFT dataset** (~200K samples) by pairing geotagged images with coordinate labels formatted as visual QA. Fine-tune the base VLM (e.g., Qwen2.5-VL-7B) on this data for foundational geo-knowledge alignment.

5. **Synthesize agentic trajectories** using a two-agent framework: an Observer VLM that generates hierarchical visual descriptions (macro: architecture/terrain to micro: text/signage) and a Tool-Call LLM that plans verification strategies. Run this pipeline on a held-out image set to produce raw reasoning traces.

6. **Apply rejection sampling** to the raw trajectories: keep only those where (a) every tool call parsed and executed successfully and (b) the final prediction is within 5km of ground truth. Expect ~30% acceptance rate, yielding ~6K high-quality trajectories from ~20K raw.

7. **Fine-tune on agentic trajectories** (Agentic Cold Start) by training on the accepted traces with loss computed only on assistant tokens -- mask user prompts and tool response tokens. This stage instills tool-calling syntax and strategic tool selection.

8. **Run RL with spatially-aware filtering**: Generate 8 rollouts per image using the cold-start model. Compute Pass@K at multiple distance thresholds (1km, 25km, 200km, 750km). Discard trivial (Pass@K=K) and intractable (Pass@K=0) samples. Train with GRPO on the learnable set using the piecewise distance reward.

9. **Implement curriculum phases for RL**: Phase I uses strict thresholds (1km, 25km) to refine precision on easier samples. Phase II loosens to (200km, 750km) to incorporate harder coarse-localization challenges.

10. **Evaluate with hierarchical distance metrics**: Report accuracy at 1km, 25km, 200km, 750km, and 2500km thresholds using geodesic (great-circle) distance. Compare tool-assisted vs. tool-free modes to quantify the grounding benefit.

## Concrete Examples

**Example 1: Building the ReAct Geo-Localization Agent**

User: "I want to build a Python agent that takes an image and predicts its GPS coordinates using reasoning and external tools."

Approach:
1. Define tool schemas with typed inputs/outputs:
```python
TOOLS = [
    {
        "name": "GeoCoding",
        "description": "Convert a place name or address to GPS coordinates",
        "parameters": {"address": {"type": "string"}},
    },
    {
        "name": "WebSearch",
        "description": "Search the web to verify a landmark, sign text, or visual clue",
        "parameters": {"query": {"type": "string"}},
    },
    {
        "name": "ImageTool",
        "description": "Zoom into a bounding box region of the query image",
        "parameters": {"bbox": {"type": "array", "items": {"type": "number"}}},
    },
]
```

2. Build the ReAct loop with budget enforcement:
```python
def geolocalize(image_path: str, vlm_client, max_tool_calls: int = 6) -> dict:
    history = [{"role": "user", "content": [
        {"type": "image", "path": image_path},
        {"type": "text", "text": SYSTEM_PROMPT},
    ]}]
    tool_calls_used = 0

    while tool_calls_used < max_tool_calls:
        response = vlm_client.generate(history, tools=TOOLS)

        if "<answer>" in response.text:
            return parse_coordinates(response.text)

        if "<tool_call>" in response.text:
            tool_name, params = parse_tool_call(response.text)
            result = execute_tool(tool_name, params, image_path)
            history.append({"role": "assistant", "content": response.text})
            history.append({"role": "tool", "content": f"<tool_response>{result}</tool_response>"})
            tool_calls_used += 1
        else:
            history.append({"role": "assistant", "content": response.text})

    return parse_coordinates(force_answer(vlm_client, history))
```

3. The system prompt enforces structured reasoning:
```
You are a geo-localization expert. Analyze the image and determine its GPS coordinates.
You MUST wrap reasoning in <think>...</think> tags before every tool call.
You MUST wrap tool calls in <tool_call>{"name": "...", "arguments": {...}}</tool_call>.
You have a budget of 6 tool calls. When confident, output <answer>{"lat": ..., "lng": ...}</answer>.
```

Output: An agent that reasons step-by-step -- e.g., "I see Cyrillic text on a storefront sign. Let me zoom in to read it... The sign says 'Apteka'. Let me web search 'Apteka pharmacy chain locations'... Results suggest Eastern Europe. The architecture resembles Soviet-era buildings. Let me geocode 'central Minsk Belarus'..." -- producing verified coordinates.

---

**Example 2: Multi-Agent Trajectory Synthesis for Training Data**

User: "I need to generate training data for a tool-calling geo-localization model. How do I synthesize high-quality reasoning trajectories?"

Approach:
1. Set up the two-agent pipeline:
```python
def synthesize_trajectory(image_path: str, ground_truth: tuple) -> dict | None:
    # Observer: hierarchical visual description
    observation = observer_vlm.describe(image_path, prompt=(
        "Describe this image for geo-localization at three levels: "
        "1) Macro: landscape, climate zone, architecture style, road type "
        "2) Meso: specific landmarks, infrastructure, vehicle types "
        "3) Micro: text on signs, license plates, specific brand logos"
    ))

    # Tool-Call Agent: plan and execute verification
    trajectory = tool_agent_llm.reason(
        context=observation,
        tools=TOOLS,
        image_path=image_path,
        max_steps=6,
    )

    # Rejection sampling
    if not trajectory.all_tool_calls_valid():
        return None
    pred = trajectory.final_prediction()
    if geodesic_distance(pred, ground_truth) > 5.0:  # km
        return None

    return trajectory.to_training_format()
```

2. Run at scale with parallel processing and expect ~30% acceptance:
```python
accepted = []
for image, gt in tqdm(dataset):
    traj = synthesize_trajectory(image, gt)
    if traj:
        accepted.append(traj)
# ~6K accepted from ~20K attempts
```

3. Format for training with selective masking (loss only on assistant turns):
```python
def format_for_training(trajectory):
    tokens, loss_mask = [], []
    for turn in trajectory:
        toks = tokenize(turn["content"])
        tokens.extend(toks)
        # Only compute loss on assistant reasoning + tool calls
        mask_val = 1.0 if turn["role"] == "assistant" else 0.0
        loss_mask.extend([mask_val] * len(toks))
    return {"input_ids": tokens, "loss_mask": loss_mask}
```

Output: A dataset of ~6K validated trajectories where each sample is a complete reasoning chain with interleaved thoughts, tool calls, tool responses, and a correct final answer within 5km of ground truth.

---

**Example 3: Spatially-Aware RL Filtering**

User: "My RL training is inefficient -- the model wastes compute on images it already solves or can never solve. How do I filter to the learnable frontier?"

Approach:
1. Run Pass@K evaluation with the cold-start model:
```python
def compute_pass_at_k(model, dataset, k=8, thresholds=[1, 25, 200, 750]):
    results = {}
    for image, gt in dataset:
        preds = [model.predict(image) for _ in range(k)]
        distances = [geodesic_distance(p, gt) for p in preds]
        results[image.id] = {
            f"pass@{k}_at_{t}km": sum(1 for d in distances if d < t)
            for t in thresholds
        }
    return results
```

2. Filter to the learnable set per threshold:
```python
def filter_learnable(pass_k_results, k=8):
    learnable = set()
    for img_id, scores in pass_k_results.items():
        for threshold_key, pass_count in scores.items():
            if 0 < pass_count < k:  # Neither trivial nor intractable
                learnable.add(img_id)
                break
    return learnable
```

3. Apply curriculum phases:
```python
# Phase I: strict thresholds for precision
phase1_set = filter_by_thresholds(learnable, thresholds=[1, 25])
train_grpo(model, phase1_set, reward_fn=piecewise_distance_reward)

# Phase II: relaxed thresholds for harder samples
phase2_set = filter_by_thresholds(learnable, thresholds=[200, 750])
train_grpo(model, phase2_set, reward_fn=piecewise_distance_reward)
```

Output: An RL training set focused on the model's learning frontier, typically 40-60% of the original dataset, with curriculum progression from precision refinement to coarse-localization mastery.

## Best Practices

- **Do:** Always require a `<think>` step before every tool invocation -- this prevents degenerate tool-spamming and forces the model to justify why a tool is needed
- **Do:** Implement strict budget limits (6 tool calls max) to prevent infinite loops and encourage efficient tool selection
- **Do:** Use rejection sampling with both format validity AND distance thresholds when synthesizing training trajectories -- format-only filtering produces plausible but geographically wrong traces
- **Do:** Evaluate at multiple distance thresholds (1km, 25km, 200km, 750km) since different applications need different precision levels
- **Avoid:** Training the RL stage without the Agentic Cold Start -- models jump from zero tool-call capability to RL exploration, which produces garbage trajectories and near-zero learning signal
- **Avoid:** Using a flat reward function for RL. The piecewise linear decay (1.0 at <1km, 0.75 at 25km, 0.0 at 200km) provides a shaped gradient that guides the model toward finer precision

## Error Handling

- **Tool call parse failures**: If the model emits malformed JSON in `<tool_call>` tags, return a structured error message in `<tool_response>` and let the model retry within its budget. During training, reject these trajectories entirely.
- **Geocoding API returns no results**: Return an explicit "No results found for this address" rather than empty output. The model should learn to reformulate queries or try alternative place names.
- **Web search returns irrelevant results**: Truncate search results to top-3 snippets to limit noise. If results are clearly off-topic, the model's next `<think>` step should acknowledge this and try a different query.
- **Budget exhaustion without answer**: Force the model to produce its best-guess `<answer>` after the budget is spent. During RL, these forced answers naturally receive lower reward, teaching the model to converge faster.
- **Invalid coordinate format**: Validate that output coordinates are within valid ranges (lat: -90 to 90, lng: -180 to 180). Invalid coordinates receive zero reward in RL regardless of any other factors.

## Limitations

- **Requires external API access**: The framework depends on geocoding and web search APIs at inference time. Offline or air-gapped deployments lose the verification benefit and fall back to internal-knowledge-only prediction.
- **Base model size matters**: The paper uses a 7B-parameter VLM (Qwen2.5-VL-7B). Smaller models may lack sufficient visual reasoning to generate useful tool-call strategies even after cold-start training.
- **Geographic bias in training data**: The MP16-Pro dataset and web-sourced images skew toward tourist landmarks and populated areas. Performance degrades significantly on wilderness, ocean, or featureless terrain images.
- **Tool latency at inference**: Each tool call adds network round-trip latency. With up to 6 calls per image, real-time applications may find the agent too slow without caching or batching strategies.
- **Language-dependent**: Web search and text recognition on signs work best for Latin and CJK scripts. Underrepresented scripts (Georgian, Amharic, Khmer) may yield poor search results and weaker localization.

## Reference

- **Paper**: [SpotAgent: Grounding Visual Geo-localization in Large Vision-Language Models through Agentic Reasoning](https://arxiv.org/abs/2602.09463v2) -- Look for Section 3 (methodology) covering the ReAct framework and tool definitions, Section 4 (training pipeline) covering the 3-stage process and spatially-aware filtering, and Section 5 (experiments) for ablation studies showing the contribution of each component.