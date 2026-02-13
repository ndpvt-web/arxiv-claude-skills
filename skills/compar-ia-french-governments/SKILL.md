---
name: "compar-ia-french-governments"
description: "Build multilingual LLM evaluation arenas and preference data collection pipelines modeled on France's compar:IA platform. Collects human preference pairs for RLHF/DPO training in non-English languages using blind pairwise comparison, Bradley-Terry ranking, and privacy-preserving filtering. Trigger phrases: 'build an LLM arena', 'collect preference data for DPO', 'create a chatbot comparison platform', 'multilingual RLHF data pipeline', 'pairwise LLM evaluation system', 'French language model leaderboard'."
---

# compar:IA — Multilingual LLM Arena & Preference Data Collection Pipeline

This skill enables Claude to design and implement LLM evaluation arenas that collect human preference data for non-English languages, following the architecture pioneered by the French government's compar:IA platform. The core technique is a blind pairwise comparison interface backed by a FastAPI + SvelteKit stack, where users submit unconstrained prompts, receive anonymized side-by-side model responses, and cast preference votes that feed directly into RLHF/DPO training pipelines. The platform uses Bradley-Terry ranking models to aggregate pairwise judgments into a leaderboard, and applies conservative full-conversation exclusion (rather than span-level redaction) for privacy filtering.

## When to Use

- When the user asks to build an LLM arena or chatbot comparison platform for any language
- When designing a pipeline to collect human preference data for DPO or RLHF fine-tuning
- When the user needs a Bradley-Terry ranking system for pairwise model evaluation
- When building a privacy-preserving data collection platform that filters PII at the conversation level
- When the user wants to create a multilingual model leaderboard from crowdsourced votes
- When deploying a blind A/B testing system for comparing LLM outputs
- When building infrastructure to route prompts to multiple inference providers (OpenRouter, HuggingFace Inference) simultaneously
- When the user needs to measure and display environmental impact of LLM inference

## Key Technique

**Blind Pairwise Comparison with Two-Level Feedback.** The compar:IA approach collects preference data through a three-step flow: (1) the user submits an unconstrained free-form prompt, (2) two randomly selected anonymous models generate side-by-side responses, and (3) the user provides feedback at two granularities — message-level reactions (per-turn thumbs up/down) and conversation-level votes (preferred model A, model B, or tie). Only after voting does the platform reveal model identities and metadata. This dual-granularity feedback yields richer training signal than single-vote systems: conversation-level votes produce standard DPO preference pairs, while message-level reactions enable turn-level reward modeling.

**Bradley-Terry Ranking from Noisy Crowdsourced Data.** Rather than raw win-rate tallies, the leaderboard uses the Bradley-Terry statistical model to aggregate pairwise votes into consistent global rankings. This handles the transitivity problem (model A beats B, B beats C, but C beats A) and accounts for uneven matchup frequencies. The method treats each vote as evidence for a latent "strength" parameter per model, then fits via maximum likelihood. This is the same approach used by Chatbot Arena (LMSYS) but applied here to French-language evaluation with 250,000+ votes across 104 models.

**Conservative Privacy Filtering via Full-Conversation Exclusion.** Instead of attempting span-level PII anonymization (which risks incomplete masking and semantic distortion), compar:IA uses an LLM-based classifier to flag conversations likely containing personal data, then excludes the entire conversation and all associated votes. This removes approximately 5% of data but avoids the residual re-identification risks inherent in token-level redaction. This is a key architectural decision: accept modest data loss for strong privacy guarantees.

## Step-by-Step Workflow

### 1. Define the Data Schema

Create three complementary datasets with clear separation of concerns:

```python
# conversations schema
conversation = {
    "conversation_id": "uuid",
    "timestamp": "ISO-8601",
    "language": "fr",  # detected via langdetect or fasttext
    "turns": [
        {
            "role": "user",
            "content": "Explique-moi la photosynthèse",
            "turn_index": 0
        },
        {
            "role": "assistant_a",
            "content": "La photosynthèse est le processus...",
            "model_id": "hidden_until_vote",
            "turn_index": 1
        },
        {
            "role": "assistant_b",
            "content": "C'est un mécanisme biologique...",
            "model_id": "hidden_until_vote",
            "turn_index": 1
        }
    ]
}

# votes schema (conversation-level preference)
vote = {
    "vote_id": "uuid",
    "conversation_id": "uuid",
    "winner": "assistant_a" | "assistant_b" | "tie",
    "timestamp": "ISO-8601"
}

# reactions schema (message-level feedback)
reaction = {
    "reaction_id": "uuid",
    "conversation_id": "uuid",
    "turn_index": 1,
    "target": "assistant_a" | "assistant_b",
    "value": "positive" | "negative",
    "timestamp": "ISO-8601"
}
```

