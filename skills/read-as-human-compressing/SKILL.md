---
name: "read-as-human-compressing"
description: "Compress long contexts using the RAM (Read As Human) strategy: partition text into segments, score relevance against a query, fully retain high-relevance segments (close reading), and compress low-relevance segments into compact summaries (skimming). Use when: 'compress this long document for my query', 'summarize only the relevant parts', 'reduce context length before processing', 'skim this codebase for relevant sections', 'adaptive context compression', 'parallel segment relevance filtering'."
---

# Read As Human: Adaptive Context Compression via Close Reading and Skimming

This skill enables Claude to compress long contexts (documents, codebases, logs, conversation histories) by mimicking how humans read: closely reading important passages while skimming less relevant ones. Based on the RAM framework, the technique partitions input into segments, scores each segment's relevance to the user's query using cosine similarity, retains high-relevance segments verbatim, and collapses low-relevance segments into brief summaries. The result is a compressed context that preserves critical information while dramatically reducing token count — enabling faster downstream processing and fitting long inputs within context windows.

## When to Use

- When a user provides a long document (>4K tokens) and a specific question, and you need to reduce context before answering
- When processing large codebases where only certain files/functions are relevant to the user's query
- When summarizing logs, transcripts, or chat histories while preserving query-relevant details
- When the user explicitly asks to "compress", "reduce", or "distill" a long text relative to a task
- When preparing context for a downstream LLM call that has a limited context window
- When triaging multiple documents to extract only passages relevant to a specific topic
- When a user asks to "skim" a document and highlight what matters for their question

## Key Technique

RAM (Read As Human) treats context compression as a two-mode reading problem. Instead of uniformly summarizing or uniformly truncating, it **partitions the input into fixed-size segments** (~50 tokens each) and computes a relevance score for each segment against the user's query. The scoring uses cosine similarity between segment and query representations, normalized via temperature-scaled softmax (temperature = 0.1). This produces a probability distribution over segments that reflects how much each one matters for the task at hand.

The framework then applies a **compression ratio** (e.g., 4x, 8x, 16x) to determine how many segments to retain verbatim. The top-k highest-scoring segments are kept in full ("close reading"), while all remaining segments are compressed into single-sentence summaries ("skimming"). The key insight is that this is **not uniform compression** — a 4x compression ratio doesn't mean every segment loses 75% of its tokens. Instead, the most relevant quarter is preserved exactly, and the remaining three-quarters are aggressively summarized. This adaptive allocation concentrates information density where it matters most.

What makes this practical for Claude is the **parallelizable architecture**: each segment is scored independently against the query, so relevance computation scales linearly rather than quadratically. In the original paper, this yields up to 12x end-to-end speedup on 16K-32K token inputs. For Claude's application, the analogous benefit is that segment scoring and summarization can proceed chunk-by-chunk without needing to hold the entire context in working memory simultaneously.

## Step-by-Step Workflow

1. **Identify the query and context.** Separate the user's question or task (the "query") from the long input text (the "context"). If the user hasn't provided an explicit query, infer the intent from their request.

2. **Partition the context into segments.** Split the input into chunks of approximately 50 tokens (roughly 2-3 sentences or 4-5 lines of code). Respect natural boundaries: split at paragraph breaks, function boundaries, log entry separators, or section headers rather than mid-sentence.

3. **Choose a compression ratio.** Based on the user's needs and context length, select a target compression ratio:
   - **2x**: Light compression — retain half the segments verbatim. Use when accuracy is critical.
   - **4x**: Standard compression — retain one quarter. Good default for most tasks.
   - **8x**: Aggressive compression — retain one eighth. Use for very long inputs (>16K tokens).
   - **16x or higher**: Extreme compression — use only when the context is massive and the query is narrow.

4. **Score each segment for query relevance.** For each segment, assess how relevant it is to the query. Consider: Does this segment contain keywords, entities, or concepts from the query? Does it provide direct answers, supporting evidence, or necessary context? Assign a mental relevance tier: HIGH (directly answers or critically supports the query), MEDIUM (provides useful background), or LOW (tangential or boilerplate).

5. **Select segments for close reading (full retention).** Calculate k = total_segments / compression_ratio. Select the top-k highest-relevance segments for verbatim retention. These are reproduced exactly as they appear in the original text, preserving code syntax, exact quotes, data values, and formatting.

6. **Compress remaining segments via skimming.** For each non-retained segment, generate a **query-aware one-line summary**. The summary should capture only what is relevant to the query from that segment. If a segment has zero relevance to the query, compress it to a brief label like "[unrelated discussion of X]" or "[boilerplate imports]".

7. **Assemble the compressed context.** Concatenate the results in original document order. Retained segments appear in full; skimmed segments appear as their one-line summaries. Mark boundaries clearly so the user can distinguish retained from summarized content.

8. **Use the compressed context to answer the query.** Process the assembled compressed context to generate the final response. If the answer isn't found in retained segments, check whether skimmed summaries hint at relevant information that was compressed away, and flag this to the user.

