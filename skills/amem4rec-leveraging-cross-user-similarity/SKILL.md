---
name: "amem4rec-leveraging-cross-user-similarity"
description: "Build agentic recommendation systems that learn collaborative filtering signals through cross-user memory evolution -- no CF model pre-training needed. Use when: 'build a recommender with memory', 'add collaborative filtering to LLM recommendations', 'cross-user pattern memory pool', 'agentic recommender system', 'memory-augmented ranking', 'evolving user behavior patterns for recommendations'."
---

# AMEM4Rec: Cross-User Memory Evolution for Agentic LLM Recommenders

This skill enables Claude to implement AMEM4Rec, a recommendation architecture where an LLM-based agent learns collaborative filtering signals end-to-end by abstracting user behaviors into a shared global memory pool, linking similar memories via dual validation (embedding similarity + semantic LLM check), and iteratively evolving those memories to reinforce cross-user patterns. At inference time, retrieved memories augment the LLM ranker with collaborative context, eliminating the need for a separate pre-trained CF model.

## When to Use

- When the user wants to build an LLM-powered recommender that captures "users who liked X also liked Y" patterns without training a traditional collaborative filtering model
- When designing an agentic system that maintains a shared memory pool of user behavior patterns across a user population
- When the user asks how to combine semantic understanding (LLM reasoning) with collaborative signals in a recommendation pipeline
- When implementing memory evolution or memory linking mechanisms for multi-user behavioral data
- When the user needs to augment LLM-based ranking with cross-user preference patterns
- When building a recommendation agent that must generalize across users without per-user fine-tuning

## Key Technique

**The core insight**: Instead of relying on a pre-trained collaborative filtering model (matrix factorization, graph neural networks, etc.), AMEM4Rec makes CF signals *emerge* from a shared memory pool. User interaction histories are processed through sliding windows and abstracted by an LLM into structured memory entries -- each containing a high-level behavior explanation and a concrete interaction pattern description. These memories are encoded via Sentence-BERT into an embedding space and stored in a global pool shared across all users.

**Memory linking and evolution**: When a new memory is created, it is compared against existing memories using cosine similarity. A dual-validator system decides what to do: a *similarity validator* uses a distribution-aware decision tree with soft thresholds (tau_low=0.55, tau_high=0.9) to classify the new memory into STORE-only, UPDATE-and-STORE, or UPDATE-only actions. A *semantic validator* (an LLM call) confirms whether candidate links are truly semantically related, filtering out false positives from embedding similarity alone. Linked memories are then evolved -- the LLM merges and reinforces shared patterns, causing related behavior archetypes to cluster tighter in embedding space over iterations.

**Inference**: At recommendation time, the user's recent history is encoded and matched against the evolved memory pool. The top-k retrieved memories, which now encode cross-user collaborative patterns, are provided as context to an LLM ranker alongside the user's personal history and candidate items. The ranker reorders candidates informed by both individual preferences and population-level behavioral patterns.

## Step-by-Step Workflow

### 1. Define the memory entry schema

Create a structured format for memories. Each memory `m_k` is a tuple `(p_k, e_k)` where `p_k` contains two fields and `e_k` is the embedding:

```python
@dataclass
class Memory:
    behavior_explanation: str  # 1-2 sentences: stable, high-level user tendency
    pattern_description: str   # 1-2 sentences: concrete interaction structure
    embedding: np.ndarray      # Sentence-BERT encoding of both fields concatenated
    linked_memory_ids: list[str]  # IDs of semantically linked memories
    evolution_count: int = 0   # how many times this memory has been updated
```

### 2. Extract behavior patterns from user histories using sliding windows

Partition each user's interaction sequence into overlapping windows of size `w` (default w=3). For each window, prompt the LLM to abstract the behavior:

```
Prompt: Given the following user interactions:
1. [Item title, category, action]
2. [Item title, category, action]
3. [Item title, category, action]

Generate:
- behavior_explanation: A 1-2 sentence description of the stable, high-level user tendency shown here. Do NOT mention specific item names.
- pattern_description: A 1-2 sentence description of the concrete interaction structure (e.g., "explores budget options then upgrades to premium").
```

Critical rule: behavior explanations must NEVER leak specific item names -- they must describe abstract behavioral tendencies only.

### 3. Encode memories and initialize the global memory pool

Encode each memory's concatenated text (`behavior_explanation + " " + pattern_description`) using Sentence-BERT (or any sentence embedding model). Store all memories in a shared pool with an efficient nearest-neighbor index (FAISS or similar):

```python
from sentence_transformers import SentenceTransformer
encoder = SentenceTransformer('all-MiniLM-L6-v2')

memory_pool = {}  # id -> Memory
index = faiss.IndexFlatIP(384)  # cosine similarity via inner product on normalized vectors
```

### 4. Compute similarity scores against existing memories

For each new memory `m_n`, retrieve top-k (k=5) nearest neighbors from the pool by cosine similarity. Compute the score distribution metrics:

