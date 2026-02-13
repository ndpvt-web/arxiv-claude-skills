---
name: "evaluating-kubernetes-performance-genai"
description: "Design and optimize Kubernetes-native GenAI inference platforms using Kueue job queuing, Dynamic Accelerator Slicer (DAS) GPU partitioning, and Gateway API Inference Extension (GAIE) with llm-d for multi-stage AI pipelines. Use when: 'set up Kubernetes for AI inference', 'configure Kueue for batch GPU jobs', 'partition GPUs with MIG slicing on Kubernetes', 'optimize LLM inference routing on Kubernetes', 'build a Whisper-to-LLM pipeline on K8s', 'reduce TTFT latency for LLM serving'."
---

# Kubernetes-Native GenAI Inference Platform Design

This skill enables Claude to architect, configure, and optimize multi-stage GenAI inference pipelines on Kubernetes using three complementary components: **Kueue** for batch job scheduling with priority and preemption, **Dynamic Accelerator Slicer (DAS)** for just-in-time GPU partitioning via NVIDIA MIG, and the **Gateway API Inference Extension (GAIE)** with **llm-d** for prefix-cache-aware LLM request routing. The technique is drawn from an industry paper demonstrating that these components form a cohesive platform delivering up to 15% makespan reduction (Kueue), 36% faster mean job completion (DAS), and 90% improvement in tail TTFT latency (GAIE) on real ASR-to-summarization workloads.

## When to Use

- When the user asks to deploy batch AI inference jobs (e.g., Whisper transcription, image generation) on Kubernetes and needs efficient GPU scheduling
- When the user wants to run more parallel GPU jobs on limited hardware by slicing GPUs with MIG profiles
- When the user is deploying an LLM serving layer (vLLM, llm-d) and needs to reduce Time to First Token under load
- When the user needs to build a multi-stage pipeline where batch output (e.g., transcripts) feeds into online inference (e.g., summarization)
- When the user asks how to configure Kueue ClusterQueues, ResourceFlavors, or priority-based preemption for GPU workloads
- When the user wants to understand or implement prefix-cache-aware routing for LLM inference on Kubernetes

## Key Technique

The paper's core insight is that three Kubernetes-native projects address three distinct bottlenecks in GenAI inference and, when combined, eliminate compounding inefficiencies:

**Kueue** replaces Kubernetes' default scheduler for batch workloads by introducing a hierarchical queue system (ResourceFlavors -> ClusterQueues -> LocalQueues). It admits jobs based on available cluster capacity rather than letting pods pile up in Pending state. With BestEffortFIFO ordering, priority classes, and preemption policies, Kueue ensures high-value jobs (e.g., large Whisper models) preempt lower-priority work, reducing total makespan by up to 15%. Critically, Kueue's admission latency stays under 25ms at P99 even with complex multi-queue configurations.

**DAS (Dynamic Accelerator Slicer)** solves GPU underutilization by creating NVIDIA MIG partitions just-in-time based on pod resource requests. Instead of allocating a full A100 (40GB) to a job that only needs 5GB, DAS provisions a `1g.5gb` slice, enabling 25 concurrent jobs where only 8 could run before. Individual jobs run ~20% slower on slices, but the 3x parallelism gain produces a net 36% reduction in mean completion time. DAS coordinates with the NVIDIA GPU Operator and binds scheduling to physical slice allocation.

**GAIE + llm-d** tackles online inference latency through inference-aware request routing. The Endpoint Picker Plugin (EPP) in GAIE inspects each request's prompt tokens and routes it to the vLLM replica whose KV-cache already contains the longest matching prefix. This "prefix-cache-aware" scheduling achieves a 6.25x improvement in TTFT (500ms -> 80ms) compared to random routing, with P99 tail latency improving by up to 90%. The overhead of the routing layer itself is only 3-11ms.

## Step-by-Step Workflow

1. **Assess the workload type**: Classify each stage of the pipeline as batch inference (finite jobs processing a dataset) or online inference (request-response serving). Batch stages use Kueue + DAS; online stages use GAIE + llm-d.

2. **Define ResourceFlavors for your GPU inventory**: Create ResourceFlavor CRDs that describe each distinct GPU type or MIG profile available in the cluster (e.g., `nvidia-a100-40gb`, `nvidia-a100-1g5gb`). Map these to node labels and taints.

3. **Configure Kueue's queue hierarchy**: Create a ClusterQueue with resource quotas matching your GPU capacity. For multi-tenant or multi-priority workloads, create multiple ClusterQueues with borrowing limits. Attach LocalQueues in each namespace where jobs will be submitted.

4. **Assign priority classes to job types**: Create PriorityClass resources for each workload tier (e.g., `large-model-high`, `medium-model-default`). Configure preemption policies on the ClusterQueue so higher-priority jobs can reclaim resources from lower-priority ones.

