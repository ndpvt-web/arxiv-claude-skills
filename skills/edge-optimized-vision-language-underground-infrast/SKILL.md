---
name: "edge-optimized-vision-language-underground-infrast"
description: "Build edge-deployable two-stage pipelines that combine lightweight segmentation with quantized Vision-Language Models for automated infrastructure inspection and defect summarization. Triggers: 'build an edge AI inspection pipeline', 'deploy VLM on Jetson', 'sewer defect detection and summarization', 'quantize vision-language model for edge', 'RAPID-SCAN segmentation pipeline', 'infrastructure assessment with VLM'"
---

# Edge-Optimized Vision-Language Pipelines for Underground Infrastructure Assessment

This skill enables Claude to help users build, fine-tune, quantize, and deploy two-stage AI pipelines that pair ultra-lightweight segmentation models (~0.64M parameters) with quantized Vision-Language Models (VLMs) on edge hardware like NVIDIA Jetson. The architecture follows the RAPID-SCAN + Phi-3.5 VLM approach from Lopez et al. (2026): Stage 1 segments defects at <50ms/frame, Stage 2 generates structured natural-language summaries via a QLoRA-fine-tuned VLM running in INT8+TensorRT at ~2.3s per inference — achieving end-to-end image-to-summary latency of ~3.1 seconds on a Jetson AGX Orin.

## When to Use

- When the user wants to build a defect detection + natural-language summarization pipeline for infrastructure inspection (sewers, culverts, bridges, tunnels)
- When deploying a Vision-Language Model on edge devices (Jetson Orin, Jetson Nano, or similar ARM+GPU platforms) and needs quantization guidance
- When the user asks to fine-tune Phi-3.5-vision (or similar small VLMs like Phi-4-vision, Florence-2, Moondream) with QLoRA for domain-specific image captioning
- When building a lightweight segmentation model under 1M parameters using Dynamic Feature Pyramid Networks with Squeeze-and-Excitation modules
- When the user needs to convert a PyTorch VLM pipeline to TensorRT with mixed INT8/FP16 precision for real-time inference
- When creating structured inspection reports from segmentation masks — mapping visual defects to condition, location, severity, and maintenance implications
- When integrating an AI vision pipeline into ROS (Robot Operating System) for autonomous robotic inspection

## Key Technique

The core innovation is a **two-stage decoupled architecture** that separates fast pixel-level detection from slow language generation. Stage 1 uses RAPID-SCAN, an extremely compact segmentation network (0.64M parameters, 0.19 GFLOPs) built on a Dynamic Feature Pyramid Network with adaptive routing and Squeeze-and-Excitation attention. It achieves 0.834 F1-score and 0.729 mIoU on infrastructure defect datasets — competitive with models 50x larger — while running at <50ms per frame on edge GPUs. This decoupling means the expensive VLM only processes frames where defects are actually detected, drastically reducing compute load.

Stage 2 fine-tunes Microsoft's Phi-3.5-vision (3.8B parameters) using **QLoRA** — quantizing base weights to 4-bit NormalFloat4 (NF4) while training only 67M adapter parameters (1.8% of the model). The fine-tuning uses structured prompts that combine the RGB image, the segmentation mask overlay, and defect class labels to generate summaries covering four dimensions: Condition (what defect), Location (where in the pipe), Severity (how bad), and Implications (maintenance urgency). Training uses rank r=16, scaling alpha=32, dropout 0.1, learning rate 2e-4 with cosine annealing, batch size 4 with 8 gradient accumulation steps, for 3 epochs.

For deployment, the fine-tuned model undergoes **post-training quantization**: symmetric per-channel INT8 for language transformer layers and FP16 for the vision encoder, compiled with TensorRT 8.6. This reduces the model from 6.8GB to 2.1GB (69% reduction) and achieves 3.2x speedup over PyTorch, with ROUGE-L degradation under 2%. The full pipeline fits within 8GB GPU memory (6GB weights + 2GB dynamic tensors) on a Jetson AGX Orin under 75C thermal constraints.

## Step-by-Step Workflow