```python
scores = cosine_similarity(m_n.embedding, pool_embeddings)  # top-k
s_max = max(scores)
p_high = sum(1 for s in scores if s >= tau_high) / k    # tau_high = 0.9
p_medium = sum(1 for s in scores if tau_low <= s < tau_high) / k  # tau_low = 0.55
p_low = sum(1 for s in scores if s < tau_low) / k
```

### 5. Apply the similarity validator decision tree

Use the soft-threshold mechanism to decide the action:

```python
if s_max < tau_low:
    action = "STORE_ONLY"           # novel pattern, no similar memories
elif s_max < tau_high:
    action = "UPDATE_AND_STORE"     # partially similar, keep both
elif p_high >= 0.6:
    action = "UPDATE_ONLY"          # highly redundant, merge into existing
else:
    action = "UPDATE_AND_STORE"     # high max but sparse distribution
```

### 6. Run the semantic validator (LLM call)

For candidate links identified by embedding similarity, ask the LLM to confirm semantic relatedness:

```
Prompt: Given these two user behavior patterns:
Memory A: [behavior_explanation + pattern_description]
Memory B: [behavior_explanation + pattern_description]

Are these describing fundamentally the same or closely related user behavior tendency?
Answer YES or NO with a one-sentence justification.
```

Only link memories that pass both the similarity threshold AND the semantic validation.

### 7. Evolve linked memories

For UPDATE actions, prompt the LLM to merge the new memory with each validated linked memory:

```
Prompt: You have two related behavior patterns from different users:
Existing: [memory text]
New: [memory text]

Produce an evolved version that reinforces the shared pattern while preserving nuances.
Output the same format: behavior_explanation + pattern_description.
```

Re-encode the evolved memory and update it in the pool. Increment `evolution_count`.

### 8. Iterate across the full user population

Process all users' windows sequentially (or in batches). The memory pool grows and evolves organically -- early memories get reinforced as more users with similar patterns are processed. Memories with high `evolution_count` represent strong cross-user behavioral archetypes.

### 9. Build the memory-augmented ranker for inference

At inference time for a target user:

```python
# Encode user's recent history
user_context = encode_user_history(user_recent_interactions)
# Retrieve top-k_mem memories (k_mem=5)
relevant_memories = memory_pool.search(user_context, k=5)

# Construct ranking prompt
prompt = f"""Given this user's recent interactions:
{format_history(user_recent_interactions)}

Cross-user behavioral patterns relevant to this user:
{format_memories(relevant_memories)}

Rank the following candidate items from most to least relevant:
{format_candidates(candidate_items)}

Return a ranked list with brief justifications."""
```

### 10. Evaluate and tune thresholds

Measure NDCG@K (K in {1, 5, 10}) on a held-out test set. Key hyperparameters to tune:
- Window size `w` (default 3)
- Number of neighbors `k` (default 5)
- Thresholds `tau_low` (default 0.55) and `tau_high` (default 0.9)
- Number of retrieved memories at inference `k_mem`

## Concrete Examples

**Example 1: E-commerce product recommender**

User: "I have a dataset of Amazon purchase histories. Build me a recommender that uses LLM reasoning but also captures collaborative patterns across users."

Approach:
1. Parse purchase histories into per-user interaction sequences with item titles and categories
2. Slide a window of size 3 across each user's history; for each window, call the LLM to abstract a behavior pattern (e.g., "Explores budget fitness equipment before committing to a mid-range option" / "Sequences: resistance bands -> yoga mat -> adjustable dumbbells")
3. Encode all memories with Sentence-BERT, build the FAISS index
4. Process memories through the dual-validator pipeline -- memories like "budget fitness exploration" from different users get linked and evolved into a reinforced archetype
5. At inference, retrieve the 5 most relevant evolved memories for a target user and feed them alongside their history to the LLM ranker

Output:
```json
{
  "user_id": "U123",
  "retrieved_memories": [
    {"pattern": "Budget-conscious fitness equipment exploration with gradual upgrade trajectory", "evolution_count": 47},
    {"pattern": "Home workout setup building across complementary equipment categories", "evolution_count": 31}
  ],
  "ranked_items": [
    {"rank": 1, "item": "Adjustable Kettlebell 25lb", "reason": "Fits budget upgrade trajectory + home workout pattern"},
    {"rank": 2, "item": "Exercise Mat Premium", "reason": "Complements existing equipment pattern"},
    {"rank": 3, "item": "Protein Shaker Bottle", "reason": "Adjacent category to fitness equipment interest"}
  ]
}
```

**Example 2: News article recommender**

User: "I'm building a news recommendation system. Users have short reading histories and I need to capture what types of articles similar readers consume."

Approach:
1. Use the MIND dataset format: user ID -> sequence of clicked article titles + categories
2. Set window size w=3 (news sessions are short); abstract patterns like "Follows breaking political stories then seeks analysis pieces" or "Scans tech headlines, deep-dives on AI/ML topics"
3. Build the global memory pool -- cross-user linking will surface archetypes like "tech-curious reader who progresses from headlines to technical analysis"
4. At inference, a new user with 2-3 clicks gets matched to evolved memories that represent the reading patterns of thousands of similar users
5. The LLM ranker uses these memories to recommend articles even when the user's own history is sparse (cold-start mitigation)

