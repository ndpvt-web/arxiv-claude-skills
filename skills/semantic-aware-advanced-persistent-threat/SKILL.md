---
name: "semantic-aware-advanced-persistent-threat"
description: >
  Build anomaly detection pipelines for Advanced Persistent Threat (APT) detection
  by encoding system logs into semantic embeddings with sentence-transformer LLMs
  and training autoencoders on reconstruction error. Trigger phrases: "detect APT
  in system logs", "semantic log anomaly detection", "autoencoder on log embeddings",
  "LLM-based threat detection", "provenance log analysis", "system log anomaly pipeline".
---

# Semantic-Aware APT Detection via LLM-Encoded System Logs

This skill enables Claude to build end-to-end anomaly detection pipelines that catch Advanced Persistent Threats (APTs) in system logs by combining semantic embeddings from a sentence-transformer model (`all-mpnet-base-v2`) with autoencoder-based reconstruction error analysis. The core insight is that converting raw, unstructured system log entries into 768-dimensional semantic vectors captures the *intent* behind system activities -- something statistical features and shallow ML miss entirely -- allowing an autoencoder trained only on normal behavior to flag stealthy, "low-and-slow" attack patterns through elevated reconstruction error.

## When to Use

- When the user asks to detect APTs, intrusions, or anomalies in system/audit logs (syslog, auditd, provenance graphs, CDM records)
- When building an unsupervised anomaly detection pipeline over unstructured log data and needs to go beyond Isolation Forest / OC-SVM / PCA baselines
- When the user has DARPA Transparent Computing (TC) dataset traces (THEIA, TRACE, CADETS, CLEARSCOPE, 5DIR) and wants to run detection experiments
- When the user wants to convert raw provenance records or system call logs into meaningful embeddings for downstream ML
- When the user asks about using LLMs or transformers for cybersecurity log analysis
- When designing a reconstruction-error-based anomaly scoring system for security event streams

## Key Technique

**Semantic embedding of logs**: Raw system logs are first transformed from structured fields (process IDs, file paths, network connections, syscall names) into natural language descriptions. For example, a provenance record becomes: *"Process 1054 started /bin/bash and connected socket 192.168.1.5:80 and changed /etc/passwd."* This natural language sentence is then encoded using `sentence-transformers/all-mpnet-base-v2` (a 768-dim MPNet model fine-tuned on 1B sentence pairs) to produce a dense semantic vector that captures the behavioral meaning of the log entry -- not just its tokens.

**Autoencoder anomaly scoring**: A symmetric feed-forward autoencoder (768 -> 512 -> 128 -> 512 -> 768, ReLU activations, MSE loss, Adam optimizer at lr=0.001) is trained *exclusively* on embeddings from benign/normal activity for 15 epochs with batch size 128. Because the autoencoder learns to reconstruct only normal behavioral patterns, malicious activity -- which has different semantic structure -- produces higher reconstruction error (MSE between input and output embedding). A threshold on this error, tuned on a validation set, classifies each log entry as normal or anomalous.

**Why it works better**: Traditional methods (IForest, OC-SVM, PCA) operate on raw features or shallow statistics and miss the non-linear semantic relationships between system activities. The LLM embedding captures contextual meaning (e.g., that `/bin/bash` connecting to an external IP while modifying `/etc/passwd` is semantically unusual), giving the autoencoder richer signal. On the DARPA TC dataset, this approach achieves AUC > 0.95 in 11 out of 23 experimental conditions across five provenance contexts (ProcessEvent, ProcessExec, ProcessParent, ProcessNetflow, ProcessAll).

## Step-by-Step Workflow

1. **Ingest and parse raw logs**: Read system logs (audit logs, provenance CDM records, syslog entries) and extract structured fields -- process ID, parent process, binary path, syscall type, file paths accessed, network connections (IP:port), timestamps.

2. **Convert log entries to natural language sentences**: For each log entry, construct a descriptive sentence combining the extracted fields. Use templates like: `"Process {pid} ({binary}) [started|read|wrote|connected|changed] {target} [and {additional_actions}]."` Include only fields present in the entry -- omit absent fields rather than using placeholders.

3. **Generate semantic embeddings**: Load `sentence-transformers/all-mpnet-base-v2` and encode each natural language log sentence into a 768-dimensional vector. Batch the encoding for efficiency (batch size 256-512 depending on GPU memory).

   ```python
   from sentence_transformers import SentenceTransformer
   model = SentenceTransformer('all-mpnet-base-v2')
   embeddings = model.encode(log_sentences, batch_size=256, show_progress_bar=True)
   ```

4. **Split data for training**: Separate embeddings into training (benign-only), validation (benign + small known-malicious sample for threshold tuning), and test sets. The autoencoder must train on *only* normal behavior.

