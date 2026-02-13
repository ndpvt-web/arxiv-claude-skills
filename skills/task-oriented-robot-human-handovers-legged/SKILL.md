---
name: "task-oriented-robot-human-handovers-legged"
description: "Implement task-oriented robot-to-human object handover systems using LLM-driven affordance reasoning and exemplar-based affordance transfer (AFT-Handover). Use when: 'build a handover planner for a robot arm', 'implement affordance-based grasp planning', 'create an LLM-driven object manipulation pipeline', 'design a task-aware object presentation system', 'implement zero-shot affordance transfer between objects', 'build a robot handover system that considers how the human will use the tool'."
---

# Task-Oriented Robot-Human Handovers via AFT-Handover

This skill enables Claude to implement the AFT-Handover framework from Tulbure et al. (HRI 2026), which solves the problem of handing objects to humans in a task-appropriate orientation and grasp region. The core technique combines LLM-driven affordance reasoning (determining *which parts* of an object matter for a given task) with texture-based affordance transfer (mapping those affordance labels from known exemplar objects onto novel objects via point cloud correspondences). This achieves zero-shot generalization to unseen object-task pairs without retraining.

## When to Use

- When building a robotic handover pipeline that must present objects based on the human's intended use (e.g., handing a knife handle-first for cutting vs. blade-first for inspection)
- When implementing LLM-based affordance reasoning to decompose objects into semantic parts and label which parts are functional for a task
- When designing an exemplar retrieval database that maps novel objects to known proxy objects for affordance transfer
- When implementing point cloud feature matching to transfer spatial affordance labels between geometrically dissimilar but functionally similar objects
- When building a grasp planner that selects robot grasp poses to leave the human's intended grasp region unoccluded
- When creating a zero-shot manipulation system that generalizes to novel tools, utensils, or objects without per-object training data

## Key Technique

**The core insight** of AFT-Handover is that task-oriented handovers can be decomposed into three tractable subproblems: (1) *What are the functional parts of this object for the given task?* (2) *Where on the object's surface do those parts live?* (3) *How should the robot grasp so the human can grab the task-relevant region?* The paper solves (1) with an LLM, (2) with exemplar-based affordance transfer through point cloud textures, and (3) with constrained grasp planning that avoids the human's intended contact region.

**Affordance reasoning via LLM:** Given a novel object (e.g., "ladle") and task (e.g., "scooping soup"), the system prompts an LLM to enumerate the object's semantic parts (handle, bowl, rim) and classify each as either a *functional part* (the part the human needs to use -- the bowl for scooping) or a *grasp part* (where the human will hold it -- the handle). This replaces hand-crafted affordance databases with generalizable language-based reasoning.

**Texture-based affordance transfer:** Rather than predicting affordances directly on novel geometry, the system retrieves a proxy exemplar from a database -- an object whose parts have already been annotated with spatial affordance labels on its point cloud. It then uses the LLM to establish part-level correspondences between the exemplar and the novel object (e.g., "ladle handle" maps to "spoon handle"). Affordance labels are "texturized" onto the exemplar's surface, and a feature-based point cloud registration (using geometric and color/texture descriptors) transfers those labels onto the novel object's point cloud. The robot then plans a grasp on the non-functional region and orients the object so the human can grab the functional/grasp part naturally.

## Step-by-Step Implementation Workflow

1. **Define the object-task input schema.** Create a data structure that accepts `(object_name: str, object_description: str, task: str, point_cloud: PointCloud)` as input. The point cloud should be an Open3D or NumPy array of shape `(N, 6)` with XYZ + RGB channels.

2. **Build the affordance reasoning LLM prompt.** Construct a structured prompt that gives the LLM the object name, a brief description, and the task. Ask it to: (a) list all semantic parts of the object, (b) classify each part as `functional` (needed for the task), `grasp` (where the human holds), or `structural` (neither), and (c) return a JSON dictionary. Use few-shot examples for consistency.

   ```
   System: You are an expert in object affordances for robot manipulation.
   User: Object: {object_name} ({object_description}). Task: {task}.
   List the semantic parts of this object. For each part, classify it as:
   - "functional": the part the human directly uses to perform the task
   - "grasp": where the human naturally holds the object during the task
   - "structural": parts not directly involved in task execution
   Return JSON: {"parts": [{"name": str, "role": str, "description": str}]}
   ```