9. **Report compression statistics.** Tell the user: original token count, compressed token count, compression ratio achieved, and how many segments were retained vs. skimmed.

## Concrete Examples

**Example 1: Compressing a long error log for debugging**

User: "Here are 500 lines of application logs. Why is the payment service timing out?"

Approach:
1. Query identified: payment service timeout root cause
2. Partition 500 log lines into ~25 segments of 20 lines each
3. Target compression: 4x (retain ~6 segments)
4. Score segments: lines mentioning "payment", "timeout", "connection refused", "retry", or related service names score HIGH; startup logs, health checks, and unrelated service logs score LOW
5. Retain the 6 highest-scoring segments verbatim
6. Compress remaining 19 segments into one-liners

Output:
```
## Compressed Log (4x compression: 500 → ~130 lines)

[Segments 1-3: Normal startup sequence for auth, user, and inventory services]
[Segment 4: Health checks passing for all services, 0 errors]

--- RETAINED (relevant) ---
2024-03-15 14:23:01 [payment-svc] ERROR: Connection to stripe-gateway timed out after 30000ms
2024-03-15 14:23:01 [payment-svc] ERROR: Retry attempt 1/3 for charge request ch_abc123
2024-03-15 14:23:31 [payment-svc] ERROR: Connection to stripe-gateway timed out after 30000ms
2024-03-15 14:23:31 [payment-svc] WARN: Circuit breaker OPEN for stripe-gateway
...
--- END RETAINED ---

[Segments 7-9: Inventory service processing orders normally]

--- RETAINED (relevant) ---
2024-03-15 14:22:58 [gateway] INFO: DNS resolution for stripe-gateway.internal: 10.0.3.47 (TTL: 30s)
2024-03-15 14:22:59 [gateway] WARN: TLS handshake with 10.0.3.47:443 failed: connection reset
2024-03-15 14:23:00 [gateway] ERROR: Upstream 10.0.3.47 marked unhealthy after 3 consecutive failures
...
--- END RETAINED ---

[Segments 12-25: Auth service logs, cron jobs, metric collection — no payment references]

Compression: 500 lines → 128 lines (3.9x). 6 of 25 segments retained.
Root cause: The gateway's TLS handshake to stripe-gateway.internal is failing,
causing the payment service to time out waiting for a connection.
```

**Example 2: Extracting relevant code from a large file**

User: "This 800-line React component handles user settings. How does the password change flow work?"

Approach:
1. Query: password change implementation
2. Partition into ~16 segments of 50 lines each
3. Target compression: 4x (retain ~4 segments)
4. HIGH relevance: segments containing `password`, `changePassword`, `handlePasswordSubmit`, password validation logic, API calls to `/auth/password`
5. LOW relevance: theme settings, notification preferences, profile picture upload, layout/CSS

Output:
```
## Compressed Component (4x: 800 → ~220 lines)

[Lines 1-50: Imports, theme context, notification preference types]
[Lines 51-100: Profile picture upload state and handlers]

--- RETAINED ---
// Lines 101-150: Password change state and validation
const [currentPassword, setCurrentPassword] = useState('');
const [newPassword, setNewPassword] = useState('');
const [confirmPassword, setConfirmPassword] = useState('');
const [passwordError, setPasswordError] = useState<string | null>(null);

const validatePassword = (pw: string): boolean => {
  if (pw.length < 12) return false;
  if (!/[A-Z]/.test(pw)) return false;
  if (!/[0-9]/.test(pw)) return false;
  return true;
};
--- END RETAINED ---

[Lines 151-250: Theme toggle, language selector, notification channel config]

--- RETAINED ---
// Lines 251-350: Password submission handler and API call
const handlePasswordChange = async () => {
  if (newPassword !== confirmPassword) {
    setPasswordError('Passwords do not match');
    return;
  }
  if (!validatePassword(newPassword)) {
    setPasswordError('Password must be 12+ chars with uppercase and number');
    return;
  }
  try {
    await api.post('/auth/password', {
      current: currentPassword,
      new: newPassword,
    });
    toast.success('Password updated');
    resetPasswordFields();
  } catch (err) {
    if (err.status === 401) setPasswordError('Current password is incorrect');
    else setPasswordError('Failed to update password');
  }
};
--- END RETAINED ---

[Lines 351-600: Data export settings, connected accounts, accessibility options]

--- RETAINED ---
// Lines 601-650: Password change form JSX
<Section title="Change Password">
  <PasswordInput label="Current" value={currentPassword} onChange={setCurrentPassword} />
  <PasswordInput label="New" value={newPassword} onChange={setNewPassword} />
  <PasswordInput label="Confirm" value={confirmPassword} onChange={setConfirmPassword} />
  {passwordError && <ErrorText>{passwordError}</ErrorText>}
  <Button onClick={handlePasswordChange}>Update Password</Button>
</Section>
--- END RETAINED ---

[Lines 651-800: Layout wrapper, footer links, export default]

Compression: 800 → 205 lines (3.9x). 4 of 16 segments retained.
```