### 2. Build the Backend API (FastAPI)

Implement these core endpoints:
- `POST /conversations` — accepts a user prompt, selects two models via weighted random sampling, fans out inference requests in parallel, returns anonymized responses
- `POST /conversations/{id}/votes` — records conversation-level preference (A, B, or tie) and reveals model identities
- `POST /conversations/{id}/reactions` — records message-level thumbs up/down per turn
- `GET /leaderboard` — returns current Bradley-Terry rankings

Route inference through a provider abstraction layer that supports OpenRouter, HuggingFace Inference Providers, and direct API calls, so models can be added or swapped without code changes.

### 3. Implement Blind Comparison Frontend (SvelteKit or React)

Build a split-pane chat interface where:
- Left and right panels show "Model A" and "Model B" with no identifying information
- Both panels stream responses simultaneously from the same user prompt
- Multi-turn conversation is supported — users can follow up before voting
- Vote buttons (A wins / B wins / Tie) appear after at least one exchange
- After voting, a reveal overlay shows model names, parameter counts, and inference cost

### 4. Implement Model Selection and Pairing Strategy

```python
import random

def select_model_pair(models: list, elo_ratings: dict) -> tuple:
    """Select two models for comparison, favoring similar-strength pairings
    to maximize information gain for Bradley-Terry fitting."""
    # Weight pairings by proximity in current ELO ratings
    # to get more discriminative comparisons
    weights = []
    pairs = []
    for i, m1 in enumerate(models):
        for m2 in models[i+1:]:
            diff = abs(elo_ratings.get(m1, 1200) - elo_ratings.get(m2, 1200))
            weights.append(1.0 / (1.0 + diff / 400.0))
            pairs.append((m1, m2))
    chosen = random.choices(pairs, weights=weights, k=1)[0]
    # Randomize left/right assignment to prevent position bias
    return chosen if random.random() > 0.5 else (chosen[1], chosen[0])
```

### 5. Implement Bradley-Terry Ranking

```python
import numpy as np
from scipy.optimize import minimize

def fit_bradley_terry(matchups: list[dict], model_names: list[str]) -> dict:
    """Fit Bradley-Terry model from pairwise votes.
    matchups: [{"winner": "modelA", "loser": "modelB"}, ...]
    Returns dict of model_name -> strength score.
    """
    n = len(model_names)
    idx = {name: i for i, name in enumerate(model_names)}

    def neg_log_likelihood(params):
        nll = 0.0
        for m in matchups:
            wi = idx[m["winner"]]
            li = idx[m["loser"]]
            nll -= params[wi] - np.logaddexp(params[wi], params[li])
        # L2 regularization to prevent divergence
        nll += 0.01 * np.sum(params ** 2)
        return nll

    result = minimize(neg_log_likelihood, np.zeros(n), method="L-BFGS-B")
    strengths = result.x
    # Convert to ELO-scale ratings (centered at 1200)
    elo = 1200 + 400 * (strengths - strengths.mean())
    return {name: float(elo[i]) for i, name in enumerate(model_names)}
```

### 6. Build the Privacy Filtering Pipeline

Run an LLM-based PII classifier on every conversation before dataset export. Use full-conversation exclusion, not token-level redaction:

```python
async def filter_conversation(conv: dict, classifier_model: str) -> bool:
    """Returns True if conversation is safe to publish, False if it
    likely contains PII and should be excluded entirely."""
    full_text = " ".join(t["content"] for t in conv["turns"])
    prompt = (
        "Does the following conversation contain personal information "
        "(names, addresses, phone numbers, emails, ID numbers, medical "
        "details, or any data that could identify a real person)? "
        "Answer ONLY 'yes' or 'no'.\n\n" + full_text
    )
    response = await call_llm(classifier_model, prompt)
    return response.strip().lower() == "no"
```

