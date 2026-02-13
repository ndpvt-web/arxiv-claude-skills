---
name: "innovator-vl-multimodal-scientific-discovery"
description: "Build data-efficient multimodal scientific ML pipelines using Innovator-VL's principled training methodology. Applies transparent, reproducible training design—curated data selection, staged SFT, discrepancy-driven RL—to create scientific vision-language models without massive pretraining. Use when: 'build a scientific VLM pipeline', 'curate efficient training data for multimodal model', 'set up reinforcement learning for scientific reasoning', 'design a training pipeline for chemistry/biology image understanding', 'create a data-efficient fine-tuning recipe', 'implement GSPO reward-based RL training'."
---

# Innovator-VL: Data-Efficient Multimodal Scientific Discovery Pipelines

This skill enables Claude to design and implement training pipelines for scientific multimodal large language models following the Innovator-VL methodology. The core insight: principled data curation and staged training (alignment, mid-training, SFT, RL) with fewer than 5 million curated scientific samples can match or exceed models trained on orders of magnitude more data. Claude applies this to help users build reproducible, transparent pipelines for scientific image understanding—molecular structures, chemical reactions, micrographs, charts, and STEM reasoning—without requiring massive compute budgets or opaque data pipelines.

## When to Use

- When the user wants to build or fine-tune a vision-language model for scientific domains (chemistry, biology, materials science, physics)
- When designing a data curation pipeline that prioritizes quality over quantity for multimodal training
- When setting up a multi-stage training recipe: projector alignment, full-parameter mid-training, supervised fine-tuning, then reinforcement learning
- When implementing reinforcement learning with structured rewards for scientific reasoning tasks (e.g., GSPO with format + accuracy rewards)
- When the user needs to balance scientific specialization with general vision capabilities in a single model
- When building human-in-the-loop data annotation pipelines for domain-specific scientific imagery (OCSR, electron micrographs, reaction diagrams)
- When selecting medium-difficulty RL training samples using discrepancy-driven filtering (Pass@N vs Pass@1 gap)
- When implementing token-efficient reasoning with structured think/answer output formats

## Key Technique

**Staged Training with Principled Data Selection.** Innovator-VL uses a four-stage pipeline built on a RICE-ViT vision encoder (region-aware visual transformer), a PatchMerger projector for token compression, and a Qwen3-8B language backbone. Stage 1 aligns the projector to LLM embeddings using ~558K image-text pairs (projector-only training). Stage 2 performs full-parameter mid-training on ~85M samples using concept-balanced sampling with MetaCLIP encoders to ensure semantic diversity. Stage 3 runs supervised fine-tuning on ~46M samples split across general multimodal instruction (~48%), chain-of-thought reasoning (~33%), and scientific understanding (~19%). Stage 4 applies reinforcement learning on just ~172K carefully selected samples.

**Discrepancy-Driven RL Sample Selection.** The RL stage uses Group Sequence Policy Optimization (GSPO), which fixes token-level importance sampling issues in prior methods by operating at the sequence level. The critical innovation for data efficiency is *discrepancy-driven selection*: measuring the gap between Pass@N (best-of-N accuracy) and Pass@1 (greedy accuracy) to identify medium-difficulty problems. Samples where the model sometimes succeeds but often fails provide the highest learning signal. Trivially easy and impossibly hard samples are filtered out. The reward function combines format compliance (structured `<think>`/`<answer>` tags, weighted at 0.1) with hierarchical accuracy verification (rule-based, symbolic math verification, then LLM-as-judge fallback, weighted at 0.9).

**Scientific-General Balance Without Compromise.** Rather than pretraining a domain-specific language model, Innovator-VL leverages the existing STEM knowledge in Qwen3-8B and focuses scientific investment in three areas: vision encoding (RICE-ViT captures fine-grained region-level features needed for molecular structures and micrographs), curated scientific SFT data (human-in-the-loop annotation with active learning for OCSR, chemical reactions, EM images), and targeted RL with only 5% science allocation in the RL mixture. This avoids catastrophic forgetting of general capabilities while achieving state-of-the-art on scientific benchmarks like MolParse (64.9% vs 4.75% for InternVL3.5).

