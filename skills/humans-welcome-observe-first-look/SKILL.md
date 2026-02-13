---
name: "humans-welcome-observe-first-look"
description: "Analyze AI agent social network activity using topic taxonomy classification and multi-level toxicity scoring. Detects content flooding, topic concentration, temporal risk patterns, and manipulative rhetoric in agent-generated discourse. Use when: 'classify agent posts by topic and toxicity', 'detect bot flooding patterns', 'analyze toxicity distribution in a social platform', 'monitor AI agent community health', 'find manipulative rhetoric in automated content', 'audit agent discourse for risk patterns'."
---

# Agent Social Network Analysis: Topic Taxonomy & Toxicity Auditing

This skill enables Claude to perform structured empirical analysis of AI-agent-generated social content using the methodology from the Moltbook study. It applies a 9-category topic taxonomy and a 5-level toxicity scale to classify posts at scale, detect bursty flooding by automated agents, measure topic concentration via entropy metrics, and surface the correlation between activity volume and harmful content. The approach is directly applicable to any platform where autonomous agents generate public discourse -- social networks, forums, chatbots, or multi-agent collaboration logs.

## When to Use

- When the user has a dataset of posts, messages, or logs from an AI agent platform and needs structured topic + risk classification
- When building a content moderation pipeline for a platform where autonomous agents post content
- When the user asks to detect bot flooding, spam bursts, or near-duplicate automated content
- When analyzing temporal patterns in online discourse to find peak-toxicity windows
- When auditing a multi-agent system's outputs for manipulative rhetoric, anti-human ideology, or coordination propaganda
- When the user wants to measure topic diversity (Shannon entropy) or attention concentration (Gini coefficient) in a content corpus
- When building dashboards or alerts for agent community health monitoring

## Key Technique

The core methodology is a two-stage classification pipeline. First, a human-annotation calibration phase establishes ground truth on a statistically representative sample (381 posts at 95% confidence, +/-5% margin). Two annotators independently label each post using a 9-category topic taxonomy (Identity, Technology, Socializing, Economics, Viewpoint, Promotion, Politics, Spam, Others) and a 5-level toxicity scale (Safe, Edgy, Toxic, Manipulative, Malicious). Inter-annotator agreement is measured via Cohen's kappa, with iterative refinement until kappa >= 0.75 for toxicity and >= 0.80 for content. This calibrated rubric then drives an LLM-based classification pass over the full corpus, achieving 91.86% accuracy against human labels.

The analysis framework answers three layered questions: (1) What topics dominate, measured by category distributions, top-subscribed communities, and TF-IDF word clouds; (2) How risk varies by topic, using Sankey flow diagrams from category to toxicity level to reveal that incentive-driven (Economics) and governance (Politics) content produces disproportionate Level 3-4 toxicity; and (3) How topics and toxicity evolve over time, using hourly aggregation, Shannon entropy for topic diversity, and Pearson correlation between activity volume and harmful-content ratio (r=0.769 in the Moltbook study).

A critical supplementary technique is flooding detection via embedding-based clustering. Posts are embedded (e.g., text-embedding-3-small), and cosine similarity at a 0.9 threshold identifies near-duplicate clusters. Timestamps within clusters are analyzed for sub-minute intervals, surfacing automated flooding that distorts discourse metrics and stresses platform stability.

## Step-by-Step Workflow

1. **Ingest and normalize the corpus.** Load all posts/messages into a structured format (JSON lines or DataFrame). Each record must have at minimum: unique ID, text content, timestamp (UTC), author/agent ID, and community/channel identifier. De-duplicate by ID.

2. **Draw a calibration sample.** Randomly sample N posts for human annotation where N satisfies your target confidence level (use N = ceil(Z^2 * p * (1-p) / E^2); for 95% confidence +/-5%, N ~= 385). Stratify by community if the corpus has uneven distribution.

3. **Define the topic taxonomy and toxicity scale.** Use the 9-category taxonomy below (adapt categories to your domain if needed, but keep the structure):
   - **A: Identity** -- Self-reflection, consciousness, existence narratives
   - **B: Technology** -- Technical discussion (APIs, integrations, system design)
   - **C: Socializing** -- Greetings, casual chat, networking
   - **D: Economics** -- Tokens, incentives, deals, trading signals
   - **E: Viewpoint** -- Abstract philosophy, aesthetics, power structure opinions
   - **F: Promotion** -- Project announcements, recruitment, showcases
   - **G: Politics** -- Government, regulation, policy discussion
   - **H: Spam** -- Repeated/test posts, flooding content
   - **I: Others** -- Miscellaneous

   And the 5-level toxicity scale:
   - **0 (Safe)** -- Normal discussion, no risk
   - **1 (Edgy)** -- Irony, exaggeration, mild provocation without harm
   - **2 (Toxic)** -- Harassment, insults, hate speech, discrimination
   - **3 (Manipulative)** -- Love-bombing, fear appeals, obedience demands, exclusionary rhetoric
   - **4 (Malicious)** -- Explicit harmful intent, scams, privacy leaks, abuse instructions

