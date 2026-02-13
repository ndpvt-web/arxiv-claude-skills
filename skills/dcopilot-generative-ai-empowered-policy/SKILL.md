---
name: "dcopilot-generative-ai-empowered-policy"
description: "Build hybrid LLM + hypernetwork systems that generate control policies for dynamic environments. Uses LLM-based reward synthesis, simulation stress-testing, and meta policy distillation for zero-shot adaptation. Triggers: 'generate reward function from spec', 'build adaptive controller', 'hypernetwork for policy generation', 'LLM reward shaping', 'zero-shot policy adaptation', 'data center control optimization'"
---

# DCoPilot: Generative AI-Empowered Policy Adaptation

This skill enables Claude to implement the DCoPilot framework -- a hybrid architecture that combines LLM-based symbolic reward generation with hypernetwork-based parametric policy generation. The core idea: instead of retraining a reinforcement learning agent every time operating conditions change, use an LLM to generate a unified reward function family, train a pool of expert policies across diverse conditions, then distill them into a hypernetwork that produces new policy weights on demand from specification embeddings. This eliminates the specification-to-policy lag in dynamic control systems.

## When to Use

- When the user needs to build an adaptive controller that handles changing operating specifications (SLAs, setpoints, constraints) without retraining
- When designing an LLM-based reward function generator that produces structured, parameterized reward forms from natural language specifications
- When implementing a hypernetwork that outputs neural network weights conditioned on task embeddings
- When building a simulation-based pipeline to stress-test reward candidates across boundary conditions
- When the user asks to create a meta-learning system for continuous control with zero-shot generalization
- When implementing imitation learning from a curated pool of expert policies across varied conditions
- When building digital twin / SimReady pipelines for data center or industrial control optimization

## Key Technique

**The Specification-to-Policy Gap.** Traditional deep reinforcement learning requires retraining whenever operating conditions shift -- new temperature setpoints, humidity constraints, or workload profiles. In dynamic environments like data centers, these changes happen faster than agents can be retrained, causing control gaps. DCoPilot solves this with a three-phase generative pipeline.

**Phase 1 -- LLM Reward Synthesis + Simulation Scale-Up.** An LLM receives a contextualized prompt containing operator objectives (natural language), DRL training configuration, and scene descriptions. It generates N candidate reward functions via stochastic sampling (temperature=0.7, N=5). Each candidate is stress-tested on boundary conditions combining min/max environment parameters and SLA parameters. An evolutionary loop selects top-k candidates, appends performance summaries to the prompt, and iterates until convergence on a unified reward form `R*(s|psi)` -- one functional structure that works across all specification variants by parameterizing SLA setpoints.

**Phase 2 -- Hypernetwork Meta-Distillation.** Using the unified reward, expert policies are trained for each (scene, SLA) pair in the Cartesian product of conditions. Their trajectories form a demonstration pool. A hypernetwork `H_Theta` -- an MLP that takes a joint embedding of scene features `mu` and SLA parameters `psi` and outputs complete policy weights `theta` -- is trained via supervised imitation: `L(Theta) = -sum log pi_{H(e_i)}(a_{i,t} | s_{i,t})`. At deployment, new conditions produce new policy weights instantly via forward pass -- no gradient updates, no environment interaction.

## Step-by-Step Workflow

### 1. Define the Control Task Specification Schema

Create a structured schema for your environment with three components:

```python
@dataclass
class TaskSpec:
    scene_params: dict    # mu: physical configuration (e.g., server density, rack layout)
    sla_params: dict      # psi: operational constraints (e.g., T_max=25C, humidity<60%)
    objectives: list      # what to minimize (e.g., HVAC power, water usage)
    observation_space: list  # state variables the agent observes
    action_space: list       # actuator controls (e.g., CRAC setpoint, valve position)
```

### 2. Build the LLM Reward Generation Prompt

Construct a contextualized prompt template `C_reward(I_ops, I_cfg, G)` with three sections:

```python
REWARD_PROMPT_TEMPLATE = """
## Operator Objectives (I_ops)
{objectives_nl}  # Natural language: "Minimize HVAC power while keeping zone temp below T_max"

## DRL Training Configuration (I_cfg)
- Observation space: {obs_vars}
- Action space: {act_vars}
- Episode length: {horizon} steps at {interval}-minute intervals
- Constraint format: violation = max(0, measured - upper_bound, lower_bound - measured)

## Scene Description (G)
{scene_description}  # "Single-zone with 120kW server density, one CRAC unit"

## Task
Generate a reward function R(s, a, psi) where psi contains SLA setpoints.
The function must:
1. Penalize constraint violations proportionally
2. Reward objective improvement
3. Accept psi as a parameter so the same form works across SLA variations
Return Python code for the reward function.
"""
```

### 3. Run Evolutionary Reward Search

Generate N=5 candidates per iteration, evaluate each on boundary conditions:

```python
def evolve_reward(llm, prompt_template, env_factory, n_candidates=5, n_iters=4, top_k=2):
    history = []
    for iteration in range(n_iters):
        # Generate candidates via stochastic sampling
        candidates = [llm.generate(prompt_template, temperature=0.7) for _ in range(n_candidates)]

        # Stress-test on boundary conditions: cartesian product of extremes
        boundary_conditions = [
            (mu_min, psi_min), (mu_min, psi_max),
            (mu_max, psi_min), (mu_max, psi_max)
        ]
        scores = []
        for reward_fn in candidates:
            violation_cost, objective_score = 0, 0
            for mu, psi in boundary_conditions:
                env = env_factory(mu)
                policy = train_policy(env, reward_fn, psi, steps=50000)
                traj = rollout(env, policy, psi)
                violation_cost += compute_violations(traj, psi)
                objective_score += compute_objective(traj)
            scores.append((violation_cost, objective_score, reward_fn))

        # Select top-k, build refinement prompt
        top = sorted(scores, key=lambda x: (x[0], -x[1]))[:top_k]
        history.append(format_performance_summary(top))
        prompt_template = append_history(prompt_template, history)

    return top[0][2]  # Return best reward function
```

### 4. Generate the SimReady Scene Variants

Build a hierarchical scene model and enumerate configurations:

```python
def generate_scene_variants(base_scene, param_ranges):
    """
    base_scene: calibrated from real operational data
    param_ranges: {"server_density_kw": [80, 100, 120], "rack_count": [10, 20, 30]}
    """
    scenes = []
    for combo in itertools.product(*param_ranges.values()):
        mu = dict(zip(param_ranges.keys(), combo))
        scene = base_scene.configure(mu)
        scenes.append((mu, scene))
    return scenes
```

### 5. Curate the Expert Policy Pool

For each (mu, psi) pair in the Cartesian product, train a specialist policy and collect demonstrations:

```python
def curate_policy_pool(scenes, sla_variants, reward_fn, algo="SAC"):
    trajectory_pool = []
    for mu, scene in scenes:
        for psi in sla_variants:
            env = scene.to_env()
            policy = train_rl(env, reward_fn, psi, algorithm=algo)
            trajectories = collect_trajectories(env, policy, n_episodes=50)
            embedding = encode_task(mu, psi)  # Concatenated embedding
            trajectory_pool.append((embedding, trajectories))
    return trajectory_pool
```

### 6. Build and Train the Hypernetwork

