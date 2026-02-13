---
name: "visual-cognitive-demands-model-powered"
description: "Evaluate visual and cognitive demands of in-vehicle LLM interfaces using the Monk et al. (2026) dual-metric framework. Implements DRT-based cognitive load estimation, NHTSA-compliant glance analysis, and multi-tier demand benchmarking for voice and visual HMI designs. Use when: 'evaluate cognitive load of my voice UI', 'analyze glance behavior for my car HMI', 'is my in-vehicle interface safe', 'benchmark my conversational agent against NHTSA thresholds', 'assess driver distraction for my app', 'run a visual-cognitive demand analysis on my interface design'."
---

# Visual & Cognitive Demand Evaluation for In-Vehicle LLM Interfaces

This skill enables Claude to apply the dual-metric evaluation framework from Monk et al. (2026) for assessing whether an in-vehicle conversational interface — particularly LLM-powered voice agents — meets established safety thresholds for driver distraction. The framework combines objective cognitive load measurement (via Detection Response Task proxies), NHTSA/Alliance-compliant visual attention analysis (glance durations, total eyes-off-road time), and subjective workload scoring to classify an interface along a validated demand spectrum from low-load baseline to high-load anchor.

## When to Use

- When the user is building or evaluating an in-vehicle voice assistant, HMI, or infotainment feature and needs to assess its distraction profile
- When designing a driving simulator study or user study to measure cognitive and visual demands of a new interface
- When the user asks to implement DRT (Detection Response Task) data collection and analysis in a research codebase
- When benchmarking a conversational agent's demand level against NHTSA or Alliance of Automobile Manufacturers safety guidelines
- When building telemetry or analytics pipelines that classify interaction sessions by cognitive load tier
- When the user needs to analyze eye-tracking data (glance durations, on-road vs. off-road fixations) for automotive safety compliance
- When writing evaluation code that determines whether an interface passes the 2-second glance rule or 12-second TEORT threshold

## Key Technique

The core innovation of Monk et al. is a **four-tier demand benchmarking framework** that positions any in-vehicle interaction on a validated cognitive load spectrum. Rather than measuring distraction in isolation, each interface is compared against two anchors: a **low-load baseline** (visual turn-by-turn navigation, DRT miss rate ~0.287) and a **high-load anchor** (OSPAN working memory task, DRT miss rate ~0.523). Hands-free phone calls (~0.431) serve as the established moderate-load reference. An interface that falls between the baseline and the phone-call tier is considered safe for deployment; one that approaches or exceeds OSPAN demands warrants redesign.

The framework measures three orthogonal demand dimensions simultaneously: (1) **Cognitive load** via DRT miss rates and response times — the DRT delivers a tactile stimulus every 3-5 seconds and measures whether the driver responds within 100ms-2500ms; higher miss rates signal greater cognitive capture. (2) **Visual demand** via eye-tracking glance metrics — mean glance duration (MGD) off-road must remain below the NHTSA 2-second threshold, total eyes-off-road time (TEORT) must stay below 12 seconds (NHTSA) or 20 seconds (Alliance), and at least 85% of participants must meet these criteria. (3) **Subjective workload** via modified NASA-TLX Likert scales across effort, mental demand, frustration, and perceived distraction dimensions.

A critical finding is the **on-road/off-road glance ratio analysis**: even when TEORT is elevated, if on-road mean glance durations exceed 6 seconds between off-road glances (sufficient for situation awareness rebuilding), the total TEORT figure is less consequential. This ratio-based interpretation prevents false positives when evaluating voice-primary interfaces that naturally produce brief, frequent off-road glances. Additionally, cognitive load was shown to remain **stable across extended multi-turn conversations**, meaning duration alone does not escalate demand.

## Step-by-Step Workflow

1. **Define the demand tier targets** for the interface under evaluation. Establish numeric thresholds: DRT miss rate < 0.45 for moderate acceptability (phone-call equivalent), MGD off-road < 2.0s (NHTSA), TEORT < 12s (NHTSA strict) or < 20s (Alliance permissive), and on-road MGD > 6.0s for situation awareness recovery.

2. **Implement DRT data collection** in your evaluation harness. Configure a tactile or visual stimulus with 1-second duration, random inter-stimulus interval of 2-4 seconds (total cycle 3-5s), and record hit/miss plus response time for each trial. Filter responses faster than 100ms (anticipatory) or slower than 2500ms (missed).