5. **Build and train the autoencoder**:
   ```python
   import torch.nn as nn

   class LogAutoencoder(nn.Module):
       def __init__(self):
           super().__init__()
           self.encoder = nn.Sequential(
               nn.Linear(768, 512), nn.ReLU(),
               nn.Linear(512, 128), nn.ReLU()
           )
           self.decoder = nn.Sequential(
               nn.Linear(128, 512), nn.ReLU(),
               nn.Linear(512, 768)
           )
       def forward(self, x):
           return self.decoder(self.encoder(x))
   ```
   Train with Adam (lr=0.001), MSE loss, batch size 128, for 15 epochs. Monitor validation loss for convergence -- expect rapid early convergence then plateau.

6. **Compute reconstruction errors**: Pass all embeddings through the trained autoencoder. Calculate per-sample MSE between input and reconstructed embedding. This error is the anomaly score.

7. **Set the anomaly threshold**: Using the validation set (which contains labeled benign and malicious samples), select the threshold on reconstruction error that maximizes the F1 score or achieves the desired precision-recall tradeoff. Alternatively, use the 95th or 99th percentile of training-set reconstruction errors.

8. **Evaluate with AUC-ROC**: Score the test set and compute AUC-ROC. Compare against baselines (IForest, OC-SVM, PCA on the same embeddings) to validate the autoencoder advantage.

9. **Analyze detected anomalies**: For flagged entries, decode back to the original log sentence and inspect the semantic content. Cluster high-error embeddings with t-SNE to identify attack campaign structure or lateral movement patterns.

10. **Deploy as a streaming detector** (optional): Wrap the embedding + autoencoder inference in a service that processes log streams in micro-batches, emitting alerts when reconstruction error exceeds the threshold.

## Concrete Examples

**Example 1: Building an APT detector on DARPA TC THEIA traces**

User: "I have DARPA TC THEIA provenance logs in JSON format. Help me build an APT detector using LLM embeddings."

Approach:
1. Parse THEIA JSON records, extracting subject (process), object (file/socket), action (syscall), and metadata fields
2. Convert each record to natural language: `"Process 2381 (firefox) connected to socket 10.0.2.15:443 and wrote /tmp/.cache_update"`
3. Encode all sentences with `all-mpnet-base-v2` -> 768-dim vectors
4. Filter known-benign time windows for training set; reserve attack-window data for test
5. Train autoencoder (768->512->128->512->768) on benign embeddings for 15 epochs
6. Score all test embeddings by MSE reconstruction error
7. Tune threshold on validation split, compute AUC-ROC

Output:
```
Anomaly Detection Results (THEIA - ProcessAll context):
  AE (proposed):  AUC-ROC = 0.967
  IForest:        AUC-ROC = 0.823
  OC-SVM:         AUC-ROC = 0.791
  PCA:            AUC-ROC = 0.854

Top anomalous entries (by reconstruction error):
  1. [err=0.142] "Process 2381 (firefox) connected to socket 10.0.2.15:443 and wrote /tmp/.cache_update"
  2. [err=0.138] "Process 4820 (/bin/sh) read /etc/shadow and connected to socket 203.0.113.5:8080"
  3. [err=0.131] "Process 4820 (/bin/sh) started /usr/bin/scp with args ['-r', '/var/log', '203.0.113.5:/exfil']"
```

**Example 2: Adding semantic anomaly detection to an existing SIEM pipeline**

User: "We have auditd logs streaming into Kafka. I want to add an LLM-based anomaly scoring stage."

Approach:
1. Write a Kafka consumer that reads auditd log batches
2. Parse each auditd record into structured fields (uid, pid, exe, syscall, path, addr)
3. Template into sentences: `"User root (uid=0) process 891 (/usr/sbin/sshd) executed EXECVE /bin/bash and accessed /etc/passwd"`
4. Batch-encode with `all-mpnet-base-v2` (pre-load model at startup)
5. Run through pre-trained autoencoder, compute MSE per entry
6. Publish anomaly scores back to a Kafka topic; alert on scores above threshold

Output (streaming service pseudocode):
```python
from sentence_transformers import SentenceTransformer
import torch

model = SentenceTransformer('all-mpnet-base-v2')
autoencoder = LogAutoencoder()
autoencoder.load_state_dict(torch.load('apt_detector.pt'))
autoencoder.eval()
THRESHOLD = 0.089  # tuned on validation set

for batch in kafka_consumer:
    sentences = [to_natural_language(record) for record in batch]
    embeddings = torch.tensor(model.encode(sentences))
    with torch.no_grad():
        reconstructed = autoencoder(embeddings)
    errors = ((embeddings - reconstructed) ** 2).mean(dim=1)
    for record, err in zip(batch, errors):
        if err > THRESHOLD:
            publish_alert(record, anomaly_score=err.item())
```

