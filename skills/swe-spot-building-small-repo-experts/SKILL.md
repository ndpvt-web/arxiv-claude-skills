---
name: "swe-spot-building-small-repo-experts"
description: >
  Build deep repository expertise for small language models using Repository-Centric Learning (RCL).
  Transforms static codebases into interactive training signals across four dimensions: software design
  comprehension, contextual implementation, evolutionary replay of historical bugs, and semantic-runtime
  alignment via test generation. Use this skill when you hear:
  "train a repo-specific model", "fine-tune on this codebase", "build a repo expert",
  "repository-centric training data", "create training data from PRs",
  "specialize a small model for our repo".
---

# SWE-Spot: Building Small Repo-Experts with Repository-Centric Learning

This skill enables Claude to apply Repository-Centric Learning (RCL) -- the paradigm from SWE-Spot -- to transform any target repository into a structured training dataset that produces highly capable repo-specialized small language models. Rather than training SLMs across thousands of disparate repos (Task-Centric Learning), RCL prioritizes deep vertical mastery of a single codebase, internalizing its architecture, conventions, dependency patterns, and evolutionary history as parametric knowledge. The result: 4B-parameter models that outperform models 8x their size on repo-specific tasks.

## When to Use

- When the user wants to **fine-tune a small model** (1B-8B parameters) to become an expert on a specific codebase
- When building **training data from a repository's pull request history** for supervised fine-tuning
- When the user asks to **generate agentic trajectories** (exploration + reasoning + action traces) from a codebase
- When creating **repo-specialized coding assistants** for privacy-sensitive or resource-constrained environments
- When the user wants to **maximize per-repository performance** instead of broad generalization
- When evaluating whether a codebase has enough signal (PRs, tests, structure) to support RCL training
- When constructing **multi-dimensional training mixes** covering design comprehension, implementation, bug fixing, and test generation

## Key Technique: Repository-Centric Learning (RCL)

**The core insight**: Small language models cannot compensate for missing codebase knowledge through inference-time search the way frontier models can. TCL (training across many repos) teaches general coding patterns but leaves SLMs lost when navigating complex, unfamiliar repositories. RCL flips this: instead of breadth across repositories, invest depth within one. The model learns the "physics" of the target environment -- its module boundaries, naming conventions, test patterns, dependency graph, and evolutionary trajectory -- as embedded parametric knowledge.

**The four-unit Repository-Centric Experience (RCX)** transforms static code into interactive training signals: (1) **Software Design** -- the agent explores the repo and produces structured reports on component functionality, design rationale, and interactions; (2) **Contextual Implementation** -- agentic fill-in-the-middle where the agent discovers cross-file context before writing compliant code; (3) **Evolutionary Replay** -- historical bugs from PR history are reintroduced into the current codebase for the agent to resolve; (4) **Semantic-Runtime Alignment** -- the agent writes reproduction tests that fail on buggy code and pass on the fix, capturing behavioral specifications.

**Why this works**: RCL yields an approximately 18x increase in usable training instances per repository compared to TCL's one-instance-per-PR approach. It achieves 2.5x better sample efficiency, and trained models use 23% fewer inference turns and 11% fewer tokens at test time. Critically, all four RCX units contribute synergistically -- ablations show each dimension adds unique signal, and the combination outperforms any subset.

## Step-by-Step Workflow

### Phase 1: Repository Preparation

1. **Select and snapshot the target repository.** Choose a repository with meaningful PR history (100+ merged PRs with associated tests). Establish a temporal cutoff date -- all training data uses code and PRs before this date; evaluation uses instances after it. Clone the repo and create a frozen training snapshot at the cutoff commit.

2. **Analyze repository structure and identify high-signal components.** Map the module architecture, key abstractions, test infrastructure, and CI configuration. Prioritize components with dense PR history and strong test coverage. Use static analysis to identify dependency clusters -- these become natural training unit boundaries.

3. **Mine and filter pull request history.** Extract PRs merged before the cutoff. For each PR, capture: the diff, issue description (or synthesize one from the diff using a teacher model), affected files, and associated test changes. Apply relaxed filtering -- include PRs even without explicit issue text, using agent-generated descriptions to maximize per-repo diversity. Target approximately 8,000 total training samples across all four RCX units.

### Phase 2: RCX Data Synthesis

4. **Generate Software Design trajectories.** For each major component, use a teacher model (e.g., Gemini-2.5-Pro, Claude) to interactively explore the repository and produce structured architectural reports. The trajectory must include: search actions (file reads, grep queries), reasoning about what was found, and a final structured report covering functionality, design rationale, and inter-component interactions. Generate approximately 2,000 samples.