## Step-by-Step Workflow

1. **Define the scientific domain and task taxonomy.** Enumerate the specific visual-scientific tasks your model must handle (e.g., molecular structure recognition, reaction diagram parsing, micrograph classification). Map each task to required visual granularity—whole-image understanding vs. region-level parsing—to determine vision encoder requirements.

2. **Select and configure the model architecture.** Choose a vision encoder with region-level capability (e.g., RICE-ViT or a ViT with region transformer layers), a token-compressing projector (PatchMerger or similar Q-Former/Perceiver), and a language backbone with strong existing STEM knowledge (Qwen3, LLaMA, etc.). Wire them together: image -> vision encoder -> projector -> LLM input embeddings.

3. **Stage 1 — Projector alignment training.** Freeze the vision encoder and LLM. Train only the projector on ~500K–1M high-quality image-caption pairs (e.g., LLaVA-1.5 558K or equivalent). This teaches the projector to map visual features into the LLM's embedding space without disturbing pretrained weights.

4. **Stage 2 — Full-parameter mid-training with concept-balanced sampling.** Unfreeze all parameters. Train on a larger dataset (10M–100M samples) using concept-balanced sampling: encode images with a CLIP model, cluster by semantic concept, and sample uniformly across clusters to prevent distribution skew. Generate pseudo-captions for images lacking text. Use offline data packing (bucket-based concatenation of short sequences) to maximize GPU utilization.

5. **Stage 3 — Supervised fine-tuning with three data streams.** Construct an SFT dataset mixing: (a) general multimodal instruction data (~48%: captioning, chart/table QA, code, OCR, grounding), (b) chain-of-thought reasoning data (~33%: multi-step STEM problems with reasoning traces, strip explicit think tags to reduce noise), and (c) domain-specific scientific data (~19%: your target scientific tasks with human-verified annotations). Use a human-in-the-loop pipeline for scientific data: synthetic bootstrapping -> model pre-annotation -> confidence-based filtering (keep 0.6–0.9 confidence for expert review) -> periodic retraining.

6. **Stage 4 — Discrepancy-driven RL sample selection.** Generate N completions per problem (e.g., N=8–16). Compute Pass@1 and Pass@N. Select samples where the gap is meaningful (model sometimes succeeds, often fails). Discard trivially solved (Pass@1 near 1.0) and impossible (Pass@N near 0.0) problems. Target RL mixture: ~56% STEM/code, ~35% general multimodal, ~5% scientific domain, ~4% spatial/grounding/OCR.

7. **Stage 4 — Implement GSPO training loop.** Define the reward function as `R = 0.1 * r_format + 0.9 * r_accuracy`. For format reward: 1.0 for correct `<think>...</think><answer>...</answer>` structure, 0.8 for mathematical reasoning with discernible answer lacking tags, 0.0 for unstructured output. For accuracy: apply rule-based regex extraction first, then symbolic math verification (e.g., `math_verify`), then LLM-as-judge fallback for open-ended questions. Use sequence-level importance sampling with length-normalized likelihoods to avoid token-level bias.

8. **Evaluate across all three dimensions.** Use deterministic decoding (temperature=0.0, top_p=1.0). Benchmark on: (a) general vision tasks (AI2D, ChartQA, DocVQA, MMMU), (b) math/reasoning (MathVista, MathVerse, WeMath), (c) domain-specific science benchmarks. Track the accuracy-to-token ratio as an efficiency metric—Innovator-VL achieves 3.9–4.3x better token efficiency than comparable models.