3. **Implement eye-tracking glance extraction** from your eye-tracker's raw data stream (e.g., Tobii Pro at 100Hz). Classify each fixation as on-road or off-road based on area-of-interest definitions. Remove glances shorter than 100ms as noise. Compute per-session: mean off-road glance duration, mean on-road glance duration, total eyes-off-road time, and the on-road/off-road MGD ratio.

4. **Implement subjective workload collection** using a post-task questionnaire. Use a 5-point Likert scale (1=Strongly Disagree, 5=Strongly Agree) across these dimensions: effort, frustration, mental demand, physical demand, temporal demand, performance satisfaction, visual distraction, road awareness, willingness to use while driving.

5. **Run the baseline and anchor conditions** first: have participants complete a visual turn-by-turn navigation task (low-load baseline) and an OSPAN working memory task (high-load anchor) so you have per-participant calibration data for the demand spectrum.

6. **Collect data for the target interface** under the same driving conditions. For voice agents, test both single-turn (one question/answer) and multi-turn (sustained conversation, 5+ exchanges) interaction patterns separately.

7. **Compute the demand classification** by comparing the target interface's DRT miss rate against the tier boundaries: < 0.30 = low demand (baseline-tier), 0.30-0.45 = moderate demand (phone-call-tier, acceptable), 0.45-0.52 = elevated demand (approaching OSPAN), > 0.52 = high demand (redesign required).

8. **Run NHTSA visual compliance checks**: verify (a) mean off-road glance duration < 2.0s, (b) TEORT < 12s for NHTSA or < 20s for Alliance, (c) at least 85% of participants meet the TEORT criterion. If TEORT fails but on-road MGD > 6.0s, flag as "TEORT elevated but situation awareness maintained" rather than outright failure.

9. **Perform multi-turn stability analysis** if the interface supports extended conversations. Regress DRT miss rate and response time against conversation turn number and elapsed time. A non-significant slope (p > 0.05) confirms demand stability; a significant positive slope indicates escalating load requiring mitigation.

10. **Generate the evaluation report** with a summary classification (pass/caution/fail), per-metric results tables, tier placement visualization, and specific recommendations keyed to any threshold violations.

## Concrete Examples

**Example 1: Evaluating a voice-based restaurant finder for an in-vehicle infotainment system**

User: "I have DRT and eye-tracking data from a 24-participant study comparing our new voice restaurant finder against a hands-free phone baseline. Can you help me analyze whether it meets NHTSA safety thresholds?"

Approach:
1. Parse the DRT log files, filtering responses < 100ms and > 2500ms
2. Compute per-condition DRT miss rates and mean response times
3. Parse eye-tracking exports, classify glances, remove sub-100ms fixations
4. Compute MGD (off-road), MGD (on-road), TEORT per participant per condition
5. Run linear mixed-effects models (participant as random effect) comparing conditions
6. Check each metric against the NHTSA thresholds

Output:
```
DEMAND EVALUATION REPORT — Voice Restaurant Finder v2.1
========================================================

COGNITIVE LOAD (DRT Miss Rate)
  Baseline (turn-by-turn):    0.291 (SE=0.041)  [LOW]
  Hands-free phone:           0.428 (SE=0.038)  [MODERATE]
  Restaurant finder:          0.402 (SE=0.035)  [MODERATE]  ✓ PASS
  OSPAN anchor:               0.519 (SE=0.039)  [HIGH]

  Classification: MODERATE — comparable to hands-free phone (p=.42, ns)

VISUAL DEMAND
  Off-road MGD:  0.68s (SE=0.04)   ✓ PASS  (threshold: < 2.0s)
  On-road MGD:   8.14s (SE=0.91)   ✓ Situation awareness maintained (> 6.0s)
  TEORT:         11.8s (SE=0.92)   ✓ PASS  (NHTSA < 12s)
  85% criterion: 87.5% below 12s   ✓ PASS

SUBJECTIVE (Median Likert, 1-5)
  Effort: 1.0 | Mental Demand: 2.0 | Distraction: 1.0 | Road Awareness: 4.0

OVERALL: ✓ PASS — Safe for deployment. Demand profile equivalent to hands-free calling.
```

**Example 2: Building a real-time cognitive load classifier for an adaptive voice agent**

User: "I want my in-car LLM agent to detect when the driver is under high cognitive load and simplify its responses. Help me implement the classification logic."

Approach:
1. Implement a rolling-window DRT miss rate calculator (window = last 10 stimuli)
2. Define tier thresholds from the Monk et al. calibration data
3. Build an adaptive response strategy that maps tiers to agent behavior

