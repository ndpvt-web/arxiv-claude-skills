---
name: "c3box-clip-based-class-incremental-learning"
description: "Set up, configure, and run CLIP-based class-incremental learning experiments using the C3Box toolbox. Supports 17 CIL algorithms (CLIP-native, prompt-based, adapter-based, traditional) across 10 benchmark datasets with JSON-driven configuration. Triggers: 'set up class-incremental learning', 'run continual learning experiment with CLIP', 'configure C3Box pipeline', 'benchmark CIL methods', 'avoid catastrophic forgetting with CLIP', 'incremental learning toolbox setup'"
---

C3Box (CLIP-based Class-inCremental learning toolBOX) is a unified Python framework that integrates 17 class-incremental learning algorithms — spanning traditional CIL, ViT prompt-based, adapter-based, and CLIP-native methods — into a single reproducible pipeline. This skill enables Claude to help users install, configure, run, and interpret experiments from C3Box, select appropriate methods for their continual learning scenario, author JSON experiment configs, debug training pipelines, and adapt the toolbox to custom datasets.

## When to Use

- When the user wants to set up a class-incremental learning experiment using CLIP as the backbone
- When the user asks to benchmark multiple continual learning methods side-by-side with fair comparisons
- When the user needs to configure a JSON experiment file for a specific CIL protocol (e.g., B0-Inc10, B50-Inc10)
- When the user wants to adapt C3Box to a custom image classification dataset
- When the user asks how to avoid catastrophic forgetting when fine-tuning CLIP on a stream of new classes
- When the user wants to reproduce results from CLIP-based CIL papers (RAPF, PROOF, BOFA, ENGINE, MG-CLIP, etc.)
- When the user needs to select between prompt-tuning, adapter, or CLIP-native approaches for their use case
- When the user asks to automate multi-seed or multi-dataset experiment sweeps with C3Box

## Key Technique

C3Box unifies three families of class-incremental learning under CLIP's vision-language backbone. **Traditional CIL methods** (FOSTER, MEMO) originally designed for CNNs are adapted to use only CLIP's visual encoder while keeping their core mechanisms — feature boosting/compression and memory-efficient decoupled layers. **Prompt-based ViT methods** (L2P, DualPrompt, CODA-Prompt) learn task-specific prompt tokens inserted into the frozen ViT layers, steering CLIP's representations without modifying its weights. **CLIP-native methods** (RAPF, PROOF, BOFA, MG-CLIP, ENGINE, CLG-CBM) exploit both visual and textual branches, using techniques like cross-modal fusion, orthogonal low-rank adapters in bridge layers, modality gap preservation, and external knowledge injection.

The framework follows the standard CIL protocol defined as "B-mm Inc-nn": the model first learns `mm` base classes, then incrementally learns `nn` new classes per session. At each session, the model is evaluated on all classes seen so far. Replay-based methods retain a fixed number of exemplars per class (typically 20) selected via the herding algorithm. All configuration — method selection, backbone weights, dataset, incremental schedule, memory budget, optimizer, and hyperparameters — is specified in a single JSON file passed to `main.py`, eliminating code modification for standard experiments.

The critical architectural insight is that CLIP's pre-trained semantic alignment provides a strong initialization that reduces forgetting by default (zero-shot CLIP already achieves competitive baselines), so the incremental methods focus on *efficiently adapting* rather than *learning from scratch*. This means methods that minimally perturb CLIP's feature space (SimpleCIL, MG-CLIP) can outperform heavyweight replay approaches.

## Step-by-Step Workflow

1. **Clone and install C3Box** with its dependencies (PyTorch 2.0.1+, torchvision, timm 0.6.12, open_clip 2.17.1, numpy, scipy, tqdm, easydict):
   ```bash
   git clone https://github.com/LAMDA-CL/C3Box && cd C3Box
   pip install torch==2.0.1 torchvision==0.15.2 timm==0.6.12 open_clip_torch==2.17.1 numpy scipy tqdm easydict
   ```

2. **Select a method** by identifying the user's scenario — use the method selection guide below to choose among the 17 supported algorithms based on whether memory storage is allowed, whether both CLIP modalities are available, and the compute budget.

3. **Prepare the dataset** — CIFAR-100 auto-downloads; for others (CUB-200, ImageNet-R, ObjectNet, Cars, UCF, Aircraft, Food, SUN, TV100), download manually and set `train_dir` and `test_dir` paths in `utils/data.py` under `download_data()`.

4. **Author a JSON experiment config** in the `exps/` directory specifying: `model_name`, `backbone_type` (LAION-400M or OpenAI ViT-B/16 weights), `init_cls`, `increment`, `seed` (default 1993), `memory_size` or `memory_per_class`, and method-specific hyperparameters (`batch_size`, `learning_rate`, `tuned_epoch`, `optimizer`, `weight_decay`).

