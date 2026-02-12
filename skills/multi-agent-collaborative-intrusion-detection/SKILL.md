---
name: "multi-agent-collaborative-intrusion-detection"
description: "Build multi-agent intrusion detection systems using LLM-enhanced collaborative agents for network traffic classification. Triggers: 'build an intrusion detection system', 'multi-agent IDS', 'classify network traffic with agents', 'LLM-based anomaly detection', 'detect network intrusions with AI agents', 'collaborative threat detection pipeline'"
---

# Multi-Agent Collaborative Intrusion Detection

This skill enables Claude to design and implement multi-agent intrusion detection systems (IDS) inspired by the LLM-enhanced agentic AI framework from Li et al. (2026). The core technique decomposes network intrusion detection into three specialized agents — a Perception and Memory Agent that transforms raw traffic into visual features via diffusion models, a Reasoning Agent that uses LLM-guided optimization for intelligent feature selection, and an Adaptive Classification Agent that selects models based on available resources. This perception-reasoning-action loop achieves >90% classification accuracy across benchmark datasets while adapting to resource-constrained environments like IoT and edge networks.

## When to Use

- When the user asks to build a network intrusion detection system using multiple collaborating AI agents
- When implementing an IDS pipeline that must adapt to resource-constrained devices (IoT, edge, UAV)
- When the user wants to classify network traffic (benign vs. attack types like DDoS, port scanning, SQL injection) using LLM-enhanced agents
- When designing a system where raw packet data needs automated feature extraction without manual engineering
- When the user needs an IDS that handles multiple attack categories with balanced precision and recall
- When building anomaly detection that uses diffusion models to learn normal traffic distributions
- When the user asks to integrate LLMs into a cybersecurity detection pipeline for intelligent decision-making

## Key Technique

The framework replaces monolithic IDS classifiers with three specialized agents operating in a continuous perception-reasoning-action loop. The **Perception and Memory Agent** converts raw network sessions into 2D grayscale images (mapping packet bytes to pixel intensities), then trains a self-supervised denoising diffusion probabilistic model (DDPM) on benign traffic. This learns the distribution of normal traffic without labeled attack data, producing universal feature embeddings stored in a persistent memory module. Anomalous traffic deviates from this learned distribution, providing a natural detection signal.

The **Reasoning Agent** embeds LLM capabilities to perform intelligent feature selection. It constructs an offline knowledge repository using LLM-guided particle swarm optimization (PSO): the LLM observes PSO iteration progress, diagnoses issues like premature convergence, and dynamically adjusts swarm parameters (inertia weight, cognitive/social coefficients). At inference time, it performs zero-shot generalization — correlating low-confidence alerts across multiple sources to distinguish false positives from genuine distributed attacks without retraining.

The **Adaptive Classification Agent** maintains a model registry ranging from lightweight models (LightGBM) to heavier alternatives, selecting the appropriate classifier based on real-time device resource status (CPU, memory, battery). This makes the framework deployable across heterogeneous environments where not every node can run a full deep learning stack.

## Step-by-Step Workflow

1. **Ingest and sessionize raw network traffic.** Parse PCAP files or live packet captures into discrete network sessions (flows). Extract standard flow-level fields: source/destination IP, ports, protocol, packet sizes, inter-arrival times, flags, and payload bytes. Use libraries like `scapy`, `pyshark`, or `cicflowmeter`.

2. **Convert sessions to 2D grayscale images.** Map each session's raw byte payload into a fixed-size 2D pixel array (e.g., 28x28 or 32x32). Each byte becomes a pixel intensity (0-255). Pad short sessions with zeros; truncate long ones. This eliminates manual feature engineering and enables visual representation learning.

3. **Train a DDPM on benign traffic images.** Using only normal/benign traffic samples, train a denoising diffusion probabilistic model to learn the distribution of legitimate traffic. At inference, compute the reconstruction error for new samples — high error signals anomalous traffic. Store the trained encoder's feature embeddings in a memory module `M = {phi_1, phi_2, ..., phi_n}`.

4. **Build the LLM-guided feature selection pipeline.** Implement PSO with an LLM advisor loop: (a) initialize a swarm of candidate feature subsets, (b) evaluate fitness using a lightweight classifier on validation data, (c) after each iteration, serialize the population state (best fitness, diversity metrics, convergence rate) into a structured prompt, (d) query the LLM to diagnose stagnation and suggest parameter adjustments, (e) apply LLM-recommended corrections and continue optimization until convergence.