Output:
```python
from dataclasses import dataclass
from collections import deque
from enum import Enum

class DemandTier(Enum):
    LOW = "low"           # Turn-by-turn equivalent
    MODERATE = "moderate"  # Hands-free phone equivalent
    ELEVATED = "elevated"  # Approaching OSPAN
    HIGH = "high"          # At or above OSPAN

@dataclass
class DRTConfig:
    stimulus_duration_s: float = 1.0
    min_isi_s: float = 2.0
    max_isi_s: float = 4.0
    min_rt_s: float = 0.100    # Below this = anticipatory, discard
    max_rt_s: float = 2.500    # Above this = miss
    window_size: int = 10

class CognitiveLoadClassifier:
    """Classifies cognitive demand tier from DRT responses using
    Monk et al. (2026) calibrated thresholds."""

    TIER_BOUNDARIES = {
        DemandTier.LOW: (0.0, 0.30),       # Baseline: ~0.287
        DemandTier.MODERATE: (0.30, 0.45),  # Phone-call: ~0.431
        DemandTier.ELEVATED: (0.45, 0.52),  # Approaching OSPAN
        DemandTier.HIGH: (0.52, 1.0),       # OSPAN anchor: ~0.523
    }

    def __init__(self, config: DRTConfig = DRTConfig()):
        self.config = config
        self._responses: deque[bool] = deque(maxlen=config.window_size)

    def record_response(self, rt_seconds: float | None) -> None:
        """Record a DRT trial. rt_seconds=None means no response (miss)."""
        if rt_seconds is None or rt_seconds > self.config.max_rt_s:
            self._responses.append(False)  # Miss
        elif rt_seconds < self.config.min_rt_s:
            return  # Anticipatory — discard, do not count
        else:
            self._responses.append(True)   # Hit

    @property
    def miss_rate(self) -> float | None:
        if len(self._responses) < 3:
            return None
        misses = sum(1 for r in self._responses if not r)
        return misses / len(self._responses)

    @property
    def demand_tier(self) -> DemandTier | None:
        mr = self.miss_rate
        if mr is None:
            return None
        for tier, (low, high) in self.TIER_BOUNDARIES.items():
            if low <= mr < high:
                return tier
        return DemandTier.HIGH

    def get_agent_strategy(self) -> dict:
        """Return LLM agent behavior parameters for current demand tier."""
        strategies = {
            DemandTier.LOW: {
                "max_response_tokens": 200,
                "allow_follow_up_questions": True,
                "proactive_suggestions": True,
            },
            DemandTier.MODERATE: {
                "max_response_tokens": 100,
                "allow_follow_up_questions": True,
                "proactive_suggestions": False,
            },
            DemandTier.ELEVATED: {
                "max_response_tokens": 50,
                "allow_follow_up_questions": False,
                "proactive_suggestions": False,
            },
            DemandTier.HIGH: {
                "max_response_tokens": 25,
                "allow_follow_up_questions": False,
                "proactive_suggestions": False,
            },
        }
        tier = self.demand_tier
        return strategies.get(tier, strategies[DemandTier.HIGH])
```

**Example 3: Implementing NHTSA-compliant glance analysis from eye-tracker CSV data**

User: "I have Tobii Pro Glasses 3 CSV exports. Help me compute the NHTSA visual demand metrics."

Approach:
1. Parse the eye-tracker CSV with fixation classifications
2. Filter sub-100ms glances as noise
3. Compute MGD, TEORT, on-road/off-road ratio, and 85% compliance

Output:
```python
import pandas as pd
import numpy as np

@dataclass
class NHTSACompliance:
    mgd_off_road_s: float
    mgd_on_road_s: float
    teort_s: float
    pct_below_teort_12s: float
    sa_recovery_adequate: bool  # on-road MGD > 6.0s

    @property
    def mgd_pass(self) -> bool:
        return self.mgd_off_road_s < 2.0

    @property
    def teort_nhtsa_pass(self) -> bool:
        return self.teort_s < 12.0

    @property
    def teort_alliance_pass(self) -> bool:
        return self.teort_s < 20.0

    @property
    def eighty_five_pct_pass(self) -> bool:
        return self.pct_below_teort_12s >= 85.0

def analyze_glances(
    df: pd.DataFrame,
    participant_col: str = "participant_id",
    aoi_col: str = "aoi",         # "road" or "off_road"
    duration_col: str = "duration_s",
    min_glance_s: float = 0.100,
) -> dict[str, NHTSACompliance]:
    """Compute NHTSA visual demand metrics per participant."""
    # Filter noise glances
    df = df[df[duration_col] >= min_glance_s].copy()
    results = {}

    for pid, group in df.groupby(participant_col):
        off_road = group[group[aoi_col] == "off_road"][duration_col]
        on_road = group[group[aoi_col] == "road"][duration_col]

        mgd_off = off_road.mean() if len(off_road) > 0 else 0.0
        mgd_on = on_road.mean() if len(on_road) > 0 else 0.0
        teort = off_road.sum()

        results[pid] = NHTSACompliance(
            mgd_off_road_s=mgd_off,
            mgd_on_road_s=mgd_on,
            teort_s=teort,
            pct_below_teort_12s=0.0,  # Computed across participants below
            sa_recovery_adequate=mgd_on > 6.0,
        )

    # Compute 85% criterion across all participants
    teort_values = [r.teort_s for r in results.values()]
    pct_below = 100 * sum(1 for t in teort_values if t < 12.0) / len(teort_values)
    for r in results.values():
        r.pct_below_teort_12s = pct_below

    return results
```

