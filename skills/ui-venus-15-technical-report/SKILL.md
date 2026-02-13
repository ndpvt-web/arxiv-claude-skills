---
name: "ui-venus-15-technical-report"
description: "Build GUI automation agents using UI-Venus-1.5 patterns: screenshot-only perception, coordinate-based grounding, trajectory-level RL, and model merging for unified web/mobile agents. Use when: 'build a GUI agent', 'automate mobile app navigation', 'screen grounding pipeline', 'web automation with vision', 'unified UI agent architecture', 'screenshot-based action prediction'."
---

This skill teaches Claude to design, architect, and implement GUI automation agents following the UI-Venus-1.5 methodology: a screenshot-only, end-to-end approach where a vision-language model perceives raw screenshots, grounds UI elements to pixel coordinates, predicts actions (click, type, scroll, swipe), and executes multi-step navigation trajectories across web and mobile environments — all without relying on DOM trees or accessibility APIs.

## When to Use

- When the user wants to build a GUI agent that navigates apps or websites using screenshots alone
- When designing a training pipeline for a vision-language model that must ground UI elements to coordinates
- When the user asks to automate mobile app interactions (Android/iOS) via visual perception
- When building a web automation system that doesn't depend on selectors or DOM access
- When the user wants to merge multiple specialist models (grounding, web nav, mobile nav) into one unified agent
- When implementing reinforcement learning with trajectory-level rollouts for multi-step UI tasks
- When creating an agent loop that perceives screenshots, reasons about next actions, and executes them iteratively

## Key Technique

**Screenshot-Only Perception with Coordinate Grounding.** UI-Venus-1.5 processes raw screenshots as the sole observation input — no DOM, no accessibility tree, no HTML. The model predicts pixel coordinates `[x, y]` for each action target, making it platform-agnostic. This visual grounding approach works identically on Android, iOS, desktop, and web because every platform renders to pixels. The action space is compact: `click(x, y)`, `type(x, y, text)`, `scroll(x, y, direction)`, `swipe(x1, y1, x2, y2)`, `long_press(x, y)`, and terminal actions like `finish` or `go_back`.

**Four-Stage Training with Online RL.** The training pipeline proceeds through: (1) Mid-training on 10B tokens across 30+ GUI datasets to inject foundational UI semantics (element recognition, spatial relationships, text-in-image understanding); (2) Offline RL with task-specific optimization for grounding, web, and mobile objectives separately; (3) Online RL using full-trajectory rollouts where the agent interacts with live environments and receives task-completion rewards, aligning training with the actual multi-step navigation objective; (4) Model merging to combine the three domain-specific checkpoints into one unified agent. The online RL stage is critical — offline training alone produces agents that drift during long trajectories, while trajectory-level rollouts teach recovery from errors.

**Model Merging for Unified Agents.** Rather than training a single model on all tasks (which causes interference) or deploying multiple models (which is expensive), UI-Venus-1.5 trains separate specialists for grounding, web navigation, and mobile navigation, then merges their weights into a single checkpoint. This preserves domain-specific capabilities while eliminating the need for routing or model selection at inference time.

## Step-by-Step Workflow

1. **Define the action space as structured JSON.** Create an enum of supported actions with typed parameters. Each action must map to a screen coordinate plus optional metadata:
   ```json
   {"action": "click", "coordinates": [x, y]}
   {"action": "type", "coordinates": [x, y], "text": "search query"}
   {"action": "scroll", "coordinates": [x, y], "direction": "down"}
   {"action": "swipe", "start": [x1, y1], "end": [x2, y2]}
   {"action": "long_press", "coordinates": [x, y]}
   {"action": "finish", "status": "success"}
   ```

2. **Build the perception module.** Accept a screenshot (PNG/JPEG) as input and feed it to a vision-language model (e.g., Qwen3-VL via vLLM). The prompt should include the user's task instruction and request the model to identify the target UI element and output its coordinates. Normalize coordinates to the screenshot resolution.

3. **Construct the agent loop.** Implement a perceive-reason-act cycle:
   - Capture screenshot of current environment state
   - Send screenshot + task instruction + action history to the VLM
   - Parse the model's response into a structured action
   - Execute the action in the target environment (via ADB for Android, Playwright for web, etc.)
   - Check for task completion or maximum step limit
   - Loop until done

4. **Design the system prompt with explicit action format.** The prompt must specify available actions, coordinate format, and expected output structure. Include 1-2 in-context examples of successful action predictions to anchor the model's output format.

5. **Implement environment connectors.** Write adapters for each target platform:
   - **Android**: Use ADB to capture screenshots (`adb screencap`) and execute taps/swipes (`adb input tap x y`)
   - **Web**: Use Playwright or Selenium to screenshot pages and dispatch click/type events at coordinates
   - **Desktop**: Use platform-specific screen capture and input simulation

6. **Add trajectory tracking and loop detection.** Maintain a history of (screenshot_hash, action) pairs. If the agent repeats the same action on a visually identical screen 3+ times, trigger a recovery strategy: try an alternative action, scroll to reveal new content, or report failure.

