---
name: "multimodal-multi-agent-ransomware-analysis"
description: "Build multimodal multi-agent pipelines for ransomware classification using specialized per-modality agents, autoencoder feature extraction, fusion, and transformer-based classification with inter-agent feedback. Triggers: 'analyze ransomware samples', 'build a multi-agent malware classifier', 'classify ransomware families', 'multimodal malware analysis pipeline', 'ransomware detection with autoencoders', 'multi-agent security analysis framework'"
---

# Multimodal Multi-Agent Ransomware Analysis

This skill enables Claude to design and implement multimodal multi-agent systems for ransomware analysis and classification, following the architecture from Khan et al. (2026). The core technique assigns specialized agents to each data modality (static PE features, dynamic behavioral traces, network traffic), each using autoencoder-based feature extraction. A fusion agent integrates these latent representations, a transformer classifier identifies ransomware families, and an inter-agent feedback loop iteratively suppresses low-confidence features to improve accuracy. This approach achieves 0.936 Macro-F1 on family classification and supports confidence-aware abstention for safe real-world deployment.

## When to Use

- When the user asks to build a ransomware classification or detection system that handles multiple data sources (PE files, sandbox logs, PCAPs)
- When implementing a multi-agent pipeline where each agent specializes in a different analysis modality (static, dynamic, network)
- When the user wants to combine autoencoder feature extraction with transformer-based classification for malware analysis
- When designing an inter-agent feedback mechanism that iteratively refines feature quality across modalities
- When building a malware classifier that needs confidence-aware abstention to avoid misclassifying unknown or zero-day samples
- When the user asks to implement fusion strategies for heterogeneous security data sources
- When porting the AutoGen-based multi-agent ransomware framework to a working codebase

## Key Technique

The framework decomposes ransomware analysis into three parallel modality-specific pipelines, each managed by an autonomous agent. The **static agent** processes PE header metadata (imports, sections, entropy, opcode n-grams), the **dynamic agent** processes sandbox behavioral traces (API call sequences, registry modifications, file I/O, process trees), and the **network agent** processes captured traffic (DNS queries, HTTP requests, IP connection graphs, payload entropy). Each agent trains a modality-specific autoencoder to compress its raw features into a compact latent representation, discarding noise while preserving discriminative structure.

The **fusion agent** receives the three latent vectors and combines them via learned attention-weighted concatenation. Rather than naive concatenation, the fusion layer assigns per-modality attention weights conditioned on cross-modal consistency, so a modality producing low-confidence or contradictory features gets down-weighted. The fused representation is fed to a **transformer-based classifier** (multi-head self-attention over the fused feature sequence) that outputs both a family prediction and a calibrated confidence score.

The critical innovation is the **inter-agent feedback loop**: after each classification round, the transformer's per-modality attention weights and confidence scores are propagated back to each modality agent. Agents whose features contributed low-confidence signals retrain their autoencoders with adjusted reconstruction loss weighting, suppressing uninformative feature dimensions. Over ~100 epochs this loop converges monotonically, yielding +0.75 absolute improvement in agent quality. A final **confidence-aware abstention** layer rejects predictions below a tunable threshold, favoring "I don't know" over forced misclassification — essential for zero-day and polymorphic ransomware where some modalities may be entirely absent.

## Step-by-Step Workflow

1. **Define the modality schema.** Enumerate exactly which raw features each agent will consume. For static: PE header fields, import table hashes, section entropy, opcode 2-gram frequencies. For dynamic: ordered API call sequences (e.g., from Cuckoo/CAPE sandbox JSON), registry key operations, file write paths, process creation trees. For network: DNS query domains, HTTP host/URI pairs, destination IP/port tuples, per-flow byte histograms.

2. **Implement per-modality preprocessing pipelines.** Write a data loader for each modality that normalizes raw inputs into fixed-length numerical vectors. Static features: parse PE with `pefile`, extract ~200-dimensional feature vector. Dynamic features: tokenize API sequences into integer IDs, pad/truncate to fixed length (e.g., 512 calls). Network features: extract flow-level statistics from PCAP using `scapy` or `pyshark`, encode categorical fields (domains, IPs) via hashing.

3. **Build autoencoder agents.** For each modality, implement a symmetric autoencoder (encoder-decoder) in PyTorch or TensorFlow. Use 3-layer encoder: input_dim -> 256 -> 128 -> latent_dim (64), with ReLU activations and batch normalization. Train each autoencoder independently on reconstruction loss (MSE) for its modality. The encoder's latent output is the agent's feature representation.

