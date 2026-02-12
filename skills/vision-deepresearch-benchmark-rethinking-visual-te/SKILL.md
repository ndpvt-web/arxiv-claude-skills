---
name: "vision-deepresearch-benchmark-rethinking-visual-te"
description: "Build and evaluate Vision-DeepResearch pipelines that combine cropped visual search with multi-hop textual search for robust multimodal fact-finding. Use when: 'build a visual search QA pipeline', 'evaluate my MLLM visual retrieval', 'implement cropped image search workflow', 'create a VQA benchmark with search', 'multi-round visual search agent', 'prevent answer leakage in VQA benchmarks'."
---

# Vision-DeepResearch: Multi-Round Cropped-Search for Multimodal Fact-Finding

This skill enables Claude to help users build, evaluate, and debug Vision-DeepResearch systems -- multimodal pipelines where an MLLM iteratively crops regions from an image, issues visual search queries, performs textual web search, and synthesizes multi-hop answers. The core technique is **Multi-turn Visual Forcing (MVF)**: a zero-shot workflow that guides models through fine-grained, multi-scale visual retrieval with explicit cross-modal evidence verification, shown to nearly double answer accuracy on realistic visual QA tasks (e.g., Gemini: 16.2 -> 30.0 on VDR-Bench).

## When to Use

- When the user is building a visual QA system that must search the web using image crops (not just full-image reverse search)
- When designing a benchmark for multimodal search and needing to prevent answer leakage through textual cues or model priors
- When implementing an agentic loop where an MLLM iteratively crops, queries, and reasons over search results
- When evaluating whether a VQA dataset genuinely requires visual search or can be shortcut via text-only retrieval
- When the user wants to add multi-hop reasoning over visual entities linked to knowledge graphs
- When debugging a "lazy search" problem where a strong model relies on priors instead of actually searching

## Key Technique

**The Problem with Existing Visual Search:** Standard Vision-DeepResearch pipelines query a search engine with the full input image. In practice, this retrieves near-exact duplicates whose metadata leaks the answer, creating a "perfect-retrieval bias." Real-world images contain multiple entities, background distractors, and ambiguous cues -- full-image search fails in these conditions. Additionally, many VQA benchmarks allow models to answer from world knowledge or textual cues alone, never actually testing visual retrieval.

**Multi-turn Visual Forcing (MVF):** The paper's proposed workflow forces the model through iterative region cropping, refined visual querying, and explicit cross-modal evidence verification. Instead of one full-image query, the system: (1) localizes candidate entities within the image, (2) crops salient regions (objects, logos, landmarks, faces), (3) issues multiple cropped-image search queries to a web-scale image search engine, (4) refines hypotheses across rounds based on returned results, and (5) verifies answers through cross-modal reasoning that links visual evidence to textual knowledge. This operates under a fixed search budget -- the same number of total queries as baseline methods -- yet nearly doubles accuracy by spending those queries on targeted crops rather than redundant full-image searches.

**Five-Stage Benchmark Curation:** To build leak-proof VQA datasets, the paper uses: (1) image pre-filtering for multi-entity, visually rich scenes, (2) manual cropping and visual search to identify ground-truth entities, (3) entity extraction with LLM verification plus human validation, (4) seed QA generation requiring explicit visual grounding, and (5) complexity expansion via knowledge-graph random walks to create multi-hop questions, with automatic solvability checks and human quality filtering at every stage.

## Step-by-Step Workflow

### For Building a Visual Search QA Pipeline

1. **Ingest the image and generate an entity map.** Use an MLLM to identify all visually distinct entities in the image (people, objects, logos, landmarks, text, architecture). Output a list of bounding boxes or region descriptions with confidence scores.

2. **Rank and crop candidate regions.** Sort entities by salience and relevance to the query. For each top-k candidate, extract a tightly cropped sub-image that isolates the entity from background distractors. Aim for crops that contain a single identifiable entity with minimal clutter.

3. **Issue cropped-image search queries.** Send each cropped region to a reverse image search API (Google Lens, Bing Visual Search, or a CLIP-based retrieval index). Collect the top-N results per crop, capturing both the matched image URLs and any associated metadata (titles, descriptions, page text).

4. **Filter and verify search results.** For each returned result, use an MLLM to assess visual similarity between the crop and the matched image. Discard results where the match is coincidental (similar color/shape but different entity). Score remaining matches by semantic relevance to the original query.

5. **Extract entities and link to knowledge sources.** From verified search results, extract named entities (people, places, organizations, dates). Cross-reference these against a knowledge base or perform targeted text search queries to gather additional facts needed for multi-hop reasoning.

