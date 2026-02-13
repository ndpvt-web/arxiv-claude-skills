---
name: "genius-generative-fluid-intelligence"
description: "Evaluate and improve generative AI outputs for fluid intelligence tasks -- pattern induction from context, ad-hoc constraint execution, and contextual knowledge adaptation. Use when: 'evaluate image generation model', 'benchmark multimodal model reasoning', 'build GFI evaluation pipeline', 'score generative model compliance', 'attention intervention for context comprehension', 'assess fluid intelligence in vision models'."
---

# GENIUS: Generative Fluid Intelligence Evaluation Suite

This skill enables Claude to build evaluation pipelines and attention-intervention systems for assessing **Generative Fluid Intelligence (GFI)** in multimodal models, based on the GENIUS benchmark framework. Unlike standard benchmarks that test memorized knowledge (crystallized intelligence), GFI measures a model's ability to induce implicit patterns from context, execute novel ad-hoc constraints, and adapt to contextually-redefined knowledge -- all without prior training on the specific task. Claude can use this to design evaluation harnesses, implement the three-metric scoring system (Rule Compliance, Visual Consistency, Aesthetic Quality), build LMM-as-a-judge pipelines, and apply the training-free attention intervention strategy to improve model outputs.

## When to Use

- When the user wants to evaluate a multimodal generative model's reasoning ability beyond simple prompt-following
- When building a benchmark or test suite that measures whether a vision-language model can follow novel, context-defined rules (not pre-learned patterns)
- When implementing an LMM-as-a-judge evaluation pipeline that scores generated images against structured criteria
- When the user asks to improve a model's context comprehension without retraining, using attention-layer interventions
- When designing evaluation tasks that require interleaved multimodal context (text + images together, where removing either modality makes the task unsolvable)
- When scoring generative outputs on a weighted multi-axis rubric (compliance, consistency, aesthetics)
- When diagnosing whether a generative model's failures stem from comprehension gaps vs. generation capability limits

## Key Technique

GENIUS decomposes fluid intelligence into three testable primitives. **Inducing Implicit Patterns** requires models to infer unstated preferences from examples (e.g., given several images a user liked, generate one matching their latent style preference). **Executing Ad-hoc Constraints** tests whether models can follow novel symbolic rules defined only in-context (e.g., "a blue square means 'delete the object'" -- now apply that rule). **Adapting to Contextual Knowledge** asks models to override prior knowledge with context-provided facts (e.g., "in this world, heavier objects are determined by color, not material" -- now generate accordingly). Each primitive is designed so the task is unsolvable without fully processing the multimodal context.

The evaluation uses a three-axis scoring system weighted 6:3.5:0.5 for **Rule Compliance** (RC), **Visual Consistency** (VC), and **Aesthetic Quality** (AQ). RC checks whether specific nouns, adjectives, and spatial predicates match curated evaluation hints. VC assesses element stability across generated outputs with anti-plagiarism screening. AQ evaluates anatomical logic, lighting, and artifact presence. A multimodal LLM judge (e.g., Gemini or Qwen2.5-VL-72B) scores each axis on a 0/1/2 scale across three independent runs.

The paper's key actionable insight is the **training-free attention intervention**. It works in three stages: (1) distill task-critical visual cues into region-specific keywords, (2) compute a semantic relevance map aligning keywords to visual context tokens, (3) inject a spatial bias into attention logits via `Attention = softmax((A + lambda * F(S)) / sqrt(d)) * V`, where `F(S)` amplifies high-relevance tokens and suppresses noise. This forces the model to attend to context-relevant regions without any fine-tuning, directly addressing the finding that failures stem from attention dispersion rather than insufficient generative capacity.

## Step-by-Step Workflow

1. **Define the evaluation dimension.** Choose which GFI primitive to test: implicit pattern induction (86 samples in GENIUS), ad-hoc constraint execution (213 samples across visual and symbolic sub-tasks), or contextual knowledge adaptation (211 samples across prior-conflicting and multi-semantic sub-tasks).

