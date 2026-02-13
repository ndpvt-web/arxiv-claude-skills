---
name: "deepimagesearch-benchmarking-multimodal-agents"
description: "Build agentic image retrieval systems that perform multi-step contextual reasoning over visual histories instead of isolated semantic matching. Use when: 'build a context-aware image search agent', 'retrieve images using temporal reasoning', 'search photos by contextual clues across events', 'implement dual-memory agent for image retrieval', 'create a visual history exploration pipeline', 'benchmark multimodal agents on retrieval tasks'."
---

# DeepImageSearch: Agentic Multi-Step Image Retrieval over Visual Histories

This skill teaches Claude to build image retrieval systems that go beyond single-query semantic matching. Based on the DeepImageSearch paradigm, it reformulates image retrieval as an autonomous exploration task where an agent plans search trajectories, coordinates fine-grained perception tools, and connects scattered clues distributed across temporal sequences of images. The core insight is that target relevance often depends on other images in the collection (P(R|Q,C)), not just isolated query-image similarity -- requiring corpus-level contextual reasoning with a modular agent framework and dual-memory architecture.

## When to Use

- When the user needs to search a large personal photo library using contextual clues that span multiple events (e.g., "find photos of the dog we saw at the same park where we had Sarah's birthday")
- When building an agent that must reason across temporal sequences of images rather than matching against individual frames
- When implementing a retrieval system where queries are ambiguous without cross-referencing metadata, timestamps, locations, or recurring entities across images
- When designing a modular tool-use agent with memory management for long-horizon visual navigation tasks
- When constructing benchmarks for evaluating multimodal agents on context-dependent retrieval
- When the user wants to build a memory graph over a photo collection linking people, places, events, and visual clues

## Key Technique

**Agentic Retrieval vs. Embedding-Based Retrieval:** Traditional systems encode a query and each image independently, then rank by cosine similarity. DeepImageSearch shows this hits a fundamental ceiling (~10-14% Recall@3) on context-dependent queries because the answer depends on relationships *between* images. The agentic approach instead gives a model a toolkit -- similarity search, metadata filtering, photo viewing, web lookup -- and lets it plan multi-step reasoning chains: narrowing candidates by time/location, visually inspecting subsets, cross-referencing recurring entities, and iteratively refining.

**Dual-Memory System for Long Horizons:** When exploring large photo histories (100K+ images spanning years), conversation context fills up fast. The framework uses two memory layers: (1) *Explicit state memory* -- named photo subsets persisted as dictionary mappings (subset_name -> photo_IDs) enabling chained operations like filter->save->search_within->filter, and (2) *Compressed context memory* triggered at token limits, which distills the full interaction history into a *session memory* (high-level goals and key findings) plus a *working memory* (current subgoal and immediate action plans). This lets the agent maintain reasoning coherence across dozens of tool calls without losing track of its search trajectory.

**Memory Graph for Association Discovery:** To find non-obvious connections, the method builds a heterogeneous graph G=(V,E) with node types (Photo, Photoset, VisualClue, Person) and edge types (structural: photoset->photo->clue/person; associative: cross-event links between matching visual elements). Queries are synthesized by sampling subgraphs via balanced random walks, ensuring they require genuine multi-step reasoning rather than single-hop matching.

## Step-by-Step Workflow

1. **Decompose the query into structured components.** Parse the user's natural language query into three parts: *Episode* (the latent spatiotemporal context, e.g., "a beach trip last summer"), *Episode Breakdown* (the logical reasoning chain with explicit relations, e.g., "find beach photos -> filter to July-August -> identify ones with the red umbrella"), and *Target* (specific visual/metadata constraints on the final results, e.g., "photos showing the sunset behind the umbrella").

2. **Index the visual corpus with per-user embedding.** Use a vision-language embedding model (e.g., CLIP, SigLIP, or Qwen-VL-Embedding) to encode all images in the collection. Store embeddings in a vector index partitioned per user or per collection. Extract and store metadata (timestamps, GPS coordinates, album/photoset membership) in a structured sidecar.

3. **Implement the five core tools as callable functions:**
   - `ImageSearch(query, top_k, search_within?)` -- multimodal similarity search accepting text, image, or interleaved queries; supports scoped search within a named subset.
   - `GetMetadata(photo_ids, fields)` -- retrieves timestamps, addresses, album info for specified photos.
   - `FilterMetadata(expression, filter_within?)` -- applies boolean expressions over metadata fields (date ranges, location matching with alias normalization, album filters).
   - `ViewPhotos(photo_ids, max=20)` -- injects actual images into the agent context for direct visual inspection.
   - `WebSearch(query)` -- external entity resolution for landmarks, events, or objects the agent cannot identify from photos alone.

4. **Initialize explicit state memory as an empty dictionary.** Each tool that produces a photo subset (ImageSearch, FilterMetadata) can save results to a named key. Subsequent tools accept a `search_within` or `filter_within` parameter referencing these keys, enabling incremental candidate narrowing without re-processing the full corpus.