### 1. Define Defect Taxonomy and Dataset Structure

Establish the defect categories for your domain. For sewer/culvert inspection, use the 8-class taxonomy: cracks, roots, holes, joint problems, deformation, fracture, erosion/deposits, loose gasket. Create a dataset directory with RGB images, pixel-level segmentation masks, and structured text annotations covering condition, location, severity, and implications.

```python
# Dataset directory structure
# dataset/
#   train/
#     images/        # RGB images (1920x1080 or resized)
#     masks/         # Segmentation masks (class indices per pixel)
#     summaries.json # {"image_id": {"condition": "...", "location": "...",
#                    #   "severity": "...", "implications": "..."}}
#   val/
#     images/
#     masks/
#     summaries.json
```

### 2. Build the Lightweight Segmentation Model (RAPID-SCAN Architecture)

Implement a compact encoder-decoder with a Dynamic Feature Pyramid Network, adaptive routing, and Squeeze-and-Excitation (SE) modules. Target <1M parameters.

```python
import torch
import torch.nn as nn

class SEBlock(nn.Module):
    """Squeeze-and-Excitation block for channel attention."""
    def __init__(self, channels, reduction=16):
        super().__init__()
        self.pool = nn.AdaptiveAvgPool2d(1)
        self.fc = nn.Sequential(
            nn.Linear(channels, channels // reduction, bias=False),
            nn.ReLU(inplace=True),
            nn.Linear(channels // reduction, channels, bias=False),
            nn.Sigmoid()
        )

    def forward(self, x):
        b, c, _, _ = x.size()
        w = self.pool(x).view(b, c)
        w = self.fc(w).view(b, c, 1, 1)
        return x * w.expand_as(x)

class DynamicFPN(nn.Module):
    """Dynamic Feature Pyramid with adaptive routing."""
    def __init__(self, in_channels_list, out_channels=64):
        super().__init__()
        self.lateral_convs = nn.ModuleList([
            nn.Conv2d(c, out_channels, 1) for c in in_channels_list
        ])
        self.se_blocks = nn.ModuleList([
            SEBlock(out_channels) for _ in in_channels_list
        ])
        # Adaptive routing: learnable weights for multi-scale fusion
        self.route_weights = nn.Parameter(torch.ones(len(in_channels_list)))

    def forward(self, features):
        laterals = [conv(f) for conv, f in zip(self.lateral_convs, features)]
        laterals = [se(l) for se, l in zip(self.se_blocks, laterals)]
        weights = torch.softmax(self.route_weights, dim=0)
        # Upsample all to largest resolution and fuse
        target_size = laterals[0].shape[2:]
        fused = sum(
            w * nn.functional.interpolate(l, size=target_size, mode='bilinear',
                                          align_corners=False)
            for w, l in zip(weights, laterals)
        )
        return fused

class RAPIDSCANSegmenter(nn.Module):
    """Lightweight segmentation model targeting <1M parameters."""
    def __init__(self, num_classes=9, base_channels=16):
        super().__init__()
        # Compact encoder (depthwise separable convolutions)
        self.enc1 = self._make_block(3, base_channels)           # 16
        self.enc2 = self._make_block(base_channels, base_channels * 2)  # 32
        self.enc3 = self._make_block(base_channels * 2, base_channels * 4) # 64
        self.pool = nn.MaxPool2d(2)
        self.fpn = DynamicFPN([base_channels, base_channels * 2,
                               base_channels * 4], out_channels=base_channels * 2)
        self.head = nn.Conv2d(base_channels * 2, num_classes, 1)

    def _make_block(self, in_c, out_c):
        return nn.Sequential(
            nn.Conv2d(in_c, in_c, 3, padding=1, groups=in_c),  # depthwise
            nn.Conv2d(in_c, out_c, 1),  # pointwise
            nn.BatchNorm2d(out_c),
            nn.ReLU(inplace=True),
        )

    def forward(self, x):
        f1 = self.enc1(x)
        f2 = self.enc2(self.pool(f1))
        f3 = self.enc3(self.pool(f2))
        fused = self.fpn([f1, f2, f3])
        return self.head(fused)
```

