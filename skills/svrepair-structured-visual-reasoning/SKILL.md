---
name: "svrepair-structured-visual-reasoning"
description: "Fix bugs using structured visual reasoning -- converts screenshots, control-flow graphs, and UI artifacts into semantic scene graphs for precise fault localization and patch generation. Use when: 'fix this UI bug from the screenshot', 'the layout is broken, here is what it looks like', 'debug this using the control-flow graph', 'repair the rendering issue shown in this image', 'use the screenshot to find what went wrong', 'convert this visual bug report into a code fix'."
---

# SVRepair: Structured Visual Reasoning for Automated Program Repair

This skill enables Claude to fix software bugs by extracting structured, actionable information from visual artifacts -- screenshots, control-flow graphs, UI renderings, and flowcharts. Rather than feeding raw images directly into the reasoning process (which causes context loss and hallucination), this technique first converts visual inputs into **Semantic Scene Graphs (SSGs)**: directed graphs where nodes represent visual elements (buttons, text fields, CFG basic blocks) and edges represent relationships (control flow, data flow, compositional hierarchy). The SSG is serialized as Mermaid syntax, giving Claude a precise, code-relevant representation that bridges visual observations and executable repairs.

## When to Use

- When a user shares a **screenshot of a UI bug** (layout breakage, missing widget, misaligned elements) and asks you to find and fix the code causing it
- When a user provides a **control-flow graph** or flowchart and asks you to debug execution path issues
- When a bug report contains **visual artifacts** (rendered HTML, GUI screenshots, error state images) alongside code
- When fixing **front-end rendering bugs** where the visual output diverges from expected behavior
- When a user says "here's what it looks like" and provides an image showing incorrect application behavior
- When debugging **widget state bugs** (disabled buttons that should be enabled, incorrect form validation visuals, missing elements)
- When translating visual flowcharts or diagrams into working code implementations

## Key Technique: Semantic Scene Graphs

The core insight from SVRepair is that raw visual inputs are too dense and noisy for reliable bug repair. A screenshot of a web page contains thousands of pixels, but only a small region is relevant to the bug. Feeding the entire image causes the model to lose focus on the critical signal. The solution is a two-stage abstraction:

**Stage 1 -- Visual Abstraction to SSG:** Convert the visual artifact into a Semantic Scene Graph G=(V, E), where nodes V are discrete visual elements (HTML elements, CFG basic blocks, UI components) and edges E are typed directional relationships (`contains`, `flows-to`, `sibling-of`, `data-depends`). This graph is serialized in Mermaid syntax so it can be processed as structured text. For a screenshot, this means identifying the DOM-like hierarchy of visible elements. For a control-flow graph, this means extracting basic blocks and their execution edges.

**Stage 2 -- Iterative Sub-Artifact Segmentation:** If the initial SSG is still too broad, narrow the focus. Identify the bounding region of the bug within the visual artifact (e.g., the specific form section with the broken widget), extract a sub-artifact, and rebuild a focused SSG from just that region. This iterative narrowing (up to 3 rounds) eliminates noise and concentrates reasoning on the fault location.

**Stage 3 -- SSG-Guided Localization and Repair:** Use the SSG node labels and relationships to derive search queries into the codebase. An SSG node labeled "FromName inside Origin container" directly maps to file paths and component names. This enables precise fault localization followed by targeted SEARCH/REPLACE patch generation.

## Step-by-Step Workflow

1. **Receive and classify the visual artifact.** Determine whether the input is a screenshot (UI rendering), a control-flow graph, a flowchart, or another visual diagnostic. Note the artifact type, as each requires different element extraction.

2. **Build the Semantic Scene Graph (SSG).** Analyze the visual artifact and extract a structured graph:
   - For **screenshots/UI renderings**: identify visible GUI elements (buttons, inputs, containers, text labels, images), their bounding hierarchy (which contains which), and their visual states (disabled, error, hidden, misaligned).
   - For **control-flow graphs**: identify basic blocks (code regions), branch conditions, loop edges, and execution paths. Capture entry/exit nodes.
   - For **flowcharts/diagrams**: identify process nodes, decision nodes, and directional edges.

3. **Serialize the SSG as Mermaid syntax.** Convert the graph into a Mermaid diagram that captures nodes, edges, and relationship types. This becomes the structured representation used for all downstream reasoning.

   ```mermaid
   graph TD
     A[Page Container] -->|contains| B[Header]
     A -->|contains| C[Form: Origin]
     C -->|contains| D[Input: FromName - ERROR_STATE]
     C -->|contains| E[Input: ToName - OK]
     C -->|contains| F[Button: Submit - DISABLED]
     D -->|triggers| F
   ```