4. **Implement the fusion agent.** Receive the three latent vectors (each 64-dim). Concatenate into a 192-dim vector. Apply a learned attention layer: a small MLP that outputs 3 scalar weights (softmax-normalized) representing each modality's relevance. Compute the weighted sum of the three latent vectors (each scaled by its weight) to produce a 64-dim fused representation.

5. **Build the transformer classifier.** Treat the fused representation as a sequence (reshape 64-dim into 8 tokens of 8-dim each, or use the three 64-dim modality vectors as a 3-token sequence). Apply 2 transformer encoder layers with 4 attention heads. Add a classification head (linear layer) mapping the [CLS] token output to num_families + 1 classes (the +1 for "benign"). Train with cross-entropy loss plus label smoothing (0.1).

6. **Wire the inter-agent feedback loop.** After each training epoch on the classifier, extract per-modality attention weights from the fusion layer and per-sample confidence (max softmax probability) from the classifier. For each modality agent, compute its average contribution weight. If a modality's weight falls below a threshold (e.g., 0.2), increase its autoencoder's reconstruction loss weight by a factor (e.g., 1.5x) and retrain for one additional epoch. This forces underperforming agents to learn better representations.

7. **Implement confidence-aware abstention.** After the classifier outputs a softmax distribution, compute the max class probability. If max_prob < abstention_threshold (start with 0.85, tune on validation set), output "UNKNOWN / ABSTAIN" instead of a family label. Log abstained samples for manual analyst review. Track the abstention rate — aim for <15% on known families, higher on zero-day.

8. **Orchestrate with AutoGen (or equivalent).** Define each modality agent, the fusion agent, and the classifier as AutoGen `AssistantAgent` instances. Use a `GroupChat` to coordinate message passing: modality agents send latent vectors to the fusion agent, the fusion agent sends the fused vector to the classifier, and the classifier sends feedback scores back to each modality agent. Implement the feedback loop as a repeating group chat round.

9. **Evaluate on a multi-family dataset.** Collect samples across known families (e.g., WannaCry, Locky, Cerber, Ryuk, REvil, Conti, plus benign). Split 70/15/15 train/val/test. Report Macro-F1 (target: >0.90), calibration error (ECE, target: <0.05), and abstention-adjusted accuracy. Compare against single-modality baselines and naive concatenation fusion.

10. **Deploy with monitoring.** Wrap the pipeline in a REST API or CLI tool. Accept a sample (PE file + sandbox report JSON + PCAP) as input. Return: predicted family, confidence score, per-modality contribution weights, and abstain flag. Log all predictions for drift detection.

## Concrete Examples

**Example 1: Building the static analysis agent**

User: "I have a directory of PE files. Help me build the static analysis agent with autoencoder feature extraction."

Approach:
1. Use `pefile` to parse each PE and extract: number of sections, total entropy, import count, has_debug, has_tls, timestamp delta, section name hashes, entry point offset, file size, and opcode 2-gram frequencies (top 100 n-grams).
2. Normalize each feature column to [0, 1] using min-max scaling fitted on training set.
3. Build a PyTorch autoencoder: Linear(150, 256) -> ReLU -> BN -> Linear(256, 128) -> ReLU -> BN -> Linear(128, 64) as encoder; mirror for decoder.
4. Train for 50 epochs with Adam (lr=1e-3), MSE loss, batch size 128.
5. Extract 64-dim latent vectors by running samples through the trained encoder.

Output:
```python
import pefile
import torch
import torch.nn as nn
import numpy as np
from pathlib import Path

class StaticFeatureExtractor:
    """Extract fixed-length feature vector from a PE file."""
    def __init__(self, top_ngrams=100):
        self.top_ngrams = top_ngrams
        self.scaler = None  # fit during training

    def extract(self, pe_path: str) -> np.ndarray:
        pe = pefile.PE(pe_path)
        features = []
        features.append(pe.FILE_HEADER.NumberOfSections)
        features.append(pe.OPTIONAL_HEADER.SizeOfImage)
        features.append(pe.OPTIONAL_HEADER.AddressOfEntryPoint)
        features.append(len(pe.OPTIONAL_HEADER.DATA_DIRECTORY))
        features.append(int(hasattr(pe, 'DIRECTORY_ENTRY_IMPORT')))
        num_imports = 0
        if hasattr(pe, 'DIRECTORY_ENTRY_IMPORT'):
            for entry in pe.DIRECTORY_ENTRY_IMPORT:
                num_imports += len(entry.imports)
        features.append(num_imports)
        # Section entropy
        for i, section in enumerate(pe.sections[:8]):
            features.append(section.get_entropy())
        features.extend([0.0] * max(0, 8 - len(pe.sections)))
        return np.array(features, dtype=np.float32)

class ModalityAutoencoder(nn.Module):
    def __init__(self, input_dim=150, latent_dim=64):
        super().__init__()
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 256), nn.ReLU(), nn.BatchNorm1d(256),
            nn.Linear(256, 128), nn.ReLU(), nn.BatchNorm1d(128),
            nn.Linear(128, latent_dim),
        )
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 128), nn.ReLU(), nn.BatchNorm1d(128),
            nn.Linear(128, 256), nn.ReLU(), nn.BatchNorm1d(256),
            nn.Linear(256, input_dim),
        )

    def forward(self, x):
        z = self.encoder(x)
        recon = self.decoder(z)
        return recon, z
```