**Example 3: Comparing provenance contexts for detection accuracy**

User: "Which log context gives the best APT detection -- process events, network flows, or the combined view?"

Approach:
1. From the same dataset, extract five provenance views: ProcessEvent (syscalls only), ProcessExec (binary executions), ProcessParent (parent-child relationships), ProcessNetflow (network connections), ProcessAll (combined)
2. Generate separate embedding sets for each view using context-specific sentence templates
3. Train a separate autoencoder per view on benign data
4. Evaluate AUC-ROC for each

Output:
```
Context Comparison (CADETS dataset):
  ProcessAll:     AUC = 0.971  <- best (richest semantic context)
  ProcessNetflow: AUC = 0.943
  ProcessExec:    AUC = 0.921
  ProcessEvent:   AUC = 0.908
  ProcessParent:  AUC = 0.887

Recommendation: Use ProcessAll (combined) context for production.
Network-only context is a strong second choice with lower compute cost.
```

## Best Practices

- **Do:** Train the autoencoder exclusively on verified benign/normal data. Any malicious contamination in training will teach the model to reconstruct attacks as "normal," destroying detection capability.
- **Do:** Use context-rich natural language templates that include process names, file paths, network addresses, and action verbs. Richer sentences produce more discriminative embeddings.
- **Do:** Evaluate across multiple provenance contexts (process, network, combined) since different APT campaigns manifest differently -- some are network-heavy, others are filesystem-heavy.
- **Do:** Normalize embeddings (L2 normalization) before feeding into the autoencoder if you observe training instability or widely varying reconstruction errors.
- **Avoid:** Using generic or overly terse log templates like `"event: write, pid: 1054"`. The semantic model needs natural language to activate its learned representations.
- **Avoid:** Setting the anomaly threshold on the training set alone. Always use a held-out validation set with at least some labeled malicious samples to calibrate the decision boundary.
- **Avoid:** Treating this as a supervised classifier. The autoencoder is unsupervised by design -- it learns normality, not attack signatures. Mixing labeled attack data into training defeats the purpose.

## Error Handling

- **Embedding model fails to load**: Ensure `sentence-transformers` is installed (`pip install sentence-transformers`) and the model name is exactly `all-mpnet-base-v2`. For air-gapped environments, download the model files ahead of time and load from a local path.
- **GPU out of memory during encoding**: Reduce the encoding batch size (try 64 or 32). The `all-mpnet-base-v2` model is ~420MB and needs ~2GB VRAM for inference at batch size 256.
- **Autoencoder loss doesn't converge**: Check that input embeddings are not all zeros (indicates failed encoding). Verify learning rate (0.001 is the starting point; try 0.0001 if loss oscillates). Ensure training data contains only benign samples.
- **Too many false positives**: The threshold is too low. Re-tune on the validation set targeting higher precision. Consider per-context thresholds if using multiple provenance views.
- **Too many false negatives (misses APT)**: The log-to-sentence template may be too sparse. Include more fields (parent process, command-line arguments, file permissions). Also try the ProcessAll combined context.
- **Reconstruction error distribution is bimodal on training data**: Likely indicates training data contamination with anomalous entries. Audit and clean the training set.

## Limitations

- **Requires clean benign training data**: If the training logs already contain undetected APT activity, the autoencoder will learn to reconstruct those attacks as normal. A baseline period of verified-clean activity is essential.
- **Latency**: Encoding each log entry through a 110M-parameter transformer adds inference latency (~5-15ms per entry on GPU). For high-throughput environments (>10K events/sec), batching and GPU acceleration are mandatory.
- **Embedding model domain gap**: `all-mpnet-base-v2` is trained on general English text, not security logs specifically. While the natural language templating bridges this gap, fine-tuning on security corpora could improve results.
- **No attack attribution**: The method detects anomalies but does not classify attack type, stage, or campaign. Detected anomalies require analyst review or downstream correlation.
- **Threshold sensitivity**: The binary normal/anomalous decision depends on a single threshold. In practice, use tiered alerting (high/medium/low confidence) based on reconstruction error magnitude.
- **Single-entry analysis**: Each log entry is scored independently. The method does not model temporal sequences or causal chains across entries -- it may miss multi-step attacks where each individual step looks benign.

## Reference

**Paper**: [Semantic-Aware Advanced Persistent Threat Detection Using Autoencoders on LLM-Encoded System Logs](https://arxiv.org/abs/2602.00204v1) (Khan Mohammed et al., 2026). Key sections: Section III for the log-to-embedding pipeline and autoencoder architecture, Section IV for DARPA TC dataset setup, Section V for AUC-ROC results across five provenance contexts and four detection methods.