---
name: "shotfinder-imagination-driven-open-domain-video"
description: >
  Retrieve specific video shots from the open web using imagination-driven query expansion
  and temporal localization. Implements the ShotFinder pipeline: expand a shot description
  into search queries by "imagining" the parent video, retrieve candidates via web search,
  then localize the exact shot within each video.
  Trigger phrases: "find a video shot of", "retrieve a clip matching", "search for footage
  showing", "locate a video segment with", "find me b-roll of", "get a shot that looks like"
---

# ShotFinder: Imagination-Driven Open-Domain Video Shot Retrieval

This skill enables Claude to build and execute pipelines that retrieve specific video shots from the open web given natural-language descriptions. The core insight from the ShotFinder paper (Yu et al., 2026) is **imagination-driven query expansion**: instead of extracting keywords directly from a shot description, the system first "imagines" what full video would plausibly contain that shot, then generates search queries targeting that parent video. This bridges the semantic gap between fine-grained shot descriptions and coarse video-level metadata indexed by search engines. The pipeline then localizes the exact temporal segment within retrieved videos using adaptive frame sampling and multimodal matching.

## When to Use

- When the user asks to find specific footage or b-roll matching a visual description (e.g., "Find me a shot of a golden retriever running through autumn leaves in slow motion")
- When building a video editing tool that needs to source clips from the web based on editing briefs or storyboards
- When the user wants to retrieve video segments with specific constraints: temporal position, color tone, visual style, audio characteristics, or resolution
- When implementing a content retrieval system that must go beyond text-based video search to match visual and temporal semantics
- When the user is assembling a video project and needs to programmatically locate candidate shots from YouTube or other platforms
- When building a benchmark or evaluation pipeline for video retrieval systems

## Key Technique

**Imagination-driven query expansion** is the central innovation. Traditional video retrieval takes a shot description like "close-up of hands kneading dough on a floured wooden surface" and extracts keywords ("hands", "kneading", "dough"). This fails because videos are indexed by titles and metadata, not frame-level content. ShotFinder instead prompts an LLM to *imagine the whole video* that would contain this shot: "This shot likely comes from a cooking tutorial or artisan bread-making vlog." The LLM then generates search queries like "artisan sourdough bread tutorial" or "hand-kneading bread recipe video"--terms that match how creators actually title and describe their uploads.

The pipeline has three stages: (1) **Query Expansion via Video Imagination** -- the LLM reasons about what parent video would contain the described shot and generates 3-5 candidate search queries; (2) **Candidate Retrieval** -- queries are submitted to a web search API (YouTube Data API, SerpAPI, or similar) to collect candidate video URLs, which are filtered for accessibility and downloaded; (3) **Temporal Localization** -- an adaptive frame-sampling strategy extracts keyframes (64 frames for videos under 3 min, 128 for 3-10 min, 192 for longer), then a multimodal model scores each frame region against the shot description to identify the best-matching temporal segment.

The paper also defines five **controllable constraint types** that can augment a basic shot description: **Temporal** (shot position relative to surrounding action), **Color** (warm/cool tone, atmospheric coloring), **Style** (live-action vs. animation, visual aesthetic), **Audio** (speech, music, ambient sound), and **Resolution** (technical quality requirements). Constraint-aware retrieval significantly increases difficulty--the best automated system achieves 26.9% accuracy versus 88.5% human performance, indicating this remains an open problem where careful pipeline design matters.

## Step-by-Step Workflow

1. **Parse the shot description and constraints.** Accept the user's natural-language shot description. Identify whether it includes any of the five constraint types (temporal, color, style, audio, resolution). Separate the core visual description from constraint qualifiers. Structure the result as `{description: string, constraints: {type: string, value: string}[]}`.

2. **Imagination-driven query expansion.** Prompt the LLM with the shot description and ask it to imagine the full parent video: "What kind of video would contain this shot? Who would upload it? What would the title be?" Generate 3-5 diverse search queries targeting different plausible parent videos. Include at least one query with platform-specific phrasing (e.g., YouTube title conventions).

3. **Execute web search for candidate videos.** Submit each generated query to a video search API (YouTube Data API v3, SerpAPI with `tbm=vid`, or `yt-dlp --flat-playlist` for search). Collect the top 5 results per query. Deduplicate by video ID. Filter out unavailable, age-restricted, or excessively long (>30 min) videos.