6. **Perform multi-hop reasoning.** Chain the visual evidence (crop -> matched entity) with textual evidence (entity -> knowledge graph facts) to construct a reasoning path. For multi-hop questions (e.g., "Who founded the company whose logo appears in this image?"), explicitly trace: visual entity -> company name -> founder.

7. **Verify cross-modal consistency.** Before producing a final answer, check that the visual evidence and textual evidence are mutually consistent. If a crop matched "Eiffel Tower" but text search says the building was designed by someone who never worked in Paris, flag the inconsistency and re-search.

8. **Iterate if confidence is low.** If the answer confidence is below threshold or evidence is contradictory, generate new crops (different scale, different region), issue additional search queries with refined terms, and repeat steps 3-7 within the remaining search budget.

### For Building a Leak-Proof VQA Benchmark

1. **Pre-filter images for complexity.** Select images containing multiple entities and realistic visual complexity. Reject images where a single dominant entity makes full-image search trivially effective. Use an MLLM as a filter to score visual richness.

2. **Manually crop and search to establish ground truth.** Human annotators extract salient local regions and verify that these crops (not the full image) are required to identify the entity via reverse image search.

3. **Generate candidate QA pairs requiring visual grounding.** Use an LLM to synthesize questions that explicitly require recognizing a visual entity in the image -- not answerable from the text of the question alone.

4. **Expand complexity via knowledge-graph walks.** Link each visual entity to a knowledge graph node. Perform random walks (1-3 hops) to generate multi-hop questions whose answers require both visual identification and factual retrieval.

5. **Run leakage tests.** Test each QA pair under text-only conditions (replace image with caption) and direct-answer conditions (no search). If accuracy exceeds a threshold under these degraded conditions, the question leaks information and should be revised or removed.

6. **Final expert review.** Human reviewers verify that reasoning paths are explicit, valid, and unambiguous, and that no shortcut solutions exist.

## Concrete Examples

**Example 1: Building a Multi-Round Cropped Search Agent**

User: "I want to build a Python agent that takes an image and a question, then uses Google Lens to search for entities in the image and answer multi-hop questions."

