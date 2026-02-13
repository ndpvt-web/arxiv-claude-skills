---
name: "hidden-licensing-risks-llmware"
description: "Detect license incompatibilities across LLM supply chains (OSS repos, models, datasets) using the LiAgent multi-agent extraction and compatibility analysis framework. Use when: 'check my project for license conflicts', 'are my HuggingFace model dependencies compatible', 'audit LLM supply chain licenses', 'find licensing risks in my ML pipeline', 'is this model license compatible with my repo', 'analyze license compatibility for my AI project'."
---

# Hidden Licensing Risks in LLMware: Supply Chain License Compatibility Analysis

This skill enables Claude to perform ecosystem-level license compatibility analysis across LLM supply chains — tracing dependencies from source code repositories through HuggingFace models down to training datasets, extracting license terms, and detecting incompatibilities using the LiAgent framework from Wang et al. (2026). Unlike traditional OSS license checkers that only handle code dependencies, this approach handles the three-layer supply chain unique to LLM-powered software: OSS code -> LLM models -> base models and training datasets, including AI-specific licenses (LLaMA, OpenRAIL, Gemma) that conventional tools miss entirely.

## When to Use

- When a user asks to audit license compatibility of a project that depends on HuggingFace models
- When checking whether a model's license is compatible with its training dataset's license
- When a user wants to know if their MIT/Apache-2.0 repo can legally use a specific LLM
- When reviewing a `requirements.txt` or code that calls `from_pretrained()` or `pipeline()` for licensing risks
- When a user is choosing between models and license compatibility is a factor
- When creating or publishing a model and selecting an appropriate license given upstream dependencies
- When a user reports a license conflict warning and needs help understanding the supply chain implications

## Key Technique: LiAgent Multi-Agent License Analysis

The core insight from the paper is that 52% of 68,778 LLM supply chains exhibit license conflicts, yet existing detection tools achieve only 58-76% F1 because they cannot handle AI-specific licenses or the three-layer dependency structure of LLM software. LiAgent solves this with a two-agent architecture:

**Extraction Agent**: Analyzes full license text against 23 predefined legal terms (commercial use, distribution, modification, patent grant, attribution, etc.) and classifies each term's attitude as `can`, `cannot`, or `must`. This structured extraction handles both traditional OSS licenses and AI-specific licenses like LLaMA2, OpenRAIL++, and CreativeML Open RAIL-M that conventional SPDX matchers cannot parse.

**Compatibility Logic**: Once terms are extracted for each artifact in the supply chain, a downstream-vs-upstream compatibility matrix determines conflicts. The rule is: a downstream license must not be more permissive than its upstream dependency. Specifically — if upstream says `cannot` (prohibits something), downstream saying `can` or `must` for that same term is a conflict. If upstream says `must` (requires something), downstream saying `can` or `cannot` is a conflict. Undefined terms default to `cannot` for rights and `can` for obligations, erring on the restrictive side.

## Step-by-Step Workflow

1. **Identify the supply chain layers.** Parse the user's project to find three dependency types: (a) which HuggingFace models the code references (look for `from_pretrained()`, `pipeline()`, `AutoModel`, `AutoTokenizer` calls), (b) what base models those HuggingFace models derive from (check model card metadata), and (c) what training datasets were used.

2. **Collect license declarations for each artifact.** For the OSS repo, check `LICENSE`, `LICENSE.md`, `package.json`, or `pyproject.toml`. For HuggingFace models, check the model card's `license` field. For datasets, check the dataset card. Flag any artifact with no license declaration — 35.4% of LLMware components lack one, and unlicensed code is legally "all rights reserved" by default.

3. **Classify each license.** Categorize into: traditional OSS (MIT, Apache-2.0, GPL-3.0, BSD-3-Clause), content licenses (CC-BY-4.0, CC-BY-NC-4.0, CC-BY-SA-4.0, CC0), AI-specific licenses (LLaMA2/3, OpenRAIL, OpenRAIL++, Gemma, CDLA-Perm-2.0, BigScience RAIL), or unlicensed.

4. **Extract legal terms and attitudes.** For each license, determine the attitude (`can`/`cannot`/`must`) for key terms: commercial use, distribution, modification, sublicensing, patent grant, attribution, share-alike, no-derivatives, and any use restrictions specific to AI (deployment constraints, acceptable use policies, output ownership).

5. **Check consistency within each license.** If a single license contains contradictory attitudes for the same term across different clauses, flag this and attempt resolution by re-reading surrounding context (up to 3 rounds). For example, an OpenRAIL license may broadly permit commercial use but restrict specific deployment scenarios.

