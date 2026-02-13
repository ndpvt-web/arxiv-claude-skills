---
name: "puda-private-user-dataset"
description: "Build privacy-preserving personalized AI systems using Puda's multi-granularity user data architecture. Implements client-side data aggregation with three privacy levels (raw browsing history, extracted keywords, predefined category subsets) and scoped access control for third-party AI agents. Use when: 'build a privacy-preserving user profile system', 'implement tiered data sharing for personalization', 'create a browser extension that collects user data locally', 'design an agent that personalizes without leaking private data', 'add granular privacy controls to a recommendation system', 'build client-side user data management with OAuth scoping'."
---

# Puda: Private User Dataset Agent

This skill enables Claude to build user-sovereign personalized AI systems where personal data is aggregated client-side across services and shared with AI agents at user-controlled privacy granularities. The core architecture from the Puda paper separates data collection (browser extension), multi-granularity processing (Dataset Agent), and scoped access control (Access Control Agent), so that third-party AI services receive only the abstraction level the user authorizes -- from full browsing history down to predefined category labels -- while retaining 97% of personalization quality even at the most private level.

## When to Use

- When the user asks to build a personalization system that respects user privacy (e.g., recommendation engines, travel planners, content curators)
- When designing a browser extension or client-side agent that collects and processes user browsing data locally
- When implementing tiered/granular privacy controls for data sharing with external AI services
- When building an OAuth/OpenID Connect-based access control layer that scopes what user data an AI agent can retrieve
- When the user needs to extract keywords or categorize browsing history into a privacy-safe taxonomy
- When evaluating personalization quality vs. privacy trade-offs using LLM-as-a-Judge scoring

## Key Technique

Puda solves the privacy-personalization trade-off by processing raw user data into three progressively more abstract representations, each with decreasing re-identification risk:

**Level 1 -- Detailed Browsing History**: Full URLs, page titles, and content summaries. Maximum personalization, maximum privacy risk. An LLM (e.g., Gemma 3 4B) generates per-page summaries, which are then aggregated into long and short user-level summaries.

**Level 2 -- Extracted Keywords**: Per-page keywords with sentiment labels and confidence scores, aggregated across all pages. This strips away specific URLs and page content but preserves semantic signals. Keywords above a confidence threshold (e.g., >= 0.85) can actually outperform full browsing history on serendipity metrics because they surface latent interests.