3. **Build and query the exemplar database.** Maintain a database of known objects, each stored as `{object_name, category, parts: [{name, role, annotated_point_indices}], point_cloud}`. For a novel object, query the LLM to select the best proxy exemplar: "Given this novel object [X] and these database entries [list], which object is most functionally similar for the task [T]?" Retrieve the exemplar's annotated point cloud.

4. **Establish part-level correspondences via LLM.** Prompt the LLM with both the novel object's part list and the exemplar's part list. Ask it to produce a mapping: `{exemplar_part_name -> novel_part_name}` for each semantically corresponding part. This handles cases where part names differ (e.g., exemplar "blade" maps to novel object "cutting edge").

   ```
   System: Match corresponding parts between two objects.
   Exemplar ({exemplar_name}): parts = {exemplar_parts}
   Novel ({novel_name}): parts = {novel_parts}
   Return JSON: {"correspondences": [{"exemplar_part": str, "novel_part": str}]}
   ```

5. **Transfer affordances via point cloud feature matching.** For each corresponding part pair: (a) extract the exemplar's sub-cloud for that part using the stored indices, (b) compute local geometric features (FPFH or SHOT descriptors) on both the exemplar sub-cloud and the novel object's full cloud, (c) use feature-based registration (RANSAC + ICP refinement) to align the part regions, (d) propagate the affordance labels (functional/grasp/structural) from the exemplar's points to the nearest novel-object points. Use Open3D's `compute_fpfh_feature` and `registration_ransac_based_on_feature_matching`.

6. **Texturize affordance labels onto the point cloud.** Assign each point in the novel object's cloud a color or label based on its transferred affordance role. This creates a visual affordance map: red for functional regions, green for grasp regions, grey for structural. This texturized cloud is both a debugging visualization and the input to grasp planning.

7. **Compute the robot grasp pose.** The robot must grasp the object on the *non-grasp* region (structural or the opposite side of functional) so the human can access the grasp region. Filter candidate grasp poses (from a grasp sampler like GraspIt! or Contact-GraspNet) to those that: (a) contact only structural/non-grasp points, (b) maintain force closure, (c) orient the grasp region toward the expected human hand position.

8. **Plan the handover trajectory.** Given the selected grasp pose and a target handover position (typically in front of the human at waist-to-chest height), plan an arm trajectory that presents the object's grasp region facing the human's dominant hand. For legged manipulators, include base positioning to reach the handover pose.

9. **Implement a release trigger.** Monitor force/torque at the end-effector or use a simple pull-detection threshold. When the human grasps and tugs, release the object. A well-chosen handover pose minimizes regrasping -- the human should not need to rotate or reposition the object before using it.

10. **Evaluate with a handover quality metric.** Measure: (a) grasp region accessibility -- is the human's intended grasp region unoccluded by the robot's gripper? (b) orientation correctness -- is the functional part oriented for immediate use? (c) regrasp rate -- does the human need to adjust their grip before performing the task? Compare against a baseline that ignores task context.

## Concrete Examples

**Example 1: Implementing the LLM affordance reasoner**

User: "I need a Python function that takes an object name and a task, and returns which parts of the object are functional vs. where the human should grasp it."

Approach:
1. Define a function `reason_affordances(object_name: str, task: str, llm_client) -> dict`
2. Build the affordance reasoning prompt with the object and task
3. Parse the LLM's JSON response into a structured `AffordanceMap`

Output:
```python
from dataclasses import dataclass
from typing import Literal
import json

@dataclass
class PartAffordance:
    name: str
    role: Literal["functional", "grasp", "structural"]
    description: str

def reason_affordances(object_name: str, task: str, llm_client) -> list[PartAffordance]:
    """Use an LLM to decompose an object into semantic parts with affordance roles."""
    prompt = f"""Object: {object_name}. Task: {task}.
List the semantic parts of this object and classify each as:
- "functional": the part directly used to perform the task
- "grasp": where the human holds the object during the task
- "structural": parts not directly involved
Return JSON: {{"parts": [{{"name": str, "role": str, "description": str}}]}}"""

    response = llm_client.chat(
        messages=[
            {"role": "system", "content": "You are an expert in object affordances for robotic manipulation."},
            {"role": "user", "content": prompt},
        ],
        response_format={"type": "json_object"},
    )
    data = json.loads(response.content)
    return [PartAffordance(**p) for p in data["parts"]]

# Usage:
# parts = reason_affordances("hammer", "driving a nail", client)
# => [PartAffordance(name="head", role="functional", description="strikes the nail"),
#     PartAffordance(name="handle", role="grasp", description="held by the user"),
#     PartAffordance(name="claw", role="structural", description="not used for this task")]
```

