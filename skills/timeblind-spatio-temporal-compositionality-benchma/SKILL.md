---
name: "timeblind-spatio-temporal-compositionality-benchma"
description: "Build and evaluate spatio-temporal reasoning benchmarks for video LLMs using the TimeBlind minimal-pairs methodology. Generates contrastive video-question pairs that isolate temporal dynamics from static visual cues. Trigger phrases: 'evaluate temporal reasoning', 'benchmark video understanding', 'test video LLM compositionality', 'create temporal contrastive pairs', 'diagnose video model weaknesses', 'Allen interval relations evaluation'."
---

# TimeBlind: Spatio-Temporal Compositionality Benchmarking for Video LLMs

This skill enables Claude to design, implement, and apply the TimeBlind benchmark methodology for evaluating how well video language models understand temporal dynamics. The core technique uses **minimal contrastive pairs** — two videos sharing identical static content but differing solely in temporal structure — paired with **complementary questions** whose correct answers flip between videos. This isolates genuine temporal reasoning from static visual shortcut exploitation, exposing brittle temporal understanding in frontier models (best MLLM: 48.2% Instance Accuracy vs. human: 98.2%).

## When to Use

- When building evaluation pipelines for video understanding models and need to test temporal reasoning specifically
- When designing contrastive video-question datasets that resist language priors and visual shortcuts
- When implementing a three-level cognitive taxonomy (atomic events, event properties, event relations) for structured video analysis
- When evaluating whether a video LLM genuinely tracks temporal dynamics or relies on single-frame heuristics
- When implementing Allen's 13 interval algebra relations (before, after, meets, overlaps, during, etc.) as a reasoning framework
- When auditing a video QA system for shortcut exploitation using diagnostic accuracy metrics
- When generating structured benchmark schemas with GPT or other LLMs for temporal video evaluation

## Key Technique

### Minimal-Pairs with Complementary Questions

TimeBlind's central insight is that standard video QA benchmarks conflate static scene understanding with temporal reasoning. A model can often answer "What happened?" by looking at a single frame. To fix this, TimeBlind constructs **minimal pairs**: two videos (v1, v2) sharing identical objects, backgrounds, and camera angles, differing **only** in the targeted temporal factor (e.g., speed increasing vs. decreasing, or event A before B vs. B before A). Each pair comes with complementary questions (q1, q2) where the ground-truth answer flips between v1 and v2. This creates four trials per instance: (v1,q1), (v1,q2), (v2,q1), (v2,q2). A model must get all four correct to earn Instance Accuracy, which is the primary metric that proves genuine temporal discrimination.

### Three-Level Compositional Taxonomy

Inspired by cognitive science, temporal understanding is decomposed into three hierarchical levels. **Level I (Atomic Events)** tests recognition of fine-grained actions (opening vs. closing) and state transitions (color change, shape change). **Level II (Event Attributes)** tests how events unfold — kinematics (speed, direction, duration, repetition) and dynamics (force, magnitude). **Level III (Structural Event Logic)** tests compositional reasoning — temporal topology using all 13 Allen interval relations, causal contingency between events, and cross-event comparison. This hierarchy lets you pinpoint exactly where a model's temporal understanding breaks down.

### Diagnostic Metric Hierarchy

Four accuracy levels expose different failure modes: **Accuracy (Acc)** is standard per-trial accuracy. **Video Accuracy (V-Acc)** requires both questions correct for a single video — testing within-video consistency. **Question Accuracy (Q-Acc)** requires one question correct across both videos — testing cross-video discrimination. **Instance Accuracy (I-Acc)** requires all four trials correct — the only metric that proves a model genuinely distinguishes temporal structure rather than exploiting shortcuts. The gap between Acc and I-Acc directly measures shortcut reliance.

## Step-by-Step Workflow

1. **Define the temporal dimension to test.** Select from the three-level taxonomy: atomic events (actions, state transitions), event attributes (speed, direction, duration, repetition, force), or structural logic (Allen relations, causality, cross-event comparison). Each dimension becomes a test category.

2. **Generate paired video schemas.** For each category, produce structured descriptions of two videos that share all static content but differ in exactly one temporal factor. Use an LLM to draft schemas following this format:
   ```json
   {
     "category": "kinematics/speed",
     "level": 2,
     "video_1_description": "A ball rolls left to right, accelerating over 5 seconds",
     "video_2_description": "A ball rolls left to right, decelerating over 5 seconds",
     "static_invariants": ["same ball", "same background", "same trajectory", "same camera angle"],
     "temporal_difference": "acceleration direction"
   }
   ```

3. **Write complementary question pairs.** Design questions where the correct answer flips between v1 and v2. Ensure questions cannot be answered from a single frame or from language bias alone:
   ```json
   {
     "q1": "Is the ball speeding up?",
     "q1_answer_v1": "Yes",
     "q1_answer_v2": "No",
     "q2": "Is the ball slowing down?",
     "q2_answer_v1": "No",
     "q2_answer_v2": "Yes",
     "single_frame_sufficient": false,
     "reasoning": "Speed change requires comparing frames across time"
   }
   ```

