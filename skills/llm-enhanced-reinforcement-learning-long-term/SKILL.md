---
name: "llm-enhanced-reinforcement-learning-long-term"
description: "Build hierarchical recommendation systems that combine LLM semantic planning with RL fine-grained optimization for long-term user satisfaction. Use when asked to: 'build a recommendation system that avoids filter bubbles', 'implement hierarchical RL for recommendations', 'use LLM as a planner for RL agent', 'optimize long-term user engagement with diverse content', 'combine LLM reasoning with reinforcement learning', 'break out of recommendation homogeneity'."
---

# LLM-Enhanced Reinforcement Learning for Long-Term User Satisfaction

This skill enables Claude to implement the LERL (LLM-Enhanced Reinforcement Learning) framework: a hierarchical recommendation architecture where an LLM serves as a high-level semantic planner selecting diverse content categories, and an RL agent operates as a low-level policy recommending specific items within those categories. The key innovation is decomposing the recommendation problem into two levels -- strategic diversity planning (LLM) and tactical personalization (RL) -- which narrows the action space, prevents filter bubbles, and optimizes cumulative long-term user satisfaction rather than greedy short-term clicks.

## When to Use

- When building an interactive recommendation system that must balance diversity and personalization over many interaction rounds
- When the user wants to prevent filter bubble effects or content homogeneity in a recommender
- When implementing a hierarchical RL system where an LLM guides the high-level strategy and RL handles fine-grained decisions
- When the user needs to optimize for long-term user retention/satisfaction rather than immediate click-through rate
- When designing a recommender with a large item catalog where flat RL struggles with action space explosion
- When combining pretrained LLM reasoning capabilities with online RL adaptation in any sequential decision-making system
- When the user asks about reflection-augmented LLM planning where the model learns from past trajectory outcomes

## Key Technique

**Hierarchical Decomposition**: LERL splits the recommendation decision into two levels. The **High-level Semantic Planner (HSP)** is an LLM (e.g., Llama-3-8B) that receives the user's category-level interaction history and a pool of textual reflections from past sessions. It selects a subset of content categories for the current timestep, ensuring semantic diversity. The **Low-level Policy Learner (LPL)** is a PPO-based RL agent that, constrained to items within the LLM-selected categories, recommends a personalized top-k list using a Gaussian policy over item embeddings masked by the category selection.

**Reflection-Augmented Planning**: After each user session, the LLM generates a textual reflection summarizing what worked and what didn't for that user trajectory. These reflections are stored in a pool and sampled (weighted by trajectory reward) as in-context examples for future planning decisions. This creates a feedback loop where the LLM's planning improves over time without gradient updates -- it learns through curated in-context experience.

**Action Space Narrowing with Category Masking**: The RL agent samples a virtual item embedding from a learned Gaussian distribution, computes similarity scores against all item embeddings, then applies a binary category mask derived from the LLM's selection. This masks out items outside the chosen categories before selecting the top-k. The masking dramatically reduces the effective action space while the LLM ensures the selected categories are semantically diverse, preventing the RL agent from collapsing into a narrow content niche.

## Step-by-Step Workflow

1. **Define the item taxonomy**: Organize items into semantic categories (e.g., genres, topics, product types). Each item maps to one or more categories via a binary item-category matrix `W` where `W[i][c] = 1` if item `i` belongs to category `c`. This structure is required for the category masking mechanism.

2. **Build the user state representation**: Implement a Transformer encoder that processes the user's interaction history `H_t = {(a_i, r_i)}` (item-reward pairs) into a fixed-dimensional user preference embedding `e_p`. This captures temporal dependencies in user behavior and serves as the RL agent's state input.

3. **Implement the LLM high-level planner**: Create a prompt template that includes: (a) the list of candidate categories, (b) the user's category-level interaction history (which categories were shown and average reward per category), and (c) sampled reflections from the reflection pool. The LLM outputs a subset of categories `c_t` to activate for this timestep.

4. **Implement the RL low-level actor**: Build a Gaussian policy network (MLP) that takes the user state and outputs mean `mu` and variance `sigma` parameters. Sample a virtual item embedding `p_t ~ N(mu, sigma^2)`, compute cosine similarity with all item embeddings, apply the category mask `a_mask = W * c_t`, compute final scores as `a_score = a_sim * a_mask`, and select top-k items.

5. **Implement the RL critic**: Build a value network `V_phi(s_t)` that estimates the expected cumulative reward from state `s_t`. Train it to minimize TD error: `L_v = E[(V_phi(s_t) - (r_t + gamma * V_phi'(s_{t+1})))^2]`.

6. **Train the RL policy with PPO**: Use the clipped surrogate objective with advantage estimation. The ratio `pi_theta(a|s) / pi_old(a|s)` is clipped within `[1-epsilon, 1+epsilon]` to ensure stable updates. Train the actor to maximize `L_a = -E[min(ratio * A_hat, clip(ratio) * A_hat)]`.