### 3. Train the Segmentation Model

Train with combined cross-entropy and Dice loss, targeting F1 > 0.80 and mIoU > 0.70. Use aggressive augmentation (random flips, rotation, color jitter, elastic transforms) given the typically small inspection datasets.

```python
# Training configuration
config = {
    "epochs": 100,
    "lr": 1e-3,
    "optimizer": "AdamW",
    "scheduler": "CosineAnnealingLR",
    "loss": "CrossEntropy + DiceLoss (0.5 each)",
    "augmentation": ["HorizontalFlip", "RandomRotate90",
                     "ColorJitter", "ElasticTransform"],
    "input_size": (512, 512),  # resize from original resolution
    "batch_size": 16,
}
```

### 4. Prepare VLM Fine-Tuning Data

Create structured prompt-completion pairs that combine image, mask overlay, and defect labels into a four-aspect summary format.

```python
# Prompt template for VLM fine-tuning
PROMPT_TEMPLATE = """<|image|>
You are an underground infrastructure inspection expert. Given the inspection
image with segmentation overlay showing detected defects ({defect_classes}),
provide a structured assessment.

Generate a concise summary covering:
1. Condition: What defects are present
2. Location: Where in the structure they appear
3. Severity: Assessment of urgency
4. Implications: Recommended maintenance actions"""

# Example completion target:
# "Condition: A longitudinal crack extends along the pipe crown with root
#  intrusion at the 2 o'clock position. Location: Approximately 15 feet from
#  the manhole entry, spanning a 3-foot section. Severity: Moderate - the
#  crack shows active infiltration but no structural compromise. Implications:
#  Schedule CIPP lining within 6 months; monitor root growth quarterly."
```

### 5. Fine-Tune the VLM with QLoRA

Apply QLoRA to Phi-3.5-vision (or comparable small VLM), quantizing base weights to NF4 and training only LoRA adapters.

```python
from transformers import AutoModelForCausalLM, AutoProcessor, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

# 4-bit quantization config for base model
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)

model = AutoModelForCausalLM.from_pretrained(
    "microsoft/Phi-3.5-vision-instruct",
    quantization_config=bnb_config,
    trust_remote_code=True,
)
model = prepare_model_for_kbit_training(model)

# LoRA adapter config (paper's exact settings)
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    lora_dropout=0.1,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                     "gate_proj", "up_proj", "down_proj"],
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
# Trainable: ~67M / 3.8B total = 1.8%

# Training args
training_args = {
    "num_train_epochs": 3,
    "per_device_train_batch_size": 4,
    "gradient_accumulation_steps": 8,
    "learning_rate": 2e-4,
    "lr_scheduler_type": "cosine",
    "max_grad_norm": 1.0,
    "optimizer": "adamw_torch",
    "bf16": True,
}
```

### 6. Merge LoRA Adapters and Export for Edge Deployment

After training, merge adapters back into the base model and export to ONNX or TensorRT format.

```bash
# Merge LoRA weights
python -c "
from peft import PeftModel
from transformers import AutoModelForCausalLM
base = AutoModelForCausalLM.from_pretrained('microsoft/Phi-3.5-vision-instruct',
                                             trust_remote_code=True)
model = PeftModel.from_pretrained(base, './qlora-checkpoint')
merged = model.merge_and_unload()
merged.save_pretrained('./merged-model')
"

# Export segmentation model to ONNX
python -c "
import torch
model = torch.load('rapid_scan.pth')
dummy = torch.randn(1, 3, 512, 512)
torch.onnx.export(model, dummy, 'rapid_scan.onnx', opset_version=17)
"
```

### 7. Apply Post-Training Quantization with TensorRT

Quantize language layers to INT8 and keep vision encoder in FP16 for quality preservation.

