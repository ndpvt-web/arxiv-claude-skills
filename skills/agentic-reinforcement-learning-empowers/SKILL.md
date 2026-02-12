---
name: "agentic-reinforcement-learning-empowers"
description: "Build tool-augmented agent systems that decouple domain reasoning from knowledge storage, following the ChemCRAFT pattern of sandbox interaction + reinforcement learning. Use when: 'build an agent that calls external tools instead of memorizing data', 'design a tool-calling reward function', 'create a trajectory dataset for tool-use training', 'decouple reasoning from knowledge in a small model', 'set up a sandbox for agent-tool interaction', 'train a model to orchestrate domain-specific tools with RL'."
---

# Agentic Reinforcement Learning for Tool-Orchestrating Agents

This skill teaches Claude to design and implement **agent systems where a small language model orchestrates external tools instead of memorizing domain knowledge** -- the core pattern from ChemCRAFT (arXiv:2601.17687v2). The key insight: scientific/domain reasoning is not an emergent ability of scale, but a **learnable policy of tool orchestration**. By decoupling reasoning (what to do) from knowledge storage (domain data held in external tools), a 7B-parameter model can outperform cloud-scale LLMs on specialized tasks while running locally with lower cost and full data privacy.

## When to Use

- When the user wants to build an agent that calls external APIs/tools for domain-specific tasks instead of relying on memorized parametric knowledge
- When designing a reinforcement learning reward function for tool-calling behavior (especially with structured outputs like SMILES, SQL, chemical formulas)
- When constructing a training dataset of tool-use trajectories from expert demonstrations
- When the user needs to make a small LM competitive with larger models by augmenting it with a tool sandbox
- When building a two-stage training pipeline (supervised fine-tuning on trajectories, then RL refinement)
- When implementing a "Think -> Call Tool -> Observe -> Reason" agent loop for any scientific or data-intensive domain

## Key Technique: Cognitive Decoupling via Agentic RL

ChemCRAFT resolves a fundamental tension: small models hallucinate domain facts, while large cloud models are expensive and leak private data. The solution is **cognitive decoupling** -- the model learns *when and how* to call tools, while the tools handle *what the facts are*. This is implemented through three components:

**1. Chemical-Agent Sandbox.** A suite of external tools organized into three tiers: (a) deterministic computational tools (e.g., RDKit for molecular graphs, RDChiral for stereochemistry), (b) deep learning prediction services (e.g., property predictors for QED/LogP/ADMET via PyTDC), and (c) retrieval-based agents (substructure search, reaction template matching). The model never memorizes molecular properties -- it queries the sandbox.

**2. Trajectory Construction Pipeline.** Training data is built in two phases. First, a teacher model generates raw tool-calling trajectories across a task decomposition (9 major tasks, 22 subtasks). Then a **reflective refinement** step injects verified tool outputs back into context and rewrites mechanical action logs into fluid expert-level reasoning narratives. This produces ChemToolDataset -- paired (query, reasoning-with-tool-calls, answer) examples.

**3. SMILES-GRPO (Group Relative Policy Optimization).** Instead of a binary reward, the RL stage uses a **dense, multi-dimensional reward function** combining: format compliance (syntactic correctness of tool calls), scaffold similarity (Tanimoto on Bemis-Murcko scaffolds), functional group fidelity, property improvement magnitude, and synthesis pathway validity. Advantages are normalized within sampled output groups, eliminating the need for a separate value model. A KL-divergence penalty prevents policy collapse.

## Step-by-Step Workflow

### 1. Define the Tool Sandbox

Enumerate every external tool the agent can call. For each tool, specify: (a) name, (b) input schema, (c) output schema, (d) determinism (exact vs. probabilistic), (e) latency class. Organize tools into tiers: deterministic computations, ML predictions, and retrieval/search.

```python
SANDBOX_TOOLS = {
    "compute_molecular_weight": {
        "input": {"smiles": "str"},
        "output": {"mw": "float"},
        "tier": "deterministic",
        "backend": "rdkit"
    },
    "predict_qed": {
        "input": {"smiles": "str"},
        "output": {"qed_score": "float"},
        "tier": "ml_prediction",
        "backend": "pytdc"
    },
    "search_reaction_templates": {
        "input": {"product_smiles": "str"},
        "output": {"templates": "list[dict]"},
        "tier": "retrieval",
        "backend": "reaction_db"
    }
}
```

### 2. Design the Tool-Calling Protocol

Define the structured format the model must emit to invoke tools. Use explicit delimiters that are easy to parse and reward:

```
<think>The molecule has a hydroxyl group that may affect solubility. I need to check LogP.</think>
<tool_call>{"name": "compute_logp", "args": {"smiles": "CCO"}}</tool_call>
<observation>{"logp": -0.31}</observation>
<think>LogP of -0.31 confirms high hydrophilicity. Now I should check...</think>
```

### 3. Decompose the Domain into Tasks and Subtasks

Break the target domain into a task taxonomy. ChemCRAFT uses 9 major tasks (understanding, editing, optimization, reaction prediction) with 22 subtasks. Each subtask maps to specific tool sequences. This taxonomy drives both dataset construction and evaluation.