4. **Acquire or generate contrastive video pairs.** Use one of three sources: internet retrieval with manual curation, human recording with controlled conditions, or programmatic generation via Unity/simulation engines. Verify that both videos share identical static content.

5. **Validate temporal minimality.** Run a manual or automated check: (a) extract random single frames from both videos and confirm they are visually indistinguishable, (b) verify the temporal difference is the sole discriminative factor, (c) confirm questions cannot be answered from static frames alone.

6. **Structure the evaluation harness.** Implement the four-trial evaluation: for each instance, run the model on all combinations of (v1,q1), (v1,q2), (v2,q1), (v2,q2). Store raw predictions for all four trials.

7. **Compute the diagnostic metric hierarchy.** Calculate Acc (per-trial), V-Acc (both questions correct per video), Q-Acc (one question correct across both videos), and I-Acc (all four correct). Report per-category breakdowns across the three taxonomy levels.

8. **Run ablation diagnostics.** Test single-frame input (feed one frame instead of video), language-only input (question without video), and frame-shuffled input (randomize frame order). Compare against full-video performance to quantify shortcut exploitation.

9. **Analyze failure patterns by taxonomy level.** Identify whether the model fails at atomic recognition (Level I), attribute discrimination (Level II), or compositional reasoning (Level III). The gap between levels reveals the model's compositional ceiling.

10. **Generate a structured diagnostic report.** Produce a per-category breakdown showing which temporal dimensions are weakest, the Acc-to-I-Acc gap (shortcut reliance score), and specific failure examples with reasoning about why the model likely failed.

## Concrete Examples

**Example 1: Evaluating a Video LLM on Temporal Ordering**

User: "I want to test whether my video QA model actually understands event ordering or just guesses based on typical sequences."

Approach:
1. Select Level III / Temporal Topology category, focusing on Allen relations "before" and "after"
2. Generate schema: v1 shows "person picks up cup THEN drinks", v2 shows "person drinks THEN picks up cup" (reversed via editing)
3. Create complementary questions: q1="Does the person pick up the cup before drinking?" (v1:Yes, v2:No), q2="Does the person drink before picking up the cup?" (v1:No, v2:Yes)
4. Run model on all four (video, question) combinations
5. Compute metrics

Output:
```
Category: Temporal Topology / before-after
Instance: event_order_001

Trial Results:
  (v1, q1) "Pick up before drinking?" → Model: Yes ✓
  (v1, q2) "Drink before picking up?" → Model: No  ✓
  (v2, q1) "Pick up before drinking?" → Model: Yes ✗ (should be No)
  (v2, q2) "Drink before picking up?" → Model: No  ✗ (should be Yes)

Metrics:
  Acc:   2/4 = 50.0%
  V-Acc: v1=1/1, v2=0/1 → 50.0%
  Q-Acc: q1=0/1, q2=0/1 → 0.0%
  I-Acc: 0/1 = 0.0%

Diagnosis: Model answers correctly for v1 (typical order) but fails on v2
(atypical order), indicating reliance on language prior for typical
event sequences rather than genuine temporal perception.
```

**Example 2: Building a Speed Discrimination Benchmark**

User: "Create a benchmark to test if models can distinguish between fast and slow versions of the same action."

Approach:
1. Select Level II / Kinematics / Speed category
2. Generate 10 paired schemas: same action performed at different speeds
3. Write complementary questions about speed
4. Define the evaluation harness

Output:
```python
# Schema generation for speed discrimination
import json

def generate_speed_schema(action, object_desc):
    return {
        "category": "kinematics/speed",
        "level": 2,
        "video_1_description": f"{action} performed quickly (2x normal speed)",
        "video_2_description": f"{action} performed slowly (0.5x normal speed)",
        "static_invariants": [object_desc, "same environment", "same camera"],
        "temporal_difference": "execution speed",
        "questions": [
            {
                "q": f"Is the {action} being performed quickly?",
                "answer_v1": "Yes", "answer_v2": "No"
            },
            {
                "q": f"Is the {action} being performed slowly?",
                "answer_v1": "No", "answer_v2": "Yes"
            }
        ],
        "single_frame_check": "A single frame cannot distinguish speed"
    }

# Evaluation harness
def evaluate_instance(model, v1, v2, q1, q2, answers):
    trials = [
        model.predict(v1, q1), model.predict(v1, q2),
        model.predict(v2, q1), model.predict(v2, q2)
    ]
    gt = [answers["v1_q1"], answers["v1_q2"],
          answers["v2_q1"], answers["v2_q2"]]

    acc = sum(t == g for t, g in zip(trials, gt)) / 4
    v_acc_1 = int(trials[0] == gt[0] and trials[1] == gt[1])
    v_acc_2 = int(trials[2] == gt[2] and trials[3] == gt[3])
    q_acc_1 = int(trials[0] == gt[0] and trials[2] == gt[2])
    q_acc_2 = int(trials[1] == gt[1] and trials[3] == gt[3])
    i_acc = int(all(t == g for t, g in zip(trials, gt)))

    return {
        "acc": acc, "v_acc": (v_acc_1 + v_acc_2) / 2,
        "q_acc": (q_acc_1 + q_acc_2) / 2, "i_acc": i_acc
    }
```

