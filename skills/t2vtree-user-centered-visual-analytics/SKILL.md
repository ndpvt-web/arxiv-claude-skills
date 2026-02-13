---
name: "t2vtree-user-centered-visual-analytics"
description: >
  Build tree-structured, agent-assisted thought-to-video authoring systems where
  each generation step is a node binding intent, prompts, parameters, and outputs.
  Four collaborating agents (Master, Knowledge, Workflow, Prompt) translate user
  intent into editable executable plans. Supports branching exploration, provenance
  tracking, and convergent stitching assembly.
  Trigger phrases: "build a video authoring pipeline", "tree-based video generation",
  "agent-assisted video creation", "thought-to-video workflow", "branching video
  exploration system", "multi-scene video authoring with agents"
---

# T2VTree: Tree-Structured Agent-Assisted Thought-to-Video Authoring

This skill enables Claude to design and implement **tree-structured, agent-assisted video authoring systems** based on the T2VTree methodology. The core idea: represent the entire thought-to-video creation process as a tree where each node binds an editable specification (user intent, reference inputs, workflow choice, prompts, parameters) with its resulting multimodal outputs. Four collaborating agents decompose high-level creative intent into inspectable, user-editable execution plans before any generation runs. This makes branching exploration, localized comparison, provenance tracking, and convergent assembly into a final video all first-class operations rather than afterthoughts.

## When to Use

- When the user wants to build a **multi-step video generation pipeline** with branching and refinement capabilities
- When designing a system where users explore **alternative creative directions** (image/video/audio variants) without losing prior work
- When implementing **multi-agent planning** that translates natural-language intent into executable generation workflows (e.g., text-to-image, image-to-video, audio generation)
- When building a **visual analytics interface** for managing complex generative AI workflows with provenance
- When the user asks to create a **multi-scene video authoring tool** that stitches outputs from different generation branches
- When implementing **tree-based state management** for any iterative creative AI process (not limited to video)

## Key Technique

**Tree-as-authoring-history.** T2VTree models the entire creative process as a rooted tree on an infinite canvas. The root is an `Init` node. Each user action (generate an image, refine a prompt, produce a video clip) creates a child node extending its parent. Trying alternatives from the same starting point creates sibling nodes. This means every exploration trace is preserved, comparable, and resumable. Parent-child edges represent progressive refinement; sibling edges represent parallel alternatives. Nodes are color-coded by output modality: blue for images, green for video, red for audio.

**Four-agent collaborative planning.** Rather than dumping the user into raw workflow configuration, T2VTree interposes four specialized agents between intent and execution. The **Master Agent** reads scene-level context and proposes an action category (e.g., "generate background image," "animate character"). The **Knowledge Agent** resolves ambiguous entities or concepts. The **Workflow Agent** selects a compatible generation module given the action category and available inputs. The **Prompt Agent** materializes a modality-aware specification: prompt text, negative prompts, numerical parameters (steps, CFG scale, resolution), and seed values. Critically, the resulting plan surfaces as an editable node artifact — the user inspects and revises before triggering execution. This keeps agents assistive, not autonomous.

**Convergent stitching assembly.** After divergent exploration across branches, users collect candidate assets (images, video clips, audio tracks) from any node in the tree into a multi-track timeline. Drag-and-drop ordering produces a final linear video while maintaining provenance links back to originating tree nodes, so upstream revision propagates without reconstructing intermediate steps.

## Step-by-Step Workflow

### 1. Define the Node Data Model

Each tree node must store exactly these five specification fields plus outputs:

```typescript
interface T2VNode {
  id: string;
  parentId: string | null;
  intent: string;                    // Natural-language goal for this step
  referencedInputs: AssetRef[];      // Prior outputs or uploaded assets used as input
  workflowChoice: {
    actionCategory: string;          // e.g., "text-to-image", "image-to-video", "audio-gen"
    moduleId: string;                // Specific pipeline/model identifier
  };
  prompts: {
    positive: string;
    negative: string;
  };
  parameters: Record<string, number | string>;  // CFG scale, steps, resolution, seed, etc.
  outputs: MultimodalOutput[];       // Generated images, videos, or audio
  modality: "image" | "video" | "audio";
  createdAt: string;
  metadata: Record<string, any>;     // Agent planning trace, timing, model info
}
```

### 2. Implement the Tree Structure with Branching Semantics

Build the tree with three operations:
- **Extend** (create child): progressive refinement from a selected node
- **Branch** (create sibling): explore an alternative from the same parent context
- **Prune**: collapse or delete abandoned branches while preserving the rest

Store nodes in a flat collection keyed by `id`, with `parentId` forming the tree. Render using a hierarchical layout algorithm (e.g., Reingold-Tilford or layered DAG layout).

### 3. Implement the Four-Agent Planning Pipeline

Wire up four LLM-based agents that execute sequentially on each user action:

