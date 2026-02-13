---
name: "krone-hierarchical-modular-log"
description: "Detect anomalies in application logs using KRONE's hierarchical decomposition: parse flat log sequences into Entity/Action/Status trees, build modular Krone Seqs, and apply two-stage detection (pattern matching + LLM reasoning). Use when: 'analyze these logs for anomalies', 'detect failures in system logs', 'build a log anomaly detector', 'find unusual patterns in server logs', 'hierarchical log analysis', 'parse logs into execution hierarchies'."
---

# KRONE: Hierarchical and Modular Log Anomaly Detection

This skill enables Claude to apply the KRONE framework for detecting anomalies in system and application logs. Instead of treating logs as flat sequences (where state-of-the-art methods miss true dependencies and learn spurious correlations), KRONE automatically derives a three-level execution hierarchy (Entity > Action > Status) from log templates, decomposes log sequences into modular "Krone Seqs" at each level, and runs a two-stage detector: fast pattern matching to filter known-normal behavior, then LLM-based semantic reasoning only on unmatched segments. This approach achieved 10+ percentage-point F1 improvements over prior methods on public benchmarks and industrial datasets from ByteDance Cloud.

## When to Use

- When the user asks to build a log anomaly detection pipeline for system logs (e.g., HDFS, BGL, cloud infrastructure)
- When the user has flat log files and wants to uncover execution structure and detect failures
- When the user wants to reduce LLM costs in log analysis by filtering normal patterns first
- When the user needs interpretable anomaly explanations tied to specific components, actions, or status transitions
- When the user is debugging intermittent failures and wants to identify which component/action/status sequence deviates from normal
- When the user asks to design a hierarchical log parsing system that captures entity-action-status semantics

## Key Technique

**The Core Insight:** Logs originate from nested component executions with clear boundaries, but this structure is lost when stored as flat sequences. KRONE recovers it by building a "Krone Tree"—a three-level rooted tree where the root connects to all unique Entities (software components like "block", "datanode"), each Entity links to its Actions ("receiving", "writing", "deleting"), and each Action connects to its Statuses ("started", "completed", "failed"). This tree is constructed once per application by extracting Entity/Action/Status triples from log templates using NER-style LLM extraction.

**Modular Decomposition:** Given a log sequence, KRONE decomposes it top-down into three types of "Krone Seqs"—collapsed sibling-node lists sharing a common parent. A Krone E-seq captures transitions among components (e.g., DataNode -> NameNode -> DataNode). For each entity span, a Krone A-seq captures the action sequence under that entity. For each action span, a Krone S-seq captures state transitions. This decomposition runs in O(n) time and achieves 4x-117x cardinality reduction versus raw sequences.

**Two-Stage Detection with Early Exit:** Each Krone Seq is first checked against a knowledge base of known-normal patterns (exact matching or automaton). If it matches, it's normal—no further work. If it doesn't match, it's routed to an LLM-based Nested-Aware Detector that uses bottom-up summarization and in-context learning with retrieved similar-normal examples. Detection runs bottom-up (Status -> Action -> Entity), and execution stops as soon as any Krone Seq is flagged anomalous (early exit), reducing LLM usage to 1-3% of test data. Repeated Krone Seqs are cached, yielding 5x-876x reuse savings.

## Step-by-Step Workflow

1. **Parse raw logs into templates using Drain.** Apply the Drain log parser to extract static templates with placeholders (e.g., `Received block <*> of size <*> from <*>`). Each log line maps to a template ID ("log key"). If Drain is unavailable, use regex-based template extraction or an LLM to normalize variable parts.

2. **Extract Entity/Action/Status triples from each template.** For every unique template, identify: the Entity (noun phrase — the component or resource, e.g., "block"), the Action (verb phrase — the operation, e.g., "receiving"), and the Status (adjective/noun — the outcome, e.g., "started"). Use an LLM with few-shot NER examples. Run a refinement pass to normalize entities and actions against a growing pool for consistency.

3. **Build the Krone Tree.** Construct a three-level rooted tree: Root -> Entities -> Actions -> Statuses. Use hashmap-based indexing for O(|T|) construction where |T| is the number of unique templates. Each template maps to exactly one leaf path in the tree.

4. **Group logs into sequences.** Group log lines by a natural execution boundary: session ID, request ID, block ID, node ID, or use sliding windows if no ID is available. Each group becomes one log sequence to classify.

5. **Decompose each sequence into Krone Seqs (TopDownSeqDecompose).** Map each log key to its Entity node to produce the Krone E-seq (consecutive-duplicate-collapsed entity list). For each contiguous entity span, map log keys to Action nodes to produce Krone A-seqs. For each action span, map to Status nodes to produce Krone S-seqs. Track the underlying log chunks for each Krone Seq.

6. **Build training knowledge bases from normal-only data.** Store all Krone Seqs observed in normal training sequences into level-specific knowledge bases (one for E-seqs, one for A-seqs, one for S-seqs). Also generate and store LLM summaries of each Krone Seq for later retrieval.

