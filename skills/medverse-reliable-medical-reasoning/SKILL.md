---
name: "medverse-reliable-medical-reasoning"
description: "Decompose complex medical reasoning into DAG-structured parallel execution paths using Petri net theory. Improves accuracy up to 8.9% and throughput 1.7x over sequential chain-of-thought. Triggers: 'differential diagnosis reasoning', 'parallel medical reasoning', 'DAG medical analysis', 'structured clinical reasoning', 'MedVerse reasoning', 'multi-hypothesis diagnosis'"
---

# MedVerse: DAG-Structured Parallel Medical Reasoning

This skill enables Claude to decompose complex medical reasoning problems into directed acyclic graph (DAG) structures based on Petri net theory, rather than forcing all clinical logic through a single sequential chain-of-thought. By identifying independent reasoning branches (e.g., evaluating multiple differential diagnoses, analyzing parallel lab findings, assessing concurrent organ systems), the approach produces structured plans with explicit dependency annotations, executes independent branches in parallel, and merges results at convergence points — yielding more thorough and reliable medical analysis.

## When to Use

- When the user asks for a **differential diagnosis** involving multiple competing hypotheses that share evidence
- When analyzing a **complex clinical case** with findings spanning multiple organ systems or specialties
- When asked to **evaluate medical evidence** where multiple independent lines of reasoning converge on a conclusion
- When building a **clinical decision support system** that must explore parallel diagnostic or treatment paths
- When the user wants **structured medical reasoning** that makes dependency relationships between clinical facts explicit
- When implementing a **medical QA pipeline** that needs to decompose multi-step reasoning into parallelizable units
- When reasoning about **drug interactions, comorbidities, or multi-system pathology** where several independent assessments feed into a unified conclusion

## Key Technique

MedVerse models medical reasoning as a Colored Petri Net `N = (P, T, F, M₀)` where **places** (P) are clinical entities or reasoning states (e.g., "elevated troponin," "chest pain differential"), **transitions** (T) are reasoning steps that consume input evidence and produce conclusions, **flow arcs** (F) encode causal dependencies, and **initial markings** (M₀) seed the root observations. Each token carries textual history and cache references, enabling efficient memory reuse across branches. The critical insight: clinical reasoning is *not* inherently linear. A physician simultaneously considers cardiac, pulmonary, and GI causes of chest pain — MedVerse makes this parallelism explicit and structured.

The framework operates in two phases. **Phase I (Linear Planning)** uses standard autoregressive generation to produce multiple reasoning paths and consolidate them into a `<Plan>` block with topological dependency annotations. This is essential — ablation studies show that skipping linear planning and jumping directly to DAG execution *degrades* performance below baseline. **Phase II (Frontier-Based Execution)** identifies the set of enabled transitions (those whose prerequisites are all satisfied) and processes them concurrently. Fork operations share prefix context across branches; join operations merge completed branches before downstream reasoning continues.

