---
name: "physprover-advancing-automatic-theorem"
description: "Build formal theorem proving pipelines for physics and scientific domains using conjecture-based data generation, Lean 4 verification, and RLVR training. Use when: 'formalize a physics theorem in Lean', 'generate training data for a theorem prover', 'build a conjecture pipeline with verification', 'set up RLVR training for formal reasoning', 'create a Lean 4 verification harness', 'extend a math prover to physics domains'."
---

# PhysProver: Formal Theorem Proving for Physics and Scientific Domains

This skill enables Claude to help users build formal theorem-proving systems that extend beyond pure mathematics into physics and other scientific domains. The core technique, from the PhysProver paper, combines a conjecture-based data generation pipeline (seed theorems -> LLM-generated conjectures -> syntax validation -> provability verification) with Reinforcement Learning with Verifiable Rewards (RLVR) using GRPO to train provers that leverage domain-specific context. The method achieves meaningful gains with as few as ~5K training samples by using Lean 4 as a binary verifier for reward signals.

## When to Use

- When the user wants to formalize physics equations, laws, or derivations in Lean 4 and needs guidance on structuring statements with proper headers, imports, and namespace dependencies
- When the user is building a data generation pipeline that produces formally verified conjectures from a seed corpus of theorems (in any scientific domain, not just physics)
- When the user wants to fine-tune or RLVR-train an existing math theorem prover (e.g., DeepSeek-Prover-V2) on a new domain
- When the user needs to set up a Lean 4 verification loop as a reward signal for reinforcement learning
- When the user is designing a curriculum learning strategy that orders training samples by difficulty (proof length)
- When the user asks about filtering synthetic formal data through multi-stage validation (syntax check, then provability check with multiple provers)

## Key Technique

**Conjecture-Based Formal Data Generation.** The central insight is that formal theorem provers trained on mathematics can be extended to physics (or other domains) by generating a small, high-quality dataset of domain-specific conjectures and training with verifiable rewards. The pipeline works in three stages: (1) Use an LLM to generate candidate conjectures from seed theorem-header pairs, producing ~10 conjectures per seed; (2) Filter through Lean 4 syntax validation, retaining only well-formed statements with proper variable definitions (~24% survival rate); (3) Verify provability by running multiple provers (each attempting 16 proof attempts), keeping only conjectures where at least one prover succeeds (~9% overall pipeline yield). This aggressive filtering ensures every training sample is both syntactically valid and provably correct.

**RLVR with GRPO Instead of SFT.** A key finding is that supervised fine-tuning (SFT) on the generated data actually degrades performance (-6.4%), while Group Relative Policy Optimization (GRPO) with binary Lean verification rewards improves it. GRPO samples a group of G proof attempts per theorem, scores each 0 or 1 based on Lean verification, and uses the group-relative advantage to update the policy. Proofs containing escape hatches (`sorry`, `admit`, `apply?`) are assigned zero reward. Curriculum learning orders training statements by proof length (short proofs first), enabling stable easy-to-hard progression.

**Domain-Specific Context Leverage.** Physics theorems differ from pure math in that they require rich contextual headers (imports, namespaces, domain-specific definitions). The trained model learns to exploit these headers to apply the right lemmas -- for example, correctly using `timeContract_eq_superCommute` in quantum field theory proofs instead of hallucinating non-existent lemmas. This context-awareness is the mechanism behind cross-domain generalization.

## Step-by-Step Workflow

### Building a Conjecture-Based Data Generation Pipeline

1. **Curate a seed corpus from the target domain's Lean library.** Extract provable lemmas (those with complete proofs) from the domain repository (e.g., PhysLean for physics). Filter to statements under 4,096 tokens. Retain both the statement and its header (imports, open namespaces, preceding definitions) as a pair.

2. **Generate candidate conjectures with an LLM.** For each seed header-lemma pair, prompt an LLM to produce 10 novel conjectures that are structurally similar but explore variations (different variable bindings, stronger/weaker conditions, related properties). Include the full header context in the prompt so generated conjectures reference real definitions.

3. **Run Lean 4 syntax validation on every candidate.** Compile each conjecture (with its header) through Lean 4 to confirm it parses correctly, all referenced types and functions exist, and variable definitions are well-formed. Expect roughly 24% survival. Discard the rest -- do not attempt to repair syntax errors at this stage.

4. **Verify provability using an ensemble of provers.** Run 2-3 different open-source theorem provers (e.g., DeepSeek-Prover-V2-7B, Kimina-Prover, Goedel-Prover), each generating 16 proof attempts per conjecture. A conjecture passes if any single attempt from any prover is verified by Lean 4. Expect roughly 37% of syntax-valid conjectures to be provable (~9% of initial candidates).

5. **Combine seed theorems with verified conjectures.** Merge the original seed corpus (~60% of final dataset) with the generated-and-verified conjectures (~40%) to form the training set. A total of ~5K samples is sufficient for meaningful gains.