5. **Run the experiment** via the single entry point:
   ```bash
   python main.py --config=./exps/your_config.json
   ```

6. **Monitor training** — `trainer.py` orchestrates the incremental loop, logging per-session accuracy. Watch for Last Accuracy (A_B), Average Accuracy (A-bar), and Forgetting (F_B) in the output.

7. **Compare methods** by running multiple configs with identical `seed`, `init_cls`, `increment`, and dataset settings, changing only `model_name` and its method-specific parameters.

8. **Automate sweeps** by scripting over seeds (1993, 1994, 1995) and datasets to produce statistically robust comparisons:
   ```bash
   for seed in 1993 1994 1995; do
     python main.py --config=./exps/mg_clip.json --seed=$seed
   done
   ```

9. **Integrate a custom dataset** by adding a new entry in `utils/data.py` following the existing pattern: define class count, train/test directory paths, and any preprocessing transforms matching CLIP's expected input (224x224, normalized to CLIP stats).

10. **Interpret results** — compare A-bar (higher is better) and F_B (lower is better) across methods. Methods with no memory requirement (SimpleCIL, EASE, TUNA, CLG_CBM, MG_CLIP, ENGINE, BOFA) are preferable when storage is constrained.

## Method Selection Guide

| Scenario | Recommended Methods | Memory Required |
|---|---|---|
| Zero engineering, strong baseline | ZS-CLIP, SimpleCIL | No |
| Minimal forgetting, no replay buffer | MG-CLIP, ENGINE, BOFA | No |
| Prompt-tuning approach | L2P, DualPrompt, CODA-Prompt | Yes |
| Adapter-based PEFT | EASE, APER, TUNA | No (EASE, TUNA) / Yes (APER) |
| Cross-modal fusion with CLIP text | PROOF, RAPF, CLG-CBM | No (CLG_CBM) / Yes (RAPF, PROOF) |
| Traditional CIL on CLIP visual encoder | FOSTER, MEMO | Yes |

## Concrete Examples

**Example 1: Benchmark CLIP-native methods on CIFAR-100**

User: "I want to compare MG-CLIP, BOFA, and ENGINE on CIFAR-100 with 10 classes per increment starting from scratch (B0-Inc10)."

Approach:
1. Create three JSON configs in `exps/`, each with `init_cls: 0`, `increment: 10`, `dataset: cifar100`, `backbone_type: "clip_laion400m"`, `seed: 1993`
2. Set `model_name` to `mg_clip`, `bofa`, and `engine` respectively
3. Run all three experiments sequentially or in parallel

Config for MG-CLIP (`exps/mg_clip_cifar100.json`):
```json
{
    "model_name": "mg_clip",
    "backbone_type": "clip_laion400m",
    "dataset": "cifar100",
    "init_cls": 0,
    "increment": 10,
    "seed": 1993,
    "batch_size": 64,
    "learning_rate": 0.001,
    "tuned_epoch": 20,
    "optimizer": "adam",
    "weight_decay": 5e-4
}
```

Execution:
```bash
python main.py --config=./exps/mg_clip_cifar100.json
python main.py --config=./exps/bofa_cifar100.json
python main.py --config=./exps/engine_cifar100.json
```

Output: Per-session accuracy table plus final A_B, A-bar, and F_B for each method. Expected: MG-CLIP ~88.7%, ENGINE and BOFA in similar range on CIFAR-100.

---

**Example 2: Add a custom dataset (Stanford Dogs) to C3Box**

User: "I have the Stanford Dogs dataset and want to run SimpleCIL and RAPF on it with B0-Inc20."

Approach:
1. Download Stanford Dogs, organize into `train/` and `test/` directories with one subfolder per class
2. Register the dataset in `utils/data.py` by adding a new entry in the `download_data()` function:

```python
elif dataset_name == "stanford_dogs":
    train_dir = "/path/to/stanford_dogs/train"
    test_dir = "/path/to/stanford_dogs/test"
    num_classes = 120
```

3. Create JSON configs:

```json
{
    "model_name": "simplecil",
    "backbone_type": "clip_laion400m",
    "dataset": "stanford_dogs",
    "init_cls": 0,
    "increment": 20,
    "seed": 1993
}
```

4. Run: `python main.py --config=./exps/simplecil_dogs.json`

---

**Example 3: Automate a full reproducibility sweep**

User: "Run FOSTER, L2P, and PROOF across three seeds on ImageNet-R with B0-Inc10 and 20 exemplars per class."

Approach:
1. Ensure ImageNet-R paths are set in `utils/data.py`
2. Write a shell script to iterate over methods and seeds:

```bash
#!/bin/bash
METHODS=("foster" "l2p" "proof")
SEEDS=(1993 1994 1995)

for method in "${METHODS[@]}"; do
  for seed in "${SEEDS[@]}"; do
    cat > /tmp/config_${method}_${seed}.json <<JSONEOF
{
    "model_name": "${method}",
    "backbone_type": "clip_laion400m",
    "dataset": "imagenet_r",
    "init_cls": 0,
    "increment": 10,
    "seed": ${seed},
    "fixed_memory": true,
    "memory_per_class": 20,
    "batch_size": 64,
    "learning_rate": 0.001,
    "tuned_epoch": 20,
    "optimizer": "sgd",
    "weight_decay": 5e-4
}
JSONEOF
    python main.py --config=/tmp/config_${method}_${seed}.json
  done
done
```

3. Aggregate results by computing mean and std of A-bar and F_B across the three seeds per method.

## Best Practices

- **Do:** Always use the same `seed`, `init_cls`, `increment`, and `backbone_type` across methods when benchmarking — inconsistent settings invalidate comparisons.
- **Do:** Start with ZS-CLIP and SimpleCIL as baselines before running more complex methods; they establish the floor and a surprisingly strong reference.
- **Do:** Use `backbone_type: "clip_laion400m"` for consistency with C3Box's published results; OpenAI weights may yield different rankings.
- **Do:** Set `fixed_memory: true` with `memory_per_class: 20` for replay methods to match the standard protocol in CIL literature.
- **Avoid:** Setting `memory_size` or `memory_per_class` for methods that don't use exemplars (ZS-CLIP, SimpleCIL, EASE, TUNA, CLG_CBM, MG_CLIP, ENGINE, BOFA) — it wastes storage and may cause unexpected behavior.
- **Avoid:** Modifying CLIP's backbone weights directly; the framework is designed around frozen or minimally-adapted encoders. Unfreezing all parameters leads to severe forgetting.

## Error Handling

- **CUDA out of memory**: Reduce `batch_size` in the JSON config. Prompt-based methods (L2P, DualPrompt, CODA-Prompt) consume more GPU memory due to prompt pool storage; start with batch size 32.
- **Dataset not found**: Verify that `train_dir` and `test_dir` are correctly set in `utils/data.py` for non-CIFAR100 datasets. CIFAR-100 auto-downloads; all others require manual setup.
- **OpenCLIP model download failure**: The CLIP backbone downloads on first run. If behind a proxy, set `HTTP_PROXY`/`HTTPS_PROXY` environment variables or pre-download the ViT-B/16 weights and point `backbone_type` to the local path.
- **Mismatched class counts**: Ensure `init_cls + N * increment` equals the total number of classes in the dataset. For example, CIFAR-100 with B0-Inc10 means 10 sessions of 10 classes each (100 total).
- **Seed mismatch in comparisons**: The seed controls class order shuffling. Different seeds produce different class orderings, making cross-seed accuracy comparisons meaningless for individual sessions — always aggregate with mean/std.
- **Method not recognized**: Verify `model_name` matches one of the 17 supported identifiers exactly: `finetune`, `zs_clip`, `foster`, `memo`, `simplecil`, `l2p`, `dual`, `coda`, `ease`, `aper`, `tuna`, `rapf`, `clg_cbm`, `mg_clip`, `proof`, `engine`, `bofa`.

## Limitations

- **Backbone locked to ViT-B/16**: C3Box currently supports only CLIP ViT-B/16 (LAION-400M or OpenAI). Larger backbones (ViT-L/14, ViT-G) are not yet integrated.
- **Single-GPU only**: The toolbox does not include distributed training support. Large datasets like ImageNet-R may train slowly on a single GPU.
- **Image classification only**: C3Box handles image classification tasks. It does not support detection, segmentation, or other vision tasks in an incremental setting.
- **No task-ID at inference**: All methods operate in the class-incremental (task-agnostic) setting. If your scenario provides task identifiers at test time, C3Box does not exploit them.
- **Limited hyperparameter search**: The toolbox provides default configs that reproduce published results but does not include automated hyperparameter tuning utilities.
- **Ten datasets only**: Adding a new dataset requires manual registration in `utils/data.py` — there is no plugin mechanism or dataset registry.

## Reference

Sun, H. & Zhou, D.-W. (2026). *C3Box: A CLIP-based Class-Incremental Learning Toolbox*. arXiv:2601.20852. [https://arxiv.org/abs/2601.20852](https://arxiv.org/abs/2601.20852)

Look for: Table 1 (method taxonomy), Section 3 (framework architecture and JSON config design), Section 4 (benchmark results across 10 datasets showing where CLIP-native methods surpass traditional CIL).

Code: [https://github.com/LAMDA-CL/C3Box](https://github.com/LAMDA-CL/C3Box)