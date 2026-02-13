---
name: "probing-knowledge-boundary-interactive"
description: >
  Systematically extract deep knowledge from LLMs using an interactive agentic
  framework with four adaptive exploration policies and a three-stage deduplication
  pipeline. Use this skill when the user says: "extract everything the model knows
  about X", "probe knowledge boundaries", "deep knowledge extraction", "exhaustive
  topic mining", "knowledge audit of an LLM", or "map out what you know about X".
---

This skill enables Claude to act as a systematic knowledge extraction engine. Rather than answering a question once and moving on, Claude deploys iterative exploration policies -- sequential probing, self-reflective refinement, recursive taxonomy decomposition, and multi-perspective parallel probing -- to exhaustively mine its own parametric knowledge on a topic. Extracted knowledge atoms are then deduplicated and validated through a three-stage pipeline (vector-based filtering, semantic adjudication, domain-relevance auditing) to produce a clean, comprehensive knowledge inventory. The technique is based on the interactive agentic framework from Yang et al. (2026).

## When to Use

- When the user asks to "extract everything you know about [topic]" or "give me an exhaustive breakdown of [domain]"
- When building a knowledge base or ontology from LLM outputs and completeness matters
- When comparing what different models or prompting strategies yield on a topic
- When the user needs a structured taxonomy of a technical domain (e.g., "map out all ML optimization techniques")
- When auditing an LLM's knowledge coverage before deploying it for a domain-specific task
- When the user wants to find gaps in model knowledge on a subject ("what don't you know about X?")
- When performing systematic literature-style reviews powered by parametric memory rather than retrieval

## Key Technique

**Adaptive Exploration Policies.** The framework defines four strategies for probing knowledge at different granularities. (1) *Sequential Probing* (P2): iterative "what else?" prompts conditioned on prior responses -- a baseline that tests associative retrieval depth. (2) *Self-Reflective Refinement* (P3): a critic-actor loop where the model audits its previous output for missing sub-domains or logical gaps, then generates targeted prompts to fill them. (3) *Recursive Taxonomy Explorer* (P4): the strongest strategy -- it recursively decomposes a topic into a tree governed by a branching factor W and maximum depth D_max, then exhaustively mines each leaf node. (4) *Multi-Perspective Probing* (P5): instantiates N distinct expert personas (e.g., engineer, legal scholar, ethicist) that independently contribute knowledge, with a global set synchronized across all experts each turn.

**Three-Stage Knowledge Processing Pipeline.** Raw extracted statements are noisy and redundant. Stage 1 uses embedding-based cosine similarity: pairs above 0.92 are merged as exact duplicates. Stage 2 handles the ambiguity zone (0.70-0.92 similarity) via LLM-based adjudication -- a judge determines whether two atoms describe the same core fact, accounting for logical negations and subtle technical differences. Stage 3 applies domain-relevance auditing using Bloom's Taxonomy criteria: only factual statements, conceptual knowledge (relationships, principles), and procedural knowledge (methods, algorithms) are retained; meta-statements, generic assertions, and incomplete fragments are discarded.

**Saturation Detection.** Extraction terminates automatically when knowledge growth drops below 1% per turn, extraction efficiency falls below 10%, fewer than 3 novel atoms are discovered per turn, or a maximum of 15 turns is reached. This prevents wasted computation once the model's knowledge boundary has been reached.

## Step-by-Step Workflow

1. **Define the extraction scope.** Clarify with the user the target topic, desired granularity (broad survey vs. deep technical), and any domain constraints (e.g., "only ML optimization, not general math optimization").

2. **Select the exploration policy.** Default to Recursive Taxonomy (P4) with branching factor W=5 and depth D_max=3-5 for most topics. Use Multi-Perspective (P5) when the topic spans disciplines. Use Self-Reflection (P3) for narrow, deep topics. Use Sequential (P2) only as a quick baseline.

3. **Build the initial taxonomy.** For P4, decompose the topic into W top-level subtopics. For each subtopic, recursively decompose into W children until reaching D_max. Present this taxonomy tree to the user for validation before proceeding.

4. **Mine leaf nodes exhaustively.** For each leaf node in the taxonomy, generate all known facts, concepts, and procedures as discrete "knowledge atoms" -- single verifiable statements. Tag each atom with its taxonomy path (e.g., `ML > Optimization > Second-Order Methods > L-BFGS > Memory efficiency`).

5. **Apply Stage 1: Vector-based deduplication.** Compare all extracted atoms pairwise using semantic similarity. Flag pairs with similarity > 0.92 as duplicates and merge them, keeping the more precise formulation.