```python
def plan_next_step(scene_context: str, current_path: list[T2VNode], user_intent: str):
    # Agent 1: Master — classify the action
    action_category = master_agent.classify(scene_context, current_path, user_intent)

    # Agent 2: Knowledge — resolve ambiguous references
    clarifications = knowledge_agent.resolve(user_intent, scene_context)

    # Agent 3: Workflow — select generation module
    workflow_spec = workflow_agent.select(action_category, available_inputs(current_path))

    # Agent 4: Prompt — materialize full specification
    node_spec = prompt_agent.materialize(
        intent=user_intent,
        action=action_category,
        workflow=workflow_spec,
        clarifications=clarifications,
        style_context=scene_context
    )
    return node_spec  # User reviews and edits BEFORE execution
```

Each agent call uses ~500-1000 tokens. Keep planning calls lightweight (2k-4k tokens total per step) since generation is the expensive part.

### 4. Make Agent Plans Editable Before Execution

Never auto-execute agent output. Present the generated `node_spec` as an editable form:
- Show the proposed prompt text (positive and negative) in editable text fields
- Expose numerical parameters (CFG scale, steps, resolution, seed) as sliders/inputs
- Display the selected workflow module with alternatives available via dropdown
- Include the Knowledge Agent's clarifications as collapsible context

### 5. Execute Generation and Attach Outputs to the Node

After the user confirms or edits the plan, dispatch to the appropriate generation backend. Attach all outputs (thumbnails, full-resolution assets, metadata) to the node. Support in-place preview: image thumbnails, looping video playback, and audio waveform display directly within the tree node.

### 6. Enable Provenance Inspection

Every node retains its full specification context. Implement a detail panel that shows:
- What the user originally requested (intent)
- What the agents proposed vs. what was actually executed (diff view)
- Which upstream outputs were referenced as inputs
- Generation timing and model configuration

### 7. Implement the Stitching/Assembly View

Build a separate timeline view for convergent assembly:
- Users star/collect candidate assets from any tree node
- Multi-track timeline with video, image (as stills), and audio lanes
- Drag-and-drop ordering with transition controls
- Each segment links back to its source node for upstream editing
- Export produces the final concatenated video

### 8. Maintain Scene-Level Context

Store global scene context at the project root: base model preferences, style direction, mood, color palette, and reference materials. Pass this context into every agent planning call so that generations across different branches maintain visual coherence.

## Concrete Examples

**Example 1: Building a T2VTree-style authoring backend in Python/Flask**

User: "I want to build a video authoring tool where users can explore different generation options as a tree and stitch the best results together."

Approach:
1. Set up a Flask backend with SQLite storing the node data model above
2. Create REST endpoints: `POST /nodes` (create child or sibling), `GET /tree` (fetch full tree), `PUT /nodes/:id` (edit spec before execution), `POST /nodes/:id/execute` (trigger generation), `POST /stitch` (assemble timeline)
3. Implement the four-agent planning pipeline using an LLM API, returning editable JSON specs
4. Wire generation dispatch to ComfyUI, Stable Diffusion, or other local backends based on `workflowChoice.moduleId`
5. Build the tree retrieval query that reconstructs parent-child relationships from flat storage

Output structure:
```
/backend
  app.py                 # Flask app with REST endpoints
  models.py              # T2VNode SQLAlchemy model
  agents/
    master_agent.py      # Action category classification
    knowledge_agent.py   # Entity/concept resolution
    workflow_agent.py    # Module selection logic
    prompt_agent.py      # Prompt materialization
  workflows/
    text_to_image.py     # Generation dispatch wrappers
    image_to_video.py
    audio_gen.py
  stitcher.py            # Video concatenation with ffmpeg
```

**Example 2: Implementing the four-agent planning pipeline**

User: "Help me implement the agent planning system that turns user intent into a generation plan."

Approach:
1. Define system prompts for each of the four agents with their specific roles
2. Chain them sequentially, passing accumulated context forward
3. Return a structured JSON plan the user can edit

```python
# master_agent.py
MASTER_SYSTEM = """You classify video authoring actions into categories.
Given the scene context, current branch history, and user intent,
output exactly one of: text-to-image, image-to-image, image-to-video,
video-to-video, text-to-audio, audio-to-audio, text-to-video."""

async def classify(scene_ctx: str, path_summary: str, intent: str) -> str:
    response = await llm.chat([
        {"role": "system", "content": MASTER_SYSTEM},
        {"role": "user", "content": f"Scene: {scene_ctx}\nHistory: {path_summary}\nIntent: {intent}"}
    ])
    return response.strip()

# prompt_agent.py
PROMPT_SYSTEM = """You materialize generation specifications.
Given an action category, workflow module, and user intent, produce:
- positive_prompt: detailed generation prompt
- negative_prompt: what to avoid
- parameters: dict of numerical controls with sensible defaults
Output valid JSON only."""

async def materialize(intent, action, workflow, clarifications, style_ctx) -> dict:
    response = await llm.chat([
        {"role": "system", "content": PROMPT_SYSTEM},
        {"role": "user", "content": f"Action: {action}\nWorkflow: {workflow}\n"
         f"Intent: {intent}\nClarifications: {clarifications}\nStyle: {style_ctx}"}
    ])
    return json.loads(response)
```

