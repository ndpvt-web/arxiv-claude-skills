---
name: "clustering-driven-memory-compression-on-device"
description: |
  Compress user-specific memories for LLM personalization by clustering semantically similar memories
  and merging within clusters, reducing token count while preserving generation quality. Based on
  Bohdal et al. (ICASSP 2026). Use this skill when the user mentions:
  - "compress memories for context window"
  - "reduce memory tokens for on-device LLM"
  - "cluster and merge user memories"
  - "personalization with limited context budget"
  - "memory-efficient prompt construction"
  - "on-device LLM memory management"
---

# Clustering-Driven Memory Compression for On-Device LLMs

This skill enables Claude to design and implement systems that compress user-specific memories (past interactions, preferences, documents) for personalized LLM generation under tight context budgets. Instead of naively concatenating all memories (which exhausts context) or averaging them into a single representation (which destroys semantic distinctions), this technique clusters memories by similarity and merges within each cluster. The result: 2-8x token reduction with equal or better generation quality compared to full concatenation.

## When to Use

- When building a personalization layer for an on-device or context-limited LLM that stores user memories as text or embeddings
- When a user's memory store has grown too large to fit in the prompt context window
- When implementing a retrieval-augmented generation (RAG) system that needs to inject multiple retrieved documents but faces token budget constraints
- When the user asks how to balance compression ratio against personalization quality for mobile/edge LLM deployments
- When designing a memory management module that must handle heterogeneous user data (e.g., news preferences mixed with writing style samples)
- When the user needs to reduce prompt length for cost optimization while retaining multi-document context

## Key Technique

**The core problem:** On-device LLMs (1-3B parameters) have small context windows (2K-4K tokens). Personalization requires injecting user-specific memories (past articles, tweets, preferences) into the prompt. Concatenating N memories of D_m tokens each costs N x D_m tokens — quickly exceeding the budget. Naive averaging (collapsing all memories into a single D_m-token representation) loses critical distinctions between semantically different memories, degrading output quality.

**The clustering solution:** Before merging, group the N memories into K clusters using K-means on their embedding representations. Memories within each cluster share semantic similarity, so averaging within a cluster preserves coherent information. The final prompt receives K merged representations instead of N individual ones, reducing tokens from N x D_m to K x D_m. For example, with N=8 memories of 128 tokens each and K=4 clusters, total memory tokens drop from 1,024 to 512 — a 2x reduction that actually *outperforms* full concatenation on benchmarks (ROUGE-L: 44.32 vs 43.79 on tweet paraphrasing, 40.07 vs 36.42 on movie tagging).

**Why clustering beats naive averaging:** Heterogeneous memories create semantic conflicts when averaged together. A user's sports article preferences and cooking recipe history produce a meaningless centroid when merged blindly. Clustering respects these natural boundaries — sports memories merge with sports memories, cooking with cooking — preserving the distinct signals that drive personalization quality.

## Step-by-Step Workflow

1. **Retrieve candidate memories** from the user's memory store. Use BM25 or semantic search to select the top-N most relevant memories for the current query. The paper uses N=8 as a practical default; adjust based on your context budget.

