---
name: "improve-systems-user-logs"
description: "Implement the UNO (User log-driveN Optimization) framework to improve LLM-powered systems by distilling user interaction logs into structured rules, preference pairs, and adaptive experience modules. Use when asked to 'learn from user logs', 'improve system from feedback', 'optimize responses using interaction history', 'build a feedback loop for LLM systems', 'reduce noise in user feedback data', or 'create a continual learning pipeline from deployment logs'."
---

# Improving LLM Systems with User Logs (UNO Framework)

This skill enables Claude to implement the UNO (User log-driveN Optimization) framework for systematically improving LLM-powered systems using real user interaction logs. UNO converts unstructured, noisy user feedback into semi-structured rules and preference pairs, clusters them by intent and feedback similarity, quantifies the cognitive gap between what the model already knows and what the logs teach, then builds two types of experience modules -- direct parameter adapters for clear feedback and critique-based refiners for ambiguous signals. This approach significantly outperforms RAG and memory-based baselines for continual learning from deployment data.

## When to Use

- When building a pipeline to improve an LLM application's responses using accumulated user interaction logs (chat histories, thumbs-up/down, edit traces, correction messages)
- When a deployed system collects user feedback but the feedback is noisy, contradictory, or unstructured and needs to be filtered before training
- When implementing continual learning for a production LLM system that must improve over time without full retraining
- When designing a feedback loop that distinguishes genuine user preference signals from noise or adversarial input
- When the user wants to cluster heterogeneous user feedback into coherent improvement themes before fine-tuning
- When replacing naive RAG-over-feedback or simple memory systems with a more principled optimization approach
- When building an automated evaluation pipeline that decides whether extracted feedback actually improves model behavior before deploying it

## Key Technique

**The core insight of UNO is that not all user feedback is equally useful, and the value of feedback depends on the gap between what the model already knows and what the feedback teaches.** Standard approaches -- retrieving raw feedback at inference time (RAG) or storing it in memory -- fail because they cannot distinguish signal from noise, and they suffer from distribution mismatch between collected logs and future queries. UNO solves this with a three-stage pipeline: distillation, clustering with cognitive gap assessment, and dual-path module construction.

**Log Distillation** converts each user session (query, response, user feedback) into a set of actionable rules and a preference pair. An LLM extracts up to 5 concrete rules from the feedback (e.g., "Use accessible, non-technical language in news-style writing"), then generates a revised response following those rules. The original response becomes the "rejected" sample and the revised response becomes the "preferred" sample, forming a DPO training pair. Sessions that yield empty rule sets are filtered as uninformative.

**Cognitive Gap Assessment** is the key differentiator. For each distilled sample, the base model independently predicts what rules it *thinks* should apply to the query (without seeing the actual feedback). A reranker computes the semantic distance between predicted and actual rules, producing a gap score in [0,1]. Clusters with low average gap (below threshold 0.45) contain feedback the model mostly already understands -- these are safe for direct parameter optimization. High-gap clusters contain risky or novel feedback where direct training may cause reward hacking -- these get a gentler critique-based module instead. This assessment reduces wasted training compute by 53-78% by skipping clusters that would introduce noise.

## Step-by-Step Workflow

1. **Collect and normalize user logs.** Gather interaction sessions containing: the user's original query, the system's response, and any user feedback (explicit corrections, follow-up messages indicating dissatisfaction, thumbs-down signals, or edit traces). Normalize into a consistent format: `{query, response, feedback}` triples.

2. **Distill rules from each session.** For each session, prompt an LLM to extract up to 5 actionable improvement rules from the user feedback. Use a structured prompt that asks: "Given the query, the model's response, and the user's feedback, extract concrete rules the model should follow." Output as a JSON list of rule strings. Discard sessions where no rules can be extracted.

3. **Generate preference pairs.** For each session with extracted rules, prompt the base model to generate a revised response following the rules. This creates a preference pair: `(query, preferred=revised_response, rejected=original_response)`. Apply quality filtering: remove pairs where the revised response is empty or has BLEU < 0.05 relative to the original (indicating no meaningful change).

4. **Create dual-feature embeddings and cluster.** For each sample, compute a composite embedding vector: `v_i = [Normalize(Embed(query)) || Normalize(Embed(rules))]` using a sentence encoder. Apply hierarchical agglomerative clustering with Ward linkage, merging until the intra-cluster variance increment exceeds a threshold (default: 4.0). This groups samples that share both semantic intent and applicable feedback themes.

5. **Quantify cognitive gap per cluster.** For each query in a cluster, have the base model predict rules independently (without seeing actual feedback). Compute the semantic distance between predicted and actual rules using a reranker model. Average these gap scores across the cluster. Classify clusters as Low-Gap (mean gap <= 0.45) or High-Gap (mean gap > 0.45).

6. **Build Primary Experience Modules for low-gap clusters.** Train a LoRA adapter using DPO loss + NLL loss (weighted 0.5 each) on the preference pairs from low-gap clusters. Use LoRA rank 64, dropout 0.05, learning rate 5e-4, 8 epochs. This Expert LoRA directly modifies response generation.