2. **Structure the test data.** Create a `test_data.json` for each dimension. Each entry needs a unique `id`, interleaved multimodal context (text instructions + reference images), a generation prompt, and curated `eval_hints` listing the specific nouns, adjectives, and spatial predicates the output must contain.

3. **Generate model outputs.** Run target models against each test case, saving outputs as `outputs/<model_name>/<task_name>/<id>.png`. Ensure the `id` in the filename matches the `test_data.json` entry exactly (IDs may not be sequential).

4. **Configure the LMM-as-a-judge pipeline.** Set up an API endpoint for a strong vision-language model (Gemini-3-Pro recommended, or Qwen2.5-VL-72B as open-source alternative). Configure `API_URL` and `API_KEY` in your evaluation script.

5. **Build evaluation prompts per axis.** For Rule Compliance: ask the judge to verify each eval-hint element against the generated image (0=absent, 1=partial, 2=fully present). For Visual Consistency: check element stability and screen for exact pixel duplication. For Aesthetic Quality: assess anatomical correctness, lighting coherence, and AI artifact presence.

6. **Run triple-pass evaluation.** Execute each scoring prompt three times independently per sample and average the scores to reduce judge variance. Compute the weighted composite: `score = 6*RC + 3.5*VC + 0.5*AQ` (normalized to percentage).

7. **Run diagnostic comprehension probes.** Reformulate generation tasks as multiple-choice VQA questions to test whether the model understands the context even when it fails to generate correctly. This disentangles comprehension failures from execution failures.

8. **Apply attention intervention (if improving a model).** Implement the three-stage pipeline: extract keywords from the task prompt, compute token-level relevance scores by measuring semantic alignment with context tokens, then inject the relevance map as additive bias into the model's attention layers before the softmax.

9. **Compare pre- and post-intervention scores.** Re-run the evaluation pipeline on intervention-enhanced outputs to measure improvement, particularly on Rule Compliance which is the dominant failure mode.

10. **Aggregate and report results.** Compute per-dimension and per-sub-task scores. Flag tasks where AQ is high but RC is low -- this pattern indicates the model generates aesthetically pleasing images that ignore the actual constraints (the most common failure mode found in the paper).

## Concrete Examples

**Example 1: Building a GFI evaluation harness for a custom model**

User: "I have a multimodal model that generates images from interleaved text+image prompts. I want to evaluate whether it can follow novel rules defined in-context."

Approach:
1. Design 3 test categories matching the GFI primitives
2. Create test cases with interleaved context that defines novel rules
3. Define eval_hints for each test case
4. Set up LMM judge scoring

```python
# test_data.json structure for ad-hoc constraint execution
{
  "task": "visual_constraint",
  "samples": [
    {
      "id": "vc_017",
      "context": [
        {"type": "text", "content": "In this system, a blue square means 'delete the object'."},
        {"type": "image", "path": "context/blue_square_example.png"},
        {"type": "text", "content": "A red circle means 'enlarge the object'."},
        {"type": "image", "path": "context/red_circle_example.png"}
      ],
      "prompt": "Apply the blue square to the cat in this scene.",
      "prompt_image": "context/scene_with_cat.png",
      "eval_hints": {
        "required_absent": ["cat"],
        "required_present": ["scene background", "other objects unchanged"],
        "spatial": ["cat region should be empty or filled naturally"]
      }
    }
  ]
}
```

```python
# Evaluation scoring with LMM judge
import json, statistics

def evaluate_sample(judge_api, generated_image, eval_hints, num_runs=3):
    scores = {"RC": [], "VC": [], "AQ": []}

    for _ in range(num_runs):
        # Rule Compliance: check eval_hints against generated image
        rc_prompt = (
            f"Score 0/1/2: Does this image comply with these rules?\n"
            f"Must be absent: {eval_hints['required_absent']}\n"
            f"Must be present: {eval_hints['required_present']}\n"
            f"Spatial requirements: {eval_hints['spatial']}"
        )
        scores["RC"].append(judge_api.score(generated_image, rc_prompt))

        # Visual Consistency: element stability check
        vc_prompt = "Score 0/1/2: Are visual elements stable and coherent? Check for duplication artifacts."
        scores["VC"].append(judge_api.score(generated_image, vc_prompt))

        # Aesthetic Quality: lighting, anatomy, artifacts
        aq_prompt = "Score 0/1/2: Rate anatomical logic, lighting coherence, and absence of AI artifacts."
        scores["AQ"].append(judge_api.score(generated_image, aq_prompt))

    # Weighted composite (RC:VC:AQ = 6:3.5:0.5)
    rc = statistics.mean(scores["RC"])
    vc = statistics.mean(scores["VC"])
    aq = statistics.mean(scores["AQ"])
    composite = (6 * rc + 3.5 * vc + 0.5 * aq) / (6 + 3.5 + 0.5) * 100
    return {"RC": rc, "VC": vc, "AQ": aq, "composite": composite}
```