2. **Encode memories into embeddings.** Pass each memory text through an embedding model (e.g., sentence-transformers, or the LLM's own hidden states) to obtain a vector representation per memory. Each memory yields a vector of dimension D_e (e.g., 768 or 2048).

3. **Choose the number of clusters K.** Set K = N/2 as a starting point (e.g., K=4 for N=8). More clusters preserve more detail but use more tokens. Fewer clusters compress more aggressively. The constraint is: total tokens = K x D_m must fit your context budget.

4. **Run K-means clustering on the memory embeddings.** Use standard K-means (scikit-learn's `KMeans` or a lightweight implementation for on-device). Assign each memory to its nearest cluster centroid based on cosine or Euclidean distance.

5. **Merge memories within each cluster.** For each cluster, average the token-level representations (embeddings) of all memories assigned to that cluster. This produces one merged memory representation per cluster. If working with text rather than embeddings, concatenate texts within each cluster and summarize using the LLM or a dedicated summarizer.

6. **Construct the compressed prompt.** Append the K merged memory representations to the input query/instruction. The format is: `[merged_memory_1] [merged_memory_2] ... [merged_memory_K] [instruction/query]`.

7. **Generate the personalized response.** Pass the compressed prompt to the LLM. The model receives the same semantic coverage as full concatenation but at a fraction of the token cost.

8. **Evaluate and tune K.** Measure output quality (ROUGE-L, user satisfaction, task accuracy) across different K values. Increase K if quality drops unacceptably; decrease K if you need more context headroom for the instruction itself.

9. **Handle edge cases.** If N <= K, skip clustering entirely and concatenate all memories directly. If a cluster contains only one memory, pass it through without averaging.

10. **Monitor and re-cluster periodically.** As the memory store grows or shifts in topic distribution, re-run clustering to keep cluster assignments current. Stale clusters degrade both compression quality and personalization.

## Concrete Examples

**Example 1: Personalized news headline generation**

```
User: I'm building a news app with a 1.5B parameter on-device LLM. Each user has
~20 past article interactions stored as text. I need to generate personalized
headlines but my context window is only 2048 tokens. How do I fit the user's
history into the prompt?

Approach:
1. Retrieve the top 8 most relevant past articles using BM25 against the
   current article topic.
2. Encode each article snippet (truncated to ~128 tokens) using the LLM's
   encoder or a lightweight embedding model.
3. Cluster the 8 embeddings into K=4 groups using K-means.
4. Average embeddings within each cluster to produce 4 merged memories
   (4 x 128 = 512 tokens).
5. Construct prompt: [4 merged memories] + [article body] + [instruction:
   "Write a headline in the user's preferred style"].
6. Total memory cost: 512 tokens, leaving 1536 tokens for article + instruction.

Output (Python implementation sketch):
```

```python
import numpy as np
from sklearn.cluster import KMeans
from sentence_transformers import SentenceTransformer

def compress_memories(memories: list[str], k: int = 4) -> list[str]:
    """Cluster and merge user memories for context-efficient personalization."""
    if len(memories) <= k:
        return memories  # No compression needed

    # Step 1: Encode memories
    encoder = SentenceTransformer('all-MiniLM-L6-v2')
    embeddings = encoder.encode(memories)

    # Step 2: Cluster by semantic similarity
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(embeddings)

    # Step 3: Merge within clusters (text-level merging)
    merged = []
    for cluster_id in range(k):
        cluster_texts = [m for m, l in zip(memories, labels) if l == cluster_id]
        # Concatenate cluster members into a single summary block
        merged.append(" | ".join(cluster_texts))

    return merged

# Usage
user_memories = [
    "Loved the piece on renewable energy policy shifts",
    "Shared the solar panel cost analysis article",
    "Read 3 articles on battery storage breakthroughs",
    "Clicked on the local election coverage",
    "Spent 5 min on the city council budget article",
    "Bookmarked the EV adoption statistics piece",
    "Skipped all celebrity gossip articles",
    "Engaged with the climate summit recap",
]

compressed = compress_memories(user_memories, k=4)
# Result: 4 merged memory strings instead of 8, grouped by topic similarity
# Cluster 0 (energy): "Loved the piece on renewable energy... | Shared the solar panel..."
# Cluster 1 (local politics): "Clicked on the local election... | Spent 5 min on city council..."
# etc.
```

**Example 2: Tweet style paraphrasing with token budget**

```
User: I want to paraphrase tweets in a user's personal style. I have their last
50 tweets stored. My model has a 4096-token context. The input tweet + instruction
takes ~200 tokens. How do I use their tweet history effectively?

Approach:
1. Use BM25 to retrieve the 8 most stylistically relevant past tweets
   for the input tweet.
2. Encode all 8 tweets into embeddings.
3. Cluster into K=4 groups (token budget: 3896 available - allocate
   ~512 tokens to memories, leaving 3384 for other context).
4. For each cluster, concatenate member tweets and prepend a label:
   "Style examples (humor): [tweet1] [tweet2]"
   "Style examples (professional): [tweet3] [tweet4]"
5. Append to prompt before the instruction.

Output (prompt structure):
```

```text
[Memory Cluster 1 - Casual/humor style]:
"just mass-deleted 47 emails without reading them. productivity king."
"why does every meeting need to be an email and every email need to be a meeting"

[Memory Cluster 2 - Tech commentary style]:
"the new API rate limits are going to break half the indie apps out there"
"hot take: typescript saved more projects than any framework ever did"

[Memory Cluster 3 - Motivational style]:
"shipped the feature at 2am. sleep is temporary, deploy is forever."

[Memory Cluster 4 - Observational style]:
"noticed my code reviews get harsher after lunch. sorry afternoon PRs"
"every senior engineer's origin story starts with a production outage"

Instruction: Rewrite the following tweet in this user's style:
Original: "Artificial intelligence is transforming the healthcare industry significantly."
```

**Example 3: Embedding-level compression for on-device deployment**

```
User: I'm working at the embedding level, not text level. My on-device model
processes memory tokens as learned embeddings (128 tokens x 2048 dims per memory).
How do I implement clustering-driven compression in PyTorch?

Approach:
1. Stack all N memory tensors into shape [N, D_m, D_e].
2. Compute per-memory mean embeddings [N, D_e] for clustering.
3. Run K-means on the [N, D_e] matrix to get cluster assignments.
4. Average the full [D_m, D_e] tensors within each cluster.
5. Stack K merged tensors into [K, D_m, D_e] as compressed memory input.

Output (PyTorch implementation):
```

```python
import torch
from sklearn.cluster import KMeans

def cluster_compress_embeddings(
    memory_tokens: torch.Tensor,  # [N, D_m, D_e] e.g. [8, 128, 2048]
    k: int = 4,
) -> torch.Tensor:
    """Compress N memory embeddings into K via clustering and averaging."""
    N, D_m, D_e = memory_tokens.shape

    if N <= k:
        return memory_tokens

    # Compute per-memory summary vector for clustering
    memory_summaries = memory_tokens.mean(dim=1).numpy()  # [N, D_e]

    # Cluster memories
    kmeans = KMeans(n_clusters=k, random_state=42, n_init=10)
    labels = kmeans.fit_predict(memory_summaries)

    # Merge within clusters by averaging token-level representations
    compressed = torch.zeros(k, D_m, D_e)
    for cluster_id in range(k):
        mask = torch.tensor(labels == cluster_id)
        cluster_members = memory_tokens[mask]  # [n_i, D_m, D_e]
        compressed[cluster_id] = cluster_members.mean(dim=0)  # [D_m, D_e]

    return compressed  # [K, D_m, D_e]

# Example: 8 memories x 128 tokens x 2048 dims -> 4 x 128 x 2048
memories = torch.randn(8, 128, 2048)
compressed = cluster_compress_embeddings(memories, k=4)
print(f"Compressed: {memories.shape} -> {compressed.shape}")
# Compressed: torch.Size([8, 128, 2048]) -> torch.Size([4, 128, 2048])
# Token reduction: 1024 -> 512 (2x compression)
```

## Best Practices

- **Do:** Choose K based on your remaining context budget. Calculate: `K = (context_window - instruction_tokens - query_tokens) / tokens_per_memory`. This gives you the maximum K that fits.
- **Do:** Use more clusters (higher K) when memories are semantically diverse. If a user interacts with 5 distinct topics, K >= 5 prevents cross-topic averaging artifacts.
- **Do:** Apply BM25 or semantic retrieval *before* clustering. Pre-filter to the N most relevant memories, then cluster those. Clustering all memories wastes compute on irrelevant ones.
- **Do:** Re-cluster when the memory store changes significantly (e.g., >20% new entries since last clustering).
- **Avoid:** Setting K=1, which degenerates to naive averaging and loses the clustering benefit entirely.
- **Avoid:** Clustering on raw text with string similarity. Always use embedding-based representations — text-level similarity (edit distance, n-gram overlap) misses semantic relationships.
- **Avoid:** Using very large K relative to N (e.g., K=7 for N=8). The compression ratio becomes negligible and you lose the efficiency benefit. Target K <= N/2.

## Error Handling

- **Cluster becomes empty:** K-means can produce empty clusters if K is too large or data is highly concentrated. Detect empty clusters and reduce K by 1, then re-run. Alternatively, use K-means++ initialization (default in scikit-learn) which mitigates this.
- **All memories are near-identical:** If embedding variance is very low, clustering produces arbitrary assignments. Detect this by checking if the within-cluster sum of squares is close to total variance. Fall back to naive averaging when memories are homogeneous — it works fine when there are no semantic conflicts.
- **Memory count less than K:** Skip clustering entirely and concatenate all memories directly. Guard with `if N <= K: return memories`.
- **Embedding model mismatch:** Ensure the embedding model used for clustering matches or is compatible with the model used for memory encoding. Clustering on embeddings from model A while the LLM consumes embeddings from model B produces poor cluster assignments.
- **Context overflow after compression:** If K x D_m still exceeds the budget, reduce K further or reduce D_m (tokens per memory) via additional compression before clustering.

## Limitations

- **Requires meaningful semantic diversity.** If all memories are about the same topic, clustering provides no benefit over averaging. The technique shines when users have diverse interaction histories.
- **K-means assumes spherical clusters.** Memories with complex, non-convex semantic relationships may be poorly served. Consider hierarchical clustering or DBSCAN for more complex memory distributions, at the cost of additional compute.
- **Averaging within clusters is lossy.** Fine-grained distinctions between memories in the same cluster (e.g., two slightly different opinions on the same topic) are smoothed out. For tasks requiring high-fidelity recall of specific memories, direct concatenation with aggressive retrieval filtering may be preferable.
- **Static K across queries.** The optimal K may vary per query — a broad query benefits from more clusters (wider coverage) while a narrow query benefits from fewer (focused detail). Adaptive K selection is not addressed in the paper.
- **On-device compute cost of K-means.** While K-means is lightweight on modern hardware, extremely resource-constrained devices (sub-1GB RAM) may need to use approximate clustering or pre-computed cluster assignments.
- **Text-level merging is approximate.** When working with text strings rather than embeddings, merging within clusters requires either concatenation (which only partially compresses) or LLM-based summarization (which adds inference cost). The full benefit comes from embedding-level averaging.

## Reference

**Paper:** [Clustering-driven Memory Compression for On-device Large Language Models](https://arxiv.org/abs/2601.17443v1) — Bohdal et al., ICASSP 2026. Focus on Section 3 (Method) for the clustering-merging pipeline, Table 1 for comparison against concatenation and averaging baselines across three LaMP benchmark tasks, and Figure 4 for the effect of varying K on generation quality.