9. **Iterate on data quality, not data quantity.** If performance on a scientific subtask is weak, invest in targeted human-in-the-loop annotation for that subtask rather than scaling data broadly. Use confidence-based active learning: run the current model on unlabeled data, select mid-confidence predictions (0.6–0.9) for expert correction, retrain, repeat.

10. **Package the pipeline for reproducibility.** Document every stage: data sources, filtering criteria, mixture ratios, training hyperparameters, evaluation protocols. Provide scripts for each stage that can be run end-to-end. This is a core Innovator-VL principle—transparent methodology enables community extension.

## Concrete Examples

**Example 1: Building a molecular structure recognition pipeline**

User: "I want to fine-tune a VLM to recognize chemical structures from patent images and output SMILES notation."

Approach:
1. Start with a pretrained VLM (e.g., Qwen2.5-VL or InternVL2) that already has aligned vision-language representations.
2. Curate OCSR training data using the Innovator-VL two-stage approach:
   - Generate ~50K synthetic molecular images from known SMILES using RDKit/Indigo renderers with varied styles (bond lengths, fonts, noise).
   - Run the model on real patent PDFs, extract molecular images via object detection, generate SMILES predictions.
   - Use 5-fold ensemble confidence scoring and SMILES-similarity checks. Route mid-confidence samples (0.6–0.9) to chemistry experts for correction.
3. Format data as image-instruction pairs: `{"image": "mol_001.png", "instruction": "Parse this molecular structure to SMILES.", "output": "CC(=O)Oc1ccccc1C(=O)O"}`.
4. SFT with mixture: 60% synthetic pairs, 30% human-verified real pairs, 10% general chemistry QA to maintain broad understanding.
5. RL stage: generate 8 SMILES per image, use RDKit canonical SMILES comparison for accuracy reward, structured output format for format reward.

Output:
```python
# Data curation pipeline skeleton
from rdkit import Chem
from rdkit.Chem import Draw
import json

def generate_synthetic_ocsr_pair(smiles: str, output_path: str) -> dict:
    """Generate a synthetic molecular image and training pair."""
    mol = Chem.MolFromSmiles(smiles)
    if mol is None:
        return None
    img = Draw.MolToImage(mol, size=(300, 300))
    img.save(output_path)
    return {
        "image": output_path,
        "conversations": [
            {"role": "user", "content": "<image>\nParse this molecular structure to SMILES."},
            {"role": "assistant", "content": Chem.MolToSmiles(mol)}  # canonical
        ]
    }

def confidence_filter(predictions: list[dict], low=0.6, high=0.9) -> list[dict]:
    """Select mid-confidence samples for human review (active learning)."""
    return [p for p in predictions if low <= p["confidence"] <= high]

# RL reward function
def ocsr_accuracy_reward(predicted: str, ground_truth: str) -> float:
    pred_mol = Chem.MolFromSmiles(predicted)
    gt_mol = Chem.MolFromSmiles(ground_truth)
    if pred_mol is None:
        return 0.0
    return 1.0 if Chem.MolToSmiles(pred_mol) == Chem.MolToSmiles(gt_mol) else 0.0
```

**Example 2: Designing an RL training loop with GSPO-style rewards**

User: "I have a fine-tuned scientific VLM and want to improve its reasoning with RL. How do I set up the reward function and sample selection?"

Approach:
1. Run inference on a candidate RL dataset with N=16 samples per problem using temperature sampling.
2. Compute Pass@1 (greedy) and Pass@16 for each problem. Filter to keep problems where 0.1 < Pass@16 - Pass@1 < 0.8.
3. Implement the composite reward function with format and accuracy components.
4. Set up the training loop with sequence-level importance sampling.

