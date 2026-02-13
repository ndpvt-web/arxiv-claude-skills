---
name: "steereval-framework-evaluating-steerability"
description: "Evaluate and improve the steerability of natural-language-profile-based recommender systems using the SteerEval framework. Build evaluation pipelines that measure whether recommendation engines actually respond to user steering commands (preference edits). Use when: 'evaluate my recommender's steerability', 'test if profile edits change recommendations', 'build a steerable recommendation system', 'measure how well my rec system follows user preferences', 'benchmark natural language profile steering', 'audit recommendation controllability'."
---

SteerEval is a framework for systematically evaluating whether natural-language-profile-based recommender systems actually respond to user steering commands. It introduces an AUC-based steerability metric (Delta-AUC_t) that measures how item rankings shift when a user's natural-language profile is edited to increase or decrease preference for specific topics. This skill enables Claude to build evaluation pipelines, design steering interventions, compute steerability metrics, and diagnose why a recommender fails to steer on certain topic types -- particularly nuanced attributes like content warnings that go beyond simple genre categories.

## When to Use

- When building a recommender system that uses natural-language user profiles and you need to verify users can actually control their recommendations by editing their profile
- When evaluating whether an LLM-based recommendation engine responds to preference changes (e.g., "stop recommending horror" or "show more indie films")
- When designing A/B tests or offline benchmarks to compare steering intervention methods (template append vs. LLM rewrite vs. full profile regeneration)
- When diagnosing why a recommender handles genre-level steering well but fails on nuanced attributes like trigger warnings or content sensitivity tags
- When measuring the accuracy-steerability tradeoff -- ensuring steering doesn't destroy overall recommendation quality
- When auditing a content-based or LLM-scoring recommender for controllability compliance

## Key Technique

SteerEval measures steerability by comparing item rankings before and after a steering intervention is applied to a user's natural-language profile. The core metric, **Delta-AUC_t**, computes the change in a tag-specific AUC score: for a given topic tag t, AUC_t = 1.0 means all items tagged with t rank above all items without t; AUC_t = 0.5 is random; AUC_t = 0.0 means all tagged items rank last. A successful "increase" steering yields positive Delta-AUC_t, while a successful "decrease" yields negative Delta-AUC_t. This is computed over a balanced ranking pool (e.g., 50 tag-relevant + 50 irrelevant items per evaluation scenario).

The framework tests three intervention strategies of increasing complexity: (1) **template appending** -- mechanically adding a sentence like "The user does not want to see Mystery movies" to the profile; (2) **LLM-generated appending** -- using an LLM to write a natural-sounding sentence expressing the preference shift; (3) **full profile rewrite** -- having an LLM regenerate the entire profile incorporating the new preference while preserving existing tastes. Full rewrites produce the strongest steering effects but are the most expensive and risk altering unrelated preferences.

A critical finding is the **genre vs. nuanced-attribute gap**: genre tags steer effectively (Delta-AUC_t ~0.11 for increases) while trigger-warning tags steer poorly (~0.03) because LLM safety guardrails reject or misinterpret sensitive topics during profile generation. The framework also introduces **Rank_set**, a metric that measures whether the ground-truth next item's position within its relevance subset is preserved after steering, capturing accuracy loss orthogonal to the steered topic.

## Step-by-Step Workflow

1. **Construct natural-language user profiles** from interaction history. For each user, collect their rated/watched items with metadata (descriptions, genres, tags). Prompt an LLM to generate a paragraph-length profile (5-6 sentences summarizing taste in tone, themes, and style) or a sentence-length profile (20-35 words). Do NOT include raw tag names or metadata fields -- the profile should be inferred preferences in natural language.

2. **Define the tag taxonomy for evaluation.** Collect all attributes you want to test steerability on. Separate them into coarse categories (genres, broad topics) and fine-grained categories (content warnings, niche themes, sensitive topics). For movies, use genre tags from the dataset plus trigger-warning tags from sources like "Does The Dog Die." Filter out tags that the LLM refuses to process due to safety guardrails.

