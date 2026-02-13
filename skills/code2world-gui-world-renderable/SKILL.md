---
name: "code2world-gui-world-renderable"
description: "Predict and simulate GUI state transitions by generating renderable HTML/CSS/SVG code from screenshots and user actions. Use when asked to: 'simulate what happens when I click this button', 'predict the next UI state', 'generate HTML that reproduces this screenshot', 'build a GUI world model', 'create a visual sandbox for UI testing', 'convert a mobile screenshot to interactive HTML'."
---

# Code2World: GUI State Prediction via Renderable Code Generation

This skill enables Claude to simulate GUI state transitions by generating self-contained, renderable HTML/CSS/SVG code that represents the next visual state of a user interface after an action is performed. Based on the Code2World paper (arXiv:2602.09856), the core insight is that instead of predicting pixels directly or describing changes in text, you generate structured code that a browser can deterministically render into a pixel-accurate UI state. This bridges high visual fidelity with fine-grained structural controllability -- the code is inspectable, editable, and renders consistently.

## When to Use

- When a user provides a screenshot of an app and asks "what happens if I tap X?" -- predict the next UI state as renderable HTML
- When building a GUI testing sandbox that needs to simulate user interactions without a live backend
- When converting mobile app screenshots into self-contained HTML reproductions for prototyping or documentation
- When creating action-conditioned UI mockups (e.g., "show me what this form looks like after validation fails")
- When building an agent evaluation harness that needs deterministic GUI state prediction
- When prototyping UI flows by chaining predicted states: screenshot -> action -> predicted HTML -> render -> next action

## Key Technique

**Code as intermediate representation for UI prediction.** Traditional GUI world models either generate text descriptions (losing spatial/visual detail) or synthesize pixels directly (lacking structural control and producing blurry or hallucinated outputs). Code2World sidesteps this tradeoff by predicting the *code* that, when rendered by a browser engine, produces the next UI state. The prediction is modeled as: given a current screenshot `I_t`, an action `a_t` (e.g., click at coordinates, type text), and a task goal `G`, generate HTML code `C_{t+1}` such that rendering `R(C_{t+1})` produces a faithful visual representation of the next screen.

**Self-contained HTML with semantic placeholders.** The generated code uses a fixed-dimension root container matching the original screenshot's coordinate space. Images are replaced with descriptive text placeholders (e.g., `[IMG: Red sneaker product photo]`) since image URLs are unreliable and unnecessary for structural prediction. UI icons are rendered as inline SVGs. No external assets or dependencies are required -- the HTML is fully self-contained and renderable in any browser.

**Visual-feedback refinement loop.** Code quality is ensured by rendering the generated HTML and comparing it visually against the target screenshot. When the rendered output diverges (measured by visual similarity scoring), the code is revised by identifying specific discrepancies and correcting them. This render-compare-revise cycle is the key to achieving high fidelity without pixel-level generation.

## Step-by-Step Workflow

1. **Analyze the source screenshot.** Identify all visible UI elements: navigation bars, buttons, text fields, lists, images, icons, status bars, and their spatial layout. Note coordinates, sizes, colors, and hierarchy. For mobile UIs, identify the platform (Android/iOS) and standard UI patterns.

2. **Parse the specified action.** Determine the action type (tap/click at coordinates, long press, scroll, type text, swipe) and its target element. Map the action coordinates to the specific UI element being interacted with. Understand what this action would logically trigger (navigation, state change, modal, input focus, etc.).

3. **Determine the expected state transition.** Based on the action and standard UI behavior, predict what changes: new screens appearing, elements expanding/collapsing, text being entered, selections changing, dialogs opening, navigation occurring. Preserve all elements that should remain unchanged.

4. **Generate the HTML document structure.** Create a self-contained HTML file with a root `<div>` container set to the exact dimensions of the original screenshot (typically 1080x2400 for mobile). Use absolute positioning to place elements at their correct coordinates. Apply inline styles for all visual properties.

5. **Encode visual elements as renderable code.** Translate each UI component into HTML/CSS: use `<div>` elements with background colors and border-radius for buttons, inline SVG for icons, styled `<span>` elements for text, and descriptive placeholder blocks (`[IMG: description]`) for images. Match colors, fonts, spacing, and sizing from the original screenshot.

6. **Apply the action-conditioned changes.** Modify the HTML to reflect the predicted next state: add new elements for newly visible content, remove or hide dismissed elements, update text content for input actions, change visual states (e.g., button pressed, checkbox toggled), and adjust layout for scroll or navigation actions.

7. **Validate structural consistency.** Review the generated HTML to ensure: the coordinate system is preserved, no elements overlap incorrectly, text is readable, interactive elements are visually distinct, and the overall layout matches expected platform conventions.

