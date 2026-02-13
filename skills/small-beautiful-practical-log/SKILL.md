---
name: "small-beautiful-practical-log"
description: "Build efficient log parsing systems that extract structured templates from raw log messages using a dual-cache architecture with LLM-powered correction -- optimized for small/local models. Use when the user says 'parse logs', 'extract log templates', 'build a log parser', 'log analysis pipeline', 'structure unstructured logs', or 'log template mining'."
---

# EFParser-Style Log Parsing: Efficient Template Extraction with Small LLMs

This skill enables Claude to build log parsing systems based on the EFParser architecture (FSE'26). The core technique transforms raw, unstructured log messages into structured templates by separating constant text from dynamic variables -- using a dual-cache system, adaptive template merging, and a three-stage correction module. The approach is specifically designed to work with small, resource-efficient LLMs (4B-14B parameters) while matching or exceeding the accuracy of systems that require large-scale models, making it practical for on-premise deployment where data privacy and compute constraints matter.

## When to Use

- When the user needs to parse raw log files into structured templates (e.g., turning `Connection from 192.168.1.5 port 22` into `Connection from <*> port <*>`)
- When building a log analysis pipeline that must run on local/small models (Gemini-Flash-8B, LLaMA-3-8B, Qwen-4B/8B) due to privacy or cost constraints
- When the user has high-volume logs (millions of lines) and needs an efficient parser that avoids redundant LLM calls
- When existing log parsers like Drain or Brain produce too many incorrect templates on variable-heavy log sources
- When the user wants to implement log grouping, anomaly detection, or monitoring that depends on accurate template extraction upstream
- When refactoring a log parsing system to reduce API costs while maintaining accuracy

## Key Technique

**The Problem.** Log parsing separates each raw log message into a constant *template* (the code-defined structure) and *dynamic variables* (runtime values like IPs, paths, timestamps). Traditional syntax-based parsers (Drain, AEL) use heuristics that break on complex variable patterns. LLM-based parsers achieve better generalization but require large models (70B+) -- smaller models suffer "performance collapse" where they hallucinate templates, over-generalize constants into wildcards, or produce format errors.

**The EFParser Solution.** Three architectural innovations close the gap:

1. **Dual-Cache System**: A *tree cache* (prefix tree) enables O(n) lookup for incoming logs against known templates. A *bucket cache* groups templates by token count, enabling global similarity search during updates. Both caches store the same templates but serve complementary purposes -- speed for matching, comprehensiveness for merging. Since even datasets with tens of millions of logs rarely exceed 5,000 unique templates, memory overhead is negligible.

2. **Adaptive Updating with Sequential Assessment**: When a new template is generated, edit-distance similarity identifies candidate matches in the bucket cache. Candidates undergo three sequential checks before merging: (a) *structural* -- do differences occur only at wildcard positions? (b) *token composition* -- do differing tokens share data types (alphabetic, numeric, alphanumeric)? (c) *grammatical* -- are differing tokens plausible variables (not verbs)? This prevents incorrect merges that would corrupt the cache.

3. **Three-Stage Correction Module**: Every LLM-generated template passes through validation before caching: (a) *format correction* -- ensures the template actually matches the source log by token alignment; (b) *over-specific correction* -- detects variables incorrectly left as constants (e.g., repeated special-character patterns like file paths); (c) *over-general correction* -- reverts constants incorrectly replaced with wildcards by checking against an English vocabulary (spaCy). This gatekeeper prevents error injection that would cascade through the cache.

## Step-by-Step Workflow

1. **Preprocess raw logs**: Strip timestamps, log levels, and PID prefixes using regex. Tokenize each log line by whitespace and common delimiters (`:`, `=`, `/`). Normalize obvious variables (IP addresses, hex strings, UUIDs) to `<*>` placeholders before any LLM call to reduce noise.

2. **Initialize dual-cache data structures**: Create a prefix tree (trie) for the tree cache, keyed on the first N tokens of each template. Create a dictionary keyed by token count for the bucket cache. Both start empty.

3. **For each incoming log, query the tree cache first**: Walk the prefix tree matching log tokens to template tokens (treating `<*>` as a wildcard match). If a template matches completely, assign the log to that template group and skip the LLM. This avoids redundant model calls for recurring patterns.

4. **Select exemplars for LLM prompting**: When no cache hit occurs, pick up to 3 reference logs that balance similarity (edit distance to target) and diversity (coverage of different variable patterns). Include these as few-shot demonstrations in the prompt.

5. **Call the LLM to generate a candidate template**: Prompt the model to replace dynamic variable tokens with `<*>` while preserving constant structure. Use temperature=0 for deterministic output. The prompt should instruct: "Given the log message, extract the log template by replacing variable parts with `<*>`. Keep all constant words, punctuation, and structure intact."

6. **Run the three-stage correction module on the candidate**:
   - *Format check*: Tokenize both the candidate template and original log. Verify token counts match (accounting for `<*>`). If mismatched, realign by replacing mismatched positions in the original log with `<*>`.
   - *Over-specific check*: Scan for tokens with repeated special characters (paths like `/var/log/app.log`, URLs, qualified names). Use pattern matching or a secondary LLM call to identify these as variables and replace with `<*>`.
   - *Over-general check*: For each `<*>` in the candidate, check if the original token exists in a standard English dictionary (e.g., spaCy's vocabulary). If it does and is a verb or common noun, revert it to the constant.

7. **Search the bucket cache for mergeable templates**: Retrieve all templates whose token count falls within the range `ceil(0.75 * L) <= count <= floor(L / 0.75)` where L is the candidate's token count. Compute edit-distance similarity against each. For candidates above the 0.75 threshold, run the three sequential assessments (structural, token composition, grammatical).

8. **Merge or insert the template**: If a merge partner passes all three assessments and has equal token count, do fast token-by-token merge (replace differing tokens with `<*>`). If token counts differ, use LCS (longest common subsequence) to identify shared delimiters, partition into segments, and merge segment-by-segment. If no partner qualifies, insert as a new template.

9. **Update both caches**: Insert or update the template in both the tree cache and bucket cache. If a merge occurred, remove the old template entry from both caches and reassign any logs previously grouped under it.

10. **Export results**: Output the final template list with group assignments. Each log line maps to exactly one template. Provide both the template string and extracted variable values for downstream consumption (anomaly detection, monitoring dashboards, search indexing).

## Concrete Examples

**Example 1: SSH Log Parsing**

User: "I have a file with 50,000 sshd log lines. Build me a parser that extracts templates."

Approach:
1. Preprocess: strip syslog headers (`Jan 30 12:05:33 server01 sshd[12345]:`)
2. Initialize empty dual caches
3. First log: `Accepted publickey for root from 10.0.0.1 port 52341 ssh2`
   - Tree cache miss -> select exemplars (none yet) -> LLM generates: `Accepted publickey for <*> from <*> port <*> ssh2`
   - Correction: format OK, no over-specific issues, "Accepted"/"publickey"/"port"/"ssh2" are constants -> passes
   - Insert into both caches
4. Next log: `Accepted password for admin from 172.16.0.5 port 49222 ssh2`
   - Tree cache miss (prefix "Accepted password" differs from "Accepted publickey")
   - LLM generates: `Accepted password for <*> from <*> port <*> ssh2`
   - Bucket cache match (same token count, similarity 0.875)
   - Sequential assessment: structural (differ at token 2: "publickey" vs "password"), token composition (both alphabetic), grammatical (neither is a verb -> merge allowed)
   - Merged template: `Accepted <*> for <*> from <*> port <*> ssh2`
5. Subsequent matching logs hit tree cache directly -- no LLM calls needed

Output:
```
Template 1: Accepted <*> for <*> from <*> port <*> ssh2
  -> 23,412 log lines matched

Template 2: Failed password for invalid user <*> from <*> port <*> ssh2
  -> 18,901 log lines matched

Template 3: Connection closed by authenticating user <*> <*> port <*> [preauth]
  -> 7,687 log lines matched
```

**Example 2: Building a Correction Module for an Existing Parser**

User: "My Drain-based parser keeps merging `User logged in` and `User logged out` into one template. How do I fix this?"

Approach:
1. The problem is over-general merging -- "in" and "out" are being treated as variables
2. Implement the grammatical assessment check from EFParser:
   - Before merging two templates, check each differing token pair
   - "in" and "out" are both common English words in spaCy's vocabulary
   - They function as adverbs/prepositions, not variables
   - Block the merge: keep as two separate templates
3. Add the over-general correction to the post-LLM pipeline

Output (Python implementation):
```python
import spacy
nlp = spacy.load("en_core_web_sm")

def should_merge_tokens(token_a: str, token_b: str) -> bool:
    """Grammatical assessment: block merge if tokens are real English words."""
    doc_a = nlp(token_a)
    doc_b = nlp(token_b)
    # If both tokens are recognized vocabulary words, don't merge
    if doc_a[0].is_alpha and doc_b[0].is_alpha:
        if doc_a[0].has_vector and doc_b[0].has_vector:
            return False  # These are constants, not variables
    return True

# Result: "User logged in" and "User logged out" stay as separate templates
```

**Example 3: Cost-Efficient Log Parsing Pipeline**

User: "We process 10M logs/day but can't send them to an external API. Set up local parsing with a small model."

Approach:
1. Deploy Gemini-1.5-Flash-8B or LLaMA-3-8B locally (fits in 16GB RAM)
2. Implement the dual-cache architecture to minimize model calls:
   - After initial warm-up (~500 unique logs), the tree cache intercepts 95%+ of incoming logs
   - Only genuinely novel patterns trigger LLM inference
3. Add the three-stage correction module to compensate for the small model's tendency to produce format errors and over-generalizations
4. Set similarity threshold to 0.75, max demonstrations to 3, temperature to 0

Output (architecture):
```
Raw Logs (10M/day)
  |
  v
[Preprocessor] -- strip headers, normalize IPs/UUIDs
  |
  v
[Tree Cache Lookup] -- O(n) prefix match
  |            |
  HIT          MISS
  |            |
  v            v
[Assign     [Exemplar Selector] -- pick 3 diverse refs
 Group]        |
               v
            [Local LLM] -- 8B model, temp=0
               |
               v
            [Correction Module]
            ├─ Format check
            ├─ Over-specific check (path/URL detection)
            └─ Over-general check (vocab lookup)
               |
               v
            [Bucket Cache Merge] -- similarity >= 0.75
               |
               v
            [Update Both Caches]
```

Expected: ~200 LLM calls/day after warm-up (for novel templates), vs 10M calls without caching.

## Best Practices

- **Do**: Implement the correction module even if using a capable model. Format errors and over-generalizations occur with all model sizes -- the correction module catches ~15% of generated templates that contain errors.
- **Do**: Use the dual-cache from the start, even for small datasets. The tree cache pays for itself immediately by eliminating redundant LLM calls on recurring log patterns.
- **Do**: Set temperature to 0 for deterministic template extraction. Non-zero temperature causes the same log to produce different templates across runs, corrupting cache consistency.
- **Do**: Pre-normalize obvious variables (IPs, UUIDs, hex strings, timestamps) with regex before sending to the LLM. This reduces the model's task complexity and improves accuracy on small models significantly.
- **Avoid**: Merging templates without the three sequential assessments (structural, token composition, grammatical). Naive similarity-based merging is the primary source of template corruption.
- **Avoid**: Using a similarity threshold below 0.7 -- it causes aggressive merging that collapses distinct templates. The paper's default of 0.75 is well-calibrated across 14 diverse log sources.

## Error Handling

- **LLM returns empty or malformed template**: Fall back to treating the entire log message as a single template with no variables. Flag it for re-processing in the next batch.
- **Template fails format correction** (token count mismatch after alignment): Reconstruct the template directly from the original log by replacing only regex-detected variables (IPs, numbers, paths) with `<*>`. This is less accurate but always produces a valid template.
- **Cache grows unexpectedly large** (>5,000 templates for a single log source): This signals the similarity threshold is too high or the correction module is too aggressive. Lower the threshold by 0.05 increments and re-run merging on the existing cache.
- **spaCy vocabulary check produces false positives** (legitimate variables happen to be English words like "error", "null"): Maintain a domain-specific allowlist of tokens that should always be treated as constants (HTTP methods, log levels, common keywords from the application's codebase).

## Limitations

- The approach assumes log messages have consistent delimiters (spaces, colons, equals signs). Logs with freeform natural language messages (e.g., user comments embedded in logs) will produce poor templates.
- The grammatical assessment relies on English vocabulary. Logs containing non-English constants or domain-specific jargon may be incorrectly classified as variables. Supplement with a domain dictionary.
- Small LLMs (4B-8B) still struggle with extremely long log lines (>200 tokens). Consider truncating or chunking very long messages.
- The dual-cache assumes templates are relatively stable over time. In systems with frequent code deployments that change log formats, the cache may need periodic invalidation or an aging/eviction policy.
- The correction module adds latency per novel template (~50ms for spaCy + regex checks). This is negligible when the cache hit rate is high but can bottleneck during cold-start ingestion of highly diverse log sources.

## Reference

**Paper**: "Small is Beautiful: A Practical and Efficient Log Parsing Framework" by Minxing Wang and Yintong Huo (FSE'26). arXiv: [2601.22590](https://arxiv.org/abs/2601.22590v1). Read for: the dual-cache data structure design, the three sequential merge assessments (Section 3-4), the correction module algorithm (Algorithm 1), and ablation results showing each component's contribution to accuracy.