**Example 2: Building the exemplar database and retrieval**

User: "I need to build a database of known objects with annotated affordance regions, and retrieve the best proxy for a novel object."

Approach:
1. Define the exemplar schema with point cloud indices per part
2. Implement retrieval that queries the LLM to match novel objects to exemplars
3. Return the exemplar and its annotated point cloud

Output:
```python
import numpy as np
from dataclasses import dataclass, field

@dataclass
class ExemplarObject:
    name: str
    category: str
    parts: dict[str, dict]  # part_name -> {"role": str, "point_indices": np.ndarray}
    point_cloud: np.ndarray  # (N, 6) XYZ+RGB

class AffordanceExemplarDB:
    def __init__(self):
        self.exemplars: dict[str, ExemplarObject] = {}

    def add(self, exemplar: ExemplarObject):
        self.exemplars[exemplar.name] = exemplar

    def retrieve_proxy(self, novel_object: str, task: str, llm_client) -> ExemplarObject:
        """Ask the LLM which exemplar is the best functional proxy."""
        catalog = [
            f"- {e.name} ({e.category}): parts = {list(e.parts.keys())}"
            for e in self.exemplars.values()
        ]
        prompt = f"""Novel object: {novel_object}. Task: {task}.
Which of these known objects is the best functional proxy?
{chr(10).join(catalog)}
Return JSON: {{"exemplar_name": str, "reasoning": str}}"""

        response = llm_client.chat(
            messages=[{"role": "user", "content": prompt}],
            response_format={"type": "json_object"},
        )
        result = json.loads(response.content)
        return self.exemplars[result["exemplar_name"]]
```

**Example 3: Point cloud affordance transfer**

User: "Given an exemplar with labeled part regions and a novel object's point cloud, transfer the affordance labels using geometric feature matching."

Approach:
1. Compute FPFH features on both point clouds
2. For each exemplar part, register its sub-cloud against the novel cloud
3. Propagate labels to nearest novel-cloud points

Output:
```python
import open3d as o3d
import numpy as np

def transfer_affordances(
    exemplar_cloud: np.ndarray,       # (M, 6) XYZ+RGB
    exemplar_parts: dict[str, dict],  # part_name -> {"role": str, "point_indices": ndarray}
    novel_cloud: np.ndarray,          # (N, 6) XYZ+RGB
    correspondences: dict[str, str],  # exemplar_part -> novel_part
    voxel_size: float = 0.005,
) -> np.ndarray:
    """Transfer affordance labels from exemplar to novel object via FPFH registration.
    Returns array of shape (N,) with labels: 0=structural, 1=functional, 2=grasp."""

    role_to_label = {"structural": 0, "functional": 1, "grasp": 2}
    novel_labels = np.zeros(len(novel_cloud), dtype=np.int32)

    novel_pcd = o3d.geometry.PointCloud()
    novel_pcd.points = o3d.utility.Vector3dVector(novel_cloud[:, :3])
    novel_pcd.colors = o3d.utility.Vector3dVector(novel_cloud[:, 3:6])
    novel_tree = o3d.geometry.KDTreeFlann(novel_pcd)

    for ex_part_name, ex_part_info in exemplar_parts.items():
        if ex_part_name not in correspondences:
            continue
        role = ex_part_info["role"]
        label = role_to_label[role]
        indices = ex_part_info["point_indices"]

        # Extract exemplar sub-cloud for this part
        sub_cloud = exemplar_cloud[indices]
        ex_pcd = o3d.geometry.PointCloud()
        ex_pcd.points = o3d.utility.Vector3dVector(sub_cloud[:, :3])

        # Compute FPFH features
        ex_pcd.estimate_normals(o3d.geometry.KDTreeSearchParamHybrid(radius=voxel_size * 5, max_nn=30))
        novel_pcd.estimate_normals(o3d.geometry.KDTreeSearchParamHybrid(radius=voxel_size * 5, max_nn=30))

        ex_fpfh = o3d.pipelines.registration.compute_fpfh_feature(
            ex_pcd, o3d.geometry.KDTreeSearchParamHybrid(radius=voxel_size * 10, max_nn=100))
        novel_fpfh = o3d.pipelines.registration.compute_fpfh_feature(
            novel_pcd, o3d.geometry.KDTreeSearchParamHybrid(radius=voxel_size * 10, max_nn=100))

        # RANSAC registration
        result = o3d.pipelines.registration.registration_ransac_based_on_feature_matching(
            ex_pcd, novel_pcd, ex_fpfh, novel_fpfh,
            mutual_filter=True,
            max_correspondence_distance=voxel_size * 3,
            estimation_method=o3d.pipelines.registration.TransformationEstimationPointToPoint(),
            ransac_n=4,
            criteria=o3d.pipelines.registration.RANSACConvergenceCriteria(max_iteration=50000, confidence=0.999),
        )

        # Transform exemplar part and find nearest novel points
        ex_pcd.transform(result.transformation)
        for pt in np.asarray(ex_pcd.points):
            _, idx, _ = novel_tree.search_knn_vector_3d(pt, 1)
            novel_labels[idx[0]] = label

    return novel_labels
```

