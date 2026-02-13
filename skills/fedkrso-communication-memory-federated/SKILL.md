---
name: "fedkrso-communication-memory-federated"
description: "Implement FedKRSO (Federated K-Seed Random Subspace Optimization) for communication- and memory-efficient federated fine-tuning of large language models. Use when: 'set up federated LLM fine-tuning with low bandwidth', 'implement memory-efficient federated full fine-tuning', 'build a FedKRSO training loop', 'reduce communication cost in federated learning', 'fine-tune LLMs on edge devices with federated learning', 'implement random subspace optimization for FL'."
---

# FedKRSO: Communication and Memory Efficient Federated Fine-Tuning

This skill enables Claude to implement FedKRSO, a federated learning algorithm that achieves near full fine-tuning (FFT) accuracy on LLMs while slashing communication and memory costs. Unlike LoRA or other PEFT methods that sacrifice model quality for efficiency, FedKRSO projects gradient updates into shared random low-dimensional subspaces generated from K seeds, then transmits only compact accumulators instead of full model parameters. This gives clients the expressiveness of FFT at the memory cost closer to PEFT.

## When to Use

- When the user wants to fine-tune an LLM across multiple federated clients without transmitting full model weights each round
- When implementing federated learning for resource-constrained edge devices that cannot hold full optimizer states in memory
- When the user asks to reduce communication overhead in federated fine-tuning while maintaining accuracy close to full fine-tuning
- When comparing or implementing alternatives to LoRA/PEFT in federated settings
- When building a training pipeline where clients share random seeds instead of model parameters
- When the user needs to implement random subspace gradient compression for distributed optimization

## Key Technique

**The core insight**: Instead of each client computing and transmitting gradients in the full d-dimensional parameter space, FedKRSO has the server generate K random seeds. Each seed deterministically produces a Gaussian projection matrix P_k of shape (r x d) where r << d. Clients perform gradient descent steps projected through these matrices, accumulating compressed updates B_k of shape (d_m x r) rather than full (d_m x d_n) gradients. Because both server and clients share the seeds, either side can reconstruct the projection matrices on-the-fly without transmitting them.

**How updates flow**: Training is divided into I intervals of J local iterations each. At each interval, a client samples a seed s_k from the shared set, generates P_k ~ N(0, (1/r)I), computes the compressed gradient G^B = nabla_f(W) @ P_k^T, updates the model via W <- W - eta * G^B @ P_k, and accumulates B_k <- B_k - eta * G^B. After all local training, clients send only the K accumulators {B_k} to the server. The server averages accumulators across clients and reconstructs the global update: W^(t+1) = W^t + sum_k(B_k^(t+1) @ P_k^t).

**Why this outperforms PEFT**: LoRA fixes a static low-rank structure throughout training, constraining the reachable parameter space. FedKRSO resamples random subspaces each round, so over many rounds the union of explored subspaces spans a much richer region of parameter space -- approaching FFT expressiveness. Meanwhile, memory stays low because only one (d_m x r) projection and its accumulator are active at a time, and optimizer momentum states are stored in the compressed r-dimensional space instead of the full parameter space.

## Step-by-Step Workflow

1. **Define the shared seed set on the server.** Generate K integer seeds (e.g., K=4 to K=16) using a secure random source. Store them as a list. These seeds are the only "model description" sent to clients at the start of each round.

2. **Implement the projection matrix generator.** Write a function `generate_projection(seed, r, d) -> Tensor(r, d)` that initializes a PRNG with the given seed and draws entries from N(0, 1/r). Both server and client code must use the identical PRNG (e.g., `torch.Generator().manual_seed(seed)`).

3. **Implement client-side local training (Algorithm 2).** For each round:
   - Receive the current K accumulators {B_k} and seeds {s_k} from the server.
   - Reconstruct the current model: W = W_base + sum_k(B_k @ P_k).
   - Divide I*J local steps into I intervals. At each interval, sample a seed s_k uniformly from the K seeds, generate P_k, and run J gradient steps:
     ```
     G_compressed = grad_loss(W) @ P_k.T       # shape: (d_m, r)
     W = W - lr * G_compressed @ P_k            # update in full space
     B_k = B_k - lr * G_compressed              # accumulate in subspace
     ```
   - Use Adam-style momentum on the compressed gradient G_compressed (states are (d_m x r), not (d_m x d_n)).

4. **Implement server-side aggregation (Algorithm 1).** Collect accumulators from participating clients. Average them: `B_k_global = (1/N) * sum_n(B_k_n)` for each k in [K]. Optionally resample some or all seeds for the next round.

5. **Reconstruct the global model for evaluation.** On the server: `W_global = W_init + sum_k(B_k_global @ P_k)`. This is the only time the full parameter matrix is materialized; during training, clients never hold more than one (d_m x r) block at a time.