5. **Execute the first reasoning step: broad candidate retrieval.** Use ImageSearch with a text description derived from the Episode Breakdown to pull an initial candidate set (top-50 to top-200). Save this as a named subset (e.g., `initial_candidates`).

6. **Iteratively narrow candidates through metadata filtering and visual inspection.** Apply FilterMetadata to constrain by date range, location, or album. Then use ViewPhotos on the remaining candidates (batches of up to 20) to visually verify which images match the target criteria. Save verified candidates to new named subsets at each step.

7. **Cross-reference entities across events for inter-event queries.** When the query requires connecting information across different events (53% of realistic queries do), use ImageSearch with a *visual* query -- feed a candidate image back as the query to find visually similar items in other photosets. Verify associations by checking for matching unique identifiers (text on signs, license plates), visual feature consistency, or shared metadata patterns.

8. **Trigger context compression when approaching token limits.** When the conversation history nears the context window (e.g., 128K tokens), generate a compressed summary with two parts: *session memory* capturing accumulated high-level goals and confirmed findings, and *working memory* capturing the current subgoal and next planned actions. Replace the full history with this compressed state to free context space for continued reasoning.

9. **Assemble final results with confidence justification.** Collect all verified target photo IDs, deduplicate, and return with a reasoning trace explaining how each target was located (which clues led to which intermediate steps). Include the subset chain that produced each result for auditability.

10. **Evaluate results using Exact Match (EM) and F1 metrics.** EM requires the retrieved set to exactly match ground truth; F1 measures precision-recall at the photo level. For benchmarking, run N parallel instances and report Best@k (best F1 across k independent runs) to measure test-time scaling behavior.

## Concrete Examples

**Example 1: Intra-Event Retrieval -- Finding Specific Photos Within an Event**

```
User: "I need to find the photos of the fireworks from our New Year's
Eve party -- specifically the ones taken from the rooftop, not the
street-level ones."

Approach:
1. Decompose: Episode="New Year's Eve party", Breakdown="find NYE
   photos -> filter to nighttime Dec 31 -> identify fireworks ->
   distinguish rooftop vantage point from street level",
   Target="fireworks photos with downward/elevated viewing angle"
2. ImageSearch("New Year's Eve fireworks celebration", top_k=100)
   -> save as `nye_candidates`
3. FilterMetadata("date >= 2024-12-31 AND date <= 2025-01-01",
   filter_within=`nye_candidates`) -> save as `nye_filtered`
4. ViewPhotos(`nye_filtered`[:20]) -- visually inspect for
   elevated vantage point (city skyline below, rooftop railing
   visible, downward angle on fireworks)
5. Separate rooftop shots from street-level based on visual cues

Output:
{
  "targets": ["photo_4821", "photo_4823", "photo_4825"],
  "reasoning": "Filtered 100 NYE candidates to 34 on Dec 31 night.
   Visual inspection identified 3 photos with elevated perspective
   showing rooftop railing and city skyline below fireworks.",
  "subset_chain": ["nye_candidates(100)", "nye_filtered(34)",
   "rooftop_verified(3)"]
}
```

**Example 2: Inter-Event Retrieval -- Connecting Clues Across Events**

```
User: "Find all photos of the stray cat we kept seeing -- it showed
up at the cafe in Rome and then again at our Airbnb in Florence."

Approach:
1. Decompose: Episode="Italy trip, multiple cities",
   Breakdown="find Rome cafe photos -> locate cat photos among
   them -> use cat's visual features to search Florence Airbnb
   photos -> verify same cat", Target="photos of the specific
   stray cat across both locations"
2. FilterMetadata("match_address('Rome')", top_k=500)
   -> save as `rome_photos`
3. ImageSearch("stray cat at outdoor cafe", search_within=`rome_photos`,
   top_k=30) -> save as `rome_cat_candidates`
4. ViewPhotos(`rome_cat_candidates`[:20]) -- identify the specific
   cat (orange tabby with notched ear)
5. Save confirmed cat photos; use one as visual query:
   ImageSearch(image=photo_2341, search_within=None, top_k=50)
   -> save as `cat_everywhere`
6. FilterMetadata("match_address('Florence')",
   filter_within=`cat_everywhere`) -> save as `florence_cat`
7. ViewPhotos(`florence_cat`) -- verify same cat by visual features
   (orange tabby, notched left ear, white chest patch)

Output:
{
  "targets": ["photo_2341", "photo_2343", "photo_5102", "photo_5107"],
  "reasoning": "Located orange tabby at Rome cafe (2 photos). Used
   visual query to find similar cats across full collection. Filtered
   to Florence, verified 2 additional photos of same cat by matching
   notched ear and white chest patch.",
  "subset_chain": ["rome_photos(487)", "rome_cat_candidates(30)",
   "rome_cat_verified(2)", "cat_everywhere(50)",
   "florence_cat(5)", "florence_verified(2)"]
}
```