```python
# TensorRT quantization script for Jetson
import tensorrt as trt

def build_engine(onnx_path, output_path, precision="int8"):
    logger = trt.Logger(trt.Logger.WARNING)
    builder = trt.Builder(logger)
    network = builder.create_network(
        1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH))
    parser = trt.OnnxParser(network, logger)

    with open(onnx_path, 'rb') as f:
        parser.parse(f.read())

    config = builder.create_builder_config()
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, 2 << 30)

    if precision == "int8":
        config.set_flag(trt.BuilderFlag.INT8)
        # Use calibration dataset for INT8 quantization
        config.int8_calibrator = EntropyCalibrator(calib_dataset)
    elif precision == "fp16":
        config.set_flag(trt.BuilderFlag.FP16)

    engine = builder.build_serialized_network(network, config)
    with open(output_path, 'wb') as f:
        f.write(engine)

# Segmentation model: FP16 (already tiny)
build_engine("rapid_scan.onnx", "rapid_scan.engine", precision="fp16")
```

### 8. Build the End-to-End Inference Pipeline

Wire segmentation and VLM into a single pipeline with a defect-gating mechanism — only invoke the VLM when defects are detected.

```python
class InspectionPipeline:
    def __init__(self, seg_engine_path, vlm_model_path):
        self.segmenter = load_tensorrt_engine(seg_engine_path)
        self.vlm = load_quantized_vlm(vlm_model_path)
        self.defect_classes = [
            "crack", "root", "hole", "joint_problem",
            "deformation", "fracture", "erosion", "loose_gasket"
        ]

    def process_frame(self, frame):
        # Stage 1: Fast segmentation (<50ms)
        mask = self.segmenter.infer(preprocess(frame))
        detected = self.get_detected_classes(mask)

        if not detected:
            return {"has_defects": False, "summary": None}

        # Stage 2: VLM summarization (~2.3s, only when defects found)
        overlay = self.create_overlay(frame, mask)
        prompt = format_prompt(detected)
        summary = self.vlm.generate(overlay, prompt, max_new_tokens=256)

        return {
            "has_defects": True,
            "defects": detected,
            "mask": mask,
            "summary": parse_structured_summary(summary),
        }

    def get_detected_classes(self, mask):
        unique = set(mask.flatten().tolist()) - {0}  # exclude background
        return [self.defect_classes[i - 1] for i in unique]
```

### 9. Deploy on Edge Hardware with Resource Monitoring

Set up GPU memory budgets and thermal monitoring for sustained field operation.

```python
# Jetson resource monitoring during inference
import subprocess

def check_jetson_resources():
    """Verify pipeline fits within edge constraints."""
    result = subprocess.run(["tegrastats", "--interval", "100"],
                           capture_output=True, text=True, timeout=5)
    # Enforce: <85% GPU memory, <75C thermal
    return parse_tegrastats(result.stdout)

# ROS integration for robotic deployment
# Launch file snippet for ROS Noetic
ROS_LAUNCH = """
<launch>
  <node name="camera" pkg="axis_camera" type="axis.py">
    <param name="hostname" value="192.168.0.90"/>
  </node>
  <node name="inspection_pipeline" pkg="inspection_ai" type="pipeline_node.py">
    <param name="seg_engine" value="$(find inspection_ai)/models/rapid_scan.engine"/>
    <param name="vlm_model" value="$(find inspection_ai)/models/phi35_int8/"/>
    <param name="publish_rate" value="0.3"/>  <!-- ~3.1s per summary -->
  </node>
</launch>
"""
```

### 10. Evaluate with Multi-Metric Assessment

Validate summarization quality using the paper's metric suite, with minimum quality thresholds.

```python
from evaluate import load

metrics = {
    "rouge": load("rouge"),       # Target: ROUGE-L >= 0.34
    "bleu": load("bleu"),         # Target: BLEU >= 0.14
    "bertscore": load("bertscore"),  # Target: F1 >= 0.88
    "meteor": load("meteor"),     # Target: METEOR >= 0.32
}

# Quality gate: reject quantized model if ROUGE-L drops >2% from FP32
def quality_gate(fp32_scores, quantized_scores, max_degradation=0.02):
    rouge_l_drop = fp32_scores["rougeL"] - quantized_scores["rougeL"]
    if rouge_l_drop > max_degradation:
        raise ValueError(
            f"ROUGE-L degradation {rouge_l_drop:.3f} exceeds threshold "
            f"{max_degradation}. Re-calibrate quantization."
        )
```