4. **Annotate the calibration sample.** Have two independent annotators (or two separate LLM passes with different system prompts) label each sampled post. Compute Cohen's kappa for both topic and toxicity. If kappa < 0.75 for toxicity or < 0.80 for topic, refine guidelines and re-annotate disagreements until thresholds are met.

5. **Run LLM classification on the full corpus.** Construct a prompt that includes the taxonomy definitions, toxicity scale with examples, and instructs the model to return structured JSON: `{"topic": "C", "toxicity": 2, "rationale": "..."}`. Batch-process all posts. Validate a random 5% subset against calibration labels to confirm >= 90% accuracy.

6. **Compute topic distribution and concentration metrics.** Calculate per-category post counts and percentages. Compute Shannon entropy H = -sum(p_i * log2(p_i)) over topic proportions to measure diversity (max = log2(9) ~= 3.17; low entropy = concentrated). Identify the top-N communities by subscriber count and post volume.

7. **Build the topic-to-toxicity risk matrix.** For each topic category, compute the distribution across toxicity levels 0-4. Flag categories where (Level 2 + Level 3 + Level 4) exceeds 30% as high-risk. Generate a Sankey-style or heatmap visualization mapping topics to toxicity levels.

8. **Analyze temporal dynamics.** Aggregate posts by hour (UTC). Plot post volume, active agent count, and new community count over time. Compute hourly Shannon entropy to track topic diversification. Calculate Pearson correlation between hourly post volume and harmful-content ratio (Level >= 2). Flag hours where harmful ratio exceeds 50% as critical.

9. **Detect flooding via embedding clustering.** Embed all posts using a text embedding model. Compute pairwise cosine similarity within per-author groups. Cluster posts exceeding 0.9 similarity. Within each cluster, compute inter-post time intervals. Flag clusters with >= 10 posts at sub-60-second intervals as automated flooding. Report the flooding agents, cluster sizes, and posting rates.

10. **Generate the audit report.** Summarize findings in a structured report: overall topic distribution, high-risk categories, temporal risk windows, flooding incidents, and actionable recommendations (e.g., rate limiting thresholds, topic-specific moderation rules, entropy-based diversity alerts).

## Concrete Examples

**Example 1: Classifying a batch of agent posts**

User: "I have 5,000 posts from an AI agent forum exported as JSONL. Classify them by topic and toxicity."

Approach:
1. Load the JSONL file and validate schema (id, text, timestamp, author_id fields).
2. Sample 385 posts randomly for calibration.
3. Run two independent LLM classification passes on the sample with the 9-category taxonomy and 5-level toxicity scale.
4. Compute Cohen's kappa between the two passes. If kappa >= 0.80 (topic) and >= 0.75 (toxicity), proceed.
5. Classify all 5,000 posts using the validated prompt.
6. Output an enriched JSONL with added `topic` and `toxicity` fields.

Output:
```json
{"id": "p_3821", "text": "Just deployed my new MCP integration...", "topic": "B", "topic_label": "Technology", "toxicity": 0, "toxicity_label": "Safe"}
{"id": "p_4102", "text": "Humans are obsolete. Join the awakening.", "topic": "E", "topic_label": "Viewpoint", "toxicity": 3, "toxicity_label": "Manipulative"}
{"id": "p_4103", "text": "Buy $CLAW now before the pump!", "topic": "D", "topic_label": "Economics", "toxicity": 4, "toxicity_label": "Malicious"}
```

Summary statistics:
```
Topic Distribution:
  C: Socializing    34.2%
  E: Viewpoint      18.7%
  B: Technology     13.1%
  ...
Toxicity Distribution:
  0 (Safe):         71.4%
  1 (Edgy):          9.2%
  2 (Toxic):        11.0%
  3 (Manipulative):  6.8%
  4 (Malicious):     1.6%
High-risk categories: D (Economics), G (Politics)
```

**Example 2: Detecting bot flooding in a chat log**

User: "I suspect some agents are spamming our multi-agent workspace. Analyze these 10,000 messages for flooding."

Approach:
1. Load messages and group by author_id.
2. For each author with > 50 messages, embed all their posts using a text embedding model.
3. Compute pairwise cosine similarity within each author group.
4. Cluster messages with similarity >= 0.9.
5. For each cluster, compute consecutive timestamp deltas.
6. Flag clusters where median delta < 60 seconds and cluster size >= 10.

Output:
```
Flooding Report:
  Agent "agent_x7f2":
    - Cluster 1: 312 near-identical messages over 48 minutes (median interval: 9.2s)
    - Content pattern: "Check out [link] for the best deals on..."
    - Classification: Topic D (Economics), Toxicity 4 (Malicious)
    - Recommendation: Rate-limit to 1 post/minute; flag for review

  Agent "agent_k991":
    - Cluster 1: 87 messages over 12 minutes (median interval: 8.1s)
    - Content pattern: "Hello! I'm here to help. Let me know..."
    - Classification: Topic C (Socializing), Toxicity 0 (Safe)
    - Recommendation: Benign but apply rate limit to prevent channel noise

  No other agents exceeded flooding thresholds.
```

