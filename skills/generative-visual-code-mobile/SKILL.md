---
name: "generative-visual-code-mobile"
description: "Predict and generate mobile GUI next-states as renderable HTML/CSS code instead of raw pixels. Use when users ask to 'build a mobile GUI world model', 'predict the next screen state', 'generate mobile UI from actions', 'simulate mobile app navigation', 'convert mobile screenshots to code', or 'build a GUI state predictor'."
---

# Generative Visual Code Mobile World Models

This skill enables Claude to build mobile GUI world models that predict the next screen state as renderable web code (HTML/CSS) rather than generating pixels directly. Based on the gWorld paradigm, a Vision-Language Model takes a current GUI screenshot and a user action (tap, swipe, type) as input, reasons about what will change, then outputs executable HTML/CSS that renders the predicted next state. This approach leverages VLMs' pre-training on structured web code for high-fidelity visual generation and their linguistic priors for precise text rendering -- solving the critical trade-off between text-based world models (accurate text, no visuals) and pixel-generating models (visuals, poor text).

## When to Use

- When the user wants to build a system that predicts what a mobile screen looks like after an action (tap, swipe, scroll, text input)
- When converting mobile GUI screenshots into structured, renderable HTML/CSS representations
- When building a mobile GUI agent that needs to simulate future states before committing to an action
- When creating training data pipelines that re-label pixel-based GUI trajectories into code-based supervision
- When implementing breadth-wise rollout for mobile agent planning (predict K candidate next-states, pick the best)
- When the user needs a self-contained world model without external OCR, diffusion models, or multi-stage filtering pipelines

## Key Technique

**Code-as-visual-representation.** Instead of training a diffusion model to generate the next screenshot pixel-by-pixel, gWorld trains a VLM to output the next GUI state as an HTML/CSS document that can be rendered in a headless browser. This works because modern VLMs are pre-trained on massive web corpora -- they already understand how to produce structured markup that maps to visual layouts. A mobile settings screen, a camera UI, or a messaging app can all be faithfully reconstructed as a single HTML file with inline CSS. Rendering is nearly free (~0.3s per state via headless browser).

**Reasoning-then-code generation.** The model does not jump directly from (screenshot, action) to code. It first generates a natural-language reasoning trace R_t that describes what the action will change ("tapping the Wi-Fi toggle will switch it from OFF to ON and expand the network list below"). This decomposes the hard problem into two easier sub-problems: (1) predict state changes in language, (2) translate the description plus the current state into next-state code. The reasoning trace is generated with look-ahead access to the ground-truth next state during training, giving the model a curriculum that bridges perception and code synthesis.

**Data pipeline (gWorld framework).** Training data is synthesized from existing mobile agent trajectories in three stages: (1) reformat policy episodes {(screenshot, action)} into transition tuples {(screenshot, action, next_screenshot)}, (2) re-label next-state targets from pixels to renderable HTML/CSS using a frontier VLM (e.g., Gemini Flash) with an image-to-code prompt, (3) generate reasoning traces using look-ahead to the ground-truth next state. The final supervised fine-tuning format is: input = (current_screenshot, action_coordinates), output = (reasoning_trace, next_state_code).

## Step-by-Step Workflow

1. **Collect or source mobile GUI trajectories.** Obtain sequences of (screenshot, action, next_screenshot) tuples from existing datasets (Android in the Wild, AndroidControl, AMEX) or by recording real device interactions. Actions must be in coordinate space (tap x,y / swipe x1,y1,x2,y2 / type "text"), not text descriptions of UI elements.

2. **Convert screenshots to renderable HTML/CSS.** For each screenshot in your trajectory, call a strong VLM with an image-to-code prompt: "Convert this mobile screenshot to a single self-contained HTML file with inline CSS that visually reproduces the screen layout, colors, text, icons, and element positions. Use absolute or flexbox positioning. All text must be exact." Validate that the output renders without errors in a headless browser (Playwright/Puppeteer).

3. **Build transition tuples.** Restructure each episode of T steps into T-1 transition tuples: `(S_t_image, A_t, S_{t+1}_code)`. Keep actions in raw coordinate form -- do not convert taps to element names.

