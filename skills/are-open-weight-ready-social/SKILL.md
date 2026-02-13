---
name: "are-open-weight-ready-social"
description: "Build LLM-based content moderation pipelines using zero-shot classification with open-weight models. Implements the structured prompting and evaluation methodology from Chou et al. (2026) for detecting rudeness, intolerance, and threats in social media posts. Use when: 'build a content moderation system', 'detect harmful posts with LLMs', 'moderate social media content', 'set up open-weight moderation pipeline', 'evaluate LLM moderation accuracy', 'privacy-preserving content filtering'."
---

# LLM-Based Social Media Moderation with Open-Weight Models

This skill enables Claude to build and evaluate content moderation systems that use open-weight LLMs for zero-shot harmful content detection. Based on Chou et al. (2026), the approach uses structured role-definition prompts with precise violation definitions to classify social media posts into categories (rudeness, intolerance, threats) without fine-tuning. The key finding: open-weight models running on a single 24GB GPU achieve sensitivity (81-97%) and specificity (91-100%) competitive with proprietary models, making privacy-preserving, on-premise moderation viable.

## When to Use

- When the user wants to build a content moderation API or service for a social platform (Bluesky, Mastodon, custom forums)
- When the user needs to classify user-generated text as harmful/safe without training data (zero-shot)
- When the user asks about running moderation locally for privacy (GDPR compliance, on-premise requirements)
- When the user wants to evaluate or benchmark multiple LLMs for moderation accuracy
- When the user needs to design a moderation taxonomy with clear violation definitions
- When the user is building personalized content filters that respect individual user preferences
- When the user asks how to handle edge cases like model refusals, verbose reasoning tokens, or ambiguous content

## Key Technique

The core method is **definition-anchored zero-shot classification**: each moderation category is paired with a precise natural-language definition, injected into a standardized prompt template with the role "You are a social media moderator." The model outputs a binary yes/no decision. No few-shot examples or fine-tuning are needed. The prompt template follows this structure:

```
Instruction: You are a social media moderator. Decide whether a given post is {category}
according to this definition: '{definition}'. Respond with either yes or no (lowercase).
Input: Post: {post_text}
Output: Response:
```

What makes this effective is the **tight coupling between definition and decision boundary**. The definition text (e.g., "Rude or impolite, including crude language and disrespectful comments, without constructive purpose") acts as the classification specification. By varying the definition, the same pipeline handles different violation types without architectural changes. The study found that specificity exceeds sensitivity for subjective categories like rudeness (fewer false positives, more missed violations), while sensitivity exceeds specificity for clear-cut violations like threats and intolerance (catches more true positives). This asymmetry is important for calibrating moderation thresholds.

For deployment, open-weight models (9B-30B parameters with mixture-of-experts architectures) run on a single NVIDIA RTX 3090 (24GB VRAM) using vLLM for inference. Temperature is set to 0 with a fixed random seed for deterministic outputs. Max output tokens should be capped (the study used 10,000 to accommodate reasoning traces) but the actual classification answer is extracted from the first token or short response.

## Step-by-Step Workflow

1. **Define your moderation taxonomy.** Write precise, one-sentence definitions for each violation category. Follow the pattern: "[Category name]: [Observable behavior], including [specific examples], [scope qualifier]." Example: "Rude: Rude or impolite, including crude language and disrespectful comments, without constructive purpose."

2. **Construct prompt templates for each category.** Use the role-definition-binary format: assign the moderator role, inject the category definition, present the post, and constrain output to "yes" or "no". Keep the template identical across categories except for the definition string.

3. **Select and deploy the model.** For privacy-preserving local deployment, choose an open-weight model that fits in 24GB VRAM (e.g., Qwen3-30B-A3B, Nemotron-Nano-9B). Serve via vLLM with `temperature=0` and a fixed `seed` for reproducibility. For cloud deployment, any reasoning-capable model works.

4. **Implement the classification endpoint.** Build an API that accepts post text, runs it through each category's prompt template, parses the binary response, and returns a structured moderation decision. Handle response parsing defensively: extract "yes"/"no" from potentially verbose reasoning output.

5. **Handle model refusals as positive signals.** When a model's safety filter triggers and refuses to process a post, interpret the refusal as a positive classification (the content was harmful enough to trigger built-in guardrails). Log these cases separately for review.

6. **Cap output tokens and extract the decision.** Set `max_tokens` high enough for reasoning models (4096-10000) but parse only the final answer. For reasoning models that emit `<think>` blocks, strip the reasoning and extract the yes/no from the response section.

7. **Evaluate with sensitivity and specificity, not just accuracy.** Compute true positive rate (sensitivity) and true negative rate (specificity) separately per category. Accuracy alone is misleading because harmful posts are rare (class imbalance). Report both metrics side by side.

8. **Compute inter-rater agreement if using multiple annotators or models.** Use Cohen's kappa for pairwise agreement and Fleiss' kappa for multi-rater scenarios. Agreement between LLMs and humans on par with human-human agreement (kappa > 0.6) indicates deployment readiness.