6. **Order training data by proof length for curriculum learning.** Sort all training samples by the token length of their shortest verified proof. Present shorter (easier) proofs first during training. This stabilizes GRPO training and improves convergence.

7. **Configure GRPO training with Lean verification as the reward.** Set up the RL training loop: for each theorem statement, sample a group of G proof candidates from the model, verify each with Lean 4, assign reward 1 (verified) or 0 (failed). Zero out rewards for proofs containing `sorry`, `admit`, or `apply?`. Use learning rate 1e-6, batch size 256, and train for 2 epochs.

8. **Integrate a Lean 4 verifier into the training loop.** Use a framework like `verl` with Lean 4 (version 4.20.0+) as the verification backend. The verifier must run in-loop to provide reward signals during GRPO rollouts. Ensure the verifier has access to all domain library dependencies (e.g., Mathlib, PhysLean).

9. **Evaluate on held-out domain test set and out-of-distribution benchmarks.** Test on domain-specific theorems (held out from seed corpus) with pass@k metrics. Also evaluate on general math benchmarks (e.g., MiniF2F-Test) to measure cross-domain generalization. Positive transfer to math indicates the model learned generalizable reasoning patterns.

10. **Iterate: use newly proved theorems as seeds for the next generation round.** Feed successfully proved conjectures back into step 2 as additional seeds, expanding the corpus. Each iteration increases domain coverage and proof difficulty.

## Concrete Examples

**Example 1: Setting Up a Conjecture Generation Prompt**

User: "I have a Lean 4 library of classical mechanics theorems. How do I generate training conjectures from them?"

Approach:
1. Extract a seed theorem with its full header context
2. Design a structured prompt that produces syntactically plausible conjectures
3. Filter through Lean 4 compilation

```lean
-- SEED INPUT (header + lemma pair)
import PhysLean.ClassicalMechanics.Energy
open ClassicalMechanics Energy

lemma kinetic_energy_nonneg (m : ℝ) (v : ℝ) (hm : 0 ≤ m) :
    0 ≤ (1/2) * m * v^2 := by
  apply mul_nonneg
  · apply mul_nonneg <;> linarith
  · exact sq_nonneg v
```

```text
-- LLM PROMPT FOR CONJECTURE GENERATION
Given this Lean 4 header and lemma from classical mechanics:
[insert header + lemma above]

Generate 10 novel conjectures that:
- Use the same imports and namespaces
- Reference existing definitions (Energy, kinetic_energy, etc.)
- Vary conditions (e.g., strict positivity, specific velocity values,
  relationships between kinetic and potential energy)
- Are plausible formal statements (not trivially true or false)

Output each conjecture as a complete Lean 4 `lemma` statement with
its type signature but WITHOUT a proof (end with `:= by sorry`).
```

```lean
-- EXAMPLE GENERATED CONJECTURE (after syntax validation passes)
lemma kinetic_energy_zero_iff (m : ℝ) (v : ℝ) (hm : 0 < m) :
    (1/2) * m * v^2 = 0 ↔ v = 0 := by sorry

-- This conjecture would then go to provability verification (step 4)
```

**Example 2: Configuring GRPO Training with Lean Verification**

User: "I have 5K verified physics theorem-proof pairs. How do I set up RLVR training?"

Approach:
1. Sort dataset by proof length for curriculum ordering
2. Configure GRPO with binary Lean rewards
3. Set up the training run

```python
# Training configuration for GRPO with Lean 4 verification
grpo_config = {
    "base_model": "deepseek-ai/DeepSeek-Prover-V2-7B",
    "training_data": "physlean_data_5541_samples.jsonl",

    # GRPO hyperparameters
    "group_size": 16,           # G proof attempts per theorem
    "learning_rate": 1e-6,
    "batch_size": 256,
    "num_epochs": 2,
    "clip_ratio": 0.2,         # PPO-style clipping for GRPO

    # Curriculum learning: sort by proof token length ascending
    "curriculum": {
        "enabled": True,
        "sort_key": "proof_token_length",
        "order": "ascending"    # easy (short proofs) first
    },

    # Lean 4 verification reward
    "reward": {
        "type": "lean4_binary",
        "lean_version": "4.20.0",
        "project_deps": ["Mathlib", "PhysLean"],
        "banned_tactics": ["sorry", "admit", "apply?"],
        "reward_verified": 1.0,
        "reward_failed": 0.0,
        "reward_banned_tactic": 0.0  # zero reward even if Lean accepts
    },

    # Hardware
    "gpus": 8,
    "framework": "verl"
}
```

```text
Output expectations after training (~8 hours on 8xH200):
- Physics domain accuracy: ~2-4% improvement over base model
- MiniF2F-Test (math): ~1-2% improvement (cross-domain transfer)
- Key diagnostic: check that model uses domain-specific lemmas from
  headers rather than hallucinating non-existent ones
```

**Example 3: Building a Multi-Stage Verification Filter**

