---
name: "vision-deepresearch-incentivizing-deepresearch-cap"
description: "Multi-turn, multi-entity, multi-scale visual and textual deep research agent for answering complex questions about images. Implements the Vision-DeepResearch paradigm: iterative reasoning-then-search with progressive visual cropping and text retrieval. Use when: 'research this image', 'identify everything in this photo', 'what is the history behind this building', 'find information about objects in this picture', 'deep research this visual scene', 'multi-hop visual question answering'."
---

# Vision-DeepResearch: Multi-Turn Multi-Scale Visual and Textual Deep Research

This skill enables Claude to conduct deep, iterative research on images and visual scenes by orchestrating multi-turn reasoning with multi-entity, multi-scale visual and textual search. Rather than making a single image query or a few text searches, this approach performs dozens of reasoning steps across hundreds of search interactions — progressively cropping image regions at different scales, issuing targeted text queries, visiting web pages, and aggregating evidence from diverse sources until the question is answered with confidence. The technique is based on the Vision-DeepResearch paradigm (Huang et al., 2026) which demonstrated that iterative, noise-robust multi-scale search dramatically outperforms single-shot retrieval for complex visual questions.

## When to Use

- When the user provides an image and asks a question requiring external factual knowledge (e.g., "What year was this building constructed?")
- When identifying multiple entities in a complex scene and researching each one separately
- When a single reverse-image search would fail due to visual noise, clutter, or partial visibility
- When answering multi-hop questions that chain visual identification to factual retrieval (e.g., "Who designed the bridge in this photo, and what other bridges did they design?")
- When the user asks to "deep research" or "investigate" a photograph, artwork, landmark, or product
- When the image contains multiple objects of interest that each need independent identification and fact-gathering
- When prior simple searches returned irrelevant results and a more systematic approach is needed

## Key Technique

**Multi-Scale Visual Search.** The core insight is that the same visual entity yields markedly different search results depending on the query scale. A full-image search may return generic matches overwhelmed by background noise. An entity-level crop (tight bounding box around a specific object) may match better but miss context. Vision-DeepResearch addresses this by performing searches at multiple scales per entity: full-image, entity-level bounding box, and zoomed region-level crops. Each scale is tried iteratively — if one scale fails to produce relevant results, the agent reasons about why, adjusts the crop, and retries. This trial-and-error approach is critical for robustly hitting real-world search engines.

**Multi-Turn Reasoning with Evidence Accumulation.** Instead of a linear pipeline, the agent operates in a reasoning loop: think about what information is still missing, decide whether to do a visual search (at what scale and on which entity) or a text search (with what query), execute the search, observe results, and update its working hypothesis. The agent maintains a running evidence buffer and a judge function that evaluates whether accumulated evidence is sufficient to answer the question. This loop can run for up to 50 turns, enabling deep multi-hop chains like: identify landmark → find architect → find architect's other works → cross-reference dates.

**Cold-Start Then Reinforce.** The training pipeline first builds supervised trajectories by having an MLLM generate reasoning chains paired with actual search tool calls and verified answers (cold-start SFT). Then reinforcement learning (GRPO with binary accuracy reward) fine-tunes the model to discover shorter, more efficient search strategies while maintaining accuracy. For Claude's purposes, this translates to a discipline: start with broad exploratory searches, then progressively narrow and refine based on what you learn — mimicking the RL-optimized behavior of producing efficient trajectories.

## Step-by-Step Workflow

1. **Analyze the image and decompose the question.** Identify all distinct visual entities relevant to the question. For each entity, note its position in the image, distinguishing features, and what factual information is needed. Write an explicit decomposition: "To answer X, I need to identify entity A, find fact B about it, then connect to C."

2. **Prioritize entities by information need.** Rank entities by how critical they are to answering the question. Start with the entity most likely to unlock downstream answers. If the question is single-entity, identify the best crop region.

3. **Perform multi-scale visual search for the primary entity.** Execute searches at three scales in sequence:
   - **Full-image**: Search with the entire image to see if the scene itself is recognizable.
   - **Entity-level crop**: Crop a tight bounding box around the target entity and search.
   - **Region-level crop**: If entity-level fails, try a slightly wider or narrower crop, or crop from a different angle/portion of the entity.
   After each search, evaluate whether results are relevant. If a scale produces useful leads, follow them; otherwise move to the next scale.

