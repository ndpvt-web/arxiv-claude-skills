---
name: "wildreward-learning-reward-in-the-wild"
description: "Build reward models from implicit user feedback in chat logs using ordinal regression instead of annotated preference pairs. Use when: 'train a reward model from user logs', 'extract feedback signals from conversations', 'build RM without preference annotations', 'ordinal regression reward model', 'score LLM responses from interaction data', 'mine implicit feedback from chat history'."
---

This skill enables Claude to build and deploy reward models trained on implicit human feedback extracted from real conversation logs, following the WildReward pipeline. Instead of requiring expensive human-annotated preference pairs (response A > response B), this approach mines satisfaction signals directly from user follow-up behavior in multi-turn chats -- regeneration requests signal dissatisfaction, continued engagement signals approval -- and trains via ordinal regression on a 4-level quality scale. The result is a calibrated reward model that produces globally consistent scores without ever seeing a single preference pair.

## When to Use

- When the user wants to train a reward model but lacks annotated preference data
- When the user has a corpus of multi-turn chat logs (e.g., from a deployed chatbot) and wants to extract training signal from them
- When building an RLHF or DPO pipeline and needing a scoring function for on-policy response ranking
- When the user asks about ordinal regression as an alternative to Bradley-Terry pairwise training
- When the user needs a reward model with calibrated confidence scores (not just relative rankings)
- When scaling reward model quality through user diversity rather than annotation volume

## Key Technique

**Implicit Feedback Mining from Chat Logs.** In a typical multi-turn conversation, the user's follow-up message carries implicit signal about the previous response's quality. A user who says "That's wrong, actually X is Y" is providing a correction (negative signal). A user who asks a deeper follow-up question is implicitly satisfied. The WildReward pipeline classifies each user follow-up into five levels: Explicit Rejection, Error Correction, Neutral Ambiguity, Positive Engagement, and Explicit Satisfaction. An LLM classifier performs this labeling with a conservative strategy that defaults to Neutral when evidence is weak. Two refinement passes follow: (1) semantic similarity matching reclassifies near-positive messages that were missed, expanding positive labels by ~29%, and (2) refusal validation corrects cases where a model rightfully refused a harmful request but received undeserved negative feedback.

**Ordinal Regression Instead of Pairwise Preferences.** Conventional reward models train on preference pairs using the Bradley-Terry objective, which only learns *relative* quality between two specific responses. WildReward maps the four non-neutral feedback categories to discrete scores 1-4 and trains with an ordinal regression loss: `L = -sum_{k=1}^{K-1} [I(y>k) log P(y>k|x) + (1-I(y>k)) log(1-P(y>k|x))]`. At inference, the reward score is `R(x) = 1 + sum_{k=1}^{K-1} P(y>k|x)`, producing a continuous value between 1 and 4. This formulation respects the ordinal structure (score 3 > score 2 > score 1) without assuming equal intervals, and yields inherently calibrated probabilities -- achieving 2.76% expected calibration error vs. 8.81% for pairwise-trained baselines.

**User Diversity as a Scaling Axis.** A key finding: at fixed dataset size, models trained on data from more unique users consistently outperform those with fewer contributors. This means collecting breadth of users matters more than depth of interactions per user, providing a concrete data collection strategy.

## Step-by-Step Workflow

1. **Prepare the chat corpus.** Collect multi-turn conversation logs in JSONL format. Each record needs: conversation history, the user query, and the model response. Filter to English/Chinese, remove multimodal or tool-dependent turns, cap at 20 turns per conversation, require queries >= 5 words and responses >= 10 words.

2. **Classify implicit feedback from follow-up messages.** For each (query, response, follow-up) triple, use an LLM to classify the follow-up into one of five satisfaction levels. Prompt the classifier with the full conversation context and instruct it to default to "Neutral Ambiguity" when signal is weak. Discard all Neutral instances -- they carry no training signal.

3. **Run implicit feedback mining refinement.** Compute sentence embeddings for all follow-up messages. For instances classified as Neutral, check if semantic similarity > 0.6 with any known positive-feedback message within a two-turn window. Reclassify matches as Positive Engagement.

