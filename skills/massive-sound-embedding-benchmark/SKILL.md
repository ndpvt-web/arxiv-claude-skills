---
name: "massive-sound-embedding-benchmark"
description: |
  Build, evaluate, and integrate audio embedding pipelines using the MSEB framework from Google Research.
  Covers all 8 MSEB super-tasks: transcription, classification, retrieval, reasoning, segmentation,
  clustering, reranking, and reconstruction. Guides Claude through encoder design, evaluation harness
  setup, metric selection, and result interpretation for sound embeddings.
  Trigger phrases:
  - "Evaluate my audio embedding model"
  - "Benchmark sound embeddings with MSEB"
  - "Set up audio classification / retrieval / transcription evaluation"
  - "Compare audio encoders across tasks"
  - "Build an MSEB-compatible encoder pipeline"
  - "Assess multimodal audio system quality"
---

# Massive Sound Embedding Benchmark (MSEB)

This skill enables Claude to help users build, evaluate, and improve audio embedding systems using the MSEB framework. MSEB treats every audio task as a two-stage pipeline: first encode raw audio into an embedding (a fixed vector, a token sequence, or a structured representation), then evaluate that embedding on 8 standardized super-tasks spanning transcription, classification, retrieval, reasoning, segmentation, clustering, reranking, and reconstruction. The framework is model-agnostic, supports bulk inference via beam pipelines, and uses a shared embedding cache so one encoding pass serves multiple downstream evaluations.

## When to Use

- When the user wants to evaluate an audio or speech embedding model against standardized benchmarks
- When building a multimodal system that consumes audio and needs to verify embedding quality across diverse tasks
- When comparing multiple audio encoders (end-to-end, cascade ASR-to-text, multi-tower) on a common footing
- When designing a new audio embedding architecture and needing to select appropriate evaluation metrics
- When the user asks to set up evaluation for speech transcription, speaker clustering, audio retrieval, sound event classification, or audio reconstruction
- When integrating the `google-research/mseb` library into a CI/evaluation pipeline
- When the user needs to contribute a new dataset or task to the MSEB framework

## Key Technique

MSEB's core insight is **decoupling encoding from evaluation**. Rather than coupling each audio task to a bespoke model, MSEB defines a universal `MultiModalEncoder` interface that accepts audio (and optionally text/labels) and outputs embeddings. These embeddings are written once to a key-value **Embedding Cache**, then consumed by independent, model-agnostic **Evaluators** for each of the 8 super-tasks. This means a single bulk-inference pass over a dataset produces all the embeddings needed for transcription WER, retrieval MRR, classification mAP, clustering V-measure, and every other metric -- drastically reducing compute cost when benchmarking.

The framework supports three encoder architectures: **End-to-End** (raw audio directly to embedding), **Cascade** (chained stages, e.g., ASR followed by text embedding), and **Collections** (multi-tower encoders handling multiple modalities). Each encoder also reports universal efficiency metrics -- Compression Ratio (memory footprint vs. original audio) and FLOPS -- so quality and cost can be compared jointly.

A key finding from the initial experiments is that significant performance headroom exists between sound-based encoders and text-oracle baselines across all tasks, meaning there is substantial room for improvement. Specialized encoders (e.g., Perch for bioacoustics, CLAP for environmental sounds) dramatically outperform general-purpose models on domain-specific tasks, while simple spectrogram baselines remain surprisingly competitive for speaker clustering in clean conditions. This tells practitioners to profile their task distribution before committing to an encoder architecture.

## Step-by-Step Workflow

1. **Install the MSEB library** from `google-research/mseb` on GitHub. Verify dependencies (Python 3.10+, Apache Beam for bulk inference, relevant audio I/O libraries like `soundfile` or `librosa`).

2. **Identify the target super-tasks** relevant to the user's application. Map them to MSEB's 8 categories:
   - Retrieval (MRR, EM) -- spoken query to document/passage/span
   - Reranking (mAP, WER) -- reorder candidate hypotheses by acoustic relevance
   - Reasoning (gmean-F1) -- extract answer spans from context or abstain
   - Classification (accuracy / mAP) -- categorize audio into predefined classes
   - Transcription (WER, CER) -- speech-to-text across 26 locales, 4 noise conditions
   - Segmentation (NDCG) -- localize salient moments with timestamps
   - Clustering (V-measure) -- group audio segments without labels
   - Reconstruction (FAD, KAD) -- regenerate audio from embeddings

3. **Select or configure datasets.** MSEB ships with several:
   - **SVQ** (Simple Voice Questions): 171K recordings, 700 speakers, 26 locales, 17 languages, 4 noise environments
   - **Speech-MASSIVE**: 12 languages, 18 domains, 60 intents, 55 slots
   - **FSD50K**: 51K clips, 200 sound event classes
   - **BirdSet**: 6,800+ hours bioacoustics, ~10K bird species
   Add custom datasets by implementing the MSEB task interface (mapping dataset rows to encoder inputs).