User: "I generated 30K candidate conjectures. How do I filter them efficiently?"

Approach:
1. Stage 1: Lean syntax check (fast, parallel)
2. Stage 2: Multi-prover provability verification (expensive, batched)

```bash
# Stage 1: Syntax validation (parallel, ~minutes)
# Compile each conjecture with its header through Lean 4
for conjecture in conjectures/*.lean; do
    lean --run "$conjecture" 2>/dev/null && echo "$conjecture" >> valid.txt
done
# Expected: ~7,200 / 30,000 pass (24%)

# Stage 2: Provability verification (batched GPU inference)
# Run each valid conjecture through 3 provers x 16 attempts
python run_provers.py \
    --input valid.txt \
    --provers deepseek-prover-v2-7b,kimina-prover-8b,goedel-prover-v2-8b \
    --attempts-per-prover 16 \
    --lean-verify \
    --output verified.jsonl
# Expected: ~2,600 / 7,200 pass (37% of valid, 9% of total)
```

```json
// verified.jsonl entry format
{
  "header": "import PhysLean.QFT.NormalOrder\nopen QFT ...",
  "statement": "lemma normalOrder_timeContract (φ ψ : ℱ.FieldOp) : 𝒩(timeContract φ ψ) = 0",
  "proof": "by simp [normalOrder, timeContract_eq_superCommute]",
  "proof_token_length": 42,
  "proved_by": "deepseek-prover-v2-7b",
  "domain": "quantum_field_theory"
}
```

## Best Practices

- **Do:** Include the full Lean 4 header (imports, open namespaces, preceding definitions) with every theorem statement. Physics theorems depend heavily on context that math theorems often do not need. The header is what allows the prover to find and apply domain-specific lemmas.
- **Do:** Use multiple diverse provers during the provability verification stage. A conjecture that one prover cannot solve may be trivial for another. The ensemble approach maximizes yield from expensive conjecture generation.
- **Do:** Apply curriculum learning ordered by proof length. GRPO training is sensitive to difficulty distribution, and starting with shorter proofs provides stable gradient signals before tackling harder theorems.
- **Avoid:** Supervised fine-tuning (SFT) as the sole training method. The PhysProver experiments show SFT on formal physics data degrades performance by 6.4%. RLVR with GRPO is the correct approach for this domain.
- **Avoid:** Accepting proofs that use `sorry`, `admit`, or `apply?`. These are escape hatches that bypass verification. Always filter them out and assign zero reward during training, even if Lean technically compiles the file.
- **Avoid:** Generating conjectures without header context. Conjectures produced without knowledge of available definitions and imports will reference non-existent types and functions, causing near-total failure at the syntax validation stage.

## Error Handling

- **Low syntax validation rate (<15%):** The LLM generating conjectures is not grounding in the header context. Increase the amount of header included in the prompt, or add few-shot examples of valid conjectures from the same library.
- **Low provability rate (<20% of syntax-valid):** The generated conjectures may be too difficult or too loosely related to the seed theorems. Constrain generation to closer variations (e.g., "modify one hypothesis" rather than "invent a new property"). Alternatively, add a stronger prover to the ensemble.
- **GRPO training divergence:** Likely caused by too many hard samples early in training. Verify curriculum ordering is active and sorted ascending by proof length. Also check that batch size is large enough (256+) for stable advantage estimation.
- **Model hallucinates non-existent lemmas:** The base model is falling back to mathematical habits. Ensure training data includes the full header context and that the model sees diverse examples from the target domain. This is the specific failure mode that RLVR on domain data corrects.
- **Cross-domain performance drops on specific math categories:** RLVR on physics data may cause regression on some narrow math subcategories (e.g., AIME-style competition problems) while improving others. Monitor per-category metrics and consider mixing a small proportion of math training data to stabilize.

## Limitations

- The approach requires an existing formalized library in the target domain (e.g., PhysLean for physics). Domains without a Lean 4 library of definitions and basic lemmas cannot use this pipeline directly -- the library must be built first.
- The conjecture generation pipeline has a ~9% overall yield, meaning substantial compute is spent generating conjectures that are ultimately discarded. This is acceptable at 30K candidates but may be costly at larger scales.
- Theorem proving accuracy remains modest in complex physics subdomains (e.g., ~27% on quantum field theory), meaning human guidance is still needed for hard proofs.
- The method is validated on Lean 4 only. Adaptation to other proof assistants (Coq, Isabelle) would require re-engineering the verification pipeline.
- Training improvements are meaningful but incremental (~2.4% overall). This is a paradigm for extending formal provers to new domains, not a method that solves physics theorem proving outright.

## Reference

- **Paper:** [PhysProver: Advancing Automatic Theorem Proving for Physics](https://arxiv.org/abs/2601.15737v1) -- Read sections 3 (data generation pipeline) and 4 (RLVR training with GRPO) for implementation details. Section 5.3 contains the key ablation showing SFT harms performance while RLVR helps.