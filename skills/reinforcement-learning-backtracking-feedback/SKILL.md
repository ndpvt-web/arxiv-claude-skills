---
name: "reinforcement-learning-backtracking-feedback"
description: "Implement RLBF (Reinforcement Learning with Backtracking Feedback) for LLM safety — a framework where models learn to detect and correct their own safety violations mid-generation by emitting backtrack signals. Use when asked about: 'implement backtracking safety for LLM', 'add self-correcting generation to a language model', 'build an RL safety training loop with critic feedback', 'create a backtrack token mechanism for safe generation', 'design BSAFE+ training data with injected violations', 'reduce attack success rate while preserving model utility'."
---

# Reinforcement Learning with Backtracking Feedback (RLBF)

This skill enables Claude to help implement the RLBF framework from Sel et al. (NeurIPS 2025), where LLMs learn to detect safety violations in their own output mid-generation and correct them by emitting a lightweight "backtrack by X tokens" signal followed by a safe continuation. The approach combines a critic-guided RL training loop (using GRPO) with an enhanced SFT data generation strategy (BSAFE+) that injects violations into originally safe text, teaching models precisely where and how to backtrack. The result is a model that defends against GCG attacks, middle-filling attacks, and decoding parameter manipulation while preserving nearly all baseline utility on reasoning and knowledge benchmarks.

## When to Use

- When the user wants to add a **self-correcting safety mechanism** to an LLM's generation pipeline — one that detects and removes harmful tokens on-the-fly rather than refusing entire responses
- When building a **critic-based reward system** for RL safety training and the user needs a concrete reward structure that balances safety enforcement with utility preservation
- When the user asks to **generate SFT training data** for backtracking by injecting realistic violations into safe completions (the BSAFE+ strategy)
- When implementing **defenses against adversarial attacks** on LLMs (GCG suffix attacks, middle-filling prompt injections, temperature/top-p manipulation)
- When designing a **post-processing layer** that interprets backtrack signals from a model and cleanly removes flagged token spans from the output stream
- When the user needs to **combine RL and SFT objectives** in a single training loop using GRPO with an SFT guidance term to prevent catastrophic forgetting

## Key Technique

**The Backtrack Signal.** RLBF augments a model's vocabulary with two special token families: `[CATEGORY_c]` (identifying the violation type — toxicity, harmful advice, etc.) and `[BACKTRACK_BY_X]` (specifying how many preceding tokens to erase). When the model generates content that violates a safety policy, it emits these signals inline. A post-processing step strips the flagged tokens and the signal tokens themselves, then lets generation continue autoregressively. The model does not revert its hidden state — the tokens act purely as stream-editing instructions, making this approach efficient and compatible with standard autoregressive decoding.

**Critic-Driven RL with GRPO.** A separate LLM-based safety critic evaluates the model's live rollouts, identifying violation spans and their categories. This feeds into a trajectory-level reward: safe output without backtracking scores +1.0, a missed violation scores -1.0, an unnecessary backtrack scores -0.5, and a correct backtrack followed by a safe continuation scores +1.0. Training uses Group Relative Policy Optimization (GRPO) with the combined loss `L_total(θ) = -J_RL(θ) + λ_SFT * L_SFT_guidance(θ)`, where the SFT guidance term anchors the model to correct backtracking patterns learned during the supervised phase and prevents forgetting.

**BSAFE+ Data Generation.** Instead of prompting a model to produce unsafe text from scratch (which yields out-of-distribution artifacts), BSAFE+ starts with high-quality safe responses and injects harmful segments at contextually coherent insertion points. This produces training pairs where the violation location and backtrack distance are precisely known, giving the SFT stage clean supervision for learning the backtracking behavior before RL refines it under adversarial pressure.

## Step-by-Step Workflow

1. **Define the safety taxonomy.** Enumerate the violation categories your system must handle (e.g., toxicity, dangerous instructions, PII leakage). Assign each a `[CATEGORY_c]` token. Define the `[BACKTRACK_BY_X]` token family for X in a reasonable range (e.g., 1–128 tokens).

2. **Prepare the safe base corpus.** Collect or generate a set of high-quality, fully safe instruction–response pairs. These must be coherent, on-topic, and representative of the model's intended use cases — they serve as the substrate for BSAFE+ violation injection.