Output:
```
Retrieved memories for user with history ["GPT-5 Announced", "OpenAI Valuation Soars"]:
- "AI industry follower: tracks product launches then reads business/financial implications" (evolved 82 times)
- "Tech investor mindset: reads announcements then seeks market analysis" (evolved 56 times)

Top recommendations:
1. "What GPT-5 Means for Enterprise AI Adoption" (analysis piece matching both memory patterns)
2. "AI Chip Stocks Rally After OpenAI News" (financial angle from investor pattern)
```

**Example 3: Adding memory evolution to an existing agent framework**

User: "I already have an LLM agent that makes recommendations from user history. How do I add the AMEM4Rec memory pool to it?"

Approach:
1. Keep existing agent's user profiling and candidate generation intact
2. Add a `MemoryPool` module with Sentence-BERT encoder and FAISS index
3. Run a one-time batch job: process all historical user interactions through the sliding window + LLM abstraction + dual-validator pipeline to populate the memory pool
4. Modify the ranking step: before the LLM ranks candidates, retrieve top-5 memories from the pool and inject them into the ranking prompt as "Cross-user behavioral insights"
5. Set up periodic memory pool refresh (e.g., nightly) to process new interactions and evolve memories

Integration code skeleton:
```python
class MemoryAugmentedRecommender:
    def __init__(self, base_agent, memory_pool):
        self.agent = base_agent
        self.pool = memory_pool

    def recommend(self, user_id, candidates):
        history = self.agent.get_user_history(user_id)
        history_embedding = self.pool.encode(history)
        memories = self.pool.retrieve(history_embedding, k=5)

        ranking_context = {
            "user_history": history,
            "cross_user_patterns": [m.to_text() for m in memories],
            "candidates": candidates
        }
        return self.agent.rank(ranking_context)
```

## Best Practices

- **Do**: Keep behavior explanations abstract -- never include specific item names in memory entries. This ensures memories generalize across users and items rather than memorizing specific products.
- **Do**: Use both validators (embedding similarity AND semantic LLM check) before linking. Embedding similarity alone produces false positives; the LLM semantic check filters noise at the cost of one extra call per candidate link.
- **Do**: Track `evolution_count` on memories. Highly-evolved memories (reinforced by many users) represent strong population-level signals and should be weighted higher during retrieval.
- **Do**: Process users in randomized order during memory pool construction to avoid ordering bias in how memories evolve.
- **Avoid**: Setting tau_high too low (below 0.85) -- this causes aggressive merging that collapses distinct behavior patterns into overly generic memories.
- **Avoid**: Skipping the sliding window and feeding entire user histories to the LLM at once. Long histories exceed context limits and produce vague, unhelpful abstractions. Windows of size 3 yield focused, specific patterns.

## Error Handling

- **LLM refuses to abstract a pattern**: Some interaction windows may be too sparse or incoherent. Fall back to storing a raw pattern description without the LLM abstraction, or skip the window if fewer than 2 interactions have meaningful metadata.
- **Memory pool grows too large**: Implement a pruning strategy -- remove memories with `evolution_count == 0` after all users are processed (orphan memories that never linked to any other). For active systems, set a TTL or cap pool size by evicting lowest-evolution-count entries.
- **Embedding model and LLM disagree**: If the semantic validator consistently rejects high-similarity candidates (>0.85), the embedding model may not suit your domain. Switch to a domain-specific sentence encoder or fine-tune on your item descriptions.
- **Cold-start users at inference**: Users with fewer than `w` interactions cannot fill a sliding window. Handle by encoding their partial history directly and retrieving memories -- the cross-user memory pool specifically helps here since it provides population-level context.
- **Duplicate/near-duplicate memories**: If many memories converge to nearly identical text after evolution, deduplicate by merging memories whose post-evolution cosine similarity exceeds 0.95.

## Limitations

- **LLM cost at scale**: Memory creation requires one LLM call per sliding window per user, plus validator calls during linking. For millions of users, this becomes expensive. Batch processing with a cost-efficient model (e.g., Gemini Flash, GPT-4o-mini) is necessary.
- **Memory staleness**: The evolved memory pool reflects historical patterns. In fast-changing domains (trending news, seasonal products), memories need periodic refresh or decay mechanisms not covered in the base paper.
- **No explicit temporal modeling**: The sliding window captures local sequence structure but the memory pool itself has no notion of time. A pattern from 2 years ago and yesterday carry equal weight unless you add recency weighting.
- **Embedding bottleneck**: Memory retrieval quality depends heavily on the sentence embedding model. Domain-specific corpora (medical, legal) may need specialized encoders.
- **Sequential processing assumption**: The paper processes users sequentially during pool construction. Memory evolution outcomes depend on processing order, though randomization mitigates this.

## Reference

**Paper**: [AMEM4Rec: Leveraging Cross-User Similarity for Memory Evolution in Agentic LLM Recommenders](https://arxiv.org/abs/2602.08837v1) (Nguyen, Kieu, Le, 2026). Key sections: Section 3 for the full architecture, Algorithm 1 for the training/inference pseudocode, and Appendix A for the exact prompts used in memory abstraction, linking validation, evolution, and ranking.