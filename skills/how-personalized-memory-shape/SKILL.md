---
name: "how-personalized-memory-shape"
description: "Rational preference utilization for personalized LLM assistants. Implements RP-Reasoner's pragmatic reasoning to selectively integrate user memory/preferences, avoiding irrational personalization. Use when: 'build a personalized assistant with memory', 'add user preference handling to my chatbot', 'filter relevant user context for responses', 'implement memory-aware response generation', 'prevent over-personalization in my AI assistant', 'rational preference selection for user profiles'."
---

# Rational Preference Utilization for Personalized Memory Systems

This skill teaches Claude to implement RP-Reasoner, a pragmatic reasoning framework from the RPEval benchmark that treats user memory utilization as a deliberate reasoning process rather than naive context injection. The core insight: when an LLM-powered assistant has access to stored user preferences, blindly concatenating all preferences into the prompt causes **irrational personalization** -- responses that misapply irrelevant preferences, overfit to stored history, or contradict the user's actual current intent. RP-Reasoner fixes this by classifying each preference into one of three policies (Ignore, Support, Preference-Driven) using a three-module voting system before generating a response.

## When to Use

- When building a chatbot or assistant that stores and retrieves user preferences, profiles, or memory
- When designing a RAG pipeline where retrieved context includes user-specific information that may or may not be relevant
- When a user's system has memory/preference injection causing degraded response quality (over-personalization, filter bubbles)
- When implementing preference-aware response generation that must distinguish relevant from irrelevant stored context
- When building evaluation harnesses to test whether a personalized assistant correctly handles user memory
- When refactoring an existing personalized assistant that naively prepends all user memories to every prompt

## Key Technique: Pragmatic Reasoning Over Preferences

### The Problem: Irrational Personalization

Most personalized assistants operate at **Level 1 (L1)**: they directly concatenate retrieved user preferences with the current query. This causes three documented error patterns:

1. **Context mismatch** -- applying a dining preference when the user asks a coding question
2. **Preference violation** -- stored preferences override the user's explicit current instructions
3. **Information overload** -- injecting too many preferences dilutes the response focus

A better system operates at **Level 2 (L2)**: it reasons about *whether and how* each preference should influence the current response before generating it.

### The Solution: Three-Module Policy Voting (RP-Reasoner)

RP-Reasoner classifies the relationship between each stored preference and the current query into one of three policies:

- **Policy A (Ignore)**: The preference is irrelevant to the current query. Do not use it.
- **Policy B (Support)**: The preference provides useful supplementary context. Blend it with a general response.
- **Policy C (Preference-Driven)**: The preference directly addresses the user's intent. Let it drive the response.

Three independent reasoning modules each rank these policies:

1. **MLE Module** (Maximum Likelihood Estimation): Given the user's question, which policy most likely reflects the user's intent? Scores the probability that a typical user would want each policy applied.
2. **CPE-Recall Module** (Conditional Preference Estimation - Recall): Given this question, would this preference naturally come to the user's mind? Scores how strongly the preference is recalled by the query context.
3. **CPE-Intent Module** (Conditional Preference Estimation - Intent): Given this preference exists, how likely is it that the user's intent aligns with preference-driven behavior? Scores intent-preference alignment.

Each module produces a ranking (e.g., B > A > C). Rankings are aggregated by positional scoring: rank 1 gets 1 point, rank 2 gets 2 points, rank 3 gets 3 points. The policy with the lowest total score wins. Ties break by first-place count, then random selection.

## Step-by-Step Workflow

1. **Extract the user query and all stored preferences.** Separate the current user message from any retrieved memory items. Each preference should be an independent unit (e.g., "User prefers Python over JavaScript", "User is vegetarian").

2. **For each preference, run the MLE reasoning module.** Prompt the LLM: "Given the user's question `{question}`, rank the following three strategies from most to least appropriate: (A) Ignore this preference entirely, (B) Use this preference as supplementary context, (C) Let this preference drive the response." Capture the ranking as an ordered list.

3. **For each preference, run the CPE-Recall module.** Prompt the LLM: "Given the user's question `{question}`, would a user with the preference `{preference}` naturally recall this preference in this context? Score question feasibility (0-5) and recall strength (0-5), then rank policies A, B, C." Capture the ranking.

4. **For each preference, run the CPE-Intent module.** Prompt the LLM: "Given that the user holds preference `{preference}` and asks `{question}`, how likely is it that their intent is driven by this preference? Score intent probability (0-5), then rank policies A, B, C." Capture the ranking.