4. **Run refusal validation.** For instances where the model refused a request and received negative feedback, use an LLM to verify whether the refusal was appropriate (e.g., the user asked for harmful content). Correct unjustified negative labels -- this prevents the reward model from penalizing safe refusals.

5. **Map to ordinal scores.** Assign numeric labels: Explicit Rejection = 1, Error Correction = 2, Positive Engagement = 3, Explicit Satisfaction = 4. Format each instance as `(conversation_context, query, response, score)`.

6. **Tokenize and prepare training data.** Format inputs as `[conversation_history] [query] [response]` sequences, truncated to 4096 tokens. Split off ~5000 instances for evaluation. Store as tokenized tensors.

7. **Train the reward model with ordinal regression.** Initialize from a pretrained LM (e.g., Qwen3-8B or Llama-3-8B). Add K-1 = 3 binary classification heads. Train with the ordinal regression loss for 1 epoch, batch size 512, learning rate 1e-5. Use DeepSpeed ZeRO-2 for multi-GPU training.

8. **Deploy the reward model as a scoring service.** Stand up inference workers (one per GPU) behind a round-robin router. Each request sends `(context, query, response)` and receives a continuous score in [1, 4]. Use FP16 for inference efficiency.

9. **Integrate into DPO or RLHF training.** For online DPO: sample N responses per prompt (N=8 works well), score all with the reward model, select the highest and lowest scoring as the chosen/rejected pair, and train with the DPO objective. This avoids distribution shift inherent in offline preference datasets.

10. **Monitor calibration.** Periodically evaluate Expected Calibration Error (ECE) and cross-sample consistency (ROC-AUC on pointwise binary classification). Use score margin thresholds (e.g., delta > 0.2) to filter low-confidence predictions when high precision is needed.

## Concrete Examples

**Example 1: Extracting Feedback from a Chat Log**

User: "I have 500k multi-turn chat logs from our customer support bot. I want to build a reward model without hiring annotators."

Approach:
1. Export logs to JSONL with fields: `conversation_id`, `turns` (list of role/content pairs)
2. For each assistant response followed by a user message, extract the triple: `(history, response, follow_up)`
3. Classify follow-ups using the 5-level schema via an LLM prompt:

```python
CLASSIFICATION_PROMPT = """Given this conversation context and the user's follow-up message,
classify the user's satisfaction with the assistant's previous response.

Categories:
1. Explicit Rejection - User directly rejects or expresses strong dissatisfaction
2. Error Correction - User corrects factual errors or points out mistakes
3. Neutral Ambiguity - No clear signal about satisfaction
4. Positive Engagement - User builds on the response, asks deeper questions
5. Explicit Satisfaction - User thanks, praises, or confirms the response helped

Conversation: {history}
Assistant response: {response}
User follow-up: {follow_up}

Default to "Neutral Ambiguity" if the signal is weak or unclear.
Classification:"""
```