**Example 3: Diagnosing Allen Interval Relation Failures**

User: "My model handles 'before/after' fine but I want to test harder temporal relations like 'overlaps' and 'during'."

Approach:
1. Select Level III / Temporal Topology, targeting Allen relations: overlaps, overlapped-by, during, contains
2. Generate schemas where two concurrent events have specific temporal overlap patterns
3. Write questions that require distinguishing overlap type

Output:
```
Allen Relation Test: "overlaps" vs "during"

Schema:
  v1: Person A waves (0-6s) while Person B claps (3-9s)
      → A overlaps B (A starts first, B ends last, they share middle)
  v2: Person A waves (0-9s) while Person B claps (3-6s)
      → B during A (B fully contained within A's interval)

  Static invariants: same people, same room, same actions
  Temporal difference: interval containment structure

Questions:
  q1: "Does the clapping happen entirely within the waving period?"
      v1: No (clapping extends beyond waving)  v2: Yes
  q2: "Does the clapping extend beyond the end of the waving?"
      v1: Yes  v2: No

Ablation check:
  Single-frame test: Any frame showing both actions looks identical
  in v1 and v2 → single frame cannot distinguish
  Language-only: No prior about which relation is more common
```

## Best Practices

- **Do:** Always verify temporal minimality — confirm that a single random frame from v1 is indistinguishable from a single random frame from v2. This is the core guarantee that forces temporal reasoning.
- **Do:** Report I-Acc as the primary metric. Standard accuracy inflates scores because models can exploit language priors or get 2/4 trials correct by chance.
- **Do:** Include all three taxonomy levels in any comprehensive evaluation. A model that excels at Level I but fails at Level III has shallow temporal understanding.
- **Do:** Run the single-frame ablation. If a model scores similarly on single-frame vs. full-video input, it is not using temporal information at all.
- **Avoid:** Using questions answerable from static visual content (e.g., "What color is the ball?" works for both videos identically and tests nothing temporal).
- **Avoid:** Unbalanced answer distributions. Complementary question design ensures 50/50 Yes/No balance, preventing accuracy inflation from answer bias.

## Error Handling

- **Videos not temporally minimal:** If single-frame checks reveal distinguishable static differences (different lighting, objects shifted), reject the pair and re-acquire. Static leakage invalidates the entire instance.
- **Questions answerable from language alone:** If a language-only baseline (no video) exceeds 60% accuracy on a category, the questions have language bias. Rewrite with more neutral phrasing or swap complementary question structure.
- **Model refuses to answer or outputs unexpected format:** Wrap model calls with answer extraction that maps free-form text to the expected answer set. Log unparseable responses separately as "invalid" rather than scoring them as incorrect.
- **Insufficient category coverage:** If a taxonomy level has fewer than 30 instances, statistical conclusions are unreliable. Aim for 50+ instances per fine-grained category for robust per-category analysis.

## Limitations

- **Video acquisition is expensive.** The minimal-pairs constraint requires careful curation — random web videos rarely form valid contrastive pairs. Programmatic generation (Unity, simulation) helps but limits visual realism.
- **Binary question format is restrictive.** The complementary Yes/No structure works for diagnostic isolation but does not test open-ended temporal narration or free-form video description.
- **Short video bias.** The benchmark averages 8.5-second clips. Performance on longer videos with complex multi-event narratives is not directly measured by this methodology.
- **Does not test generation.** TimeBlind evaluates discriminative temporal understanding only. It does not assess whether a model can generate temporally coherent video descriptions or plans.
- **Allen relations assume crisp intervals.** Real-world events often have fuzzy boundaries. The 13-relation framework may oversimplify gradual transitions.

## Reference

**Paper:** [TimeBlind: A Spatio-Temporal Compositionality Benchmark for Video LLMs](https://arxiv.org/abs/2602.00288v1) (Li et al., 2026)

Key takeaway: Frontier video LLMs achieve only 48.2% Instance Accuracy vs. 98.2% human performance when temporal shortcuts are eliminated via minimal contrastive pairs, revealing that current models rely heavily on static visual cues rather than genuine temporal reasoning. The three-level taxonomy and four-metric diagnostic hierarchy provide a reusable framework for any temporal evaluation pipeline.