### 4. Generate Raw Trajectories with a Teacher Model

Use a capable teacher model (e.g., GPT-4, Claude) to solve each subtask while calling sandbox tools. Capture the full trajectory: query, chain-of-thought, tool calls, observations, and final answer. Generate multiple trajectories per subtask for diversity.

### 5. Apply Reflective Refinement to Trajectories

For each raw trajectory: (a) execute all tool calls against the real sandbox to get verified outputs, (b) replace any hallucinated tool outputs with real ones, (c) prompt the teacher to rewrite the reasoning from mechanical logs into expert-level scientific narratives that explain *why* each tool was called. This transforms `Action: calc_mw; Obs: 150.2` into `I computed the molecular weight (150.2 Da) to verify this falls within Lipinski's Rule of Five range.`

### 6. Stage 1: Supervised Fine-Tuning on Trajectories

Fine-tune the base small model (7B-14B) on the refined trajectory dataset. The loss is standard next-token prediction over the interleaved sequence of reasoning tokens, tool-call tokens, and answer tokens. This cold-start SFT establishes a stable initial policy (pi_ref) that can already call tools.

### 7. Design the Multi-Dimensional Reward Function

Build a dense reward combining multiple signals. Each component should be independently computable:

```python
def compute_reward(generated, reference, tool_calls):
    r_format = 1.0 if all_tool_calls_syntactically_valid(tool_calls) else 0.0
    r_scaffold = tanimoto_similarity(get_scaffold(generated), get_scaffold(reference))
    r_functional = functional_group_fidelity(generated, reference)
    r_property = property_improvement(generated, reference)  # delta QED, delta LogP
    r_synthesis = synthesis_validity(generated)  # template alignment score
    return w1*r_format + w2*r_scaffold + w3*r_functional + w4*r_property + w5*r_synthesis
```

### 8. Stage 2: GRPO Reinforcement Learning

Sample groups of K outputs per prompt. Compute rewards for each, normalize advantages within the group (no value model needed). Update policy to maximize:

```
L = E[min(r * A, clip(r, 1-eps, 1+eps) * A)] - beta * KL(pi || pi_ref)
```

Where `r` is the probability ratio, `A` is the group-normalized advantage, and beta controls the KL penalty against the SFT checkpoint.

### 9. Evaluate on Held-Out Task Taxonomy

Test across all subtasks. Key metrics: tool-call accuracy (did the model call the right tools?), answer correctness, token efficiency (ChemCRAFT achieves 65% fewer tokens than multi-agent baselines), and property improvement magnitude for optimization tasks.

### 10. Deploy as a Local Agent Loop

Package the trained model with the sandbox as a self-contained service. The inference loop is: receive query -> model generates think/tool_call tokens -> parse tool calls -> execute against sandbox -> inject observations -> model continues until final answer.

## Concrete Examples

**Example 1: Building a SQL tool-calling agent (applying the pattern to databases)**

User: "I want a small LM that answers business questions by querying a database instead of memorizing data."

Approach:
1. Define sandbox tools: `execute_sql`, `list_tables`, `describe_table`, `get_sample_rows`
2. Create task taxonomy: single-table lookup, multi-table join, aggregation, date filtering, subquery
3. Generate 5K trajectories with a teacher model solving NL-to-SQL with tool calls
4. Refine trajectories: verify SQL executes correctly, rewrite reasoning to explain join logic
5. SFT a 7B model on trajectories, then GRPO with rewards: SQL validity (parses?), execution success (runs without error?), result correctness (matches gold?)

Output:
```python
# Reward function for SQL agent
def sql_reward(generated_sql, gold_answer, db_connection):
    r_parse = 1.0 if sqlparse.parse(generated_sql) else 0.0
    try:
        result = db_connection.execute(generated_sql)
        r_exec = 1.0
        r_correct = 1.0 if result == gold_answer else 0.0
    except Exception:
        r_exec, r_correct = 0.0, 0.0
    return 0.2 * r_parse + 0.3 * r_exec + 0.5 * r_correct
```

**Example 2: Implementing a chemistry agent following ChemCRAFT directly**

User: "Set up a ChemCRAFT-style agent that optimizes drug-likeness of molecules."

Approach:
1. Install sandbox: `pip install rdkit-pypi pytdc`
2. Define tools: `compute_qed`, `compute_logp`, `compute_sa_score`, `get_scaffold`, `enumerate_modifications`
3. Build trajectory dataset for molecular optimization subtasks
4. Train with SMILES-GRPO using scaffold similarity + QED improvement as reward