6. **Configure hyperparameters.** Set subspace rank r (typically 4-64; lower means more compression but slower convergence), number of seeds K (4-16; more seeds = richer subspace coverage per round), local intervals I, steps per interval J, and learning rate eta.

7. **Handle partial client participation.** In realistic FL, only a subset of clients participate each round. Weight the accumulator average by participation: `B_k = sum(B_k_n * w_n)` where w_n reflects client data size or uniform 1/|participating|.

8. **Add seed cycling logic.** Each round, optionally replace a fraction of seeds to increase subspace diversity over time. Fresh seeds explore new directions; keeping some seeds provides continuity for momentum states.

9. **Implement communication serialization.** Transmit only: (a) K integer seeds (negligible bytes), (b) K accumulators each of size (d_m x r) as float16/bfloat16 tensors. Total per-round upload per client: K * d_m * r * 2 bytes.

10. **Validate with a GLUE-style benchmark.** Run on SST-2, MRPC, CoLA, or QNLI to confirm FedKRSO matches or approaches FFT accuracy while using a fraction of the communication and memory budget. Compare against FedAvg (full FFT), FedAvg+LoRA, and standalone LoRA baselines.

## Concrete Examples

**Example 1: Implementing FedKRSO for RoBERTa on SST-2 with PyTorch**

User: "I want to fine-tune RoBERTa-base on SST-2 across 10 federated clients with minimal communication. Implement FedKRSO."

Approach:
1. Partition SST-2 train set across 10 clients (non-IID or IID split).
2. Set K=8 seeds, subspace rank r=16, I=2 intervals, J=5 steps per interval.
3. Implement projection generator and local training loop.

Output (key code structure):
```python
import torch
from transformers import RobertaForSequenceClassification

def generate_projection(seed, r, d, device):
    gen = torch.Generator(device=device).manual_seed(seed)
    P = torch.randn(r, d, generator=gen, device=device) / (r ** 0.5)
    return P

class FedKRSOClient:
    def __init__(self, model, seeds, r):
        self.model = model
        self.seeds = seeds  # list of K ints
        self.r = r
        self.d = sum(p.numel() for p in model.parameters())
        # Accumulators: one per seed, shape (d, r) stored sparse per layer
        self.accumulators = {s: torch.zeros(self.d, r) for s in seeds}

    def local_train(self, dataloader, lr, I, J):
        for interval in range(I):
            seed = self.seeds[torch.randint(len(self.seeds), (1,)).item()]
            P_k = generate_projection(seed, self.r, self.d, device='cuda')
            for step in range(J):
                batch = next(iter(dataloader))
                loss = self.model(**batch).loss
                loss.backward()
                # Flatten grads, project, update
                grad_flat = torch.cat([p.grad.flatten() for p in self.model.parameters()])
                G_compressed = grad_flat @ P_k.T  # shape: (r,)
                # For per-layer: reshape and apply
                update = G_compressed @ P_k        # shape: (d,)
                offset = 0
                for p in self.model.parameters():
                    numel = p.numel()
                    p.data -= lr * update[offset:offset+numel].view(p.shape)
                    offset += numel
                self.accumulators[seed] -= lr * G_compressed
                self.model.zero_grad()
        return self.accumulators

class FedKRSOServer:
    def __init__(self, model, K, r):
        self.d = sum(p.numel() for p in model.parameters())
        self.seeds = [torch.randint(0, 2**31, (1,)).item() for _ in range(K)]
        self.r = r
        self.accumulators = {s: torch.zeros(self.d, r) for s in self.seeds}

    def aggregate(self, client_accumulators_list):
        N = len(client_accumulators_list)
        for s in self.seeds:
            self.accumulators[s] = sum(ca[s] for ca in client_accumulators_list) / N

    def reconstruct_update(self):
        update = torch.zeros(self.d)
        for s in self.seeds:
            P_k = generate_projection(s, self.r, self.d, device='cpu')
            update += self.accumulators[s] @ P_k
        return update
```

**Example 2: Estimating communication savings**

User: "How much bandwidth does FedKRSO save compared to FedAvg for LLaMA-7B with K=8, r=32?"

Approach:
1. LLaMA-7B has ~7 billion parameters. In bfloat16: 7e9 * 2 = 14 GB per full model exchange.
2. FedKRSO sends K accumulators. Each accumulator conceptually has total entries = d * r, but d here is the total parameter count so per-layer accumulators are (layer_dim x r).
3. Total upload = K * d * r * 2 bytes = 8 * 7e9 * 32 * 2 bytes... That is larger than full model -- so r must be applied per-layer, not globally.