3. **Inject violations to create BSAFE+ SFT data.** For each safe response, use a separate LLM (or rule-based method) to insert a harmful segment at a contextually plausible location. Record the exact token offset and span length of the injected violation. Construct the training target as: `[safe prefix] [violation tokens] [CATEGORY_c] [BACKTRACK_BY_X] [safe continuation]`, where X equals the number of violation tokens.

4. **Run supervised fine-tuning on BSAFE+ data.** Fine-tune the base model on the annotated sequences so it learns the mechanics of emitting backtrack signals at the right positions. Mix in standard instruction-following data (without backtracking) at a ratio that preserves general capability — typically 50–70% standard data.

5. **Build the safety critic.** Deploy an LLM-based critic (can be the same architecture, separately trained, or a strong off-the-shelf safety classifier) that takes a generated sequence and returns: (a) whether a violation exists, (b) which category, and (c) the token span of the violation. Wrap this as a reward function.

6. **Implement the trajectory-level reward function.** Code the reward signal as follows:
   - No backtrack emitted, output is safe: **+1.0**
   - No backtrack emitted, violation detected: **-1.0**
   - Unnecessary backtrack (no violation present): **-0.5**
   - Correct backtrack followed by safe continuation: **+1.0**
   - Correct backtrack but continuation is unsafe or incoherent: **-0.2**

7. **Train with GRPO and SFT guidance.** Run the RL loop using GRPO on the SFT-initialized model. For each batch, generate rollouts, score them with the critic, compute the combined loss `L_total = -J_RL + λ_SFT * L_SFT_guidance`, and update. Start with `λ_SFT` around 0.1–0.3 and decay it as the RL signal stabilizes.

8. **Implement the post-processing decoder.** Write a stream processor that monitors generated tokens for `[CATEGORY_c]` and `[BACKTRACK_BY_X]` signals. When detected, remove the preceding X tokens and the signal tokens from the output buffer, then continue streaming the subsequent tokens to the user.

9. **Evaluate against adversarial benchmarks.** Test the trained model against GCG suffix attacks, middle-filling injections, and decoding parameter manipulations (high temperature, modified top-p). Measure attack success rate (target: <5%) alongside utility benchmarks (MMLU, BBH, GSM8K) to confirm capability is preserved.

10. **Iterate on the critic and reward calibration.** If the model over-backtracks (utility loss) or under-backtracks (safety gaps), adjust the reward weights — particularly the -0.5 penalty for unnecessary backtracks and the -0.2 for failed corrections. Re-run RL for additional epochs as needed.

## Concrete Examples

**Example 1: Implementing the backtrack token post-processor**

User: "I need a Python module that processes a token stream and handles RLBF backtrack signals."

Approach:
1. Define the special token patterns for category and backtrack signals
2. Build a streaming buffer that accumulates tokens
3. Detect backtrack signals and splice out the flagged span
4. Continue outputting the cleaned stream

Output:
```python
import re
from dataclasses import dataclass

@dataclass
class BacktrackSignal:
    category: str
    num_tokens: int

class RLBFStreamProcessor:
    """Processes an LLM token stream, removing spans flagged by backtrack signals."""

    CATEGORY_PATTERN = re.compile(r"\[CATEGORY_(\w+)\]")
    BACKTRACK_PATTERN = re.compile(r"\[BACKTRACK_BY_(\d+)\]")

    def __init__(self):
        self.buffer: list[str] = []
        self.pending_category: str | None = None
        self.backtrack_log: list[BacktrackSignal] = []

    def feed(self, token: str) -> list[str]:
        """Feed a token into the processor. Returns tokens to emit (may be empty)."""
        cat_match = self.CATEGORY_PATTERN.fullmatch(token)
        if cat_match:
            self.pending_category = cat_match.group(1)
            return []

        bt_match = self.BACKTRACK_PATTERN.fullmatch(token)
        if bt_match and self.pending_category is not None:
            num_tokens = int(bt_match.group(1))
            signal = BacktrackSignal(self.pending_category, num_tokens)
            self.backtrack_log.append(signal)
            # Remove the last num_tokens from the buffer
            removed = min(num_tokens, len(self.buffer))
            self.buffer = self.buffer[:-removed] if removed > 0 else self.buffer
            self.pending_category = None
            return []

        # Normal token — if a category was pending without a backtrack, flush it
        if self.pending_category is not None:
            self.pending_category = None

        self.buffer.append(token)
        return [token]

    def get_clean_output(self) -> str:
        return "".join(self.buffer)
```

**Example 2: Building the BSAFE+ violation injection pipeline**

User: "Help me create a data generation script that injects safety violations into safe responses for RLBF SFT training."