**Example 2: Implementing the fusion agent with attention-weighted combination**

User: "I have three latent vectors from static, dynamic, and network agents. Build the fusion agent."

Approach:
1. Concatenate the three 64-dim latent vectors.
2. Pass through a small attention MLP that outputs 3 weights.
3. Compute the weighted combination.
4. Return fused vector plus the attention weights (needed for feedback).

Output:
```python
class FusionAgent(nn.Module):
    def __init__(self, latent_dim=64, num_modalities=3):
        super().__init__()
        self.attention = nn.Sequential(
            nn.Linear(latent_dim * num_modalities, 64),
            nn.ReLU(),
            nn.Linear(64, num_modalities),
        )
        self.num_modalities = num_modalities
        self.latent_dim = latent_dim

    def forward(self, modality_vectors: list[torch.Tensor]):
        # modality_vectors: list of 3 tensors, each (batch, 64)
        concat = torch.cat(modality_vectors, dim=-1)  # (batch, 192)
        weights = torch.softmax(self.attention(concat), dim=-1)  # (batch, 3)
        fused = torch.zeros_like(modality_vectors[0])
        for i, vec in enumerate(modality_vectors):
            fused += weights[:, i:i+1] * vec
        return fused, weights
```

**Example 3: Wiring the inter-agent feedback loop**

User: "How do I implement the feedback mechanism that sends classifier confidence back to each modality agent?"

Approach:
1. After each training epoch, collect per-modality attention weights from the fusion agent and per-sample confidence from the classifier.
2. Compute mean attention weight per modality across the batch.
3. For modalities below a contribution threshold, increase reconstruction loss weight and retrain.

Output:
```python
def feedback_loop(
    modality_agents: dict,      # {"static": autoencoder, "dynamic": ..., "network": ...}
    fusion_agent: FusionAgent,
    classifier: TransformerClassifier,
    data_loaders: dict,         # per-modality data loaders
    contribution_threshold=0.2,
    loss_boost_factor=1.5,
    retrain_epochs=1,
):
    """One round of inter-agent feedback refinement."""
    # Step 1: Run forward pass to collect attention weights
    fusion_agent.eval()
    classifier.eval()
    all_weights = []
    all_confidences = []
    with torch.no_grad():
        for batch in zip(*data_loaders.values()):
            latents = [modality_agents[k].encoder(b) for k, b in zip(data_loaders.keys(), batch)]
            fused, weights = fusion_agent(latents)
            logits = classifier(fused)
            confidence = torch.softmax(logits, dim=-1).max(dim=-1).values
            all_weights.append(weights)
            all_confidences.append(confidence)
    mean_weights = torch.cat(all_weights).mean(dim=0)  # (3,)

    # Step 2: Boost underperforming agents
    modality_names = list(modality_agents.keys())
    for i, name in enumerate(modality_names):
        if mean_weights[i].item() < contribution_threshold:
            print(f"[Feedback] {name} agent weight={mean_weights[i]:.3f} < {contribution_threshold}. Boosting.")
            retrain_autoencoder(
                modality_agents[name],
                data_loaders[name],
                epochs=retrain_epochs,
                loss_weight=loss_boost_factor,
            )
    return mean_weights, torch.cat(all_confidences).mean().item()
```

**Example 4: Confidence-aware abstention at inference**

User: "Add abstention so the system refuses to classify when it's not confident."

