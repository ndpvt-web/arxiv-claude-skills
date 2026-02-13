---
name: "high-fidelity-textual-user"
description: "Build RL-optimized unified textual user representations from heterogeneous data sources (profiles, activity logs, search history). Use when asked to: 'create a user representation from multiple data sources', 'synthesize user profiles for LLM-based recommendations', 'build personalized user summaries from logs and profiles', 'generate concise user descriptions for downstream models', 'combine heterogeneous user signals into text', 'optimize user representations with engagement feedback'."
---

# High Fidelity Textual User Representation via Reinforcement Learning

This skill enables Claude to design and implement systems that synthesize unified, concise textual representations of users from heterogeneous data sources -- profiles, behavioral logs, search queries, and engagement histories -- using a reinforcement learning framework. The core technique, from LinkedIn research (arXiv:2602.07333), uses Group Relative Policy Optimization (GRPO) to train an LLM policy that distills multi-source user data into a single interpretable text summary, optimized directly against implicit engagement signals (clicks, applies, purchases) rather than requiring hand-labeled training data.

## When to Use

- When the user needs to combine multiple user data sources (profiles, activity logs, search history, transaction records) into a single textual representation for downstream LLM consumption
- When building a recommendation system that needs interpretable, text-based user embeddings compatible with LLM pipelines
- When the user asks to replace fragmented feature engineering with a unified user summary that an LLM can reason over
- When designing a personalization system where latency constraints demand a pre-computed, compact user representation rather than feeding raw heterogeneous data at inference time
- When the user wants to optimize user representations end-to-end against business metrics (CTR, conversion, engagement) without manually labeling training data
- When migrating a traditional ID-based or embedding-based user model to an LLM-native text-based representation

## Key Technique

**The core insight** is treating user representation generation as a reinforcement learning problem rather than a supervised learning or template-filling task. An LLM acts as the policy model: it receives heterogeneous user data as its state (profile fields, job history, recent search queries, interaction logs) and generates a structured textual summary as its action. The reward comes from two sources: (1) implicit engagement signals from downstream tasks -- if a recommendation system using this representation produces clicks or applications, the representation was good; (2) rule-based rewards that enforce formatting constraints like length limits, required sections, and structural consistency. This dual reward avoids the need for human-labeled "good representation" examples entirely.

**GRPO (Group Relative Policy Optimization)** is the training algorithm. Unlike standard PPO which requires a separate value network, GRPO generates multiple candidate representations (rollouts) for the same user, evaluates each against the reward function, computes advantages relative to the group mean, and updates the policy via advantage-weighted gradient ascent. This is more stable and sample-efficient for LLM policies than PPO, and avoids the complexity of training a critic model. In practice, you generate K candidate summaries per user, score each, and use the relative ranking to update the generator.

**Why this matters for implementation:** The resulting representations are (a) interpretable text that humans and LLMs can read, (b) optimized for actual business outcomes rather than proxy metrics, (c) compact enough for latency-sensitive serving, and (d) naturally compatible with any downstream LLM system without adapter layers or embedding alignment.

## Step-by-Step Workflow

1. **Define and catalog heterogeneous data sources.** Enumerate every user data source available: structured profiles (name, title, skills), semi-structured data (job history, education), behavioral logs (search queries, click sequences, dwell times), and engagement signals (applies, purchases, saves). Create a schema for each source specifying field names, types, and expected cardinality.

2. **Design the input prompt template.** Build a structured prompt that presents all user data sources to the LLM policy. Use clearly labeled sections with delimiters:
   ```
   === PROFILE ===
   Title: {title} | Location: {location} | Skills: {skills}
   === WORK HISTORY ===
   {formatted_job_entries}
   === RECENT SEARCHES (last 30 days) ===
   {search_queries_with_timestamps}
   === ENGAGEMENT SIGNALS ===
   Clicked: {recent_clicked_items} | Applied: {recent_applied_items}
   === TASK ===
   Generate a concise user representation summarizing this member's
   professional identity, current intent, and preferences. Max 200 words.
   ```

3. **Define the target output format.** Specify the structure of the generated representation -- typically 3-5 labeled sections (Professional Identity, Current Intent, Key Skills, Preferences) with strict length bounds (e.g., 100-200 tokens total). Encode these constraints explicitly in both the prompt and the rule-based reward.

4. **Implement the rule-based reward function.** Score candidate representations on structural compliance:
   - Length penalty: `R_len = -alpha * max(0, |tokens| - max_len) - beta * max(0, min_len - |tokens|)`
   - Format check: +1 if all required sections present, -1 per missing section
   - Deduplication: penalize verbatim copying from input (reward paraphrasing/synthesis)
   - Coherence: penalize repetition of phrases within the output

5. **Implement the engagement-based reward function.** Use downstream task performance as the primary reward signal. For each candidate representation, feed it to the downstream model (e.g., a recommendation ranker) and measure the quality of resulting predictions against held-out engagement data:
   - Compute NDCG@K or hit-rate on a validation set of user-item interactions
   - Normalize rewards across the group of K rollouts per user to get relative advantages