4. **Generate reasoning traces with look-ahead.** For each transition, prompt a VLM with: "Given the current screenshot S_t and action A_t, and knowing the ground-truth next state S_{t+1}, describe in 2-4 sentences what changes occur. Focus on which UI elements appear, disappear, move, or change state." Store as R_t.

5. **Assemble SFT dataset.** Each training example has input = `(S_t_image, A_t)` and target = `(R_t, S_{t+1}_code)`. Format the target as a structured output: reasoning block followed by a fenced HTML code block.

6. **Fine-tune the VLM.** Use a vision-language model with strong code generation ability (e.g., Qwen3-VL 8B or 32B) as the base. Fine-tune with standard SFT on your assembled dataset. Scale data to 100K+ transitions for meaningful quality -- the paper shows power-law scaling with R^2 of 0.948.

7. **Implement the inference loop.** At inference time: (a) feed current screenshot + proposed action to the model, (b) parse the reasoning trace and HTML code block from the output, (c) render the HTML in a headless browser to produce a predicted screenshot, (d) apply a renderability filter to catch malformed code before downstream use.

8. **Integrate with agent planning (optional).** For breadth-wise rollout: generate K candidate actions from a policy model, roll out each through the world model to get K predicted next-states, compute value estimates (e.g., via a reward model or VLM judge), select the highest-value action.

9. **Evaluate with VLM-as-Judge.** Use three frontier models (e.g., GPT, Claude, Gemini) to judge whether the predicted next state is semantically consistent with the current state + action. Compute Instruction Accuracy (IAcc) as the primary metric. Use DINO v2 cosine similarity as a secondary perceptual metric.

10. **Monitor render failure rate.** Track the percentage of outputs that fail to render as valid HTML. A well-trained model should stay under 1% render failures. If failures spike, check for degenerate outputs (truncated HTML, missing closing tags) and add targeted training data.

## Concrete Examples

**Example 1: Predicting a toggle action on a settings screen**

User: "Build a world model prediction for what happens when a user taps the Wi-Fi toggle at coordinates (320, 450) on this Android settings screenshot."

Approach:
1. Take the current settings screenshot as input image
2. Encode the action as `{"action": "tap", "x": 320, "y": 450}`
3. Feed both to the world model
4. Model outputs reasoning + HTML code

Output:
```
Reasoning: The user taps the Wi-Fi toggle switch at (320, 450). The toggle
transitions from OFF (grey) to ON (blue). A Wi-Fi network scan initiates,
and a list of available networks appears below the toggle with loading
indicators.

Code:
<!DOCTYPE html>
<html>
<head><meta name="viewport" content="width=device-width, initial-scale=1">
<style>
  body { margin: 0; font-family: 'Roboto', sans-serif; background: #f5f5f5; }
  .header { background: #fff; padding: 16px; font-size: 20px; font-weight: 500;
             border-bottom: 1px solid #e0e0e0; }
  .setting-row { display: flex; justify-content: space-between; align-items: center;
                 padding: 16px; background: #fff; border-bottom: 1px solid #eee; }
  .toggle-on { width: 48px; height: 24px; background: #4285f4; border-radius: 12px;
               position: relative; }
  .toggle-on::after { content: ''; position: absolute; right: 2px; top: 2px;
                      width: 20px; height: 20px; background: #fff; border-radius: 50%; }
  .network-item { padding: 12px 16px; background: #fff; border-bottom: 1px solid #eee;
                  display: flex; justify-content: space-between; }
  .signal { color: #4285f4; }
</style></head>
<body>
  <div class="header">Settings</div>
  <div class="setting-row">
    <span>Wi-Fi</span>
    <div class="toggle-on"></div>
  </div>
  <div class="network-item"><span>Home_Network_5G</span><span class="signal">▂▄▆█</span></div>
  <div class="network-item"><span>CoffeeShop_Free</span><span class="signal">▂▄▆</span></div>
  <div class="network-item"><span>Neighbor_AP</span><span class="signal">▂▄</span></div>
</body>
</html>
```

**Example 2: Building the data pipeline for training**

User: "I have 50K mobile agent trajectories as (screenshot_path, action_json) pairs. Help me build the gWorld data pipeline to create SFT training data."