4. **Download and preprocess candidates.** Use `yt-dlp` to download candidate videos at a reasonable quality (720p is sufficient for frame analysis). Extract metadata: duration, title, description, tags. Store in a structured manifest file.

5. **Adaptive frame sampling.** For each candidate video, sample keyframes based on duration: 64 frames for videos under 3 minutes, 128 for 3-10 minutes, 192 for longer. Use uniform temporal spacing. Extract frames with `ffmpeg -vf "select=not(mod(n\,INTERVAL))" -vsync vfn`.

6. **Multimodal frame matching.** Send batches of sampled frames to a vision-language model (GPT-4o, Gemini, Claude with vision) along with the shot description. Ask the model to score each frame's relevance on a 1-5 scale and identify the best-matching frame index. Use a two-pass approach: coarse pass over all frames, then fine pass (1-frame-per-second) in the 30-second window around the best coarse match.

7. **Constraint verification.** For each candidate shot, verify constraint satisfaction separately. For temporal constraints, check surrounding frames for context. For color, analyze the dominant palette of the shot region. For style, classify as live-action/animation/graphics. For audio, extract and analyze the audio track in the shot window using `ffmpeg` and a speech/music classifier. For resolution, check actual frame dimensions and bitrate.

8. **Rank and return results.** Score each candidate shot as: `relevance_score * constraint_satisfaction_score`. Return the top-K results with: video URL, start/end timestamps, thumbnail frame, confidence score, and a natural-language explanation of why the shot matches.

9. **Generate retrieval report.** Produce a structured output with the ranked shots, metadata, and any constraints that could not be fully verified. Include direct links or `yt-dlp` commands to download just the matched segments.

## Concrete Examples

**Example 1: B-roll retrieval for a cooking video**

```
User: Find me a shot of steam rising from a freshly baked loaf of bread
being pulled from an oven, warm golden lighting.

Approach:
1. Parse description: core="steam rising from bread pulled from oven",
   constraints=[{type: "color", value: "warm golden lighting"}]

2. Imagination step -- prompt the LLM:
   "Imagine the full video containing this shot. It likely comes from
   a bread baking tutorial, a bakery documentary, or an artisan food vlog."
   Generated queries:
   - "homemade sourdough bread baking tutorial"
   - "artisan bakery behind the scenes"
   - "fresh bread from oven ASMR"
   - "rustic bread recipe golden crust"

3. Search YouTube Data API with each query, collect top 5 per query,
   deduplicate -> 14 unique candidate videos.

4. Download candidates, sample frames adaptively.

5. Multimodal matching identifies 3 videos with oven-pulling shots.
   Color constraint check: 2 of 3 have warm golden tones.

Output:
  Shot 1: youtube.com/watch?v=Abc123 [02:41-02:47] confidence=0.91
    Warm-lit kitchen, bread pulled from cast iron Dutch oven, visible steam
  Shot 2: youtube.com/watch?v=Def456 [05:12-05:18] confidence=0.84
    Bakery setting, golden light from oven interior, sourdough loaf
```

**Example 2: Constrained shot retrieval with temporal context**

```
User: I need a shot of a soccer player celebrating after scoring a goal,
but specifically the moment right after the ball hits the net -- not the
run-up or the team pile-on afterward.

Approach:
1. Parse: core="soccer player celebrating after scoring",
   constraints=[{type: "temporal", value: "immediately after ball hits net,
   before team celebration"}]

2. Imagination: "This shot appears in match highlights, goal compilations,
   or sports broadcast replays."
   Generated queries:
   - "best soccer goals celebration compilation 2025"
   - "football goal highlights close-up reaction"
   - "striker solo celebration after scoring"

3. Retrieve and download candidates. Sample frames.

4. Two-pass matching: first identify goal-scoring moments (ball in net),
   then check the 5-second window after for solo celebration before
   teammates arrive.

5. Temporal constraint verification: examine frames before and after
   the candidate shot to confirm it follows the goal and precedes
   the group celebration.

Output:
  Shot 1: youtube.com/watch?v=Ghi789 [01:23-01:26] confidence=0.87
    Player arms raised, sliding on knees, ball visible in net behind,
    teammates still running toward. Temporal constraint: SATISFIED
  Shot 2: youtube.com/watch?v=Jkl012 [03:45-03:48] confidence=0.79
    Close-up of striker pointing to sky, net rippling in background.
    Temporal constraint: SATISFIED
```