6. **Set up the GRPO training loop.** For each training batch:
   - Sample a batch of users with their heterogeneous data
   - Generate K candidate representations per user (K=4 to 8 is typical)
   - Score each candidate with combined reward: `R = w_engage * R_engagement + w_rule * R_rules`
   - Compute group-relative advantages: `A_i = (R_i - mean(R_group)) / std(R_group)`
   - Update the LLM policy with advantage-weighted log-probability gradient ascent
   - Apply KL penalty against a reference (initial) policy to prevent reward hacking

7. **Build the offline evaluation pipeline.** Before any online deployment, evaluate on held-out users:
   - Measure downstream recommendation quality (NDCG, MRR, recall)
   - Measure representation quality (avg length, format compliance rate, vocabulary diversity)
   - Compare against baselines: raw concatenation, template-based summaries, supervised summarization

8. **Implement inference-time caching and serving.** Pre-compute representations for all active users in batch, store in a key-value store (Redis, DynamoDB), and refresh on a schedule (e.g., daily or on significant profile updates). Serve the pre-computed text representation at query time rather than generating on-the-fly.

9. **Set up incremental retraining.** As new engagement data accumulates, retrain the policy periodically. Use the previous policy checkpoint as the reference model for KL regularization. Monitor for reward hacking (representations that game the reward but lose interpretability).

10. **Instrument monitoring and drift detection.** Track representation statistics over time: average length, section coverage, vocabulary shift, and downstream metric correlation. Alert on significant distributional shifts that may indicate data pipeline issues or model degradation.

## Concrete Examples

**Example 1: Job Platform User Representation**

User: "I have a database of LinkedIn-style user profiles with job history, skills, and their recent job search queries. I want to create a compact text summary for each user that I can feed into an LLM-based job recommender."

Approach:
1. Extract and format each user's profile fields, work history, and last 20 search queries
2. Define target format: 150-token summary with sections [Professional Identity | Current Intent | Key Skills]
3. Initialize an LLM policy (e.g., fine-tuned Llama 3 8B) with a supervised warmstart on template-generated examples
4. Define rewards: engagement reward from click-through on job recommendations, rule reward for length (100-200 tokens) and section presence
5. Train with GRPO: K=6 rollouts per user, 10K users per epoch, KL coefficient=0.04

Output representation:
```
[Professional Identity] Senior backend engineer with 8 years in
distributed systems at mid-to-large tech companies. Strong Python and
Go background with recent Kubernetes certification.
[Current Intent] Actively exploring principal/staff-level backend roles
at companies with strong remote-work policies. Interested in platform
and infrastructure teams.
[Key Skills] Distributed systems, microservices, Python, Go, Kubernetes,
system design, technical leadership.
```

**Example 2: E-commerce Shopper Profile Synthesis**

User: "We have shopper profiles, browsing history, purchase records, and search logs. Build a system that creates a text-based shopper summary optimized for our LLM-powered product recommendation engine."

Approach:
1. Schema: profile (demographics, preferences), browsing (last 50 page views with categories), purchases (last 6 months), searches (last 30 queries)
2. Input template concatenates all sources with labeled sections
3. Target output: 100-150 tokens with [Shopper Profile | Purchase Patterns | Current Interest]
4. Rule rewards: length compliance, no PII leakage (penalize if output contains email/phone), section completeness
5. Engagement reward: measure recommendation NDCG on held-out purchase data when using the generated summary as user context
6. GRPO training with K=4 rollouts, combined reward weight 0.7 engagement + 0.3 rules

Output representation:
```
[Shopper Profile] Budget-conscious parent, suburban household,
primarily shops home goods and children's clothing. Prefers
mid-range brands with free shipping.
[Purchase Patterns] Seasonal buyer with spikes in back-to-school
and holiday periods. Average order value $45-65. High repeat
purchase rate for household consumables.
[Current Interest] Browsing outdoor furniture and garden tools,
suggesting spring home improvement planning. Recent searches
focus on patio sets under $500.
```

**Example 3: Implementing the GRPO Training Loop in Python**

User: "Show me how to implement the GRPO reward computation for user representation training."