4. **Implement the `MultiModalEncoder` interface** for the model under test. Choose the architecture pattern:
   - End-to-End: `encode(audio) -> embedding`
   - Cascade: `encode(audio) -> text -> text_embed(text) -> embedding`
   - Collection: multiple sub-encoders, one per modality tower
   Define the output shape: fixed-size vector, variable-length sequence of continuous frames, or discrete token IDs.

5. **Run bulk inference** using the MSEB runner (Apache Beam pipeline or local executor). This processes all selected datasets through the encoder and populates the Embedding Cache (a key-value store mapping sample IDs to embedding tensors).

6. **Execute evaluators** for each selected super-task. Evaluators are model-agnostic: they read from the Embedding Cache, map embeddings to task-specific output spaces (e.g., nearest-neighbor retrieval, linear probe classification, greedy decode for transcription), and compute the primary metric.

7. **Record efficiency metrics** alongside quality scores: Compression Ratio (embedding bytes / raw audio bytes) and computational FLOPS. These are mandatory for fair comparison.

8. **Analyze results against baselines.** Compare to the text-oracle upper bound (which bypasses audio encoding entirely) to gauge the quality gap. Check cross-lingual vs. in-language performance. Note whether a specialized encoder would outperform the general model on specific task slices.

9. **Iterate on the encoder.** Common improvement paths:
   - Switch from cascade to end-to-end if ASR errors dominate downstream metrics
   - Add domain-specific pretraining data if specialized benchmarks (bioacoustics, environmental sounds) underperform
   - Increase embedding dimensionality if reconstruction FAD is poor but classification is good (information bottleneck)
   - Reduce dimensionality or quantize if compression ratio is too large for deployment

10. **Submit results to the MSEB leaderboard** via a pull request to the GitHub repository, following the contribution template.

## Concrete Examples

**Example 1: Evaluate a custom speech encoder on transcription and retrieval**

User: "I have a speech encoder model that outputs 768-dim vectors. I want to benchmark it on MSEB transcription and retrieval tasks."

Approach:
1. Install MSEB: `pip install git+https://github.com/google-research/mseb.git`
2. Wrap the user's model in a `MultiModalEncoder` subclass:
```python
from mseb.encoders import MultiModalEncoder

class MyEncoder(MultiModalEncoder):
    def __init__(self, model_path):
        self.model = load_model(model_path)

    def encode_audio(self, audio_samples, sample_rate=16000):
        # Returns dict with 'embedding': np.array of shape (768,)
        features = self.model(audio_samples, sample_rate)
        return {"embedding": features}

    @property
    def embedding_dim(self):
        return 768

    @property
    def modalities(self):
        return ["audio"]
```
3. Configure evaluation for transcription (SVQ dataset, WER metric) and retrieval (SVQ retrieval split, MRR metric):
```python
from mseb.tasks import TranscriptionTask, RetrievalTask
from mseb.runner import BulkRunner
from mseb.evaluation import evaluate_all

encoder = MyEncoder("path/to/model")
tasks = [
    TranscriptionTask(dataset="svq", locales=["en-US", "de-DE"]),
    RetrievalTask(dataset="svq", granularity="passage"),
]
runner = BulkRunner(encoder=encoder, tasks=tasks, cache_dir="./embeddings")
runner.run()
results = evaluate_all(tasks=tasks, cache_dir="./embeddings")
```
4. Output:
```
Transcription (SVQ en-US clean): WER=12.3%, CER=4.1%
Transcription (SVQ de-DE clean): WER=18.7%, CER=6.9%
Retrieval (SVQ passage):         MRR=0.71, EM=0.58
Compression Ratio:               0.023
FLOPS:                           1.2 GFLOPS/s
```

**Example 2: Compare cascade vs. end-to-end encoders for classification**

User: "Should I use an ASR cascade or end-to-end encoder for environmental sound classification? How do I test both?"

Approach:
1. Define two encoders -- one cascade (Whisper ASR + text embedder), one end-to-end (CLAP):
```python
from mseb.encoders import CascadeEncoder, EndToEndEncoder

cascade = CascadeEncoder(
    asr_model="whisper-large-v3",
    text_encoder="sentence-transformers/all-MiniLM-L6-v2",
)
e2e = EndToEndEncoder(model_name="laion/clap-htsat-unfused")
```
2. Run both on FSD50K classification (multi-label, primary metric: mAP):
```python
from mseb.tasks import ClassificationTask

task = ClassificationTask(dataset="fsd50k", label_type="multi_label")
for name, enc in [("cascade", cascade), ("e2e", e2e)]:
    runner = BulkRunner(encoder=enc, tasks=[task], cache_dir=f"./cache_{name}")
    runner.run()
    results = evaluate_all(tasks=[task], cache_dir=f"./cache_{name}")
    print(f"{name}: mAP={results['fsd50k']['mAP']:.3f}")
```
3. Output:
```
cascade: mAP=0.312   (ASR errors lose non-speech acoustic information)
e2e:     mAP=0.487   (CLAP retains spectral features relevant to sound events)
```
4. Recommendation: For environmental sound classification, end-to-end encoders that preserve spectral information strongly outperform cascade approaches that discard non-linguistic audio content through the ASR bottleneck.