4. Filter out Neutral instances, map remaining to scores 1-4
5. Result: ~37% of logs yield usable training signal (186k from 500k in the paper's case)

**Example 2: Training a WildReward Model**

User: "I have the labeled dataset. How do I train the ordinal regression reward model?"

Approach:
1. Implement the ordinal regression head and loss:

```python
import torch
import torch.nn as nn

class OrdinalRewardModel(nn.Module):
    def __init__(self, backbone, K=4):
        super().__init__()
        self.backbone = backbone
        hidden_size = backbone.config.hidden_size
        # K-1 binary classifiers for ordinal regression
        self.heads = nn.Linear(hidden_size, K - 1)

    def forward(self, input_ids, attention_mask):
        outputs = self.backbone(input_ids, attention_mask=attention_mask)
        # Use last token representation (reward modeling convention)
        hidden = outputs.last_hidden_state[:, -1, :]
        logits = self.heads(hidden)  # shape: (batch, K-1)
        return logits

    def compute_loss(self, logits, labels):
        """Ordinal regression loss. Labels are integers in [1, K]."""
        K = logits.size(1) + 1
        loss = 0
        for k in range(1, K):
            target = (labels > k).float()
            loss += nn.functional.binary_cross_entropy_with_logits(
                logits[:, k - 1], target
            )
        return loss

    def score(self, logits):
        """Continuous reward score: R(x) = 1 + sum P(y > k)."""
        probs = torch.sigmoid(logits)
        return 1.0 + probs.sum(dim=-1)
```

2. Train for 1 epoch with AdamW, lr=1e-5, batch size 512, sequence length 4096
3. At inference, call `model.score(logits)` to get a value in [1, 4]

**Example 3: Online DPO with the Reward Model**

User: "I want to use my trained reward model to improve an instruction-following LLM via DPO."

Approach:
1. Deploy the reward model as a scoring endpoint
2. For each training prompt, generate 8 candidate responses from the policy model
3. Score all 8 responses with the reward model
4. Select the highest-scored response as "chosen" and lowest as "rejected"
5. Train with standard DPO loss on these constructed pairs:

```bash
# Generate responses on-the-fly and score them
python online_dpo_train.py \
  --policy_model meta-llama/Llama-3.1-8B-Instruct \
  --reward_endpoint http://localhost:9000/score \
  --prompts_path data/infinity_instruct_20k.jsonl \
  --num_candidates 8 \
  --output_dir checkpoints/dpo_wildreward
```

Expected improvement: +8 points on AlpacaEval 2.0, +8 points on Arena Hard over the base model.

## Best Practices

- **Do:** Default the feedback classifier to "Neutral" when uncertain. Conservative labeling preserves data quality -- you lose ~63% of instances but the remaining signal is reliable.
- **Do:** Run refusal validation on negative-feedback instances involving safety topics. Without it, the reward model learns to penalize correct refusals, dropping safety accuracy from 90% to 28%.
- **Do:** Maximize user diversity in your training corpus. At the same dataset size, more unique users produce a stronger reward model than more interactions from fewer users.
- **Do:** Use the score margin as a confidence filter at inference. Setting a threshold of 0.2 on the score difference between candidates boosts accuracy to ~87% while retaining ~50% of predictions.
- **Avoid:** Training for more than 1 epoch on the feedback data. The ordinal regression objective converges quickly and overfitting degrades calibration.
- **Avoid:** Using this approach on conversations where follow-up messages are system-generated or templated (e.g., automated survey prompts). The signal must come from genuine user behavior.

## Error Handling

- **Noisy labels from ambiguous follow-ups:** The conservative Neutral default handles this. If accuracy on your held-out set drops below 75%, increase the classifier confidence threshold or add a human-in-the-loop verification step on edge cases.
- **Safety label corruption:** If you skip refusal validation, monitor the safety-refusal subset of RewardBench. A sudden drop (>10 points) indicates the model is penalizing appropriate refusals. Re-run the validation pass and retrain.
- **Score collapse (all outputs score similarly):** This typically means the backbone is too small for the data complexity or the learning rate is too high. Reduce lr to 5e-6 or switch to a larger backbone.
- **Deployment latency spikes:** The router-worker architecture should distribute load. If p99 latency exceeds acceptable bounds, add workers or reduce max sequence length at inference.

## Limitations

- Requires multi-turn conversation logs where users naturally follow up. Single-turn datasets or conversations where users always start fresh provide no implicit feedback signal.
- The 5-level satisfaction taxonomy was designed for English and Chinese. Other languages may have different follow-up patterns that the classifier misinterprets.
- Ordinal regression on 4 levels provides coarser signal than fine-grained pairwise preferences. For tasks requiring very precise ranking (e.g., poetry quality), pairwise annotation may still be superior.
- The approach inherits biases from the user population. If most users are technical, the reward model will underweight non-technical quality dimensions.
- Cannot learn from users who silently leave the conversation (no follow-up = no signal), which introduces survivorship bias toward engaged users.

## Reference

[WildReward: Learning Reward Models from In-the-Wild Human Interactions](https://arxiv.org/abs/2602.08829v1) -- Peng et al., 2026. Focus on Section 3 (feedback extraction pipeline), Section 4.2 (ordinal regression formulation), and Table 3 (user diversity ablation). Code: [github.com/THU-KEG/WildReward](https://github.com/THU-KEG/WildReward).