4. **Identify the bug-relevant subgraph.** If the SSG is large, isolate the portion related to the reported bug. Look for nodes with anomalous states (ERROR_STATE, MISSING, MISALIGNED) or unexpected edge patterns (missing control-flow edges, orphaned nodes). This is the sub-artifact segmentation step.

5. **Map SSG nodes to codebase locations.** Use element names, labels, CSS classes, component names, or block identifiers from the SSG to search the codebase. For a node labeled `Input: FromName`, search for component files containing "FromName", "from-name", or related identifiers. For CFG blocks, match function names and line ranges.

6. **Localize the fault in source code.** Using the SSG-to-code mapping, read the relevant source files. Compare the expected behavior (from the SSG's structural relationships) against the actual code logic. Identify where the code diverges from what the visual artifact reveals.

7. **Generate a targeted patch.** Write a SEARCH/REPLACE edit that fixes the localized fault. The patch should be minimal -- change only what is necessary to resolve the visual discrepancy. Avoid broad modifications that introduce side effects.

8. **Validate the patch against the visual expectation.** Mentally trace the patch through the SSG: does the fix restore the expected node states and edge relationships? If the user can run tests, recommend doing so. If not, explain why the patch resolves the visual bug by referencing specific SSG nodes.

9. **Iterate if the fix is insufficient.** If validation fails or the user reports the bug persists, refine the sub-artifact focus. Zoom into a tighter region of the visual artifact, rebuild the SSG at higher granularity, and repeat localization. Allow up to 3 refinement rounds before escalating to a broader investigation.

10. **Document the visual-to-code mapping.** In your response, explicitly state which visual elements mapped to which code locations. This traceability helps the user understand the fix and verify it independently.

## Concrete Examples

**Example 1: UI layout bug from screenshot**

User: "This NumberInput component should be read-only but users can still type in it. Here's a screenshot of the form."

Approach:
1. Analyze the screenshot and build an SSG identifying the NumberInput element, its parent form container, and sibling elements (increment/decrement buttons, label).
2. Serialize as Mermaid:
   ```mermaid
   graph TD
     Form[Form Container] -->|contains| NI[NumberInput - EDITABLE]
     Form -->|contains| Inc[Button: Increment]
     Form -->|contains| Dec[Button: Decrement]
     NI -->|missing_attr| RO[readOnly: not present]
   ```
3. The SSG reveals `NumberInput` lacks a `readOnly` attribute despite the requirement. Search codebase for the NumberInput component definition.
4. Locate `components/NumberInput.jsx` -- find that the `readOnly` prop is accepted but never forwarded to the underlying `<input>` element.
5. Generate patch:
   ```
   SEARCH: <input type="number" value={value} onChange={handleChange} />
   REPLACE: <input type="number" value={value} onChange={handleChange} readOnly={readOnly} />
   ```

Output: A single-line fix that forwards the `readOnly` prop to the native input, preventing user editing when the component is in read-only mode. The SSG node `NumberInput - EDITABLE` should transition to `NumberInput - READONLY`.

---

**Example 2: Control-flow graph reveals unreachable error handler**

User: "Here's the control-flow graph for our payment processing function. Error transactions are not being logged."

Approach:
1. Parse the CFG into an SSG with basic blocks as nodes and control-flow edges:
   ```mermaid
   graph TD
     Entry[processPayment entry] -->|true| Validate[validateCard]
     Validate -->|success| Charge[chargeCard]
     Validate -->|failure| EarlyReturn[return false]
     Charge -->|success| Log[logTransaction]
     Charge -->|failure| EarlyReturn
     Log --> Exit[return true]
   ```
2. Identify the anomaly: the `Charge -->|failure| EarlyReturn` edge bypasses `logTransaction`. Error transactions are never logged because the failure path skips directly to return.
3. Search for `processPayment` in the codebase. Find the early return on charge failure at line 47.
4. Generate patch that routes charge failures through a logging step before returning:
   ```
   SEARCH:
     if (!chargeResult.success) {
       return false;
     }
   REPLACE:
     if (!chargeResult.success) {
       logTransaction(cardInfo, chargeResult, 'FAILED');
       return false;
     }
   ```

Output: The fix adds error-path logging. The corrected CFG now has `Charge -->|failure| LogError[logTransaction FAILED] --> EarlyReturn`, ensuring all transactions are logged regardless of outcome.

---

**Example 3: Visual flowchart to code implementation**

User: "Convert this flowchart diagram into a Python function for user authentication."

Approach:
1. Build SSG from the flowchart, identifying decision nodes (diamonds), process nodes (rectangles), and terminal nodes:
   ```mermaid
   graph TD
     Start[Start] --> CheckToken{Token valid?}
     CheckToken -->|yes| CheckExpiry{Token expired?}
     CheckToken -->|no| Login[Prompt login]
     CheckExpiry -->|yes| Refresh[Refresh token]
     CheckExpiry -->|no| Grant[Grant access]
     Refresh -->|success| Grant
     Refresh -->|failure| Login
     Login -->|success| Grant
     Login -->|failure| Deny[Deny access]
   ```
2. Map each node to a code block. Decision nodes become `if/else`, process nodes become function calls, terminals become return statements.
3. Generate implementation following the SSG topology:
   ```python
   def authenticate(token: str | None) -> bool:
       if token and is_valid(token):
           if is_expired(token):
               refreshed = refresh_token(token)
               if not refreshed:
                   return prompt_login()
               return True
           return True
       return prompt_login()
   ```

Output: A function whose control flow exactly mirrors the flowchart's SSG structure, with each graph path corresponding to a code path.

## Best Practices

- **Do:** Always build the SSG explicitly before attempting repairs. Writing out the graph structure forces precise identification of elements and relationships, preventing vague or misdirected fixes.
- **Do:** Use Mermaid syntax for SSG serialization. It is compact, human-readable, and keeps the structured representation in a form that supports further reasoning.
- **Do:** Narrow the visual focus iteratively. Start with the full artifact, then zoom into the bug-relevant region. Each iteration should halve the number of SSG nodes under consideration.
- **Do:** Map SSG nodes to code by searching for element names, component identifiers, CSS classes, and function names extracted from the graph.
- **Avoid:** Feeding raw screenshots directly into repair reasoning without first building a structured representation. This causes context dilution and hallucinated fixes.
- **Avoid:** Making broad code changes based on visual artifacts. The SSG should point to a specific, narrow fault location. If your patch touches more than 2-3 code regions, re-examine whether the SSG was precise enough.
- **Avoid:** Guessing UI element relationships. If the visual artifact is ambiguous, ask the user to clarify the element hierarchy or provide additional context (DOM tree, component tree, HTML source).

## Error Handling

| Problem | Cause | Resolution |
|---|---|---|
| SSG contains too many irrelevant nodes | Visual artifact is too broad | Apply sub-artifact segmentation: crop to the bug region and rebuild the SSG |
| Cannot map SSG nodes to code | Element names are generic (e.g., "div", "container") | Ask the user for the component tree, inspect CSS classes, or request the HTML source for cross-referencing |
| Patch fixes one visual issue but introduces another | SSG missed a dependency between elements | Expand the SSG to include sibling and parent nodes of the patched element, then check for cascading effects |
| Control-flow SSG has ambiguous branching | CFG image is low-resolution or overlapping | Ask for the source code of the function to reconstruct the CFG textually instead of visually |
| Iterative refinement exceeds 3 rounds without resolution | Bug is not visually localizable | Fall back to traditional text-based debugging -- read logs, stack traces, and test output instead |

## Limitations

- **Non-visual bugs:** This technique adds no value for bugs that have no visual manifestation (e.g., race conditions, memory leaks, pure logic errors with no UI or CFG artifact). Use standard debugging for those.
- **Image quality dependency:** Low-resolution screenshots or hand-drawn diagrams may produce inaccurate SSGs. The quality of the structured representation is bounded by the clarity of the input artifact.
- **Dynamic UI states:** Screenshots capture a single moment. Bugs involving animations, transitions, or state changes across time cannot be fully represented in a static SSG.
- **Large-scale applications:** For applications with hundreds of UI components, even a focused sub-artifact may produce an SSG too large for effective reasoning. Pair with component-tree filtering.
- **No automated visual validation:** Claude cannot re-render the patched code to visually confirm the fix. The user must verify the visual output after applying the patch.

## Reference

**Paper:** [SVRepair: Structured Visual Reasoning for Automated Program Repair](https://arxiv.org/abs/2602.06090v1) (Tang et al., 2026)

Key takeaway: Converting visual artifacts into Semantic Scene Graphs (SSGs) serialized as Mermaid diagrams improves bug localization accuracy by eliminating visual noise and creating explicit element-to-code mappings. The iterative sub-artifact segmentation strategy (narrowing focus over up to 3 rounds) is critical for handling complex UIs. On SWE-Bench M, this approach achieved 36.47% pass@1, outperforming both vision-only and text-only baselines.