```python
import torch
import torch.nn.functional as F

def compute_grpo_loss(
    policy_model,
    ref_model,
    user_contexts: list[str],
    num_rollouts: int = 6,
    kl_coeff: float = 0.04,
    reward_fn=None,  # callable: (context, candidate) -> float
):
    """
    GRPO training step for user representation generation.
    """
    all_losses = []

    for context in user_contexts:
        # Generate K candidate representations
        candidates = []
        log_probs = []
        for _ in range(num_rollouts):
            output = policy_model.generate(
                context, do_sample=True, temperature=0.8, max_new_tokens=200
            )
            candidates.append(output.text)
            log_probs.append(output.log_prob_sum)

        # Score each candidate
        rewards = torch.tensor([reward_fn(context, c) for c in candidates])

        # Group-relative advantage normalization
        advantages = (rewards - rewards.mean()) / (rewards.std() + 1e-8)

        # Compute KL penalty against reference policy
        with torch.no_grad():
            ref_log_probs = torch.tensor([
                ref_model.score(context, c) for c in candidates
            ])
        kl_penalty = torch.stack(log_probs) - ref_log_probs

        # GRPO objective: maximize advantage-weighted log prob minus KL
        policy_log_probs = torch.stack(log_probs)
        loss = -(advantages * policy_log_probs - kl_coeff * kl_penalty).mean()
        all_losses.append(loss)

    return torch.stack(all_losses).mean()


def combined_reward(context: str, candidate: str, downstream_model=None) -> float:
    """Dual reward: engagement proxy + rule compliance."""
    # Rule-based rewards
    tokens = candidate.split()
    length_ok = 1.0 if 80 <= len(tokens) <= 200 else -0.5
    sections = ["[Professional Identity]", "[Current Intent]", "[Key Skills]"]
    section_score = sum(1 for s in sections if s in candidate) / len(sections)
    repetition_penalty = -0.3 if has_repeated_phrases(candidate) else 0.0
    rule_reward = length_ok + section_score + repetition_penalty

    # Engagement-based reward (downstream NDCG proxy)
    if downstream_model:
        ndcg = downstream_model.evaluate_with_representation(context, candidate)
        engagement_reward = ndcg
    else:
        engagement_reward = 0.0

    return 0.7 * engagement_reward + 0.3 * rule_reward
```

## Best Practices

- **Do:** Start with a supervised warmstart before RL training. Generate template-based representations, fine-tune the LLM on them for 1-2 epochs, then switch to GRPO. This prevents early instability from random policy outputs.
- **Do:** Use aggressive length constraints in rule rewards. Without them, the policy tends toward verbose outputs that technically score well on engagement but are impractical for serving. Target 100-200 tokens.
- **Do:** Normalize engagement rewards per-user, not globally. Users with more interaction history naturally generate higher raw engagement scores. Group-relative normalization within each user's rollout set handles this.
- **Do:** Cache representations and refresh periodically rather than generating at query time. The entire point of pre-computed textual representations is to avoid LLM inference latency in the serving path.
- **Avoid:** Using the generated representation as the sole input to downstream models. It should augment, not replace, structured features like user ID embeddings or collaborative filtering signals.
- **Avoid:** Training on stale engagement data. Engagement signals older than 30-60 days may reflect outdated user intent. Weight recent interactions more heavily or use a sliding window.
- **Avoid:** Allowing the policy to copy input verbatim. Without a deduplication penalty, the model learns to paste raw profile text, which defeats the synthesis objective. Penalize n-gram overlap with input.

## Error Handling

- **Degenerate representations:** If the policy collapses to producing near-identical outputs for all users, increase the KL coefficient or add a diversity reward term. Check that the engagement reward has sufficient variance across candidates.
- **Reward hacking:** If representations become unreadable but score well on engagement, strengthen rule-based rewards (increase `w_rule`) and add a fluency reward using a separate language model's perplexity score.
- **Missing data sources:** When a user lacks one data source (e.g., no search history), the prompt template should include an explicit "[No data available]" marker rather than omitting the section silently. Train with such examples so the policy handles partial data gracefully.
- **Downstream metric regression:** If NDCG drops after deploying new representations, compare against the previous version's outputs. Common causes: training data distribution shift, reward function miscalibration, or KL divergence from the reference policy being too large (policy drifted too far).

## Limitations

- **Requires a downstream task with measurable engagement signals.** If you cannot define an engagement-based reward (e.g., no click/conversion data), the RL framework reduces to rule-based summarization, losing its primary advantage.
- **Cold-start users with minimal data** will produce low-quality representations regardless of policy quality. The system needs a fallback (e.g., demographic-based templates) for users with fewer than N interactions.
- **Computationally expensive training.** Generating K rollouts per user per training step is K times the cost of standard supervised fine-tuning. Budget 4-8x the GPU hours of SFT for equivalent data volume.
- **Representations encode a point-in-time snapshot.** Rapidly changing user intent (e.g., someone who just switched career goals) may not be captured until the next refresh cycle. Critical for time-sensitive applications.
- **Interpretability is not guaranteed.** While the output is text, RL-optimized text may contain subtle patterns that optimize the reward function in ways humans find unintuitive. Manual auditing of samples is essential.

## Reference

**Paper:** [High Fidelity Textual User Representation over Heterogeneous Sources via Reinforcement Learning](https://arxiv.org/abs/2602.07333v1) (Arora et al., 2026). Look for: the GRPO training procedure, dual reward function design (engagement + rules), and the ablation study showing the contribution of each reward component to downstream metrics.