The topology-aware attention mechanism prevents information leakage between concurrent branches (branch A's intermediate reasoning cannot contaminate branch B) while preserving full causal history at join points. Adaptive position indices assign identical starting positions to forked branches and use the maximum predecessor index at joins, maintaining logical consistency within standard transformer attention.

## Step-by-Step Workflow

1. **Extract clinical observations from the input.** Identify all stated symptoms, lab values, imaging findings, history items, and demographic factors. List each as a discrete clinical entity — these become the initial places (M₀) in the Petri net.

2. **Identify independent reasoning branches.** Determine which clinical analyses can proceed in parallel. For a differential diagnosis, each candidate diagnosis is a branch. For multi-system assessment, each organ system is a branch. Group findings by which branch they feed into.

3. **Map causal dependencies as a DAG.** For each reasoning step, specify its prerequisites explicitly. Use the notation `Step X depends on [Step A, Step B]` to encode the flow arcs. Ensure the graph is acyclic — if you find a cycle, break it by identifying which direction the causal relationship truly flows.

4. **Generate the structured plan with dependency annotations.** Produce a `<Plan>` block listing each reasoning outline with its dependencies:
   ```
   <Plan>
     <Outline id="1" depends_on="[]">Assess cardiac causes given troponin and ECG</Outline>
     <Outline id="2" depends_on="[]">Assess pulmonary causes given CXR and SpO2</Outline>
     <Outline id="3" depends_on="[]">Assess GI causes given epigastric tenderness</Outline>
     <Outline id="4" depends_on="[1,2,3]">Synthesize differential and rank by probability</Outline>
   </Plan>
   ```

5. **Validate the DAG structure.** Check: (a) all referenced dependency IDs exist, (b) no cycles exist (topological sort must succeed), (c) at least one root node has empty dependencies, (d) exactly one terminal node produces the final conclusion.

6. **Execute the first frontier of independent branches.** Process all steps whose dependencies are empty (the initial enabled transitions). For each branch, reason through the clinical evidence specific to that hypothesis, producing a `<Step>` output with findings and intermediate conclusions.

7. **Merge completed branches at join points.** When all prerequisites for a downstream step are satisfied, aggregate the conclusions from predecessor branches. At the join, the full causal history from all parent branches is available — use it to compare, contrast, and integrate findings.

8. **Continue frontier execution until the terminal node.** Repeat the frontier-identify-and-execute cycle. Each iteration processes all newly enabled transitions. Track which steps are complete, which are pending, and which are blocked.

9. **Synthesize the final conclusion from the terminal step.** The last step must integrate all completed reasoning branches into a unified answer — a ranked differential, a treatment recommendation, or a diagnostic conclusion. Explicitly cite which branch contributed which evidence.

10. **Review for logical consistency across the DAG.** Verify that no branch contradicts another without acknowledgment, that join points correctly weigh all inputs, and that the final conclusion follows from the aggregated evidence.

## Concrete Examples

**Example 1: Differential Diagnosis for Chest Pain**

User: "A 55-year-old male presents with acute substernal chest pain radiating to the left arm, diaphoresis, and nausea. Troponin is elevated at 0.8 ng/mL, ECG shows ST elevation in leads II, III, aVF, CXR shows clear lung fields, and he has a history of GERD. What is the most likely diagnosis and differential?"

Approach:

```
<Plan>
  <Outline id="1" depends_on="[]">Evaluate for acute coronary syndrome (ACS):
    Analyze ST elevation pattern, troponin level, symptom presentation</Outline>
  <Outline id="2" depends_on="[]">Evaluate for aortic dissection:
    Assess pain character, check for pulse/BP differentials, risk factors</Outline>
  <Outline id="3" depends_on="[]">Evaluate for GI etiology (GERD exacerbation):
    Consider history, symptom overlap, atypical features</Outline>
  <Outline id="4" depends_on="[1,2,3]">Rank differential by posterior probability
    given all evidence streams</Outline>
</Plan>
```

Step 1 (ACS branch): ST elevation in II, III, aVF indicates inferior STEMI. Troponin 0.8 is significantly elevated. Classic symptom triad (substernal pain + radiation + diaphoresis). **Strong positive signal.**

Step 2 (Dissection branch): No mention of tearing/ripping quality, no BP differential noted, no widened mediastinum on CXR. **Low probability but not fully excluded** — would recommend CT angiography if ACS treatment doesn't resolve.

Step 3 (GI branch): GERD history is relevant, but ST elevation and troponin are not explained by GERD. Nausea can be vagal response to inferior MI. **Very low probability as primary cause.**

Step 4 (Synthesis — joins steps 1, 2, 3): **Primary diagnosis: Inferior STEMI** (high confidence). Differential: (1) Inferior STEMI — 95%, (2) Aortic dissection — 3%, (3) GERD exacerbation — 2%. Recommend emergent cardiac catheterization.

**Example 2: Multi-System Assessment for Sepsis Workup**

User: "A 68-year-old female post-hip replacement presents with fever 39.2C, HR 112, BP 88/55, WBC 18.5k, lactate 4.2, creatinine 2.1 (baseline 0.9), urinalysis positive for nitrites and leukocyte esterase, and blood cultures pending. Assess the clinical picture."

Approach:

```
<Plan>
  <Outline id="1" depends_on="[]">Assess sepsis criteria (SIRS/qSOFA/Sepsis-3)</Outline>
  <Outline id="2" depends_on="[]">Evaluate infectious source: urinary tract</Outline>
  <Outline id="3" depends_on="[]">Evaluate infectious source: surgical site</Outline>
  <Outline id="4" depends_on="[]">Assess end-organ damage (renal, hemodynamic)</Outline>
  <Outline id="5" depends_on="[1,2,3]">Determine most likely source and sepsis severity</Outline>
  <Outline id="6" depends_on="[4,5]">Formulate management priorities</Outline>
</Plan>
```

Branches 1-4 execute concurrently. Branch 5 joins 1+2+3. Branch 6 joins 4+5.

Output:
- **Sepsis-3 criteria met**: Suspected infection + qSOFA >= 2 (hypotension + tachycardia). Lactate 4.2 indicates tissue hypoperfusion — **septic shock**.
- **Source**: Urosepsis most likely (positive UA with nitrites/LE). Surgical site infection possible but no wound findings described.
- **Organ damage**: AKI Stage 2 (creatinine >2x baseline). Hemodynamic compromise present.
- **Management**: (1) 30 mL/kg crystalloid bolus, (2) Blood cultures then empiric broad-spectrum antibiotics targeting urinary pathogens, (3) Vasopressors if MAP <65 after fluids, (4) Monitor urine output and lactate clearance.

**Example 3: Implementing a DAG-Structured Medical QA Pipeline in Code**

User: "Build me a Python module that takes a medical question and decomposes it into parallel reasoning branches using the MedVerse DAG approach."

Approach:

```python
from dataclasses import dataclass, field
from collections import defaultdict

@dataclass
class ReasoningNode:
    id: str
    description: str
    depends_on: list[str] = field(default_factory=list)
    output: str | None = None

class MedVerseDAG:
    def __init__(self):
        self.nodes: dict[str, ReasoningNode] = {}
        self.adj: dict[str, list[str]] = defaultdict(list)

    def add_node(self, node: ReasoningNode):
        self.nodes[node.id] = node
        for dep in node.depends_on:
            self.adj[dep].append(node.id)

    def get_frontier(self) -> list[str]:
        """Return all enabled transitions: nodes whose dependencies
        are all satisfied (have output) and that have no output yet."""
        frontier = []
        for nid, node in self.nodes.items():
            if node.output is not None:
                continue
            if all(self.nodes[d].output is not None for d in node.depends_on):
                frontier.append(nid)
        return frontier

    def is_complete(self) -> bool:
        return all(n.output is not None for n in self.nodes.values())

    def validate(self) -> bool:
        """Verify DAG is acyclic via topological sort."""
        in_degree = {nid: len(n.depends_on) for nid, n in self.nodes.items()}
        queue = [nid for nid, d in in_degree.items() if d == 0]
        visited = 0
        while queue:
            current = queue.pop(0)
            visited += 1
            for child in self.adj[current]:
                in_degree[child] -= 1
                if in_degree[child] == 0:
                    queue.append(child)
        return visited == len(self.nodes)


def execute_dag(dag: MedVerseDAG, reason_fn) -> str:
    """Execute the DAG frontier-by-frontier.
    reason_fn(node, predecessor_outputs) -> str"""
    if not dag.validate():
        raise ValueError("Invalid DAG: cycle detected")

    while not dag.is_complete():
        frontier = dag.get_frontier()
        if not frontier:
            raise RuntimeError("Deadlock: no enabled transitions")
        # Frontier nodes can be executed in parallel
        for nid in frontier:
            node = dag.nodes[nid]
            pred_outputs = {d: dag.nodes[d].output for d in node.depends_on}
            node.output = reason_fn(node, pred_outputs)

    # Return terminal node output (node with no successors)
    terminal = [nid for nid in dag.nodes if not dag.adj[nid]]
    return dag.nodes[terminal[0]].output
```

The `reason_fn` callback can be backed by an LLM call, a retrieval pipeline, or rule-based logic. The `get_frontier()` method implements the Petri net firing rule: a transition is enabled when all input places hold tokens.

## Best Practices

- **Do:** Always start with a linear planning phase before constructing the DAG. The paper's ablation shows skipping planning drops accuracy below baseline — the sequential pass identifies *what* branches exist before structuring *how* they relate.
- **Do:** Make dependency annotations explicit and machine-parseable. Use structured formats (`depends_on: [1,2]`) rather than prose references ("after considering the above").
- **Do:** Validate the DAG before execution — check for cycles, orphan nodes, and unreachable terminals. A malformed graph produces incomplete or contradictory reasoning.
- **Do:** At join points, actively compare and contrast findings from predecessor branches rather than just concatenating them. The join is where diagnostic discrimination happens.
- **Avoid:** Creating branches that are too granular (one finding per branch). Each branch should represent a meaningful clinical reasoning thread — typically a diagnostic hypothesis, an organ system assessment, or an evidence category.
- **Avoid:** Letting parallel branches access each other's intermediate reasoning. The topology-aware attention mechanism in MedVerse explicitly prevents this leakage. In practice, keep each branch reasoning only from its stated inputs until the join step.

## Error Handling

- **Cycle detected in DAG:** This means a bidirectional dependency was introduced (A depends on B and B depends on A). Resolve by determining the true causal direction — in clinical reasoning, ask "which finding would I need to know *first* to reason about the other?"
- **Deadlock (no enabled frontier):** All remaining nodes have unsatisfied dependencies, but no node is ready to fire. This usually indicates a missing intermediate node. Add an explicit aggregation step to bridge disconnected subgraphs.
- **Branch contradiction at join:** Two parallel branches reach conflicting conclusions. This is *expected* in differential diagnosis. The join step must explicitly acknowledge the conflict and weigh evidence strength, pretest probability, and specificity to resolve it.
- **Excessive branching:** If a plan generates more than 6-8 parallel branches, the reasoning becomes unwieldy. Consolidate related hypotheses into grouped branches (e.g., "cardiac causes" rather than separate branches for STEMI, NSTEMI, unstable angina, pericarditis).

## Limitations

- **Not suited for simple, linear clinical questions.** If the reasoning is truly sequential (e.g., "What is the mechanism of action of metformin?"), DAG decomposition adds overhead without benefit. Use standard chain-of-thought instead.
- **DAG quality depends on domain knowledge.** Incorrect dependency annotations produce incorrect reasoning structures. The planner must understand clinical causality to build valid graphs.
- **Parallel execution gains are architectural.** In a standard chat context, Claude processes branches sequentially even if structured as a DAG. The throughput gains (1.7x) from the paper require the custom inference engine with KV-cache sharing and frontier-based scheduling. The *accuracy* benefits (up to 8.9%) still apply from the structured decomposition alone.
- **Not a substitute for clinical judgment.** The framework improves reasoning structure but does not guarantee correctness. All medical reasoning outputs should be reviewed by qualified clinicians.

## Reference

**Paper:** [MedVerse: Efficient and Reliable Medical Reasoning via DAG-Structured Parallel Execution](https://arxiv.org/abs/2602.07529v2) — Chen et al., 2026. Focus on Section 3 (Petri net formalization), Section 3.3 (topology-aware attention), and Section 3.4 (inference engine) for implementation details. The MedVerse-14K dataset construction pipeline (Section 3.1) is valuable if building training data for structured medical reasoning.