6. **Apply Stage 2: Semantic adjudication.** For atom pairs in the 0.70-0.92 similarity zone, explicitly reason about whether they encode the same core fact. Merge genuine duplicates; retain both if they capture distinct information (e.g., a fact and its negation, or two related but different properties).

7. **Apply Stage 3: Domain-relevance auditing.** Filter each remaining atom against Bloom's Taxonomy criteria. Retain factual knowledge (specific data, terminology, definitions), conceptual knowledge (relationships, categories, principles), and procedural knowledge (methods, algorithms, techniques). Discard meta-statements ("this is a broad field"), generic assertions ("many approaches exist"), and fragments that cannot be independently verified.

8. **Monitor saturation.** Track the number of novel atoms per extraction turn. If growth rate < 1%, efficiency < 10%, or novel count < 3, stop extraction for that branch and report the boundary reached.

9. **Compile and structure the knowledge inventory.** Organize validated atoms into the taxonomy tree. Report total atom count, coverage per subtopic, and any identified gaps (branches where saturation was reached unusually early).

10. **Present results with metadata.** Deliver the final knowledge set as a structured document (Markdown, JSON, or both) with per-atom taxonomy paths, confidence indicators (number of extraction turns that surfaced the atom), and an explicit "knowledge boundary" section listing topics where extraction saturated quickly or yielded few atoms.

## Concrete Examples

**Example 1: Exhaustive domain knowledge extraction**

```
User: Extract everything you know about reinforcement learning exploration strategies.

Approach:
1. Select Recursive Taxonomy (P4) with W=5, D_max=4
2. Build taxonomy:
   RL Exploration
   ├── Count-Based Methods (UCB, MBIE, pseudo-counts, ...)
   ├── Intrinsic Motivation (curiosity, RND, ICM, empowerment, ...)
   ├── Thompson Sampling & Posterior Methods (PSRL, ensemble sampling, ...)
   ├── Information-Theoretic (VIME, max entropy, InfoGain, ...)
   └── Structured Exploration (options, goal-conditioned, go-explore, ...)
3. Mine each leaf node for knowledge atoms
4. Deduplicate: e.g., "UCB selects arms with highest upper confidence bound"
   and "UCB1 picks the action maximizing mean + sqrt(2 ln t / n_i)"
   → similarity 0.85 → adjudicate → merge (same core fact, keep precise version)
5. Audit: discard "Exploration is important in RL" (generic assertion)
6. Report: 247 unique knowledge atoms across 5 branches, with count-based
   methods showing early saturation (boundary reached at turn 3)

Output structure:
## RL Exploration Knowledge Inventory
- **Total atoms:** 247
- **Branch coverage:** Count-Based (38), Intrinsic (72), Thompson (41),
  Info-Theoretic (44), Structured (52)
- **Knowledge boundary:** Count-based methods saturated early (turn 3/15),
  suggesting limited parametric memory in this sub-area
- **Atoms sample:**
  - [Count-Based > UCB] UCB1 selects the arm maximizing x̄_i + √(2 ln t / n_i),
    balancing exploitation (sample mean) with exploration (confidence width)
  - [Intrinsic > RND] Random Network Distillation trains a predictor to match
    a fixed random network's outputs; prediction error serves as novelty signal
  ...
```

**Example 2: Comparative knowledge audit across sub-domains**

```
User: Map out what you know about cryptographic hash functions -- I want to
find where your knowledge is weakest.

Approach:
1. Select Recursive Taxonomy (P4) with W=4, D_max=3
2. Build taxonomy:
   Crypto Hash Functions
   ├── Construction (Merkle-Damgård, sponge, tree hashing, HAIFA)
   ├── Specific Algorithms (MD5, SHA-1, SHA-2, SHA-3, BLAKE)
   ├── Security Properties (collision, preimage, second-preimage, length-ext)
   └── Applications (HMAC, digital signatures, password hashing, commitments)
3. Mine all leaf nodes, tracking atoms-per-turn for saturation
4. Deduplicate and audit
5. Compare branch densities to identify weak spots

Output:
## Cryptographic Hash Functions — Knowledge Boundary Map
| Branch         | Atoms | Saturated at Turn | Coverage |
|----------------|-------|--------------------|----------|
| Construction   | 31    | Turn 4             | Medium   |
| Algorithms     | 58    | Turn 8             | High     |
| Security Props | 42    | Turn 6             | High     |
| Applications   | 23    | Turn 3             | Low      |

**Identified gaps:**
- Applications/password-hashing: only 4 atoms extracted (bcrypt, scrypt, Argon2
  basics). Missing: memory-hardness proofs, side-channel resistance details,
  parameter tuning guidelines
- Construction/HAIFA: 2 atoms only. Knowledge boundary reached immediately
```