5. **Configure DAS profiles for GPU slicing**: Define DASProfile CRDs specifying which MIG profiles to use for each workload size. For example: medium Whisper (769M params) -> `1g.5gb`; large Whisper (1.55B params) -> `3g.20gb`. Ensure the NVIDIA GPU Operator is installed with MIG support enabled.

6. **Create batch job templates**: Write Kubernetes Job manifests that request specific GPU slice resources (e.g., `nvidia.com/mig-1g.5gb: 1`). Set the Kueue queue label (`kueue.x-k8s.io/queue-name`) so jobs are admitted through the queue system rather than directly scheduled.

7. **Deploy the LLM serving layer with llm-d**: Deploy vLLM replicas serving your target model (e.g., Qwen3-8B) with prefix caching enabled. Create InferencePool and InferenceModel CRDs to register the model endpoint with GAIE.

8. **Configure GAIE routing with the Endpoint Picker Plugin**: Set up the InferenceGateway with an Envoy-based service mesh. Configure the EPP to use prefix-cache-aware scheduling ("precise" mode) rather than random or round-robin routing.

9. **Wire the pipeline stages together**: Implement a controller or workflow engine (e.g., Argo Workflows, Tekton) that submits batch transcription jobs, collects outputs, then sends them as requests to the LLM serving endpoint for summarization.

10. **Benchmark and tune**: Use kube-burner for batch workload orchestration and GuideLLM for online inference benchmarking. Monitor Kueue admission latency, DAS slice utilization, and GAIE TTFT/E2E latency. Adjust queue borrowing limits, MIG profiles, and vLLM replica count based on results.

## Concrete Examples

**Example 1: Configuring Kueue for GPU batch transcription**

User: "I have 8 A100 GPUs and need to run 32 Whisper transcription jobs. Some are large model, some medium. How do I set up Kueue to prioritize the large jobs?"

Approach:
1. Create two PriorityClasses: `whisper-large` (value: 1000) and `whisper-medium` (value: 500)
2. Define a ResourceFlavor for `nvidia.com/gpu` on the GPU node
3. Create a ClusterQueue with quota of 8 GPUs, preemption policy `ReclaimWithinCohort`
4. Create a LocalQueue in the workload namespace
5. Label each Job with the appropriate priority class and queue name

Output:
```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: gpu-queue
spec:
  queueingStrategy: BestEffortFIFO
  preemption:
    reclaimWithinCohort: Any
    withinClusterQueue: LowerPriority
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: a100-40gb
      resources:
      - name: nvidia.com/gpu
        nominalQuota: 8
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  name: transcription-queue
  namespace: asr-pipeline
spec:
  clusterQueue: gpu-queue
---
apiVersion: batch/v1
kind: Job
metadata:
  name: whisper-large-001
  namespace: asr-pipeline
  labels:
    kueue.x-k8s.io/queue-name: transcription-queue
spec:
  template:
    spec:
      priorityClassName: whisper-large
      containers:
      - name: whisper
        image: openai/whisper:large-v3
        resources:
          limits:
            nvidia.com/gpu: 1
      restartPolicy: Never
```

Expected behavior: Kueue admits jobs up to the 8-GPU quota. When a `whisper-large` job arrives and all GPUs are occupied by `whisper-medium` jobs, Kueue preempts a medium job to free a GPU. Admission latency stays under 25ms.

**Example 2: Enabling DAS GPU slicing for higher parallelism**

User: "My Whisper medium model only needs 5GB of GPU memory but each job gets a full 40GB A100. Can I run more jobs in parallel?"

Approach:
1. Enable MIG mode on the A100 GPUs via the NVIDIA GPU Operator
2. Configure DAS with a profile mapping: medium Whisper -> `1g.5gb` MIG slice
3. Update job manifests to request `nvidia.com/mig-1g.5gb` instead of `nvidia.com/gpu`
4. DAS dynamically creates/destroys MIG partitions as pods are scheduled

Output:
```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: DASProfile
metadata:
  name: whisper-medium-profile
spec:
  migProfile: 1g.5gb
  accelerator: nvidia-a100-40gb
---
apiVersion: batch/v1
kind: Job
metadata:
  name: whisper-medium-017
  labels:
    kueue.x-k8s.io/queue-name: transcription-queue
spec:
  template:
    spec:
      containers:
      - name: whisper
        image: openai/whisper:medium
        resources:
          limits:
            nvidia.com/mig-1g.5gb: 1
      restartPolicy: Never
```

Expected behavior: Each A100 is split into up to 7 MIG slices. Cluster capacity rises from 8 concurrent jobs (full GPUs) to ~25 (sliced). Individual jobs take ~17 min instead of ~14 min, but mean completion across all 32 jobs drops from 28 min to 18 min (36% reduction).