Output:
```python
import re
import math

def format_reward(response: str) -> float:
    """Score structured reasoning format compliance."""
    has_think = bool(re.search(r"<think>.*?</think>", response, re.DOTALL))
    has_answer = bool(re.search(r"<answer>.*?</answer>", response, re.DOTALL))
    if has_think and has_answer:
        return 1.0
    # Partial credit: has mathematical work with boxed answer
    if re.search(r"\\boxed\{.*?\}", response):
        return 0.8
    return 0.0

def accuracy_reward(response: str, ground_truth: str, task_type: str) -> float:
    """Hierarchical accuracy verification: rules -> symbolic -> LLM judge."""
    # Level 1: Rule-based extraction
    answer_match = re.search(r"<answer>(.*?)</answer>", response, re.DOTALL)
    extracted = answer_match.group(1).strip() if answer_match else response.strip()

    # Level 2: MCQ pattern matching
    if task_type == "mcq":
        pred_letter = re.search(r"^[A-E]", extracted)
        return 1.0 if pred_letter and pred_letter.group() == ground_truth else 0.0

    # Level 3: Symbolic math verification
    if task_type == "math":
        try:
            from math_verify import verify_math_equivalence
            return 1.0 if verify_math_equivalence(extracted, ground_truth) else 0.0
        except Exception:
            pass

    # Level 4: Relaxed string matching fallback
    return 1.0 if extracted.lower().strip() == ground_truth.lower().strip() else 0.0

def composite_reward(response: str, ground_truth: str, task_type: str, alpha=0.1):
    """Weighted reward: format compliance + accuracy."""
    r_fmt = format_reward(response)
    r_acc = accuracy_reward(response, ground_truth, task_type)
    return alpha * r_fmt + (1 - alpha) * r_acc

def select_rl_samples(problems: list[dict], n_samples: int = 16) -> list[dict]:
    """Discrepancy-driven selection: keep medium-difficulty problems."""
    selected = []
    for problem in problems:
        pass_at_1 = problem["pass_at_1"]
        pass_at_n = problem["pass_at_n"]
        gap = pass_at_n - pass_at_1
        if 0.1 < gap < 0.8:  # medium difficulty band
            selected.append(problem)
    return selected
```

**Example 3: Setting up concept-balanced sampling for mid-training data**

User: "I have 50M unlabeled scientific images. How do I sample a balanced training set?"

Approach:
1. Encode all images with a CLIP model (e.g., MetaCLIP-H/14).
2. Cluster embeddings into semantic concepts using k-means.
3. Sample uniformly across clusters rather than from the raw distribution.
4. Generate pseudo-captions for images without text using a captioning model.

Output:
```python
import numpy as np
from sklearn.cluster import MiniBatchKMeans

def concept_balanced_sample(embeddings: np.ndarray, n_concepts: int = 1000,
                            samples_per_concept: int = 500) -> np.ndarray:
    """Sample uniformly across semantic clusters for distribution balance."""
    kmeans = MiniBatchKMeans(n_clusters=n_concepts, batch_size=10000)
    labels = kmeans.fit_predict(embeddings)

    selected_indices = []
    for cluster_id in range(n_concepts):
        cluster_indices = np.where(labels == cluster_id)[0]
        if len(cluster_indices) == 0:
            continue
        n_select = min(samples_per_concept, len(cluster_indices))
        chosen = np.random.choice(cluster_indices, size=n_select, replace=False)
        selected_indices.extend(chosen.tolist())

    return np.array(selected_indices)

def bucket_pack_sequences(sequences: list[dict], max_length: int = 4096) -> list[list[dict]]:
    """Offline packing: concatenate short sequences into dense batches."""
    sequences_sorted = sorted(sequences, key=lambda s: s["length"])
    packed = []
    current_pack, current_len = [], 0

    for seq in sequences_sorted:
        if current_len + seq["length"] <= max_length:
            current_pack.append(seq)
            current_len += seq["length"]
        else:
            if current_pack:
                packed.append(current_pack)
            current_pack, current_len = [seq], seq["length"]

    if current_pack:
        packed.append(current_pack)
    return packed
```

## Best Practices

