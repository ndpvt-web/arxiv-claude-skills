---
name: "3d-space-as-scratchpad-editable"
description: "Build agentic pipelines that use 3D scene layout as an intermediate reasoning workspace for controllable, spatially-accurate image generation. Trigger phrases: 'generate a scene with precise spatial layout', 'place objects in 3D then render', '3D scratchpad image generation', 'editable scene composition', 'spatially accurate text-to-image', 'multi-object scene with depth control'"
---

# 3D Space as a Scratchpad for Editable Text-to-Image Generation

This skill teaches Claude to implement the **3D Scratchpad** pipeline -- a multi-agent system that converts a text prompt into an explicit 3D scene layout before generating the final image. Instead of asking a diffusion model to reason about spatial relationships implicitly, the pipeline externalizes spatial reasoning into a manipulable 3D workspace: it parses subjects from the prompt, creates 3D proxy meshes, arranges them via LLM-driven agentic planning, renders depth maps and identity-preserving cues, and feeds those into a controllable image generator. This yields a 32% improvement in text-alignment over direct generation and makes edits (move, resize, rotate objects) trivial.

## When to Use

- When the user asks to build an image generation pipeline that handles complex multi-object scenes with specific spatial relationships (e.g., "a cat sitting on top of a bookshelf to the left of a window")
- When the user wants editable scene composition -- generating an image, then repositioning or resizing individual objects without re-generating from scratch
- When implementing a multi-agent orchestration system where each agent handles one stage of a visual reasoning pipeline (parsing, layout, refinement, rendering)
- When the user needs depth-conditioned image generation with per-subject identity preservation
- When building a tool that converts natural-language scene descriptions into 3D bounding-box layouts for downstream rendering
- When the user asks how to improve spatial accuracy in text-to-image systems beyond what DALL-E or Flux achieve natively

## Key Technique

The core insight is that **3D space serves as an explicit reasoning substrate** -- analogous to chain-of-thought for language. Rather than asking a VLM or diffusion model to implicitly infer where "a dog is lying next to a fire hydrant with a bird perched on top," the system builds a concrete 3D scene with proxy meshes positioned in world coordinates. This 3D arrangement is then rendered into depth maps and multi-view images that condition the final image generator. Because the intermediate representation is geometric and editable, spatial relationships are exact, not probabilistic.

The pipeline uses **four specialized LLM agents** operating sequentially: (1) a **Subject Decomposer** that parses the prompt into individual subjects plus background, generating per-subject descriptions and an enhanced global prompt; (2) a **BboxPlanner** that places 3D bounding boxes in a coordinate system with defined axes, ground plane, and scale rulers; (3) a **TransformPlanner** (with an OrientationEstimator sub-agent) that refines each mesh's rotation, translation, and scale by analyzing cropped renders; and (4) a **CameraPicker** that selects the best viewpoint from five proposal views. 3D meshes are generated via a text-to-image-to-3D path (Flux for identity images, Hunyuan-3D for mesh conversion), rendered with PyTorch3D, and the final image is produced by SIGMA-Gen conditioned on depth maps and identity images.

What makes this practically powerful is **edit propagation**: to move, resize, or reorient a subject, you modify its 3D transform, re-render the depth map, and regenerate only the affected region using latent blending. A SubjectEditor agent translates natural-language edit instructions (e.g., "move the dog closer to the camera") into concrete 3D transform deltas. ObjectClear handles removal of old subject pixels before re-compositing.

## Step-by-Step Workflow

1. **Parse the prompt into subjects and background.** Use an LLM call to extract each distinct subject (e.g., "golden retriever", "red fire hydrant") with per-subject descriptions, plus a background description and an enhanced global prompt P' that adds spatial detail.

2. **Generate identity images for each subject.** For every parsed subject, call a text-to-image model (e.g., Flux) with the subject description to produce a single canonical reference image. These identity images anchor visual consistency throughout the pipeline.

3. **Convert identity images to 3D proxy meshes.** Pass each identity image through an image-to-3D model (e.g., Hunyuan-3D 2.5) to obtain a textured or color-coded mesh. Assign fixed distinguishable colors if textures are unreliable -- geometry matters more than surface detail at this stage.

4. **Plan 3D bounding boxes via LLM agent.** Feed the enhanced prompt, subject descriptions, identity images, and a 3D coordinate system description (axis labels, ground plane at Y=0, unit scale) to a BboxPlanner agent. It outputs a 3D bounding box (center, dimensions) and a coarse orientation hint (e.g., "facing camera", "lying flat") for each subject.