6. **Apply the compatibility matrix across each supply chain edge.** For every dependency pair (repo->model, model->dataset, model->base-model), compare term attitudes using this rule set:
   - Upstream `can` + Downstream `can`/`cannot`/`must` = all compatible
   - Upstream `cannot` + Downstream `can` or `must` = **CONFLICT**
   - Upstream `must` + Downstream `can` or `cannot` = **CONFLICT**
   - Upstream `must` + Downstream `must` = compatible (same obligation preserved)

7. **Report conflicts with specific terms and remediation options.** For each conflict, identify the exact legal term, the two licenses involved, the direction of the dependency, and suggest concrete fixes: relicense the downstream component, switch to a compatibly-licensed alternative model/dataset, or contact the upstream maintainer.

8. **Assess downstream impact.** Note download counts and dependent repos for conflicted artifacts. A conflict in a model with millions of downloads has ecosystem-wide implications.

9. **Generate a structured compatibility report.** Output a table listing each supply chain edge, the licenses on each side, conflicting terms, severity (blocking vs. advisory), and recommended actions.

## Concrete Examples

**Example 1: Auditing a Python project using HuggingFace models**

User: "Check if my project has any license conflicts. It's MIT-licensed and uses `meta-llama/Llama-2-7b-chat-hf` via transformers."

Approach:
1. Project license: MIT (permissive — allows commercial use, modification, distribution)
2. Model license: LLaMA 2 Community License (AI-specific — restricts commercial use above 700M monthly active users, requires acceptable use policy compliance, prohibits use to train other LLMs)
3. Supply chain edge: MIT repo -> LLaMA 2 model
4. Term comparison:
   - Commercial use: MIT = `can` (unrestricted), LLaMA 2 = `can` with conditions (700M MAU threshold) -> **potential conflict**: MIT downstream promises unrestricted commercial use but upstream LLaMA 2 imposes conditions
   - Sublicensing: MIT = `can`, LLaMA 2 = `cannot` (no sublicensing to third parties without Meta agreement) -> **CONFLICT**
   - Use restrictions: MIT = none, LLaMA 2 = `must` follow acceptable use policy -> **CONFLICT**: MIT makes no such obligation on downstream users

Output:
```
LICENSE COMPATIBILITY REPORT
============================
Project: user-repo (MIT)
  -> meta-llama/Llama-2-7b-chat-hf (LLaMA 2 Community License)

CONFLICTS FOUND: 2 blocking, 1 advisory

| # | Term            | Upstream (LLaMA 2) | Downstream (MIT) | Status       |
|---|-----------------|---------------------|-------------------|--------------|
| 1 | Sublicensing    | cannot              | can               | CONFLICT     |
| 2 | Use restrictions| must (AUP)          | can (none)        | CONFLICT     |
| 3 | Commercial use  | can (conditional)   | can (unrestricted) | ADVISORY    |

RECOMMENDATION:
- Relicense your project under a custom license that passes through
  LLaMA 2's restrictions to downstream users, OR
- Switch to an Apache-2.0 licensed alternative (e.g., mistralai/Mistral-7B-v0.1)
```

**Example 2: Checking a model's compatibility with its training data**

User: "I fine-tuned an Apache-2.0 model on the `ultrachat_200k` dataset. Any issues?"

Approach:
1. Model license: Apache-2.0 (permissive, patent grant, attribution required)
2. Dataset license: Check HuggingFace — `ultrachat_200k` is MIT-licensed
3. Supply chain edge: Apache-2.0 model -> MIT dataset
4. Term comparison:
   - Patent grant: Apache-2.0 = `must` (explicit patent grant), MIT = `can` (silent on patents) -> **CONFLICT**: upstream dataset makes no patent grant, but downstream model license promises one
   - Attribution: Apache-2.0 = `must`, MIT = `must` -> compatible

Output:
```
LICENSE COMPATIBILITY REPORT
============================
Model: your-model (Apache-2.0)
  -> stingning/ultrachat_200k (MIT)

CONFLICTS FOUND: 1 advisory

| # | Term         | Upstream (MIT) | Downstream (Apache-2.0) | Status   |
|---|--------------|----------------|--------------------------|----------|
| 1 | Patent grant | can (silent)   | must (explicit grant)    | ADVISORY |

NOTE: MIT's silence on patents means no patent rights are explicitly
granted from the dataset. Your Apache-2.0 license promises patent
grants to users, but you can only grant patents you hold yourself.
This is low-risk for datasets but worth documenting.
```