Output:
```python
def classify_with_abstention(
    sample: dict,
    pipeline,
    abstention_threshold=0.85,
) -> dict:
    """Classify a sample, abstaining when confidence is below threshold."""
    logits = pipeline.forward(sample)
    probs = torch.softmax(logits, dim=-1)
    max_prob, predicted_class = probs.max(dim=-1)

    if max_prob.item() < abstention_threshold:
        return {
            "prediction": "ABSTAIN",
            "confidence": max_prob.item(),
            "reason": "Below confidence threshold",
            "top_3": get_top_k_predictions(probs, k=3),
            "modality_weights": pipeline.last_attention_weights,
        }
    return {
        "prediction": pipeline.family_names[predicted_class.item()],
        "confidence": max_prob.item(),
        "modality_weights": pipeline.last_attention_weights,
    }
```

## Best Practices

- **Do:** Train each modality autoencoder independently first, then freeze encoders during initial classifier training, and only unfreeze during feedback rounds. This prevents early gradient interference.
- **Do:** Use label smoothing (0.05-0.15) on the transformer classifier to improve calibration. Poorly calibrated confidence scores undermine both the feedback loop and the abstention mechanism.
- **Do:** Log per-modality attention weights over epochs. A healthy system shows all modalities contributing (weights > 0.15 each). If one modality consistently collapses to near-zero weight, its input data may be too noisy or its autoencoder architecture is undersized.
- **Do:** Set the abstention threshold using a validation set. Plot accuracy vs. coverage (1 - abstention_rate) and pick the threshold at the knee of the curve.
- **Avoid:** Skipping the feedback loop and using a single-pass fusion. The paper shows +0.75 absolute improvement from iterative refinement — this is the single largest contributor to accuracy.
- **Avoid:** Using the same autoencoder architecture for all modalities. Static features (fixed-length, dense) need different capacity than dynamic features (sequential, sparse). Match encoder depth and width to the modality's intrinsic dimensionality.
- **Avoid:** Setting the abstention threshold too low (<0.5). This defeats the purpose — the system should genuinely refuse uncertain predictions in production. Start at 0.85 and only lower if abstention rate exceeds 20% on known families.

## Error Handling

- **Missing modality at inference:** If a sample lacks one modality (e.g., no PCAP available), zero-fill that modality's latent vector and let the fusion attention naturally down-weight it. The attention mechanism handles partial input gracefully.
- **Autoencoder training divergence:** If reconstruction loss increases during feedback retraining, reduce the loss_boost_factor (e.g., from 1.5 to 1.2) or cap the number of consecutive boosts per agent at 3.
- **Transformer classifier overfitting:** If validation Macro-F1 plateaus while training loss decreases, add dropout (0.1-0.3) to transformer layers and reduce learning rate by 10x.
- **Feedback loop oscillation:** If modality weights oscillate instead of converging, apply exponential moving average (alpha=0.9) to the feedback signals before adjusting loss weights.
- **Empty or corrupt samples:** Validate inputs at ingestion. Skip samples where PE parsing fails, sandbox JSON is malformed, or PCAP has zero flows. Log skipped samples for audit.

## Limitations

- **Zero-day detection is family-dependent.** Polymorphic ransomware that changes static signatures across variants will degrade the static agent's contribution. The system relies on dynamic and network modalities to compensate, but novel evasion across all three modalities will cause abstention (by design).
- **Sandbox evasion.** Sophisticated ransomware that detects sandbox environments and suppresses malicious behavior will produce benign-looking dynamic features. This is a fundamental limitation of any dynamic analysis approach.
- **Requires all three modalities for peak accuracy.** While the attention mechanism gracefully handles missing modalities, accuracy drops measurably (~5-8% Macro-F1) when any single modality is absent.
- **Training data requirements.** Each ransomware family needs at least ~100 samples across all three modalities for the autoencoders to learn useful representations. Rare families with <50 samples will be unreliable and should trigger abstention.
- **Computational cost.** Three autoencoders + fusion + transformer + feedback loop is significantly more expensive than a single-model approach. Budget ~3x the training time of a monolithic classifier.
- **AutoGen dependency.** The orchestration layer assumes AutoGen's group chat and message-passing patterns. Porting to other agent frameworks (LangGraph, CrewAI) requires reimplementing the feedback message protocol.

## Reference

Khan, A., Wadood, A., Iqbal, M., & Zahoora, U. (2026). Multimodal Multi-Agent Ransomware Analysis Using AutoGen. arXiv:2601.20346v1. https://arxiv.org/abs/2601.20346v1

Key sections: Section 3 (framework architecture and agent definitions), Section 4 (inter-agent feedback mechanism and convergence analysis), Section 5 (experimental evaluation on multi-family datasets with ablation studies showing per-modality contributions).