**Example 3: Add a custom bioacoustics dataset to MSEB**

User: "I have a whale song dataset. How do I add it to MSEB and run clustering evaluation?"

Approach:
1. Implement the MSEB task interface for the custom dataset:
```python
from mseb.tasks import ClusteringTask

class WhaleClusteringTask(ClusteringTask):
    def __init__(self, data_dir):
        super().__init__(
            dataset_name="whale_songs",
            audio_dir=data_dir,
            # No predefined labels -- clustering discovers structure
            metric="v_measure",
        )

    def load_samples(self):
        # Yield (sample_id, audio_path, metadata) tuples
        for wav in Path(self.audio_dir).glob("*.wav"):
            yield wav.stem, str(wav), {"species": wav.parent.name}
```
2. Run clustering evaluation with a domain-specific encoder (e.g., Perch):
```python
task = WhaleClusteringTask(data_dir="./whale_data")
encoder = EndToEndEncoder(model_name="google/perch")
runner = BulkRunner(encoder=encoder, tasks=[task], cache_dir="./whale_cache")
runner.run()
results = evaluate_all(tasks=[task], cache_dir="./whale_cache")
```
3. Output:
```
Whale Clustering: V-measure=0.62, Homogeneity=0.71, Completeness=0.55
```

## Best Practices

- **Do:** Run all 8 super-tasks when evaluating a general-purpose audio encoder. A model that excels at transcription may fail at clustering or reconstruction, revealing blind spots.
- **Do:** Always report Compression Ratio and FLOPS alongside quality metrics. A model with 1% better WER but 10x the compute cost may not be the right choice.
- **Do:** Test across noise conditions (clean, background speech, traffic, media) using SVQ's 4 environments. Real-world audio is rarely clean.
- **Do:** Use the Embedding Cache to avoid redundant inference. One encoding pass should serve all downstream evaluations.
- **Avoid:** Evaluating only on English. SVQ covers 26 locales; cross-lingual performance gaps are real and significant.
- **Avoid:** Assuming a cascade (ASR + text embed) approach is always best. For non-speech audio tasks (environmental sounds, bioacoustics, music), cascade pipelines discard critical acoustic information through the ASR bottleneck.
- **Avoid:** Fine-tuning on MSEB evaluation datasets. The benchmark is designed for zero-shot evaluation of pretrained embeddings. Task-specific fine-tuning undermines comparability.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| OOM during bulk inference | Large dataset + high-dim embeddings | Use Apache Beam distributed runner or process in sharded batches; reduce batch size |
| Low retrieval MRR despite good WER | ASR cascade loses prosodic/acoustic cues relevant to semantic matching | Switch to end-to-end encoder or fuse ASR text embeddings with acoustic features |
| V-measure near zero for clustering | Embedding space lacks speaker/class discriminability | Check embedding dimensionality; try larger model or domain-specific pretraining |
| FAD/KAD anomalously high for reconstruction | Embedding is too compressed (high CR) | Increase embedding sequence length or dimensionality; use continuous rather than discrete tokens |
| Inconsistent WER across locales | Training data imbalance across languages | Report per-locale scores; augment underrepresented languages in pretraining |
| Embedding Cache corruption | Interrupted bulk inference run | Delete cache directory and re-run; enable checkpointing in the runner config |

## Limitations

- MSEB v1 covers 8 tasks but does not yet include source separation, speaker diarization with overlap, or music-specific tasks like beat tracking or chord recognition. These are planned for future releases.
- The SVQ dataset, while large (171K recordings), is crowdsourced and may not represent all acoustic environments encountered in production (e.g., industrial noise, underwater acoustics).
- Evaluation is zero-shot by design. If the user's use case benefits from task-specific fine-tuning, MSEB scores may understate achievable performance.
- The framework currently supports Sound and Text modalities. Video-audio multimodal evaluation is not yet integrated.
- Leaderboard submissions require a public GitHub pull request, which may not suit proprietary models. Users can still run evaluation locally without submitting.

## Reference

- **Paper:** [Massive Sound Embedding Benchmark (MSEB)](https://arxiv.org/abs/2602.07143v1) -- Heigold et al., 2026. Focus on Section 3 (Task Definitions and Metrics), Section 4 (SVQ Dataset), and Section 5 (Experimental Results) for implementation guidance.
- **Code:** [github.com/google-research/mseb](https://github.com/google-research/mseb) -- the official library with encoder interfaces, task definitions, runners, and evaluators.