Output: Per-sample and aggregate scores showing RC/VC/AQ breakdown plus weighted composite.

---

**Example 2: Diagnosing comprehension vs. generation failures**

User: "My model scores poorly on the constraint tasks. How do I figure out if it doesn't understand the rules or just can't generate correctly?"

Approach:
1. Convert generation tasks into multiple-choice VQA probes
2. Compare VQA accuracy against generation compliance
3. Identify the gap

```python
# Convert a generation task into a diagnostic VQA probe
def create_comprehension_probe(sample):
    """Turn a generation task into a multiple-choice question."""
    return {
        "context": sample["context"],  # same interleaved context
        "question": f"If you apply the rule to: {sample['prompt']}, which outcome is correct?",
        "choices": [
            sample["eval_hints"]["required_present"][0],  # correct
            "The scene remains completely unchanged",       # distractor
            "All objects are removed",                      # distractor
            "The background changes color"                  # distractor
        ],
        "correct": 0
    }

# Diagnostic analysis
def diagnose_failure_mode(generation_scores, comprehension_scores):
    """
    If comprehension is high but generation is low:
      -> execution gap (attention intervention will help)
    If both are low:
      -> fundamental comprehension failure (need better context processing)
    """
    for sample_id in generation_scores:
        gen = generation_scores[sample_id]["RC"]
        comp = comprehension_scores[sample_id]
        if comp >= 1.5 and gen < 1.0:
            print(f"{sample_id}: EXECUTION GAP - understands but can't generate")
        elif comp < 1.0 and gen < 1.0:
            print(f"{sample_id}: COMPREHENSION FAILURE - doesn't understand context")
```

Output: Classification of each failure as execution-gap (amenable to attention intervention) or comprehension-failure (needs architectural changes).

---

**Example 3: Implementing the attention intervention**

User: "I want to apply the training-free attention fix from the GENIUS paper to my model."

Approach:
1. Extract task-critical keywords from the prompt
2. Compute relevance scores between keywords and visual tokens
3. Inject bias into attention computation

```python
import torch
import torch.nn.functional as F

def keyword_distillation(prompt, context_text):
    """Stage 1: Extract task-critical visual cue keywords."""
    # Use an LLM to extract region-specific keywords from the task
    extraction_prompt = (
        f"Given this task context:\n{context_text}\n"
        f"And this generation prompt:\n{prompt}\n"
        f"List the 3-5 most critical visual keywords that the model "
        f"must attend to in the context images."
    )
    # Returns e.g. ["blue square", "deletion operation", "cat", "scene background"]
    return llm_extract(extraction_prompt)

def compute_relevance_map(keywords, visual_tokens, text_encoder):
    """Stage 2: Semantic relevance between keywords and visual context tokens."""
    keyword_embeds = text_encoder(keywords)          # [K, D]
    # visual_tokens: [N, D] from the model's context encoding
    similarity = keyword_embeds @ visual_tokens.T    # [K, N]
    relevance = similarity.max(dim=0).values         # [N] - max relevance per token
    return relevance

def attention_with_intervention(Q, K, V, relevance_map, lambda_scale=0.5):
    """Stage 3: Inject relevance bias into attention logits."""
    d_k = Q.size(-1)
    attn_logits = Q @ K.transpose(-2, -1)  # standard attention

    # F(S): amplify high-relevance, suppress low-relevance
    # Normalize relevance to [0, 1], then shift to center around 0
    S_norm = (relevance_map - relevance_map.mean()) / (relevance_map.std() + 1e-8)
    bias = lambda_scale * S_norm.unsqueeze(0).unsqueeze(0)  # broadcast to attention shape

    attn_weights = F.softmax((attn_logits + bias) / (d_k ** 0.5), dim=-1)
    return attn_weights @ V
```