Expect approximately 5% exclusion rate. Provide a public reporting form so users can flag conversations that slipped through or were incorrectly removed.

### 7. Add Language Detection and Categorization

Classify each prompt by language (using fasttext `lid.176.bin`) and topic category. The compar:IA taxonomy uses 15 categories: Natural Science & Technology (17.7%), Education (14.3%), Business & Economics (10.1%), and others. Apply classification post-hoc for dataset analysis — do not constrain user input.

### 8. Integrate Environmental Impact Tracking

Use the `ecologits` Python library to estimate energy consumption per inference call based on token count, model architecture, and GPU manufacturing amortization. Display per-conversation carbon estimates to users after they vote.

### 9. Export DPO-Ready Training Data

Convert conversation-level votes into preference pairs formatted for DPO training:

```python
def export_dpo_pairs(conversations: list, votes: list) -> list[dict]:
    """Convert arena votes to DPO training format."""
    pairs = []
    for vote in votes:
        if vote["winner"] == "tie":
            continue  # ties are excluded from DPO pairs
        conv = get_conversation(vote["conversation_id"], conversations)
        prompt = extract_user_turns(conv)
        chosen = extract_assistant_turns(conv, vote["winner"])
        rejected = extract_assistant_turns(
            conv, "assistant_b" if vote["winner"] == "assistant_a" else "assistant_a"
        )
        pairs.append({
            "prompt": prompt,
            "chosen": chosen,
            "rejected": rejected,
            "language": conv["language"]
        })
    return pairs
```

### 10. Publish Datasets Under Open License

Release three dataset splits (conversations, votes, reactions) on HuggingFace with Etalab 2.0 or CC-BY-4.0 licensing. Gate raw data for research-only access. Add a restriction that proprietary model responses may be used for analysis and evaluation but not for training.

## Examples

**Example 1: Building a French LLM Arena**

User: "I want to build a platform like Chatbot Arena but focused on French. Help me set up the backend."

Approach:
1. Scaffold a FastAPI project with endpoints for `/conversations`, `/votes`, `/reactions`, and `/leaderboard`
2. Create a provider abstraction layer supporting OpenRouter and HuggingFace Inference APIs
3. Implement the blind model pairing logic with position randomization
4. Set up a PostgreSQL schema with the three-table design (conversations, votes, reactions)
5. Add the Bradley-Terry ranking computation as a background job that recomputes after every 100 new votes
6. Implement the PII filtering pipeline as an async post-processing step before dataset export

Output:
```
project/
  app/
    main.py              # FastAPI app with CORS, routes
    routers/
      conversations.py   # prompt submission, parallel inference
      votes.py           # preference recording, model reveal
      leaderboard.py     # Bradley-Terry rankings
    services/
      inference.py       # provider abstraction (OpenRouter, HF, direct)
      pairing.py         # model selection with ELO-proximity weighting
      ranking.py         # Bradley-Terry MLE fitting
      privacy.py         # LLM-based PII filter, full-conversation exclusion
    models/
      schemas.py         # Pydantic models for all three datasets
    db/
      migrations/        # Alembic migrations for PostgreSQL
```

**Example 2: Converting Arena Votes to DPO Training Data**

User: "I have 50,000 pairwise preference votes from my LLM arena. How do I convert them into a DPO dataset?"

Approach:
1. Load conversations and votes datasets, join on `conversation_id`
2. Filter out ties (they carry no preference signal for DPO)
3. For each vote with a clear winner, extract the user prompt, the chosen response (winner), and the rejected response (loser)
4. Detect and tag language per conversation using fasttext
5. Apply the PII filter — exclude any conversation flagged as containing personal data
6. Export in the standard DPO format: `{"prompt": ..., "chosen": ..., "rejected": ...}`

Output:
```json
{
  "prompt": "Quels sont les avantages et inconvénients du nucléaire en France ?",
  "chosen": "L'énergie nucléaire représente environ 70% de la production électrique française. Parmi les avantages : faibles émissions de CO2, indépendance énergétique, coût de production stable...",
  "rejected": "Le nucléaire c'est bien parce que ça produit beaucoup d'énergie. Les inconvénients c'est les déchets.",
  "language": "fr"
}
```