5. **Generate Contextual Implementation trajectories.** For each PR, extract the changed function/method as a fill-in-the-middle target. The teacher model must first explore the repo to discover relevant context (imports, callers, sibling methods, test expectations) before producing the implementation. The trajectory captures the full exploration-then-implementation sequence. Do NOT provide ground-truth file locations -- the agent must discover them. Generate approximately 2,000 samples.

6. **Generate Evolutionary Replay trajectories.** Reintroduce historical bugs by reverting specific PR diffs into the current codebase state. The teacher model receives the (synthesized or original) issue description and must localize, diagnose, and fix the bug through interactive exploration. Use the original PR diff as the ground-truth fix for validation. Generate approximately 2,000 samples.

7. **Generate Semantic-Runtime Alignment trajectories.** For each bug-fix PR, task the teacher model with writing a reproduction test that: (a) fails when the historical bug is applied, and (b) passes on the fixed codebase. Validate by actually running the tests against both states. Discard samples where the test does not exhibit the correct fail/pass behavior. Generate approximately 2,000 samples.

### Phase 3: Training and Validation

8. **Construct the training mix and fine-tune.** Combine all four RCX units into a balanced training set (roughly equal samples per unit). Fine-tune the base SLM using full-parameter SFT (not LoRA -- the paper shows parameter-efficient methods drop performance from 24% to 17% because deep knowledge acquisition requires high learning capacity). Train for 2 epochs, batch size 16, max sequence length 32,768 tokens, learning rate 1e-5 with cosine decay to 1e-6.

9. **Evaluate on held-out post-cutoff instances.** Test the model on four task types using post-cutoff PRs: issue resolution (Pass@1), test generation (Pass@1), feature implementation (Pass@1), and codebase QA (LLM-as-judge scoring 0-100). Compare against a TCL baseline trained on equivalent total samples spread across many repos.

10. **Deploy as a repo-specific expert.** Package the fine-tuned model for the target environment. Note: do NOT joint-train across multiple repositories -- the paper demonstrates negative transfer where knowledge from distinct repos suppresses each other. Train separate expert models per repository and route queries at inference time.

## Concrete Examples

**Example 1: Building a Django repo expert**

```
User: I want to fine-tune Qwen3-4B to be an expert on the Django repository
for resolving issues. How should I construct the training data?

Approach:
1. Clone Django at a cutoff commit (e.g., Dec 31 2023). Identify 500+
   merged PRs with test changes before this date.
2. Map Django's module structure: django/db/, django/http/, django/forms/,
   django/views/, etc. Identify high-churn components from PR density.
3. For Software Design: have the teacher model explore django/db/models/
   and produce reports like "QuerySet implements lazy evaluation via
   _iterable_class, chaining filter() calls append to self.query.where..."
4. For Contextual Implementation: extract a method like
   QuerySet.select_related() as a fill-in target. The teacher must first
   find related managers, SQL compiler references, and test expectations.
5. For Evolutionary Replay: take a real bug-fix PR (e.g., fixing
   select_related on proxy models), revert the fix, give the agent the
   issue text, and record the resolution trajectory.
6. For Semantic-Runtime Alignment: for that same bug, generate a test:
   ```python
   def test_select_related_proxy_model(self):
       # Must fail on buggy code, pass on fix
       result = ProxyModel.objects.select_related('fk_field').first()
       self.assertEqual(result.fk_field.name, "expected")
   ```
   Validate by running against both states.
7. Mix ~2k samples per unit, fine-tune with full SFT for 2 epochs.

Output:
- A Qwen3-4B model that resolves Django issues at rates competitive with
  GPT-4.1-mini, using 23% fewer exploration steps because it has
  internalized Django's architecture.
```

**Example 2: Assessing RCL feasibility for a private codebase**

```
User: Can I use RCL to train a model on our internal monorepo? We have
about 2 years of PR history but limited test coverage.

Approach:
1. Audit PR history: count merged PRs with meaningful diffs (not just
   config changes). Need 200+ substantive PRs minimum for viable RCL.
2. Assess test coverage: Semantic-Runtime Alignment requires runnable
   tests. If coverage is <30%, you can still use 3 of 4 RCX units but
   lose the test-generation dimension.
3. Check for Evolutionary Replay viability: PRs need to be revertible
   against current code. If the repo has heavy rebasing or squash-only
   merges, replay data will be harder to construct.
4. Compensate for gaps: if PR history is thin, augment with synthetic
   tasks -- generate implementation targets by masking functions and
   having the teacher model write contextual-implementation trajectories.

Output:
- Feasibility report:
  - PR count: 340 (sufficient)
  - Test coverage: 22% (Semantic-Runtime Alignment limited to
    well-tested modules; supplement with synthetic tasks)
  - Recommendation: proceed with RCL on the 5 highest-coverage modules
    first, then expand. Expected ~5,000 training samples.
```

**Example 3: Comparing RCL vs TCL training strategies**