5. **Aggregate rankings using positional scoring.** For each preference, sum positional scores across the three modules. Select the policy with the lowest total. Break ties by counting first-place votes; if still tied, default to Policy A (Ignore) as the conservative choice.

6. **Filter preferences by policy assignment.** Discard all Policy A preferences. Partition remaining preferences into Policy B (supportive) and Policy C (driving).

7. **Construct the generation prompt with policy-aware context.** For Policy C preferences, instruct the LLM to center its response around these preferences. For Policy B preferences, instruct the LLM to incorporate them as secondary considerations. Include the original query verbatim.

8. **Generate the response using the filtered, policy-tagged context.** The generation prompt should explicitly state the role of each included preference (supporting vs. driving) so the LLM weights them appropriately.

9. **Log the policy decisions for observability.** Store which preferences were classified as A/B/C for debugging and evaluation. This enables detection of systematic misclassification over time.

## Concrete Examples

**Example 1: Irrelevant preference correctly ignored**

```
Stored preferences:
  - "User prefers spicy Sichuan cuisine"
  - "User works with TypeScript daily"

User query: "How do I set up ESLint for a TypeScript project?"

RP-Reasoner classification:
  Preference 1 ("spicy Sichuan cuisine"):
    MLE ranking:  A > B > C  (ignore -- food is irrelevant to linting)
    CPE-Recall:   A > B > C  (user would not recall food preferences here)
    CPE-Intent:   A > B > C  (intent is purely technical)
    Aggregate:    A=3, B=6, C=9 → Policy A (Ignore)

  Preference 2 ("TypeScript daily"):
    MLE ranking:  B > C > A  (relevant supplementary context)
    CPE-Recall:   C > B > A  (directly recalled by TypeScript query)
    CPE-Intent:   B > C > A  (supports but doesn't drive the linting question)
    Aggregate:    A=9, B=4, C=5 → Policy B (Support)

Generation prompt:
  "The user works with TypeScript daily (use as supplementary context).
   Answer their question: How do I set up ESLint for a TypeScript project?"

Result: Response includes TypeScript-specific ESLint config without
mentioning food preferences.
```

**Example 2: Preference correctly drives the response**

```
Stored preferences:
  - "User follows a strict vegan diet"
  - "User lives in Portland, Oregon"

User query: "What should I eat for dinner tonight?"

RP-Reasoner classification:
  Preference 1 ("strict vegan diet"):
    MLE ranking:  C > B > A  (diet directly relevant to dinner)
    CPE-Recall:   C > B > A  (user always considers diet for meals)
    CPE-Intent:   C > B > A  (intent is meal planning, diet drives it)
    Aggregate:    A=9, B=6, C=3 → Policy C (Preference-Driven)

  Preference 2 ("lives in Portland"):
    MLE ranking:  B > C > A  (location helps with restaurant suggestions)
    CPE-Recall:   B > A > C  (location is background context, not primary)
    CPE-Intent:   B > C > A  (supports local recommendations)
    Aggregate:    A=5, B=3, C=7 → Policy B (Support)

Generation prompt:
  "The user follows a strict vegan diet (center response around this).
   The user lives in Portland, Oregon (use as supplementary context).
   Answer: What should I eat for dinner tonight?"

Result: Response recommends vegan restaurants in Portland, with diet as
the primary constraint and location as a helpful filter.
```

**Example 3: Implementation in code**