**Example 3: Adding a New Language to an Existing Arena**

User: "Our arena works for French. We want to add Swedish and Lithuanian support as compar:IA did with their European expansion."

Approach:
1. Add language detection at the prompt ingestion step — use fasttext `lid.176.bin` to auto-tag each conversation
2. Configure inference providers that support the target languages — verify each model's training data coverage
3. Adjust the PII classifier prompt to handle Swedish/Lithuanian PII patterns (personnummer, asmens kodas)
4. Create per-language leaderboard views by partitioning Bradley-Terry fitting by detected language
5. Update the topic categorization model or prompts to handle multilingual input
6. Set up separate dataset export splits per language

Output: The leaderboard now shows separate rankings per language, and the exported HuggingFace datasets include a `language` column for filtering.

## Best Practices

- **Do:** Randomize left/right model assignment on every comparison to prevent position bias. compar:IA found this is a significant confound if not addressed.
- **Do:** Use full-conversation exclusion for PII rather than token-level redaction. The 5% data loss is worth the privacy guarantee. Span-level anonymization leaves residual re-identification risk.
- **Do:** Support multi-turn conversations before voting. Single-turn comparisons miss how models handle context, follow-ups, and clarifications.
- **Do:** Record both conversation-level votes AND message-level reactions. Conversation votes give DPO pairs; message reactions enable turn-level reward models.
- **Avoid:** Constraining prompts with pre-set categories or templates. compar:IA found that fewer than 6% of users used suggested prompts — unconstrained input yields more realistic, diverse data.
- **Avoid:** Using raw win rates for leaderboards. Bradley-Terry properly handles transitive inconsistencies and uneven matchup frequencies. Raw percentages mislead when some models face tougher opponents.
- **Avoid:** Assuming API providers serve the exact model advertised. Quantization, preprocessing, and routing logic on the provider side can alter outputs. Log provider metadata and flag known discrepancies.

## Error Handling

- **Inference timeout or failure:** If one model fails to respond, do not show the comparison. Return the user to the prompt screen with an apology. Never show a one-sided comparison — it biases against the failed model.
- **Inconsistent system prompts:** Open-weight models via third-party providers may lack system prompts while proprietary APIs inject them. Standardize a minimal system prompt across all models or document the inconsistency in dataset metadata.
- **PII filter false positives:** Some legitimate conversations about public figures or fictional characters may trigger the PII classifier. Provide a manual review queue and a public reporting form for users to contest exclusions.
- **Language detection errors:** Short prompts (under 10 words) cause unreliable language detection. For prompts below a character threshold, either skip language tagging or use a secondary classifier.
- **Vote spam or gaming:** Monitor for patterns — same IP casting rapid votes, always choosing the same side. Apply rate limiting and flag anomalous voting patterns for manual review.

## Limitations

- **Self-selection bias:** Arena users skew toward digitally literate, tech-savvy populations. The resulting preference data may not represent general population preferences. This cannot be fully corrected post-hoc.
- **Verbosity and style bias:** Human evaluators systematically prefer longer, more confident, better-formatted responses regardless of factual accuracy. The collected preferences encode these biases, which then propagate into DPO/RLHF-trained models.
- **Proprietary model licensing:** Responses from proprietary models (GPT-4, Claude, Gemini) cannot be used for training due to terms of service — only for evaluation and analysis. This limits the usable preference pairs for fine-tuning.
- **Pairwise evaluation hides absolute quality:** A model that wins every comparison might still produce mediocre outputs if all models are weak. Bradley-Terry measures relative strength, not absolute capability.
- **Environmental cost estimates are approximate:** For proprietary models, architecture and parameter count are estimated from public information. Actual energy consumption may differ significantly.

## Reference

**Paper:** [compar:IA: The French Government's LLM arena to collect French-language human prompts and preference data](https://arxiv.org/abs/2602.06669v1) — Termignon et al., 2026. Focus on Section 3 (platform architecture and data schema), Section 4 (Bradley-Terry ranking methodology), and Section 5 (privacy filtering pipeline) for implementation details.