4. **Extract and verify web evidence.** For each promising search result, visit the source page. Extract the specific claim or fact needed. Cross-reference against at least one other source when possible. Record the evidence with its source URL.

5. **Issue targeted text searches to fill gaps.** Based on what the visual search revealed (e.g., a name, a location, a date), construct precise text queries to retrieve additional facts. Use entity names, dates, or keywords discovered in step 4. Avoid vague queries; be as specific as the accumulated evidence allows.

6. **Repeat for secondary entities.** Move to the next entity in the priority list. Repeat steps 3-5. Cross-reference findings between entities to detect contradictions or confirmations.

7. **Evaluate evidence sufficiency.** After each search round, explicitly assess: "Do I have enough verified evidence to answer the original question? Are there contradictions that need resolution? Is there a critical gap?" If sufficient, proceed to synthesis. If not, identify the specific gap and plan the next search.

8. **Handle search failures gracefully.** If visual search returns no relevant results after all three scales, try: (a) describe the entity in text and search by description, (b) search for contextual clues (text in the image, logos, architectural style), (c) broaden the search to the category level (e.g., "Art Deco skyscraper" instead of the specific building).

9. **Synthesize the final answer.** Combine all gathered evidence into a coherent response. Cite specific sources. Distinguish between confirmed facts and reasonable inferences. Structure the answer to directly address the original question first, then provide supporting detail.

10. **Report confidence and limitations.** State which claims are well-sourced, which are inferred, and which remain uncertain. If the research hit dead ends, explain what was tried and why it failed.

## Concrete Examples

**Example 1: Identifying a Historical Building**

```
User: [uploads photo of an ornate building] What is this building and when was it built?

Approach:
1. Analyze image: Single ornate building, appears European, Baroque style,
   visible dome and columned facade. No visible text or signage.
2. Full-image visual search → returns generic "Baroque architecture" results,
   not specific enough.
3. Entity-level crop of the distinctive dome and upper facade → visual search
   returns a match: "Karlskirche, Vienna."
4. Visit top 3 result pages. Two sources confirm: Karlskirche (St. Charles's
   Church), Vienna, Austria.
5. Text search: "Karlskirche Vienna construction date architect" →
   Built 1716-1737, designed by Johann Bernhard Fischer von Erlau.
6. Cross-reference on a second source (Wikipedia, architectural database) →
   confirmed dates and architect.
7. Evidence sufficient: building identified, dates confirmed from 2+ sources.

Output:
This is the Karlskirche (St. Charles's Church) in Vienna, Austria. It was
built between 1716 and 1737, designed by the Baroque architect Johann Bernhard
Fischer von Erlau. The distinctive oval dome and the two flanking columns
modeled after Trajan's Column in Rome are signature features of the design.
Sources: [Vienna Tourism Board], [Architectural Heritage Database]
```

**Example 2: Multi-Entity Scene Research**

```
User: [uploads photo of a museum gallery with three paintings visible]
Tell me about each painting in this photo.

Approach:
1. Decompose: Three distinct paintings visible. Label them Left, Center, Right.
   Each needs independent identification and fact-gathering.
2. Entity-level crop of Left painting → visual search returns match:
   "The Arnolfini Portrait" by Jan van Eyck (1434).
3. Entity-level crop of Center painting → visual search returns ambiguous
   results. Region-level crop focusing on the distinctive figure in the
   foreground → match: "The Ambassadors" by Hans Holbein the Younger (1533).
4. Entity-level crop of Right painting → no clear match. Describe: "Portrait
   of a woman in blue dress, Renaissance style, possibly Flemish."
   Text search: "Flemish Renaissance portrait woman blue dress National Gallery"
   → match: "A Lady with a Squirrel and a Starling" by Holbein.
5. For each painting, text search for key facts: date, current location,
   notable features.
6. Cross-reference: All three are in the National Gallery, London — consistent
   with the gallery setting in the photo.

Output:
Left: "The Arnolfini Portrait" by Jan van Eyck, 1434. Oil on oak panel,
one of the most complex paintings of the early Northern Renaissance.

Center: "The Ambassadors" by Hans Holbein the Younger, 1533. Famous for
the anamorphic skull in the foreground.

Right: "A Lady with a Squirrel and a Starling" by Hans Holbein the Younger,
c. 1526-28. Portrait believed to depict Anne Lovell.

All three are housed in the National Gallery, London.
```

