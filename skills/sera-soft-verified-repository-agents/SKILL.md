---
name: sera-soft-verified-repository-agents
description: >
  Train and deploy repository-specialized coding agents using SERA's Soft Verified Generation (SVG)
  pipeline. Generates synthetic training trajectories from any codebase without test infrastructure,
  enabling cheap SFT-based agent specialization. Use this skill when:
  "specialize a coding agent to my repo", "generate training data from this codebase",
  "set up SERA for my project", "create synthetic coding trajectories",
  "train a repo-specific coding agent", "deploy SERA with Claude Code"
---

# SERA: Soft-Verified Efficient Repository Agents

This skill enables Claude to help users generate synthetic coding agent trajectories from any code repository using SERA's Soft Verified Generation (SVG) pipeline, train repository-specialized coding agents via supervised finetuning, and deploy those agents through Claude Code integration. SERA eliminates the need for test infrastructure or reinforcement learning, making agent specialization practical at $1,300 per repository instead of tens of thousands of dollars.

## When to Use

- When a user wants to create a coding agent specialized to their private codebase
- When generating synthetic training data (trajectories) from a code repository without existing tests
- When setting up the SERA pipeline to produce SFT training data for open-weight models (Qwen 3-32B, etc.)
- When deploying a SERA-trained model as a Claude Code backend via the sera-cli proxy
- When the user asks about cost-efficient alternatives to RL-based agent training
- When filtering, truncating, or curating coding agent trajectories for SFT quality
- When specializing an existing SERA model to a specific repository (Django, Sympy, or custom)

## Key Technique

**Soft Verified Generation (SVG)** produces training trajectories through a two-rollout process that requires no test suite. In the first rollout, a teacher model (e.g., GLM-4.5-Air) receives a randomly selected function from the target repository plus a vague bug description sampled from a library of 51 bug types. The teacher generates a trajectory T1 and patch P1. This trajectory is then converted into a synthetic pull request description using real PR examples as formatting templates.

In the second rollout, a fresh teacher instance receives only the synthetic PR description and must independently reproduce the patch, generating T2 and P2. **Soft verification** compares the patches line-by-line using recall: `r = |P2 ∩ P1| / |P1|`. When r=1.0, the trajectory is "hard-verified" (both rollouts agree exactly). When 0 < r < 1.0, it is "soft-verified" (partial agreement). The key finding is that all verification thresholds perform similarly at tested scales, meaning strict test-based verification is unnecessary -- line-level patch agreement suffices.

This approach is 26x cheaper than reinforcement learning and 57x cheaper than prior synthetic data methods (SWE-smith). A single repository yields thousands of trajectories. Training uses standard SFT on Qwen 3-32B with a 32K context window, learning rate 1e-5, 3 epochs, and tools like axolotl for training and vLLM for inference hosting.

## Step-by-Step Workflow

### Phase 1: Environment and Infrastructure Setup

1. **Create the conda environment and install SERA.**
   ```bash
   conda create -n sera python=3.12
   git clone https://github.com/allenai/sera.git && cd sera
   pip install -e . -e modules/code2flow -e modules/SWE-agent
   ```

2. **Launch the teacher model inference server.** Start a vLLM server hosting the teacher model (GLM-4.5-Air or GLM-4.6) with the provided launch scripts. You need GPUs with at least 80GB VRAM (A100, H100).
   ```bash
   bash launch_glm45.sh 1 8000 42
   # Verify: curl http://localhost:8000/v1/models
   ```

3. **Prepare the target repository.** For SWE-bench specialization, use built-in configs. For personal repositories, edit `specialization_personal.yaml` to specify `install_cmds`, `top_level_folder`, `python_version`, and set up a GitHub mirror organization with Docker registry access. Scrape existing issues if available:
   ```bash
   python scrape_github.py -o YOUR_ORG -n YOUR_REPO -c 50
   ```

### Phase 2: Trajectory Generation (SVG Pipeline)

4. **Run the SVG generation pipeline.** This executes the two-rollout process -- generating trajectories, converting to synthetic PRs, reproducing patches, and computing soft verification scores.
   ```bash
   # For SWE-bench specialization (e.g., Django):
   python sera/main.py --config-name=specialization_django \
     distill.model.name=openai/GLM-4.5-Air \
     distill.model.url=http://localhost:8000/v1

   # For personal repositories:
   python sera/main.py --config-name=specialization_personal \
     distill.model.name=openai/GLM-4.5-Air \
     distill.model.url=http://localhost:8000/v1
   ```
   The pipeline runs through stages: Generate -> Distill Stage One -> Distill Stage Two -> Eval -> Postprocess. Resume interrupted runs with `stage=STAGE_NAME`.