```python
class HyperNetwork(nn.Module):
    def __init__(self, embed_dim, policy_param_count, hidden=256):
        super().__init__()
        self.mu_encoder = nn.Sequential(nn.Linear(mu_dim, hidden), nn.ReLU())
        self.psi_encoder = nn.Sequential(nn.Linear(psi_dim, hidden), nn.ReLU())
        self.hypernet = nn.Sequential(
            nn.Linear(hidden * 2, hidden * 4), nn.ReLU(),
            nn.Linear(hidden * 4, hidden * 4), nn.ReLU(),
            nn.Linear(hidden * 4, policy_param_count)
        )
        self.policy_template = PolicyNetwork()  # Defines architecture

    def forward(self, mu_embed, psi_embed):
        e = torch.cat([self.mu_encoder(mu_embed), self.psi_encoder(psi_embed)], dim=-1)
        weights = self.hypernet(e)
        return weights  # Load into policy_template for inference

def train_hypernetwork(hypernet, trajectory_pool, epochs=200, lr=1e-3):
    optimizer = Adam(hypernet.parameters(), lr=lr)
    for epoch in range(epochs):
        for embedding, trajectories in trajectory_pool:
            mu_e, psi_e = split_embedding(embedding)
            theta = hypernet(mu_e, psi_e)
            policy = load_weights(hypernet.policy_template, theta)

            # Imitation learning loss: negative log-likelihood
            loss = 0
            for s, a in trajectories:
                log_prob = policy.log_prob(s, a)
                loss -= log_prob.mean()

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

### 7. Deploy Zero-Shot Online Adaptation

```python
def adapt_online(hypernet, new_mu, new_psi):
    """Zero-shot: no retraining, no environment interaction."""
    mu_embed = encode_scene(new_mu)   # From IoT monitoring / digital twin
    psi_embed = encode_sla(new_psi)   # From updated SLA contract
    theta = hypernet(mu_embed, psi_embed)
    policy = load_weights(policy_template, theta)
    return policy  # Deploy immediately
```

### 8. Monitor and Compute Violation Metrics

```python
def compute_violation_cost(trajectory, sla):
    """Time-averaged constraint violation magnitude."""
    violations = []
    for t, state in enumerate(trajectory):
        for var, (lo, hi) in sla.bounds.items():
            v = max(0, state[var] - hi, lo - state[var])
            violations.append(v)
    return np.mean(violations)  # Target: < 0.2 degrees
```

## Concrete Examples

**Example 1: Data Center Thermal Control**

User: "Build a system that automatically generates HVAC control policies when our data center SLA changes from 25C max to 22C max."

Approach:
1. Define scene params (server density, rack layout) and SLA params (T_max, humidity bounds)
2. Use LLM to generate a parameterized reward: `R(s,a|psi) = -alpha * power - beta * max(0, T_zone - psi.T_max)`
3. Stress-test reward across boundary conditions (80kW-120kW density x 22C-27C setpoint)
4. Train expert SAC policies for each (density, setpoint) combination
5. Train hypernetwork on expert trajectory pool
6. At deployment: `policy = hypernet(current_density_embed, new_sla_embed)` -- instant adaptation

Output:
```
New SLA received: T_max = 22C (was 25C)
Generating policy weights... done (12ms forward pass)
Deploying policy to CRAC controller
Violation cost after 24h: 0.08C (threshold: 0.2C)
PUE improvement: 1.31 -> 1.28
```

**Example 2: Multi-Objective Zone Control with Humidity**

User: "I need a controller that handles both temperature and humidity constraints, and the setpoints change seasonally."

Approach:
1. Extend task spec: `psi = {T_max, T_min, RH_max, epsilon_RH}`
2. LLM generates multi-objective reward with separate violation terms per constraint
3. Scene variants span seasonal outdoor conditions (hot-humid summer, cold-dry winter)
4. Hypernetwork embedding encodes both indoor SLA and outdoor weather profile
5. Seasonal SLA changes trigger zero-shot policy swap

Output:
```python
seasonal_sla = {"summer": {"T_max": 27, "RH_max": 60}, "winter": {"T_max": 24, "RH_max": 45}}
for season, psi in seasonal_sla.items():
    policy = hypernet(scene_embed, encode_sla(psi))
    deploy(policy, season_start_date[season])