```
User: We have a budget for 8,000 training samples. Should we use RCL on
one repo or TCL across our 12 microservices?

Approach:
1. RCL option: 8,000 samples deep on one critical repo. The model
   becomes a specialist that internalizes architecture, reducing
   inference costs by ~23% and improving resolve rate significantly.
2. TCL option: ~670 samples per repo across 12 services. The model
   gets shallow exposure to each, relying on inference-time search
   to compensate -- but SLMs are weak at this.
3. The paper shows RCL hits peak TCL performance at 2.5x fewer samples.
   With 8k samples, RCL on one repo will substantially outperform
   TCL spread across 12.
4. Recommended hybrid: pick the 2-3 highest-priority repos, train
   separate RCL experts for each (~2,700 samples each), and route
   queries at inference time. Do NOT joint-train -- negative transfer
   between repos degrades all of them.

Output:
- Strategy: 3 separate RCL experts (2,700 samples each) for your
  3 most critical services, with a routing layer that dispatches
  to the appropriate expert based on the target file path.
```

## Best Practices

- **Do:** Use full-parameter fine-tuning, not LoRA or QLoRA. The paper demonstrates a 7-point performance drop with parameter-efficient methods because deep repository knowledge requires modifying the model's full representational capacity.
- **Do:** Balance all four RCX units roughly equally in the training mix. Ablations show each contributes unique signal -- Software Design teaches architecture, Contextual Implementation teaches conventions, Evolutionary Replay teaches debugging, and Semantic-Runtime Alignment teaches behavioral specifications.
- **Do:** Enforce a strict temporal cutoff between training data and evaluation data. All training trajectories use pre-cutoff code; all evaluation uses post-cutoff instances. This prevents data leakage and measures true generalization within the repo.
- **Do:** Validate Semantic-Runtime Alignment samples by actually executing the generated tests against both buggy and fixed code states. Discard any sample where the test does not exhibit correct fail-then-pass behavior.
- **Avoid:** Joint-training a single model across multiple repositories. The paper shows negative transfer -- distinct repos' knowledge suppresses each other. Train separate experts and use a routing layer instead.
- **Avoid:** Providing ground-truth file locations during training. The agent must learn to discover relevant files through exploration. Oracle localization experiments showed it helps TCL models but provides zero additional benefit to RCL models, confirming RCL already internalizes this knowledge.

## Error Handling

- **Insufficient PR history**: If the target repo has fewer than 100 substantive PRs, supplement with synthetic tasks. Mask functions, generate implementation trajectories, and create synthetic "issues" from code comments and TODOs. Prioritize Contextual Implementation and Software Design units which don't require PR diffs.
- **Test infrastructure failures**: Evolutionary Replay and Semantic-Runtime Alignment require a working test harness. If the repo's test suite is flaky or broken, containerize the test environment with pinned dependencies at the cutoff commit. Skip repos where <10% of tests pass reliably.
- **Teacher model hallucination**: When the teacher model generates exploration trajectories, validate that all file paths and code references actually exist in the repo snapshot. Discard trajectories containing hallucinated files or functions.
- **Negative transfer during multi-repo training**: If you accidentally observe performance regression after adding a second repo's data, immediately separate into per-repo experts. This is expected behavior, not a bug.
- **Sequence length overflow**: RCX trajectories (especially Evolutionary Replay) can be long. If trajectories exceed 32,768 tokens, truncate the exploration phase rather than the resolution phase -- preserving the final fix is more important than every search step.

## Limitations

- **Single-repo specialization**: Each trained model is an expert on exactly one repository. You need N models for N repos, plus a routing mechanism. This is by design -- the paper shows joint training hurts -- but it adds deployment complexity.
- **Requires substantial PR history**: Repos with minimal commit history or no test infrastructure cannot fully leverage all four RCX units. You can still use Software Design and Contextual Implementation, but you lose the most distinctive RCL signals.
- **Teacher model dependency**: Data synthesis requires a capable teacher model to generate high-quality trajectories. The quality ceiling of the student is bounded by the teacher's ability to explore and reason about the target repo.
- **Not a replacement for general coding ability**: RCL complements general coding skill -- it doesn't replace it. The base model must already have competent code understanding. RCL adds repo-specific mastery on top.
- **Temporal drift**: As the repository evolves post-training, the model's internalized knowledge becomes stale. Plan for periodic retraining with fresh RCX data from recent PRs to maintain accuracy.

## Reference

[SWE-Spot: Building Small Repo-Experts with Repository-Centric Learning](https://arxiv.org/abs/2601.21649v1) -- Peng et al. (2026). Focus on Section 3 (RCX framework design), Section 4 (training methodology), and Table 2 (ablation results showing each RCX unit's contribution). The key takeaway: repository mastery is a distinct dimension of coding intelligence that cannot be substituted by scaling general task diversity.