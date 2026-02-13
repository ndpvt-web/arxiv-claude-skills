---
name: "chatting-images-introspective-visual"
description: "Apply introspective visual thinking by iteratively 'chatting with images' — using language-guided re-examination of visual content to reason over fine-grained details, spatial relationships, and multi-image comparisons. Use when: 'analyze this image in detail', 'compare these images', 'reason about spatial layout', 'what's different between these screenshots', 'explain the visual relationship', 'trace the visual logic step by step'."
---

# Chatting with Images: Introspective Visual Thinking

This skill enables Claude to perform **iterative, language-guided visual reasoning** instead of relying on a single-pass description of an image. Inspired by the "chatting with images" framework from Wu et al. (2026), the technique treats visual analysis as a multi-turn dialogue between linguistic reasoning and visual re-examination. Rather than describing an image once and reasoning purely from that text, Claude iteratively formulates targeted visual questions, re-examines specific image regions with those questions in mind, and refines its understanding — producing tighter coupling between what it sees and what it reasons about.

## When to Use

- When the user asks Claude to **compare two or more images** and identify differences, similarities, or relationships across them.
- When a task requires **fine-grained spatial reasoning** — e.g., "Is object A above or to the left of object B?" or "Which element is closest to the center?"
- When analyzing **complex diagrams, charts, or UI screenshots** where a single-pass description would lose important details.
- When the user asks to **trace a visual process** across multiple frames (video stills, sequential screenshots, step-by-step diagrams).
- When visual reasoning requires **cross-referencing distant regions** of a large or detailed image — e.g., comparing a legend to data points in a chart.
- When debugging **UI layout issues** by examining screenshots and identifying misalignments, overlaps, or spacing problems.
- When the user provides an image and asks a question whose answer requires reasoning over geometric relationships, relative positions, or subtle visual features that are easy to miss.

## Key Technique

### From Single-Pass to Iterative Visual Dialogue

Standard vision-language reasoning follows a pipeline: encode the image once, convert it to tokens, then reason in text. This loses fine-grained information — once the visual encoding is fixed, details not captured in that initial pass are gone. "Thinking with images" approaches attempt to fix this by calling external tools (cropping, annotating, running code on images), but the resulting visual states are disconnected from the linguistic reasoning context.

**Chatting with images** takes a different approach: instead of manipulating pixels, it treats visual re-examination as **language-guided feature modulation**. The model formulates an explicit linguistic query ("What color is the label in the top-right corner?"), then re-examines the relevant image region with that query as a lens. This produces a new visual understanding that is tightly coupled to the reasoning chain. The key insight is that *what you're looking for changes what you see* — a targeted re-examination guided by a specific question extracts different information than an open-ended first pass.

### Dynamic Re-Encoding in Practice

