---
name: "found-rl-foundation-model-enhanced-reinforcement"
description: "Architect asynchronous VLM-enhanced RL training pipelines that decouple heavy foundation model inference from simulation loops. Implements Value-Margin Regularization (VMR), Advantage-Weighted Action Guidance (AWAG), and CLIP-based reward shaping with Conditional Contrastive Action Alignment. Use when: 'build an async VLM+RL pipeline', 'integrate foundation models with reinforcement learning', 'add CLIP reward shaping to RL training', 'decouple VLM inference from simulation', 'distill VLM knowledge into lightweight RL policy', 'speed up foundation model RL training with async inference'."
---

# Found-RL: Foundation Model-Enhanced Reinforcement Learning

This skill enables Claude to design and implement training pipelines where heavyweight foundation models (Vision-Language Models, CLIP) supervise lightweight reinforcement learning agents without becoming a latency bottleneck. The core architectural insight from Found-RL is that VLM inference and simulation can run on completely separate timelines via asynchronous batch queuing, with three complementary supervision signals -- value-margin regularization, advantage-weighted action guidance, and contrastive reward shaping -- bridging the gap between slow VLM wisdom and fast RL execution.

## When to Use

- When the user wants to integrate a large VLM (e.g., LLaVA, Qwen-VL, GPT-4V) into an RL training loop without crippling throughput
- When building an autonomous driving, robotics, or game AI pipeline that needs semantic understanding from foundation models
- When the user asks to distill knowledge from a billion-parameter model into a small real-time policy network
- When designing CLIP-based reward functions for RL and encountering issues with static scene bias (CLIP rewarding parked cars over moving ones)
- When the user needs to architect a producer-consumer system where simulation workers produce observations and a VLM server produces guidance asynchronously
- When combining multiple supervision signals (value correction, action guidance, dense reward shaping) in a single RL training run

## Key Technique

**Asynchronous Batch Inference Framework.** The central bottleneck in VLM-enhanced RL is latency: a VLM takes 200-2000ms per inference call, while an RL simulation step runs in 2-5ms. Found-RL resolves this by running VLM inference on a separate process (or GPU) that consumes from a shared request queue and writes results to a stale-tolerant guidance buffer. Rollout workers continue collecting experience at full speed; when VLM results arrive (possibly 50-100 steps later), they are associated with the nearest matching state and applied retroactively or prospectively. This decoupling means VLM throughput only needs to match the *batch* rate, not the *step* rate.

**Three Supervision Channels.** Found-RL distills VLM knowledge through three complementary mechanisms: (1) **Value-Margin Regularization (VMR)** adds a margin-based penalty to the critic loss that pushes Q-values of VLM-preferred actions above non-preferred actions by a fixed margin, shaping the value landscape without replacing the RL reward signal. (2) **Advantage-Weighted Action Guidance (AWAG)** adds a weighted behavioral cloning term to the actor loss -- the weight is the exponentiated advantage of the VLM-suggested action, so the policy only imitates the VLM when it agrees with what RL already considers good. (3) **Conditional Contrastive Action Alignment (CCAA)** uses CLIP to produce dense per-step reward bonuses, but conditions CLIP prompts on discretized speed and navigation command (e.g., "vehicle turning left at moderate speed") to fix CLIP's well-known dynamic blindness problem where it scores static scenes higher than moving ones. The bonus is computed as a normalized margin between the current-action prompt score and context-specific anchor scores.

**Practical Payoff.** A lightweight CNN+MLP policy trained with Found-RL achieves near-VLM driving performance (~95% of a billion-parameter VLM) while running at ~500 FPS -- making it deployable in real-time systems where the VLM itself cannot run.

## Step-by-Step Workflow

1. **Set up the simulation-inference separation.** Create two process groups: (a) rollout workers that step the environment and collect `(obs, action, reward, next_obs)` tuples, and (b) a VLM inference server that runs on a dedicated GPU. Connect them via a thread-safe request queue (e.g., `multiprocessing.Queue` or Redis stream) and a results store (shared memory dict keyed by `(env_id, step_id)`).

2. **Implement the request queue protocol.** Each rollout worker, after collecting an observation, serializes `(env_id, step_id, camera_image, ego_state)` and pushes it to the queue. The worker does NOT block -- it continues collecting experience. Mark the step as "pending VLM guidance" in the replay buffer.

3. **Build the VLM micro-batch server.** The server process drains the queue in micro-batches (e.g., batch size 8-16), runs VLM inference to produce action suggestions and text rationales, and writes results back to the shared store. Use dynamic batching with a timeout (e.g., 50ms) to balance latency and throughput.

