---
name: "avenir-web-human-experience-imitating-multimodal-w"
description: "Build robust web automation agents using Mixture of Grounding Experts, experience-imitation planning, and task-tracking checklists. Use when: 'build a web agent', 'automate browser tasks with grounding', 'create a web scraping agent with memory', 'implement element grounding for web automation', 'build a multi-step web task agent', 'add procedural knowledge to a browser agent'."
---

# Avenir-Web: Human-Experience-Imitating Multimodal Web Agents with Mixture of Grounding Experts

This skill teaches Claude to build web automation agents that reliably execute long-horizon tasks on complex, dynamic websites. The core technique from the Avenir-Web paper (arXiv:2602.02468) combines three innovations: a **Mixture of Grounding Experts (MoGE)** that fuses multiple element-location strategies to accurately identify interactive UI elements, **Experience-Imitation Planning** that stores and retrieves site-specific procedural knowledge to guide multi-step workflows, and a **task-tracking checklist with adaptive memory** that prevents the agent from losing its place during extended task sequences. Together, these components solve the three hardest problems in web automation: finding the right element, knowing the right procedure, and staying on track.

## When to Use

- When building an autonomous web agent that must complete multi-step tasks (e.g., "book a flight", "file an expense report") across live websites
- When implementing element grounding logic that must handle diverse UI frameworks (React, Angular, server-rendered HTML, shadow DOM)
- When the user asks to add procedural memory or experience replay to a browser automation system
- When building a Playwright/Puppeteer/Selenium agent that drifts off-task during long action sequences
- When designing a DOM processing pipeline that feeds cleaned, actionable element data to an LLM
- When the user wants to combine visual (screenshot) and structural (DOM/accessibility tree) grounding into a single agent

## Key Technique

### Mixture of Grounding Experts (MoGE)

Web elements can be located through multiple signals: CSS selectors, XPath, accessibility tree roles/labels, visual bounding-box coordinates from screenshots, and textual content matching. No single strategy works reliably across all sites. MoGE runs multiple grounding experts in parallel -- each expert uses a different modality (structural DOM query, accessibility tree lookup, visual coordinate detection via set-of-mark annotation on screenshots, and text/ARIA-label matching). A fusion layer then selects or aggregates the experts' outputs, picking the candidate with the highest confidence or cross-validating across modalities. This makes grounding robust even when a site uses non-standard markup or dynamically-generated class names.

### Experience-Imitation Planning

Rather than reasoning from scratch on every task, the agent maintains a library of **procedural priors** -- step-by-step action traces collected from successful human demonstrations or prior agent runs on the same site. When a new task arrives, the planner retrieves the most relevant prior (matched by site domain + task intent), then adapts it to the current page state. This is analogous to few-shot prompting but with full action trajectories instead of text examples. The prior gives the agent a procedural skeleton ("on Amazon, checkout flow is: cart -> proceed to checkout -> select address -> select payment -> place order") that it adapts dynamically.

### Task-Tracking Checklist + Adaptive Memory

Long-horizon web tasks (10-30+ steps) exceed context windows and cause agents to forget earlier steps or repeat actions. Avenir-Web maintains an explicit **checklist** of subtasks derived from the plan, marking each as pending/in-progress/done after every action. This checklist is always included in the LLM prompt. Alongside it, **adaptive memory** compresses older interaction history (screenshot descriptions, past actions, page states) into summaries, keeping only the most recent 3-5 raw observations in full detail. This prevents context overflow while preserving awareness of prior progress.

## Step-by-Step Workflow

### 1. Parse and Decompose the User Task

Accept the high-level user goal (e.g., "Find the cheapest round-trip flight from NYC to London on Dec 15-22 and book it"). Decompose it into an ordered checklist of subtasks using chain-of-thought reasoning:

```
Checklist:
[ ] Navigate to flight search page
[ ] Enter departure city: NYC
[ ] Enter destination: London
[ ] Set dates: Dec 15 departure, Dec 22 return
[ ] Select round-trip
[ ] Search flights
[ ] Sort by price (lowest first)
[ ] Select cheapest option
[ ] Proceed to booking
[ ] Fill passenger details
[ ] Confirm booking
```

### 2. Retrieve Procedural Priors (Experience-Imitation)

Check the experience library for prior successful traces on the same domain or similar task type. Store priors as JSON:

```json
{
  "domain": "google.com/travel/flights",
  "task_type": "flight_search",
  "action_trace": [
    {"step": 1, "action": "click", "target": "input[aria-label='Where from?']", "note": "Origin field"},
    {"step": 2, "action": "type", "target": "active_element", "value": "{origin}"},
    {"step": 3, "action": "click", "target": "li[data-suggestion]", "note": "Select autocomplete"},
    ...
  ],
  "last_verified": "2026-01-15"
}
```

If a matching prior exists, use it as the planning skeleton. If not, proceed with general reasoning and save the successful trace afterward.

### 3. Capture Page State via Multi-Modal Observation

At each step, collect three parallel observations:
- **Screenshot**: Capture the viewport; optionally annotate interactive elements with numbered markers (set-of-mark)
- **Accessibility tree**: Extract the page's accessibility tree, which gives semantic roles, labels, and states
- **Simplified DOM**: Filter the raw DOM to only interactive elements (`<a>`, `<button>`, `<input>`, `<select>`, `[role="button"]`, `[onclick]`, etc.), stripping style/script tags and preserving hierarchy context

### 4. Run Mixture of Grounding Experts

For the target element identified by the plan (e.g., "the departure date field"), run each grounding expert:

| Expert | Strategy | Output |
|--------|----------|--------|
| **Selector Expert** | Generate CSS selector or XPath from DOM structure | `input#departure-date` |
| **Accessibility Expert** | Match by ARIA role + label from accessibility tree | `textbox "Departure date"` |
| **Visual Expert** | Locate element bounding box from annotated screenshot | `bbox: [245, 380, 420, 410]` |
| **Text Expert** | Find element by visible text content or placeholder | `input[placeholder="Departure"]` |

Fuse results: if 3+ experts agree on the same element, use it with high confidence. If experts disagree, prefer accessibility > selector > visual > text priority order, or fall back to the expert whose modality is most reliable for the current element type (e.g., visual for icon-only buttons).

### 5. Execute Action with Verification

Execute the chosen action (click, type, select, scroll, wait) on the grounded element. After execution, re-capture the page state and verify the expected state change occurred:
- Did a new page load?
- Did a dropdown open?
- Did the input field accept the text?
- Did an error message appear?

If verification fails, retry with the next-best grounding expert's candidate.

### 6. Update Checklist and Memory

After each successful action:
- Mark the completed subtask as `[x]` in the checklist
- Append the action to the current trace log
- If the interaction history exceeds the context budget, summarize older entries:

```
Memory (summarized): Navigated to flights page, entered NYC->London,
set dates Dec 15-22, selected round-trip. Currently on search results page.

Memory (recent, full detail):
- Step 7: Clicked "Sort by: Price" dropdown, selected "Lowest first"
- Step 8: [current] Viewing sorted results, cheapest is $487 on United
```

### 7. Handle Dynamic Page Changes and Errors

When the page changes unexpectedly (modal popup, cookie consent, login wall, CAPTCHA):
1. Detect the interruption by comparing expected vs actual page state
2. Classify it (dismissible overlay, authentication required, error state, CAPTCHA)
3. Handle dismissible interruptions automatically (close modals, accept cookies)
4. For blocking interruptions (login, CAPTCHA), pause and report to the user
5. Resume the checklist from the last verified state

### 8. Save Successful Trace as New Procedural Prior

When the task completes successfully, serialize the full action trace with element selectors, page context summaries, and timing information. Index it by domain and task type for future retrieval.

### 9. Report Results with Evidence

Return the task outcome to the user with: final status (success/partial/failure), the completed checklist, key screenshots or page state at completion, and any data extracted during the task.

## Concrete Examples

**Example 1: Building a Flight Booking Agent**