5. **Construct the offline knowledge repository.** Store the optimized feature subsets, their associated classification boundaries, and the LLM's reasoning traces as a queryable knowledge base. Index by attack category (DDoS, probe, injection, spoofing) so the reasoning agent can retrieve relevant detection context at inference time.

6. **Implement the adaptive classifier registry.** Register multiple pre-trained classifiers with metadata: model type (LightGBM, Random Forest, small MLP, CNN), accuracy on validation set, inference latency, and memory footprint. Build a selection function that takes current device resource metrics and returns the best feasible model.

7. **Wire the perception-reasoning-action loop.** Connect the three agents: Perception Agent outputs feature vectors to shared memory -> Reasoning Agent queries memory and knowledge repository, produces enriched feature vectors with confidence scores -> Classification Agent selects a model, classifies the traffic, and returns the verdict (benign, attack type, confidence).

8. **Implement cross-agent alert correlation.** When the Reasoning Agent receives low-confidence alerts from multiple sources or sensors, aggregate them using temporal and spatial correlation. Use the LLM to reason over the combined context — a cluster of low-confidence DDoS alerts from different network segments likely indicates a real distributed attack.

9. **Evaluate on benchmark datasets.** Test against standard IDS benchmarks: Edge-IIoTset, USTC-TFC, ISCX-VPN, CICIDS2017, or NSL-KDD. Report per-class accuracy, precision, recall, F1-score, and macro-averages. Target >90% overall accuracy with balanced metrics across attack categories.

10. **Deploy with resource-aware scaling.** Package each agent as an independent service (container or lightweight process). The Classification Agent monitors host resources and downgrades/upgrades its model selection in real time. Add health checks and fallback logic so degraded nodes still provide basic detection.

## Concrete Examples

**Example 1: Building a Multi-Agent IDS from PCAP Data**

User: "I have PCAP files of network traffic. Build me a multi-agent intrusion detection system that classifies traffic as benign or malicious."

Approach:
1. Parse PCAPs into session flows using `scapy`:
```python
from scapy.all import rdpcap, PcapReader
import numpy as np

def pcap_to_sessions(pcap_path, image_size=32):
    """Convert PCAP to session grayscale images."""
    packets = rdpcap(pcap_path)
    sessions = packets.sessions()
    images = []
    for session_key, session_packets in sessions.items():
        raw_bytes = b"".join(bytes(pkt) for pkt in session_packets)
        pixel_count = image_size * image_size
        byte_array = np.frombuffer(raw_bytes[:pixel_count], dtype=np.uint8)
        if len(byte_array) < pixel_count:
            byte_array = np.pad(byte_array, (0, pixel_count - len(byte_array)))
        images.append(byte_array.reshape(image_size, image_size))
    return np.array(images)
```

2. Train the Perception Agent's DDPM on benign samples:
```python
# perception_agent.py
import torch
from diffusers import DDPMScheduler, UNet2DModel

class PerceptionAgent:
    def __init__(self, image_size=32):
        self.model = UNet2DModel(
            sample_size=image_size, in_channels=1, out_channels=1,
            block_out_channels=(64, 128, 256), layers_per_block=2
        )
        self.scheduler = DDPMScheduler(num_train_timesteps=1000)
        self.memory = {}

    def train_on_benign(self, benign_images, epochs=50):
        """Learn normal traffic distribution."""
        # Standard DDPM training loop on benign-only data
        ...

    def extract_features(self, image):
        """Return feature embedding and reconstruction error."""
        with torch.no_grad():
            embedding = self.model.mid_block(image)  # intermediate features
            recon_error = self.compute_reconstruction_error(image)
        self.memory[hash(image.tobytes())] = embedding
        return embedding, recon_error
```

3. Implement the Reasoning Agent with LLM-guided feature selection:
```python
# reasoning_agent.py
import openai

class ReasoningAgent:
    def __init__(self, llm_client, knowledge_base):
        self.llm = llm_client
        self.kb = knowledge_base

    def llm_guided_pso_step(self, population_state):
        """Ask LLM to diagnose PSO and suggest corrections."""
        prompt = f"""You are a network security optimization expert.
Current PSO state for IDS feature selection:
- Iteration: {population_state['iteration']}
- Best fitness (F1-score): {population_state['best_fitness']:.4f}
- Population diversity: {population_state['diversity']:.4f}
- Stagnation count: {population_state['stagnation']}

Diagnose any convergence issues and recommend parameter adjustments
for inertia_weight, cognitive_coeff, and social_coeff."""

        response = self.llm.chat.completions.create(
            model="gpt-4", messages=[{"role": "user", "content": prompt}]
        )
        return self.parse_pso_params(response.choices[0].message.content)

    def correlate_alerts(self, alerts):
        """Use LLM to reason over low-confidence multi-source alerts."""
        alert_summary = self.format_alerts(alerts)
        prompt = f"""Analyze these network alerts for correlated attack patterns:
{alert_summary}
Determine if these represent a coordinated attack or false positives."""
        response = self.llm.chat.completions.create(
            model="gpt-4", messages=[{"role": "user", "content": prompt}]
        )
        return response.choices[0].message.content
```