4. **Implement Value-Margin Regularization (VMR) in the critic update.** During critic training, for each transition where VLM guidance is available, add a hinge loss term: `L_vmr = max(0, margin - (Q(s, a_vlm) - Q(s, a_policy)))` where `a_vlm` is the VLM-suggested action and `margin` is a hyperparameter (typically 0.1-0.5). Scale this with a coefficient `lambda_vmr` (start with 0.1) and add it to the standard critic loss (e.g., TD error).

5. **Implement Advantage-Weighted Action Guidance (AWAG) in the actor update.** Compute the advantage of the VLM-suggested action: `A_vlm = Q(s, a_vlm) - V(s)`. Add a behavioral cloning loss weighted by `exp(A_vlm / temperature)`: `L_awag = -w * log(pi(a_vlm | s))` where `w = exp(A_vlm / tau)` and `tau` is a temperature (typically 1.0-5.0). Clip the weight to `[0, max_weight]` for stability. This ensures the policy only imitates the VLM when RL's own value estimates confirm the suggestion is good.

6. **Set up CLIP-based reward shaping with Conditional Contrastive Action Alignment.** Load a CLIP model (ViT-B/32 is sufficient). For each observation, construct a conditioned text prompt by discretizing the ego speed into bins (slow/moderate/fast) and the navigation command (straight/left/right/stop). Form the prompt: `"A vehicle going {speed} while {command}"`. Compute the CLIP similarity between the current camera image and this prompt.

7. **Compute the margin-based CLIP bonus.** For each speed-command context, maintain a running set of anchor scores from a reference policy or random rollouts. The bonus is: `r_clip = clip_coeff * (sim(image, prompt) - mean_anchor) / std_anchor`. This normalization per context bin eliminates CLIP's bias toward static scenes and makes the bonus informative across different driving situations. Use `clip_coeff` in range 0.05-0.2.

8. **Combine all losses and rewards.** The total reward becomes `r_total = r_env + r_clip`. The total critic loss is `L_critic = L_td + lambda_vmr * L_vmr`. The total actor loss is `L_actor = L_policy_gradient + lambda_awag * L_awag`. Tune `lambda_vmr` and `lambda_awag` via grid search or start with 0.1 each.

9. **Handle staleness in async VLM results.** When VLM guidance arrives late (50-100 steps after the observation), apply it only if the state similarity (e.g., cosine similarity of observation embeddings) between the current state and the guided state exceeds a threshold. Discard stale guidance beyond a configurable TTL (e.g., 200 steps).

10. **Anneal VLM influence over training.** As the RL policy improves, gradually reduce `lambda_vmr`, `lambda_awag`, and `clip_coeff` on a linear or cosine schedule. The policy should eventually stand on its own without VLM crutches. Monitor the KL divergence between the policy and VLM-suggested action distributions to decide when to cut off.

## Concrete Examples

**Example 1: Async VLM Server for CARLA Autonomous Driving**

User: "I have a CARLA RL training loop for autonomous driving. I want to add VLM guidance from Qwen-VL but inference takes 800ms per image and my sim runs at 200 FPS. How do I integrate them?"

Approach:
1. Separate the VLM into its own process with a GPU and a request queue
2. Rollout workers push camera frames to the queue without blocking
3. VLM server batches 8 frames at a time, returning action suggestions
4. Results are stored in a shared buffer keyed by (env_id, step_id)
5. During training, check the buffer for available VLM guidance per transition

Output:
```python
import torch.multiprocessing as mp
from collections import defaultdict
import time

class VLMGuidanceBuffer:
    """Thread-safe buffer for async VLM results."""
    def __init__(self, ttl_steps=200):
        self.manager = mp.Manager()
        self.store = self.manager.dict()
        self.ttl_steps = ttl_steps

    def put(self, env_id, step_id, action_suggestion, rationale):
        self.store[(env_id, step_id)] = {
            'action': action_suggestion,
            'rationale': rationale,
            'timestamp': step_id
        }

    def get(self, env_id, current_step, max_staleness=None):
        staleness = max_staleness or self.ttl_steps
        for offset in range(staleness):
            key = (env_id, current_step - offset)
            if key in self.store:
                return self.store[key]
        return None

class VLMInferenceServer(mp.Process):
    """Runs on dedicated GPU, drains queue in micro-batches."""
    def __init__(self, request_queue, guidance_buffer, batch_size=8,
                 batch_timeout_ms=50):
        super().__init__(daemon=True)
        self.request_queue = request_queue
        self.guidance_buffer = guidance_buffer
        self.batch_size = batch_size
        self.batch_timeout = batch_timeout_ms / 1000.0

    def run(self):
        model = load_vlm("Qwen/Qwen-VL-Chat", device="cuda:1")
        while True:
            batch = self._collect_batch()
            if not batch:
                continue
            images = [req['image'] for req in batch]
            prompts = [self._build_prompt(req) for req in batch]
            results = model.batch_inference(images, prompts)
            for req, result in zip(batch, results):
                self.guidance_buffer.put(
                    req['env_id'], req['step_id'],
                    result['action'], result['rationale']
                )

    def _collect_batch(self):
        batch = []
        deadline = time.time() + self.batch_timeout
        while len(batch) < self.batch_size and time.time() < deadline:
            try:
                req = self.request_queue.get(timeout=0.01)
                batch.append(req)
            except Exception:
                continue
        return batch
```