**Example 3: Building a Memory Graph for a Photo Collection**

```
User: "I want to build a structured knowledge graph over my photo
library so I can do contextual searches later."

Approach:
1. Extract visual cues from each photo using a VLM: landmarks,
   text, distinctive objects, people (with face clustering)
2. Build nodes: Photo (one per image), Photoset (album/event
   groupings by date proximity), VisualClue (extracted entities),
   Person (face clusters)
3. Add structural edges: Photoset->Photo (containment),
   Photo->VisualClue (detected-in), Photo->Person (appears-in)
4. Mine associative edges: For each VisualClue, embed its
   description and retrieve top-5 candidate matches from other
   photosets. Verify with VLM that the clue refers to the same
   real-world entity (same building, same object, same sign)
5. Store graph in adjacency list format with typed edges

Output (partial):
{
  "nodes": {
    "photos": 12847,
    "photosets": 234,
    "visual_clues": 8921,
    "persons": 47
  },
  "edges": {
    "structural": 31205,
    "associative": 1843
  },
  "sample_association": {
    "clue": "blue vintage Vespa scooter",
    "source": "photoset_Rome_2024/photo_2301",
    "target": "photoset_Amalfi_2024/photo_3892",
    "confidence": 0.94
  }
}
```

## Best Practices

- **Do:** Always decompose queries into Episode / Episode Breakdown / Target before starting tool calls. This structured decomposition prevents reasoning breakdown, which accounts for 36-50% of failures in the benchmark.
- **Do:** Save intermediate results as named subsets and chain operations through them. This explicit state memory prevents the agent from losing track of candidate sets across long reasoning chains.
- **Do:** Use visual queries (image-to-image search) for inter-event association rather than relying solely on text descriptions. Text descriptions of the same entity vary, but visual similarity is stable.
- **Do:** Batch ViewPhotos calls to inspect no more than 20 images at a time. Larger batches degrade visual discrimination accuracy.
- **Avoid:** Relying on a single embedding-based retrieval pass for context-dependent queries. The paper demonstrates a fundamental ceiling of ~14% Recall@3 for this approach -- always plan for multi-step refinement.
- **Avoid:** Skipping metadata filtering when temporal or spatial constraints are available. Removing the GetMetadata tool caused the largest performance drop (-5.7 F1) in ablation studies.
- **Avoid:** Letting the full interaction history grow unbounded. Proactively trigger context compression before hitting token limits rather than waiting for truncation, which loses reasoning state unpredictably.

## Error Handling

| Failure Mode | Frequency | Mitigation |
|---|---|---|
| **Reasoning Breakdown** (incomplete plan, lost constraints mid-chain) | 36-50% of errors | Re-derive the Episode Breakdown from session memory after each compression. Validate that all query constraints are still tracked before returning results. |
| **Visual Discrimination Failure** (confusing similar-looking entities) | ~20-25% of errors | When two candidates look similar, use metadata (different dates, locations) as a tiebreaker. Request higher-resolution views or crop to distinguishing features. |
| **Episode Misgrounding** (wrong event context identified) | ~15% of errors | Cross-validate the identified event against multiple metadata signals (date + location + co-occurring people). If signals conflict, broaden the search window. |
| **Clue Mislocalization** (correct clue found but wrong photo selected) | ~10% of errors | After locating a clue, always verify the final target against ALL original query constraints, not just the clue that led there. |
| **Location Alias Mismatch** (query says "NYC" but metadata says "New York") | Common | Implement a `match_address()` normalization function that falls back to a geocoding service for unresolved aliases. |

## Limitations

- **Requires rich metadata.** The approach depends heavily on timestamps, GPS coordinates, and album structure. Collections without metadata fall back to embedding-only retrieval, which has the documented ceiling.
- **Computational cost scales with reasoning depth.** Each multi-step chain involves multiple VLM calls (ViewPhotos, verification). For real-time applications, consider caching association graphs rather than computing them on-the-fly.
- **Face clustering is fragile.** Cross-event person re-identification degrades with appearance changes (clothing, aging, lighting). The system may create duplicate person nodes for the same individual.
- **Even the best models achieve only ~29% Exact Match.** This is a genuinely hard task. Set user expectations accordingly -- the agent will often find *some* targets but miss others, particularly for inter-event queries requiring 3+ reasoning steps.
- **Context compression loses detail.** While compressed memory preserves high-level goals, fine-grained visual observations from early in the search may be lost. Critical findings should be explicitly saved to named subsets, not just noted in conversation.

## Reference

**Paper:** [DeepImageSearch: Benchmarking Multimodal Agents for Context-Aware Image Retrieval in Visual Histories](https://arxiv.org/abs/2602.10809v1) (Deng et al., 2026). Look for: Section 3 (DISBench construction pipeline and memory graph), Section 4 (ImageSeeker agent framework and dual-memory design), Table 1 (model performance comparison), and Table 3 (ablation study showing tool contribution).