8. **Render and compare.** If possible, render the HTML in a browser or headless renderer and visually compare against expectations. Identify discrepancies in element positioning, sizing, color, or missing components. Revise the code to correct any issues found.

9. **Deliver the output.** Provide the complete, self-contained HTML file that can be opened directly in a browser. Include a brief description of what changed and why, linking the action to the visual state transition.

## Concrete Examples

**Example 1: Predicting a button tap on a settings screen**

User: "Here's a screenshot of an Android settings page. What happens when I tap the 'Wi-Fi' toggle that's currently OFF?"

Approach:
1. Analyze the screenshot: standard Android settings list with toggles
2. Action: tap on Wi-Fi toggle at approximately (980, 340)
3. Predicted change: toggle switches from OFF (gray) to ON (blue/accent color), Wi-Fi network name may appear below

Output:
```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { background: #f5f5f5; font-family: 'Roboto', sans-serif; }
</style></head>
<body>
<div style="width:1080px;height:2400px;position:relative;background:#f5f5f5;">
  <!-- Status bar -->
  <div style="position:absolute;top:0;left:0;width:1080px;height:88px;background:#fff;
    display:flex;align-items:center;padding:0 32px;">
    <span style="font-size:28px;color:#333;">9:41</span>
    <!-- ... battery, signal icons as inline SVG ... -->
  </div>
  <!-- Settings header -->
  <div style="position:absolute;top:88px;left:0;width:1080px;height:120px;background:#fff;
    display:flex;align-items:center;padding:0 40px;">
    <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="#333" stroke-width="2">
      <path d="M15 18l-6-6 6-6"/></svg>
    <span style="font-size:44px;color:#1a1a1a;margin-left:32px;font-weight:500;">Settings</span>
  </div>
  <!-- Wi-Fi row - CHANGED: toggle is now ON -->
  <div style="position:absolute;top:240px;left:0;width:1080px;height:140px;background:#fff;
    display:flex;align-items:center;padding:0 40px;border-bottom:1px solid #e0e0e0;">
    <svg width="48" height="48" viewBox="0 0 24 24" fill="#1a73e8">
      <path d="M1 9l2 2c4.97-4.97 13.03-4.97 18 0l2-2C16.93 2.93 7.08 2.93 1 9zm8 8l3 3 3-3
        c-1.65-1.66-4.34-1.66-6 0zm-4-4l2 2c2.76-2.76 7.24-2.76 10 0l2-2
        C15.14 9.14 8.87 9.14 5 13z"/>
    </svg>
    <div style="margin-left:32px;flex:1;">
      <span style="font-size:34px;color:#1a1a1a;">Wi-Fi</span>
      <span style="font-size:26px;color:#666;display:block;margin-top:4px;">HomeNetwork_5G</span>
    </div>
    <!-- Toggle ON state -->
    <div style="width:100px;height:56px;background:#1a73e8;border-radius:28px;position:relative;">
      <div style="width:48px;height:48px;background:#fff;border-radius:50%;
        position:absolute;top:4px;right:4px;box-shadow:0 2px 4px rgba(0,0,0,0.2);"></div>
    </div>
  </div>
  <!-- Bluetooth row - unchanged -->
  <div style="position:absolute;top:380px;left:0;width:1080px;height:140px;background:#fff;
    display:flex;align-items:center;padding:0 40px;border-bottom:1px solid #e0e0e0;">
    <svg width="48" height="48" viewBox="0 0 24 24" fill="#666">
      <path d="M17.71 7.71L12 2h-1v7.59L6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 11 14.41V22h1
        l5.71-5.71-4.3-4.29 4.3-4.29zM13 5.83l1.88 1.88L13 9.59V5.83zm1.88 10.46L13 18.17
        v-3.76l1.88 1.88z"/>
    </svg>
    <div style="margin-left:32px;flex:1;">
      <span style="font-size:34px;color:#1a1a1a;">Bluetooth</span>
    </div>
    <div style="width:100px;height:56px;background:#bdbdbd;border-radius:28px;position:relative;">
      <div style="width:48px;height:48px;background:#fff;border-radius:50%;
        position:absolute;top:4px;left:4px;box-shadow:0 2px 4px rgba(0,0,0,0.2);"></div>
    </div>
  </div>
</div>
</body>
</html>
```

Key changes: Wi-Fi toggle moved from gray/left (OFF) to blue/right (ON), Wi-Fi icon turned blue, connected network name appeared below the label.

---

**Example 2: Simulating text input in a search field**

User: "I have a screenshot of Google's home page. Simulate what it looks like after I tap the search bar and type 'weather today'."

