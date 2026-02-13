---
name: "how-well-open-sourced"
description: |
  Select and deploy AI-generated image detection models based on threat-landscape analysis using zero-shot benchmark data from 23 detectors across 291 generators.
  Trigger phrases:
  - "detect AI-generated images"
  - "which deepfake detector should I use"
  - "benchmark image forensics models"
  - "deploy AI image detection pipeline"
  - "evaluate fake image detectors"
  - "set up content authenticity detection"
---

# AI-Generated Image Detection: Threat-Aware Detector Selection and Deployment

This skill enables Claude to guide practitioners through selecting, configuring, and deploying open-source AI-generated image detection models based on their specific threat landscape. Rather than recommending a single "best" detector, it applies findings from a comprehensive zero-shot benchmark of 23 pretrained detector variants across 12 datasets (2.6M images, 291 generators) to match detectors to deployment scenarios. The core insight: no universal winner exists, and detector selection must be driven by the generators you expect to encounter.

## When to Use

- When the user needs to detect AI-generated images in a content moderation pipeline and asks which model to deploy
- When evaluating multiple deepfake/AI-image detectors for a specific use case (social media, journalism, legal evidence)
- When building an ensemble detection system and needs to know which detectors are complementary
- When a deployed detector is failing on certain image types and the user needs to diagnose why
- When the user asks about detection accuracy expectations for specific generators (Midjourney, DALL-E, Flux, Stable Diffusion)
- When setting up a benchmarking pipeline to evaluate detectors on custom datasets before deployment
- When the user needs realistic accuracy expectations before committing to a detection approach

## Key Technique

The paper establishes that **training data alignment** is the single most important factor in detector performance -- more important than model architecture. Within architecturally identical detector families (e.g., AIDE with ResNet-50 or DRCT with CLIP-ViT), swapping only the training data causes 20-60% accuracy variance. This means selecting a detector trained on generators similar to your threat model matters far more than picking the most sophisticated architecture.

The benchmark identifies three systematic failure patterns that account for 1,075 total failures across all detector-dataset pairs: (1) **Training-test mismatch** (43% of failures) where detectors fail when test generators differ from training data, especially GAN-to-diffusion transitions; (2) **Universal evasion** (32%) where specific modern generators defeat nearly all detectors regardless of type -- Flux Dev (21% avg accuracy), Firefly v4 (18%), Midjourney v7 (24%); and (3) **Challenging dataset heterogeneity** (25%) where diverse multi-generator datasets consistently degrade all detectors.

The actionable framework is: profile your threat landscape first, then select detectors based on generator alignment rather than published leaderboard rankings. For unknown or mixed threats, ensemble methods (specifically Community-Forensics at 78% mean accuracy) provide the most stable performance. For known generator families, matched detectors dramatically outperform generalist approaches -- DRCT variants hit 96.4% on their matched Stable Diffusion versions.

## Step-by-Step Workflow

1. **Profile the threat landscape.** Identify the likely generators producing images in your domain. Categorize them: legacy GANs (ProGAN, StyleGAN), open-source diffusion (Stable Diffusion v1.4/v2.0/XL), or commercial APIs (Midjourney, DALL-E, Flux, Firefly). If unknown, treat as mixed.

2. **Set realistic accuracy expectations.** Use these baselines from the benchmark: legacy GANs 80-90%, Stable Diffusion variants 50-75%, commercial APIs (2024+) 18-35%, mixed/unknown 35-50%. Communicate these to stakeholders before choosing a detector.

3. **Select candidate detectors by threat category.**
   - Legacy deepfakes: AIDE_progan or any top-5 detector
   - Stable Diffusion content: DRCT variants matching the SD version (DRCT_clip_vit_sdv2 for SD v2.0, DRCT_clip_vit_sdv14 for SD v1.4)
   - Commercial/unknown generators: Community-Forensics (ensemble)
   - Mixed threat landscape: Community-Forensics + PatchCraft as primary pair

4. **Install and configure the selected detectors.** Clone the respective repositories, install dependencies, and load pretrained weights. Verify each detector runs on a known-real and known-fake image pair before proceeding.

5. **Build a representative validation set.** Collect 200-500 images that match your expected threat distribution: real images from your domain, and AI-generated images from the generators you identified in step 1. Balance the set 50/50 real/fake.

6. **Run zero-shot evaluation on your validation set.** Execute each candidate detector with the default 0.5 classification threshold. Record per-image predictions, then compute accuracy, AUC, TPR (true positive rate), and FPR (false positive rate).

7. **Analyze failure modes.** For each detector, examine the false negatives (missed fakes) and false positives (real images flagged). Check whether failures cluster around specific generators or image characteristics. This reveals which of the three failure patterns (training-test mismatch, universal evasion, dataset heterogeneity) affects your deployment.