7. **Implement error recovery heuristics.** When the agent appears stuck (no visual change after action, unexpected dialog/popup), inject recovery prompts: "An unexpected dialog appeared. Describe what you see and decide whether to dismiss it or interact with it."

8. **Deploy the VLM server.** Use vLLM for efficient inference:
   ```bash
   python -m vllm.entrypoints.openai.api_server \
       --model inclusionAI/UI-Venus-1.5-8B \
       --served-model-name ui-venus \
       --host 0.0.0.0 --port 8000 \
       --tensor-parallel-size 1 \
       --trust-remote-code
   ```
   Then call it via the OpenAI-compatible API at `http://localhost:8000/v1`.

9. **Evaluate on standard benchmarks.** Test grounding accuracy on ScreenSpot-Pro (element localization) and navigation success on AndroidWorld (end-to-end mobile tasks) or WebVoyager (web tasks). Track both step-level accuracy and full-trajectory success rate.

10. **Optionally fine-tune with trajectory-level RL.** Collect rollout trajectories from your target domain, assign binary task-completion rewards, and fine-tune with GRPO or similar policy-gradient methods. This closes the distribution gap between training data and your specific UI environment.

## Concrete Examples

**Example 1: Building an Android App Automation Agent**

User: "Build me a Python agent that can navigate the Settings app on Android to turn on Wi-Fi, using only screenshots."

Approach:
1. Set up ADB connection and verify device access
2. Deploy UI-Venus-1.5-8B via vLLM
3. Implement the agent loop with screenshot capture and action execution

```python
import subprocess
import base64
import requests

VLLM_URL = "http://localhost:8000/v1/chat/completions"
MODEL = "ui-venus"
MAX_STEPS = 15

SYSTEM_PROMPT = """You are a GUI agent controlling an Android device.
You can ONLY see screenshots. Output exactly one action per step as JSON.

Supported actions:
- {"action": "click", "coordinates": [x, y]}
- {"action": "type", "coordinates": [x, y], "text": "..."}
- {"action": "scroll", "coordinates": [x, y], "direction": "up"|"down"}
- {"action": "swipe", "start": [x1, y1], "end": [x2, y2]}
- {"action": "finish", "status": "success"|"failure"}

Coordinates are absolute pixels. Respond with ONLY the JSON action."""

def capture_screenshot():
    subprocess.run(["adb", "shell", "screencap", "-p", "/sdcard/screen.png"])
    subprocess.run(["adb", "pull", "/sdcard/screen.png", "/tmp/screen.png"])
    with open("/tmp/screen.png", "rb") as f:
        return base64.b64encode(f.read()).decode()

def execute_action(action):
    if action["action"] == "click":
        x, y = action["coordinates"]
        subprocess.run(["adb", "shell", "input", "tap", str(x), str(y)])
    elif action["action"] == "type":
        x, y = action["coordinates"]
        subprocess.run(["adb", "shell", "input", "tap", str(x), str(y)])
        subprocess.run(["adb", "shell", "input", "text", action["text"]])
    elif action["action"] == "scroll":
        x, y = action["coordinates"]
        dy = -300 if action["direction"] == "up" else 300
        subprocess.run(["adb", "shell", "input", "swipe",
                        str(x), str(y), str(x), str(y + dy)])

def run_agent(task: str):
    history = []
    for step in range(MAX_STEPS):
        screenshot_b64 = capture_screenshot()
        messages = [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": [
                {"type": "text", "text": f"Task: {task}\nStep {step+1}. What action should I take?"},
                {"type": "image_url", "image_url": {"url": f"data:image/png;base64,{screenshot_b64}"}}
            ]}
        ]
        resp = requests.post(VLLM_URL, json={"model": MODEL, "messages": messages})
        action = parse_json(resp.json()["choices"][0]["message"]["content"])
        if action["action"] == "finish":
            return action["status"]
        execute_action(action)
        history.append(action)
    return "timeout"

run_agent("Open Settings and turn on Wi-Fi")
```

**Example 2: Web Navigation Agent with Playwright**

User: "Create a web agent that searches for a product on an e-commerce site and adds the first result to cart."

Approach:
1. Launch a Playwright browser and navigate to the target site
2. Use the same VLM-based perceive-reason-act loop
3. Capture page screenshots and predict click/type coordinates

```python
from playwright.sync_api import sync_playwright
import base64, requests, json

def run_web_agent(url: str, task: str):
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=False)
        page = browser.new_page(viewport={"width": 1280, "height": 720})
        page.goto(url)

        for step in range(20):
            screenshot = page.screenshot()
            b64 = base64.b64encode(screenshot).decode()

            resp = call_vlm(task=task, screenshot_b64=b64, step=step)
            action = json.loads(resp)

            if action["action"] == "finish":
                break
            elif action["action"] == "click":
                page.mouse.click(action["coordinates"][0], action["coordinates"][1])
            elif action["action"] == "type":
                page.mouse.click(action["coordinates"][0], action["coordinates"][1])
                page.keyboard.type(action["text"])
            elif action["action"] == "scroll":
                delta = -300 if action["direction"] == "up" else 300
                page.mouse.wheel(0, delta)

            page.wait_for_timeout(1500)  # wait for UI to settle
        browser.close()
```

