---
name: "realistic-synthetic-household-data"
description: "Generate realistic synthetic household datasets with bidirectional persona-environment coupling for embodied AI training. Use when asked to 'generate synthetic household data', 'create home activity datasets', 'build training data for home robots', 'simulate household routines', 'generate person-environment interaction data', or 'create embodied AI training datasets'."
---

# Realistic Synthetic Household Data Generation

This skill enables Claude to build generative pipelines that produce large-scale, realistic synthetic household datasets by modeling the bidirectional influence between human personas and their living environments. Based on the loosely coupled iterative framework from Singh et al. (2026), the core idea is that personas shape environments (a work-from-home engineer gets a home office) while environments shape behavior (a kitchen stocked with baking supplies leads to baking activities). By iterating between environment generation and activity generation until convergence, the pipeline produces datasets with statistically validated realism suitable for training embodied AI agents.

## When to Use

- When the user needs synthetic training data for household robotics or embodied AI agents
- When building a data pipeline that generates diverse household floor plans with semantically consistent object placement
- When creating long-horizon human activity sequences tied to specific living environments
- When the user wants to generate human-robot interaction dialogues grounded in realistic home contexts
- When scaling dataset creation by parameterizing personas (age, lifestyle, occupation) via natural language
- When validating that synthetic household data aligns with real-world activity distributions
- When the user asks to simulate daily routines for smart home testing or digital twin environments

## Key Technique

The framework uses **loosely coupled bidirectional generation** -- two generators (environment and activity) run semi-independently but exchange influence signals across iterations. The environment generator produces room layouts, object inventories, and spatial affordances from a persona description. The activity generator then produces temporally consistent behavior sequences constrained by what the environment actually contains. Crucially, the activity generator feeds back: if it generates "doing laundry" but no laundry basket exists, the environment generator adds one in the next iteration. This loop runs until convergence, measured by three metrics: object density (objects per room), activity granularity (average duration per activity), and semantic similarity between environment and activity embeddings.