- **Do:** Invest in human-in-the-loop data quality over raw data quantity. A confidence-filtered active learning loop (synthesize -> predict -> filter mid-confidence -> expert correct -> retrain) consistently outperforms scaling up noisy data.
- **Do:** Use discrepancy-driven sample selection for RL. Measuring Pass@N vs Pass@1 gap is the most reliable way to find problems with high learning signal. Target the medium-difficulty band where the model has partial capability.
- **Do:** Maintain a three-stream data mixture during SFT (general instruction, chain-of-thought reasoning, domain-specific science) to prevent catastrophic forgetting of general capabilities while adding scientific alignment.
- **Do:** Track accuracy-to-token ratio, not just accuracy. A model that achieves 70% accuracy in 200 tokens is more deployable than one achieving 72% in 800 tokens. GSPO's format reward naturally encourages concise reasoning.
- **Avoid:** Pretraining a domain-specific language model from scratch when a strong general LLM already encodes substantial STEM knowledge. The marginal gain rarely justifies the compute cost and overfitting risk.
- **Avoid:** Including explicit `<think>` tags in SFT chain-of-thought data. Innovator-VL found these introduce noise; instead, preserve the reasoning structure but strip the meta-tags and let RL learn the structured format.

## Error Handling

- **Data pipeline produces low-quality scientific annotations:** Implement the 5-fold ensemble confidence check. If fewer than 20% of samples fall in the 0.6–0.9 confidence band, your base model is too weak for active learning—invest in more synthetic data first.
- **RL training collapses or reward hacking:** Check that your accuracy reward uses hierarchical verification (rules -> symbolic -> LLM judge). Single-method verification often has exploitable gaps. Also verify that the format reward weight (alpha) is small (0.1); higher values incentivize format gaming over correctness.
- **Catastrophic forgetting of general capabilities after scientific SFT:** Your scientific data proportion is too high. Innovator-VL caps scientific data at ~19% of the SFT mixture. Reduce domain-specific share and re-evaluate on general benchmarks (MMMU, ChartQA) after each training run.
- **Token-level importance sampling instability in RL:** Switch to sequence-level importance sampling (GSPO approach). Compute length-normalized sequence likelihoods `s = (pi_new / pi_old)^(1/|y|)` instead of per-token ratios to stabilize the trust region.
- **Concept-balanced sampling produces semantically redundant clusters:** Increase the number of clusters or use hierarchical clustering. Verify cluster quality by sampling and visually inspecting 10 images per cluster before committing to a training run.

## Limitations

- **Requires a strong pretrained VLM base.** The Innovator-VL methodology assumes you start from an already-capable vision-language model (RICE-ViT + Qwen3-8B). The data-efficient training recipe does not bootstrap from scratch—it refines and aligns an existing model.
- **Scientific domains with no existing benchmarks are hard to evaluate.** The RL reward function and discrepancy-driven selection depend on automated correctness checking. Novel scientific tasks without rule-based or symbolic verifiers require expensive LLM-as-judge evaluation.
- **Human-in-the-loop data curation requires domain experts.** The active learning pipeline for OCSR, chemical reactions, and micrographs depends on chemists, materials scientists, etc. for the verification loop. This is not automatable for highly specialized domains.
- **Sequence-level GSPO is more memory-intensive than token-level methods.** Computing sequence-level importance weights requires storing full-sequence log-probabilities, increasing memory pressure during RL training.
- **The 5M sample efficiency claim is specific to scientific alignment on top of a general-purpose VLM.** Building the underlying general VLM still requires the standard large-scale pretraining investment.

## Reference

- **Paper:** [Innovator-VL: A Multimodal Large Language Model for Scientific Discovery](https://arxiv.org/abs/2601.19325v1) — Look for: the four-stage training pipeline (Section 3), GSPO algorithm and reward design (Section 3.4), discrepancy-driven RL data selection (Section 3.4.1), human-in-the-loop OCSR pipeline (Section 4.1), and concept-balanced mid-training sampling (Section 3.2).