**Example 3: Scanning a requirements file for model licensing risks**

User: "Scan my code for HuggingFace model references and flag any license issues. My project is GPL-3.0."

Approach:
1. Search code for `from_pretrained()`, `pipeline()`, `AutoModel`, `AutoTokenizer` calls
2. Extract model identifiers from string arguments
3. Query HuggingFace API for each model's license
4. Compare each model's license against GPL-3.0

Output:
```
SUPPLY CHAIN SCAN RESULTS
==========================
Project license: GPL-3.0 (strong copyleft)

Models found in codebase:
| Model ID                          | License      | Compatible? |
|-----------------------------------|-------------|-------------|
| bert-base-uncased                 | Apache-2.0  | YES         |
| openai/whisper-large-v3           | Apache-2.0  | YES         |
| stabilityai/stable-diffusion-xl   | OpenRAIL-M  | REVIEW      |
| meta-llama/Llama-2-7b             | LLaMA 2     | NO          |

DETAILS:
- stabilityai/stable-diffusion-xl (OpenRAIL-M): Contains use-based
  restrictions that GPL-3.0's "freedom to use for any purpose" cannot
  pass through. Requires manual review of specific restricted uses.
- meta-llama/Llama-2-7b (LLaMA 2): Prohibits use to "improve other
  LLMs" and requires Meta's acceptable use policy. GPL-3.0 cannot
  impose these additional restrictions on downstream users.
```

## Best Practices

- **Do:** Always check for unlicensed artifacts. 35% of LLMware components have no license, which legally means "all rights reserved" — not "free to use."
- **Do:** Trace the full supply chain depth. A model may be Apache-2.0, but if its base model is LLaMA-licensed, those restrictions propagate downstream.
- **Do:** Treat AI-specific licenses (OpenRAIL, LLaMA, Gemma) as their own category. They combine permissive code terms with use-based restrictions that have no equivalent in traditional OSS licensing.
- **Do:** Default undefined terms to `cannot` for rights and `can` for obligations. This conservative interpretation avoids false negatives.
- **Avoid:** Assuming two permissive licenses are automatically compatible. MIT and Apache-2.0 differ on patent grants, which creates subtle conflicts.
- **Avoid:** Treating model and dataset licenses as equivalent to software licenses. Content licenses (CC-BY) and software licenses (MIT) operate on different legal theories and mixing them creates ambiguity.

## Error Handling

- **Model not found on HuggingFace**: The model ID in code may be a local path, a custom hub, or a typo. Flag as "unresolvable" and ask the user to provide the license manually.
- **License text unavailable**: Some models declare a license type but don't include the full text. Use the SPDX identifier to infer standard terms, but warn that custom modifications may exist.
- **Ambiguous license attitudes**: When a license clause is genuinely ambiguous about whether something is permitted, report it as "REVIEW NEEDED" rather than forcing a binary classification. The repair agent strategy (re-reading context up to 3 times) helps but cannot resolve genuinely vague legal language.
- **Private or gated models**: Gated models on HuggingFace require acceptance of terms before access. Note that accepting those terms may impose additional obligations beyond the stated license.
- **Multiple licenses**: Some artifacts declare dual licensing (e.g., "MIT OR Apache-2.0"). Analyze both options and report which choice minimizes conflicts.

## Limitations

- This approach analyzes license text, not legal enforceability. License compatibility is a legal question that ultimately requires legal counsel for high-stakes decisions.
- AI-specific licenses are new and untested in court. Compatibility analysis is based on textual term extraction, not legal precedent.
- The 23 predefined legal terms cover the most common licensing dimensions but may miss niche restrictions in custom licenses.
- Training data provenance is often incomplete. Many models on HuggingFace do not declare their training datasets, making full supply chain analysis impossible.
- License compatibility for model outputs (generated text, images) is an unsettled legal area not covered by this framework.
- This analysis covers direct dependencies only. Transitive dependencies through model merging, distillation, or RLHF reward models may introduce additional licensing layers not captured here.

## Reference

Wang, B., Chen, Y., Shi, J., Li, M., & Lyu, Y. (2026). *Hidden Licensing Risks in the LLMware Ecosystem.* arXiv:2602.10758v1. [https://arxiv.org/abs/2602.10758v1](https://arxiv.org/abs/2602.10758v1)

Key sections: Section IV (LiAgent framework and compatibility matrix), Section III (supply chain construction methodology), Table IV (conflict patterns by dependency type), and Section V (real-world confirmed conflicts and developer recommendations).