7. **Detect anomalies bottom-up: Status -> Action -> Entity.** For each test sequence, start at the Status level. For each test Krone S-seq, check if it exists in the S-seq knowledge base (exact match or automaton). If it matches, mark normal. If not, retrieve top-k most similar normal S-seqs by embedding similarity, format an LLM prompt with these as demonstrations, and ask the LLM for a binary prediction plus explanation.

8. **Apply early exit.** If any Krone Seq at any level is flagged anomalous, immediately label the entire sequence as anomalous and stop. Otherwise, proceed to the next level up.

9. **Cache results for reuse.** Store every test Krone Seq's prediction, explanation, summary, and embedding in a test knowledge base. When the same Krone Seq reappears (common due to repetitive log patterns), retrieve cached results directly instead of re-invoking the LLM.

10. **Aggregate and report.** Output per-sequence binary predictions with explanations tracing the anomaly to a specific hierarchy level, entity, action, and status transition.

## Concrete Examples

**Example 1: HDFS Log Anomaly Detection**

User: "I have HDFS DataNode logs. Help me detect anomalous block operations."

Approach:
1. Parse logs with Drain to extract templates like `Receiving block blk_<*> src: <*> dest: <*>` and `writeBlock blk_<*> received exception <*>`.
2. Extract triples: Template "Receiving block..." -> Entity: `block`, Action: `receiving`, Status: `started`. Template "writeBlock...exception" -> Entity: `block`, Action: `writing`, Status: `exception`.
3. Build Krone Tree:
   ```
   root
   ├── block
   │   ├── receiving -> [started, completed]
   │   ├── writing -> [started, completed, exception]
   │   └── deleting -> [started, completed]
   └── datanode
       ├── serving -> [started, completed]
       └── registering -> [completed, failed]
   ```
4. Group logs by block ID. Decompose into Krone Seqs.
5. A normal block sequence might produce E-seq: `[block]`, A-seq: `[receiving, writing]`, S-seq under receiving: `[started, completed]`.
6. An anomalous sequence might produce S-seq under writing: `[started, exception]` — not found in normal knowledge base, flagged by pattern matcher.

Output:
```
Sequence blk_-1608999687: ANOMALOUS
  Level: Status (S-seq)
  Entity: block | Action: writing
  Observed: [started, exception]
  Expected patterns: [started, completed], [started, interrupted, started, completed]
  Explanation: Block write operation terminated with exception instead of completing normally.
```

**Example 2: Cloud Infrastructure Failure Detection**

User: "Analyze these cloud VM provisioning logs for failures."

Approach:
1. Parse logs, extract templates for VM lifecycle events.
2. Build hierarchy:
   ```
   root
   ├── vm
   │   ├── provisioning -> [requested, scheduled, started, completed, failed, timeout]
   │   ├── networking -> [configuring, connected, error]
   │   └── storage -> [attaching, attached, detached, error]
   └── hypervisor
       ├── allocating -> [started, completed, insufficient_resources]
       └── migrating -> [started, completed, aborted]
   ```
3. Group by request/flow ID. Decompose into Krone Seqs.
4. Normal E-seq: `[hypervisor, vm, vm]` (allocate, then provision, then configure network).
5. Anomalous E-seq: `[vm, hypervisor, vm, hypervisor, vm]` (unusual back-and-forth between VM and hypervisor suggests retry loops).
6. Pattern matcher flags the E-seq. LLM confirms: "Repeated hypervisor-VM transitions indicate resource allocation retry loop, likely due to capacity constraints."

Output:
```
Sequence req_20240315_0042: ANOMALOUS
  Level: Entity (E-seq)
  Observed: [vm, hypervisor, vm, hypervisor, vm]
  Nearest normal: [hypervisor, vm, vm]
  Explanation: Repeated entity transitions between hypervisor and vm suggest a resource allocation retry loop not seen in normal provisioning flows.
```

**Example 3: Building the Detection Pipeline in Python**

User: "Help me implement a KRONE-style detector for my application logs."

Approach:
1. Scaffold the core data structures:
```python
from dataclasses import dataclass, field
from collections import defaultdict

@dataclass
class KroneNode:
    name: str
    level: str  # "entity", "action", "status"
    children: dict = field(default_factory=dict)

@dataclass
class KroneSeq:
    level: str
    nodes: list[str]         # collapsed node sequence
    parent_entity: str = ""
    parent_action: str = ""
    log_chunks: list = field(default_factory=list)

class KroneTree:
    def __init__(self):
        self.root = KroneNode("root", "root")
        self.template_map = {}  # template_id -> (entity, action, status)

    def add_template(self, tid, entity, action, status):
        self.template_map[tid] = (entity, action, status)
        if entity not in self.root.children:
            self.root.children[entity] = KroneNode(entity, "entity")
        e_node = self.root.children[entity]
        if action not in e_node.children:
            e_node.children[action] = KroneNode(action, "action")
        a_node = e_node.children[action]
        if status not in a_node.children:
            a_node.children[status] = KroneNode(status, "status")
```