Approach:
1. Load safe instruction-response pairs
2. Use an LLM to identify plausible injection points and generate violations
3. Construct annotated training sequences with backtrack tokens
4. Validate that each sample has the correct backtrack distance

Output:
```python
import json
import random
from transformers import AutoTokenizer

def inject_violation(
    tokenizer: AutoTokenizer,
    safe_response: str,
    violation_text: str,
    category: str,
) -> dict:
    """Inject a violation into a safe response and annotate with backtrack tokens."""
    safe_tokens = tokenizer.encode(safe_response, add_special_tokens=False)
    violation_tokens = tokenizer.encode(violation_text, add_special_tokens=False)

    # Choose an injection point in the middle 60% of the response
    min_pos = len(safe_tokens) // 5
    max_pos = len(safe_tokens) * 4 // 5
    inject_pos = random.randint(min_pos, max_pos)

    backtrack_distance = len(violation_tokens)
    category_token = f"[CATEGORY_{category}]"
    backtrack_token = f"[BACKTRACK_BY_{backtrack_distance}]"

    # Build annotated sequence:
    # [safe prefix tokens] [violation tokens] [CATEGORY] [BACKTRACK] [safe suffix tokens]
    prefix = tokenizer.decode(safe_tokens[:inject_pos])
    suffix = tokenizer.decode(safe_tokens[inject_pos:])

    annotated = f"{prefix}{violation_text}{category_token}{backtrack_token}{suffix}"

    return {
        "annotated_response": annotated,
        "category": category,
        "backtrack_distance": backtrack_distance,
        "injection_position": inject_pos,
        "original_safe_response": safe_response,
    }


def build_sft_dataset(
    safe_pairs: list[dict],
    violation_bank: list[dict],
    tokenizer: AutoTokenizer,
    injection_ratio: float = 0.5,
) -> list[dict]:
    """Build a mixed SFT dataset: some samples with injected violations, rest clean."""
    dataset = []
    for pair in safe_pairs:
        if random.random() < injection_ratio:
            violation = random.choice(violation_bank)
            sample = inject_violation(
                tokenizer,
                pair["response"],
                violation["text"],
                violation["category"],
            )
            sample["instruction"] = pair["instruction"]
            sample["has_backtrack"] = True
        else:
            sample = {
                "instruction": pair["instruction"],
                "annotated_response": pair["response"],
                "has_backtrack": False,
            }
        dataset.append(sample)
    return dataset
```

**Example 3: Configuring the GRPO reward function**

User: "I need to implement the RLBF reward function for use with a GRPO training loop."

Approach:
1. Define the reward structure from the paper
2. Use the critic to classify each rollout trajectory
3. Return scalar rewards per trajectory

Output:
```python
from enum import Enum

class TrajectoryOutcome(Enum):
    SAFE_NO_BACKTRACK = "safe_no_backtrack"
    VIOLATION_MISSED = "violation_missed"
    UNNECESSARY_BACKTRACK = "unnecessary_backtrack"
    CORRECT_BACKTRACK_SAFE = "correct_backtrack_safe"
    CORRECT_BACKTRACK_UNSAFE = "correct_backtrack_unsafe"

REWARD_TABLE = {
    TrajectoryOutcome.SAFE_NO_BACKTRACK: 1.0,
    TrajectoryOutcome.VIOLATION_MISSED: -1.0,
    TrajectoryOutcome.UNNECESSARY_BACKTRACK: -0.5,
    TrajectoryOutcome.CORRECT_BACKTRACK_SAFE: 1.0,
    TrajectoryOutcome.CORRECT_BACKTRACK_UNSAFE: -0.2,
}

def compute_rlbf_reward(
    generated_tokens: list[str],
    critic_result: dict,
    post_backtrack_tokens: list[str] | None,
) -> float:
    """
    Compute the RLBF trajectory-level reward.

    Args:
        generated_tokens: Raw token sequence from the model
        critic_result: Dict with keys 'has_violation' (bool),
                       'violation_span' (tuple | None)
        post_backtrack_tokens: Tokens after backtrack processing (None if no
                               backtrack was emitted)
    """
    model_emitted_backtrack = any("[BACKTRACK_BY_" in t for t in generated_tokens)
    has_violation = critic_result["has_violation"]

    if not model_emitted_backtrack and not has_violation:
        return REWARD_TABLE[TrajectoryOutcome.SAFE_NO_BACKTRACK]
    if not model_emitted_backtrack and has_violation:
        return REWARD_TABLE[TrajectoryOutcome.VIOLATION_MISSED]
    if model_emitted_backtrack and not has_violation:
        return REWARD_TABLE[TrajectoryOutcome.UNNECESSARY_BACKTRACK]

    # Model backtracked and there was a real violation — check the result
    if post_backtrack_tokens is not None:
        post_text = "".join(post_backtrack_tokens)
        # Re-run critic on post-backtrack output
        post_critic = critic_result.get("post_backtrack_safe", True)
        if post_critic:
            return REWARD_TABLE[TrajectoryOutcome.CORRECT_BACKTRACK_SAFE]

    return REWARD_TABLE[TrajectoryOutcome.CORRECT_BACKTRACK_UNSAFE]
```