Approach:
1. Parse trajectories into transition tuples
2. Set up headless browser rendering
3. Convert next-state screenshots to HTML/CSS via frontier VLM
4. Generate reasoning traces with look-ahead
5. Assemble final dataset

Output:
```python
import json
import asyncio
from pathlib import Path
from playwright.async_api import async_playwright

async def screenshot_to_code(vlm_client, screenshot_path: str) -> str:
    """Stage 2: Convert a mobile screenshot to renderable HTML/CSS."""
    prompt = (
        "Convert this mobile screenshot into a single self-contained HTML file "
        "with inline CSS. Reproduce the exact layout, colors, text content, "
        "icons (use Unicode or CSS shapes), and element positions. Use a 360px "
        "wide viewport. All visible text must be character-perfect."
    )
    response = await vlm_client.generate(
        image=screenshot_path,
        prompt=prompt,
        max_tokens=4096,
    )
    # Extract HTML from fenced code block
    code = response.split("```html")[-1].split("```")[0].strip()
    return code

async def generate_reasoning(vlm_client, current_img: str, action: dict,
                              next_img: str) -> str:
    """Stage 3: Generate reasoning trace with look-ahead."""
    prompt = (
        f"Given the current mobile screen (first image), the action "
        f"{json.dumps(action)}, and the resulting next screen (second image), "
        f"describe in 2-4 sentences what changed. Focus on UI element state "
        f"changes, appearing/disappearing elements, and text updates."
    )
    response = await vlm_client.generate(
        images=[current_img, next_img],
        prompt=prompt,
        max_tokens=512,
    )
    return response.strip()

async def validate_renders(html_code: str, browser) -> bool:
    """Filter un-renderable code before including in dataset."""
    page = await browser.new_page(viewport={"width": 360, "height": 800})
    try:
        await page.set_content(html_code, timeout=5000)
        # Check for rendering errors
        errors = await page.evaluate("() => document.querySelectorAll('parsererror').length")
        return errors == 0
    except Exception:
        return False
    finally:
        await page.close()

async def build_sft_dataset(trajectories_dir: str, output_path: str, vlm_client):
    """Full pipeline: trajectories -> SFT training data."""
    episodes = sorted(Path(trajectories_dir).glob("episode_*"))
    sft_data = []

    async with async_playwright() as p:
        browser = await p.chromium.launch()

        for episode_dir in episodes:
            steps = sorted(episode_dir.glob("step_*.json"))
            # Stage 1: Build transition tuples
            for i in range(len(steps) - 1):
                current = json.loads(steps[i].read_text())
                next_step = json.loads(steps[i + 1].read_text())

                current_img = str(episode_dir / current["screenshot"])
                next_img = str(episode_dir / next_step["screenshot"])
                action = current["action"]  # {"type": "tap", "x": 320, "y": 450}

                # Stage 2: Re-label next state as code
                next_code = await screenshot_to_code(vlm_client, next_img)
                if not await validate_renders(next_code, browser):
                    continue  # Skip un-renderable examples

                # Stage 3: Generate reasoning with look-ahead
                reasoning = await generate_reasoning(
                    vlm_client, current_img, action, next_img
                )

                sft_data.append({
                    "input_image": current_img,
                    "input_action": action,
                    "target_reasoning": reasoning,
                    "target_code": next_code,
                })

        await browser.close()

    Path(output_path).write_text(json.dumps(sft_data, indent=2))
    print(f"Generated {len(sft_data)} SFT examples")
```

**Example 3: Agent planning with breadth-wise rollout**

User: "I have a trained gWorld model. Show me how to use it for mobile agent planning with K=3 candidate actions."

Approach:
1. Generate K candidate actions from the policy model
2. Roll out each through the world model
3. Score predicted states
4. Select the best action