7. **Validate Primary Modules with a simulated verifier.** Use an LLM-as-Judge to compare Expert LoRA responses against base model responses on held-out samples, using the extracted rules as evaluation criteria. Sample the judge 3 times and average. Only deploy modules achieving win-rate > 0.53 (i.e., 3+ points above baseline). Modules that fail validation get demoted to the reflective path.

8. **Build Reflective Experience Modules for high-gap or failed clusters.** Train a separate Critic LoRA that learns to generate pseudo-feedback (rules) given a query and a base model response. At inference time, the base model generates an initial response, the Critic LoRA produces improvement suggestions, and the base model revises its response following those suggestions. This two-pass approach avoids risky direct parameter changes.

9. **Route queries at inference time.** When a new query arrives, compute its embedding and find the nearest cluster centroid. If the distance exceeds 1.2, fall back to the base model (no adaptation). Otherwise, route to the cluster's Primary or Reflective module as appropriate.

10. **Iterate for online evolution.** As new user logs accumulate, batch them and repeat the pipeline. Preserve cluster centroids if the cluster structure is stable; retrain clustering if the number of clusters changes. Only update Expert LoRAs if the new batch achieves a win-rate improvement of 0.03+ over the current version.

## Concrete Examples

**Example 1: Improving a customer support chatbot from interaction logs**

```
User: "I have 3 months of chat logs from our support bot. Users keep complaining
that responses are too formal and don't address their specific product version.
Help me build a pipeline to improve the bot from these logs."

Approach:
1. Parse chat logs into (query, bot_response, user_followup) triples. Identify
   sessions where user followup indicates dissatisfaction (e.g., "that's not
   what I asked", rephrasing the question, explicit complaints).

2. Run rule extraction on each dissatisfied session:
   Input:  query="How do I reset my V3 device?"
           response="To reset your device, navigate to Settings > General > Reset..."
           feedback="I'm on V2, not V3. And can you be less robotic?"
   Output: rules=["Check product version before giving instructions",
                   "Use conversational, less formal tone",
                   "Ask clarifying questions when version is ambiguous"]

3. Generate revised response following rules:
   revised="Hey! Just to make sure I help you right -- are you using V2 or V3?
            The reset steps are a bit different for each. If you're on V2,
            go to Settings > Device > Factory Reset."

4. Cluster the 500 extracted samples. Expect clusters like:
   - Cluster A: "version-specific queries" (120 samples, gap=0.32 -> Low-Gap)
   - Cluster B: "tone complaints" (80 samples, gap=0.28 -> Low-Gap)
   - Cluster C: "edge-case product issues" (40 samples, gap=0.71 -> High-Gap)

5. Train Expert LoRA on Clusters A and B (direct preference optimization).
   Train Critic LoRA on Cluster C (generates suggestions at inference time).

6. At inference: new query about V2 reset -> routes to Cluster A's Expert LoRA
   -> directly generates version-aware, conversational response.

Output: A deployment-ready pipeline with 2 Expert LoRAs, 1 Critic LoRA,
cluster centroids for routing, and a validation report showing win-rates.
```

**Example 2: Building a feedback distillation pipeline for a code assistant**

```
User: "Our code assistant gets mixed feedback -- some users give helpful
corrections but many just say 'wrong' or paste unrelated code. Help me
filter the noise and extract useful training signal."

Approach:
1. Normalize logs: each session = (code_query, assistant_code, user_reaction).
   User reactions range from "wrong" to detailed corrections with explanations.

2. Run rule extraction. For noisy sessions:
   Input:  query="Sort this list", response="list.sort()", feedback="wrong"
   Output: rules=[]  (no actionable rules -> DISCARD this session)

   For informative sessions:
   Input:  query="Parse this CSV safely"
           response="open('file.csv').read().split(',')"
           feedback="This doesn't handle quoted commas or encoding. Use the
                     csv module with proper error handling."
   Output: rules=["Use csv module instead of manual string splitting",
                   "Handle encoding with explicit encoding parameter",
                   "Wrap file operations in try/except for IO errors"]

3. After distillation: 2000 raw sessions -> 600 sessions with valid rules
   (70% noise reduction from empty-rule filtering alone).

4. Cluster and assess cognitive gap:
   - "Standard library usage" cluster (gap=0.22): model mostly knows this
     but sometimes picks naive approaches -> Expert LoRA
   - "Domain-specific patterns" cluster (gap=0.68): model unfamiliar with
     specialized libraries -> Critic LoRA (safer to suggest than to override)

5. Validate: Expert LoRA on standard library cluster achieves 0.61 win-rate
   against base model -> deploy. Critic LoRA generates suggestions like
   "Consider using the `csv` module with `DictReader`" that the base model
   incorporates in a revision pass.

Output: Filtered dataset of 600 preference pairs, 2 cluster-specific LoRAs,
routing configuration, and a noise analysis report.
```

**Example 3: Implementing cognitive gap assessment on existing feedback data**