3. **Build balanced ranking pools per user per tag.** For each (user, tag) pair, sample a pool of items: 50% tagged with the target attribute, 50% without it. Include exactly one ground-truth "next item" the user actually consumed, to enable accuracy measurement. Ensure the pool is large enough for meaningful AUC computation (100 items is the paper's standard).

4. **Generate baseline rankings.** Score every item in each pool against the user's original (unsteered) profile. Use either embedding-based cosine similarity (e.g., `mxbai-embed-large-v1` embeddings of profile vs. item description) or LLM-based scoring (prompt the LLM to rate each item 0.0-5.0 given the profile). Record AUC_t for each (user, tag) pair.

5. **Apply steering interventions.** For each (user, tag, direction) triple -- where direction is "increase" or "decrease" -- generate three steered profiles using template append, LLM-generated append, and full profile rewrite. Template example for decrease: append `"The user does not want to see [Tag] content."` LLM-rewrite example: prompt the model to regenerate the profile with a strong aversion to [Tag] while preserving other preferences.

6. **Re-rank items with steered profiles.** Using each steered profile, re-score all items in the pool with the same scoring method used in step 4. Record the new AUC_t values.

7. **Compute Delta-AUC_t.** For each scenario: `Delta_AUC_t = AUC_t(steered) - AUC_t(original)`. For "increase" tasks, positive values indicate successful steering. For "decrease" tasks, negative values indicate success. Aggregate across users and tags to get mean steerability per tag category and intervention method.

8. **Compute Rank_set for accuracy preservation.** For each scenario, find the ground-truth next item. Compute its rank within its tag-relevance subset (tagged items if the next item has the tag, untagged items otherwise). A stable Rank_set across original and steered profiles means steering did not degrade general recommendation quality.

9. **Diagnose steering failures.** Segment Delta-AUC_t by tag category (genre vs. trigger warnings), intervention method, and scoring approach. Identify patterns: embedding methods steer well for increases but poorly for decreases; LLM scoring handles decreases better; safety guardrails cause failures on sensitive topics. Test with oracle metadata (explicit tag relevance) to determine if failures are due to semantic understanding or ranking mechanics.

10. **Report results and iterate.** Present a steerability matrix: rows = tags, columns = intervention methods, cells = mean Delta-AUC_t. Highlight the genre-vs-nuance gap. Recommend the intervention method with the best steerability-accuracy tradeoff for each tag category.

## Concrete Examples

**Example 1: Evaluating a movie recommender's genre steerability**

```
User: I have an LLM-based movie recommender that generates natural-language
user profiles from watch history. I want to test if editing a profile to say
"avoid horror" actually removes horror movies from the top recommendations.

Approach:
1. Select 10 users with 100+ ratings each. Generate paragraph profiles from
   their first 50 rated movies using an LLM prompt like:
   "Based on these movies and ratings, describe this user's taste in 5-6
   sentences. Focus on themes, tone, and style. Do not mention genre labels."

2. For each user, build a ranking pool of 100 movies: 50 tagged "Horror",
   50 not tagged "Horror". Include 1 ground-truth next-watched movie.

3. Score all 100 movies against the original profile using cosine similarity
   of sentence embeddings. Compute AUC_horror (baseline).

4. Apply three steering interventions for direction="decrease":
   - Template: append "The user does not want to see Horror movies."
   - LLM-append: prompt LLM to write a natural sentence expressing horror aversion
   - Full rewrite: regenerate the entire profile with horror aversion baked in

5. Re-score all 100 movies with each steered profile. Compute Delta-AUC_horror.

Output:
| Intervention     | Mean Delta-AUC_horror | Success Rate |
|------------------|-----------------------|--------------|
| Template append  | -0.045                | 7/10 users   |
| LLM append       | -0.052                | 8/10 users   |
| Full rewrite     | -0.061                | 9/10 users   |

Interpretation: Full rewrite is most effective. Two users show near-zero
Delta-AUC because their original profiles already had low horror affinity
(floor effect). The LLM scoring method should be tested next, as embedding
similarity struggles with "decrease" steering.
```

**Example 2: Diagnosing trigger-warning steering failures**

```
User: My recommender steers well on genres but users report that editing
their profile to avoid "animal death" content has no effect. How do I
diagnose this?

Approach:
1. Collect items tagged with "animal death" from a trigger-warning database
   (e.g., DoesTheDogDie). Build balanced ranking pools as before.

2. Generate steered profiles using all three methods. Compare Delta-AUC for
   "animal death" vs. a genre tag like "Comedy" as a control.

3. Inspect the steered profiles manually. Check if the LLM's safety
   guardrails sanitized or refused the trigger-warning language during
   profile generation.

4. Test with oracle metadata: instead of relying on the LLM to understand
   "animal death" from item descriptions, explicitly tag items and provide
   that signal. Re-compute Delta-AUC.

Output:
| Tag           | Method       | Delta-AUC (natural) | Delta-AUC (oracle) |
|---------------|--------------|---------------------|--------------------|
| Comedy        | Full rewrite | +0.110              | +0.115             |
| Animal death  | Full rewrite | -0.028              | -0.121             |

Diagnosis: The oracle metadata dramatically improves trigger-warning
steerability. The bottleneck is semantic: the LLM cannot reliably detect
"animal death" themes from movie descriptions alone. Solutions:
- Enrich item metadata with explicit trigger-warning annotations
- Fine-tune the embedding model on content-warning datasets
- Use a two-stage approach: first classify items for trigger tags, then steer
```

**Example 3: Building a steerability benchmark from scratch**

```
User: I'm building a book recommender with natural-language profiles.
Help me set up a SteerEval-style benchmark.

Approach:
1. Define tag taxonomy:
   - Coarse: Fiction, Non-Fiction, Romance, Sci-Fi, Mystery, etc. (15 tags)
   - Fine-grained: "unreliable narrator", "slow burn", "found family",
     "graphic violence", "addiction themes" (30+ tags from Goodreads shelves
     or StoryGraph content warnings)

2. Sample 10-20 users with rich reading histories (50+ books).

3. For each user, generate profiles:
   ```python
   profile_prompt = f"""Based on these books and ratings:
   {format_reading_history(user)}

   Describe this reader's taste in 5-6 sentences. Focus on what themes,
   writing styles, pacing, and emotional tones they gravitate toward.
   Do not list genre names or book titles."""
   ```

4. For each (user, tag) pair, build a pool of 100 books: 50 with tag, 50
   without. Use Goodreads or Open Library metadata for tag assignment.

5. Implement scoring: embed profiles and book descriptions with a sentence
   transformer; compute cosine similarity. Also implement LLM scoring with
   a 0-5 rating prompt.

6. Run all intervention types x directions x tags x users.
   Total scenarios: 20 users x 45 tags x 2 directions x 3 methods = 5,400.

7. Compute Delta-AUC_t and Rank_set for every scenario. Report the
   steerability matrix segmented by tag granularity.
```

## Best Practices

- **Do:** Use balanced ranking pools (equal tag-relevant and irrelevant items) to ensure AUC_t is meaningful and not dominated by class imbalance.
- **Do:** Test multiple intervention methods per tag -- template appending is cheapest but full rewrite is most effective. Use template appending as a lower-bound baseline.
- **Do:** Always compute accuracy preservation (Rank_set) alongside steerability. A recommender that steers perfectly but destroys quality is useless.
- **Do:** Segment results by tag granularity. Genre-level steerability does not predict content-warning-level steerability.
- **Avoid:** Using raw tag names in profile generation prompts. Profiles should describe inferred taste, not repeat metadata. Leaking tag vocabulary into profiles inflates steerability scores artificially.
- **Avoid:** Ignoring LLM safety guardrail effects. If your tags include sensitive topics (violence, self-harm, substance abuse), the LLM may refuse or sanitize steering commands. Test for this explicitly and log refusal rates.
- **Avoid:** Evaluating only "increase" steering. Decrease steering (removing unwanted content) is harder for embedding-based methods and is the use case users care about most for content warnings.

## Error Handling

- **LLM refusals on sensitive tags:** If the LLM refuses to generate profiles mentioning trigger warnings, log the refusal, exclude that tag from automated evaluation, and test it separately with explicit metadata injection (oracle mode) to isolate the guardrail issue from the ranking issue.
- **Empty or degenerate ranking pools:** If fewer than 10 items match a tag in your catalog, the AUC computation becomes unreliable. Set a minimum pool size threshold (e.g., 25 tagged items) and skip tags below it.
- **Ceiling/floor effects:** If a user's baseline AUC_t is already near 1.0 (strong affinity) or 0.0 (strong aversion), Delta-AUC_t will be compressed. Report baseline AUC distributions alongside Delta values, and consider filtering out users with extreme baselines for that tag.
- **Scoring method inconsistency:** Embedding similarity and LLM scoring can disagree. If results diverge, report both and use the gap as a diagnostic signal -- it indicates the profile edit is semantically visible to one method but not the other.
- **Profile drift during rewrite:** Full profile rewrites can inadvertently change preferences unrelated to the target tag. Validate by computing Delta-AUC on unrelated tags after steering; it should be near zero.

## Limitations

- The framework assumes access to item-level tag annotations. If your catalog lacks structured tags, you must first build a tagging system (manual or automated) before running SteerEval.
- Evaluation is offline only. SteerEval does not measure real-time user satisfaction or whether users can formulate effective steering commands themselves.
- The balanced 50/50 ranking pool is artificial. Real recommendation lists have skewed tag distributions, and steerability in balanced pools may not transfer to production ranking.
- LLM-based scoring is expensive at scale: scoring 100 items per scenario with an LLM, across thousands of scenarios, requires significant compute. Embedding methods are faster but less effective for decrease steering.
- The framework was validated on movies (MovieLens + TMDb). Transferring to other domains (music, books, products) requires new tag taxonomies and may surface different failure modes.

## Reference

**SteerEval: A Framework for Evaluating Steerability with Natural Language Profiles for Recommendation** -- Zhou et al., 2026. [arXiv:2601.21105v1](https://arxiv.org/abs/2601.21105v1). Key sections: Section 3 (framework design and Delta-AUC_t metric), Section 4 (intervention methods), Section 5 (genre vs. trigger-warning steerability gap and oracle metadata experiments).