4. Build the Adaptive Classification Agent:
```python
# classification_agent.py
import psutil
import lightgbm as lgb

class ClassificationAgent:
    def __init__(self):
        self.model_registry = {
            "lightgbm": {"model": None, "mem_mb": 50, "latency_ms": 2},
            "random_forest": {"model": None, "mem_mb": 200, "latency_ms": 10},
            "mlp": {"model": None, "mem_mb": 500, "latency_ms": 25},
        }

    def select_model(self):
        """Pick best model that fits current resource budget."""
        available_mem = psutil.virtual_memory().available / (1024 * 1024)
        cpu_percent = psutil.cpu_percent()
        for name in ["mlp", "random_forest", "lightgbm"]:  # prefer accuracy
            entry = self.model_registry[name]
            if entry["mem_mb"] < available_mem * 0.3 and cpu_percent < 80:
                return name, entry["model"]
        return "lightgbm", self.model_registry["lightgbm"]["model"]

    def classify(self, features):
        model_name, model = self.select_model()
        prediction = model.predict(features.reshape(1, -1))
        return {"label": prediction[0], "model_used": model_name}
```

Output: A three-agent pipeline where traffic flows through Perception -> Reasoning -> Classification, producing labeled verdicts like `{"label": "DDoS", "confidence": 0.94, "model_used": "random_forest"}`.

---

**Example 2: Adding LLM-Powered Alert Correlation to an Existing IDS**

User: "My existing IDS generates too many false positives. Can you add an LLM reasoning layer that correlates alerts before raising alarms?"

Approach:
1. Wrap the existing IDS output as input to a Reasoning Agent
2. Batch alerts within a time window (e.g., 30 seconds)
3. Query the LLM with structured alert context for correlation

```python
# alert_correlator.py
from collections import defaultdict
import time

class AlertCorrelator:
    def __init__(self, llm_client, window_seconds=30):
        self.llm = llm_client
        self.window = window_seconds
        self.buffer = defaultdict(list)

    def ingest_alert(self, alert):
        """Buffer alerts by time window."""
        window_key = int(time.time() // self.window)
        self.buffer[window_key].append(alert)

    def correlate_window(self, window_key):
        alerts = self.buffer[window_key]
        if len(alerts) < 2:
            return alerts  # pass through single alerts

        prompt = f"""You are a SOC analyst. Review these {len(alerts)} IDS alerts
from a {self.window}s window and determine which are correlated attacks
vs. false positives.

Alerts:
{self._format_alerts(alerts)}

For each alert, respond with: REAL_ATTACK, FALSE_POSITIVE, or NEEDS_INVESTIGATION.
Group correlated alerts into attack campaigns."""

        response = self.llm.chat.completions.create(
            model="gpt-4", messages=[{"role": "user", "content": prompt}]
        )
        return self._parse_verdicts(response.choices[0].message.content, alerts)
```

Output: False positive rate drops as the LLM identifies that 15 low-confidence "port scan" alerts from different source IPs targeting the same subnet constitute one coordinated reconnaissance campaign, not 15 independent events.

---

**Example 3: Resource-Aware IDS for Edge/IoT Deployment**

User: "I need an IDS that works on Raspberry Pi nodes in a drone fleet. It should use lighter models when resources are scarce."

Approach:
1. Pre-train multiple classifiers at different complexity levels
2. Deploy the Classification Agent with resource monitoring
3. Implement graceful degradation