**Level 3 -- Predefined Category Subsets**: User interests mapped to a hierarchical taxonomy (e.g., Google Cloud NL API's 26 first-tier / 256 second-tier / 810 third-tier categories). This deterministically eliminates the risk of leaking unexpected proper nouns or identifiable details. Despite being the most abstract, this level achieves 97.2% of Level 1's personalization performance.

The access control layer uses OAuth 2.0 with OpenID Connect Discovery to issue scoped tokens. An external AI agent requests data, the user selects which granularity level to share, and the Access Control Agent issues a token restricted to that dataset. The agent-to-agent communication uses the A2A protocol, making the system interoperable across services.

## Step-by-Step Workflow

1. **Design the data collection layer**: Implement a browser extension (or equivalent client-side collector) that captures page URLs, titles, and HTML body content. Store records locally with timestamps in a structured format (SQLite or IndexedDB). Never transmit raw data to external servers.

2. **Implement per-page processing**: For each captured page, run an LLM to generate: (a) a one-paragraph summary, (b) 5-15 keywords each tagged with a sentiment label (positive/negative/neutral) and a confidence score (0.0-1.0). Store these alongside the raw record.

3. **Build the three privacy-level datasets**: Aggregate per-page outputs into user-level representations:
   - **Level 1**: Concatenate page summaries into a long-form user browsing profile (and optionally a condensed short-form version).
   - **Level 2**: Merge all per-page keywords, preserving sentiment and scores. Optionally apply a score threshold (e.g., >= 0.85) to surface only high-confidence interests.
   - **Level 3**: Map the long summary and keywords through a predefined category taxonomy (use Google Cloud NL API categories or a custom hierarchy). Output only category labels the user matches, with no free-text content.

4. **Store a static user profile**: Separately maintain basic profile fields (name, age, location, stated preferences) that the user explicitly provides. This profile is shared alongside whichever dataset level is selected.

5. **Implement the Access Control Agent**: Build an OAuth 2.0 authorization server with OpenID Connect Discovery. Define three scopes corresponding to the three privacy levels (e.g., `puda:history`, `puda:keywords`, `puda:categories`). When an external agent requests access, present the user with a consent screen showing exactly what data will be shared.

6. **Build the data-serving API**: Expose endpoints that accept a scoped access token and return only the authorized dataset level. Use the A2A (Agent-to-Agent) protocol pattern: the requesting agent sends its query context, and your API responds with the user dataset at the authorized granularity.

7. **Integrate with the consuming AI agent**: The downstream agent (e.g., a travel planner) receives the user dataset and incorporates it into its prompt context. Structure the prompt so the agent first infers user interests from the provided data, then generates personalized output grounded in those inferences.

8. **Implement LLM-as-a-Judge evaluation**: Score outputs on three 5-point criteria: (a) Abstracted Preference Match -- does the output reflect inferred user preferences? (b) Serendipity -- does it offer novel suggestions beyond what's explicit in the data? (c) Groundedness -- is the reasoning behind recommendations valid and traceable?

9. **Validate the privacy-personalization trade-off**: Run the same queries across all three privacy levels and compare scores. Level 3 (categories) should achieve >= 95% of Level 1 (full history) on preference match, confirming the architecture works. If it doesn't, refine category granularity (move from first-tier to second or third-tier categories).

10. **Add user controls and transparency**: Build a UI where users can review their stored data at every level, delete specific entries, adjust the score threshold for keywords, and toggle individual categories on/off before sharing.

## Concrete Examples

**Example 1: Privacy-Preserving Travel Planner**

User: "Build a travel recommendation agent that personalizes suggestions based on my browsing history but lets me control how much data I share."

Approach:
1. Create a browser extension content script that captures visited travel sites, food blogs, and activity pages into IndexedDB.
2. Process each page with an LLM to extract summary + keywords (e.g., page about Kyoto temples yields keywords: `[("temples", positive, 0.92), ("Kyoto", neutral, 0.88), ("history", positive, 0.85)]`).
3. Build three dataset levels:
   - Level 1: "User frequently visits pages about Japanese temples, Okinawan beaches, budget hostels, and street food in Southeast Asia..."
   - Level 2: `[{"keyword": "temples", "sentiment": "positive", "score": 0.92}, {"keyword": "budget travel", "sentiment": "positive", "score": 0.89}, ...]`
   - Level 3: `["Travel/Asia", "Travel/Budget", "Food & Drink/Street Food", "Arts & Culture/Historical Sites"]`
4. When the travel planner agent requests data, show a consent dialog: "TravelBot wants access to your interests. Share: [Categories only] [Keywords] [Full history]"
5. The agent receives the authorized data and generates a 5-day itinerary.

Output (using Level 3 categories only):
```json
{
  "inferred_interests": ["Historical sites in Asia", "Budget accommodations", "Local street food"],
  "destinations": [
    {
      "name": "Chiang Mai, Thailand",
      "rationale": "Matches Travel/Asia + Travel/Budget + Food & Drink/Street Food",
      "pois": [
        {"name": "Doi Suthep Temple", "category": "Arts & Culture/Historical Sites"},
        {"name": "Sunday Night Market", "category": "Food & Drink/Street Food"},
        {"name": "Old City Walking Tour", "category": "Travel/Budget"}
      ]
    }
  ]
}
```

**Example 2: Implementing the Three-Level Data Pipeline**

User: "I have raw browsing logs in a database. Help me build the processing pipeline that creates all three privacy levels."

Approach:
1. Read raw logs (URL, title, HTML body, timestamp) from the database.
2. For per-page processing, call an LLM with this prompt structure:

```python
PER_PAGE_PROMPT = """Analyze this webpage and produce:
1. A 2-3 sentence summary of the page content.
2. Up to 15 keywords, each with sentiment (positive/negative/neutral) and confidence (0.0-1.0).

Page Title: {title}
Page URL: {url}
Page Content (truncated to 3000 chars): {body[:3000]}

Output as JSON:
{{"summary": "...", "keywords": [{{"term": "...", "sentiment": "...", "score": 0.0}}]}}"""
```

3. For user-level aggregation:

```python
def build_privacy_levels(user_pages: list[dict]) -> dict:
    # Level 1: Browsing History
    summaries = [p["summary"] for p in user_pages]
    level_1_long = "\n".join(summaries)
    level_1_short = llm_summarize(level_1_long, max_tokens=500)

    # Level 2: Keywords
    all_keywords = {}
    for page in user_pages:
        for kw in page["keywords"]:
            key = kw["term"].lower()
            if key not in all_keywords or kw["score"] > all_keywords[key]["score"]:
                all_keywords[key] = kw
    level_2 = sorted(all_keywords.values(), key=lambda x: -x["score"])

    # Level 3: Categories
    TAXONOMY = load_category_taxonomy()  # 810 third-tier categories
    level_3 = llm_categorize(level_1_long, level_2, TAXONOMY)
    # Returns only matching category labels, no free text

    return {"history": level_1_long, "keywords": level_2, "categories": level_3}
```

4. Store all three levels locally; serve via scoped API.

**Example 3: OAuth-Scoped Access Control for Agent Data Requests**

User: "How do I implement the access control so external AI agents can only get the data level the user approves?"

Approach:
1. Define OAuth 2.0 scopes mapping to privacy levels:

```python
SCOPES = {
    "puda:history": "Access detailed browsing history summaries",
    "puda:keywords": "Access extracted keywords with sentiment scores",
    "puda:categories": "Access predefined category labels only",
}
```

2. Implement the authorization endpoint that presents a consent screen:

```python
@app.get("/authorize")
async def authorize(client_id: str, scope: str, redirect_uri: str):
    agent_info = await lookup_agent(client_id)
    requested_level = scope  # e.g., "puda:categories"
    return render_consent_page(
        agent_name=agent_info["name"],
        agent_purpose=agent_info["description"],
        requested_scope=SCOPES[requested_level],
        data_preview=get_preview(requested_level, current_user),
    )
```

3. Issue scoped tokens and enforce at the data endpoint:

```python
@app.get("/userdata")
async def get_userdata(token: str = Depends(oauth2_scheme)):
    claims = verify_token(token)
    scope = claims["scope"]  # "puda:categories"
    if scope == "puda:history":
        return current_user.datasets["history"]
    elif scope == "puda:keywords":
        return current_user.datasets["keywords"]
    elif scope == "puda:categories":
        return current_user.datasets["categories"]
    raise HTTPException(403, "Scope not authorized")
```

## Best Practices

- **Do**: Always default to the most private level (categories) and let users opt into more detailed sharing. The 97% performance retention means categories-first is almost always sufficient.
- **Do**: Include a data preview in the consent screen so users see exactly what will be shared before authorizing.
- **Do**: Use a well-established category taxonomy (Google Cloud NL API has 810 third-tier categories) rather than inventing your own. This ensures consistency and prevents categories from becoming re-identifiable.
- **Do**: Apply keyword score thresholds (>= 0.85) when using Level 2 -- high-confidence keywords improve serendipity by filtering noise.
- **Avoid**: Sending raw HTML or full URLs to external agents, even at Level 1. Always process into summaries first.
- **Avoid**: Allowing free-text fields in Level 3 output. The entire point of predefined categories is that they deterministically prevent proper-noun leakage. If the LLM outputs a category not in the taxonomy, discard it.
- **Avoid**: Caching user datasets server-side across sessions. Regenerate from local storage on each authorization to reflect the user's latest data and deletion choices.

## Error Handling

- **LLM extraction fails for a page**: Skip the page and log it. A few missing pages won't meaningfully degrade the user profile. Retry with a shorter content truncation if the page body exceeded context limits.
- **Category mapping produces zero matches**: The user's browsing may be too narrow. Fall back to second-tier categories (256 options) instead of third-tier (810) to increase match likelihood.
- **OAuth token scope mismatch**: If an agent requests `puda:history` but the user only authorized `puda:categories`, return a 403 with a clear message indicating the authorized scope. Do not silently downgrade -- the agent needs to know what it received.
- **Stale data after user deletion**: If a user deletes browsing entries after a token was issued, the next data request should reflect the deletion. Never cache dataset snapshots beyond the current session.
- **Keyword aggregation produces thousands of entries**: Cap at the top 200 keywords by score to keep the dataset manageable for downstream agent context windows.

## Limitations

- **Client-side LLM requirement**: Per-page processing with models like Gemma 3 4B requires meaningful local compute. For devices without GPU, consider a lightweight keyword extractor (TF-IDF or KeyBERT) as a fallback, though quality will decrease.
- **Taxonomy coverage gaps**: Predefined category taxonomies may not cover niche interests (e.g., specific hobby communities). Users with very specialized browsing patterns may see reduced personalization at Level 3.
- **Single-language bias**: The paper's evaluation used Japanese personas and taxonomy. When adapting to other languages, verify that the category taxonomy has adequate localized coverage.
- **No temporal modeling**: The current architecture treats all browsing history equally. It doesn't weight recent activity more heavily, which may produce stale interest profiles for users whose preferences shift.
- **Evaluation scope**: The 97.2% figure was validated on a travel planning task with 20 personas. Other domains (e.g., medical, financial) may show different privacy-personalization curves and should be independently evaluated.

## Reference

[Puda: Private User Dataset Agent for User-Sovereign and Privacy-Preserving Personalized AI](https://arxiv.org/abs/2602.08268v2) -- Focus on Section 3 (system architecture and three privacy levels), Section 4 (travel planning evaluation), and Table 2 (quantitative comparison across granularity levels showing that predefined categories achieve 97.2% of full-history personalization).