Approach:
1. Identify the search bar element and its coordinates
2. Action: tap search bar (focus) then type "weather today"
3. Predicted changes: search bar gains focus styling, cursor appears, typed text is visible, autocomplete suggestions may appear below

Output:
```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
</style></head>
<body>
<div style="width:1080px;height:2400px;position:relative;background:#fff;">
  <!-- Google logo placeholder -->
  <div style="position:absolute;top:600px;left:50%;transform:translateX(-50%);
    width:520px;height:180px;display:flex;align-items:center;justify-content:center;">
    <span style="font-size:96px;font-weight:bold;">
      <span style="color:#4285F4;">G</span><span style="color:#EA4335;">o</span>
      <span style="color:#FBBC05;">o</span><span style="color:#4285F4;">g</span>
      <span style="color:#34A853;">l</span><span style="color:#EA4335;">e</span>
    </span>
  </div>
  <!-- Search bar - CHANGED: focused with typed text -->
  <div style="position:absolute;top:820px;left:80px;width:920px;height:100px;
    border:2px solid #4285F4;border-radius:50px;display:flex;align-items:center;
    padding:0 32px;box-shadow:0 4px 12px rgba(0,0,0,0.15);">
    <svg width="40" height="40" viewBox="0 0 24 24" fill="#4285F4">
      <path d="M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3
        9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5
        4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14
        9.5 11.99 14 9.5 14z"/>
    </svg>
    <span style="font-size:34px;color:#1a1a1a;margin-left:20px;">weather today</span>
    <span style="display:inline-block;width:2px;height:40px;background:#4285F4;
      margin-left:2px;animation:blink 1s step-end infinite;"></span>
    <div style="flex:1;"></div>
    <svg width="40" height="40" viewBox="0 0 24 24" fill="#999" style="margin-left:16px;">
      <path d="M19 11h-1.7c0 .74-.16 1.44-.43 2.08l1.27 1.27c.56-.97.86-2.1.86-3.35zm-4.02
        .17c0-.06.02-.11.02-.17V5c0-1.66-1.34-3-3-3S9 3.34 9 5v6c0 .06 0 .11.02.17l5.96
        0zM4.27 3L3 4.27l6.01 6.01V11c0 1.66 1.33 3 2.99 3 .22 0 .44-.03.65-.08l1.66
        1.66c-.71.33-1.5.52-2.31.52-2.76 0-5.3-2.1-5.3-5.1H5c0 3.41 2.72 6.23 6 6.72V21h2
        v-3.28c.91-.13 1.77-.45 2.54-.9L19.73 21 21 19.73 4.27 3z"/>
    </svg>
  </div>
  <!-- Autocomplete suggestions - NEW element -->
  <div style="position:absolute;top:930px;left:80px;width:920px;background:#fff;
    border:1px solid #dfe1e5;border-radius:0 0 24px 24px;box-shadow:0 4px 6px rgba(0,0,0,0.1);
    padding:16px 0;">
    <div style="padding:16px 32px;display:flex;align-items:center;">
      <svg width="36" height="36" viewBox="0 0 24 24" fill="#999">
        <path d="M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3
          9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5
          4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14
          9.5 11.99 14 9.5 14z"/>
      </svg>
      <span style="font-size:30px;color:#1a1a1a;margin-left:20px;">weather today in my area</span>
    </div>
    <div style="padding:16px 32px;display:flex;align-items:center;">
      <svg width="36" height="36" viewBox="0 0 24 24" fill="#999">
        <path d="M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3
          9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5
          4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14
          9.5 11.99 14 9.5 14z"/>
      </svg>
      <span style="font-size:30px;color:#1a1a1a;margin-left:20px;">weather today hourly</span>
    </div>
  </div>
</div>
<style>@keyframes blink { 50% { opacity: 0; } }</style>
</body>
</html>
```

Key changes: search bar gained focus border (blue) and shadow, typed text appeared, cursor blinks after text, autocomplete dropdown appeared below with suggestions.

---

**Example 3: Navigation action -- tapping a list item**

User: "This is an email inbox screenshot. Predict what the screen looks like after I tap on the first email in the list."

Approach:
1. Identify the first email list item and its position
2. Action: tap to open the email
3. Predicted state: full navigation to email detail view -- new screen replaces the list