**Example 3: Prefix-cache-aware LLM routing with GAIE and llm-d**

User: "I have 8 vLLM replicas serving Qwen3-8B for summarization. Under load, TTFT spikes to 500ms+. How do I reduce it?"

Approach:
1. Deploy llm-d as the inference serving layer with prefix caching enabled on each vLLM instance
2. Create InferencePool and InferenceModel CRDs for the model
3. Configure the GAIE Endpoint Picker Plugin with prefix-cache-aware ("precise") scheduling
4. Route all summarization requests through the GAIE gateway

Output:
```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferencePool
metadata:
  name: summarization-pool
spec:
  targetPortNumber: 8000
  selector:
    matchLabels:
      app: vllm-qwen3-8b
  extensionRef:
    name: endpoint-picker
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceModel
metadata:
  name: qwen3-8b-summarizer
spec:
  modelName: Qwen/Qwen3-8B
  poolRef:
    name: summarization-pool
  criticality: Critical
```

Expected behavior: The EPP inspects each incoming request's prompt tokens and routes to the replica with the longest matching prefix in its KV-cache. TTFT drops from ~500ms (random routing) to ~80ms (prefix-aware). P99 tail latency improves by up to 90%. The routing layer adds only 3-11ms overhead, which is far outweighed by cache hit gains.

## Best Practices

- **Do** start with Kueue even before adding DAS -- the admission control alone prevents GPU thrashing from too many pending pods and gives you deterministic scheduling with <25ms overhead.
- **Do** right-size MIG profiles to your model's actual VRAM needs. A Whisper medium model (769M params) fits in `1g.5gb`; don't allocate `3g.20gb` unless the model requires it.
- **Do** enable prefix caching on vLLM replicas (`--enable-prefix-caching`) before configuring GAIE's prefix-aware routing -- the routing is useless without cached prefixes.
- **Do** benchmark with realistic prompt distributions. Prefix-cache-aware routing benefits degrade if prompts share no common prefixes.
- **Avoid** using DAS MIG slicing for models that saturate GPU compute -- slicing helps memory-bound workloads but hurts compute-bound ones.
- **Avoid** setting Kueue borrowing limits too aggressively in multi-queue setups. Overly generous borrowing defeats the purpose of resource isolation between teams.

## Error Handling

- **DAS slice creation failure**: If MIG partitioning fails (e.g., incompatible GPU driver version), pods remain Pending. Ensure the NVIDIA GPU Operator version supports MIG mode for your GPU generation. A100 and H100 support MIG; older GPUs (V100, T4) do not.
- **Kueue admission deadlock**: If all ClusterQueues are at quota and preemption is disabled, new jobs stall indefinitely. Always configure at least `withinClusterQueue: LowerPriority` preemption for production queues.
- **GAIE routing overhead under very low load**: At low request rates, prefix-cache-aware routing may show slightly higher latency than random due to the EPP lookup. This overhead (3-11ms) only matters if your latency budget is extremely tight and request volume is low.
- **KV-cache cold start**: After replica restarts or scale-up events, new vLLM instances have empty caches. The EPP will naturally route fewer requests to cold replicas, but initial requests will see higher TTFT until caches warm.
- **MIG profile mismatch**: Requesting a MIG profile that doesn't match the GPU's supported configurations causes scheduling failures. Verify supported profiles with `nvidia-smi mig -lgip` on the target node.

## Limitations

- **GPU hardware requirement**: DAS/MIG slicing requires NVIDIA Ampere (A100) or newer GPUs. Older GPUs like V100 or T4 do not support MIG and cannot use DAS.
- **Kueue-DAS integration is nascent**: As of the paper, Kueue does not natively understand MIG slice capacity. Jobs must request specific MIG device types, and Kueue treats them as opaque resources. GPU-memory-aware scheduling integration is ongoing.
- **Single-node scope in experiments**: The paper's testbed used a single `p4d.24xlarge` (8x A100). Multi-node GPU scheduling with DAS and Kueue introduces additional complexity around topology-aware placement.
- **Prefix-cache routing assumes shared prompt structure**: The GAIE prefix-aware scheduler provides the greatest benefit when requests share long common prefixes (e.g., system prompts, repeated document context). For fully unique prompts, it degrades to near-random performance.
- **GAIE maturity**: The Gateway API Inference Extension and llm-d are early-stage projects. CRD APIs may change between releases. Pin versions in production.

## Reference

[Evaluating Kubernetes Performance for GenAI Inference: From Automatic Speech Recognition to LLM Summarization](https://arxiv.org/abs/2602.04900v3) -- Malleni et al., 2026. Focus on Sections 3-5 for Kueue queue hierarchy design, DAS MIG profile configuration, and GAIE prefix-cache-aware routing benchmarks. The paper's GitHub repository contains reproducible manifests for all experiments.