## Best Practices

- **Do:** Use structured JSON output from the LLM for affordance reasoning. Unstructured free-text is fragile and hard to parse reliably in a robotics pipeline.
- **Do:** Include 3-5 few-shot examples in the affordance reasoning prompt covering diverse object categories (tools, kitchen utensils, sports equipment). This dramatically improves part decomposition consistency.
- **Do:** Store exemplar point clouds with pre-computed FPFH features to avoid recomputation on every query. The exemplar database is static and features can be cached.
- **Do:** Validate LLM correspondences by checking that matched parts have compatible roles (a "functional" exemplar part should map to a "functional" novel part). Reject and re-prompt if mismatched.
- **Avoid:** Relying solely on geometric similarity for exemplar retrieval. A spatula and a tennis racket are geometrically similar but functionally different -- the LLM's semantic understanding is the key differentiator.
- **Avoid:** Skipping ICP refinement after RANSAC. RANSAC gives a coarse alignment; ICP (with `registration_icp` in Open3D) tightens it and significantly improves label transfer accuracy at part boundaries.

## Error Handling

- **LLM returns malformed JSON:** Wrap LLM calls in a retry loop (max 3 attempts) with explicit format instructions. Validate against a Pydantic or dataclass schema before proceeding.
- **No suitable exemplar in database:** If the LLM reports low confidence in its proxy selection (or the best match has <50% part coverage), fall back to a generic heuristic: present elongated objects handle-first, flat objects by the edge.
- **RANSAC registration fails (low fitness):** If `result.fitness < 0.3`, the geometric match is poor. Try: (a) increasing `max_correspondence_distance`, (b) using a coarser voxel downsampling, (c) falling back to centroid-based alignment for that part.
- **Point cloud too sparse or noisy:** Pre-process with `statistical_outlier_removal` and ensure a minimum of ~1000 points per part for reliable FPFH computation. If the novel object's cloud has fewer than 500 points total, request a rescan.
- **Ambiguous part roles:** Some objects have parts that are both functional and grasp (e.g., a ball). When the LLM returns dual-role parts, prioritize the functional label for handover planning -- the robot should avoid occluding the part the human needs.

## Limitations

- **LLM affordance reasoning is not grounded in physics.** The LLM infers affordances from language priors, not from material properties or force analysis. It may misclassify parts for unusual tasks (e.g., using a hammer as a paperweight).
- **Exemplar database coverage determines generalization ceiling.** Truly alien objects with no functionally similar exemplar will get poor affordance transfer. The database needs at least 50-100 well-annotated objects across common categories to be practically useful.
- **Point cloud quality dependency.** The texture-based transfer relies on geometric features that degrade with low-resolution depth sensors, reflective surfaces, or transparent objects.
- **Single-task assumption.** The framework reasons about one task at a time. If the human's intent is ambiguous or multi-step, the system needs an intent disambiguation module upstream.
- **No dynamic re-planning.** Once the grasp is planned, the system commits. If the human reaches from an unexpected direction, it does not adjust the presentation angle in real time.

## Reference

Tulbure, A., Scheidemann, C., Steiner, E., & Hutter, M. (2026). *Task-Oriented Robot-Human Handovers on Legged Manipulators.* Accepted at HRI 2026. [arXiv:2602.05760](https://arxiv.org/abs/2602.05760v1). Key sections: the LLM-driven affordance reasoning pipeline (Sec. III), the texture-based affordance transfer algorithm (Sec. IV), and the user study showing reduced regrasping rates (Sec. VI).