5. **Scale with distributed sharding.** For large-scale generation (200K+ trajectories), shard across multiple GPU servers:
   ```bash
   python sera/main.py --config-name=swesmith_scaling \
     distill.shard=0 distill.total_shards=4 \
     distill.model.url=http://server1:8000/v1
   # Run shards 1-3 on other servers simultaneously
   ```

### Phase 3: Data Curation and Filtering

6. **Filter trajectories by truncation ratio.** Prioritize trajectories that fit within 32K tokens. Select trajectories with truncation ratio >= 0.88 (proportion of steps fitting in context). Trajectories with ratio ~0.95 perform best -- partial truncation outperforms both full inclusion and random slicing.

7. **Apply repository-specific filtering.** For specialization, filter aggressively:
   - Django/Sympy: exclude patches exceeding 40 edited lines
   - Sphinx: exclude trajectories with average tool output exceeding 600 tokens
   - Custom repos: profile your generated data and filter outliers in edit length and observation verbosity

### Phase 4: Training

8. **Run supervised finetuning.** Train using axolotl on the curated trajectories. Key hyperparameters: base model Qwen 3-32B, learning rate 1e-5, weight decay 0.01, 3 epochs, 32K context length. For repository specialization, use mixing ratio alpha=1.0 (pure repo-specific data) to match teacher performance at ~8K samples. See `sera/datagen/train/README.md` for training scripts.

### Phase 5: Deployment via Claude Code

9. **Deploy with sera-cli.** Install the CLI and launch the proxy that bridges Claude Code to your SERA model:
   ```bash
   uv tool install ai2-sera-cli

   # Quick start with Modal (auto-provisions GPU):
   sera --modal --model allenai/SERA-32B

   # Or connect to your own vLLM endpoint:
   sera --endpoint http://your-server:8000/v1

   # Persistent team deployment:
   deploy-sera --model allenai/SERA-32B
   ```
   The proxy server (port 8080) translates between Claude Code's tool format (Read, Edit, Write, Bash) and SWE-agent format (str_replace_editor, bash), handling path normalization automatically.

10. **Validate tool format compatibility.** After deployment, test with a simple coding task. Watch for "unproductive loops" where the model repeatedly verifies already-completed edits -- this signals a tool format mismatch between the proxy and the model's training format. The proxy must exactly match SWE-agent tool calling conventions.

## Concrete Examples

**Example 1: Specialize SERA to a Django codebase**

User: "I want to train a coding agent that's specialized to our Django monorepo so it understands our custom ORM patterns."

Approach:
1. Set up SERA environment and launch GLM-4.5-Air as teacher on an A100
2. Configure `specialization_django.yaml` pointing to the target Django repo commits
3. Run SVG pipeline: generates ~50K trajectories across 5 equally-spaced commits
4. Filter: remove patches >40 lines, select truncation ratio >= 0.88
5. SFT on Qwen 3-32B with alpha=1.0 (pure Django data), 8K-16K trajectories
6. Deploy via `sera --endpoint http://gpu-server:8000/v1`

Output:
```
Generated: 52,340 trajectories (23% hard-verified, 41% soft-verified)
After filtering: 14,200 trajectories (<=32K tokens, <=40 line patches)
Training: 3 epochs on Qwen 3-32B, ~$1,300 compute
Result: Agent resolves 51.2% of repo-specific tasks (matches teacher GLM-4.5-Air)
```

**Example 2: Generate training data from a private Python repo without tests**

User: "Our internal ML platform has no test suite but I want to generate agent training data from it."

Approach:
1. Mirror the repo to a GitHub org accessible by Docker builds
2. Edit `specialization_personal.yaml`:
   ```yaml
   repos:
     - name: ml-platform
       install_cmds: "pip install -e ."
       top_level_folder: "ml_platform"
       python_version: "3.11"
   ```
3. Scrape any existing GitHub issues: `python scrape_github.py -o myorg -n ml-platform -c 30`
4. Run SVG -- the pipeline samples functions, generates vague bug prompts from the 51-type library, and produces trajectories without needing any tests
5. Soft verification filters based on patch recall between rollouts