**Example 3: Multi-perspective extraction for interdisciplinary topic**

```
User: I need a comprehensive knowledge dump on AI ethics in healthcare.

Approach:
1. Select Multi-Perspective (P5) with 4 expert personas:
   - Clinical Researcher: medical AI validation, clinical trials, FDA pathways
   - Ethicist: autonomy, beneficence, justice, informed consent for AI
   - ML Engineer: fairness metrics, bias mitigation, model interpretability
   - Legal Scholar: HIPAA, liability, EU AI Act, malpractice implications
2. Each persona independently generates knowledge atoms (5 turns each)
3. Synchronize: merge global knowledge set, resolve cross-persona duplicates
4. Audit: discard opinions and normative claims without factual grounding
5. Compile with persona attribution for traceability

Output:
## AI Ethics in Healthcare — Multi-Perspective Knowledge Inventory
- **Total atoms:** 189
- **Per-persona:** Clinical (52), Ethics (48), ML (51), Legal (38)
- **Cross-persona overlap:** 23 atoms appeared in 2+ perspectives
- **Knowledge boundary:** Legal perspective saturated earliest (turn 3),
  likely reflecting training data composition rather than topic simplicity
```

## Best Practices

- **Do:** Start with Recursive Taxonomy (P4) as the default -- experiments show it consistently extracts the most knowledge across domains.
- **Do:** Present the taxonomy tree to the user for validation before mining. A misaligned taxonomy wastes extraction turns.
- **Do:** Track atoms-per-turn metrics and report them. The saturation curve itself is informative -- early saturation signals a knowledge gap, not topic simplicity.
- **Do:** Use the 0.92/0.70 similarity thresholds as starting points, but adjust based on domain. Highly technical domains (math, chemistry) may need tighter thresholds (0.95/0.75) to avoid merging distinct formulas.
- **Avoid:** Treating meta-statements as knowledge. "There are many approaches to X" contains zero extractable information.
- **Avoid:** Mining beyond saturation. Once growth drops below 1%, further probing yields noise, not knowledge. Stop and report the boundary.
- **Avoid:** Using Sequential Probing (P2) for complex topics. It plateaus quickly because "what else?" prompts lack the structural guidance to reach long-tail knowledge.

## Error Handling

- **Taxonomy becomes too broad:** If a branch generates > 100 atoms before reaching D_max, increase depth and narrow the branching factor for that sub-tree. This trades breadth for precision.
- **High duplicate rate (> 50% pre-dedup):** The exploration policy is circling. Switch from Sequential (P2) to Recursive Taxonomy (P4), or increase taxonomy depth to force more specific leaf nodes.
- **Ambiguous adjudication calls:** When two atoms in the 0.70-0.92 zone resist clear merge/keep decisions, retain both and flag them as "potentially overlapping" in the output. Let the user decide.
- **Domain drift:** If atoms start appearing that fall outside the stated scope (e.g., general math facts during an ML-specific extraction), tighten the domain-relevance audit criteria and re-filter.
- **Premature saturation:** If a branch saturates in fewer than 2 turns, the taxonomy node may be too narrow. Merge it with a sibling node and re-extract.

## Limitations

- This framework extracts only parametric knowledge (what the model learned during training). It cannot surface information the model never encountered in training data.
- Knowledge atoms are limited to the model's ability to articulate knowledge as discrete statements. Tacit knowledge (e.g., implicit coding patterns) may not surface through text-based probing.
- The deduplication pipeline relies on embedding similarity, which can miss paraphrase-level duplicates in highly technical notation (e.g., two equivalent mathematical formulations).
- Saturation detection assumes monotonically decreasing novelty. Topics with "knowledge pockets" reachable only through specific framing may be missed.
- The framework cannot verify factual correctness of extracted atoms -- it ensures uniqueness and domain relevance, not truth. Pair with external fact-checking for high-stakes applications.

## Reference

Yang, Y., Zhu, S., Feng, T., Liu, G., & You, J. (2026). *Probing the Knowledge Boundary: An Interactive Agentic Framework for Deep Knowledge Extraction.* arXiv:2602.00959. https://arxiv.org/abs/2602.00959v1

Key insight: Recursive Taxonomy (P4) with branching factor W=5 and depth 3-5 consistently outperforms all other exploration strategies. The three-stage deduplication pipeline (vector filtering at 0.92, LLM adjudication at 0.70-0.92, Bloom's Taxonomy auditing) is essential for producing clean knowledge inventories.