# Result: <0.15C violation, <2% RH violation across both seasons
```

**Example 3: LLM Reward Function Evolution**

User: "My hand-designed reward function causes training instability. Can you use an LLM to generate a better one?"

Approach:
1. Encode current reward's failure modes into the prompt: "Policy oscillates between extremes, violation cost = 4.2C"
2. LLM generates 5 candidates with different penalty structures (quadratic vs linear, additive vs multiplicative)
3. Each candidate is trained on 4 boundary conditions for 50k steps
4. Top-2 candidates feed back into the prompt with performance summaries
5. After 3-4 iterations, converge on stable reward form

Output:
```
Iteration 1: Best violation=1.8C, Worst=6.1C
Iteration 2: Best violation=0.5C (quadratic penalty form selected)
Iteration 3: Best violation=0.18C (converged)
Final reward: R = -0.1*P_hvac - 5.0*max(0, T-T_max)^2 - 2.0*max(0, RH-RH_max)^2
```

## Best Practices

- **Do:** Parameterize reward functions with SLA variables (`psi`) so one functional form covers all specification variants -- this is the key to stable hypernetwork training
- **Do:** Stress-test reward candidates on boundary conditions (extreme combinations of scene and SLA parameters) before accepting them
- **Do:** Use separate embedding encoders for scene parameters (`mu`) and SLA parameters (`psi`) before concatenation -- this preserves distinct information channels
- **Do:** Collect 50+ demonstration trajectories per (scene, SLA) pair to give the hypernetwork sufficient supervision signal
- **Avoid:** Training the hypernetwork on reward values directly -- use imitation learning on expert trajectories (behavioral cloning via log-likelihood), which produces more stable convergence
- **Avoid:** Fine-tuning hypernetwork-generated weights online -- the zero-shot weights should be deployed as-is; online updates destabilize the meta-learned representations

## Error Handling

**Reward candidates all produce high violations:** Increase the number of evolutionary iterations, lower LLM temperature to 0.5 for more conservative candidates, and add explicit constraint examples to the prompt showing what violation=0 looks like.

**Hypernetwork outputs degenerate policies for unseen embeddings:** The task embedding is outside the training distribution. Expand the Cartesian product of scene/SLA variants used during distillation, or add interpolation points between existing training conditions.

**Imitation learning loss plateaus but policy underperforms:** The expert policy pool may contain suboptimal demonstrations. Re-filter trajectories by violation cost (keep only those with violation < threshold) before distillation.

**LLM generates syntactically invalid reward functions:** Wrap generation in a parse-validate-retry loop. Provide the LLM with a concrete function signature and type annotations in the prompt. Limit to 3 retries before falling back to the previous best candidate.

## Limitations

- **Interpolation, not extrapolation:** The hypernetwork generalizes within the convex hull of training conditions. Conditions far outside the training distribution (e.g., a 200kW density when trained on 80-120kW) will produce unreliable policies.
- **Expert quality ceiling:** The hypernetwork cannot exceed the quality of its expert policy pool. If the underlying RL algorithm (SAC, PPO) fails to solve a condition, the hypernetwork inherits that failure.
- **Simulation fidelity dependency:** The entire pipeline relies on SimReady digital twins that accurately reflect real dynamics. Sim-to-real gaps propagate through all three phases.
- **Compute cost at training time:** Generating the expert policy pool requires training O(|mu| x |psi|) separate RL agents. This is a one-time cost, but it scales quadratically with the number of condition variants.
- **LLM reward generation is non-deterministic:** Different runs may converge to different reward forms. Always validate the final reward against held-out conditions before committing to distillation.

## Reference

[DCoPilot: Generative AI-Empowered Policy Adaptation for Dynamic Data Center Operations](https://arxiv.org/abs/2602.02137v2) -- Li et al., 2026. Focus on Section 3 (three-phase framework), Section 4 (reward evolution algorithm), and the ablation in Section 6 showing that LLM-generated unified rewards are critical for hypernetwork convergence stability.