Output:
```
Functions sampled: 1,847
First rollouts completed: 1,712 (98% self-evaluation pass rate)
Second rollouts completed: 1,580
Soft-verified (r > 0): 1,203 trajectories
Hard-verified (r = 1): 412 trajectories
Ready for SFT after truncation filtering: ~1,000 trajectories
```

**Example 3: Deploy pre-trained SERA-32B with Claude Code for immediate use**

User: "I just want to try SERA as my coding agent in Claude Code without training anything."

Approach:
1. Install sera-cli: `uv tool install ai2-sera-cli`
2. Launch with Modal (auto-provisions GPU): `sera --modal`
3. Claude Code opens automatically, connected through the local proxy on port 8080
4. Code as normal -- all tool calls route through the proxy to SERA-32B on Modal

Output:
```
$ sera --modal
Starting SERA proxy on port 8080...
Provisioning Modal GPU (A100 80GB)...
Model loaded: allenai/SERA-32B
Claude Code launched. All requests routing through SERA.
```

## Best Practices

**Do:**
- Use truncation ratio 0.95 as the sweet spot -- trajectories slightly exceeding context that get minimally truncated outperform both short-only and heavily-truncated data
- Set alpha=1.0 (100% repo-specific data) for maximum specialization; general data dilutes repo-specific knowledge
- Generate from multiple commits of the same repo (5 equally-spaced) to capture codebase evolution and avoid overfitting to a single snapshot
- Run at least 3 seeds per experiment for reliable comparisons; use SNR >= 2 as confidence threshold (requires ~4 seeds for 2% effect sizes)

**Avoid:**
- Do not skip soft verification entirely -- while strict thresholds don't help, completely unfiltered (r=0) data includes noise from failed reproductions
- Do not use random slicing for long trajectories; ordered truncation from the end preserves the reasoning chain's early context which is more important
- Do not mix SWE-agent and Claude Code tool formats in training data; format mismatches cause unproductive verification loops at inference time
- Do not assume more data always helps -- scaling laws show diminishing returns; profile your performance curve and stop when gains plateau

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Model enters unproductive edit-verify loops | Tool format mismatch between training and inference | Ensure the sera-cli proxy exactly matches SWE-agent tool format; check `str_replace_editor` vs `Edit` translation |
| Low soft verification rate (<20%) | Teacher model too weak or vague prompts too ambiguous | Use a stronger teacher (GLM-4.6 over 4.5-Air) or reduce prompt vagueness |
| Trajectories consistently exceed 32K tokens | Complex repository with deep call chains | Increase truncation ratio threshold to 0.95; filter functions by complexity before sampling |
| Pipeline fails at Distill Stage Two | Docker build issues for target repo | Verify `install_cmds` in config; ensure GitHub mirror org and Docker registry are accessible |
| SFT performance plateaus below teacher | Insufficient or low-diversity trajectories | Generate from more commits; increase bug type diversity; check for duplicate patch filtering |
| sera-cli proxy connection refused | Port conflict or vLLM server not ready | Change port with `--port`; verify vLLM endpoint responds to `/v1/models` before connecting |

## Limitations

- **GPU requirements are non-trivial.** SERA-32B requires minimum 80GB VRAM (A100/H100). The 8B variant is lighter but less capable. Quantization (AWQ, GPTQ) helps but may degrade quality.
- **Python-centric.** The current pipeline, bug type library, and evaluation are heavily oriented toward Python codebases. Applying to other languages requires adapting the function sampling and bug prompt system.
- **Teacher model ceiling.** Specialized models can match but not exceed teacher performance. If GLM-4.5-Air resolves 51% of tasks, your specialized model caps near that level.
- **Verification is approximate.** Soft verification via line-level patch recall is a heuristic. It cannot detect semantically equivalent but syntactically different patches, potentially discarding valid trajectories.
- **Context window constraint.** At 32K tokens, complex multi-file changes requiring extensive exploration get truncated. This biases the agent toward shorter, more localized fixes.
- **No interactive debugging.** Trajectories are single-pass -- the agent cannot ask clarifying questions or iterate with a human during training data generation.

## Reference

**Paper:** [SERA: Soft-Verified Efficient Repository Agents](https://arxiv.org/abs/2601.20789v2) (Shen et al., 2026)
**Code:** [github.com/allenai/sera](https://github.com/allenai/sera) | **CLI:** [github.com/allenai/sera-cli](https://github.com/allenai/sera-cli)

Key sections to read: Section 3 (SVG pipeline details), Section 4.2 (repository specialization), Section 5 (scaling laws and ablations), Appendix A (Claude Code integration and tool format translation).