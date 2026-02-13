---
name: "identifying-concurrency-bug-reports"
description: "Classify bug reports as concurrency-related using a four-level linguistic pattern taxonomy (word, phrase, sentence, report-level). Use when asked to 'triage concurrency bugs', 'find race condition reports', 'classify bug reports for threading issues', 'detect deadlock-related issues', 'filter concurrency bugs from issue tracker', or 'label threading bug reports'."
---

# Identifying Concurrency Bug Reports via Linguistic Patterns

This skill enables Claude to automatically classify bug reports, GitHub issues, and Jira tickets as concurrency-related or not, using a hierarchical linguistic pattern framework derived from 730 manually labeled concurrency bug reports across 12 open-source projects. The technique applies 58 patterns organized at four levels — word, phrase, sentence, and bug report — to achieve 91-93% precision on real-world issue trackers, far exceeding keyword-only matching (12% precision).

## When to Use

- When the user asks to triage or classify a batch of bug reports for concurrency issues
- When analyzing a GitHub issue or Jira ticket to determine if it describes a race condition, deadlock, or thread-safety problem
- When building an automated pipeline to label concurrency bugs in an issue tracker
- When reviewing bug reports to prioritize threading-related defects for a concurrent system
- When the user wants to write prompts or fine-tuning data for concurrency bug classification models
- When filtering noise from issue trackers to surface genuine concurrency defects among thousands of reports

## Key Technique

The core insight is that simple keyword matching ("deadlock", "race condition") produces extremely low precision (~12%) because concurrency terms appear in non-bug contexts (feature requests, documentation, configuration). The paper solves this by layering four progressively stricter pattern levels. **Word-level** patterns identify 23 concurrency-specific keywords across two categories: Concurrency Bug terms (deadlock, livelock, race condition, data race, atomicity violation, order violation, thread-unsafe, starvation, etc.) and Concurrency Mechanism terms (lock, thread, mutex, semaphore, writelock, transaction, etc.). **Phrase-level** patterns combine these with action/symptom verbs as bigrams and trigrams (e.g., "acquire lock", "thread deadlock", "lock timeout"), requiring co-occurrence within a sentence and appearing in >10% of known concurrency bug sentences.

**Sentence-level** patterns capture semantic roles: a "Lock Action" sentence describes acquiring/releasing locks ("I am trying to acquire a fair lock"), while a "Lock Symptom" sentence describes failure modes ("The system hangs when it tries to acquire a lock"). The framework defines 17 such semantic templates. **Bug report-level** patterns (6 total) assess whether concurrency-related sentences actually describe the root cause rather than incidental context — e.g., a "Lock Issue" report has concurrency sentences in the problem description section, not just in environment details or reproduction steps.

The key actionable strategy is to apply these levels as a filtering pipeline: cast a wide net at the word level, narrow with phrase co-occurrence, verify semantic fit at the sentence level, and confirm causal relevance at the report level. When used as a prompt-engineering technique, prepending extracted patterns before the bug report text in a structured format dramatically improves LLM classification accuracy.

## Step-by-Step Workflow

1. **Extract the bug report text.** Collect the title, description body, and any comment threads. Normalize formatting: strip code blocks, stack traces, and markdown artifacts to isolate natural language sentences.

2. **Apply word-level scanning.** Search for the 23 core keywords in two groups:
   - **Concurrency Bug terms (CBG):** deadlock, livelock, race condition, data race, atomicity violation, order violation, thread-unsafe, starvation, lock contention, priority inversion
   - **Concurrency Mechanism terms (CME):** lock, thread, mutex, semaphore, monitor, writelock, readlock, synchronized, atomic, transaction, barrier, latch, condition variable
   If zero CBG or CME terms are found, classify as non-concurrency (high confidence). If found, proceed.

3. **Apply phrase-level filtering.** For each sentence containing a CBG or CME term, check for co-occurrence patterns:
   - CBG + CME noun ("thread deadlock", "lock race condition")
   - CME + action verb ("acquire lock", "release mutex", "spawn thread", "fork process")
   - CME + symptom verb ("lock hangs", "thread stuck", "blocked waiting", "frozen on lock")
   - CME + bug synonym ("lock issue", "thread error", "synchronization failure")
   - CBG + time adverb ("deadlock again", "race condition intermittently")
   Score each sentence by the number of phrase patterns matched.