```
User: Build me a web agent that can search for flights on Google Flights
and return the cheapest option.

Approach:
1. Set up Playwright browser automation with screenshot capability
2. Define the procedural prior for Google Flights:
   - Navigate to google.com/travel/flights
   - Locate origin field via accessibility tree ("Where from?" textbox)
   - Type origin, select autocomplete suggestion
   - Locate destination field, type destination, select suggestion
   - Click date fields, navigate calendar, select dates
   - Click "Search" button
   - Wait for results, extract price data from result cards
3. Implement MoGE grounding:
   - Selector expert: use data-flt-ve attributes specific to Google Flights
   - Accessibility expert: match ARIA labels ("Where from?", "Departure")
   - Visual expert: annotate screenshot with set-of-mark overlays
   - Text expert: match visible labels ("Search", "Explore")
4. Build checklist tracker that logs each step's completion
5. Implement adaptive memory that summarizes older steps

Output (agent architecture):
```python
class FlightSearchAgent:
    def __init__(self, browser):
        self.browser = browser
        self.checklist = TaskChecklist()
        self.memory = AdaptiveMemory(max_recent=5)
        self.grounding = MixtureOfGroundingExperts(
            experts=[
                SelectorExpert(),
                AccessibilityExpert(),
                VisualExpert(screenshot_annotator=SetOfMark()),
                TextMatchExpert(),
            ],
            fusion_strategy="majority_vote_with_priority"
        )
        self.experience_library = ExperienceLibrary("./priors/")

    async def search_flights(self, origin, dest, date_dep, date_ret):
        prior = self.experience_library.retrieve(
            domain="google.com/travel/flights",
            task_type="flight_search"
        )
        plan = self.create_plan(prior, origin=origin, dest=dest,
                                date_dep=date_dep, date_ret=date_ret)
        self.checklist.load(plan.subtasks)

        for step in plan.steps:
            self.checklist.mark_in_progress(step.id)
            page_state = await self.observe()  # screenshot + a11y + DOM
            target = self.grounding.locate(step.target_desc, page_state)
            await self.execute(step.action, target, step.value)
            verified = await self.verify(step.expected_state)
            if not verified:
                target = self.grounding.fallback(step.target_desc, page_state)
                await self.execute(step.action, target, step.value)
            self.checklist.mark_done(step.id)
            self.memory.append(step.summary, page_state.screenshot_desc)
```
```

**Example 2: Adding Grounding Robustness to an Existing Scraper**

```
User: My Playwright scraper breaks every time the site updates its CSS classes.
How do I make element selection more robust?

Approach:
1. Replace single-strategy selectors with MoGE pattern
2. Implement multiple grounding fallbacks per element

Output (grounding module):
```python
class MixtureOfGroundingExperts:
    def __init__(self, page):
        self.page = page

    async def locate(self, description: str) -> ElementHandle:
        candidates = []

        # Expert 1: Accessibility tree matching
        a11y_tree = await self.page.accessibility.snapshot()
        a11y_match = self._find_in_a11y(a11y_tree, description)
        if a11y_match:
            candidates.append(("a11y", a11y_match, 0.9))

        # Expert 2: ARIA and semantic selectors
        aria_el = await self.page.query_selector(
            f'[aria-label*="{description}" i], '
            f'[placeholder*="{description}" i], '
            f'[title*="{description}" i]'
        )
        if aria_el:
            candidates.append(("aria", aria_el, 0.85))

        # Expert 3: Text content matching
        text_el = await self.page.query_selector(
            f'button:has-text("{description}"), '
            f'a:has-text("{description}"), '
            f'label:has-text("{description}")'
        )
        if text_el:
            candidates.append(("text", text_el, 0.8))

        # Expert 4: Visual grounding via screenshot + LLM
        if not candidates:
            screenshot = await self.page.screenshot()
            bbox = await self._visual_ground(screenshot, description)
            if bbox:
                el = await self.page.evaluate_handle(
                    '([x,y]) => document.elementFromPoint(x, y)',
                    [bbox['cx'], bbox['cy']]
                )
                candidates.append(("visual", el, 0.7))

        # Fusion: pick highest confidence candidate
        candidates.sort(key=lambda c: c[2], reverse=True)
        if not candidates:
            raise GroundingError(f"No expert could locate: {description}")
        return candidates[0][1]
```
```

**Example 3: Adding Task-Tracking Checklist to Prevent Drift**

```
User: My web agent keeps losing track of where it is during long
multi-step form submissions. How do I fix this?

Approach:
1. Implement an explicit checklist that persists across LLM calls
2. Always include checklist state in the system prompt
3. Use adaptive memory to compress old steps

Output (checklist + memory module):
```python
class TaskChecklist:
    def __init__(self):
        self.items = []

    def load(self, subtasks: list[str]):
        self.items = [{"task": t, "status": "pending"} for t in subtasks]

    def mark_in_progress(self, index: int):
        self.items[index]["status"] = "in_progress"

    def mark_done(self, index: int):
        self.items[index]["status"] = "done"

    def to_prompt_string(self) -> str:
        lines = ["## Current Task Checklist"]
        for i, item in enumerate(self.items):
            marker = {"pending": "[ ]", "in_progress": "[>]", "done": "[x]"}
            lines.append(f"{marker[item['status']]} {i+1}. {item['task']}")
        return "\n".join(lines)