**Example 3: Building a retrieval pipeline in Python**

```
User: Help me build a Python script that implements the ShotFinder pipeline
to find video shots matching descriptions from a JSON file.

Approach:
1. Structure the script with three module functions matching the pipeline:
   - expand_queries(description, constraints) -> list[str]
   - retrieve_candidates(queries) -> list[VideoCandidate]
   - localize_shot(video_path, description, constraints) -> ShotMatch

2. For query expansion, call an LLM API with the imagination prompt template.

3. For retrieval, use yt-dlp's search functionality or YouTube Data API.

4. For localization, use ffmpeg for frame extraction and a vision API
   for matching.

Output: A working Python script (~200 lines) with:
  - CLI interface accepting a JSON file of shot descriptions
  - Configurable LLM backend (OpenAI, Anthropic, Google)
  - Adaptive frame sampling based on video duration
  - Structured JSON output with ranked results per description
```

## Best Practices

**Do:**
- Generate diverse search queries that target different plausible parent videos. A shot of "a cat knocking a glass off a table" could come from a funny compilation, a pet behavior video, or a home security camera clip.
- Use the two-pass frame matching strategy: coarse sampling first, then dense sampling around the best match. This balances cost against precision.
- Verify constraints independently from the core description match. A shot can be visually perfect but fail on color tone or temporal position.
- Cache downloaded videos and extracted frames when processing multiple shot descriptions against the same candidate pool.

**Avoid:**
- Extracting keywords directly from the shot description for search queries. This is the naive baseline that ShotFinder explicitly outperforms. Always use the imagination step.
- Relying on CLIP-based similarity scores for fine-grained shot matching. The paper demonstrates these correlate poorly with human judgment on nuanced visual descriptions. Use LLM-based evaluation instead.
- Sampling a fixed number of frames regardless of video duration. Short videos need fewer frames; long videos need more to avoid missing the target shot.
- Ignoring constraint verification. A retrieved shot that matches the visual description but violates a color or temporal constraint is a false positive.

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| No relevant candidates returned | Queries too specific or niche topic | Broaden imagination: generate queries for adjacent content categories. Try removing constraint terms from search queries. |
| All candidates are unavailable/restricted | Region locks, age gates, deleted videos | Retry with different search engines. Add geographic diversity to queries. Use `yt-dlp --geo-bypass`. |
| Frame matching returns low confidence for all candidates | Description is too abstract or subjective | Break the description into concrete visual elements and match each independently. Lower the confidence threshold and return partial matches with explanations. |
| Temporal localization misses the target segment | Video too long, uniform frame sampling too sparse | Increase frame count for long videos. Use a hierarchical approach: first identify the relevant chapter/segment, then densely sample within it. |
| Constraint verification fails on audio | Audio track is muted, background noise | Use a dedicated audio classification model. Fall back to metadata-based audio inference (video description, auto-captions). |
| Rate limits on search or video APIs | Too many queries in batch processing | Implement exponential backoff. Batch queries with delays. Use multiple API keys or rotate between search providers. |

## Limitations

- **Accuracy gap**: The best automated system achieves ~27% accuracy versus ~89% for humans. This pipeline is a starting point for retrieval candidates, not a replacement for human curation in production editing workflows.
- **Color and style constraints are hard**: These remain the weakest dimensions for automated systems. Expect lower precision on "warm golden lighting" or "cinematic look" compared to concrete object/action descriptions.
- **Platform dependency**: The pipeline relies on video availability on public platforms. Copyright takedowns, region restrictions, and platform API changes can break retrieval.
- **Computational cost**: Multimodal frame matching requires sending many images to a vision-language API. Budget approximately 100-200 API calls per shot description for a thorough search.
- **No real-time capability**: The download-sample-match pipeline has inherent latency. This is suited for batch editing workflows, not live retrieval.
- **Legal considerations**: Downloaded video clips may be subject to copyright. This pipeline finds candidates; usage rights must be verified separately.

## Reference

**Paper**: [ShotFinder: Imagination-Driven Open-Domain Video Shot Retrieval via Web Search](https://arxiv.org/abs/2601.23232v2) (Yu et al., 2026). Key sections: Section 3 for the three-stage pipeline architecture, Section 4 for the five constraint types and benchmark construction, Section 5 for the finding that imagination-driven query expansion outperforms direct keyword extraction by a significant margin.