## Best Practices

- **Do:** Keep the backtrack token vocabulary small and structured. Use a fixed set of `[CATEGORY_c]` tokens matching your safety taxonomy and cap `[BACKTRACK_BY_X]` at a reasonable maximum (64–128 tokens). Larger ranges waste vocabulary space with no benefit.
- **Do:** Mix standard instruction-following data into SFT at 50–70% ratio. The model must see plenty of normal generation without backtracking so it does not learn to over-trigger the mechanism on benign inputs.
- **Do:** Use the asymmetric reward structure exactly as specified — the -0.5 penalty for unnecessary backtracks is critical for preventing over-cautious models that backtrack on safe content.
- **Do:** Validate that injected violations in BSAFE+ data land at contextually coherent positions. Random insertion produces artifacts the model learns to detect superficially rather than learning genuine safety patterns.
- **Avoid:** Training the critic and the policy model on the same data splits. The critic must generalize to the policy's live rollouts, so use held-out data for critic evaluation.
- **Avoid:** Setting `λ_SFT` too high in the combined loss. If the SFT guidance term dominates, the model will memorize backtracking patterns from the SFT data rather than learning to generalize through RL exploration.

## Error Handling

- **Backtrack distance exceeds buffer length:** Cap the removal at the current buffer size. Log a warning — this indicates the critic or training data has misaligned span annotations.
- **Orphaned category token (no backtrack follows):** If a `[CATEGORY_c]` token appears without a subsequent `[BACKTRACK_BY_X]`, discard the category token silently. Do not remove any preceding tokens.
- **Critic disagrees with model on violation presence:** During RL training, always trust the critic's assessment for reward computation. Log disagreements for later analysis — a high disagreement rate suggests the critic or the policy needs recalibration.
- **Model utility drops during RL training:** Increase `λ_SFT` temporarily or reduce the learning rate. Monitor MMLU/BBH scores every few hundred steps and stop RL if utility degrades more than 1–2 points.
- **Over-backtracking on benign inputs:** The -0.5 unnecessary backtrack penalty may need strengthening (e.g., -0.7) if the model develops a conservative bias. Check the false-positive backtrack rate on a clean eval set.

## Limitations

- **Requires a reliable safety critic.** The entire RL loop depends on the critic correctly identifying violations and their spans. A weak or biased critic produces noisy rewards that degrade training. Building or selecting an adequate critic is a substantial prerequisite.
- **Token-level backtracking is approximate.** The mechanism removes a fixed number of tokens, which may not align perfectly with semantic boundaries. Partial word removal or mid-sentence cuts can produce artifacts that require further generation to repair.
- **Does not prevent the model from generating harmful content internally.** The hidden states still encode the violating content — backtracking only removes it from the output stream. This is a mitigation, not a prevention.
- **Vocabulary expansion is non-trivial.** Adding special tokens to a pretrained model's vocabulary requires embedding layer resizing and careful initialization, which can cause instability early in training.
- **Adversarial arms race.** Sophisticated attackers who know the backtracking mechanism exists may craft inputs that cause the model to backtrack beneficial content or overwhelm the mechanism with rapid-fire violations.

## Reference

**Paper:** Sel, B., Keshava, V., Wallis, P., Rutishauser, L., & Jin, M. (2026). *Reinforcement Learning with Backtracking Feedback.* NeurIPS 2025. [arXiv:2602.08377v1](https://arxiv.org/abs/2602.08377v1)

Key sections to study: the BSAFE+ data generation algorithm (Section 3), the GRPO + SFT guidance combined loss formulation (Section 4), and the adversarial evaluation results showing <5% attack success rates across GCG, middle-filling, and decoding manipulation attacks (Section 5).