5. **Place meshes and render initial multi-view images.** Position each mesh within its bounding box using PyTorch3D (or Blender/Three.js). Render from front, left, right, top, and one additional proposal view. Include a gray ground plane and axis rulers as visual grounding cues in the render.

6. **Refine orientation and transforms.** Crop each subject from the rendered views and pass crops to an OrientationEstimator agent that identifies the mesh's current facing direction. A TransformPlanner agent then compares current vs. desired orientation and outputs rotation/translation/scale adjustments. Apply transforms and re-render. Iterate once or twice if needed.

7. **Select the best camera viewpoint.** Present the five proposal renders to a CameraPicker agent, which selects the viewpoint that best matches the prompt's implied perspective and aesthetic quality.

8. **Extract depth map from the chosen viewpoint.** Render the final 3D scene from the selected camera into a depth map. This depth map becomes the spatial conditioning signal for image generation.

9. **Generate the final image with identity-preserving conditioning.** Pass the depth map, per-subject identity images, and enhanced prompt P' to a controllable multi-subject image generator (e.g., SIGMA-Gen with ControlNet-depth). The depth map enforces spatial layout; identity images enforce subject appearance.

10. **Handle edits by modifying 3D transforms and re-rendering.** When the user requests changes (move, scale, rotate, remove a subject), use a SubjectEditor agent to convert the edit instruction into a 3D transform delta. Apply the transform, re-render depth, use ObjectClear to remove the old subject region, and regenerate with latent blending for seamless compositing.

## Concrete Examples

**Example 1: Multi-object scene generation pipeline**

User: "Build me a Python pipeline that generates an image of 'a red sports car parked in front of a white colonial house with a large oak tree to the right' with accurate spatial layout."

Approach:
1. Parse prompt into subjects: `["red sports car", "white colonial house", "large oak tree"]` + background: `"suburban street, daylight"`
2. Generate identity images via Flux API for each subject
3. Convert to meshes via Hunyuan-3D API calls
4. Call GPT-5 as BboxPlanner with structured output schema:
```json
{
  "subjects": [
    {"name": "red sports car", "bbox_center": [0, 0.5, 2], "bbox_size": [2, 1, 4], "orientation": "facing right"},
    {"name": "white colonial house", "bbox_center": [0, 3, 8], "bbox_size": [8, 6, 6], "orientation": "facing camera"},
    {"name": "large oak tree", "bbox_center": [5, 4, 7], "bbox_size": [4, 8, 4], "orientation": "upright"}
  ],
  "camera_hint": "eye-level, slightly angled from left"
}
```
5. Render scene with PyTorch3D, extract depth map
6. Call SIGMA-Gen with depth conditioning + identity images
7. Return final image + the 3D scene state as editable JSON

Output structure:
```python
class SceneState:
    subjects: list[SubjectMesh]  # Each has .transform, .identity_image, .mesh
    camera: CameraParams         # position, look_at, fov
    depth_map: np.ndarray
    final_image: PIL.Image

    def edit(self, instruction: str) -> "SceneState":
        """Apply natural-language edit via SubjectEditor agent."""
```

**Example 2: Editable scene with iterative refinement**

User: "I generated a scene but the dog is too far from the bench. Move it closer and make it face the bench instead of the camera."

Approach:
1. Load the existing `SceneState` with current 3D transforms
2. Call SubjectEditor agent with edit instruction and current scene metadata:
```
SubjectEditor(
    edit="move the dog closer to the bench and face it toward the bench",
    current_scene=scene_state,
    subject="dog"
)
```
3. Agent returns transform delta: `{"translate": [-1.0, 0, 0.5], "rotate_y": 90}`
4. Apply transform to the dog mesh, re-render depth map
5. Use ObjectClear to erase old dog region from previous image
6. Regenerate with latent blending -- only the dog region updates

Output: Updated image with the dog repositioned, rest of scene unchanged.

**Example 3: Orchestrating the multi-agent system**

User: "How should I architect the agent coordination for the 3D scratchpad pipeline?"