8. **Optimize thresholds if needed.** The benchmark confirms AUC and accuracy correlate at r=0.82, but threshold tuning on your validation set can recover 5-15% accuracy. Lower the threshold (e.g., 0.3) to catch more fakes at the cost of more false positives, or raise it for high-precision applications.

9. **Design the ensemble (if warranted).** If no single detector meets requirements, combine 2-3 complementary detectors using majority voting or score averaging. Pair a CNN-based detector (PatchCraft) with a transformer-based one (DRCT or Effort) and an ensemble method (Community-Forensics) for architectural diversity.

10. **Deploy with monitoring and plan for decay.** Detection accuracy degrades as new generators emerge -- the benchmark shows accuracy dropping from ~79% on 2020-era generators to ~38% on 2024 models. Log prediction confidence scores, flag low-confidence predictions for human review, and re-evaluate quarterly against newly released generators.

## Concrete Examples

**Example 1: Social media platform content moderation**

User: "We need to detect AI-generated profile photos on our platform. Users upload photos from phones but we're also seeing AI-generated headshots. Which detector should we deploy?"

Approach:
1. Threat profile: likely generators are commercial (Midjourney, DALL-E) and open-source (SD, Flux) portrait generators
2. This is a mixed/commercial threat landscape -- set expectation at 35-50% accuracy on newest generators
3. Select Community-Forensics as primary detector (most stable across generators, 78% mean accuracy)
4. Add PatchCraft as secondary (67.5% mean, low variance CV=0.282)
5. Build validation set with platform-representative real photos and samples generated from Midjourney v6/v7, DALL-E 3, and SD XL
6. Deploy as ensemble with majority voting; flag low-confidence results for human review

Output:
```python
# detector_config.py
DETECTOR_PIPELINE = {
    "primary": {
        "name": "community-forensics",
        "repo": "https://github.com/RUB-SysSec/community-forensics",
        "expected_accuracy": {"mixed": 0.75, "commercial_2024": 0.42},
        "threshold": 0.5,
    },
    "secondary": {
        "name": "patchcraft",
        "repo": "https://github.com/PatchCraft/PatchCraft",
        "expected_accuracy": {"mixed": 0.67, "commercial_2024": 0.35},
        "threshold": 0.5,
    },
    "ensemble_strategy": "majority_vote",
    "low_confidence_threshold": 0.4,  # Route to human review below this
    "monitoring": {
        "log_confidence_scores": True,
        "alert_on_accuracy_drop": 0.10,  # Alert if weekly accuracy drops 10%+
        "reeval_cadence_days": 90,
    },
}
```

**Example 2: Verifying Stable Diffusion outputs specifically**

User: "We run a stock photo site and need to reject AI-generated submissions. Most fakes we catch manually are from Stable Diffusion. What's the best detection approach?"

Approach:
1. Threat profile: known generator family -- Stable Diffusion variants (v1.4, v1.5, v2.0, XL)
2. DRCT variants trained on matching SD versions achieve 96.4% on matched generators
3. Deploy DRCT_clip_vit_sdv2 and DRCT_clip_vit_sdv14 to cover both major SD lineages
4. Add Community-Forensics to catch non-SD generators that may also appear
5. Set high threshold (0.7) since false positives (rejecting real photos) are costly for a stock photo site

Output:
```python
# sd_detection_pipeline.py
DETECTORS = [
    {
        "name": "DRCT_clip_vit_sdv2",
        "target_generators": ["SD v2.0", "SD v2.1", "SD XL"],
        "expected_accuracy_matched": 0.96,
        "threshold": 0.7,
    },
    {
        "name": "DRCT_clip_vit_sdv14",
        "target_generators": ["SD v1.4", "SD v1.5"],
        "expected_accuracy_matched": 0.94,
        "threshold": 0.7,
    },
    {
        "name": "community-forensics",
        "target_generators": ["catch-all for non-SD"],
        "expected_accuracy_mixed": 0.75,
        "threshold": 0.6,
    },
]

ENSEMBLE_RULE = "flag_if_any"  # Conservative: flag if any detector triggers
HUMAN_REVIEW = True  # All flagged images go to manual review before rejection
```

**Example 3: Benchmarking detectors on a custom dataset**

User: "I have a dataset of 5000 images (2500 real, 2500 AI-generated from various sources). How do I benchmark open-source detectors against it?"

Approach:
1. Select the top-5 most stable detectors from the benchmark: Community-Forensics, SAFE, PatchCraft, DRCT_clip_vit_sdv2, Effort
2. Set up each detector with pretrained weights in isolated environments
3. Run inference on all 5000 images with each detector
4. Compute per-detector metrics and cross-detector agreement