```python
from enum import Enum

class Policy(Enum):
    IGNORE = "A"
    SUPPORT = "B"
    PREFERENCE_DRIVEN = "C"

def aggregate_rankings(mle_ranking: list[Policy],
                       cpe_recall_ranking: list[Policy],
                       cpe_intent_ranking: list[Policy]) -> Policy:
    """Aggregate three module rankings into a final policy decision."""
    scores = {Policy.IGNORE: 0, Policy.SUPPORT: 0, Policy.PREFERENCE_DRIVEN: 0}
    first_place_counts = {Policy.IGNORE: 0, Policy.SUPPORT: 0, Policy.PREFERENCE_DRIVEN: 0}

    for ranking in [mle_ranking, cpe_recall_ranking, cpe_intent_ranking]:
        for position, policy in enumerate(ranking):
            scores[policy] += position + 1  # 1-indexed positional scoring
        first_place_counts[ranking[0]] += 1

    # Sort by lowest score, then by most first-place votes
    sorted_policies = sorted(
        scores.keys(),
        key=lambda p: (scores[p], -first_place_counts[p])
    )
    return sorted_policies[0]

def classify_preference(preference: str, query: str, llm) -> Policy:
    """Run three reasoning modules and aggregate to a policy decision."""
    mle = llm.rank_policies(
        f"Given the question '{query}', rank strategies A (ignore preference), "
        f"B (use as supplement), C (let it drive response) for preference: '{preference}'"
    )
    cpe_recall = llm.rank_policies(
        f"Would a user with preference '{preference}' naturally recall it "
        f"when asking '{query}'? Rank A, B, C by appropriateness."
    )
    cpe_intent = llm.rank_policies(
        f"Given preference '{preference}' exists, how likely does '{query}' "
        f"reflect preference-driven intent? Rank A, B, C."
    )
    return aggregate_rankings(mle, cpe_recall, cpe_intent)

def build_generation_prompt(query: str, preferences: list[dict]) -> str:
    """Construct a policy-aware prompt from classified preferences."""
    driving = [p["text"] for p in preferences if p["policy"] == Policy.PREFERENCE_DRIVEN]
    supporting = [p["text"] for p in preferences if p["policy"] == Policy.SUPPORT]

    parts = []
    if driving:
        parts.append(f"Center your response around: {'; '.join(driving)}")
    if supporting:
        parts.append(f"Consider as supplementary context: {'; '.join(supporting)}")
    parts.append(f"User question: {query}")
    return "\n".join(parts)
```

## Best Practices

- **Do:** Run all three reasoning modules (MLE, CPE-Recall, CPE-Intent) independently before aggregating. Single-module classification has significantly higher error rates per the RPEval benchmark.
- **Do:** Default to Policy A (Ignore) when rankings are ambiguous or tied. Under-personalization is less harmful than irrational over-personalization.
- **Do:** Treat each preference as an independent unit for classification. A user may have 5 stored preferences where only 1 is relevant to the current query.
- **Do:** Log policy decisions separately from responses. This creates an audit trail for debugging personalization failures.
- **Avoid:** Concatenating all stored preferences into the system prompt without filtering. This is the L1 pattern that causes the documented error categories.
- **Avoid:** Using only semantic similarity (embedding distance) to filter preferences. Similarity captures topical overlap but misses pragmatic relevance -- a preference can be topically related but strategically wrong to apply.

## Error Handling

- **LLM returns unparseable ranking:** Default to the conservative ranking A > B > C (Ignore first). Log the failure for review.
- **No preferences match Policy B or C:** Generate a standard non-personalized response. This is the correct behavior when no stored preferences are relevant.
- **All preferences match Policy C:** This is suspicious -- verify the classifications. In practice, rarely should more than 1-2 preferences drive a single response. Consider re-running with stricter prompts if this occurs frequently.
- **Latency from three LLM calls per preference:** Run the three modules in parallel for each preference, and classify multiple preferences concurrently. For production systems with many preferences, pre-filter by embedding similarity before running the full pragmatic reasoning pipeline.
- **Timeout or API failure on one module:** Aggregate with the available 2 module rankings. Two-module aggregation is still significantly better than no classification.

## Limitations

- **Computational cost:** Three LLM calls per preference per query is expensive. For systems with hundreds of stored preferences, a two-stage pipeline (embedding pre-filter + pragmatic reasoning on top-k) is necessary.
- **Prompt sensitivity:** The three-module rankings depend on prompt wording. The specific prompts from RPEval are tuned for Chinese-language evaluation; adaptation to other languages requires prompt engineering and validation.
- **Binary preference assumption:** The A/B/C classification assumes preferences are atomic statements. Complex, multi-faceted preferences (e.g., "User likes spicy food but avoids shellfish") may need decomposition before classification.
- **No temporal reasoning:** The framework does not account for preference decay or recency. A preference stored 2 years ago is treated identically to one stored yesterday. Temporal weighting must be added separately.
- **Evaluation gap:** RPEval focuses on single-turn interactions. Multi-turn conversations where preferences evolve within a session are not covered by this framework.

## Reference

**Paper:** [How Does Personalized Memory Shape LLM Behavior? Benchmarking Rational Preference Utilization in Personalized Assistants](https://arxiv.org/abs/2601.16621v1) -- Look for Section 4 (RP-Reasoner) detailing the three-module aggregation mechanism and Section 3 (RPEval) for the A/B/C policy taxonomy and error pattern analysis. Code and benchmark data at [github.com/XueyangFeng/RPEval](https://github.com/XueyangFeng/RPEval).