```python
# edge_ids.py
class EdgeIDS:
    def __init__(self):
        self.models = self._load_tiered_models()

    def _load_tiered_models(self):
        return {
            "tier1_minimal": {  # <30MB RAM, <5ms latency
                "model": load_decision_tree("ids_dt.pkl"),
                "accuracy": 0.85, "ram_mb": 20
            },
            "tier2_balanced": {  # <100MB RAM, <15ms
                "model": load_lightgbm("ids_lgbm.pkl"),
                "accuracy": 0.92, "ram_mb": 80
            },
            "tier3_full": {  # <500MB RAM, <50ms
                "model": load_mlp("ids_mlp.pkl"),
                "accuracy": 0.96, "ram_mb": 400
            },
        }

    def detect(self, flow_features, device_resources):
        tier = self._select_tier(device_resources)
        model_info = self.models[tier]
        prediction = model_info["model"].predict(flow_features)
        return {
            "prediction": prediction,
            "tier": tier,
            "expected_accuracy": model_info["accuracy"]
        }

    def _select_tier(self, resources):
        available_ram = resources["available_ram_mb"]
        battery_pct = resources.get("battery_pct", 100)
        if available_ram < 50 or battery_pct < 15:
            return "tier1_minimal"
        elif available_ram < 200 or battery_pct < 40:
            return "tier2_balanced"
        return "tier3_full"
```

## Best Practices

- **Do:** Train the DDPM exclusively on verified benign traffic. Mixing in attack samples corrupts the normal distribution baseline and defeats the purpose of anomaly-based detection.
- **Do:** Structure LLM prompts with concrete numeric metrics (fitness scores, diversity indices, convergence counts) rather than vague descriptions. The LLM reasons better with quantified state.
- **Do:** Log every LLM reasoning trace and PSO parameter adjustment. This creates an auditable knowledge base and helps debug detection failures.
- **Do:** Test on multiple benchmark datasets (Edge-IIoTset, CICIDS2017, NSL-KDD) to validate generalization. A model that only works on one dataset is overfitting to that traffic profile.
- **Avoid:** Sending raw packet payloads to the LLM. Convert to structured features or summaries first — raw bytes waste tokens and may contain sensitive data.
- **Avoid:** Using a single fixed classifier. The entire point of the adaptive agent is runtime model selection; hardcoding one model negates the resource-awareness benefit.

## Error Handling

- **DDPM training divergence:** If reconstruction errors plateau at high values, the benign training set may contain mislabeled attack traffic. Audit the training data by clustering samples and removing outliers before retraining.
- **LLM returns unparseable PSO parameters:** Validate all LLM-suggested parameter values against hard bounds (e.g., inertia_weight in [0.1, 0.9]). Fall back to default PSO parameters if parsing fails.
- **Resource monitoring fails on edge devices:** If `psutil` or equivalent is unavailable, default to the lightest model tier. Never fail to classify because resource detection broke.
- **Alert correlation hallucinates attack patterns:** Ground LLM verdicts by requiring them to cite specific alert fields (source IPs, timestamps, ports). If the LLM references data not in the input, discard the verdict and fall back to threshold-based correlation.
- **Class imbalance in benchmark datasets:** Many IDS datasets have heavily skewed class distributions. Use stratified sampling, SMOTE, or class-weighted loss functions to prevent the classifier from ignoring rare attack types.

## Limitations

- The DDPM-based perception requires meaningful packet payloads; fully encrypted traffic (TLS 1.3 with no metadata) produces near-uniform images with poor discriminative features. Flow-level metadata (timing, sizes) may be more useful in these cases.
- LLM-guided PSO adds significant latency to the offline training phase. This is acceptable for building the knowledge repository but not for real-time feature selection at inference.
- The framework assumes agents can communicate with low latency. In severely partitioned networks (disconnected drone swarms), the collaborative loop degrades and each node must fall back to local-only detection.
- LLM API calls introduce cost and availability dependencies. For production edge deployments, consider distilling the LLM reasoning into static rules or a small local model after the knowledge repository is built.
- Accuracy >90% on benchmark datasets does not guarantee the same performance on production traffic, which may contain novel attack types not represented in training data.

## Reference

Li, H., Kang, H., Li, J., Sun, G., & Zhang, R. (2026). *Multi-Agent Collaborative Intrusion Detection for Low-Altitude Economy IoT: An LLM-Enhanced Agentic AI Framework.* arXiv:2601.17817v1. [https://arxiv.org/abs/2601.17817v1](https://arxiv.org/abs/2601.17817v1)

Focus on: Section III (agentic AI framework architecture), Section IV (multi-agent collaboration protocol with perception-reasoning-action loop), and experimental results comparing per-dataset accuracy across Edge-IIoTset, USTC-TFC, and ISCX-VPN benchmarks.