Approach:
1. Define four agent roles with clear input/output contracts:
```python
agents = {
    "decomposer": {
        "model": "gpt-5",
        "input": "raw_prompt",
        "output": {"subjects": list, "background": str, "enhanced_prompt": str}
    },
    "bbox_planner": {
        "model": "gpt-5",
        "input": ["enhanced_prompt", "subject_descriptions", "identity_images", "3d_space_desc"],
        "output": {"bounding_boxes": list, "orientation_hints": list}
    },
    "transform_planner": {
        "model": "gpt-5",
        "input": ["enhanced_prompt", "rendered_crops", "current_orientations", "desired_orientations"],
        "output": {"transforms": list}  # per-subject rotation, translation, scale
    },
    "camera_picker": {
        "model": "gpt-4o",
        "input": ["enhanced_prompt", "proposal_renders"],
        "output": {"selected_view_index": int}
    }
}
```
2. Run agents sequentially -- each depends on the prior's output
3. Use structured output (JSON schema enforcement) on every LLM call to ensure parseable results
4. Include the 3D coordinate system description in every planning agent's system prompt:
```
"The scene uses a right-handed coordinate system. X is right, Y is up, Z is toward camera.
Ground plane is at Y=0. Units are meters. Rulers mark 1m intervals on each axis."
```

## Best Practices

- **Do:** Always include the coordinate system description and ground-plane reference in every spatial planning agent's context. Without this grounding, LLMs produce inconsistent coordinate outputs.
- **Do:** Use cropped per-subject renders (not full scene) when asking agents to estimate orientation. Full-scene images overwhelm the agent and reduce accuracy.
- **Do:** Generate identity images before mesh creation. The text-to-image-to-3D path preserves subject identity far better than direct text-to-3D.
- **Do:** Assign fixed, distinguishable solid colors to proxy meshes when texture quality is low. The pipeline needs geometric accuracy, not texture fidelity, at the layout stage.
- **Avoid:** Asking the camera-selection agent to output raw camera coordinates. Instead, render five diverse viewpoints and let the agent pick from them -- selection is far more reliable than coordinate generation.
- **Avoid:** Regenerating the entire image for small edits. Use ObjectClear for removal + latent blending for local regeneration. This preserves consistency in unedited regions.

## Error Handling

- **Mesh generation failure:** If Hunyuan-3D produces a degenerate mesh (near-zero volume, inverted normals), fall back to a simple primitive (box, sphere) scaled to the bounding box. The depth map still conveys correct spatial relationships.
- **BboxPlanner outputs overlapping boxes:** Add a validation pass that checks for intersection volumes. If overlap exceeds 20% of either box, call the planner again with an explicit constraint: "Subjects must not overlap. Current overlap detected between X and Y."
- **Orientation refinement diverges:** Cap the transform-refinement loop at 2 iterations. If orientation is still wrong after 2 rounds, accept the best result -- over-iteration degrades quality.
- **Identity images don't match prompt intent:** If the generated identity image misrepresents the subject (wrong breed, wrong color), regenerate with a more specific prompt or let the user supply a reference image.
- **Depth map artifacts:** If PyTorch3D produces z-fighting or clipping artifacts, adjust near/far clip planes to tightly bound the scene. Use logarithmic depth if objects span a wide depth range.

## Limitations

- **Requires external model APIs:** The pipeline depends on multiple models (GPT-5 for planning, Flux for identity images, Hunyuan-3D for meshes, SIGMA-Gen for final generation). Latency and cost scale with subject count.
- **Proxy meshes are coarse:** Auto-generated meshes may miss fine geometric details (e.g., a bicycle's spokes). The depth map reflects proxy geometry, so the final image must compensate via the diffusion model's learned priors.
- **Background handling is limited:** Backgrounds are described textually, not modeled in 3D. Complex background geometry (e.g., "a winding staircase behind the subjects") may still be spatially inaccurate.
- **Single-viewpoint output:** The system selects one camera view for the final image. It does not produce multi-view consistent outputs or 3D-renderable scenes directly.
- **Edit scope:** Edits that require adding entirely new subjects or drastically changing the scene composition effectively restart the pipeline. Only transform-level edits (move, rotate, scale, remove) are cheap.

## Reference

**Paper:** [3D Space as a Scratchpad for Editable Text-to-Image Generation](https://arxiv.org/abs/2601.14602) -- Saha et al., 2026. Focus on Section 3 (Method) for the four-agent architecture and Section 4 for GenAI-Bench evaluation showing 32% text-alignment improvement over Flux baseline.