4. **Apply sentence-level semantic analysis.** Classify each high-scoring sentence into one of these semantic categories:
   - **Lock Action:** describes acquiring, releasing, or holding a lock
   - **Lock Symptom:** describes a lock causing hangs, blocks, or failures
   - **Thread Action:** describes thread creation, joining, scheduling, or interruption
   - **Thread Symptom:** describes threads hanging, crashing, or producing wrong results
   - **Synchronization Issue:** describes ordering, visibility, or atomicity failures
   - **Timing/Nondeterminism:** describes intermittent behavior, flakiness tied to timing
   A sentence matching one of these templates is a "concurrency-relevant sentence."

5. **Apply bug report-level contextual assessment.** Determine where concurrency-relevant sentences appear:
   - In the **problem statement** (title, first paragraph, "Problem" section) → strong signal
   - In the **reproduction steps** or **expected/actual behavior** → strong signal
   - Only in **environment info**, **configuration**, or **code references** → weak signal
   - Only in **comments** but not the original report → moderate signal
   Classify as a concurrency bug only if concurrency-relevant sentences are causally central to the reported problem.

6. **Construct a structured pattern summary.** Format the extracted evidence:
   ```
   [PATTERN:WORD] deadlock, lock, thread
   [PATTERN:PHRASE] (thread, deadlock), (lock, acquire)
   [PATTERN:SENTENCE] Lock Symptom: "The process hangs waiting to acquire the write lock"
   [PATTERN:BUG_REPORT] Root cause: Lock Issue — concurrency sentences in problem statement
   ```

7. **Produce the classification verdict.** Output one of:
   - **Concurrency Bug (High Confidence):** patterns matched at all four levels, root cause is concurrency-related
   - **Concurrency Bug (Medium Confidence):** patterns matched at word + phrase + sentence level, but report-level causality is ambiguous
   - **Likely Not Concurrency:** keywords present but no phrase/sentence/report-level patterns confirmed
   - **Not Concurrency:** no concurrency keywords found

8. **Provide the evidence chain.** List the specific patterns matched at each level so the classification is interpretable and auditable.

## Concrete Examples

**Example 1: GitHub issue — True concurrency bug**

User: "Is this bug report about a concurrency issue?"

```
Issue #4521: Application hangs on shutdown
When stopping the service, it freezes indefinitely. Thread dump shows
Thread-12 waiting to acquire the write lock on CacheManager while
Thread-7 holds the read lock and is waiting for Thread-12 to release
the connection pool lock. This only happens under heavy load.
```

Approach:
1. Word-level: Found CBG: none explicit, CME: "lock" (x4), "thread" (x3)
2. Phrase-level: "acquire the write lock" (CME+action), "holds the read lock" (CME+action), "waiting for...to release...lock" (CME+symptom), "hangs on shutdown" (symptom+context)
3. Sentence-level: Lock Symptom ("freezes indefinitely"), Lock Action ("waiting to acquire the write lock"), Thread Symptom ("Thread-12 waiting...Thread-7 holds...waiting for Thread-12") — classic lock-ordering deadlock description
4. Report-level: Concurrency sentences are the problem statement itself. "Only happens under heavy load" confirms nondeterministic, load-dependent behavior.

Output:
```
Classification: Concurrency Bug (High Confidence)
Type: Deadlock (lock-ordering violation)
Evidence:
  [WORD] lock (4), thread (3)
  [PHRASE] "acquire the write lock", "holds the read lock", "waiting for...release...lock"
  [SENTENCE] Lock Symptom + Thread Symptom: circular wait described
  [BUG_REPORT] Root cause is lock ordering — central to problem statement
  [CONTEXT] "only happens under heavy load" — classic concurrency nondeterminism
```

**Example 2: GitHub issue — False positive keyword match**

User: "Classify this issue."

```
Issue #892: Add lock timeout configuration option
Currently there's no way to configure the lock acquisition timeout
for the distributed lock service. We should add a config property
`lock.timeout.ms` with a default of 30000ms. This would allow users
to tune lock behavior for their deployment environment.
```

Approach:
1. Word-level: Found CME: "lock" (x5). No CBG terms.
2. Phrase-level: "lock acquisition timeout" — matches CME+action loosely, but "configure" and "add a config property" are feature-request language, not bug symptoms.
3. Sentence-level: No Lock Symptom, no Thread Symptom. All sentences describe desired new functionality ("should add", "would allow users to tune").
4. Report-level: No problem statement — this is a feature request. No reproduction steps, no failure described.