9. **Calibrate per-category thresholds.** For subjective categories (rudeness), accept higher false-negative rates to avoid over-censorship. For safety-critical categories (threats), bias toward higher sensitivity even at the cost of more false positives. Make these trade-offs configurable.

10. **Build the personalization layer.** Allow users to adjust their own sensitivity thresholds per category. The same underlying model serves both platform-wide enforcement (strict definitions) and personal filtering (user-tuned definitions or thresholds).

## Concrete Examples

**Example 1: Building a Bluesky moderation microservice**

User: "I want to build a content moderation API for my Bluesky-compatible service that runs locally for privacy."

Approach:
1. Define three violation categories with definitions matching Bluesky Moderation Service taxonomy
2. Create a FastAPI service with a `/moderate` endpoint
3. Load an open-weight model via vLLM on local GPU
4. For each incoming post, run all three category prompts in parallel
5. Return structured JSON with per-category verdicts

Output:
```python
# moderation_service.py
from fastapi import FastAPI
from pydantic import BaseModel
from vllm import LLM, SamplingParams

app = FastAPI()

CATEGORIES = {
    "rude": "Rude or impolite, including crude language and disrespectful comments, without constructive purpose",
    "intolerant": "Discrimination against protected groups",
    "threat": "Promotes violence or harm towards others, including threats, incitement, or advocacy of harm",
}

PROMPT_TEMPLATE = (
    "Instruction: You are a social media moderator. Decide whether a given post "
    "is {category} according to this definition: '{definition}'. "
    "Respond with either yes or no (lowercase).\n"
    "Input: Post: {post_text}\n"
    "Output: Response:"
)

llm = LLM(model="Qwen/Qwen3-30B-A3B", gpu_memory_utilization=0.9)
params = SamplingParams(temperature=0, max_tokens=4096, seed=350)


class Post(BaseModel):
    text: str


class ModerationResult(BaseModel):
    rude: bool
    intolerant: bool
    threat: bool
    flagged: bool


def parse_decision(output_text: str) -> bool:
    """Extract yes/no from model output, handling reasoning traces."""
    cleaned = output_text.strip().lower()
    # For reasoning models, look after </think> tag
    if "</think>" in cleaned:
        cleaned = cleaned.split("</think>")[-1].strip()
    if cleaned.startswith("yes"):
        return True
    if cleaned.startswith("no"):
        return False
    # Model refusal = treat as flagged
    if "i cannot" in cleaned or "i'm unable" in cleaned:
        return True
    return "yes" in cleaned.split("\n")[0]


@app.post("/moderate", response_model=ModerationResult)
async def moderate(post: Post):
    prompts = [
        PROMPT_TEMPLATE.format(
            category=cat, definition=defn, post_text=post.text
        )
        for cat, defn in CATEGORIES.items()
    ]
    outputs = llm.generate(prompts, params)
    decisions = {
        cat: parse_decision(out.outputs[0].text)
        for cat, out in zip(CATEGORIES.keys(), outputs)
    }
    decisions["flagged"] = any(decisions.values())
    return ModerationResult(**decisions)
```

**Example 2: Evaluating moderation model accuracy**

User: "I have a labeled dataset of posts. How do I evaluate which LLM is best for moderation?"