**Example 3: Compressing research documents for a focused question**

User: "I have three research papers (total ~20K tokens) about transformer architectures. What do they say specifically about attention head pruning?"

Approach:
1. Query: attention head pruning techniques and findings
2. Partition each paper into segments; ~400 segments total at 50 tokens each
3. Target compression: 8x (retain ~50 segments across all three papers)
4. Score each segment for mentions of: "pruning", "attention head", "head importance", "redundant heads", "sparse attention", related metrics
5. Retain the 50 most relevant segments; compress the rest into topic labels

Output:
```
## Compressed Research Context (8x: ~20K → ~2.5K tokens)

### Paper 1: "Efficient Transformers: A Survey"
[Abstract and introduction — general efficiency overview]
[Sections on knowledge distillation, quantization — not about pruning]

--- RETAINED: Section 4.2 ---
"Attention head pruning removes entire heads deemed redundant.
Michel et al. (2019) showed that in BERT, up to 40% of heads
can be removed with <1% accuracy drop on MNLI..."
--- END RETAINED ---

[Sections on linear attention, sparse patterns — tangential]

### Paper 2: "Are Sixteen Heads Really Better Than One?"
--- RETAINED: Core findings (3 segments) ---
[Full text of abstract, methodology, and Table 2 showing per-head importance scores]
--- END RETAINED ---

[Appendix, related work — supplementary]

### Paper 3: ...

Compression: ~20K → ~2.4K tokens (8.3x). 50 of 400 segments retained.
```

## Best Practices

- **Do:** Respect natural boundaries when segmenting. Split at paragraph breaks, function definitions, log timestamps, or section headers — never mid-sentence or mid-expression.
- **Do:** Preserve retained segments exactly as-is, including formatting, indentation, and original line numbers. The value of "close reading" is that nothing is lost.
- **Do:** Make skimmed summaries query-aware. "[Discussion of caching strategies]" is better than "[Paragraph about technical topics]" because it tells the user what was compressed away in case they need to revisit it.
- **Do:** When uncertain whether a segment is relevant, err toward retention. Under-compressing is less harmful than losing critical information.
- **Avoid:** Uniform summarization that treats every segment equally. The core insight of RAM is adaptive allocation — some parts deserve full fidelity, others deserve aggressive compression.
- **Avoid:** Compressing structured data (tables, JSON configs, code with precise syntax) in skimmed segments. If a segment contains structured data even tangentially related to the query, retain it — structure doesn't survive summarization well.

## Error Handling

- **Query is too vague to score relevance:** Ask the user to refine their query before compressing. Without a clear query, all segments score similarly and compression degenerates into uniform truncation. Example: "compress this document" should become "compress this document — I need the parts about error handling."
- **All segments score HIGH relevance:** Reduce compression ratio or tell the user the document is uniformly relevant and compression will lose information. Suggest 2x instead of 8x.
- **All segments score LOW relevance:** The document may not contain the answer. Report this to the user rather than retaining arbitrary segments. Flag: "None of the content appears directly relevant to your query about X."
- **Retained segments lack coherence without surrounding context:** Include bridge sentences between retained segments, e.g., "[The following segment occurs after a discussion of authentication, where the code established a session token:]"
- **Compression artifacts in code:** If compressing source code, never summarize import statements that retained code depends on. Check for dependency references across segment boundaries.

## Limitations

- **Not suitable for tasks requiring global understanding.** If the user needs to understand the overall structure or narrative arc of a document (e.g., "summarize this entire paper"), uniform summarization is better than query-focused compression.
- **Relevance scoring degrades with ambiguous queries.** The technique works best when the query is specific. Broad queries like "tell me about this code" produce flat relevance distributions that make close-reading/skimming decisions unreliable.
- **Lossy by design.** Skimmed segments lose detail. If the answer to the query is buried in a single sentence within a low-scoring segment, it may be compressed away. Always flag this risk to the user.
- **Segment boundaries can split relevant content.** A key sentence at the boundary of two segments may cause one to score high and the adjacent one to score low, losing necessary context. Mitigate by using natural boundaries for segmentation.
- **Diminishing returns below 2x compression.** If the context is already within token limits, the overhead of segmenting, scoring, and reassembling may not be worth it. Use this technique when compression is genuinely needed.

## Reference

**Paper:** [Read As Human: Compressing Context via Parallelizable Close Reading and Skimming](https://arxiv.org/abs/2602.01840) (Tang et al., 2026)
**Key takeaway:** Adaptive query-relevance-based compression (cosine similarity scoring with temperature-scaled softmax at tau=0.1, retaining top-k segments verbatim and compressing the rest into query-aware summaries) outperforms uniform compression methods like LLMLingua-2 and Activation Beacon on QA and summarization benchmarks while achieving up to 12x speedup on 16K-32K token inputs.