**Example 3: Multi-Hop Visual Question**

```
User: [uploads photo of a distinctive bridge] Who designed this bridge,
and what was their most famous other project?

Approach:
1. Single entity (bridge), but multi-hop: need identity → architect → other works.
2. Full-image search → generic bridge results.
3. Entity-level crop focusing on the distinctive cable-stayed pylons →
   match: Millau Viaduct, France.
4. Text search: "Millau Viaduct structural engineer architect designer" →
   Structural engineer: Michel Virlogeux. Architect: Norman Foster.
5. Text search: "Norman Foster most famous building" → multiple sources
   cite 30 St Mary Axe (The Gherkin), London, or the Reichstag dome, Berlin.
6. Text search: "Michel Virlogeux other bridges" → Pont de Normandie.
7. Evidence sufficient for both the architect and engineer pathways.

Output:
This is the Millau Viaduct in southern France, the world's tallest bridge
(343m structural height). It was designed by structural engineer Michel
Virlogeux and architect Lord Norman Foster, completed in 2004.

Norman Foster's most famous other project is arguably 30 St Mary Axe
("The Gherkin") in London or the renovated Reichstag dome in Berlin.
Michel Virlogeux is also known for designing the Pont de Normandie.
```

## Best Practices

- **Do:** Search at multiple visual scales before concluding an entity is unidentifiable. The difference between a full-image and a tight crop can be the difference between no results and a perfect match.
- **Do:** Maintain an explicit evidence log as you search. Track what you've confirmed, what's contradicted, and what's still unknown. This prevents redundant searches and ensures nothing is forgotten.
- **Do:** Use findings from visual search to construct precise text queries. A name discovered via image search becomes a high-precision text query anchor.
- **Do:** Cross-reference facts from at least two independent sources before stating them as confirmed.
- **Avoid:** Relying on a single full-image search and giving up if it fails. The whole point of multi-scale search is that different crops produce different results.
- **Avoid:** Issuing vague text queries like "old building Europe" when you could search "Neoclassical church with twin columns Vienna" based on visual analysis.
- **Avoid:** Treating the first search result as definitive. Verify before reporting, especially for dates, attributions, and historical claims.
- **Avoid:** Running excessive searches when evidence is already sufficient. After each round, explicitly evaluate whether you can answer the question.

## Error Handling

| Problem | Recovery Strategy |
|---|---|
| Visual search returns zero relevant results at all scales | Fall back to text description of visual features. Search by style, era, material, visible text, or contextual clues. |
| Search results contradict each other | Seek a third source. Prefer authoritative sources (museum websites, official records) over secondary aggregators. Report the contradiction to the user if unresolved. |
| Entity is too small or blurry to crop meaningfully | Describe the entity based on available visual cues and search by description. Acknowledge limited confidence. |
| Search returns results for a visually similar but wrong entity | Use distinguishing features (color, text, surrounding context) to filter. Add negative keywords to text queries. |
| Question requires information not available online | State clearly what was found and what remains unknown. Suggest alternative approaches the user could try (e.g., contacting an institution directly). |
| Context window fills up during long research chains | Summarize intermediate findings, discard raw search results that have already been processed, and continue with the condensed evidence buffer. |

## Limitations

- This approach depends on web search tools and image search capabilities. Without access to visual search APIs, the multi-scale visual search component cannot execute — fall back to text-only description-based research.
- Effectiveness is bounded by what search engines index. Obscure, private, or very recent subjects may not return useful results regardless of search strategy.
- Multi-scale cropping assumes the image is of sufficient resolution. Low-resolution images may not benefit from region-level crops.
- The approach is designed for factual, knowledge-seeking questions. It is not intended for subjective aesthetic judgments, creative interpretation, or tasks that don't require external evidence.
- Long research chains (many turns) consume significant context. For questions answerable with one or two searches, this full paradigm is overkill — use proportional effort.

## Reference

**Paper:** [Vision-DeepResearch: Incentivizing DeepResearch Capability in Multimodal Large Language Models](https://arxiv.org/abs/2601.22060v1) (Huang et al., 2026). Focus on Section 3 (method) for the multi-scale search action space and trajectory construction, and Section 4 for the GRPO-based RL training that optimizes search efficiency. The key takeaway: iterative multi-scale visual search with evidence accumulation dramatically outperforms single-shot retrieval for complex visual questions.