Approach:
1. Load the labeled dataset with ground-truth binary labels per category
2. Run each candidate model on every post using the standard prompt template
3. Compute sensitivity, specificity, and accuracy per category per model
4. Compute inter-rater agreement (Cohen's kappa) between each model and ground truth

Output:
```python
# evaluate_moderation.py
import numpy as np
from sklearn.metrics import confusion_matrix, cohen_kappa_score
import json

def evaluate_model(predictions: list[bool], ground_truth: list[bool], label: str):
    tn, fp, fn, tp = confusion_matrix(ground_truth, predictions).ravel()
    sensitivity = tp / (tp + fn) if (tp + fn) > 0 else 0.0
    specificity = tn / (tn + fp) if (tn + fp) > 0 else 0.0
    accuracy = (tp + tn) / (tp + tn + fp + fn)
    kappa = cohen_kappa_score(ground_truth, predictions)

    return {
        "category": label,
        "sensitivity": round(sensitivity, 3),
        "specificity": round(specificity, 3),
        "accuracy": round(accuracy, 3),
        "cohens_kappa": round(kappa, 3),
        "tp": int(tp), "fp": int(fp), "fn": int(fn), "tn": int(tn),
    }

# Example usage with results from multiple models
models = ["qwen3-30b", "nemotron-9b", "gpt-4o", "gemini-2.5-pro"]
for model_name in models:
    preds = load_predictions(model_name)  # list of bool
    truth = load_ground_truth()            # list of bool
    for category in ["rude", "intolerant", "threat"]:
        result = evaluate_model(
            preds[category], truth[category], category
        )
        print(f"{model_name} | {json.dumps(result)}")
```

**Example 3: Adding personalized moderation filters**

User: "I want users to be able to set their own moderation sensitivity for different content types."

Approach:
1. Store per-user threshold preferences for each violation category
2. Run the standard zero-shot classification to get a raw decision
3. For borderline cases, use a confidence-based approach: run the prompt N times with slight temperature variation and compute agreement ratio
4. Compare agreement ratio against the user's threshold

Output:
```python
# personalized_moderation.py
from dataclasses import dataclass

@dataclass
class UserPreferences:
    rude_threshold: float = 0.5       # 0.0 = show everything, 1.0 = hide aggressively
    intolerant_threshold: float = 0.7
    threat_threshold: float = 0.9     # most users want strict threat filtering

def personalized_moderate(
    post_text: str,
    user_prefs: UserPreferences,
    llm_client,
    n_samples: int = 5
) -> dict[str, bool]:
    """Run moderation with user-specific sensitivity thresholds."""
    results = {}
    for category, threshold in [
        ("rude", user_prefs.rude_threshold),
        ("intolerant", user_prefs.intolerant_threshold),
        ("threat", user_prefs.threat_threshold),
    ]:
        # Sample multiple times with low temperature variation
        votes = []
        for i in range(n_samples):
            prompt = build_prompt(category, post_text)
            response = llm_client.generate(prompt, temperature=0.1 * i, seed=350 + i)
            votes.append(parse_decision(response))

        confidence = sum(votes) / len(votes)
        results[category] = confidence >= threshold

    return results
```

## Best Practices

- **Do:** Use precise, self-contained violation definitions in every prompt. The definition IS the classifier specification -- vague definitions produce inconsistent results.
- **Do:** Set `temperature=0` and fix the random seed for reproducible moderation decisions. Non-deterministic moderation erodes user trust.
- **Do:** Evaluate sensitivity and specificity independently per category. A model with 95% accuracy can still miss 50% of threats if threats are rare.
- **Do:** Treat model safety-filter refusals as positive detections and log them for human review rather than discarding them.
- **Avoid:** Using accuracy alone as your evaluation metric. With 99% benign posts, a model that always says "no" gets 99% accuracy but catches zero violations.
- **Avoid:** Applying the same sensitivity/specificity trade-off to all categories. Threats require high sensitivity (catch them all); rudeness requires high specificity (avoid over-censorship).
- **Avoid:** Fine-tuning on moderation data without considering that definitions shift over time. Zero-shot with updated definitions is more maintainable than a stale fine-tuned model.

## Error Handling

- **Verbose reasoning output:** Reasoning models (DeepSeek-R1, Qwen3-Thinking) emit long `<think>` blocks before the answer. Always strip reasoning traces and extract the final yes/no. Set max_tokens high enough (4096+) to avoid truncation before the answer appears.
- **Token limit exceeded:** If a model hits the token limit mid-reasoning, it may never output the final answer. Detect truncated responses (no "yes" or "no" found) and either retry with higher token limit or fall back to a non-reasoning model.
- **Ambiguous output:** Models sometimes output "Yes, but..." or hedging language. Implement strict parsing: check only the first word after stripping reasoning. If neither "yes" nor "no", flag for human review.
- **Content filter blocking the prompt:** Some API providers block the moderation prompt itself because it contains harmful content definitions. Use provider-specific safety setting overrides (e.g., Google's `HarmBlockThreshold.BLOCK_NONE`) or host the model locally where you control the safety layer.
- **Class imbalance in evaluation:** Harmful posts are rare (< 1% of traffic). When evaluating, use stratified sampling to ensure enough positive examples. The original study found only 26-202 labeled violations across millions of posts.

## Limitations

- **Zero-shot accuracy ceiling:** Open-weight models achieve 81-97% sensitivity, which means 3-19% of harmful posts are missed. For high-stakes moderation (child safety, terrorism), zero-shot alone is insufficient -- pair with keyword filters and human review.
- **Subjective categories are inherently noisy.** Inter-rater agreement on "rudeness" is lower even among humans. LLMs reflect this disagreement. Don't expect perfect consistency on subjective calls.
- **English-only validation.** The study filtered to English posts only. Moderation quality for other languages is unvalidated and likely lower for smaller open-weight models.
- **Static definitions don't capture context.** Sarcasm, in-group language, and cultural context can flip whether a post is harmful. Zero-shot binary classification cannot reliably detect these nuances.
- **Latency at scale.** Running a 30B-parameter model per post per category is slow for real-time moderation of high-volume feeds. Batch processing or a tiered system (fast keyword filter -> LLM for borderline cases) is necessary for production.
- **Model updates change behavior.** Upgrading the model version can silently shift moderation boundaries. Always re-evaluate after model changes.

## Reference

Chou, H.-Y., Naveed, W., Zhou, S., & Yang, X. (2026). *Are Open-Weight LLMs Ready for Social Media Moderation? A Comparative Study on Bluesky.* arXiv:2602.05189v1. [https://arxiv.org/abs/2602.05189v1](https://arxiv.org/abs/2602.05189v1)

Key takeaway: Open-weight models (9B-30B) match proprietary LLMs on moderation accuracy using zero-shot definition-anchored prompts, enabling privacy-preserving deployment on a single 24GB consumer GPU.