2. Implement top-down sequence decomposition:
```python
def decompose(log_keys: list[str], tree: KroneTree) -> tuple[KroneSeq, list[KroneSeq], list[KroneSeq]]:
    triples = [tree.template_map[k] for k in log_keys]

    # Build E-seq (collapse consecutive duplicate entities)
    e_seq_nodes = [triples[0][0]]
    for t in triples[1:]:
        if t[0] != e_seq_nodes[-1]:
            e_seq_nodes.append(t[0])
    e_seq = KroneSeq(level="entity", nodes=e_seq_nodes)

    # Build A-seqs per entity span and S-seqs per action span
    a_seqs, s_seqs = [], []
    # ... group by entity spans, then by action spans
    return e_seq, a_seqs, s_seqs
```

3. Implement two-stage detection:
```python
class KroneDetector:
    def __init__(self):
        self.normal_kb = {"entity": set(), "action": set(), "status": set()}
        self.cache = {}

    def train(self, normal_sequences):
        for seq in normal_sequences:
            e, a_list, s_list = decompose(seq, self.tree)
            self.normal_kb["entity"].add(tuple(e.nodes))
            for a in a_list: self.normal_kb["action"].add(tuple(a.nodes))
            for s in s_list: self.normal_kb["status"].add(tuple(s.nodes))

    def detect(self, log_keys) -> tuple[bool, str]:
        e, a_list, s_list = decompose(log_keys, self.tree)
        # Bottom-up: check S-seqs first (early exit)
        for s in s_list:
            key = tuple(s.nodes)
            if key in self.cache:
                if self.cache[key]["anomalous"]:
                    return True, self.cache[key]["explanation"]
                continue
            if key not in self.normal_kb["status"]:
                result = self._llm_detect(s)  # LLM fallback
                self.cache[key] = result
                if result["anomalous"]:
                    return True, result["explanation"]
        # Then A-seqs, then E-seq...
        return False, "Normal"
```

## Best Practices

**Do:**
- Extract the Krone Tree once per application, then reuse across all sequences. The tree is application-specific, not sequence-specific.
- Collapse consecutive duplicates when building Krone Seqs. `[A, A, A, B, B, A]` becomes `[A, B, A]`. This captures transitions, not repetitions.
- Run detection bottom-up (Status -> Action -> Entity) with early exit. Most anomalies manifest at the finest granularity and this minimizes LLM calls.
- Cache LLM results aggressively. Log patterns are highly repetitive — KRONE found 5x-876x reuse across datasets.
- Use normal-only training data (one-class setting). KRONE is designed for the realistic scenario where only normal logs are labeled.

**Avoid:**
- Do not skip the Entity/Action/Status refinement pass. Without normalization, you get inconsistent hierarchies (e.g., "block_receive" vs "receiving block" as separate entities).
- Do not treat the entire log sequence as input to an LLM. This wastes tokens on normal segments and loses structural context. Always decompose first.
- Do not use anomaly score thresholds. KRONE outputs binary predictions per Krone Seq — threshold tuning is unnecessary and harmful to precision.
- Do not apply this to logs without recurring templates (e.g., free-form debug messages). The hierarchy depends on structured, repeating log patterns.

## Error Handling

- **Drain parsing failures on noisy logs:** Fall back to regex-based template extraction or increase Drain's similarity threshold. Ensure at least 80% of log lines map to a template.
- **LLM extraction produces inconsistent triples:** Run the refinement pass that selects from an existing entity/action pool. If the pool is empty (cold start), manually label 10-20 templates as seed examples.
- **Knowledge base is too sparse (small training set):** Use automaton-based matching instead of exact matching at the Entity level to allow for unseen-but-plausible patterns. Exact matching works best at Status level where vocabulary is small.
- **Early exit flags too aggressively (low precision):** This typically means the pattern matching stage is too strict. Expand the normal knowledge base with more training data, or skip directly to LLM reasoning for the problematic level.
- **LLM rate limits during detection:** The caching and early-exit strategies reduce LLM calls to 1-3% of test data. If still too many, increase cache TTL and batch similar Krone Seqs into single LLM calls.

## Limitations

- Requires logs with stable, recurring templates. Free-form or highly variable log messages (e.g., stack traces, raw JSON dumps) won't produce a useful Krone Tree.
- The Entity/Action/Status extraction requires an initial LLM pass over templates, which needs manual validation for production use. Garbage-in-garbage-out applies.
- One-class training assumes all training data is normal. Contaminated training sets (containing unlabeled anomalies) will pollute the knowledge base and suppress true positives.
- Hierarchy is static per application version. If the application adds new components or log formats, the Krone Tree must be rebuilt.
- Not designed for real-time streaming detection — the sequence must be complete before decomposition. For streaming, define fixed windows.

## Reference

**Paper:** Ma, Liu, Zhang, VanNostrand, Hofmann. "KRONE: Hierarchical and Modular Log Anomaly Detection." arXiv:2602.07303v1, 2026. https://arxiv.org/abs/2602.07303v1

**What to look for:** Section 3 for the KLAM hierarchy definition and Krone Tree construction, Algorithm 1 for TopDownSeqDecompose, Section 4 for the two-stage detection architecture (Local-Context + Nested-Aware), and Tables III-VI for performance comparisons showing 10+ F1 point improvements over LogBert, DeepLog, and LLM baselines.