**Example 2: VMR + AWAG Loss Implementation**

User: "I have a SAC agent. How do I add the Found-RL supervision losses?"

Approach:
1. Extend the critic loss with Value-Margin Regularization
2. Extend the actor loss with Advantage-Weighted Action Guidance
3. Both are conditioned on VLM guidance availability per sample

Output:
```python
class FoundRLSACAgent:
    def __init__(self, base_sac, vmr_margin=0.3, vmr_lambda=0.1,
                 awag_lambda=0.1, awag_temperature=2.0, awag_max_weight=5.0):
        self.sac = base_sac
        self.vmr_margin = vmr_margin
        self.vmr_lambda = vmr_lambda
        self.awag_lambda = awag_lambda
        self.awag_temp = awag_temperature
        self.awag_max_w = awag_max_weight

    def critic_loss(self, batch, vlm_actions=None):
        td_loss = self.sac.compute_td_loss(batch)
        if vlm_actions is None:
            return td_loss
        # VMR: push Q(s, a_vlm) above Q(s, a_policy) by margin
        q_vlm = self.sac.critic(batch.states, vlm_actions)
        q_policy = self.sac.critic(batch.states, batch.actions)
        vmr_loss = torch.relu(
            self.vmr_margin - (q_vlm - q_policy)
        ).mean()
        return td_loss + self.vmr_lambda * vmr_loss

    def actor_loss(self, batch, vlm_actions=None):
        pg_loss = self.sac.compute_policy_loss(batch)
        if vlm_actions is None:
            return pg_loss
        # AWAG: BC toward VLM action, weighted by advantage
        q_vlm = self.sac.critic(batch.states, vlm_actions).detach()
        v = self.sac.compute_value(batch.states).detach()
        advantage = q_vlm - v
        weights = torch.exp(advantage / self.awag_temp)
        weights = weights.clamp(max=self.awag_max_w)
        log_prob = self.sac.actor.log_prob(batch.states, vlm_actions)
        awag_loss = -(weights * log_prob).mean()
        return pg_loss + self.awag_lambda * awag_loss
```

**Example 3: CLIP Reward Shaping with Conditional Contrastive Action Alignment**

User: "My CLIP reward keeps giving high scores to the car being parked. How do I fix this dynamic blindness?"

Approach:
1. Condition CLIP prompts on discretized speed and navigation command
2. Maintain per-context anchor statistics from reference rollouts
3. Compute normalized margin-based bonus instead of raw similarity

Output:
```python
import clip
import torch
import numpy as np
from collections import defaultdict

class ConditionalCLIPReward:
    SPEED_BINS = [(0, 5, "stopped"), (5, 20, "slow"),
                  (20, 40, "moderate"), (40, 999, "fast")]
    COMMANDS = ["going straight", "turning left",
                "turning right", "stopping"]

    def __init__(self, clip_model="ViT-B/32", coeff=0.1, device="cuda"):
        self.model, self.preprocess = clip.load(clip_model, device=device)
        self.coeff = coeff
        self.device = device
        # Running stats per (speed_bin, command) context
        self.anchor_stats = defaultdict(lambda: {"scores": []})

    def _discretize_speed(self, speed_kmh):
        for lo, hi, label in self.SPEED_BINS:
            if lo <= speed_kmh < hi:
                return label
        return "moderate"

    def _build_prompt(self, speed_label, command):
        return f"A vehicle {command} at {speed_label} speed safely"

    def compute_reward(self, image, speed_kmh, nav_command_idx,
                       update_anchors=True):
        speed_label = self._discretize_speed(speed_kmh)
        command = self.COMMANDS[nav_command_idx]
        context_key = (speed_label, command)
        prompt = self._build_prompt(speed_label, command)
        img_input = self.preprocess(image).unsqueeze(0).to(self.device)
        txt_input = clip.tokenize([prompt]).to(self.device)
        with torch.no_grad():
            img_feat = self.model.encode_image(img_input)
            txt_feat = self.model.encode_text(txt_input)
            sim = torch.cosine_similarity(img_feat, txt_feat).item()

        stats = self.anchor_stats[context_key]
        if update_anchors:
            stats["scores"].append(sim)
        if len(stats["scores"]) < 10:
            return 0.0  # Not enough anchors yet
        mean_a = np.mean(stats["scores"])
        std_a = max(np.std(stats["scores"]), 1e-6)
        bonus = self.coeff * (sim - mean_a) / std_a
        return float(np.clip(bonus, -1.0, 1.0))
```