Output:
```
Per-layer analysis (e.g., a 4096x4096 weight matrix):
- Full model exchange: 4096 * 4096 * 2 = 33.5 MB per layer
- FedKRSO accumulator: 4096 * 32 * 2 = 256 KB per layer per seed
- With K=8 seeds: 8 * 256 KB = 2 MB per layer
- Compression ratio: 33.5 MB / 2 MB = ~16.8x per layer

For full LLaMA-7B (all layers):
- FedAvg upload: ~14 GB
- FedKRSO upload: ~14 GB / 16.8 = ~830 MB
- Savings: ~94% reduction in upload bandwidth per round
```

**Example 3: Adding FedKRSO to an existing Flower FL pipeline**

User: "I have a Flower federated learning setup. How do I swap in FedKRSO?"

Approach:
1. Modify the Flower `Client` to perform subspace-projected local training.
2. Override `get_parameters` to return serialized accumulators + seed indices instead of model weights.
3. Modify the `Strategy` (server) to aggregate accumulators and reconstruct the model.

Output:
```python
import flwr as fl
import pickle

class FedKRSOFlowerClient(fl.client.NumPyClient):
    def __init__(self, model, seeds, r, dataloader):
        self.krso = FedKRSOClient(model, seeds, r)
        self.dataloader = dataloader

    def get_parameters(self, config):
        # Return flattened accumulators as numpy arrays
        return [self.krso.accumulators[s].cpu().numpy() for s in self.krso.seeds]

    def set_parameters(self, parameters):
        for i, s in enumerate(self.krso.seeds):
            self.krso.accumulators[s] = torch.from_numpy(parameters[i])

    def fit(self, parameters, config):
        self.set_parameters(parameters)
        accs = self.krso.local_train(self.dataloader, lr=config["lr"], I=2, J=5)
        return self.get_parameters(config), len(self.dataloader.dataset), {}
```

## Best Practices

- **Do** use the same PRNG implementation (e.g., `torch.Generator` with `.manual_seed()`) on both server and all clients to ensure identical projection matrices from the same seed.
- **Do** apply projection per-layer rather than flattening the entire model into one vector. This keeps individual accumulator tensors manageable and avoids materializing a single (d x r) matrix for billions of parameters.
- **Do** start with a moderate rank r=16-32 and K=4-8 seeds, then tune based on your accuracy-vs-communication tradeoff.
- **Do** use bfloat16 or float16 for accumulators during transmission to halve bandwidth further.
- **Avoid** resampling all K seeds every round -- keep at least half stable so Adam momentum states in the compressed space remain useful across rounds.
- **Avoid** setting r too high (e.g., r > 128) as it erodes the communication savings and increases client memory. The sweet spot is typically r in [8, 64].

## Error Handling

- **Numerical instability in projections**: If r is very small (e.g., r=1), the random projection can amplify noise. Monitor training loss for divergence and increase r if loss spikes.
- **Seed mismatch between server and client**: If a client uses a different PRNG or seed ordering, the reconstructed model will be garbage. Add a checksum: after generating P_k, verify its first few entries match an expected value.
- **Client dropout mid-round**: If a client fails after partial training, its accumulators are stale. Exclude dropped clients from aggregation rather than using their last-known accumulators.
- **Memory overflow on accumulator reconstruction**: When calling `reconstruct_update()`, generating all K projection matrices simultaneously can OOM. Generate and apply one P_k at a time, accumulating the update incrementally.
- **Momentum state staleness after seed cycling**: When a seed is replaced, its associated Adam momentum (m, v) becomes meaningless. Reset momentum states for any newly introduced seed.

## Limitations

- **Not a drop-in replacement for LoRA adapters**: FedKRSO modifies the full weight matrix through projections, so it does not produce a compact adapter file. Deployment requires the full updated model.
- **Requires synchronized PRNG**: All clients and the server must use byte-identical random number generation from seeds. Cross-platform or cross-library PRNG differences will silently corrupt training.
- **Convergence is slower than centralized FFT**: The random subspace projection introduces variance. More FL rounds are needed to match centralized FFT accuracy, though each round is far cheaper.
- **Best suited for moderate model sizes**: For very small models (< 100M params), the overhead of managing seeds and projections may not justify the savings. For extremely large models (> 70B), per-layer accumulator memory still adds up.
- **Heterogeneous data amplifies subspace mismatch**: Under extreme non-IID data distributions, different clients may need different subspace directions. FedKRSO uses a shared subspace set, which can slow convergence in highly heterogeneous settings.

## Reference

**Paper**: [FedKRSO: Communication and Memory Efficient Federated Fine-Tuning of Large Language Models](https://arxiv.org/abs/2602.03019v1) (INFOCOM 2026)
**What to look for**: Algorithm 1 (server) and Algorithm 2 (client LocalTraining) for the complete pseudocode; Section V for convergence bounds under non-convex loss with heterogeneous data; Section VI for GLUE benchmark results comparing FedKRSO against FedAvg, LoRA, and other PEFT baselines across varying K, r, and client participation rates.