Approach:
1. Define the agent loop with a fixed search budget (e.g., 8 queries max)
2. Use an MLLM (GPT-4o, Gemini, Claude) to identify regions of interest and output bounding boxes
3. Crop each region using PIL/OpenCV
4. Query Google Lens API (or SerpAPI's Google Lens endpoint) with each crop
5. Parse results, extract entity names, and run follow-up text searches
6. Synthesize answer with explicit evidence chain

Output structure:
```python
class CroppedSearchAgent:
    def __init__(self, mllm_client, search_client, max_queries=8):
        self.mllm = mllm_client
        self.search = search_client
        self.budget = max_queries
        self.queries_used = 0

    def identify_regions(self, image, question):
        """Ask MLLM to identify salient regions as bounding boxes."""
        prompt = (
            f"Given this image and the question: '{question}', "
            "identify up to 4 visually distinct regions that might "
            "contain entities relevant to answering the question. "
            "Return as JSON list of {x, y, w, h, description}."
        )
        return self.mllm.analyze(image, prompt)

    def crop_and_search(self, image, regions):
        """Crop each region and run reverse image search."""
        results = []
        for region in regions:
            if self.queries_used >= self.budget:
                break
            crop = image.crop((region.x, region.y,
                               region.x + region.w, region.y + region.h))
            search_results = self.search.reverse_image(crop)
            self.queries_used += 1
            results.append({
                "region": region.description,
                "matches": search_results[:5]
            })
        return results

    def reason_and_answer(self, question, visual_evidence, text_evidence):
        """Multi-hop reasoning over combined evidence."""
        prompt = (
            f"Question: {question}\n"
            f"Visual search found: {visual_evidence}\n"
            f"Text search found: {text_evidence}\n"
            "Chain your reasoning step by step, citing evidence."
        )
        return self.mllm.generate(prompt)
```

**Example 2: Testing a VQA Dataset for Answer Leakage**

User: "I have a VQA dataset with 500 image-question-answer triples. How do I check if models can answer without actually using visual search?"

Approach:
1. Run three evaluation conditions on every instance
2. Compare accuracy across conditions to identify leaky questions
3. Remove or revise questions that are solvable without visual search

```python
def test_leakage(dataset, mllm):
    results = []
    for item in dataset:
        # Condition 1: Direct answer, no search, no image
        text_only_score = mllm.answer(
            question=item["question"], image=None, search=False
        )
        # Condition 2: With image caption instead of image
        caption = mllm.caption(item["image"])
        caption_score = mllm.answer(
            question=item["question"], context=caption, search=False
        )
        # Condition 3: Full pipeline (ground truth condition)
        full_score = mllm.answer(
            question=item["question"], image=item["image"], search=True
        )
        results.append({
            "id": item["id"],
            "text_only": text_only_score,
            "caption_only": caption_score,
            "full_pipeline": full_score,
            "leaks": text_only_score > 0.5 or caption_score > 0.5
        })
    leaked = [r for r in results if r["leaks"]]
    print(f"{len(leaked)}/{len(dataset)} questions leak answers")
    return results
```

**Example 3: Evaluating Entity Recall in a Search Pipeline**

User: "How do I measure whether my visual search agent is finding the right entities?"

Approach:
1. Define gold entity sequences for each question (the entities the agent should find en route to the answer)
2. Use semantic matching (not string matching) to compare agent-retrieved entities against gold
3. Compute Entity Recall (ER) as the fraction of gold entities found

```python
def entity_recall(agent_entities, gold_entities, judge_llm):
    """
    Compute Entity Recall using LLM-as-judge for semantic matching.
    Handles synonyms (e.g., 'Eiffel Tower' matches 'Tour Eiffel').
    """
    matched = 0
    for gold in gold_entities:
        for agent_ent in agent_entities:
            prompt = (
                f"Are '{agent_ent}' and '{gold}' referring to the "
                f"same real-world entity? Answer yes or no."
            )
            if judge_llm.generate(prompt).strip().lower() == "yes":
                matched += 1
                break
    return matched / len(gold_entities) if gold_entities else 0.0
```

## Best Practices

- **Do:** Crop tightly around individual entities before searching. Full-image search retrieves near-duplicates whose metadata shortcuts the reasoning process.
- **Do:** Enforce a fixed search budget and allocate queries strategically -- spend more on ambiguous regions, skip clearly identified ones.
- **Do:** Use semantic entity matching (LLM-as-judge) rather than string matching when evaluating search recall. "NYC" and "New York City" should match.
- **Do:** Run leakage tests (text-only, caption-only baselines) on any VQA benchmark before trusting its results.
- **Avoid:** Sending the full uncropped image as the only search query. This produces the "perfect-retrieval bias" where metadata leaks answers.
- **Avoid:** Trusting that high direct-answer accuracy means the model is searching well. The "lazy search" phenomenon shows strong models rely on priors instead of actually searching -- check Entity Recall separately from answer accuracy.

## Error Handling

- **Search API returns no results for a crop:** Re-crop at a different scale (zoom out 20% to include more context, or zoom in to isolate a sub-element). If still no results, fall back to describing the region textually and using text search.
- **Multiple conflicting entity matches for one crop:** Present all candidates to the MLLM with the original image context and ask it to disambiguate. Use the surrounding scene as a tiebreaker.
- **Search budget exhausted before confident answer:** Return the best-supported hypothesis with an explicit confidence score and list which entities remain unverified.
- **Knowledge graph walk produces unanswerable questions:** During benchmark curation, run automatic solvability checks -- if no model in a panel of 3+ MLLMs can answer with full search access, discard the question.
- **Judge LLM disagrees with human evaluation:** Calibrate the judge by running it on a held-out set with known correct matches. Adjust prompts or switch judge models if agreement (Cohen's kappa) drops below 0.8.

## Limitations

- The cropped-search workflow assumes access to a reverse image search API with reasonable coverage. Niche or very recent entities may not be indexed.
- MVF operates under a fixed budget constraint. For images with many entities (>10), the budget may be insufficient to search all relevant regions.
- Semantic entity matching via LLM-as-judge introduces its own error rate. For high-stakes evaluation, supplement with human spot-checks.
- The approach is designed for factual VQA (entity recognition, attribute lookup, multi-hop factual chains). It does not address subjective, aesthetic, or reasoning-heavy visual questions.
- Benchmark curation requires substantial human annotation effort (manual cropping, expert review at multiple stages). Fully automated curation pipelines remain an open problem.

## Reference

**Paper:** [Vision-DeepResearch Benchmark: Rethinking Visual and Textual Search for Multimodal Large Language Models](https://arxiv.org/abs/2602.02185v1) (Zeng et al., 2026). Look for: the five-stage curation pipeline (Section 3), the Multi-turn Visual Forcing workflow (Section 4), the controlled evaluation settings isolating visual vs. textual search (Section 5), and the "lazy search" phenomenon analysis.

**Code:** [https://github.com/Osilly/Vision-DeepResearch](https://github.com/Osilly/Vision-DeepResearch)