7. **Implement the diversity-aware quit mechanism**: In the user simulation or reward design, penalize consecutive same-category recommendations. When the system recommends the same category `n` times in a row, decrement the remaining session length, simulating user fatigue with repetitive content.

8. **Build the reflection generation pipeline**: After each session ends, pass the full user trajectory (categories shown, rewards received, cumulative satisfaction) to the LLM with a reflective critic prompt. The LLM generates a textual summary of strategy insights. Store this reflection with the session's total reward in the reflection pool.

9. **Implement weighted reflection sampling**: When constructing the planner prompt, sample reflections with probability `P(u) = exp(alpha * S_u) / sum(exp(alpha * S_v))` where `S_u` is the trajectory reward. This prioritizes insights from successful sessions as in-context examples.

10. **Run the training loop**: Alternate between (a) the LLM planner selecting categories, (b) the RL agent recommending items within those categories, (c) collecting user feedback, (d) updating the RL policy via PPO, and (e) generating reflections at session boundaries. Evaluate using cumulative reward, category diversity (entropy), and long-term retention metrics.

## Concrete Examples

**Example 1: Building a LERL-style video recommendation system**

User: "I want to build a video recommendation system that doesn't keep showing users the same type of content. It should optimize for long-term watch time, not just immediate clicks."

Approach:
1. Define video categories from metadata (e.g., Comedy, Drama, Science, Sports, Music, News -- say 20 categories total)
2. Build the item-category mapping matrix from video tags
3. Set up the LLM planner with this prompt template:

```python
PLANNER_PROMPT = """You are a recommendation strategist optimizing long-term user satisfaction.

Available content categories: {categories}

User's recent category interaction history (category: avg_satisfaction):
{category_history}

Insights from similar successful sessions:
{sampled_reflections}

Select 3-5 categories for the next recommendation round that balance:
1. Exploiting categories the user has enjoyed
2. Exploring diverse categories to prevent content fatigue
3. Long-term engagement over short-term clicks

Output format: [Category1, Category2, Category3, ...]
"""
```

4. Implement the RL actor with category masking:

```python
class LERLActor(nn.Module):
    def __init__(self, state_dim, embed_dim, item_embeddings, item_category_matrix):
        super().__init__()
        self.mu_net = nn.Sequential(
            nn.Linear(state_dim, 256), nn.ReLU(),
            nn.Linear(256, embed_dim)
        )
        self.log_sigma_net = nn.Sequential(
            nn.Linear(state_dim, 256), nn.ReLU(),
            nn.Linear(256, embed_dim)
        )
        self.item_embeddings = item_embeddings  # (num_items, embed_dim)
        self.W = item_category_matrix            # (num_items, num_categories)

    def forward(self, state, selected_categories, k=6):
        mu = self.mu_net(state)
        sigma = torch.exp(self.log_sigma_net(state))
        dist = torch.distributions.Normal(mu, sigma)
        virtual_embed = dist.rsample()

        # Similarity scores
        sim_scores = torch.matmul(
            F.normalize(virtual_embed, dim=-1),
            F.normalize(self.item_embeddings, dim=-1).T
        )

        # Category mask: only items in LLM-selected categories
        cat_mask = self.W[:, selected_categories].any(dim=1).float()
        masked_scores = sim_scores * cat_mask

        # Top-k selection
        top_k_indices = masked_scores.topk(k).indices
        return top_k_indices, dist
```

5. Wire the hierarchical loop: LLM selects categories every N steps, RL recommends items each step within those categories

Output: A system where the LLM ensures the user sees Comedy, Science, *and* Music videos (not just Comedy repeatedly), while the RL agent picks the *best* Comedy/Science/Music videos for that specific user.

---

**Example 2: Adding reflection-augmented planning to an existing RL recommender**

User: "I already have a DQN-based recommender. How do I add the LERL reflection mechanism to improve its long-term performance?"

Approach:
1. Keep the existing DQN as the low-level policy
2. Add the category taxonomy layer on top of items
3. Implement the reflection pool and generation:

```python
class ReflectionPool:
    def __init__(self, max_size=1000, alpha=1.0):
        self.reflections = []  # (text, reward_score)
        self.max_size = max_size
        self.alpha = alpha

    def add(self, reflection_text, session_reward):
        self.reflections.append((reflection_text, session_reward))
        if len(self.reflections) > self.max_size:
            self.reflections.sort(key=lambda x: x[1])
            self.reflections.pop(0)  # Remove lowest-reward reflection

    def sample(self, n=3):
        if not self.reflections:
            return []
        rewards = torch.tensor([r for _, r in self.reflections])
        probs = torch.softmax(self.alpha * rewards, dim=0)
        indices = torch.multinomial(probs, min(n, len(self.reflections)),
                                    replacement=False)
        return [self.reflections[i][0] for i in indices]

    def generate_reflection(self, llm_client, trajectory, categories):
        prompt = f"""Analyze this recommendation session and provide strategic insights.

Categories available: {categories}
Session trajectory (category -> user_satisfaction):
{trajectory}

What patterns led to high/low satisfaction? What category sequencing
strategy would improve long-term engagement? Be specific and concise."""

        reflection = llm_client.generate(prompt)
        total_reward = sum(r for _, r in trajectory)
        self.add(reflection, total_reward)
        return reflection
```