Output:
```python
# benchmark_runner.py
import json
from pathlib import Path
from sklearn.metrics import accuracy_score, roc_auc_score, classification_report

DETECTORS = [
    "community-forensics",
    "safe-fatformer",
    "patchcraft",
    "drct-clip-vit-sdv2",
    "effort",
]

def run_benchmark(dataset_dir: Path, labels_file: Path):
    labels = json.loads(labels_file.read_text())
    results = {}

    for detector_name in DETECTORS:
        detector = load_detector(detector_name)  # Your loading logic
        predictions = {}
        scores = {}

        for img_path in dataset_dir.glob("*.png"):
            score = detector.predict(img_path)  # Returns float 0-1
            predictions[img_path.name] = int(score > 0.5)
            scores[img_path.name] = score

        y_true = [labels[k] for k in predictions]
        y_pred = list(predictions.values())
        y_scores = list(scores.values())

        results[detector_name] = {
            "accuracy": accuracy_score(y_true, y_pred),
            "auc": roc_auc_score(y_true, y_scores),
            "report": classification_report(y_true, y_pred, output_dict=True),
        }

    # Cross-detector agreement analysis
    agreement_matrix = compute_pairwise_agreement(results)
    # Low agreement between detectors suggests they catch different failure modes
    # -> good candidates for ensembling

    return results, agreement_matrix
```

## Best Practices

**Do:**
- Always validate detectors on images representative of your actual threat landscape before deploying. Published benchmark numbers will not match your domain.
- Use ensemble approaches (2-3 architecturally diverse detectors) for unknown or mixed generator threats. Community-Forensics alone outperforms most individual detectors.
- Match detectors to known generator families when possible. DRCT trained on SD v2.0 hits 96% on SD v2.0 content but may drop 30%+ on other generators.
- Log confidence scores in production and monitor for distribution shift, which signals new generator types appearing.

**Avoid:**
- Do not deploy a single detector and assume it covers all generators. The benchmark shows Spearman rank correlation as low as 0.01 between dataset pairs.
- Do not rely on frequency-domain detectors (FreDect, Fusing, UnivFD) as primary defenses -- they rank consistently in the bottom tier (37-49% mean accuracy).
- Do not expect any current open-source detector to reliably catch images from Flux Dev, Firefly v4, or Midjourney v7 (18-24% average accuracy). Plan for human review on these.
- Do not assume newer or more complex architectures automatically outperform older ones. Training data alignment matters more than model sophistication.

## Error Handling

- **Detector returns uniform predictions (all real or all fake):** The model likely has a distribution mismatch with your images. Check input preprocessing (resolution, normalization, color space). Many detectors expect specific input sizes (224x224 or 256x256) and ImageNet normalization.
- **Accuracy on validation set is far below published numbers:** This is expected for zero-shot deployment. The benchmark shows up to 40 percentage points between best-case and worst-case datasets for the same detector. Re-profile your threat landscape and try a different detector.
- **Ensemble members consistently disagree:** This can be informative -- high disagreement on specific images may indicate novel generator types not covered by any detector's training. Route these to human review.
- **Detection accuracy degrades over time:** New generators emerge regularly. The benchmark documents accuracy declining from ~79% (2020 generators) to ~38% (2024 generators). Schedule quarterly re-evaluation and plan to swap or retrain detectors.
- **High false positive rate on real images:** Some detectors (notably SAFE) have high variance (std=29.8%). Lower the classification threshold or add a second-stage verification step for flagged images.

## Limitations

- **Modern commercial generators are largely undetectable.** Flux Dev, Firefly v4, Midjourney v7, and Imagen 4 all achieve under 25% average detection accuracy across all 23 detector variants. No open-source zero-shot detector reliably handles these.
- **Zero-shot only.** The benchmark evaluates out-of-the-box performance. Fine-tuning detectors on target-domain data will improve results but requires labeled data and retraining infrastructure -- a different workflow than what this skill covers.
- **Threshold sensitivity.** All accuracy numbers assume a fixed 0.5 classification threshold. Real deployments need threshold optimization on held-out data, which changes the accuracy/false-positive tradeoff.
- **JPEG compression and social media processing.** The benchmark uses relatively clean images. Real-world images undergo compression, resizing, and format conversion that can degrade detection signals. Expect lower performance on images scraped from social media.
- **Rapid obsolescence.** The arms race between generators and detectors means these rankings have a shelf life. Detectors effective today may fail on generators released 6 months from now.

## Reference

**Paper:** Ren, S., Zhou, Y., Shen, X., Zewde, K., & Duong, T. (2026). "How well are open sourced AI-generated image detection models out-of-the-box: A comprehensive benchmark study." arXiv:2602.07814v1. [https://arxiv.org/abs/2602.07814v1](https://arxiv.org/abs/2602.07814v1)

**What to look for:** Tables 1-3 for full detector rankings across all 12 datasets; Section 5.1 for training data alignment analysis; Section 6.1 for the three failure patterns and their frequencies; Section 7 for deployment guidelines. The supplementary material contains per-generator accuracy breakdowns for all 291 generators.