Output:
```python
from rdkit import Chem
from rdkit.Chem import QED, Descriptors, AllChem
from tdc import Oracle

# Sandbox tool definitions
def compute_qed(smiles: str) -> float:
    mol = Chem.MolFromSmiles(smiles)
    return QED.qed(mol) if mol else -1.0

def compute_logp(smiles: str) -> float:
    mol = Chem.MolFromSmiles(smiles)
    return Descriptors.MolLogP(mol) if mol else float('inf')

# Dense reward for SMILES-GRPO
def smiles_grpo_reward(pred_smiles, ref_smiles):
    pred_mol = Chem.MolFromSmiles(pred_smiles)
    ref_mol = Chem.MolFromSmiles(ref_smiles)
    if not pred_mol:
        return 0.0  # format penalty: invalid SMILES
    fp_pred = AllChem.GetMorganFingerprintAsBitVect(pred_mol, 2)
    fp_ref = AllChem.GetMorganFingerprintAsBitVect(ref_mol, 2)
    scaffold_sim = DataStructs.TanimotoSimilarity(fp_pred, fp_ref)
    qed_improvement = max(0, QED.qed(pred_mol) - QED.qed(ref_mol))
    return 0.4 * scaffold_sim + 0.4 * qed_improvement + 0.2 * 1.0  # format ok
```

**Example 3: Designing the trajectory refinement pipeline**

User: "How do I build high-quality training data for a tool-calling agent?"

Approach:
1. Generate raw trajectories: teacher model solves tasks with tool calls logged
2. Execute all tool calls against real backends to get verified outputs
3. Replace any hallucinated observations with verified ones
4. Prompt teacher to rewrite: "Given these verified tool outputs, rewrite the reasoning as an expert narrative explaining why each tool was called and how results inform the next step"
5. Quality filter: discard trajectories where final answer is wrong even with correct tool outputs

Output:
```python
def refine_trajectory(raw_traj, sandbox):
    refined_steps = []
    for step in raw_traj.steps:
        if step.type == "tool_call":
            # Execute against real sandbox
            verified_output = sandbox.execute(step.tool_name, step.args)
            step.observation = verified_output  # replace hallucinated output
        refined_steps.append(step)

    # Reflective rewrite
    rewrite_prompt = f"""Rewrite this tool-calling trajectory as an expert-level
    scientific narrative. For each tool call, explain WHY it was chosen and HOW
    the result informs the next reasoning step. Verified observations:
    {format_observations(refined_steps)}"""

    narrative = teacher_model.generate(rewrite_prompt)
    return TrajectoryDatapoint(query=raw_traj.query, reasoning=narrative,
                                answer=raw_traj.answer)
```

## Best Practices

- **Do** keep tool interfaces narrow and deterministic where possible. A tool that returns one float (LogP) is better training signal than one returning a paragraph.
- **Do** use multi-dimensional rewards instead of binary correct/incorrect. Each reward dimension teaches a different aspect of tool-calling competence.
- **Do** apply reflective refinement to trajectories. Raw action logs train mechanical imitation; rewritten narratives train genuine reasoning about tool selection.
- **Do** normalize advantages within output groups (GRPO) rather than training a separate value model. This cuts training infrastructure in half.
- **Avoid** letting the model memorize tool outputs during SFT. If the model can shortcut by recalling a cached answer, it won't learn to call tools at inference time. Use diverse inputs.
- **Avoid** skipping the SFT stage. RL from a random policy is unstable. The SFT cold-start provides a reliable pi_ref that RL refines without collapse.

## Error Handling

- **Invalid tool calls**: If the model emits malformed tool-call syntax, return a structured error observation (`{"error": "InvalidArgs", "detail": "..."}`) and let the model retry. Reward the retry pattern during training.
- **Tool execution failures**: Sandbox tools may timeout or crash. Wrap all calls in try/except, return error observations, and cap retries at 3 per tool per turn.
- **Reward hacking**: If the model learns to call tools unnecessarily to inflate format-compliance reward, add a token-efficiency penalty or tool-call count penalty to the reward function.
- **Policy collapse during GRPO**: If KL divergence spikes, reduce learning rate and increase beta (KL penalty weight). Monitor the entropy of tool-call distributions -- it should stay above a minimum threshold.
- **SMILES validity**: Always validate generated SMILES strings with `Chem.MolFromSmiles()` before computing any downstream reward. Invalid SMILES should receive zero reward, not crash the pipeline.

## Limitations

- **Domain transfer is not automatic.** The sandbox tools, task taxonomy, and reward function must be redesigned for each new domain (chemistry, biology, finance, etc.). The *pattern* transfers; the implementation does not.
- **Trajectory quality bottleneck.** The final agent is bounded by the quality of teacher-generated trajectories. If the teacher model makes systematic errors in tool selection, the student inherits them.
- **Tool latency at inference.** Each tool call adds network/compute latency. For real-time applications, batch tool calls or pre-cache frequent queries.
- **GRPO requires multiple samples per prompt.** Typical K=4-8 outputs per query during training. This multiplies compute cost relative to standard SFT.
- **Not suitable for creative/open-ended tasks.** This pattern excels when there are verifiable, deterministic tool outputs. Tasks requiring subjective judgment or novel synthesis beyond tool capabilities won't benefit.

## Reference

**Paper:** Li et al., "Agentic reinforcement learning empowers next-generation chemical language models for molecular design and synthesis" (arXiv:2601.17687v2, 2026). Look for: Section 3 (sandbox architecture), Section 4 (SMILES-GRPO reward function details), and Figure 1 (two-stage training pipeline diagram). Code: https://github.com/HowardLi1984/ChemCraft