What makes this different from prior synthetic data frameworks (like ProcTHOR or Holodeck) is the **persona-driven parameterization** combined with **bidirectional coupling**. Instead of randomly populating scenes, every object placement is justified by who lives there and what they do. And instead of generating activities in a vacuum, every activity is grounded in what the environment physically supports. The paper validates this with intervention analysis: systematically changing persona attributes (age, organization level, sleep patterns) produces statistically significant downstream changes in both environments and behaviors (p < 0.001, Cohen's d = 0.51-1.12).

The pipeline uses LLMs as the core generation engine with structured prompting, rolling-window context to maintain consistency across long time horizons, and constraint propagation to prevent conflicts. Outputs include 3D schematics, object metadata, activity timelines, and human-robot dialogues, all adaptable to different simulator formats.

## Step-by-Step Workflow

1. **Define the persona specification** as a structured JSON or natural language description containing demographics (age, household size), lifestyle preferences (organized/messy, early-bird/night-owl), occupation, daily routine sketch, and any robot interaction preferences.

2. **Initialize the environment generator** with the persona. Produce a room list with dimensions, then populate each room with semantically appropriate objects from an asset database. Each object entry needs: name, category, dimensions, affordances, and placement coordinates. Use the persona to drive room assignment (e.g., children get bedrooms, remote workers get offices).

3. **Initialize the activity generator** with the persona and the generated environment inventory. Produce a hierarchical activity timeline: high-level daily schedule -> activity blocks -> atomic actions with timestamps, locations, objects used, and durations. Apply constraint propagation to prevent temporal overlaps and spatial impossibilities.

4. **Run the bidirectional influence loop** for up to 5 iterations:
   - `ActToEnvInfluence()`: Scan activities for referenced objects not in the environment. Add missing objects to appropriate rooms. Remove objects never referenced across all activities.
   - `EnvToActInfluence()`: Scan the environment for high-affordance objects not used in any activity. Generate plausible activities that use them. Remove activities referencing objects that were pruned.
   - Compute convergence metrics after each iteration: object density `rho = |objects| / |rooms|`, activity granularity `gamma = sum(durations) / |activities|`, semantic similarity `sigma = cosine_sim(env_embedding, activity_embedding)`.
   - Stop when delta across all three metrics falls below threshold (typically 0.05).

5. **Generate human-robot interaction dialogues** for activities that involve robot assistance. Dialogues should respect the persona's communication style, the robot's stated capabilities, and the physical context of the environment.

6. **Validate the generated dataset** using three checks:
   - Semantic alignment: compute cosine similarity between persona description embeddings and generated environment/activity embeddings (target: > 0.60).
   - Mutual information mediation: verify `MI(persona, environment) + MI(environment, behavior) > MI(persona, behavior)` to confirm environment mediates the persona-behavior relationship.
   - Intervention sensitivity: perturb one persona attribute and verify statistically significant downstream changes.

7. **Export to target format**: Convert the dataset to the required simulator format (AI2-THOR, Habitat, Isaac Sim) or a flat JSON/CSV structure for ML training pipelines. Include both static context (object graph, spatial relations) and temporal context (activity sequences, dialogues).

8. **Scale through parameterization**: Generate dataset variants by systematically varying persona attributes, LLM temperature/top_p settings, and asset database subsets. Use batch processing to create N diverse households from N persona specifications.

## Concrete Examples

**Example 1: Generate a household dataset for a retired couple**

User: "Create a synthetic household dataset for a retired couple in their 70s who enjoy gardening and cooking. They have a small dog and live in a two-bedroom house."

Approach:
1. Build the persona spec:
```json
{
  "household_id": "H001",
  "members": [
    {"name": "Person_A", "age": 72, "occupation": "retired", "hobbies": ["gardening", "cooking", "reading"]},
    {"name": "Person_B", "age": 70, "occupation": "retired", "hobbies": ["gardening", "baking", "walking"]}
  ],
  "pets": [{"type": "dog", "size": "small"}],
  "home_type": "two_bedroom_house",
  "lifestyle": {"organization": "high", "sleep_pattern": "early_bird", "activity_level": "moderate"}
}
```

2. Generate environment (iteration 0):
```json
{
  "rooms": [
    {"name": "kitchen", "objects": ["stove", "refrigerator", "cutting_board", "spice_rack", "stand_mixer", "dog_bowl"]},
    {"name": "living_room", "objects": ["sofa", "bookshelf", "reading_lamp", "dog_bed", "television"]},
    {"name": "master_bedroom", "objects": ["bed", "nightstand", "alarm_clock", "reading_glasses_tray"]},
    {"name": "guest_bedroom", "objects": ["bed", "dresser", "closet"]},
    {"name": "garden", "objects": ["raised_bed_planter", "watering_can", "garden_tools_rack", "potting_bench", "compost_bin"]},
    {"name": "bathroom", "objects": ["grab_bars", "shower_bench", "medicine_cabinet"]}
  ]
}
```

3. Generate activities, then run bidirectional loop. Iteration 1 adds a "herb_drying_rack" to the kitchen (referenced in cooking activities using garden herbs) and generates a "dog_walking" activity using the "leash_hook" that gets added to the entryway.

4. Final output includes 7-day activity timeline with ~40 daily activities per person, 12 HRI dialogues (robot assists with reaching high shelves, carrying garden supplies), and a complete 3D environment schematic.

**Example 2: Batch-generate diverse apartment datasets for robot navigation training**

User: "I need 50 diverse synthetic apartments with activity data to train a navigation policy. Vary the demographics widely."

Approach:
1. Create a persona generator that parameterizes across axes:
```python
import itertools, random

age_ranges = [(20, 30), (30, 45), (45, 60), (60, 80)]
household_types = ["single", "couple", "family_with_kids", "roommates"]
lifestyles = ["minimalist", "cluttered", "organized", "tech_heavy"]
occupations = ["remote_worker", "student", "retired", "shift_worker"]

personas = []
for i in range(50):
    personas.append({
        "id": f"APT_{i:03d}",
        "age_range": random.choice(age_ranges),
        "household_type": random.choice(household_types),
        "lifestyle": random.choice(lifestyles),
        "occupation": random.choice(occupations),
        "llm_temperature": random.uniform(0.7, 1.0)
    })
```

2. For each persona, run the full bidirectional generation pipeline (environment + activities + convergence loop).

3. Export each apartment as a self-contained directory:
```
dataset/
  APT_000/
    environment.json    # room layouts, object placements, spatial graph
    activities.json     # 7-day activity timeline per household member
    dialogues.json      # human-robot interactions
    metadata.json       # persona spec, generation params, convergence metrics
  APT_001/
  ...
  validation_report.json  # aggregate semantic alignment scores
```

4. Run validation: compute per-apartment semantic similarity, then aggregate. Flag any apartments with persona-environment cosine similarity below 0.55 for regeneration.

**Example 3: Add robot interaction data to an existing environment**

User: "I have a 3D home scene already. I want to generate realistic daily activity sequences and robot interaction data for it."

Approach:
1. Ingest the existing environment as the fixed initial state -- extract the room list and object inventory from the user's scene file.
2. Skip environment generation. Run only the activity generator, constrained to objects and rooms that already exist.
3. Run a one-directional loop: `EnvToActInfluence()` only, since the environment is fixed. Generate activities that use the available affordances.
4. Produce HRI dialogues for a configurable robot capability profile (mobile manipulator, voice assistant, etc.).
5. Export activity timelines and dialogues in the user's preferred format, with object references matching their existing asset IDs.

## Best Practices

- **Do:** Use structured persona specifications with explicit attributes rather than free-text descriptions. The more specific the persona, the more differentiated the generated data.
- **Do:** Run at least 3 bidirectional iterations before checking convergence. The first iteration typically produces the largest changes; premature stopping yields shallow coupling.
- **Do:** Track convergence metrics (object density, activity granularity, semantic similarity) across iterations and log them for reproducibility.
- **Do:** Validate with intervention analysis on at least one persona attribute before using the dataset for training. This confirms the pipeline is actually responsive to input variation.
- **Avoid:** Generating environments and activities independently without the bidirectional loop. This produces semantically disconnected data where activities reference objects that don't exist or environments contain unused objects.
- **Avoid:** Using identical LLM sampling parameters across all personas in a batch. Vary temperature (0.7-1.0) and top_p (0.8-0.95) to increase diversity.
- **Avoid:** Skipping the asset metadata requirements. Every object needs dimensions, affordances, and a category label -- without these, spatial placement becomes physically implausible.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Convergence loop never terminates | Oscillating additions/removals between environment and activity generators | Cap iterations at 5-7; use damping by only accepting changes above a relevance threshold |
| Activities reference impossible object interactions | LLM hallucinating affordances (e.g., "sit on the microwave") | Add an affordance validation layer that checks each action against the object's declared affordance set |
| Environment becomes unrealistically dense | Activity generator keeps requesting new objects | Set a per-room object cap based on room dimensions; enforce it in `ActToEnvInfluence()` |
| Low persona-environment semantic similarity (< 0.50) | Persona description too vague or contradictory | Refine the persona spec with more concrete attributes; check for conflicting lifestyle indicators |
| Temporal conflicts in activity sequences | Overlapping activities or teleportation between rooms | Apply constraint propagation post-generation: enforce minimum transition times between rooms and prevent temporal overlaps |
| Dialogue quality degrades for long interactions | Context window limitations | Use a rolling-window context mechanism that summarizes earlier dialogue turns rather than including them verbatim |

## Limitations

- The framework relies heavily on LLM knowledge of household norms, which may encode cultural biases (Western-centric home layouts, activity patterns). Generated data should be audited for cultural representativeness.
- Physical plausibility of 3D object placement is approximate -- the pipeline generates coordinates and checks basic collision constraints, but does not run physics simulation. Integration with a physics engine is needed for rigorous spatial validation.
- Activity sequences are generated at the semantic level (action descriptions with timestamps), not at the motion-planning level. A sim-to-real bridge is still needed for robot control policies.
- The paper reports cosine similarity of 0.60 against the HOMER real-world dataset -- useful but not perfect. Synthetic data should supplement, not replace, real data collection.
- Computational cost scales linearly with household complexity and iteration count (~130 seconds per household at 5 iterations). Batch generation of thousands of households requires parallelization.

## Reference

**Paper:** Singh, S., Idrees, I., & Dauhajre, A. (2026). "Realistic Synthetic Household Data Generation at Scale." arXiv:2602.07243v1. [https://arxiv.org/abs/2602.07243v1](https://arxiv.org/abs/2602.07243v1)

**Key insight to look for:** The bidirectional influence controller algorithm (Algorithm 1) and the three convergence metrics (object density, activity granularity, semantic similarity) that determine when environment and activity generators have reached mutual consistency.