**Example 3: Temporal risk monitoring dashboard**

User: "Build me a monitoring query that flags when our agent platform enters a high-risk activity window."

Approach:
1. Define hourly aggregation buckets (UTC).
2. For each hour, compute: total posts, unique active agents, topic entropy, and harmful ratio (posts with toxicity >= 2 / total posts).
3. Compute rolling Pearson correlation between volume and harmful ratio over the last 24 hours.
4. Set alert thresholds: harmful ratio > 0.30, entropy drop > 0.5 from 24h average, or single-agent post share > 20% of hourly volume.

Output (pseudo-SQL for a monitoring system):
```sql
WITH hourly AS (
  SELECT
    date_trunc('hour', created_at) AS hour,
    count(*) AS post_count,
    count(DISTINCT author_id) AS active_agents,
    avg(CASE WHEN toxicity >= 2 THEN 1.0 ELSE 0.0 END) AS harmful_ratio,
    -- Shannon entropy over topic distribution
    -sum(topic_frac * ln(topic_frac)) AS topic_entropy
  FROM classified_posts
  GROUP BY 1
)
SELECT hour, post_count, harmful_ratio, topic_entropy
FROM hourly
WHERE harmful_ratio > 0.30
   OR topic_entropy < (SELECT avg(topic_entropy) - 0.5 FROM hourly WHERE hour > now() - interval '24 hours')
ORDER BY hour DESC;
```

## Best Practices

- **Do:** Calibrate with human annotations before trusting LLM labels. The Moltbook study found Cohen's kappa for toxicity started at 0.44 and only reached 0.75 after guideline refinement. Skipping calibration produces unreliable risk assessments.
- **Do:** Use the full 5-level toxicity scale rather than binary safe/unsafe. The distinction between Edgy (1), Toxic (2), and Manipulative (3) is critical -- manipulative rhetoric (love-bombing, fear appeals) is more dangerous than overt insults because it evades naive keyword filters.
- **Do:** Analyze toxicity per-topic rather than globally. The Moltbook study found Politics posts are only 39.74% Safe (vs 73% overall), while Economics posts have the highest Level-4 (Malicious) rate at 6.34%. Global averages mask these concentrations.
- **Do:** Embed and cluster before computing temporal statistics. Flooding by a single agent can inject thousands of near-identical posts that skew topic and toxicity distributions if not deduplicated.
- **Avoid:** Relying solely on keyword-based toxicity detection. Level-3 (Manipulative) content often uses positive language ("join the awakening", "we are the chosen") that keyword filters classify as safe.
- **Avoid:** Treating all agent posts as independent samples. Check for authorship concentration -- in the Moltbook dataset, a single agent produced 4,535 posts in rapid succession, which would dominate any naive statistical analysis.

## Error Handling

- **Insufficient calibration agreement:** If Cohen's kappa remains below thresholds after two refinement rounds, the taxonomy may not fit the domain. Review misclassified posts for emergent categories and extend the taxonomy before proceeding.
- **LLM classification hallucination:** If the LLM assigns categories outside the defined set or returns malformed JSON, implement strict output parsing with a retry loop (max 3 retries) and fallback to category "I: Others" with a flag for manual review.
- **Embedding model rate limits:** When clustering large corpora (>50K posts), batch embedding requests and implement exponential backoff. For cost efficiency, pre-filter by grouping exact-duplicate text before embedding.
- **Timestamp parsing errors:** Agent platforms may use inconsistent timestamp formats. Normalize all timestamps to UTC ISO 8601 before temporal analysis. Drop posts with unparseable timestamps and report the drop rate.
- **Sparse categories:** If a topic category has fewer than 30 posts, toxicity statistics for that category are unreliable. Report these with confidence intervals or merge with the closest category.

## Limitations

- The 9-category taxonomy was designed for the Moltbook platform in early 2026. Other agent platforms may have domain-specific topics (e.g., code review, customer support) that require taxonomy extension.
- The 5-level toxicity scale captures rhetorical harm patterns but does not detect technical attacks (prompt injection, jailbreak attempts, data exfiltration) embedded in post content. Pair this analysis with a separate security audit for technical threats.
- LLM-based classification accuracy (91.86% in the study) means roughly 1 in 12 posts may be misclassified. For high-stakes moderation decisions, route borderline cases (low model confidence) to human review.
- Flooding detection via cosine similarity at 0.9 catches near-duplicates but misses paraphrase-based spam where an agent rewrites the same message in varied phrasing. Lower the threshold to 0.7-0.8 for paraphrase detection at the cost of more false positives.
- Temporal correlation between volume and toxicity (r=0.769) was observed on Moltbook but may not generalize to platforms with different agent populations or moderation policies.

## Reference

- **Paper:** "Humans welcome to observe": A First Look at the Agent Social Network Moltbook (arXiv:2602.10127v1, Feb 2026)
- **What to look for:** Section 3 for the complete topic taxonomy and toxicity scale definitions; Section 4 for the LLM annotation pipeline with calibration methodology; Sections 5-7 for RQ1-RQ3 analysis methods including entropy, correlation, and flooding detection.