4. Modify the training loop to generate reflections at session boundaries and inject sampled reflections into LLM planner prompts

Output: The existing DQN now operates within LLM-selected category constraints, with the reflection pool continuously improving the LLM's category selection strategy across sessions.

---

**Example 3: Implementing the diversity-aware quit mechanism**

User: "How do I simulate user fatigue from repetitive recommendations in my environment?"

Approach:
1. Track consecutive same-category recommendations:

```python
class DiversityAwareEnv:
    def __init__(self, base_env, max_steps=20, fatigue_threshold=3):
        self.base_env = base_env
        self.max_steps = max_steps
        self.fatigue_threshold = fatigue_threshold
        self.consecutive_same = 0
        self.last_category = None
        self.remaining_steps = max_steps

    def step(self, action):
        category = self.get_category(action)

        if category == self.last_category:
            self.consecutive_same += 1
        else:
            self.consecutive_same = 0
        self.last_category = category

        # Penalize repetition by shortening session
        if self.consecutive_same >= self.fatigue_threshold:
            self.remaining_steps -= 1
            self.consecutive_same = 0

        obs, reward, done, info = self.base_env.step(action)
        self.remaining_steps -= 1
        done = done or self.remaining_steps <= 0
        info['category_diversity'] = len(set(self.category_history))
        return obs, reward, done, info
```

2. This naturally trains the RL agent to avoid category repetition since repetitive behavior shortens episodes and reduces cumulative reward.

## Best Practices

- **Do**: Use a structured prompt template for the LLM planner with explicit category lists, quantified history, and sampled reflections. Unstructured prompts lead to inconsistent category selection.
- **Do**: Run multiple LLM server instances (e.g., 3 Ollama processes on different ports) for parallel inference during training. LLM planning latency is the bottleneck.
- **Do**: Normalize reward signals before computing reflection sampling weights. Raw rewards with high variance cause the sampling distribution to collapse to a few high-reward reflections.
- **Do**: Tune the category mask granularity -- too few categories means the LLM can't provide useful diversity signals; too many means the RL action space within each category is too small for effective learning.
- **Avoid**: Calling the LLM at every single timestep. The planner should select categories at a lower frequency (e.g., every N steps or at the start of each session) while the RL agent acts at every step. This reduces latency and cost.
- **Avoid**: Using the LLM for item-level decisions. The LLM's strength is semantic reasoning over categories, not fine-grained ranking of individual items. Keep item selection in the RL agent where gradient-based optimization excels.

## Error Handling

- **LLM returns invalid categories**: Parse the LLM output and validate against the known category set. If the response is malformed or contains unknown categories, fall back to the previous round's categories or use a uniform random category subset.
- **Category mask eliminates all items**: If the selected categories contain no available items (e.g., cold-start), expand the mask to include the top-2 closest categories by embedding similarity, or fall back to the full item set for that step.
- **Reflection pool is empty at start**: For the first few sessions, run the planner without reflections (zero-shot category selection). Begin sampling only after at least 10-20 reflections have accumulated.
- **RL policy collapses within masked space**: If the Gaussian policy variance collapses to near-zero, add an entropy bonus to the PPO objective to maintain exploration within the LLM-selected categories.
- **LLM latency causes training slowdown**: Batch planner queries across users, cache category selections for users with similar interaction histories, or use a smaller distilled model for training and the full LLM only for evaluation.

## Limitations

- Requires a well-defined item category taxonomy. If items don't have clear categorical structure (e.g., highly abstract or novel content), the hierarchical decomposition loses its advantage.
- LLM inference cost scales with training iterations. For large-scale production training with millions of users, the LLM planner becomes a significant compute bottleneck even with batching.
- The reflection mechanism assumes session-level feedback patterns are transferable across users. In highly heterogeneous user populations, reflections from one user segment may mislead planning for another.
- The Gaussian policy over item embeddings works best when item embeddings are well-trained and semantically meaningful. Poor embeddings degrade the similarity-based selection mechanism.
- Evaluated primarily on Chinese short-video datasets (KuaiRand, KuaiRec). Effectiveness on other domains (e.g., e-commerce, news, music) with different user behavior dynamics is not yet validated.

## Reference

**Paper**: [LLM-Enhanced Reinforcement Learning for Long-Term User Satisfaction in Interactive Recommendation](https://arxiv.org/abs/2601.19585v2) by Xia, Peng, and Wang (2026). Look for: Section 3 (LERL Framework) for the hierarchical architecture, Section 3.2 for the reflection-augmented planner prompt design, Section 3.3 for the category-masked Gaussian policy, and Section 4 for experimental setup on KuaiSim.

**Code**: [github.com/1163710212/LERL](https://github.com/1163710212/LERL) -- Reference implementation using Llama-3-8B via Ollama and PPO with actor-critic on KuaiRand/KuaiRec datasets.