Output:
```
Classification: Not Concurrency Bug
Reason: Feature request for lock configuration, not a bug report.
Evidence:
  [WORD] lock (5) — present but contextually irrelevant
  [PHRASE] No symptom or failure phrases
  [SENTENCE] No symptom/failure semantic patterns — imperative/request language only
  [BUG_REPORT] No root cause described; report structure is feature request
```

**Example 3: Batch triage of multiple issues**

User: "I have 5 issues to triage for concurrency bugs. Here they are: [list]"

Approach:
1. Process each issue independently through the four-level pipeline
2. Present results as a summary table

Output:
```
| Issue   | Title                              | Classification          | Key Evidence                          |
|---------|------------------------------------|-------------------------|---------------------------------------|
| #1201   | Intermittent NPE in ConnectionPool | Concurrency (High)      | Race condition on shared pool state   |
| #1205   | Add thread pool size config        | Not Concurrency         | Feature request                       |
| #1209   | Service hangs after 2 hours        | Concurrency (Medium)    | "hang" + "thread" but no clear cause  |
| #1212   | Fix typo in lock documentation     | Not Concurrency         | Documentation fix, keyword incidental |
| #1218   | Data corruption under load         | Concurrency (High)      | "under load" + atomicity symptoms     |

Recommended for concurrency review: #1201, #1218 (high), #1209 (needs investigation)
```

## Best Practices

- **Do:** Apply all four pattern levels in sequence. Word-level alone yields ~12% precision; adding phrase, sentence, and report levels pushes precision above 90%.
- **Do:** Treat "only happens under load", "intermittent", "sometimes", "flaky" as strong concurrency signals at the sentence level — nondeterminism is a hallmark of concurrency bugs.
- **Do:** Check where concurrency terms appear in the report structure. A "deadlock" mentioned only in a code comment or log snippet pasted as context is weaker than "deadlock" in the problem title.
- **Do:** Look for causal chains involving multiple threads or processes interacting — descriptions of "Thread A waits for X while Thread B holds X" are almost always true concurrency bugs.
- **Avoid:** Classifying based solely on keyword presence. Terms like "lock", "thread", "synchronization" appear frequently in configuration, feature requests, and documentation tasks.
- **Avoid:** Treating exception names (NullPointerException, TimeoutException) as concurrency indicators without sentence-level context. These exceptions have many non-concurrency causes.

## Error Handling

- **Ambiguous reports:** If a bug report contains concurrency keywords but the description is too vague to determine causality, classify as "Medium Confidence" and recommend manual review. Flag the specific ambiguity.
- **Code-heavy reports:** When a report is mostly stack traces or code with minimal natural language, extract method names and class names for word-level scanning, but lower confidence since sentence-level and report-level patterns cannot be reliably applied.
- **Non-English reports:** The 58 patterns are derived from English-language reports. For non-English issues, attempt to identify translated equivalents of core CBG/CME terms but note reduced reliability.
- **Mixed bug reports:** Some issues describe multiple problems, only one of which is concurrency-related. Flag these and identify which section contains the concurrency component.

## Limitations

- The pattern taxonomy is derived from Java-ecosystem projects (Hadoop, HBase, gRPC-Java, etc.). Concurrency idioms in Go (goroutine/channel), Rust (ownership/Send/Sync), or Erlang (actor model) use different vocabulary and may need pattern adaptation.
- Reports that describe concurrency bugs without using any standard concurrency terminology will be missed. Extremely terse reports ("app freezes sometimes") lack enough linguistic signal for confident classification.
- The technique classifies whether a report *describes* a concurrency bug, not whether the underlying defect is truly concurrency-related. A reporter may misattribute a memory leak as a deadlock.
- Performance degrades on reports that are primarily code snippets, logs, or structured data with minimal natural language prose.

## Reference

**Paper:** Shao, Xiao, Yu. "Identifying Concurrency Bug Reports via Linguistic Patterns." arXiv:2601.16338v1 (2026). https://arxiv.org/abs/2601.16338v1

Look for: Table 2 (full keyword taxonomy with 10 word types), Table 3 (phrase and sentence pattern definitions), the pattern-prepending format for LLM prompts (Section on prompt-based approach), and the fine-tuning input construction that concatenates `[PATTERN:*]` tags with bug report text.