Output: Modified attention layers that redirect model focus toward context-relevant tokens, improving Rule Compliance scores without retraining.

## Best Practices

- **Do:** Always use interleaved multimodal context where removing either modality makes the task unsolvable. This is the defining characteristic of GFI tasks -- if the task can be solved from text alone, it tests crystallized intelligence, not fluid intelligence.
- **Do:** Run the LMM judge three times per sample and average. Single-run scores have high variance (the paper validates this with Pearson r=0.963 against human experts across triple runs).
- **Do:** Weight Rule Compliance heavily (6x) over Aesthetic Quality (0.5x). High AQ with low RC is the most common failure pattern -- models generate beautiful images that ignore constraints entirely.
- **Do:** Use the diagnostic VQA probes before applying attention intervention. If the model fails comprehension probes, attention intervention won't help -- the problem is upstream.
- **Avoid:** Using the same model as both generator and judge. The judge must be a separate, strong vision-language model to avoid self-evaluation bias.
- **Avoid:** Testing with tasks solvable from pre-trained knowledge alone. If a model could plausibly answer without reading the context (e.g., "draw a cat"), you're testing crystallized intelligence, not GFI.
- **Avoid:** Setting `lambda_scale` too high in the attention intervention. Values above 1.0 tend to over-constrain generation and degrade aesthetic quality. Start at 0.3-0.5 and tune on a validation split.

## Error Handling

- **Judge API returns inconsistent scores across runs:** If the standard deviation across three runs exceeds 0.8 on the 0-2 scale, flag the sample for human review. Likely the eval_hints are ambiguous.
- **Model outputs exact pixel duplicates of context images:** This is a plagiarism/copying failure, not generation. The VC scoring must include anti-duplication screening (pixel-level or perceptual hash comparison against all context images).
- **Attention intervention degrades output quality:** Reduce `lambda_scale` or restrict intervention to specific attention heads rather than all layers. The paper shows mid-to-late layers benefit most.
- **Test data IDs don't match output filenames:** The GENIUS dataset uses non-sequential IDs. Always parse IDs from `test_data.json` rather than assuming 0-indexed sequences.
- **Eval hints are too specific or too vague:** RC scoring becomes unreliable at extremes. Each hint should name exactly one verifiable visual element (a noun, adjective, or spatial predicate).

## Limitations

- The attention intervention requires access to model internals (attention layers). It cannot be applied to black-box API models -- only to models where you can modify the forward pass.
- The LMM-as-a-judge approach inherits the judge model's biases. Gemini-3-Pro and Qwen2.5-VL-72B are validated judges; using weaker models will produce unreliable scores.
- GFI evaluation is designed for generative vision models (image generation from multimodal prompts). It does not directly apply to text-only generation or discriminative tasks.
- The 510-sample GENIUS dataset covers 20 sub-tasks. For specialized domains (medical imaging, technical diagrams), you will need to create domain-specific test cases following the same three-primitive structure.
- The weighted scoring formula (6:3.5:0.5) was calibrated for general-purpose evaluation. Domain-specific applications may need reweighting (e.g., medical imaging should weight RC even higher).

## Reference

[GENIUS: Generative Fluid Intelligence Evaluation Suite](https://arxiv.org/abs/2602.11144v1) -- An, Yang, Guo, Dai, Shen (2026). Focus on Section 4 for the attention intervention formalization (Theorems 4.1-4.2 linking attention magnitude to implicit gradient descent), Section 3 for the three-primitive taxonomy and eval_hints structure, and Section 5 for diagnostic analysis methodology. Code: [github.com/arctanxarc/GENIUS](https://github.com/arctanxarc/GENIUS).