**Example 3: Building the tree visualization frontend**

User: "I need to build the tree UI where each node shows a preview of the generated content and users can branch off alternatives."

Approach:
1. Use Vue.js (or React) with an infinite canvas library (e.g., vue-flow, reactflow)
2. Render each node as a card showing: modality color border, thumbnail preview, intent text snippet
3. Implement three interaction modes: click to inspect/edit, drag-from-edge to create child, right-click to create sibling branch
4. Color-code nodes: blue border for images, green for video, red for audio

```vue
<template>
  <div class="t2v-node" :class="`modality-${node.modality}`">
    <div class="node-preview">
      <img v-if="node.modality === 'image'" :src="node.outputs[0]?.thumbnail" />
      <video v-else-if="node.modality === 'video'" :src="node.outputs[0]?.url" loop muted autoplay />
      <div v-else class="audio-waveform">{{ node.outputs[0]?.duration }}s</div>
    </div>
    <p class="node-intent">{{ truncate(node.intent, 60) }}</p>
    <div class="node-actions">
      <button @click="$emit('extend', node.id)">Refine</button>
      <button @click="$emit('branch', node.id)">Branch</button>
      <button @click="$emit('star', node.id)">Star</button>
    </div>
  </div>
</template>
```

## Best Practices

**Do:**
- Always surface agent-generated plans as editable artifacts before execution. The user must remain the decision-maker, not the agents.
- Preserve every exploration state as a tree node. Never overwrite — create child or sibling nodes instead. This is the core design principle.
- Keep agent planning calls lightweight (2k-4k tokens total across all four agents). Reserve compute budget for actual generation.
- Color-code nodes by output modality (image/video/audio) so the tree structure communicates content type at a glance.
- Maintain scene-level context (style, mood, palette) at the project root and propagate it into every planning call for cross-branch coherence.

**Avoid:**
- Auto-executing agent plans without user review. This undermines the user-centered design that makes T2VTree effective.
- Using explicit dependency edges between nodes for asset reuse. Instead, use referenced inputs within nodes — this avoids collapsing the tree into a complex DAG.
- Storing outputs outside the node. Each node must be self-contained: specification + outputs together enable provenance inspection.
- Building a linear pipeline. The tree structure is the point — if users can only refine sequentially, you lose branching comparison and the ability to resume abandoned directions.

## Error Handling

| Failure Mode | Handling Strategy |
|---|---|
| Generation fails (OOM, timeout, model error) | Attach error state to the node with full error details. The node persists in the tree so the user can edit parameters and retry without losing context. |
| Agent planning produces invalid JSON | Validate each agent's output with a schema. On failure, retry once with a stricter prompt. If still invalid, surface raw output in the edit form for manual correction. |
| Referenced input asset is missing | Check asset availability before execution. If a referenced node's output was pruned, warn the user and offer to re-generate or select an alternative. |
| Stitching produces discontinuities | Provide per-segment preview in the timeline before final export. Allow users to insert transition frames or adjust segment boundaries. |
| Tree becomes too large to render | Implement branch collapsing (hide subtrees), pruning (soft-delete abandoned branches), and viewport-based lazy rendering on the infinite canvas. |

## Limitations

- **Compute-intensive**: Each tree node potentially triggers a full generation run. Large trees with many branches multiply GPU cost. This approach works best when generation backends are available locally or via affordable APIs.
- **Not real-time**: The four-agent planning pipeline adds latency (several seconds) before generation even starts. Suitable for deliberate authoring, not rapid prototyping.
- **Tree complexity scales with exploration**: Heavy branching produces trees that are hard to navigate without good collapse/filter UI. The visualization itself requires engineering effort.
- **Audio-visual coherence is not automatic**: While scene-level context helps, the system does not enforce temporal consistency across video clips or audio-visual sync — these remain manual stitching decisions.
- **LLM planning quality depends on model capability**: The four agents need a capable model (GPT-4 class or above) to reliably classify actions, resolve entities, and materialize good prompts. Smaller models will produce lower-quality plans that require more user editing.

## Reference

**Paper**: [T2VTree: User-Centered Visual Analytics for Agent-Assisted Thought-to-Video Authoring](https://arxiv.org/abs/2602.08368v1) (Zheng et al., 2026)
**Code**: [github.com/tezuka0210/T2VTree](https://github.com/tezuka0210/T2VTree)
**Key insight**: Model the video authoring process as a tree where nodes bind editable specs with outputs, interpose four collaborating agents for intent-to-plan translation (Master/Knowledge/Workflow/Prompt), and keep all plans user-editable before execution. Look for Section 4 (System Design) and Section 5 (Agent Planning Pipeline) for implementation-level details.