Output structure (abbreviated):
```html
<!-- New screen: Email detail view -->
<div style="width:1080px;height:2400px;position:relative;background:#fff;">
  <!-- App bar with back arrow and action icons -->
  <div style="position:absolute;top:88px;left:0;width:1080px;height:120px;...">
    <svg><!-- back arrow --></svg>
    <div style="flex:1;"></div>
    <svg><!-- archive icon --></svg>
    <svg><!-- delete icon --></svg>
    <svg><!-- more options --></svg>
  </div>
  <!-- Sender info with avatar placeholder -->
  <div style="position:absolute;top:220px;...">
    <div style="width:80px;height:80px;border-radius:50%;background:#1a73e8;
      display:flex;align-items:center;justify-content:center;">
      <span style="color:#fff;font-size:36px;font-weight:bold;">JD</span>
    </div>
    <div>
      <span style="font-size:34px;font-weight:500;">John Doe</span>
      <span style="font-size:26px;color:#666;">to me</span>
    </div>
  </div>
  <!-- Email subject -->
  <div style="position:absolute;top:340px;left:40px;">
    <span style="font-size:40px;font-weight:400;color:#1a1a1a;">Q4 Report Review</span>
  </div>
  <!-- Email body text -->
  <div style="position:absolute;top:420px;left:40px;right:40px;">
    <p style="font-size:30px;color:#333;line-height:1.6;">
      Hi, please find attached the Q4 report for your review.
      Let me know if you have any questions...
    </p>
  </div>
  <!-- Reply/Forward bar at bottom -->
  <div style="position:absolute;bottom:0;left:0;width:1080px;height:120px;...">
    <button style="...">Reply</button>
    <button style="...">Forward</button>
  </div>
</div>
```

Key change: entire screen transitioned from inbox list to email detail view, preserving platform UI conventions (Material Design app bar, avatar, action buttons).

## Best Practices

**Do:**
- Use a fixed-dimension root container that matches the original screenshot's resolution (e.g., 1080x2400 for Android phones, 1170x2532 for iPhone 14 Pro). This preserves the coordinate system for accurate element placement.
- Replace all images with descriptive semantic placeholders like `[IMG: Product thumbnail of red sneakers]`. These convey content meaning without requiring external assets.
- Render all UI icons as inline SVGs rather than referencing icon fonts or external files. This keeps the HTML completely self-contained.
- Preserve unchanged elements exactly as they are. Only modify elements affected by the action. State prediction accuracy depends on minimizing spurious changes.

**Avoid:**
- Do not reference external CSS frameworks, fonts, images, or scripts. The HTML must render correctly when opened as a standalone file with zero dependencies.
- Do not hallucinate content that wouldn't logically result from the action. If a user taps a toggle, only the toggle and its directly related elements should change -- don't invent new UI elements or alter unrelated content.
- Do not attempt pixel-perfect color matching by guessing hex codes from compressed screenshots. Use standard platform color palettes (Material Design, iOS Human Interface) as approximations.

## Error Handling

**Ambiguous action targets:** If the action coordinates fall between two UI elements or on a non-interactive area, ask the user to clarify which element they intended to interact with rather than guessing.

**Unknown navigation destinations:** When tapping a button that would navigate to a screen not visible in the provided screenshot (e.g., "Settings" from a home screen), generate the predicted screen based on standard platform conventions and clearly note that the prediction is based on typical patterns, not observed content.

**Complex dynamic content:** For actions that trigger animations, loading states, or asynchronous data fetches, generate the final settled state rather than intermediate frames. Note any loading states that would appear transiently.

**Coordinate system mismatch:** If the user provides coordinates that don't match the apparent resolution of the screenshot, normalize coordinates to the detected resolution before mapping to elements.

## Limitations

- **No real data prediction.** The model cannot predict actual content that would be fetched from a server (e.g., new emails, updated stock prices). It can only predict structural and interaction-driven changes.
- **Animation and transition states.** Only the final resting state is generated -- intermediate animation frames, transitions, and gesture-in-progress states are not modeled.
- **Complex custom widgets.** Heavily customized UI components (games, canvas-based drawing, video players) cannot be faithfully represented in static HTML and should be replaced with descriptive placeholders.
- **Multi-step chaining accuracy.** Prediction quality degrades when chaining multiple sequential predictions, as small errors in each step compound. For long interaction sequences, periodically re-anchor to actual screenshots.
- **Platform-specific behaviors.** System-level actions (notifications pulling down, app switching, keyboard appearance) follow generic patterns and may not match the exact behavior of specific device models or OS versions.

## Reference

**Paper:** [Code2World: A GUI World Model via Renderable Code Generation](https://arxiv.org/abs/2602.09856v1) -- Zheng et al., 2026. Key sections: Section 3 (methodology) for the code generation pipeline and visual-feedback revision loop; Section 4 (AndroidCode dataset) for understanding the HTML representation constraints; Section 5 (render-aware RL) for the dual reward structure that balances visual fidelity with action consistency.