## Best Practices

- **Do** start with VMR alone before adding AWAG -- VMR shapes the critic landscape and gives AWAG a more reliable advantage signal to weight against.
- **Do** use micro-batched dynamic batching on the VLM server with a timeout (50-100ms). This balances throughput and freshness better than fixed-size batching.
- **Do** log the fraction of training transitions that have VLM guidance attached. If it drops below 10%, the VLM server is a bottleneck and you should increase batch size, add a second server, or reduce observation submission rate.
- **Avoid** applying VLM guidance that is more than 200 environment steps old. State distributions shift fast in RL; stale guidance can harm more than help.
- **Avoid** setting `awag_temperature` too low (below 0.5). This makes AWAG weights spike on slightly-positive advantages, overpowering the RL gradient. Start at 2.0 and decrease only after the policy is stable.
- **Avoid** using raw CLIP similarity as a reward without context conditioning. CLIP has a strong static-scene bias and will reward the agent for stopping and producing pretty-but-useless frames.

## Error Handling

- **VLM server crashes mid-training:** The rollout workers must tolerate a missing VLM server. If the guidance buffer returns `None` for all recent steps, skip VMR/AWAG losses gracefully (fall back to pure RL). Implement a health-check ping every 30 seconds and auto-restart the server process.
- **Advantage explosion in AWAG:** If VLM-suggested actions have extremely high Q-values, the `exp(A/tau)` weight overflows. Always clamp weights to `[0, max_weight]` (default 5.0) and use float32.
- **CLIP anchor statistics cold start:** For the first ~1000 steps, the per-context anchor bins have few samples and produce noisy bonuses. Return zero CLIP bonus until at least 10 samples per context bin are collected.
- **Queue backpressure:** If the request queue grows beyond a threshold (e.g., 1000 items), start dropping the oldest requests rather than blocking workers. Stale observations in the queue waste VLM compute.
- **GPU OOM on VLM server:** Reduce micro-batch size or use 4-bit quantization (e.g., bitsandbytes) for the VLM. The guidance quality degrades minimally with quantization.

## Limitations

- **VLM quality ceiling:** The RL policy cannot surpass the VLM's own driving ability through VMR/AWAG alone. If the VLM gives bad action suggestions, VMR and AWAG will actively harm training. Validate VLM quality independently first.
- **Simulation-specific:** The async framework assumes the simulation can run independently of VLM output. Environments where VLM guidance must be available *before* each action (e.g., LLM-as-policy) cannot use this decoupled architecture.
- **CLIP context bins require domain tuning:** The speed bins and navigation commands are tailored to autonomous driving. Applying CCAA to other domains (robotics, games) requires defining domain-appropriate context variables and prompt templates.
- **Multi-GPU required:** Running VLM inference alongside RL training practically requires at least 2 GPUs -- one for the VLM server, one for simulation + RL. Single-GPU setups lose most of the async throughput benefit.
- **No safety guarantees:** The VLM guidance is advisory. The RL policy can still learn unsafe behaviors if the environment reward does not penalize them. Found-RL is a performance technique, not a safety technique.

## Reference

**Paper:** [Found-RL: Foundation Model-Enhanced Reinforcement Learning for Autonomous Driving](https://arxiv.org/abs/2602.10458v1) (Qu et al., 2026). Focus on Section 3 for the asynchronous architecture, Section 4 for VMR/AWAG loss formulations, and Section 5 for Conditional Contrastive Action Alignment. The key insight: decouple VLM inference from the simulation loop via async queuing, then use three complementary supervision signals to transfer VLM knowledge without VLM latency.