class AdaptiveMemory:
    def __init__(self, max_recent: int = 5):
        self.max_recent = max_recent
        self.summary = ""
        self.recent = []

    def append(self, action_desc: str, page_context: str):
        self.recent.append({"action": action_desc, "context": page_context})
        if len(self.recent) > self.max_recent:
            oldest = self.recent.pop(0)
            self.summary += f" {oldest['action']}."

    def to_prompt_string(self) -> str:
        parts = []
        if self.summary:
            parts.append(f"Summary of earlier steps:{self.summary}")
        parts.append("Recent actions (full detail):")
        for entry in self.recent:
            parts.append(f"- {entry['action']}")
        return "\n".join(parts)


# Usage in agent prompt construction:
def build_agent_prompt(task, checklist, memory, page_state):
    return f"""You are a web automation agent.

{checklist.to_prompt_string()}

{memory.to_prompt_string()}

Current page state:
- URL: {page_state.url}
- Interactive elements: {page_state.simplified_dom}

Determine the next action to complete the current in-progress checklist item.
Output: {{"action": "click|type|select|scroll|wait", "target": "<description>", "value": "<if applicable>"}}
"""
```
```

## Best Practices

- **Do** implement at least 3 grounding experts (accessibility, text, selector) even for simple agents -- single-strategy grounding is the #1 cause of brittle web automation
- **Do** always include the full task checklist in every LLM prompt; it is the agent's "working memory" anchor and prevents step repetition and goal drift
- **Do** save successful action traces as procedural priors indexed by (domain, task_type) -- reuse across sessions for dramatic reliability improvement
- **Do** verify state after every action; never assume a click succeeded -- check for expected DOM changes, URL updates, or visual confirmation
- **Avoid** consuming the full raw DOM in the LLM prompt; filter to interactive elements only and cap at ~200 elements per observation to stay within token limits
- **Avoid** relying solely on CSS class selectors for grounding -- modern frameworks generate random class names (e.g., `css-1a2b3c`) that change between deployments
- **Avoid** keeping full interaction history in context; always use adaptive memory with summarization for sequences longer than 5-7 steps

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| All grounding experts fail | Element not visible, behind overlay, or dynamically loaded | Scroll the page, wait for network idle, dismiss overlays, then retry grounding |
| Action verification fails | Click landed on wrong element, page didn't respond | Retry with next-best grounding candidate; if repeated failure, re-observe page state from scratch |
| Checklist item impossible | Site flow changed, feature removed, or access denied | Mark item as blocked, log reason, skip to next feasible item, report partial completion |
| Context window overflow | Too many steps accumulated in memory | Trigger aggressive summarization: compress all but last 3 observations, truncate DOM to top-50 elements |
| Unexpected modal/overlay | Cookie consent, newsletter popup, login wall | Detect via DOM mutation observer or screenshot diff; auto-dismiss known patterns (close button with aria-label="Close"); pause for unknown blockers |
| Stale procedural prior | Site redesigned since prior was recorded | Detect when >50% of prior's selectors fail; fall back to general reasoning; flag prior for update |

## Limitations

- **Visual grounding** requires multimodal LLM capabilities (vision models) and adds latency; not all base models support this
- **Procedural priors** become stale as websites update their UIs -- priors need periodic re-validation or automatic staleness detection
- **CAPTCHAs and anti-bot measures** cannot be bypassed by this architecture; the agent must pause and defer to the user
- **Authentication flows** with 2FA/MFA require human intervention at the auth step
- **Single-page applications** with heavy client-side rendering may produce accessibility trees that lag behind the visual state; add explicit wait-for-idle strategies
- **This approach optimizes for task completion reliability, not speed** -- running 4 grounding experts in parallel adds overhead per step (~1-3 seconds)
- **Experience-imitation planning assumes access to prior traces** -- cold-start performance on never-seen sites relies on the base model's general web knowledge

## Reference

**Paper**: [Avenir-Web: Human-Experience-Imitating Multimodal Web Agents with Mixture of Grounding Experts](https://arxiv.org/abs/2602.02468v1) (Li et al., 2026)

Key sections to study: the MoGE architecture for combining structural, semantic, and visual grounding strategies; the experience-imitation planning framework for encoding and retrieving site-specific procedural knowledge; and the task-tracking checklist design that anchors long-horizon execution.