## Best Practices

- **Do:** Always run both the low-load baseline (turn-by-turn navigation) and high-load anchor (OSPAN or equivalent) alongside your target condition. Without calibrated endpoints, absolute DRT numbers are uninterpretable across studies.
- **Do:** Analyze on-road/off-road glance ratios, not just TEORT alone. A voice interface with TEORT of 14s but on-road MGD of 9s is safer than one with TEORT of 10s but on-road MGD of 2s. The ratio captures situation awareness recovery.
- **Do:** Test multi-turn interactions separately from single-turn. Monk et al. showed demand remains stable over extended conversations, but this must be verified for each new agent — a poorly designed agent could show escalating load.
- **Do:** Use linear mixed-effects models with participant as a random intercept when analyzing DRT and glance data. Individual differences in baseline reaction time and glance behavior are substantial.
- **Avoid:** Using only subjective ratings to evaluate safety. Monk et al. found subjective and objective measures aligned, but subjective ratings alone lack the granularity to distinguish moderate from elevated demand tiers.
- **Avoid:** Treating the 2-second glance threshold or 12-second TEORT as binary pass/fail without context. These are guidelines within a multi-metric framework — the on-road recovery ratio and DRT convergent evidence must factor into the final assessment.

## Error Handling

- **Insufficient DRT trials**: If fewer than 30 DRT stimuli are collected per condition (roughly 2.5 minutes of data), confidence intervals on miss rates become unreliable. Flag results as provisional and recommend extending the testing window.
- **Eye-tracker signal loss**: If more than 20% of the recording is marked as lost signal or low confidence, exclude that participant from glance analysis rather than imputing values. Report the exclusion rate.
- **Anticipatory DRT responses**: A high rate of sub-100ms responses (> 15% of trials) suggests the participant is pattern-matching stimulus timing rather than genuinely responding. Exclude these participants from cognitive load analysis.
- **Ceiling effects on OSPAN**: If a participant shows DRT miss rates below 0.35 even during OSPAN, their high-load anchor is ineffective. Consider excluding them or using a within-participant normalization approach.
- **Unbalanced task durations**: Single-turn interactions are much shorter than multi-turn. Normalize TEORT by task duration when comparing these conditions, or restrict comparisons to matched time windows.

## Limitations

- The framework's tier boundaries (0.30, 0.45, 0.52) are calibrated from a 24-participant sample of US drivers aged 18-60. These thresholds may shift for different populations (elderly drivers, novice drivers, non-English speakers).
- DRT requires dedicated hardware (tactile transducer + microswitch) for the validated protocol. Software-only approximations (e.g., screen-based secondary tasks) measure a different construct and should not use these thresholds directly.
- The study evaluated Gemini Live specifically. Other LLM agents with different latency profiles, verbosity, or turn-taking patterns may produce different demand signatures even with similar content.
- All testing occurred on low-density suburban roads in Maryland. Highway driving, urban stop-and-go, and adverse weather conditions may shift baseline demands, altering the relative positions on the spectrum.
- The framework does not capture emotional or surprise-related distraction (e.g., an LLM producing unexpected or alarming content), which could produce transient demand spikes not reflected in session-level averages.

## Reference

Monk, C., Ayala, A., Yu, C. S. P., Fitch, G. M., & Gruber, D. (2026). Visual and Cognitive Demands of a Large Language Model-Powered In-vehicle Conversational Agent. [arXiv:2601.15034v1](https://arxiv.org/abs/2601.15034v1). Key sections: Table 3 (DRT miss rates by condition), Table 5 (glance metrics), Section 4.1 (demand tier classification), Section 4.3 (multi-turn stability analysis), and Section 4.4 (on-road/off-road glance ratio interpretation).