**Example 3: Designing a Training Data Pipeline for GUI Grounding**

User: "I want to create training data for a GUI grounding model. How should I structure it?"

Approach:
1. Collect screenshots from diverse platforms with element annotations
2. Format as instruction-coordinate pairs

```json
{
  "image": "screenshots/settings_wifi.png",
  "conversations": [
    {
      "role": "user",
      "content": "Click on the Wi-Fi toggle switch"
    },
    {
      "role": "assistant",
      "content": "{\"action\": \"click\", \"coordinates\": [892, 245]}"
    }
  ]
}
```

Data refinement pipeline (from UI-Venus):
1. **Prompt rewrite** — Use an LLM to clarify ambiguous instructions ("tap the thing" becomes "Tap the Wi-Fi toggle in Settings")
2. **Trace editing** — Remove redundant or incorrect steps from recorded trajectories
3. **Trace generation** — Synthesize missing steps using LLM augmentation to fill gaps in demonstrations

## Best Practices

- **Do:** Use screenshot-only input for platform independence. DOM/accessibility trees change across versions and platforms; pixels are universal.
- **Do:** Include action history in the prompt context so the model can reason about what it has already tried and avoid repetition.
- **Do:** Add a `wait_for_stable` step after each action (1-2 seconds) before capturing the next screenshot, since UI animations and network requests cause intermediate states.
- **Do:** Normalize coordinate outputs to the actual viewport size. If the model was trained on 1280x720 screenshots but your device is 1080x2400, scale accordingly.
- **Avoid:** Sending multiple screenshots per step unless required. One current-state screenshot per reasoning step keeps the context focused and reduces token cost.
- **Avoid:** Relying on the agent to handle authentication flows (CAPTCHAs, 2FA) — these require human intervention. Detect and escalate them.
- **Avoid:** Training on a single platform only. The model merging technique works because each specialist sees diverse examples from its domain. A web-only model will fail on mobile layouts.

## Error Handling

| Problem | Detection | Recovery |
|---------|-----------|----------|
| Agent clicks wrong element | Visual diff shows no expected state change | Re-prompt with "The previous click did not navigate to the expected screen. Look more carefully at the screenshot and try a different element." |
| Unexpected popup/dialog | Screenshot contains modal overlay | Inject prompt: "A dialog has appeared. Read its content and dismiss it if it's not relevant to the task." |
| Agent enters infinite loop | Same (screenshot_hash, action) pair seen 3+ times | Force a different action: scroll, go back, or report failure |
| Coordinate out of bounds | Predicted coordinates exceed screenshot dimensions | Clamp to valid range and log a warning; consider re-prompting |
| Environment timeout | Action execution exceeds time limit | Capture screenshot to diagnose; the app may be loading. Wait and retry once before failing |
| Model outputs malformed JSON | JSON parse failure | Retry with a stricter prompt emphasizing the exact output format; include a concrete example in the retry |

## Limitations

- **Screenshot-only perception misses non-visual state.** Hidden form validation, background network requests, and off-screen content are invisible to the agent. Supplement with platform APIs when critical state is not rendered.
- **Coordinate precision degrades on dense UIs.** Small buttons, overlapping elements, and complex menus (especially on mobile) cause localization errors. Higher-resolution screenshots help but increase inference cost.
- **Long trajectories accumulate errors.** Each step has a small failure probability; over 20+ steps, compounding errors cause frequent task failure. The online RL stage mitigates this but doesn't eliminate it.
- **Model merging requires compatible architectures.** You can only merge models that share the same base architecture and tokenizer. Merging a grounding specialist with a navigation specialist works when both are fine-tuned from the same base (e.g., Qwen3-VL).
- **Real-time performance is limited by VLM inference speed.** Each step requires a full vision-language forward pass. Expect 1-5 seconds per step depending on model size and hardware. The 2B variant is fastest; the 30B-A3B MoE variant is most capable.
- **Does not handle CAPTCHAs, biometric prompts, or 2FA.** These require human-in-the-loop escalation.

## Reference

**Paper:** [UI-Venus-1.5 Technical Report](https://arxiv.org/abs/2602.09082v1) — Focus on Section 3 (four-stage training pipeline), Section 4 (online RL with trajectory rollouts), and Section 5 (model merging methodology). The benchmark results in Section 6 provide baselines for evaluating your own GUI agent implementations.

**Code:** [github.com/inclusionAI/UI-Venus](https://github.com/inclusionAI/UI-Venus) | **Models:** [huggingface.co/collections/inclusionAI/ui-venus](https://huggingface.co/collections/inclusionAI/ui-venus)