## Concrete Examples

**Example 1: Setting up QLoRA fine-tuning for pipe inspection VLM**

User: "I have 3,000 labeled sewer inspection images with defect descriptions. Help me fine-tune Phi-3.5-vision to generate structured inspection reports."

Approach:
1. Structure the dataset into image + mask + four-aspect summary format (condition, location, severity, implications)
2. Split 80/10/10 for train/val/test
3. Configure QLoRA with NF4 base quantization: r=16, alpha=32, dropout=0.1
4. Target all attention and MLP projection layers for LoRA adaptation
5. Train for 3 epochs with lr=2e-4, cosine schedule, batch 4, grad accum 8
6. Evaluate on held-out test set against ROUGE-L >= 0.34 and BERTScore >= 0.88

Output:
```
Fine-tuned model statistics:
  Trainable parameters: 67M / 3.8B (1.8%)
  Training loss: 0.42 -> 0.18 over 3 epochs
  Val ROUGE-L: 0.36  |  BERTScore: 0.90  |  METEOR: 0.34
  Checkpoint: ./qlora-checkpoint/adapter_model.safetensors (134MB)
```

**Example 2: Quantizing and deploying the pipeline on Jetson AGX Orin**

User: "I need to deploy my trained segmentation + VLM pipeline on a Jetson AGX Orin for real-time field inspection."

Approach:
1. Export RAPID-SCAN segmentation model to ONNX, then compile to TensorRT FP16 engine
2. Merge QLoRA adapters into base Phi-3.5-vision model
3. Apply symmetric per-channel INT8 quantization to language layers, keep vision encoder in FP16
4. Compile with TensorRT 8.6, allocating 6GB for weights + 2GB for dynamic tensors
5. Build gated pipeline: run segmenter on every frame, invoke VLM only when defects detected
6. Validate that ROUGE-L degradation is <2% vs FP32 baseline

Output:
```
Deployment summary (Jetson AGX Orin):
  Segmentation: rapid_scan.engine (1.2MB, FP16) — 47ms/frame
  VLM: phi35_int8/ (2.1GB, INT8+FP16) — 2.3s/summary
  End-to-end latency: 3.1s (image to structured report)
  Model size reduction: 6.8GB -> 2.1GB (69%)
  Speedup: 3.2x vs PyTorch FP32
  GPU memory: 7.8GB / 8GB allocated
  Thermal: 68C sustained (under 75C limit)
```

**Example 3: Building a lightweight segmentation model for custom defect types**

User: "I need a segmentation model under 1M parameters to detect 5 types of concrete defects (spalling, rebar exposure, efflorescence, scaling, popouts) on a Jetson Nano."

Approach:
1. Implement RAPID-SCAN-style architecture: depthwise separable encoder with 3 stages (16/32/64 channels)
2. Add Dynamic Feature Pyramid Network with SE attention for multi-scale fusion
3. Use combined CE + Dice loss with heavy augmentation
4. Train to convergence (~100 epochs), validate F1 > 0.80
5. Export to ONNX and compile FP16 TensorRT engine for Jetson Nano
6. Benchmark inference time target: <50ms/frame

Output:
```
Model: ConcreteSCAN
  Parameters: 0.58M  |  GFLOPs: 0.17
  F1: 0.821  |  mIoU: 0.714
  Inference: 43ms/frame on Jetson Nano (FP16 TensorRT)
  Engine size: 1.1MB
```

## Best Practices

**Do:**
- Gate VLM inference behind the segmentation stage — only generate summaries for frames with detected defects. This is the key efficiency insight: the fast segmenter acts as a filter.
- Use separate quantization strategies per component: FP16 for vision encoders (sensitive to quantization), INT8 for language transformer layers (robust to quantization).
- Always verify quantized model quality with a quality gate (ROUGE-L degradation < 2%) before deploying. Run the full evaluation suite, not just spot checks.
- Structure VLM outputs into the four-aspect format (Condition, Location, Severity, Implications) for actionable reports that maintenance crews can use directly.
- Use depthwise separable convolutions throughout the segmentation encoder to minimize parameters while preserving spatial detail.