The ViLaVT model implements this via a dynamic vision encoder that performs **joint re-encoding** of image regions conditioned on language prompts. For Claude (which doesn't have a trainable vision encoder), we approximate this by structuring the reasoning process as an explicit multi-pass protocol: (1) initial broad examination, (2) formulation of targeted follow-up visual queries based on what's needed, (3) focused re-examination of specific regions guided by those queries, (4) synthesis of findings. This mirrors the paper's two-stage training — first learning to describe (SFT), then learning *when and what* to re-examine (RL-driven reasoning behaviors).

## Step-by-Step Workflow

1. **Perform an initial broad scan of the image(s).** Describe the overall layout, major elements, and general structure. Do NOT try to answer the user's question yet — this pass is for orientation only.

2. **Identify what the question actually requires visually.** Decompose the user's question into specific visual sub-questions. For "Are these two UIs the same?", the sub-questions might be: "What elements are in UI-A?", "What elements are in UI-B?", "Do they share the same layout grid?", "Are colors/fonts identical?"

3. **Formulate targeted re-examination prompts.** For each sub-question, write an explicit directive like: "Focus on the top-left quadrant and list every text label and its approximate position." These prompts serve as the language guidance for visual feature modulation.

4. **Re-examine specific image regions with each prompt in mind.** Look at the image again, but this time constrained to the specific region and specific visual feature your prompt targets. Record observations with precise spatial language (coordinates, relative positions, directional relationships).

5. **Cross-reference observations across regions or images.** If comparing multiple images or distant regions, explicitly align your observations: "In Image A, the button is at top-right; in Image B, it's at center-left." Use a structured comparison format.

6. **Check for contradictions or gaps.** Review your accumulated observations. If any sub-question remains unanswered, or if observations conflict, perform another targeted re-examination pass on the specific area of uncertainty.

7. **Synthesize findings into a coherent answer.** Combine all observations, explicitly linking each claim to the specific visual evidence gathered during re-examination passes.

8. **State confidence and residual uncertainty.** Flag any aspects where the image resolution, occlusion, or ambiguity prevented a definitive answer.

## Concrete Examples

**Example 1: Comparing Two UI Screenshots**

```
User: [Provides two screenshots] What changed between version A and version B of this settings page?

Approach:
1. INITIAL SCAN: "Image A shows a settings page with a sidebar on the left,
   main content area with form fields, and a save button at the bottom.
   Image B has the same general layout."

2. DECOMPOSE INTO SUB-QUESTIONS:
   - Are the sidebar items the same in both?
   - Are the form fields identical (labels, order, types)?
   - Has the save button changed (position, style, text)?
   - Are there any new or removed elements?

3. TARGETED RE-EXAMINATION:
   Prompt: "Examine the sidebar items in Image A, listing each from top to bottom."
   → "Profile, Security, Notifications, Billing, Advanced"
   Prompt: "Examine the sidebar items in Image B, listing each from top to bottom."
   → "Profile, Security, Notifications, Integrations, Billing, Advanced"

   Prompt: "Examine the form field labels in the main content area of Image A."
   → "Display Name, Email, Bio (text area), Timezone (dropdown)"
   Prompt: "Examine the form field labels in the main content area of Image B."
   → "Display Name, Email, Bio (text area), Language (dropdown), Timezone (dropdown)"

4. CROSS-REFERENCE:
   - Sidebar: Image B added "Integrations" between Notifications and Billing.
   - Form fields: Image B added "Language" dropdown before Timezone.
   - Save button: unchanged in both.

Output:
Two changes between versions:
1. **New sidebar item**: "Integrations" was added between "Notifications" and "Billing."
2. **New form field**: A "Language" dropdown was added above the existing "Timezone" dropdown.
No other visual differences detected in layout, styling, or other elements.
```

**Example 2: Spatial Reasoning in a Floor Plan**

```
User: [Provides floor plan image] Can someone in the kitchen see
      directly into the master bedroom?

Approach:
1. INITIAL SCAN: Identify room locations and overall layout shape.

2. TARGETED RE-EXAMINATION:
   Prompt: "Locate the kitchen — what are its boundaries and door/opening positions?"
   → "Kitchen is in the northwest corner. Open archway faces east into the
      living room. No other openings."

   Prompt: "Locate the master bedroom — what are its boundaries and
   door positions?"
   → "Master bedroom is in the southeast corner. Single door opens north
      into the hallway."

   Prompt: "Trace the line of sight from the kitchen archway toward
   the master bedroom door. What walls or rooms intervene?"
   → "The kitchen archway opens east into the living room. The living room's
      south wall is solid. The hallway to the master bedroom branches south
      from the living room's east end, turns east, then reaches the
      master bedroom door — not a straight line from the kitchen."

3. SYNTHESIS: No direct line of sight exists. The kitchen archway faces
   east into the living room, but the master bedroom is accessed via a
   hallway that turns, with intervening walls blocking any direct view.

Output:
No — someone in the kitchen cannot see directly into the master bedroom.
The kitchen opens east into the living room via an archway, but the master
bedroom is accessed through a hallway that turns south and then east from
the living room, with solid walls blocking any direct sightline.
```

**Example 3: Multi-Frame Visual Sequence Analysis**

```
User: [Provides 4 sequential screenshots of an animation] What is the
      trajectory of the red circle across these frames?

Approach:
1. INITIAL SCAN: Four frames showing a canvas with geometric shapes.

2. TARGETED RE-EXAMINATION per frame:
   Frame 1: "Red circle is at approximately (100, 300) — bottom-left area."
   Frame 2: "Red circle is at approximately (250, 200) — center, moved
             up and right."
   Frame 3: "Red circle is at approximately (400, 150) — upper-right area,
             continued same direction."
   Frame 4: "Red circle is at approximately (400, 300) — right side,
             dropped back down to original y."

3. CROSS-REFERENCE positions into trajectory:
   (100,300) → (250,200) → (400,150) → (400,300)
   Pattern: moves right and up for frames 1-3, then drops vertically
   in frame 4.

Output:
The red circle follows an arcing trajectory:
- Frames 1→3: Diagonal movement up and to the right (bottom-left to
  upper-right), covering ~300px horizontally and ~150px vertically.
- Frame 3→4: Vertical drop of ~150px with no horizontal movement.
Overall shape: an inverted-L or a throw-and-drop arc.
```

## Best Practices

- **Do:** Always perform an initial orientation pass before diving into details. You cannot ask targeted questions about regions you haven't broadly located yet.
- **Do:** Write out your re-examination prompts explicitly in your reasoning. The act of formulating "What specifically am I looking for?" is what makes the second pass more informative than the first.
- **Do:** Use precise spatial vocabulary — "top-left quadrant," "approximately 30% from the left edge," "directly below the header" — rather than vague terms like "near" or "around."
- **Do:** When comparing images, always examine the same region/feature in both images before moving to the next, rather than fully describing one image then the other. This catches differences more reliably.
- **Avoid:** Trying to answer spatial or comparative questions from a single pass. If you catch yourself making a claim about relative position or a difference without having explicitly re-examined both relevant areas, stop and re-examine.
- **Avoid:** Over-relying on general descriptions. "The page looks mostly the same" is not a finding — enumerate specific elements checked and confirmed as identical.

## Error Handling

- **Image resolution too low for fine details:** State explicitly what you cannot resolve. "The text in the bottom status bar is too small to read at this resolution — I can see there is text but cannot determine its content." Suggest the user provide a cropped/zoomed version of the region of interest.
- **Ambiguous spatial relationships:** When two elements are close enough that their relative position is uncertain, say so: "The red and blue markers appear to be at approximately the same y-coordinate; I cannot confidently determine which is higher without measurement."
- **Multiple images with inconsistent scale/orientation:** Flag this before comparing: "Image A appears to be at 2x zoom compared to Image B. I'll account for this when comparing element sizes."
- **Reasoning chain grows too long:** If after 3-4 re-examination passes you still cannot answer the question, summarize what you've established so far and identify the specific visual information that remains unresolved.

## Limitations

- This technique is an **approximation** of the paper's dynamic vision encoder. Claude cannot literally re-encode image features conditioned on language — instead, it structures its reasoning to emulate iterative re-examination. This works well for explicit spatial reasoning but cannot recover visual details that are genuinely below Claude's perceptual resolution.
- **Not a substitute for image processing tools.** If the task requires pixel-level measurement, color-space analysis, or programmatic image manipulation, use actual tools (Python with PIL/OpenCV). This skill is for *reasoning* about visual content, not processing it.
- **Diminishing returns on re-examination.** A second or third focused pass genuinely helps. A sixth or seventh pass on the same region rarely adds information — if multiple passes haven't resolved the question, the limitation is perceptual, not attentional.
- **Video and long sequences.** The technique works well for comparing a small number of frames (2-8). For longer sequences, summarize in batches rather than attempting to cross-reference all frames simultaneously.

## Reference

Wu, J., Guan, J., Liu, Q., Wu, S., & Wang, L. (2026). *Chatting with Images for Introspective Visual Thinking.* arXiv:2602.11073. [https://arxiv.org/abs/2602.11073v1](https://arxiv.org/abs/2602.11073v1)

Key takeaway from the paper: Visual reasoning improves substantially when the model iteratively re-examines images guided by its own evolving linguistic reasoning, rather than relying on a single initial encoding. The "chatting with images" framing — treating each re-examination as a language-guided query to the visual input — achieves tighter cross-modal alignment than either text-only reasoning or tool-based image manipulation.