```
User: "I already have structured feedback pairs but I'm not sure which ones
are safe to train on. Help me assess which feedback clusters are high-risk."

Approach:
1. For each (query, rules) pair, prompt the base model to independently
   predict what rules should apply:
   prompt="Given this query, what improvement rules would you suggest
           for a response? Output as JSON list."

2. Compare predicted vs actual rules using a reranker/similarity model:
   - query="Explain quantum entanglement to a 10-year-old"
     actual_rules=["Use simple analogies", "Avoid jargon", "Keep under 100 words"]
     predicted_rules=["Use analogies", "Simple language", "Brief explanation"]
     gap_score=0.18  (model already knows this -> low risk)

   - query="Write a lease termination letter for California"
     actual_rules=["Cite CA Civil Code 1946.1", "Include 30-day notice period",
                    "Reference specific lease clause 12.3"]
     predicted_rules=["Be formal", "State intent to terminate", "Give notice"]
     gap_score=0.82  (model lacks jurisdiction-specific knowledge -> high risk)

3. Cluster and compute per-cluster averages. Flag clusters above 0.45
   as requiring the reflective (critique-based) training path.

Output: Gap assessment report with per-cluster risk scores, recommended
training path (Primary vs Reflective) for each cluster, and estimated
compute savings from skipping high-risk direct training.
```

## Best Practices

**Do:**
- Always filter sessions with empty rule sets before any clustering or training -- they contain no actionable signal and dilute training data
- Use the dual-feature embedding (query + rules concatenated) for clustering, not query-only embeddings -- feedback theme matters as much as query topic for grouping
- Set the cognitive gap threshold conservatively (0.45 is the paper's validated default) -- overly aggressive direct training on high-gap clusters risks reward hacking
- Validate every Primary Experience Module with a simulated judge before deployment -- never skip this step even if gap scores look safe
- Implement the distance-based fallback at inference time (threshold 1.2 from nearest centroid) -- queries outside known cluster regions should use the base model unchanged

**Avoid:**
- Training directly on raw user feedback without distillation -- unstructured feedback degrades DPO training and introduces contradictory signals
- Using a single training path for all clusters -- low-gap and high-gap clusters have fundamentally different noise profiles requiring different treatment
- Skipping the preference pair quality filter (BLEU < 0.05 removal) -- degenerate pairs where the revision barely changes the original cause training instability
- Over-clustering with a low variance threshold -- too many small clusters produce unreliable gap estimates and waste compute on per-cluster LoRA training

## Error Handling

- **Empty rule extraction on most sessions**: If >80% of sessions yield no rules, the feedback format is likely too terse. Preprocess to combine multi-turn feedback into a single context window, or lower the extraction threshold by asking for even partial rules.
- **All clusters classified as high-gap**: The base model may be fundamentally misaligned with the domain. Consider doing a round of standard SFT on a curated subset before applying UNO, or rely entirely on Reflective modules for the first iteration.
- **Expert LoRA fails simulated verification**: This is expected for ~20-40% of clusters. The module is automatically demoted to the Reflective path. If all modules fail, check whether the preference pairs have sufficient quality contrast between preferred and rejected responses.
- **Cluster count instability across iterations**: If adding new batches of logs drastically changes cluster structure each time, increase the variance threshold (epsilon) to produce fewer, more stable clusters. The paper uses 4.0 as default.
- **Inference latency from Reflective modules**: The two-pass generation (base -> critique -> revise) doubles latency. For latency-sensitive applications, consider caching Critic suggestions for frequent query patterns, or invest more in expanding Primary module coverage through additional training iterations.

## Limitations

- **Requires meaningful user feedback**: UNO cannot extract signal from sessions where users silently abandon the conversation or give purely binary signals (thumbs up/down without context). At minimum, corrective text or follow-up complaints are needed.
- **Minimum data scale**: The clustering and gap assessment stages need sufficient data per cluster to be statistically reliable. Expect diminishing returns below ~200 total sessions with valid rules.
- **LoRA training infrastructure**: The Primary and Reflective module construction requires GPU resources for LoRA fine-tuning. This is not a prompt-only technique -- it fundamentally requires parameter adaptation.
- **Assumes relatively stable user intent distribution**: If the user population or task distribution shifts dramatically between log collection and deployment, the cluster routing may assign queries to inappropriate modules.
- **Language-dependent prompts**: The rule extraction and revision prompts in the reference implementation are tuned for English and Chinese. Other languages may need prompt adaptation.

## Reference

**Paper**: [Improve Large Language Model Systems with User Logs](https://arxiv.org/abs/2602.06470v1) (Wang et al., 2026). Look for: Section 3 for the full UNO pipeline, Theorem 3.2 for the noise risk bound justifying cognitive gap thresholds, and Table 1-2 for performance comparisons against RAG and memory baselines.

**Code**: [github.com/bebr2/UNO](https://github.com/bebr2/UNO) -- Reference implementation using vLLM, Qwen3-8B, and the MemoryBench evaluation framework.