**Avoid:**
- Running the VLM on every video frame. At 2.3s per inference, this would create an unmanageable backlog. Use segmentation-gated triggering.
- Quantizing the vision encoder below FP16. The paper shows that INT8 vision encoding degrades BERTScore significantly, while language layers tolerate INT8 well.
- Training with LoRA rank > 32 for this task. Rank 16 with alpha 32 achieves the best quality/efficiency trade-off; higher ranks overfit on small inspection datasets.
- Ignoring thermal throttling on edge devices. Set memory and thermal budgets (85% GPU, 75C) and monitor with tegrastats during sustained operation.

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| VLM generates generic or hallucinated defect descriptions | Insufficient or noisy fine-tuning data | Verify annotations manually; increase dataset to >2,000 annotated samples; add constrained decoding to restrict output to known defect vocabulary |
| ROUGE-L drops >5% after quantization | Aggressive quantization on sensitive layers | Use mixed precision: keep attention layers in FP16, quantize only FFN layers to INT8; re-calibrate with representative dataset |
| Segmentation model misses small defects | Input resolution too low after resize | Increase input resolution from 512x512 to 768x768; accept slightly higher latency; or use sliding window inference |
| TensorRT compilation fails on Jetson | CUDA/TensorRT version mismatch | Match exact versions: CUDA 11.8 + TensorRT 8.6 for JetPack 5.x; use NVIDIA's L4T containers for reproducibility |
| GPU out-of-memory during VLM inference | Dynamic tensor allocation exceeds budget | Reduce max_new_tokens from 256 to 128; lower beam count to 1 (greedy); enforce 6GB weight + 2GB dynamic budget |
| Segmentation F1 plateaus below 0.80 | Class imbalance (rare defects like holes: 87 samples) | Apply class-weighted loss with inverse frequency weighting; use aggressive augmentation (copy-paste) for rare classes |

## Limitations

- **VLM latency is the bottleneck**: Even with INT8+TensorRT, Phi-3.5-vision takes ~2.3s per summary. This pipeline is not suitable for frame-by-frame real-time video annotation — it works as a triggered summary system.
- **Domain-specific fine-tuning is required**: The VLM must be fine-tuned on domain-specific data with structured annotations. Zero-shot VLMs produce generic descriptions that lack the technical specificity needed for actionable maintenance reports.
- **Small dataset sensitivity**: The paper uses 5,051 images. With fewer than ~2,000 annotated samples, QLoRA fine-tuning will likely overfit. Data augmentation and careful validation splits become critical.
- **Hardware-locked optimization**: TensorRT engines are compiled for specific GPU architectures. An engine built for Jetson AGX Orin (Ampere) will not run on Jetson Nano (Maxwell/Pascal). You must recompile per target device.
- **Quantization-quality trade-off is non-trivial**: The <2% ROUGE-L degradation target requires careful calibration dataset selection. Poor calibration data can cause larger quality drops, especially for rare defect categories.
- **Not suitable for novel defect discovery**: The pipeline classifies and describes known defect types from its training taxonomy. Truly novel or compound defects outside the training distribution may be misclassified or described inaccurately.

## Reference

**Paper**: Lopez, J.J., Ferdaus, M.M., & Abdelguerfi, M. (2026). *Edge-Optimized Vision-Language Models for Underground Infrastructure Assessment*. arXiv:2602.03742v1. [https://arxiv.org/abs/2602.03742v1](https://arxiv.org/abs/2602.03742v1)

**Key takeaway**: The paper demonstrates that decoupling fast segmentation (0.64M params, <50ms) from triggered VLM summarization (3.8B params quantized to 2.1GB, 2.3s), combined with QLoRA fine-tuning and mixed-precision TensorRT quantization, enables practical end-to-end infrastructure inspection on edge devices with 3.1-second image-to-report latency.