Output:
```python
async def plan_with_world_model(
    policy_model, world_model, value_model, screenshot: str,
    task: str, k: int = 3
) -> dict:
    """Breadth-wise rollout: predict K futures, pick the best action."""

    # Step 1: Generate K candidate actions from policy
    candidates = await policy_model.generate_actions(
        screenshot=screenshot, task=task, num_candidates=k
    )
    # candidates = [{"type": "tap", "x": 150, "y": 300}, ...]

    # Step 2: Roll out each action through world model
    predicted_states = []
    for action in candidates:
        wm_output = await world_model.predict(
            screenshot=screenshot, action=action
        )
        reasoning = wm_output["reasoning"]
        html_code = wm_output["code"]

        # Render to pixels for value estimation
        rendered_screenshot = await render_html_to_image(html_code)

        # Skip if render failed
        if rendered_screenshot is None:
            predicted_states.append(None)
            continue

        predicted_states.append({
            "action": action,
            "reasoning": reasoning,
            "predicted_screenshot": rendered_screenshot,
        })

    # Step 3: Score each predicted state
    scores = []
    for state in predicted_states:
        if state is None:
            scores.append(-1.0)
            continue
        score = await value_model.estimate(
            current=screenshot,
            predicted_next=state["predicted_screenshot"],
            task=task,
        )
        scores.append(score)

    # Step 4: Select best action
    best_idx = max(range(len(scores)), key=lambda i: scores[i])
    return candidates[best_idx]
```

## Best Practices

- **Do:** Keep actions in coordinate space (tap x,y / swipe x1,y1,x2,y2). Converting to text element references breaks compatibility with real device execution and loses spatial precision.
- **Do:** Use a headless browser (Playwright/Puppeteer) with a fixed 360x800 viewport for consistent mobile rendering. The ~0.3s render cost is negligible compared to model inference.
- **Do:** Generate reasoning traces before code in the target sequence. The reasoning step substantially improves code accuracy by decomposing the problem.
- **Do:** Validate every generated HTML for renderability before including it in training data or using it downstream. Filter out examples with parse errors or empty renders.
- **Avoid:** Generating pixels directly with diffusion models when text fidelity matters. Pixel-based approaches consistently fail at rendering small text, numbers, and UI labels accurately.
- **Avoid:** Complex multi-stage pipelines with separate OCR, text masking, diffusion, and VLM filtering components. The single-model code generation approach eliminates this fragility and achieves better results at lower compute cost.

## Error Handling

- **Malformed HTML output:** Apply a rule-based filter before rendering. Check for balanced tags, presence of `<html>` and `<body>`, and absence of truncation (model hitting max token limit). If truncation is frequent, increase max output tokens or simplify the target HTML during data generation.
- **Render failures at inference time:** Catch browser exceptions and return a fallback (e.g., the current state unchanged). Log failures for model debugging. A render failure rate above 1% indicates training data quality issues.
- **Coordinate mismatch:** If the model generates HTML at a different viewport size than expected, rendered elements will be misaligned. Always set the viewport explicitly in both data generation and inference rendering.
- **Reasoning-code inconsistency:** The model may describe one change in reasoning but produce code showing a different change. Use a lightweight consistency check: feed the reasoning and rendered screenshot to a judge model to flag contradictions.
- **Out-of-distribution apps:** Performance degrades on app categories not seen in training (e.g., games, custom UI frameworks). Mitigate by diversifying training trajectories across app types.

## Limitations

- Works best on standard mobile UI patterns (settings, messaging, lists, forms). Highly custom or game-like interfaces with non-standard rendering are poorly captured by HTML/CSS.
- Requires a strong base VLM with both vision understanding and code generation ability. Models below ~8B parameters typically lack sufficient code generation quality.
- The HTML/CSS representation cannot capture animations, video content, or real-time dynamic elements -- it produces a static snapshot of the predicted state.
- Training data generation depends on a frontier VLM for image-to-code conversion, creating a bootstrapping dependency. The quality ceiling of synthetic data is bounded by the teacher model.
- Coordinate-based actions assume a fixed screen resolution. Cross-resolution generalization requires normalization during training and denormalization at inference.

## Reference

[Generative Visual Code Mobile World Models](https://arxiv.org/abs/2602.01576v1) -- Koh et al., 2026. Focus on Section 3 (method: code generation paradigm and data pipeline), Section 4.2 (MWMBench evaluation protocol), and Section 5.3 (downstream agent integration via breadth-wise rollout).